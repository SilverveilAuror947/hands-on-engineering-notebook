# Poison Message Governance When Background Job Queue Retries Won't Stop

Short answer: give each reservation-expiry job a durable attempt budget and each dead-letter redrive a separate replay budget, then make the worker record one terminal decision when either limit is reached.

For a game inventory hold, this is a latency-versus-cost control. Retrying a temporary failure quickly can release scarce inventory closer to its expiry time. Retrying an invalid payload forever only burns worker time and hides the message that needs investigation. Backoff changes *when* another attempt happens; it doesn't decide *whether* another attempt is allowed.

The useful before/after model is small. Before: receive, fail, delay, repeat. After: receive, classify, spend one durable attempt, then complete, retry, or quarantine. Redrive starts a new, explicitly approved phase without erasing the original history.

## Who should authorize a dead-letter redrive?

Treat dead-letter redrive as a governed operation, not another branch of automatic retry. The policy needs two finite budgets: `maxAttempts` for ordinary execution and `maxRedrives` for reviewed replay. It also needs a named owner who can decide whether the underlying condition changed enough to justify replay.

| Failure evidence | Automatic retry | Redrive after review | Required action |
| --- | --- | --- | --- |
| Temporary dependency timeout | Within attempt budget | Possibly | Check dependency recovery |
| Invalid payload | No | Not before correction | Fix producer or data |
| Reservation already confirmed | No | No | Complete as a no-op |
| Attempt budget exhausted | No | Only within replay budget | Review the full lineage |

A dead-letter queue is neither success nor permanent storage. Give it a retention policy, an age alert, searchable failure evidence, and a documented choice among discard, correction, and redrive. A replay must preserve the original `jobId`, increment `redriveCount`, and retain the earlier attempts. Otherwise the dashboard can look healthy while one poison message cycles through fresh identities.

Keep the rule visible: **normal attempts and redrives are two finite budgets**.

## What evidence shows why background job queue retries are not stopping?

When retries don't stop, inspect ownership of the counter before tuning a delay. A process-local counter disappears on restart. A delivery counter may describe transport activity rather than business attempts. A copied dead-letter message may also acquire a new delivery identity. None of those values can enforce a lifetime limit unless the queue contract explicitly says they survive the transitions your system performs.

Follow one stable `jobId` from its first scheduled run to its final state. The event stream should contain `attempt`, `maxAttempts`, `redriveCount`, `failureCode`, and `nextState` at every decision. Keep reservation IDs out of metric labels; put them in structured logs where an operator can search a single lineage. Metrics need bounded dimensions such as failure class and outcome.

One trace tells a lot:

```ts
type JobDecisionEvent = {
  jobId: string;
  attempt: number;
  maxAttempts: number;
  redriveCount: number;
  failureCode: "DEPENDENCY_TIMEOUT" | "INVALID_PAYLOAD";
  nextState: "retry_scheduled" | "dead_lettered";
};

const example: JobDecisionEvent[] = [
  {
    jobId: "expire-reservation-8f31",
    attempt: 4,
    maxAttempts: 4,
    redriveCount: 0,
    failureCode: "DEPENDENCY_TIMEOUT",
    nextState: "dead_lettered",
  },
];
```

If a later event for that `jobId` says `attempt: 1` without an intentional redrive transition, the lineage was reset. If `attempt` equals `maxAttempts` and `nextState` still says `retry_scheduled`, components disagree about whether the limit is inclusive. Write that boundary down. Then test it.

This is the crisp diagnostic: counters describe policy only when they are durable across every path that can re-enqueue work.

## Make the policy a pure TypeScript decision

Put classification and the terminal branch in a pure transition function. Don't scatter `maxAttempts` checks across exception handlers. The function below has no queue SDK and performs no I/O. Given the same job and failure, it always returns the same next action, so boundary tests don't need a running worker.

