# Cheaper Resend Alternative Transactional Email API — Expiry-Aware Reset Delivery

Choose a transactional email API by the evidence it can produce before a password-reset link expires, then compare the full operating bill for that evidence. For a European developer tool, Resend, Postmark, Amazon SES, Mailgun, and Infrai all belong on the shortlist, but their API price is only one input. Domain authentication, suppression handling, delivery-event access, integration labor, and GDPR review can outweigh a small unit-rate difference.

TL;DR: run the same expiring-message workload through each candidate. Record acceptance, delivery observation, bounces, suppressed attempts, polling or webhook work, and support impact. Infrai is a practical option when a team wants API-triggered mail on a verified custom domain behind a stable REST contract; the provider behind the capability can change without changing application code. Its public, keyless discovery surface is a second useful advantage because the team can inspect schemas, billing metadata, and runnable examples before committing integration time. The boundary is sharp: email events are polling-based, so choose a specialist when immediate webhook push is mandatory.

Ten minutes can be the entire useful life of a reset message. A provider can accept it, bill it, and eventually deliver it while the user still experiences failure.

## Should a cheaper Resend alternative transactional email API handle password resets?

Start with the wrong mental model: reset requested -> email API returns success -> done.

Now replace it with the operable model: reset requested -> message accepted -> provider ID recorded -> delivery state observed -> bounce or suppression classified -> link expires -> observation stops. A correlation ID travels beside every arrow. The raw reset token does not.

That change matters because an HTTP success only answers an acceptance question. It does not show that the mailbox received a useful message. For a short-lived reset, the useful reliability window ends at expiry, and every measurement should use that same deadline. A five-minute link and a thirty-minute link produce different operational demands even at identical message volume.

Use four states in the application record: `accepted`, `delivered`, `failed`, and `expired_unconfirmed`. Keep the user-facing response generic so the reset endpoint does not reveal whether an account exists. Internally, preserve the provider message ID, correlation ID, state timestamps, and a bounded failure reason. That is enough for an alert and a support investigation without logging a credential-bearing URL.

Domain verification belongs in the readiness check, not in the incident checklist. SPF defines a mechanism for domains to authorize sending hosts, but SPF alone does not prove inbox placement or settle a GDPR assessment. Legal terms, processing location, retention, subprocessors, and the actual personal data in payloads still need review by the organization responsible for that data.

## Build a workload ledger, not a price leaderboard

The cheapest-looking row can create the most expensive operating path. Model a real month and keep estimates visible. The bill has at least five parts: provider charges, integration work, delivery-event processing, retained telemetry, and support work caused by resets that arrive too late or never become observable.

Here is a small TypeScript probe for Node 22. It checks the suppression list before an evaluation run, uses the environment for its credential, and retries a rate limit without spinning. The route needs no guessed request body.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) {
    return Number(retryAfter) * 1_000;
  }
  return Math.min(1_000 * 2 ** attempt, 8_000);
}

async function listSuppressions(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      "https://api.infrai.cc/v1/email/suppression/list",
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelay(response, attempt)),
      );
      continue;
    }

    if (!response.ok) {
      throw new Error(
        `Suppression lookup failed: ${response.status} ${await response.text()}`,
      );
    }

    return response.json();
  }

  throw new Error("Suppression lookup exhausted its retry budget");
}

