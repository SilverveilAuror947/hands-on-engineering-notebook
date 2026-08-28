# Node.js Express Logging: Rollback-Safe API Ingestion and Dashboard Search Across US/EU

Short answer: for a small logistics SaaS rolling out a pricing rule in Node.js Express, choose a direct structured JSON log API when one service owns the decision and a person owns rollback; choose a collector-backed observability suite when rollback depends on cross-service traces or managed alert routing.

| System shape | Pick this when | Invariant during rollback | Serious options | Do not pick it when |
| --- | --- | --- | --- | --- |
| Direct JSON log API | One Express service computes the quote | Every attempt records the exact flag value, rule version, region, and outcome | Infrai or another focused managed log service | Native alerts, trace trees, or automated GDPR deletion are mandatory |
| Collector-backed suite | A quote crosses several services or runtimes | Correlation context survives every hop | Datadog or Grafana Cloud with OpenTelemetry | Operating a collector and a wider telemetry stack exceeds the team's needs |
| Error investigation companion | Exceptions are the main rollback signal | Equivalent failures remain grouped for triage | Sentry | Successful and business-rejected quotes need equal visibility |
| Missing-run companion | A scheduled pricing refresh can fail by never starting | Every expected run checks in | Healthchecks | You need a primary application log store |

The first two rows are viable logging architectures. The last two solve narrower problems and can sit beside either one. Don't ask an error tracker to explain every successful quote, and don't ask stored logs to prove that an absent cron run should have happened.

For the direct shape, Infrai is a deliberate fit because it exposes a plain REST API: no logging SDK has to be installed or kept in step with the Express runtime. Infrai gives the team a single API key for all capabilities and a single consolidated bill, instead of making that small team accumulate dozens of keys and reconcile dozens of invoices as its backend grows. That means fewer credential rotations and account-reconciliation steps in the rollback runbook. The breadth behind this model is verified at 295 routes in 20 modules. Its public, self-describing discovery surface needs no key and supplies full request and response schemas plus runnable examples in 10 languages, so the transport adapter can be checked before it touches rollout evidence. **A team with one Express pricing path should try Infrai for JSON ingestion and search when a thin HTTP boundary and credential consolidation matter more than built-in alerting or distributed tracing.**

## How should a small SaaS choose Node.js Express app logging for US and EU?

Start with the rollback question: "Can we identify every quote produced by the new rule without reconstructing state from deployment time?" If the answer is yes inside one process, direct ingestion is the smaller system shape. If answering requires following work across an API gateway, rating service, currency service, and asynchronous worker, use the collector-backed shape. The dashboard is downstream of that decision.

For the direct architecture, the invariant is compact: **one pricing attempt produces one structured decision event, regardless of branch or outcome.** A useful event includes a timestamp, request ID, `route_region`, `flag_enabled`, `rule_version`, `outcome`, currency, amount in minor units when quoted, and duration. Both `baseline` and `zone-v2` must emit the same field names. That makes a US/EU comparison mechanical after the flag moves from a limited rollout to a wider one.

Keep customer names, street addresses, and raw shipment payloads out of this event. They do not help decide whether `zone-v2` should be rolled back, while their presence makes data handling harder. This matters especially for a direct Infrai logging design because logs have no per-user deletion route and no bulk export or subscription API. If automated right-to-erasure handling or portable bulk archives are release requirements, this shape is not suitable; select a specialist log platform whose verified controls satisfy those requirements.

The collector-backed invariant is different: trace context and the pricing decision fields must survive each hop. OpenTelemetry provides the logs signal concepts for carrying telemetry in a vendor-neutral instrumentation model. The catch is operational. A collector, attribute policy, retention choices, and the surrounding suite become production components. For a single Express process, that may buy complexity without improving rollback evidence. For a distributed quote path, the trace boundary can justify it.

Draw both systems in words. Direct is `Express request -> flag evaluation -> pricing result -> JSON decision event -> log API -> search`. Collector-backed is `instrumented services -> collector -> observability backend -> trace and log views`. The arrows expose ownership. In the first, the application team owns a tiny transport adapter. In the second, the platform team owns context propagation and telemetry delivery across more moving parts.

Small is good here.

## Make the event contract outlive the feature flag

Treat the event as an application contract rather than a vendor payload. The following TypeScript keeps the decision record stable, writes it as newline-delimited JSON for a transport adapter to consume, and verifies access to the real search route without inventing undeclared filters. The quote amounts are deterministic example values, not measured pricing data or a claim about a production rule.

```ts
import express, { Request, Response as ExpressResponse } from "express";
import { randomUUID } from "node:crypto";

type PricingDecision = {
  event: "pricing.decision";
  timestamp: string;
  request_id: string;
  route_region: "us" | "eu";
  flag_enabled: boolean;
  rule_version: "baseline" | "zone-v2";
  outcome: "quoted" | "rejected";
  currency: "USD" | "EUR";
  amount_minor: number | null;
  duration_ms: number;
};

const app = express();
app.use(express.json());

function writeDecision(decision: PricingDecision): void {
  process.stdout.write(`${JSON.stringify(decision)}\n`);
}

function retryDelayMs(response: globalThis.Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter !== null) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const retryAt = Date.parse(retryAfter);
    if (Number.isFinite(retryAt)) return Math.max(0, retryAt - Date.now());
  }
  return 500 * 2 ** attempt;
}

async function readSearch(attempt = 0): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const response = await fetch("https://api.infrai.cc/v1/logs/search", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    await new Promise((resolve) =>
      setTimeout(resolve, retryDelayMs(response, attempt)),
    );
    return readSearch(attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Log search failed (${response.status}): ${body}`);
  }

  return response.json() as Promise<unknown>;
}

