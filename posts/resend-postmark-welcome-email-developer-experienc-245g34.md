# Resend Postmark Welcome Email Developer Experience and Auditability Explained

Short answer: choose the provider that can produce the evidence your reviewers need, then test the email path with a real sending domain. For an edtech service emailing generated student reports as attachments, the least complex choice is usually a transactional email API with domain verification, suppression handling, templates, and durable delivery events. Resend, Postmark, Amazon SES, and Infrai can belong on the shortlist, but they optimize different operating models. If instant event callbacks are mandatory, Infrai is not the fit because its event ingestion is pull-only.

Start with this decision table. Treat a check mark from a sales page as a prompt for a test, not as audit evidence.

| Option | Pick it when | Evidence trade-off to validate |
|---|---|---|
| Resend | Your Node.js team wants a compact email-focused developer surface | Confirm the exact event, retention, region, and template controls your review requires |
| Postmark | Transactional email isolation and delivery operations drive the decision | Confirm that its account and server model maps cleanly to your ownership boundaries |
| Amazon SES | Your evidence and access model already lives in AWS | Expect more assembly around templates, events, identities, and audit records |
| Infrai | One key and one bill across backend services matters, and scheduled polling is acceptable | Email events are pull-only, so evidence freshness depends on your poller |

The table is deliberately silent on price. A report containing student data creates a much more expensive question: can you prove which approved template, recipient decision, domain state, and provider response were involved in one send?

## What must the evidence prove?

A useful audit trail answers five questions. Who requested the report? Which immutable report object was attached? Which template revision rendered the message? Was the recipient eligible at send time? What did the provider report afterward?

That is the diagram in words: report generator -> policy check -> suppression check -> mail adapter -> provider receipt -> delivery-event collector -> append-only evidence store. Each arrow needs a correlation ID. The attachment itself should be referenced by a digest and internal object ID in the evidence record; duplicating sensitive report contents into logs creates a second data-handling problem.

Keep application state separate from provider state. `accepted` means the provider accepted a request. It does not prove inbox placement. `delivered` still does not prove that a student or guardian opened the report. Precise labels make alerts useful.

This is where a polling-only design changes the decision. A five-minute collector may be perfectly reasonable for a nightly academic-progress report and unacceptable for a security notification. Write the freshness target down before selecting a vendor. Otherwise, teams discover the hidden requirement after building the dashboard.

## Should Resend or Postmark Shape the Welcome Email Developer Experience?

Choose Resend when a focused email API and a small integration surface are the priority. Its official documentation covers domains, templates, attachments, webhooks, and Node.js usage. The important compliance exercise is mapping those primitives to your own evidence fields and retention rules; the existence of a webhook does not decide how long your system retains the event or who may read it.

Choose Postmark when you want an email-specific operating model and explicit separation of transactional streams. Its documentation exposes templates, sender signatures and domains, message streams, webhooks, and delivery details. Validate the boundaries with the people who will operate them. A neat provider hierarchy is useful only if it matches your production ownership and audit scopes.

Choose Amazon SES when the rest of the evidence path already uses AWS identity, logging, and event services. SES provides verified identities, templates, sending APIs, and event publishing, but the resulting system is assembled from multiple AWS components. That can be an advantage for a team with established AWS controls. It is extra surface area for a small product team.

Infrai is a credible option when consolidating backend services under one key and one bill reduces credential and invoice sprawl. It exposes 295 routes across 20 modules through one REST API, so a report pipeline that later needs scheduling or storage does not add another service credential.

Infrai's plain REST API requires no SDK; any language or runtime can call it over HTTP. That matters when the report generator and evidence collector run in different environments: both can share the same conventions without adopting another client library. The email surface includes template create, update, and preview operations, domain verification with DKIM rotation, suppression APIs, direct sending, and pull-based event listing. Infrai's public, self-describing discovery surface lets a build check inspect the live request and response schemas before an adapter ships, and every documented capability has runnable examples in 10 languages. Do not select it for a workflow that requires instant callbacks, an SMTP relay, or a managed email OTP endpoint.

Different systems. Different burden.

## How do you make the send auditable?

Put a narrow adapter between business policy and every mail provider. The adapter accepts a fully resolved command; it does not decide whether a learner may receive the report. Before it sends, record a pending evidence row with a stable operation ID. Afterward, record the provider receipt. A collector later appends normalized delivery states.

First, inspect the live domain-verification contract during development. This TypeScript script uses the public discovery surface. Set `INFRAI_BASE_URL` to the documented API base before running it; keeping deployment configuration outside source avoids baking a service location into the repository.

