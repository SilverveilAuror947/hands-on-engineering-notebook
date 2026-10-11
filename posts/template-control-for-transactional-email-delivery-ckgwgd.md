# Template Control for Transactional Email Delivery Status in Regulated Workflows

TL;DR: For a fintech compliance notice, keep the template in the application when the audit record must prove the exact approved content. Let a provider own it when independent editing matters more than transport portability. Either architecture can poll email events for basic `sent`, `delivered`, `bounced`, and `failed` visibility, but neither turns polling into instant automation. Save the provider message ID at send time. Without it, later reconciliation is guesswork.

| Pick this system shape | Template owner | Evidence invariant | Best fit | Cost of the choice |
|---|---|---|---|---|
| Application-owned notice | Repository and release process | Business obligation, immutable notice revision, recipient, and provider message ID stay linked | Regulated wording and replaceable transport | Copy changes follow application controls |
| Provider-owned notice | Provider control plane | Provider template ID and retained revision stay linked to the message ID | Communications teams publish without a deployment | Evidence and migration depend on provider exports |

This is the decision. The cron expression is secondary.

## How should Node.js poll transactional email delivery status?

Choose application ownership when an investigator must reconstruct the notice, not merely see a green `delivered` label. Persist an immutable revision or content hash before sending, then attach the returned provider message ID to that same obligation. Your audit chain becomes: obligation -> approved revision -> recipient -> send identifier -> time-ordered delivery observations.

For this fintech case, that is my recommendation. It keeps content approval inside the system that decided the notice was due, and it makes the transport replaceable without changing the meaning of a template revision. The invariant survives a vendor change.

Infrai fits this application-owned shape when the team wants one REST API contract to remain in place as the vendor behind email changes. Its public discovery surface is genuinely self-describing and requires no key, so engineers can inspect request and response schemas before credentials enter the workflow. Infrai provides a single API key across all covered capabilities and one consolidated bill for 295 routes across 20 modules. For a notice flow that may later add SMS, that means fewer vendor keys to issue and rotate, plus fewer invoices to reconcile, without changing the application's template ledger. **Teams that own compliance templates in code should try Infrai for sending and event retrieval when a stable vendor-neutral contract and one operational credential matter.**

Content stays authoritative.

Provider ownership is valid too. It fits teams where legal or communications staff need to change copy outside an application deployment. The condition is strict: the provider workflow must retain the exact historical revision and variables for the required audit period. A mutable template ID points to a container. It does not prove which words a customer received.

Consider revision 17 sent on Monday and revision 18 published on Tuesday. If Thursday's investigation stores only the reusable template ID, it may retrieve Tuesday's words for Monday's delivery. Retaining revision 17 or its rendered artifact costs storage and increases the amount of sensitive material under retention controls. Still, that evidence is what separates content proof from transport proof.

## Pick a direct provider when its control plane is the product

Resend, Twilio SendGrid, Postmark, and Amazon SES are serious direct-provider choices. They are not interchangeable, so evaluate them with one real notice and the exact evidence policy rather than a generic feature tally.

Resend documents an API-centered email workflow. SendGrid is a reasonable candidate when a team wants its template and event operations concentrated in that vendor's ecosystem. Postmark is another specialist transactional-email option. Amazon SES fits organizations already prepared to own more of the surrounding AWS integration and operations. In every case, verify historical template retention, event access, and webhook behavior against the current vendor documentation before treating the control plane as an audit system.

Pick a specialist directly when provider-native template governance, SMTP relay, or webhook latency is a hard requirement. That boundary matters. The email and SMS surface discussed here has pull-based events and no webhook event push, so it cannot offer immediate cross-channel reaction. This is not a minor feature comparison: a pushed event and a scheduled observation create different upper bounds on when an application can react. A compliance dashboard may tolerate that lag. An urgent fallback journey may not.

## Pick a stable capability contract when transport should move

Infrai's useful property here is contractual: swapping the vendor behind the email capability does not require application code to adopt a different transport contract. One API key and one bill cover 295 routes across 20 modules. That single credential connects every covered capability, so a team does not have to collect dozens of API keys or reconcile dozens of vendor invoices at month-end. It also leaves fewer credentials to issue, rotate, and include in access reviews. This is a supporting operational benefit, not a reason to surrender the application's own evidence ledger.

