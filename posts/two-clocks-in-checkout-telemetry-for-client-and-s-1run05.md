# Two Clocks in Checkout: Telemetry for Client and Server Flag Cache Mismatch

Every polled feature flag setup trades freshness against noise, and you can't have both for free: a shorter polling interval narrows the window in which the client and the server disagree, and it multiplies the events you store and pay for. Pick the interval from the blast radius of the flag, then spend the logging budget on the moment of decision instead of the moment of refresh. In an edtech checkout — seat selection, coupon, payment step — that means one structured record per evaluation that actually rendered, stamped with the snapshot version it came from.

| Delivery approach | Pick this when | Evidence worth keeping | What it costs |
| --- | --- | --- | --- |
| Long poll, 60s or more | The flag changes copy, layout, or a non-blocking upsell | Evaluated value, snapshot etag, evaluation layer, age of the cache | The stale window is long enough that support tickets arrive before convergence |
| Short poll, 5–15s | Checkout-critical kill switches and payment routing | Same fields, plus the request id that ties the decision to the order attempt | More refresh traffic, more evaluation events, higher log ingestion bills |
| Server-sent updates or a websocket channel | You need convergence in seconds and already run a persistent connection | Connection state transitions alongside evaluations | Reconnect storms become a new failure mode you have to observe |
| Server decides once, client renders the decision | The mismatch itself is unacceptable — money, grades, eligibility | One evaluation record per request, no separate client record | The client can no longer react to a flag change without a round trip |
| Config baked at build time | Flag flips are rare and a deploy is cheap | The build id and the config hash in the release record | No runtime control at all during an incident |

The table is the whole decision. Everything after it explains how to prove which row you are actually living in.

## How do you tell a stale flag cache from a real client and server mismatch during checkout?

Two clocks. That's the whole bug.

The server process refreshes its snapshot on its own timer, renders HTML from whatever it holds at that instant, and ships the response. The browser bundle refreshes on a separate timer that started whenever the tab was opened, and a student who left the checkout page open during a lecture may be holding a snapshot from forty minutes ago. Between those two moments the flag can flip. The server renders the new express-checkout panel, the client script evaluates the old value and hides the submit handler, and the learner gets a form that looks complete and does nothing when clicked. Nobody's evaluator is wrong. The system is eventually consistent, and the disagreement window is exactly the sum of both cache ages plus the propagation delay of the flag service itself — which is why a dashboard showing the intended flag state is useless here: it shows the value you set, never the value some particular render used.

Draw it as a line of arrows and put a timestamp on every one: flag write, server refresh, HTML response, hydration, client refresh, click. Every mismatch report lands in one of those gaps. If you can't name the gap, you don't have enough telemetry yet, and no amount of log searching will conjure it.

The practical test is boring and reliable. Take a failed checkout, pull every flag evaluation attached to it, and compare the snapshot etag and cache age on the server record with the ones on the client record. Same etag means both layers agreed and your bug is somewhere else entirely — the flag was a red herring. Different etags with both ages inside their configured intervals means the system worked as designed and your interval is too long for that flag's risk. Different etags with a client age far past its interval means the refresh loop is starved: a background tab throttled by the browser, a failed request that nobody retried, a service worker serving an old response.

## Pick the row before you write the instrumentation

Long polling is the right default for the flags that decide colors and copy. A ninety-second disagreement on an upsell banner is invisible to the business, and paying for high-frequency refreshes across every open tab in a school district buys you nothing.

Short polling earns its keep on kill switches, where the window between "payments are failing" and "everyone stops seeing the payment step" is the whole point of having a flag at all. Ten seconds of drift is tolerable. Ten minutes is an incident report.

Push delivery converges fastest and brings its own tail. Connections drop, proxies buffer, and mobile clients suspend, so you inherit a second class of stale client that looks healthy until you instrument reconnects and last-message-received timestamps. Worth it above a few thousand concurrent sessions with fast-moving flags; overkill for a nightly enrollment window.

Server-authoritative rendering is the only row that removes the mismatch instead of measuring it. One evaluation per request, embedded into the response, and the client is forbidden from re-evaluating anything that gates money. The catch is that you lose live reaction: a flag flip during a session takes effect on the next navigation, not immediately. For a checkout that is usually the correct trade, and I'd argue most teams reach for client evaluation out of habit rather than a requirement.

## Instrument the decision, not the poll

Here's the discipline that keeps the signal high and the bill low: emit an event when a value is consumed by a decision, never when a cache refreshes. Refreshes are periodic and therefore high-volume and low-information — a 10-second interval across 5,000 open tabs is 43 million events a day that tell you almost nothing. Decisions are proportional to real user activity and each one is directly attributable to something a person saw.

The OpenTelemetry semantic conventions already define a feature-flag attribute group — the flag key and the resolved variant — so use those names rather than inventing your own, and your evaluations join cleanly to spans and error events you already collect. OpenFeature's specification covers the evaluation API and hook points where this recording naturally belongs.