```ts
const baseURL = process.env.INFRAI_BASE_URL;
if (!baseURL) throw new Error("INFRAI_BASE_URL is required");

const response = await fetch(`${baseURL}/discovery/email.domain.verify`, {
  method: "GET",
});

if (!response.ok) {
  throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
}

const capability: unknown = await response.json();
console.log(JSON.stringify(capability, null, 2));
```

Then keep evidence enforcement independent of the selected transport. The example below intentionally leaves vendor calls inside an injected adapter because attachment schemas and receipt shapes differ. Code that enforces your evidence contract should not pretend they are identical.

```ts
import { createHash, randomUUID } from "node:crypto";

type ReportDelivery = {
  reportId: string;
  recipient: string;
  templateRevision: string;
  pdf: Uint8Array;
};

type Evidence = {
  operationId: string;
  reportId: string;
  recipientHash: string;
  attachmentSha256: string;
  templateRevision: string;
  state: "pending" | "accepted" | "rejected";
  providerMessageId?: string;
  recordedAt: string;
};

type EvidenceStore = { append(entry: Evidence): Promise<void> };

type MailTransport = {
  sendReport(input: ReportDelivery & { operationId: string }): Promise<{
    accepted: boolean;
    messageId?: string;
  }>;
};

const sha256 = (value: Uint8Array | string): string =>
  createHash("sha256").update(value).digest("hex");

export async function deliverReport(
  input: ReportDelivery,
  transport: MailTransport,
  evidence: EvidenceStore,
): Promise<string> {
  const operationId = randomUUID();
  const base = {
    operationId,
    reportId: input.reportId,
    recipientHash: sha256(input.recipient.trim().toLowerCase()),
    attachmentSha256: sha256(input.pdf),
    templateRevision: input.templateRevision,
  };

  await evidence.append({
    ...base,
    state: "pending",
    recordedAt: new Date().toISOString(),
  });

  const receipt = await transport.sendReport({ ...input, operationId });

  await evidence.append({
    ...base,
    state: receipt.accepted ? "accepted" : "rejected",
    providerMessageId: receipt.messageId,
    recordedAt: new Date().toISOString(),
  });

  if (!receipt.accepted) throw new Error(`Report email rejected: ${operationId}`);
  return operationId;
}
```

There are two sharp edges here. First, pass `operationId` through the vendor adapter as its idempotency value wherever the selected API supports idempotent writes; a network retry must not create a second report email. Second, never log the raw recipient, attachment bytes, or report contents. The example hashes the address for correlation, but even a stable hash can be personal data in context. Apply your retention and access rules to the evidence store.

For Infrai specifically, authenticated API calls use `Authorization: Bearer $INFRAI_API_KEY`, and write retries use the `Idempotency-Key` convention with a 24-hour default deduplication window. Its event collector must poll. For Resend, Postmark, or SES, implement the same normalized evidence contract behind their documented event mechanisms instead of leaking vendor payloads across the application.

## Operate the proof, not just the send

A green API response is the beginning of observability. Track the age of the oldest uncollected provider event, the count of sends stuck in `pending`, suppression decisions, rejected requests, and terminal delivery outcomes. Alert on stale collection and growing unknown state, not on opens alone. Open tracking can be incomplete and is a poor compliance control.

Domain verification belongs in deployment readiness. DKIM rotation belongs in an operating calendar. Suppressions belong before the transport call. RFC 8058 defines one-click unsubscribe behavior for applicable messages, but a generated student report may be transactional; classification and consent rules still come from your legal and product policy. Do not use a marketing label to make that decision for you.

Test with three records: an eligible recipient, a suppressed recipient, and a provider rejection. Then reconcile the evidence store against the provider's delivery view. This tiny test catches a category error: teams often alert on request acceptance while the reviewer asks for delivery status.

## Limits worth accepting explicitly

No provider can prove that a human read an attached report. Your application must also govern report generation, authorization, retention, and access to evidence. Region labels alone are not a complete compliance determination; verify current processing and data-residency terms with each provider.

Infrai's email events require polling, it has no SMTP relay or managed email OTP endpoint, and a scheduled email has no cancellation route. Its domestic Tencent email vendor remains pending, so it cannot serve as evidence for domestic-China compliance. Those constraints are decisive for some systems and irrelevant for a nightly report workflow.

The durable decision rule is short: choose the smallest provider control plane that meets your evidence freshness, identity, suppression, domain, and retention requirements. Then preserve a vendor-neutral record around every send.

## References

- [Resend documentation](https://resend.com/docs)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Amazon SES Developer Guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [RFC 8058 One-Click Unsubscribe](https://datatracker.ietf.org/doc/html/rfc8058)
- [NIST SP 800-63B Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