```ts
type FailureCode = "DEPENDENCY_TIMEOUT" | "INVALID_PAYLOAD";

type ExpiryJob = {
  jobId: string;
  reservationId: string;
  attempt: number;
  maxAttempts: number;
  redriveCount: number;
  maxRedrives: number;
};

type RetryDecision =
  | { kind: "retry"; nextAttempt: number; delayMs: number }
  | { kind: "dead_letter"; reason: "permanent" | "attempts_exhausted" };

const BASE_DELAY_MS = 1_000;
const MAX_DELAY_MS = 30_000;

function retryDelayMs(nextAttempt: number): number {
  return Math.min(BASE_DELAY_MS * 2 ** (nextAttempt - 1), MAX_DELAY_MS);
}

function decideAfterFailure(
  job: ExpiryJob,
  code: FailureCode,
): RetryDecision {
  if (code === "INVALID_PAYLOAD") {
    return { kind: "dead_letter", reason: "permanent" };
  }

  const nextAttempt = job.attempt + 1;
  if (nextAttempt > job.maxAttempts) {
    return { kind: "dead_letter", reason: "attempts_exhausted" };
  }

  return {
    kind: "retry",
    nextAttempt,
    delayMs: retryDelayMs(nextAttempt),
  };
}
```

The sample chooses attempt one as the first execution. `maxAttempts` therefore includes that execution. It treats malformed input as permanent and a dependency timeout as retryable; real classification needs the contracts of the dependencies in your own stack. I'm not sure a generic handler can infer that safely from every thrown JavaScript value, so resolve ambiguity at adapters and pass a narrow failure code into the policy.

The executor then applies the returned action and records it durably. It must reload the reservation and conditionally release only a still-held, expired version. That protects a confirmed or renewed hold from a late worker. Applying the decision and acknowledging the delivery must also be coordinated so a crash cannot leave contradictory outcomes. Depending on the storage and queue, that coordination can be an atomic queue operation, a transaction, or an outbox-style handoff. The invariant matters more than the mechanism: one consumed attempt creates one durable next state.

Keep redrive outside this handler. An operator or a reviewed automation should copy a dead-lettered job only when `redriveCount < maxRedrives`, increment the count, retain the stable `jobId`, and preserve the prior failure evidence. The catch is straightforward — malformed payloads and broken business invariants are not suitable for automatic redrive. Fix or correct their cause first; otherwise replay just buys another loop.

## Turn the charter into executable review checks

Test the ledger, not merely the thrown error. Start one case at the attempt immediately below the limit and assert that a temporary failure schedules exactly one retry. Start another at the limit and assert that the next failure produces `dead_lettered`, with no retry record. Feed `INVALID_PAYLOAD` on the first execution and expect immediate quarantine. Then replay a quarantined job at the redrive ceiling and verify that policy refuses a new generation.

Race tests matter in this gaming scenario. Confirm a reservation after its expiry job was enqueued but before the worker reads it; the worker should complete as a no-op. Change the version between the read and conditional release; again, no newer state should be overwritten. Finally, inject a process stop between each durable state change and acknowledgement. The recovered job must converge on the same terminal outcome without consuming an invisible extra budget.

That's the suite.

## Answer the latency and cost objections together

Sometimes, but the gain has a ceiling. Exponential backoff spaces repeated attempts with progressively longer delays. It can reduce repeated contention during a temporary failure, while the attempt budget supplies the stopping condition. Those are separate controls. A short delay may improve expiry lag after a brief dependency interruption, yet it increases executions and datastore reads. Once the expiry-lag objective is met, more aggressive retries are extra cost rather than extra correctness.

Measure both sides on the same dashboard: the distribution of `releasedAt - expiresAt`, worker executions per completed expiry, datastore operations per expiry, dead-letter rate by bounded failure code, and oldest quarantined-message age. Logs explain one lineage. Metrics show whether a policy change moved the population. Alerts should use rates, lag, and age; one isolated retry is expected behavior, not a page.

A periodic reconciliation sweep is the other lever. Cron is a conventional way to run commands on a schedule, so a sweep can query still-held reservations whose expiry time has passed. Its interval imposes delay, and each scan costs datastore work. Per-reservation scheduling aims for lower latency but creates one scheduled job per hold. A hybrid provides a slower recovery path behind the primary jobs, at the cost of operating both paths.

Choose from the inventory promise. Use per-reservation jobs when prompt release matters and the system can carry the scheduling volume. Stick with an indexed sweep when bounded delay is acceptable and scanning is cheaper for the actual workload. Use the hybrid only when the additional recovery coverage justifies its operational cost. There isn't a universal interval or retry count; load tests and the game's tolerated expiry lag have to settle those numbers.

## Further reading

- Cron: https://en.wikipedia.org/wiki/Cron
- Exponential backoff: https://en.wikipedia.org/wiki/Exponential_backoff
