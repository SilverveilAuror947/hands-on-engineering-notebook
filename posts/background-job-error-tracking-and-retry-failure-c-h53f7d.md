# Background Job Error Tracking and Retry Failure Capture Explained

Short answer: a Node.js Postgres cron worker should capture every background job retry failure as one durable, structured event with a stable run ID, then forward a redacted exception or increment an alert metric. For a nightly fintech pipeline, attach tenant, cost-center, worker, and attempt fields at capture time. The event is the attribution record; the alert is only the wake-up signal.

| Pick this approach | When it fits | Main trade-off | Cost attribution quality |
| --- | --- | --- | --- |
| Durable failure ledger plus alerts | Finance or platform teams must explain retry work by tenant, job, or cost center | Another table, retention policy, and cleanup path | High, if attribution fields are required at write time |
| Exception forwarding plus metrics | Fast diagnosis matters more than allocating retry activity | Vendor-neutral allocation queries are harder later | Medium, depending on labels retained |
| Metrics only | A coarse service-level failure rate is enough | No per-run evidence or error context | Low |

## How should a Node.js Postgres cron worker implement background job retry failure capture?

Treat the worker as three connected paths. In words: the cron trigger creates a run; the run creates numbered attempts; every failed attempt emits a compact event to a store outside the business transaction. From that event, a separate path can notify an error tracker and update metrics. BullMQ, Agenda, or a small Postgres-backed scheduler can own the trigger and retry timing. They should not be the only place where failure evidence lives. The boundary matters because a transaction rollback should undo partial ledger changes, not the evidence that the attempt failed. If the same database transaction contains both the business update and its failure event, the rollback erases the exact record needed for diagnosis. Use a separate short transaction or connection for the observability insert after the business transaction has rolled back.

The rollback is the trap.

A useful event answers four questions without opening a free-form message: which run failed, which numbered attempt failed, who should bear the work, and what class of failure occurred? For the nightly reconciliation job, that means fields such as `run_id`, `attempt`, `job_name`, `tenant_id`, `cost_center`, `error_code`, `retryable`, `duration_ms`, and `occurred_at`. A synthetic example might describe attempt 3 of run `recon_2026_08_17_01` with error code `UPSTREAM_TIMEOUT`, cost center `payments-ops`, and a 42,318 ms duration. Those numbers demonstrate shape, not a benchmark or production incident. Don't copy payment payloads, access tokens, database connection strings, or full stack-local context into that record. The OWASP Logging Cheat Sheet calls out data that should usually be removed, masked, sanitized, hashed, or encrypted, and it also recommends protecting logs against tampering and unauthorized access. In a fintech system, a tenant identifier can support attribution while a raw account number creates avoidable exposure. Define that line before deployment.

## How should the team attribute the cost of each retry?

Pick the durable ledger when an engineer, finance partner, or service owner will ask, “Why did this tenant's nightly run consume six attempts?” The schema makes that query possible without parsing prose. Aggregate attempt count and duration by `cost_center`, `tenant_id`, and `job_name`; keep the original event keyed by run and attempt for investigation. This is cost attribution in operational units. It does not pretend that duration equals a currency amount, because infrastructure pricing and shared overhead may be allocated elsewhere.

Pick exception forwarding when the immediate question is, “Which code path failed?” Stack traces and grouping are useful for diagnosis, but the catch is that a diagnostic tool may not retain the stable business dimensions needed for internal allocation. Send the same `run_id` and safe attribution keys with the exception so an investigator can join it back to the ledger. Do not make a monitoring destination the system of record for financial allocation unless its retention, export, access, and deletion behavior has been reviewed for that purpose.

Pick metrics alone only when aggregate rates are enough. A counter labeled by job and error class can drive an alert, but tenant IDs and run IDs often have too many possible values for sensible metric labels. Metrics also lose the sequence of attempt-level evidence. The cheap-looking option becomes awkward as soon as someone asks which retry belonged to which run.

Choose deliberately.

This is a real trade-off. A durable ledger is not suitable when the job volume is tiny, no team needs per-tenant allocation, and an existing exception stream already meets the audit and retention requirements. In that case, stick with exception forwarding and a low-cardinality failure counter. At the other extreme, metrics-only is not suitable when disputed allocation or regulated data handling requires an inspectable record.

## How can the team build and test a durable attempt event?

The focused implementation below uses a generic Postgres pool. The unique key is `(run_id, attempt)`, so a repeated observer call does not create a second chargeable-looking failure event. The business work and failure insert use separate connections. Replace the synthetic `executeNightlyReconciliation` body with the actual job, and let the scheduler decide when the next attempt should run.

