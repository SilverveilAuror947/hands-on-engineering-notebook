# Node.js Domain Verification Polling for Scheduled Retries and Customer Rechecks

Short answer: treat domain verification as an intent-reconciliation loop, not as a timer that eventually says yes. Store the exact DNS record the media system expects, resolve the relevant record type on a bounded schedule, and let a customer-triggered check run the same verifier immediately. The useful result is a transition with evidence: pending, verified, or actionable mismatch.

That is the contract.

No magic timer.

That distinction matters during a namespace migration. A registrar-specific API may tell an onboarding service that a change was submitted, but submission is not the same thing as the record published for resolvers to observe. The system of record is the DNS answer at the time of the check.

Use two snapshots. The desired snapshot is an immutable verification intent: owner name, record type, expected value, creation time, and an expiration or cancellation state. The observed snapshot is the answer returned by a DNS resolver, plus the resolver context and timestamp. Reconciliation compares those snapshots; it does not infer success from a previous attempt.

For a media company moving a zone between providers, that gives a clean before-and-after mental model. Before: an onboarding record is waiting, and each retry records what was seen. After: the record matches the intent, so downstream activation can proceed. A stale value, an empty answer, or a lookup error remains visible as evidence rather than being flattened into “not ready.”

DNS names and values deserve normalization, but only the normalization that the record type permits. A TXT answer can be represented as one logical value after the resolver library has assembled its character strings; comparison should still be explicit about whitespace and quoting. Do not lowercase a value merely because DNS names are case-insensitive. A verification token is data.

The same principle applies to DMARC policy records. RFC 7489 defines a DMARC record as a DNS TXT record and describes discovery under the `_dmarc` subdomain. That is a standards boundary worth keeping in the verifier: construct the owner name from the domain, query TXT, and validate the content you actually received.

## Why should scheduled polling retries support customer domain verification?

Scheduled retries absorb the normal delay without making a customer sit on an open browser tab. Customer-triggered rechecks handle the moment a person has just changed DNS and wants an answer. They should share one verification function and differ only in how work enters the queue.

The retry policy needs a budget. For example, an installation can receive attempts at 1 minute, 5 minutes, 15 minutes, and then a slower cadence until its deadline. Those intervals are policy, not a claim about universal DNS propagation time. Add jitter so a migration affecting many zones does not release a synchronized burst of lookups. Stop at the deadline, and expose the next action instead of retrying forever.

A small TypeScript core keeps the boundary testable:

```ts
type Intent = {
  name: string;
  type: "TXT" | "CNAME";
  expected: string;
  deadline: number;
};

type Observation = {
  values: string[];
  checkedAt: number;
  resolver: string;
};

function reconcile(intent: Intent, observation: Observation, now: number) {
  const matches = observation.values.includes(intent.expected);

  if (matches) return { state: "verified" as const, observation };
  if (now >= intent.deadline) {
    return { state: "expired" as const, observation };
  }
  return { state: "pending" as const, observation };
}
```

The queue worker should persist the observation before emitting a state change. A customer recheck can then return the same evidence model as a scheduled attempt. If two attempts overlap, an idempotency key based on the intent identifier and check window prevents duplicate activation events. The important event is “verified from this observation,” not “job ran successfully.”

That boundary is the point.

## What does a useful failure look like?

A timeout is not a diagnosis. The operator needs to distinguish no answer, an answer containing the wrong token, a resolver failure, and a valid answer that arrived after the previous scheduled attempt. Those cases lead to different customer actions.

Metrics should count verification outcomes by record type and state, with latency from intent creation to verified observation. Logs should include a redacted intent identifier, queried name, record type, attempt number, resolver context, and decision. Never put a private onboarding token into a dashboard label.

Alerts belong on symptoms that require an operator: an unusual rise in expired intents, a growing queue age, or repeated resolver errors. A single pending domain is a customer workflow, not an incident. This is where the before-and-after view pays off: the alert compares system behavior over time, while the customer sees the concrete record mismatch.

## Does more polling make propagation faster?

No. Polling changes when the service observes DNS; it does not publish the record or control every cache between the authoritative server and a recursive resolver. More frequent checks can add load and create the illusion of progress. A bounded schedule, clear evidence, and a manual recheck are more honest.

The verifier should also avoid treating one resolver response as universal truth. Record visibility can differ by resolver and cache state. If the migration is high risk, make resolver selection an explicit policy and record it with the observation. A disagreement should produce a reviewable state, not an automatic success.

The durable choice is the workflow with one comparison rule, two entry paths, a finite retry budget, and evidence attached to every decision. Scheduled work gives the platform continuity; a customer action gives it responsiveness. Neither should bypass the same record-level validation.

For a media namespace migration, that design also keeps ownership clear. The DNS operator publishes the record. The onboarding service observes and reconciles it. The customer can ask for another observation. Once those responsibilities are separate, a registrar API becomes an optional change interface rather than the authority for verification.

The tradeoff is real: this design is a poor fit for a one-off internal hostname where an operator already has direct visibility and no customer workflow. A small script may be enough there. It earns its complexity when many domains, asynchronous changes, and customer-visible activation states must remain explainable.

## References

- RFC 7489, “Domain-based Message Authentication, Reporting, and Conformance (DMARC)”: https://datatracker.ietf.org/doc/html/rfc7489
- RFC 1034, “Domain Names — Concepts and Facilities”: https://datatracker.ietf.org/doc/html/rfc1034
- RFC 1035, “Domain Names — Implementation and Specification”: https://datatracker.ietf.org/doc/html/rfc1035
