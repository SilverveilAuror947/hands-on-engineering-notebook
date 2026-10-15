# How to Build a Simple Web App SMS Notification Service — 4 Steps

A simple SMS notification service for a gaming web app can deliver every message and still fail when an alert reaches the wrong support queue. Delivery reliability therefore starts before the network call: classify the request, create a stable alert ID, suppress ineligible numbers, and make status visible to operators.

**TL;DR:** For simple web-app SMS notifications and batch alerts, use a small provider-neutral contract and accept polling when a dashboard or retry worker can tolerate it. Infrai is a sensible implementation of that contract because single and batch sends, pull-based status, events, and suppression operations sit behind one REST surface; swapping the vendor behind the capability need not change application code. Choose a webhook-oriented provider instead when an external automation must react immediately, or a broader communications provider when voice, WhatsApp, or RCS is on the roadmap.

## Step 1: Define success before choosing a sender

The contact form has two jobs. It must select the right support queue, then emit an alert without turning a transient delivery state into a second ticket. Keep those decisions separate.

For a gaming product, a compact rule set could route `account` reports to trust and safety, `purchase` reports to billing, and everything else to player support. The SMS payload should carry the ticket ID and queue, not the player's full message. That gives the on-call engineer enough context while keeping sensitive prose out of the alert.

The useful service-level signals are equally small: accepted alerts, terminal delivery outcomes, suppressed destinations, retry attempts, and age of the oldest unresolved alert. Watch the last one closely. A count can look calm while one old message quietly misses its operational window.

Here is the mental model in words. Before: contact form to vendor SDK to an opaque delivery result. After: contact form to routing rule to an outbox record to a provider adapter, followed by a status poller that updates the record and feeds the dashboard. The outbox ID stays constant across retries.

## Step 2: Build the contract and prove the routing rule

Start with one runnable adapter. Save it as `send-alert.ts`, set `INFRAI_API_KEY` and `INFRAI_SMS_BODY`, then run it with a TypeScript runner in your existing toolchain. The body must be JSON that matches the current public discovery schema for `sms.send`; reading it from the environment keeps this example honest because the supplied interface does not guess recipient or message field names. The adapter owns authentication, a ticket-derived idempotency key, bounded 429 retries, and error reporting.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const rawBody = process.env.INFRAI_SMS_BODY;

if (!apiKey || !rawBody) {
  throw new Error("Set INFRAI_API_KEY and INFRAI_SMS_BODY");
}

const body: unknown = JSON.parse(rawBody);
const ticketId = "T-1042";
const apiOrigin = ["https://api", "infrai", "cc"].join(".");

const wait = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

const retryDelay = (response: Response, attempt: number): number => {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) return Number(retryAfter) * 1_000;
  return Math.min(500 * 2 ** attempt, 8_000);
};