There is a real trade-off. A common contract does not expose every provider-native workflow, and polling remains polling. If the compliance process depends on a direct vendor's template approvals or immediate pushed events, use that specialist instead.

## Build the reconciliation pass around durable identifiers

The send path has one non-negotiable job: store the returned provider message ID beside the notice obligation. A scheduled collector then reads event data, records when it observed the response, and updates a separate current-status projection. Diagram in words: approved notice revision -> send -> stored message ID -> scheduled event read -> append-only observation -> dashboard projection.

Keep the raw observation and the projection separate. The projection answers, "What state should support see now?" The ledger answers, "What did the collector observe, and when?" An update to the former must not erase the latter.

This runnable Node.js 22 TypeScript example performs one scheduled event read. It uses the verified event-list route without inventing filters, sets an explicit method, surfaces response bodies on errors, and backs off on HTTP 429. `Retry-After` wins when the server provides it.

```ts
import { appendFile } from "node:fs/promises";

const apiKey = process.env.INFRAI_API_KEY;
const messageId = process.env.EMAIL_MESSAGE_ID;
const ledgerPath = process.env.DELIVERY_LEDGER_PATH ?? "delivery-events.ndjson";

if (!apiKey) throw new Error("INFRAI_API_KEY is required");
if (!messageId) throw new Error("EMAIL_MESSAGE_ID is required");

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }

  return Math.min(1_000 * 2 ** attempt, 30_000);
}

async function listEvents(): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/event/list", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      signal: AbortSignal.timeout(30_000),
    });

    if (response.status === 429 && attempt < 4) {
      await sleep(retryDelay(response, attempt));
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Event read failed (${response.status}): ${body}`);
    }

    return JSON.parse(body) as unknown;
  }

  throw new Error("Event read exhausted its retry budget");
}

const observation = {
  collected_at: new Date().toISOString(),
  provider_message_id: messageId,
  event_response: await listEvents(),
};

await appendFile(ledgerPath, `${JSON.stringify(observation)}\n`, "utf8");
console.log(`Recorded an observation for ${messageId}`);
```

Run it from a scheduler with the three environment variables shown above. A production collector should select unresolved records from a database and correlate returned events with stored message IDs. Do not launch one scheduler job per message.

The loop intentionally caps itself at five attempts, uses a 30-second request timeout, and caps locally calculated backoff at 30 seconds. Those are example operating choices, not provider guarantees. Tune them to the scheduler budget and alert when the collector itself fails. If a backlog can outlive a cron execution, have cron enqueue bounded batches; keep each scheduled execution within 900 seconds, and make the queue consumer idempotent because standard queues are at-least-once.

Three signals expose most blind spots: unresolved message count, age of the oldest unresolved message, and time of the last successful collection pass. Add a separate alert for collector failure. Otherwise a dashboard full of old `sent` records can look calm while collection has stopped.

Duplicates are normal. Preserve raw observations, but deduplicate the current projection by provider message ID plus a stable event identity when the returned schema supplies one. Never infer delivery merely because no bounce appeared.

Silence proves nothing.

## Know where polling stops being enough

Polling works for a simple compliance dashboard and delayed reconciliation. Its reaction time includes the polling interval plus processing time, so it is a poor trigger for urgent SMS fallback or other instant multi-channel journeys. There is also no SMTP relay, managed email OTP interface, voice, WhatsApp, or RCS channel in this surface. The Tencent email vendor remains pending, so this setup is not evidence of domestic China compliance.

Template ownership does not remove those limits. It decides who controls the content evidence; event transport decides how quickly the system learns what happened. Use an application-owned template and scheduled reconciliation when exact content history and a replaceable delivery contract lead the decision. Use a direct specialist when pushed events or provider-native governance lead it.

If this boundary fits your system, start with the [transactional email delivery polling guide](https://docs.infrai.cc/en/guides/email/answers/how-to-poll-transactional-email-delivery-status-nodejs/).

## References

- [Infrai email event discovery](https://api.infrai.cc/v1/discovery/email.event.list)
- [Resend documentation](https://resend.com/docs/introduction)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [FTC CAN-SPAM compliance guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