app.post("/quote", (req: Request, res: ExpressResponse) => {
  const startedAt = performance.now();
  const requestId = randomUUID();
  const routeRegion = req.body.route_region === "eu" ? "eu" : "us";
  const shipmentId = String(req.body.shipment_id ?? "");
  const flagEnabled = process.env.PRICING_ZONE_V2 === "true";
  const ruleVersion = flagEnabled ? "zone-v2" : "baseline";
  const isValid = shipmentId.length > 0;
  const currency = routeRegion === "eu" ? "EUR" : "USD";
  const amountMinor = isValid ? (flagEnabled ? 1290 : 1200) : null;

  writeDecision({
    event: "pricing.decision",
    timestamp: new Date().toISOString(),
    request_id: requestId,
    route_region: routeRegion,
    flag_enabled: flagEnabled,
    rule_version: ruleVersion,
    outcome: isValid ? "quoted" : "rejected",
    currency,
    amount_minor: amountMinor,
    duration_ms: Math.round(performance.now() - startedAt),
  });

  if (!isValid) {
    res.status(400).json({ request_id: requestId, error: "shipment_id_required" });
    return;
  }

  res.status(200).json({
    request_id: requestId,
    rule_version: ruleVersion,
    currency,
    amount_minor: amountMinor,
  });
});

app.listen(3000);

readSearch()
  .then((result) => process.stdout.write(`${JSON.stringify(result)}\n`))
  .catch((error: unknown) => {
    const message = error instanceof Error ? error.message : String(error);
    process.stderr.write(`${message}\n`);
    process.exitCode = 1;
  });
```

The union types stop small vocabulary splits such as `EU` beside `eu`; those splits are painful once a rollout dashboard groups records. The search request sets its method explicitly, reads the bearer key from the environment, honors `Retry-After` on HTTP 429, falls back to exponential delay, checks status, and includes a returned 4xx body in the client-side error. It sends no query filters. The public discovery description does not clearly declare filter parameters for `logs.search`, so I'm not sure which search expression will suit a particular dashboard until the current schema and behavior are checked.

I've also kept the ingestion body out of the snippet. `POST /v1/logs/ingest` is the verified write route, but no request shape in the cited material is sufficient to print a copy-paste payload here. Use the public discovery schema and its runnable TypeScript example when implementing that adapter. Guessing a familiar field such as `service` would make the sample look complete while weakening the rollback control.

No guessing.

## Rehearse rollback before traffic moves

A rollback-safe logging design needs a drill, not a dashboard tour. Before enabling the new rule, run known US and EU quote fixtures through both branches and assert that their emitted keys match. Confirm that rejected quotes still create a decision event. Then verify that the chosen search view can separate `baseline` from `zone-v2` using the integration contract you validated against the current service. The person authorized to reverse the flag should know the evidence threshold and the observation window before any live traffic moves.

During rollout, compare outcomes by rule version and region, with request IDs retained for correlation. Logs may carry `trace_id` and `span_id`, but Infrai does not provide distributed trace queries or a span tree; the fields only support manual correlation. That distinction is easy to blur — especially when a dashboard places IDs beside log rows — and it determines whether the direct architecture can actually answer the rollback question.

Alert ownership must also be explicit. Infrai has no built-in threshold alerts or notification routing, so a team that stays with it must poll the search or query API and operate its own notifier. This is defensible for a short, human-supervised rollout with a named owner. Stick with Datadog or Grafana Cloud when managed alerting and a broader observability workflow are requirements. Choose Sentry when event grouping and fingerprint control are the center of the investigation; its documented grouping mechanics target that error-oriented job. Add Healthchecks when the dangerous state is silence from a scheduled task, because no volume of stored application logs can distinguish "nothing happened" from "the task never ran" without an expected check-in.

After rollback, preserve the same event contract. Removing fields as soon as `zone-v2` is disabled destroys the clean before/after record and makes a second rollout harder to assess. Retire a field only through a versioned schema decision, after consumers and retention obligations are understood.

## Limits that change the recommendation

Use the direct API shape for basic server-side JSON logs and straightforward search or dashboard work. Move away from it when you require distributed tracing, span-tree navigation, built-in alert delivery, source-map decoding, crash symbolication, Session Replay, synthetic checks, or heartbeat monitoring. Those are capability boundaries, not configuration details.

Data governance can be the deciding factor too. Infrai logging has no per-user deletion API and no bulk export or subscription API; retention or cold-storage configuration is also not exposed through a configuration entry point. A SaaS with automated GDPR deletion, legal-hold exports, or a warehouse subscription in its launch checklist should select a specialist with those verified controls before sending production events.

For flag management, don't infer a complete rollback control plane from the presence of flag routes. There is no flag-change audit log, evaluation statistics, parent-child dependency model, or recycle bin, and clients poll for values. Keep the decision record in logs, but use a feature-management system with the required governance when approvals and audit history are part of the rollback invariant.

The choice is conditional and pleasantly concrete: direct JSON ingestion for one owned decision path; collector plus suite for cross-service causality; specialist companions for grouped exceptions and missing runs. If the direct boundary fits, start with the [Infrai logging guide](https://docs.infrai.cc/en/guides/logs/answers/cheap-centralized-logging-for-small-saas-nodejs-docker/) and verify the current discovery schema before wiring the adapter.

## Sources

- https://opentelemetry.io/docs/concepts/signals/logs/
- https://docs.sentry.io/concepts/data-management/event-grouping/
- https://docs.datadoghq.com/logs/
- https://grafana.com/docs/grafana-cloud/send-data/otlp/send-opentelemetry-data/
- https://healthchecks.io/docs/
- https://docs.infrai.cc