```ts
type Snapshot = { etag: string; fetchedAt: number; values: Record<string, boolean> };

const FLAG_URL = "https://flags.internal.example.edu/snapshot";
const POLL_MS = 15_000;

let snapshot: Snapshot = { etag: "", fetchedAt: 0, values: {} };

async function refresh(): Promise<void> {
  const res = await fetch(FLAG_URL, {
    method: "GET",
    headers: snapshot.etag ? { "if-none-match": snapshot.etag } : {},
  });

  if (res.status === 304) {
    snapshot = { ...snapshot, fetchedAt: Date.now() };
    return;
  }
  if (!res.ok) throw new Error(`flag refresh returned ${res.status}`);

  snapshot = {
    etag: res.headers.get("etag") ?? "",
    fetchedAt: Date.now(),
    values: (await res.json()) as Record<string, boolean>,
  };
}

// Critical flags are always recorded. Everything else is sampled, because the
// point of the event is attribution, not a census of every render.
const ALWAYS_RECORD = new Set(["checkout_express_pay", "checkout_coupon_v2"]);

type Decision = {
  key: string;
  value: boolean;
  variant: string;
  layer: "server" | "client";
  etag: string;
  cacheAgeMs: number;
  requestId: string;
  at: string;
};

function evaluate(key: string, requestId: string, sink: Decision[]): boolean {
  const value = snapshot.values[key] ?? false;
  const decision: Decision = {
    key,
    value,
    variant: value ? "on" : "off",
    layer: "server",
    etag: snapshot.etag,
    cacheAgeMs: Date.now() - snapshot.fetchedAt,
    requestId,
    at: new Date().toISOString(),
  };

  sink.push(decision); // kept in-request so a failure can carry it
  if (ALWAYS_RECORD.has(key) || Math.random() < 0.01) {
    console.log(JSON.stringify({ event: "feature_flag.evaluation", ...decision }));
  }
  return value;
}

export function captureCheckoutFailure(err: unknown, step: string, decisions: Decision[]): void {
  console.error(JSON.stringify({
    event: "checkout.failure",
    step,
    message: err instanceof Error ? err.message : String(err),
    // Every decision that shaped this attempt, sampled or not.
    flags: decisions.map((d) => `${d.key}=${d.variant}@${d.etag}+${d.cacheAgeMs}ms`),
    at: new Date().toISOString(),
  }));
}

setInterval(() => { void refresh().catch(() => {}); }, POLL_MS);
```

Two things in there matter more than the rest. The in-request `sink` means a failed checkout carries every flag decision that shaped it even when the individual evaluation events were sampled away — full fidelity where you need it, one percent everywhere else. And `cacheAgeMs` is what turns "the flag was wrong" into a measurement: it's the number that separates a working-as-designed stale read from a starved refresh loop.

The client version is the same shape with `layer: "client"` and its own etag. Ship both through the same pipeline with the same field names, because debugging this means joining them on the request id.

```text
checkout.failure  step=submit  request=req_7f2  flags=[checkout_express_pay=on@"v-8814"+2100ms]
feature_flag.evaluation  layer=client  key=checkout_express_pay  variant=off  etag="v-8791"  cacheAgeMs=2646000
```

Two etags, one request. The client held its snapshot for forty-four minutes against a fifteen-second interval, which is not eventual consistency doing its job — it's a refresh loop that stopped and never told anyone. Add a heartbeat on the refresh path and that whole class of ticket becomes an alert instead of a mystery.

## Where this approach stops working

Sampled evaluation events are a debugging tool, not an audit trail. If a grading or eligibility decision has to be reconstructable for every learner months later, you need durable per-evaluation records with retention guarantees, and that is a different system with a different budget — cloud log services bill by ingested gigabyte, so unsampled evaluation logs on a high-traffic checkout get expensive fast and the finance conversation arrives before the reliability one.

Client cache age is also only as trustworthy as the client clock. Wall-clock timestamps from a browser drift, get spoofed, and jump when a laptop wakes; compute the age from a monotonic timer where you can, and treat a single client-reported age as a hint rather than proof. If your flag values themselves depend on user attributes evaluated locally, the etag comparison narrows the search but won't finish it — you also need the targeting inputs, which drags personal data into your logging pipeline and back into the retention conversation.

And if the answer is that no mismatch is acceptable, stop tuning intervals. Move the decision to the server, render the outcome, and accept that a flip lands on the next request. Polling is a cache, caches are stale by definition, and observability tells you how stale — it can't make the property go away.

## Sources

- OpenFeature specification: https://openfeature.dev/specification/
- OpenTelemetry semantic conventions: https://opentelemetry.io/docs/specs/semconv/
- RFC 9111, HTTP Caching (freshness, Age, and conditional requests): https://www.rfc-editor.org/rfc/rfc9111.html
- MDN, HTTP conditional requests: https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Conditional_requests
- Google SRE Book, Monitoring Distributed Systems: https://sre.google/sre-book/monitoring-distributed-systems/
- Amazon CloudWatch pricing (per-GB log ingestion): https://aws.amazon.com/cloudwatch/pricing/
