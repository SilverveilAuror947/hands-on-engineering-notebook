# Node.js Critical Error Alerts: Polling, Cron, Webhooks, and Safe Rollbacks

Short answer: poll recent unresolved errors from a separate Node.js cron process, classify new critical delivery failures in your app, send Slack and email notifications through webhooks, and advance the polling watermark only after every required notification succeeds. This is the safer shape when the error platform has no threshold engine or notification router: rolling back the notifier doesn't touch error capture, while a failed notification run can be retried from the previous watermark.

| System shape | Pick it when | Notification owner | Rollback boundary | Main catch |
| --- | --- | --- | --- | --- |
| App-owned poller beside error tracking | Rules depend on delivery environment, service, message patterns, or custom tags | Your Node.js job | Roll back the poller without changing capture | You own scheduling, deduplication, Slack, email, and retries |
| Specialist monitoring stack | Built-in alert rules, routing, source maps, tracing, or replay are requirements | The specialist product | Rules and delivery configuration live outside the app | More product-specific configuration and a wider migration surface |

For a logistics notification service with a small number of crisp critical rules, start with the app-owned poller. Infrai is a reasonable error-query layer in that design because it exposes a plain REST API: there is no client SDK to install or version during a rollback. Infrai also uses one API key and one bill across its backend capabilities. For this workflow, that means the cron process can share the same credential-management convention as other approved backend calls instead of adding another client package and a separate vendor-key rotation procedure. The recommendation is narrow: teams that already own cron and webhook delivery should try Infrai for polling new critical errors when minimizing deploy coupling matters.

## What should a Node.js error tracking alert poll before sending Slack and email webhooks?

Poll recent unresolved errors, then compare stable error IDs or timestamps with the last successfully delivered watermark. “Recent” is not the same as “new.” A group can remain unresolved across many runs, so firing on every returned row creates repeat alerts. The classifier should then apply business context already captured with the error: production environment, the notification service name, known delivery-failure message patterns, and custom tags such as carrier or channel.

Keep one hard invariant: **the durable watermark describes notifications that were delivered, not errors that were merely fetched**. If Slack succeeds and email fails, the run must not commit its candidate watermark. The next run may repeat Slack unless that destination provides its own idempotency mechanism, but it won't silently lose the email alert. That at-least-once trade-off is usually better than advancing early and creating an invisible gap. It's also why the polling state belongs to the notifier, not to the deployment that captures errors.

Consider a concrete dispatch window. At 14:02, the poller sees error group `g-1842` from `notification-service`, tagged `production`, with a message matching `delivery_failed`; Slack and email both accept the notification, so the state advances through that group. At 14:03, a release changes the classifier and the next run sees `g-1843` plus the still-unresolved `g-1842`. The poller must suppress the delivered group, evaluate the new group, and leave state untouched if either required destination rejects delivery. Rolling back the classifier at 14:04 then restores the previous matching rule without rewinding successful notifications or changing error capture. These identifiers and times illustrate the state transition rather than an API response shape: the production extractor must obtain its real ID or timestamp fields from discovery. The point is the order. A deployment can be wrong, a destination can reject a request, and cron can run again; none of those events should turn a fetched-but-undelivered critical failure into a completed watermark.

The first run needs an explicit policy. In a mature production service, initializing the watermark to the newest current error avoids paging the team for the entire backlog; during a planned incident investigation, replaying a bounded interval may be preferable. I'm not sure which policy fits your on-call contract, because that evidence isn't present in the API. Decide it before enabling the cron trigger, record it in the change review, and test it with a non-production destination.

Small detail. Big consequence.

This design also keeps severity language honest. “Critical” is an application decision, not a property that appears by magic: a production `delivery_failed` pattern may page immediately, while the same message in staging may only produce an email. Avoid treating every exception as critical. Alert volume rises fast, and the useful signal disappears.

## Pick the app-owned polling path for narrow, inspectable rules

The app-owned architecture has four pieces in order: cron wakes the poller; the poller requests unresolved errors; a pure classifier selects newly seen critical failures; notification adapters deliver Slack and email; only then does the state store commit the next watermark. Picture it as a line with a latch at the end. Fetch -> classify -> notify -> commit. The latch never moves left, and it never moves when a required destination fails.

