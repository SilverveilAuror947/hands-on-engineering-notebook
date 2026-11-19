# Compliance Evidence: 5 Checks Before SMS Timeout Retry Causes Duplicate Sends

Record a compliance notice's delivery intent before sending SMS event notifications, and reconcile an ambiguous timeout before authorizing a retry. The deciding constraint is evidence: a developer tools team must be able to explain what it tried, what it observed, and what remains unknown.

TL;DR: A timeout leaves the send outcome unresolved. Keep one durable operation per notice, recipient, and channel. Reuse its idempotency key only within the sender's documented guarantees, poll for evidence when lookup is supported, and retain the observations behind every status change. An accepted request must never silently become a claim that the recipient read the notice.

The teaching example below concerns a compliance notice for a developer workspace. Its identifiers and interfaces are illustrative application design, not a commercial API contract. The goal is an auditable delivery record, including the uncomfortable case where the available evidence cannot establish delivery.

## 1. What does a timeout actually prove?

It proves uncertainty.

Before: notice event → SMS request → timeout → another SMS request. That mental model makes the network response the only memory of the operation. If the first request took effect before the response was lost, the second attempt can create a duplicate.

After: notice event → durable delivery intent → send attempt → recorded observation → reconciliation → explicit next decision. The record survives a missing response. This is the useful diagram-in-words: the send is one activity inside an evidence trail, and a timeout is an observation inside that activity.

For this design, separate three questions. Did the application record an intent? Did the sender accept responsibility for a message? What subsequent delivery evidence arrived? Define the answers locally, then map external statuses to those definitions. Keep the original external status alongside the mapping so a future reviewer can inspect what the application actually received. Avoid inventing a universal SMS state vocabulary.

There is a specific trap here: an HTTP success from your own endpoint can mean only that the intent was durably recorded. If a dashboard labels that response “delivered,” the evidence has already been overstated, even if the SMS eventually arrives.

## 2. Give each notice one durable identity

For the example, use a unique delivery identity derived from a notice revision, an internal recipient reference, and the channel. A repeated event for the same revision should resolve to the same operation. A revised notice should get a new identity. Decide that boundary before adding retry logic; otherwise a harmless event replay and a legitimate new communication look identical.

Keep the destination and rendered message attached to that operation as an immutable snapshot under your access policy. If the recipient's phone number changes after a timeout, retrying the old key with the new number creates an ambiguous audit record. A deliberate correction needs an explicit relationship to the earlier operation.

Enforce uniqueness in durable storage. A process-local map cannot provide that guarantee across a restart or multiple workers. Couple event acknowledgement to the durable intent, and make worker ownership an atomic storage operation. The storage contract matters more than the choice of web framework.

The trade-off is more storage coordination and operational work. This design is not suitable for establishing human acknowledgement when the channel supplies only transport evidence. If acknowledgement is required, define a separate authenticated acknowledgement step and connect its record to the notice. Keep phone numbers and notice bodies out of ordinary logs; use restricted records for the evidence that actually requires them.

Identity comes first.

## 3. Make the request handler record intent, then return its identity

Here is a focused TypeScript example for a Node.js application using Express request and response types. It intentionally stops at the durable queue boundary. The worker owns network calls and reconciliation, so an inbound HTTP retry cannot directly trigger a second SMS request.

The example assumes authentication and authorization run first. Its store is an application interface: implement the uniqueness check, immutable payload comparison, and initial work scheduling in one transaction. `conflict` means an existing identity was presented with different content or a different destination. Never silently accept that mismatch.