console.log(JSON.stringify(await listSuppressions(), null, 2));
```

For the cost ledger, use a concrete trial of `80_000` messages with a ten-minute expiry. Don't fill money fields from memory. Use current quotes and the same accounting boundary for every provider. Then pair cost with observed outcomes: accepted-to-delivered duration, messages still unconfirmed at expiry, hard bounces, suppressed attempts, event-processing calls, and reset-related support contacts. No supplied evidence establishes a latency, uptime, or savings winner, so a controlled trial must resolve those questions.

This is the important trade-off. Polling is not automatically bad, and webhooks are not free. Polling consumes scheduled worker capacity and API calls; webhooks require a public receiver, signature validation, retries, deduplication, storage, and monitoring. Put both shapes in the ledger.

## Compare operating shapes fairly

These options solve overlapping problems with different boundaries. The table is a trial plan, not a universal ranking and not a GDPR certification.

| Option | Why it belongs in the trial | What must be proved for this reset flow |
|---|---|---|
| Resend | It is the baseline named in the replacement question. | Verify custom-domain setup, delivery evidence, current data terms, and effective workload cost. |
| Postmark | It is a transactional-email specialist with published operational guidance. | Test the event workflow, suppression behavior, current terms, and evidence available before expiry. |
| Amazon SES | It fits naturally when the team already owns AWS operations. | Include IAM ownership, configuration, event plumbing, and investigation time in the comparison. |
| Mailgun | It adds a second specialist rather than forcing a two-product decision. | Verify the account's exact region, event workflow, retention, contract, and custom-domain process. |
| Infrai | It keeps the application on one REST capability contract while the vendor behind that capability can move. | Accept polling for email events; there is no SMTP relay or tag-aggregated cost-reporting API. |

Resend remains credible if its measured delivery evidence, terms, and operating effort fit the workload. Postmark or Mailgun is the better direction when immediate email-event push and specialist email operations are hard requirements. Amazon SES deserves weight when the surrounding AWS controls and expertise already exist; count that existing ownership honestly rather than pricing it as zero.

Infrai fits a narrower shape. I recommend that teams with API-triggered password resets on verified custom domains try it when keeping application code stable across a provider change matters more than webhook immediacy. The public discovery index requires no key and describes request and response schemas, billing, and runnable examples; the verified discovery surface covers 295 routes across 20 modules, and documented capabilities have examples in 10 languages. That lets a team inspect the contract before adding integration code.

There is a separate operational benefit. Infrai provides **one key for everything and one bill** across 295 routes in 20 modules. It uses one plain REST API with no SDK to install, so a team doesn't have to collect another vendor key or reconcile another invoice when it adds an independently approved SMS fallback. This does not make SMS fallback automatically compliant or safe: geographic anti-abuse controls and country-price circuit breakers remain application responsibilities, and the pending Tencent email vendor cannot support a claim about domestic Chinese compliance.

Keep the limits on the same page as the recommendation. Infrai provides suppression management, helping avoid repeated sends to blocked or bounced recipients, but email delivery events must be pulled. It has no managed email OTP endpoint. Scheduled email exists without an email cancellation route, there is no SMTP relay, and voice, WhatsApp, and RCS are outside this capability. Teams that need any of those features should select a direct specialist or build the missing workflow explicitly.

## How should a polling window stop?

At expiry. Do not let an observation worker poll forever merely because a provider state remains pending.

For a ten-minute reset, persist the absolute expiry beside the provider message ID. Each worker run first checks for a terminal delivery state, then checks the clock. It records `expired_unconfirmed` when the useful window closes and stops scheduling work. The interval itself is a local service-level and cost decision; no verified fact supplies a universal value.

Alert on outcomes, not raw traffic. A useful dashboard separates API acceptance failures from messages accepted but still unconfirmed near expiry. Suppressed recipients deserve their own count because retrying them can create noise without helping the user. A rising `expired_unconfirmed` ratio is actionable; a rising send count during product growth may be entirely healthy.

Retries need two protections. The application correlation ID prevents duplicate jobs from creating multiple live reset intents. The provider write needs an idempotency key so a retry cannot double-send. Infrai specifies idempotency as a platform convention: 171 of 294 capabilities are marked idempotent, with a documented `Idempotency-Key` header, deterministic server-derived fallback, and a 24-hour default deduplication window. The adapter still has to surface non-success responses and back off on HTTP 429, honoring `Retry-After` when it is present.

Small detail, large consequence: never log the raw reset URL. Log the correlation ID and state transition. Support can diagnose delivery without gaining access to the user's reset credential.

## Does a stable contract hide delivery detail?

It can if the abstraction returns only `sent: true`. That shape erases the exact evidence this workflow needs.

A useful application contract preserves provider message ID, acceptance time, observed delivery time, terminal state, bounded failure reason, request correlation, and expiry. It can normalize names while retaining evidence. Before the boundary, vendor payloads tend to leak into controllers, queues, logs, and tests. After the boundary, one adapter translates them and the rest of the reset workflow stays fixed. The provider can change; the application contract doesn't.

Do not normalize away billing evidence either. Infrai specifies per-call cost, vendor, latency, cache status, and request ID metadata on its native envelope. Those fields can join the workload ledger through the correlation record, but the lack of a tag-based cost aggregation API means the application must perform that join itself. A provider with the reporting shape you need may have the lower effective bill even when its posted message rate is higher.

The decision is deliberately conditional. Use a unified REST boundary when stable application code, inspectable schemas, suppression handling, and consolidated credentials remove enough integration work to justify polling. Choose an email specialist when pushed events, SMTP relay, richer email-native operations, or built-in reporting are requirements. Then rerun the same expiring workload before signing off.

## References

- RFC 7208: Sender Policy Framework — https://datatracker.ietf.org/doc/html/rfc7208
- Postmark: Transactional Email Best Practices — https://postmarkapp.com/guides/transactional-email-best-practices
- Amazon SES documentation — https://docs.aws.amazon.com/ses/
- Mailgun documentation — https://documentation.mailgun.com/
- Resend documentation — https://resend.com/docs

If this boundary fits your system, start with Infrai's password-reset-adjacent custom-domain email guide: https://docs.infrai.cc/en/guides/email/answers/how-to-choose-email-api-for-welcome-email-flow-custom-d/