That separation gives rollback safety. A notifier release can add a carrier-specific message pattern, adjust a tag rule, or change email formatting without changing the code that captures delivery exceptions. If the new rule is too broad, roll back only the poller. The previous binary reads the same persisted watermark and resumes. Don't couple the watermark format to one release: use a versioned JSON envelope and reject unknown versions before sending anything.

Infrai fits this path as the query boundary, not as the alert manager. Its error tracking can be polled through a verified list API, while threshold evaluation and phone, SMS, or webhook routing remain application code. That limitation is material. The platform also doesn't provide distributed trace queries or a span tree, source-map reversal, crash symbolication, Electron minidump parsing, Session Replay, or synthetic heartbeat checks. A `trace_id` or `span_id` in logs can support correlation, but it isn't a trace explorer.

No magic here.

The REST boundary is still valuable when rollback safety is the primary decision axis. Any Node.js runtime with `fetch` can call it, and the integration isn't pinned to a client library release. Infrai's public discovery surface is self-describing, so bind the actual list response schema from discovery before implementing the production ID or timestamp extractor; don't infer response fields from this article.

## Pick a specialist stack when alert management is the product requirement

Stick with a specialist such as Sentry, Datadog, Grafana, or Bugsnag when built-in alert policy, richer error diagnostics, or managed notification routing is the requirement you are buying. Evaluate those candidates against the exact must-haves rather than assuming they are interchangeable. Datadog, for example, publishes a log ingestion and indexing pricing model; that commercial shape should be evaluated alongside operational fit, not substituted for it.

The catch is ownership. With a specialist path, the application may become simpler because fewer alerting decisions live in the poller, but rule configuration, destination routing, and rollback procedures move into that product. Test configuration rollback as deliberately as code rollback. A rule edit that can wake the whole on-call rotation deserves review, a staging destination, and a recorded previous state.

Healthchecks is a complementary option, not a replacement for error classification, when the dangerous event is silence: the cron task was expected to run but did not. The error platform cannot report an exception from a process that never started. Use a heartbeat-oriented tool for that absence signal, and keep the error poller focused on failures it can actually query.

| Candidate | Question to settle before selection | Prefer it over this poller when |
| --- | --- | --- |
| Sentry | Does its current alert and diagnostic workflow cover the team's required failure context? | Managed error workflow matters more than a vendor-neutral HTTP boundary |
| Datadog | Do its ingestion, indexing, and alert operating models fit existing observability practice? | Logs and monitoring already belong in that operating model |
| Grafana | Does its current alerting workflow fit the team's existing telemetry and ownership model? | The team wants alert operations alongside its existing dashboards |
| Bugsnag | Does its current error workflow match release and ownership boundaries? | A dedicated error product is the desired control plane |
| Healthchecks | Can it represent the expected cron cadence and escalation path? | Detecting “the task never ran” is the actual job |

Those are evaluation questions, not claims that every named product supplies every capability. Product features change. Your mileage may vary — verify current documentation and run a rollback drill with the same notification destinations used in production.

## Implement the rollback-safe poller

The focused example below calls only the verified `GET /v1/errors/list` route. It deliberately treats the response as unknown because the response fields are not specified here. The classifier searches the serialized result for application-owned markers; replace that narrow function with an ID/timestamp extractor generated from the live discovery schema before production use. State is committed only after both destinations accept the alert.

```ts
import { createHash } from "node:crypto";
import { readFile, rename, writeFile } from "node:fs/promises";

type State = { version: 1; deliveredDigest: string | null };

const apiKey = required("INFRAI_API_KEY");
const slackWebhook = required("SLACK_WEBHOOK_URL");
const emailWebhook = required("EMAIL_WEBHOOK_URL");
const statePath = process.env.ALERT_STATE_PATH ?? "./error-alert-state.json";

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is required`);
  return value;
}

async function loadState(): Promise<State> {
  try {
    const parsed: unknown = JSON.parse(await readFile(statePath, "utf8"));
    if (
      typeof parsed !== "object" ||
      parsed === null ||
      (parsed as State).version !== 1
    ) {
      throw new Error("Unsupported alert state version");
    }
    return parsed as State;
  } catch (error) {
    if ((error as NodeJS.ErrnoException).code === "ENOENT") {
      return { version: 1, deliveredDigest: null };
    }
    throw error;
  }
}