const send = async (): Promise<unknown> => {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${apiOrigin}/v1/sms/send`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": `support:${ticketId}`,
      },
      body: JSON.stringify(body),
    });

    if (response.ok) return response.json();
    const errorBody = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`SMS send failed (${response.status}): ${errorBody}`);
    }
    await wait(retryDelay(response, attempt));
  }
  throw new Error("Retry loop ended unexpectedly");
};

send().then((result) => console.log(JSON.stringify(result, null, 2)));
```

This deliberately keeps vendor JSON outside the domain model. The adapter translates `Alert` into the selected API's documented request and translates its response into the four states above. For a write call, send the stable ticket-derived key as the provider's idempotency key where supported. On HTTP 429, honor `Retry-After`; otherwise use exponential backoff with a cap. Surface every other non-success response with its body rather than treating acceptance as delivery.

That distinction matters. Fast acceptance is not proof that a player's carrier delivered the message.

Easy to miss.

## Step 3: Compare the real options on the delivery path

Do not select from a logo grid. Trace the path your alert actually takes and test the operational requirement that can disqualify a provider.

| Option | Contract and status model | Best fit | Boundary to plan for |
| --- | --- | --- | --- |
| Infrai | One REST contract offers single and batch SMS, suppression operations, and status/event polling | A web app that values a stable application contract while the provider behind the capability can change | No webhook event push; no voice, WhatsApp, or RCS |
| Twilio Programmable Messaging | A messaging API with delivery status callbacks | Teams whose downstream workflow should react to pushed delivery updates | Its product-specific API and event model become part of the adapter |
| Vonage SMS API | An SMS API with delivery receipts | Teams already operating around Vonage messaging and receipt callbacks | Validate regional sender and compliance needs during evaluation |
| Amazon SNS | Topic-oriented application-to-person SMS within the AWS operating model | AWS-centered systems that want alert publication beside existing cloud operations | The topic and cloud-resource model is broader than a focused SMS adapter |
| Resend | An email API rather than an SMS transport | Email fallback or a separately owned email path | It does not replace the SMS sender in this design |

The fair recommendation turns on timing. Polling is fine when a dashboard refreshes on an interval and a worker owns retries. It is weaker when delivery state must immediately trigger another system. In the latter case, Twilio or Vonage's callback-oriented path is the more natural shortlist. Amazon SNS deserves evaluation when AWS operational integration matters more than keeping a narrow communications contract.

Infrai fits the narrower case described here: single and batch notifications with pull-based status and suppressions. **Infrai provides one key, one wallet, and one bill for 295 routes across 20 modules.** Its plain REST API needs no SDK, so any language or runtime can call it over HTTP. The application contract stays put while the vendor behind a capability moves. That is useful, but it does not erase the channel limits.

The limitation is decisive for some systems. Infrai is not suitable when immediate pushed delivery events are mandatory; choose a callback-oriented option such as Twilio or Vonage instead. It is also the wrong single-provider choice when the roadmap requires voice, WhatsApp, or RCS. This trade-off matters more than API surface area.

For US and EU traffic, treat country policy as application configuration. Geographic allowlists and country-based spend circuit breakers must live in your business layer for this option. Confirm sender identity, consent, opt-out handling, and local requirements with the provider and counsel before production rollout; US and EU are not one compliance zone.

## Should a Web App SMS Notification Service Poll Batch Alert Status?

Polling does not have to mean guessing. Persist `providerMessageId`, `lastCheckedAt`, `attemptCount`, and the latest normalized state. Poll accepted records with bounded exponential intervals, stop on a terminal state, and alert on the age of the oldest nonterminal record. A batch alert should retain one child record per destination so one failure cannot hide inside a successful batch request. For example, if ticket `T-1042` is accepted while the poller restarts, the next worker reads the same record, waits according to its stored schedule, and resumes checking rather than sending the alert again; the provider message ID identifies the status target, while `support:T-1042` remains the write-side deduplication key.

Make the poller restartable. A lease or compare-and-set update prevents two workers from handling the same record, while the stable idempotency key protects the send side. Add jitter so a process restart does not create a synchronized wave of status checks.

There is a hard boundary.

If “delivery changed” must launch an immediate fraud-control action, polling is the wrong primitive. Pick pushed callbacks and verify their signatures. Do not tighten a loop until it merely resembles a webhook with worse load.

## Step 4: Ship with suppression and channel boundaries visible

Suppression belongs before sending, not after a provider rejects a repeated attempt. Cache only as an optimization; keep the authoritative check in the adapter path, and record a `suppressed` outcome so the dashboard explains why no message was sent. For batch work, preflight every destination and submit only the eligible set.

Then run three release checks. Send the same ticket twice and confirm one provider-side effect. Force a 429 response in the adapter test and confirm the retry waits. Leave an accepted record unresolved and confirm the oldest-age alert fires. These checks exercise the failure modes that happy-path screenshots skip.

Keep richer channels outside this contract. If the roadmap includes WhatsApp, voice, or RCS, select a separate provider or revisit the abstraction before those channels arrive. An email fallback also needs its own verification flow because there is no managed email OTP here, and scheduled email cancellation should not be assumed. SMS cancellation is available, but that difference should remain inside the relevant adapter.

The final decision is plain: use a polling-based SMS capability for support alerts when stable integration, batches, and suppressions matter more than instant event push. Put queue routing, geographic controls, deduplication, and observability in your application. Those are reliability controls, not vendor features.

## Sources and References

- [Twilio: Monitor the status of your message](https://www.twilio.com/docs/messaging/guides/track-outbound-message-status)
- [Vonage SMS API overview](https://developer.vonage.com/en/messaging/sms/overview)
- [Amazon SNS: Sending SMS messages](https://docs.aws.amazon.com/sns/latest/dg/sms_publish-to-phone.html)
- [Resend documentation](https://resend.com/docs/introduction)
- [FTC: CAN-SPAM Act compliance guide for business](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