```ts
import type { Request, Response } from "express";

type Notice = {
  noticeRevision: string;
  recipientRef: string;
  destination: string;
  body: string;
};

type IntentResult =
  | { kind: "recorded"; operationId: string }
  | { kind: "conflict" };

interface IntentStore {
  // Atomically ensure one immutable intent and its initial work item.
  ensureIntent(identity: string, notice: Notice): Promise<IntentResult>;
}

function isNotice(value: unknown): value is Notice {
  if (typeof value !== "object" || value === null) return false;
  const candidate = value as Record<string, unknown>;
  return ["noticeRevision", "recipientRef", "destination", "body"]
    .every((field) => typeof candidate[field] === "string"
      && (candidate[field] as string).trim().length > 0);
}

export function noticeHandler(store: IntentStore) {
  return async (req: Request, res: Response): Promise<void> => {
    const input: unknown = req.body;
    if (!isNotice(input)) {
      res.status(400).json({ error: "Invalid notice" });
      return;
    }

    const identity = JSON.stringify([
      input.noticeRevision, input.recipientRef, "sms",
    ]);

    try {
      const result = await store.ensureIntent(identity, input);
      if (result.kind === "conflict") {
        res.status(409).json({ error: "Notice identity has changed content" });
        return;
      }
      res.status(202).json({
        operationId: result.operationId,
        evidence: "intent_recorded",
      });
    } catch {
      res.status(503).json({ error: "Intent could not be confirmed" });
    }
  };
}
```

`noticeRevision` must be unique within the authorized workspace scope. In a multi-tenant implementation, include the authenticated tenant identity in the durable key; don't trust a tenant identifier supplied in the body. Validate destination rules and message limits at the application boundary as well. The small validator above checks only the example's object shape.

The response deliberately reports `intent_recorded`. It says nothing about the worker's progress. If the database commits but the HTTP response disappears, submitting the same immutable intent again returns its existing operation identity. That is the storage behavior to verify before wiring up a sender.

Next comes the worker. Persist an attempt identifier and a stable sender idempotency key before the network call. Save the sender's message identifier when available. After a timeout, record an unresolved outcome and schedule reconciliation. A lookup by message identifier helps only if you obtained that identifier; recovery from a lost initial response requires another supported correlation mechanism, such as lookup by a caller reference or a documented idempotent replay contract.

That missing identifier is the pitfall to test.

## 4. Can status polling prevent duplicate SMS sends after a timeout?

Only when it can answer the unresolved question. Polling is an evidence collection step, and another send is a new side effect. Keep their permissions separate.

Polling has limits.

Define a reconciliation deadline and a bounded polling schedule as application policy. Respect documented rate limits and retry instructions. Persist the next check time so a worker restart does not reset the schedule. If lookup returns no match, interpret that result according to the lookup contract: do not assume it proves that an earlier send never happened. Likewise, do not assume an idempotency key remains valid indefinitely or across changed payloads.

For operations, log the operation ID, attempt ID, observation time, external reference when available, and the reason for each state transition. Track unresolved-operation count, oldest unresolved age, and reconciliation failures. Use operation IDs in logs or traces; keep them out of metric labels to avoid one time series per notice. Alert on the application's evidence deadline, with enough context for an operator to find the restricted record.

Test four boundaries before deployment: a repeated inbound event, a committed intent followed by a lost HTTP response, an accepted send followed by a lost sender response, and a worker restart during reconciliation. The expected assertions concern durable identity and evidence transitions. A mocked timeout alone does not establish that the external sender deduplicates anything; verify that behavior against its documented contract and an appropriate integration environment.

Keep an append-only observation history alongside the current status. Reconciliation can then explain a transition instead of overwriting the only clue about how it happened. Choose retention and access controls for that history deliberately; an application-level append-only policy by itself does not establish tamper resistance.

## 5. Does a delivery record prove compliance?

The evidence has limits. Define the required notice content, recipient mapping, timestamps, and acceptable delivery evidence with the people responsible for the compliance obligation. A sender's acceptance, a delivery report, and a human acknowledgement answer different questions. Let the record name the specific evidence it holds rather than presenting a single green “compliant” badge.

Email standards also have boundaries. RFC 8058 describes one-click unsubscribe signaling for email; it does not define SMS delivery evidence or establish that a compliance notice reached its recipient. Use it only for the email behavior it actually specifies. Moving the same notice to another channel needs its own evidence model and an explicit decision about whether another communication is permitted.

The practical decision rule is straightforward: choose the send and reconciliation contracts that let your team explain an unresolved operation without guessing. When the available contract cannot settle whether a timed-out request took effect, preserve that uncertainty and route the decision to a defined review process. Automated repeat sends require a justified duplicate-control boundary.

An audit trail should make that decision inspectable.

## References

- RFC 8058, Signaling One-Click Functionality for List Email Headers. Scope: email unsubscribe behavior, not SMS delivery confirmation. https://datatracker.ietf.org/doc/html/rfc8058