```ts
import { randomUUID } from "node:crypto";
import { Pool } from "pg";

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

const failureSchema = `
  CREATE TABLE IF NOT EXISTS job_attempt_failures (
    run_id text NOT NULL,
    attempt integer NOT NULL CHECK (attempt > 0),
    event_id uuid NOT NULL,
    job_name text NOT NULL,
    tenant_id text NOT NULL,
    cost_center text NOT NULL,
    error_code text NOT NULL,
    retryable boolean NOT NULL,
    duration_ms integer NOT NULL CHECK (duration_ms >= 0),
    occurred_at timestamptz NOT NULL,
    PRIMARY KEY (run_id, attempt)
  )
`;

type JobContext = {
  runId: string;
  attempt: number;
  tenantId: string;
  costCenter: string;
};

type SafeFailure = {
  code: "UPSTREAM_TIMEOUT" | "VALIDATION_REJECTED" | "UNKNOWN";
  retryable: boolean;
};

function classifyFailure(error: unknown): SafeFailure {
  if (error instanceof Error && error.name === "TimeoutError") {
    return { code: "UPSTREAM_TIMEOUT", retryable: true };
  }
  if (error instanceof Error && error.name === "ValidationError") {
    return { code: "VALIDATION_REJECTED", retryable: false };
  }
  return { code: "UNKNOWN", retryable: false };
}

async function recordFailure(
  context: JobContext,
  failure: SafeFailure,
  durationMs: number,
): Promise<void> {
  await pool.query(
    `INSERT INTO job_attempt_failures (
       run_id, attempt, event_id, job_name, tenant_id, cost_center,
       error_code, retryable, duration_ms, occurred_at
     ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, NOW())
     ON CONFLICT (run_id, attempt) DO NOTHING`,
    [
      context.runId,
      context.attempt,
      randomUUID(),
      "nightly_reconciliation",
      context.tenantId,
      context.costCenter,
      failure.code,
      failure.retryable,
      durationMs,
    ],
  );
}

async function executeNightlyReconciliation(
  client: Awaited<ReturnType<Pool["connect"]>>,
  context: JobContext,
): Promise<void> {
  await client.query(
    "UPDATE reconciliation_runs SET checked_at = NOW() WHERE run_id = $1",
    [context.runId],
  );
}

export async function runAttempt(context: JobContext): Promise<void> {
  const startedAt = Date.now();
  const client = await pool.connect();

  try {
    await client.query("BEGIN");
    await executeNightlyReconciliation(client, context);
    await client.query("COMMIT");
  } catch (error: unknown) {
    await client.query("ROLLBACK");
    const failure = classifyFailure(error);

    await recordFailure(context, failure, Date.now() - startedAt);
    throw error;
  } finally {
    client.release();
  }
}

export async function prepareFailureStore(): Promise<void> {
  await pool.query(failureSchema);
}
```

The scheduler wrapper should create `runId` once and increment `attempt` for each retry. Preserve both values across process restarts. A retry policy should consume `retryable`; it should not infer retryability from the human-readable error message. For a validation rejection, stop. For a timeout classified as retryable, schedule the next attempt under the scheduler's policy and keep the original run ID. Two failure paths deserve tests. First, force `executeNightlyReconciliation` to throw after its update and verify that the business update rolls back while one failure row remains. Second, call `recordFailure` twice for the same run and attempt and verify that the primary key leaves one row. Add a third test asserting that raw input data and the original error message never enter the insert parameters. A crisp before-and-after check catches more than a screenshot of an alert.

One row. One attempt.

## Set the operational boundary before deployment

Deployment order is short: create the table, deploy code that can write it, then enable alerting from the resulting event stream. Restrict read access, monitor insert failures through a separate low-cardinality signal, and define retention before the table grows. This pattern captures application-visible attempts. It does not prove that every scheduled run started, so pair it with a run manifest or heartbeat if missing executions matter. It also does not allocate shared database, queue, or networking overhead by itself. Those costs need an explicit allocation rule, and that rule should be versioned so a month-to-month report can explain changes. I'm not sure a universal retention window exists for this data; the right period depends on the purpose, applicable legal duties, and the evidence your organization must retain. GDPR Article 17 defines a right to erasure and also lists exceptions, so counsel and data governance should resolve the policy rather than a worker library default.

Scope matters.

The concise decision rule is: use durable attempt events when per-tenant or per-cost-center questions must be answered later; use exception forwarding for diagnostic depth; use low-cardinality metrics for alerting. Combine them when the nightly pipeline is material enough to justify the operational surface. Keep sensitive transaction data out of all three.

## Further reading

The OWASP Logging Cheat Sheet is the practical starting point for deciding which fields must be excluded or transformed. GDPR Article 17 frames the erasure question and its exceptions; retention policy still needs review against the system's purpose and legal obligations.

## References

- OWASP Logging Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- GDPR Article 17: https://gdpr-info.eu/art-17-gdpr/