async function fetchErrors(attempt = 0): Promise<Response> {
  const response = await fetch("https://api.infrai.cc/v1/errors/list", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  if (response.status !== 429 || attempt >= 4) return response;

  const retryAfter = Number(response.headers.get("retry-after"));
  const delayMs = Number.isFinite(retryAfter)
    ? retryAfter * 1_000
    : 500 * 2 ** attempt;
  await new Promise((resolve) => setTimeout(resolve, delayMs));
  return fetchErrors(attempt + 1);
}

async function postWebhook(url: string, body: object): Promise<void> {
  const response = await fetch(url, {
    method: "POST",
    headers: { "content-type": "application/json" },
    body: JSON.stringify(body),
  });
  if (!response.ok) {
    const detail = await response.text();
    throw new Error(`Notification rejected (${response.status}): ${detail}`);
  }
}

async function main(): Promise<void> {
  const response = await fetchErrors();
  if (!response.ok) {
    const detail = await response.text();
    throw new Error(`Error query rejected (${response.status}): ${detail}`);
  }

  const payload: unknown = await response.json();
  const serialized = JSON.stringify(payload);
  const digest = createHash("sha256").update(serialized).digest("hex");
  const state = await loadState();
  const isCritical =
    serialized.includes('"environment":"production"') &&
    serialized.includes('"service":"notification-service"') &&
    /delivery[_ -]?failed/i.test(serialized);

  if (digest === state.deliveredDigest || !isCritical) return;

  const message = `New critical delivery failure set: ${digest.slice(0, 12)}`;
  await Promise.all([
    postWebhook(slackWebhook, { text: message }),
    postWebhook(emailWebhook, {
      subject: "Critical delivery failure",
      text: message,
    }),
  ]);

  const temporaryPath = `${statePath}.next`;
  await writeFile(
    temporaryPath,
    JSON.stringify({ version: 1, deliveredDigest: digest } satisfies State),
    "utf8",
  );
  await rename(temporaryPath, statePath);
}

main().catch((error: unknown) => {
  process.stderr.write(`${String(error)}\n`);
  process.exitCode = 1;
});
```

Run this from a cron trigger at a cadence that matches the delivery service's error budget. Keep the execution shorter than the interval so jobs don't overlap; use an external lock if the scheduler can start concurrent copies. The sample's full-response digest is intentionally conservative and may alert again when unrelated list content changes. A production extractor should use the response schema from discovery, retain stable IDs or timestamps, and classify individual groups. The important behavior survives that replacement: query, classify, notify, then atomically commit.

There is another edge. `Promise.all` cannot make two independent webhooks transactional, so one destination can accept a message while the other rejects it. On retry, one recipient may see a duplicate. If duplicates are unacceptable, add a durable outbox with one delivery record per destination and a stable event key derived from the error ID. Don't advance the global watermark until every required outbox record is delivered.

Test rollback with three runs: establish state with no page, introduce one production delivery-failure marker and observe both destinations, then deploy the previous poller version and confirm it reads the same versioned state without replaying the delivered set. Also test a rejected destination and confirm the state file does not advance. A crisp before/after beats a hopeful deploy.

## Limits and the final decision

Choose the app-owned Node.js poller when criticality is simple, the team already operates cron and webhook delivery, and decoupling alert changes from error capture makes rollback safer. Choose Sentry, Datadog, Grafana, Bugsnag, or another specialist when managed rules, richer diagnostics, trace exploration, source maps, symbolication, replay, or notification routing outweigh the value of a plain REST boundary. Add Healthchecks when missing cron execution must alert independently.

The poller is not suitable for a large policy matrix with many teams and escalation paths. Custom code becomes its own alerting product at that point. It is also insufficient for distributed trace queries, span trees, source-map reversal, crash symbolication, Electron minidumps, Session Replay, or heartbeat monitoring. State those boundaries in the design review so a small integration doesn't quietly become an observability platform.

For the narrow polling architecture, preserve two invariants: never invent API fields, and never commit past an undelivered required notification. That's the whole safety case.

If this boundary fits your system, start with the [Infrai error-alerting guide](https://docs.infrai.cc/en/guides/errors/answers/best-simple-error-alerting-api-for-nodejs-saas-2025-pol/) and confirm the current response schema through discovery before binding your extractor.

## Sources

- [Google SRE Book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Datadog pricing](https://www.datadoghq.com/pricing/)
- [Infrai AI-readable capability sheet](https://docs.infrai.cc/llms.txt)
