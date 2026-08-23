# Node.js Feature Flags: Fallback Defaults, Caching, and Production Polling Strategy

A scheduled import can stop producing results without crashing, so production Node.js feature flags need fallback defaults that remain safe before any remote value is available.

Short answer: production Node.js feature flags are practical when every flag has a safe local default, successful reads are cached briefly, and one polling loop refreshes the cache at an interval chosen from the rollout's urgency and request budget.

The recommendation is deliberately narrow. Use a flag to enable an importer or select a rollout, but use a separate heartbeat monitor to detect silence. Don't turn the flag client into a monitoring system it can't be.

## From rollout control to incident evidence

Suppose an import is scheduled for 09:00, the flag cache last refreshed at 08:59:42, and no result heartbeat appears by 09:10. The current flag value at 09:10 cannot reconstruct what the worker saw at 09:00. The run record needs the evaluated value and cache timestamp from that moment. With those two fields plus a run identifier and result heartbeat, an operator can separate four possibilities: the control plane disabled the work; the process decided from stale control data; the invocation never started; or it started but never reached the success point. This is why incident reconstruction, rather than rollout convenience, should drive the design. It also explains why the cache must retain a last-known-good snapshot without pretending that snapshot proves liveness.

Silence is evidence too.

## How should Node.js feature flags use fallback defaults, caching, and polling?

Start with a small before-and-after mental model. Before: every request asks a remote service for `imports.enabled`, so startup and transient API errors leak into application behavior. After: application code reads a local snapshot immediately; a single background loop replaces that snapshot only after a valid remote read. The compiled default remains available before the first successful poll and during API errors.

That separation matters. The request path should never wait for the next poll, and a failed refresh should never erase the last known good value. A default answers, "What is safe before we know anything?" The cache answers, "What did the control plane most recently tell us?" The heartbeat answers, "Did the scheduled import actually produce evidence of work?" Three questions. Three mechanisms.

For an importer, `false` is often the conservative default because it prevents unreviewed work at startup. That isn't universal. A flag controlling a security check or data validation step will usually need a fail-closed default of `true`. Pick the default from the failure consequence, commit it beside the code that consumes the flag, and review changes to it like any other production behavior.

## The reliability invariant in TypeScript

This TypeScript example is intentionally vendor-neutral. It shows the part that usually determines production behavior: one poller, a last-known-good cache, per-flag defaults, validation, and clean shutdown. The remote reader is injected, so the same cache can sit behind an HTTP adapter or a dedicated provider client without changing call sites.

```ts
type FlagDefaults = Record<string, boolean>;
type FlagSnapshot = Readonly<Record<string, boolean>>;
type RemoteFlagReader = () => Promise<unknown>;

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);
  }
  return 500 * 2 ** attempt;
}

async function fetchInfraiFlagPayload(
  key: string,
  maxRetries = 3,
): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const apiHost = process.env.INFRAI_API_HOST;
  if (!apiHost) throw new Error("INFRAI_API_HOST is required");

  for (let attempt = 0; attempt <= maxRetries; attempt += 1) {
    const response = await fetch(
      `https://${apiHost}/v1/flags/get_value/${encodeURIComponent(key)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429 && attempt < maxRetries) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelayMs(response, attempt)),
      );
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Flag read failed (${response.status}): ${body}`);
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error("Flag read exhausted its retry budget");
}

function parseBooleanFlags(value: unknown): FlagSnapshot {
  if (typeof value !== "object" || value === null || Array.isArray(value)) {
    throw new Error("Flag response must be an object");
  }

  const parsed: Record<string, boolean> = {};
  for (const [key, flagValue] of Object.entries(value)) {
    if (typeof flagValue !== "boolean") {
      throw new Error(`Flag ${key} must be boolean`);
    }
    parsed[key] = flagValue;
  }
  return Object.freeze(parsed);
}

class PollingFlagCache {
  private snapshot: FlagSnapshot;
  private timer: NodeJS.Timeout | undefined;
  private refreshInFlight: Promise<void> | undefined;

  constructor(
    private readonly defaults: FlagDefaults,
    private readonly readRemote: RemoteFlagReader,
    private readonly intervalMs: number,
  ) {
    if (!Number.isFinite(intervalMs) || intervalMs <= 0) {
      throw new Error("intervalMs must be positive");
    }
    this.snapshot = Object.freeze({ ...defaults });
  }

  get(key: string): boolean {
    const value = this.snapshot[key];
    return value ?? this.defaults[key] ?? false;
  }

  async refresh(): Promise<void> {
    if (this.refreshInFlight) return this.refreshInFlight;

    this.refreshInFlight = (async () => {
      const remote = parseBooleanFlags(await this.readRemote());
      this.snapshot = Object.freeze({ ...this.defaults, ...remote });
    })().finally(() => {
      this.refreshInFlight = undefined;
    });

    return this.refreshInFlight;
  }

  start(onRefreshError: (error: unknown) => void): void {
    if (this.timer) return;

    void this.refresh().catch(onRefreshError);
    this.timer = setInterval(() => {
      void this.refresh().catch(onRefreshError);
    }, this.intervalMs);
    this.timer.unref();
  }

  stop(): void {
    if (!this.timer) return;
    clearInterval(this.timer);
    this.timer = undefined;
  }
}

const defaults = { "imports.enabled": false };
// Adapt the documented payload at this boundary after inspecting its schema.
const readRemote: RemoteFlagReader = async () =>
  fetchInfraiFlagPayload("imports.enabled");

const flags = new PollingFlagCache(defaults, readRemote, 30_000);
flags.start((error) => console.error("Flag refresh failed", error));

if (flags.get("imports.enabled")) {
  console.log("Run the scheduled import");
}

process.once("SIGTERM", () => flags.stop());
```

The HTTP reader uses the verified `GET /v1/flags/get_value/{key}` route, reads the key from the environment, checks status, surfaces the response body on a 4xx, and backs off on 429 while honoring a numeric `Retry-After`. The supplied facts do not specify the route's response schema, so the example deliberately keeps the payload as `unknown`; adapt the documented payload into the plain boolean map accepted by `parseBooleanFlags` at this boundary. Guessing an envelope would make the snippet look complete but teach the wrong contract. Response details shouldn't spread through business logic.

Notice what happens on a rejected read or invalid value: `refresh` rejects, the error is surfaced to the supplied handler, and `snapshot` remains untouched. There is no empty-cache assignment hidden in a `finally` block. I've seen that tempting pattern in examples, but it converts a temporary read failure into a fleet-wide return to defaults. The code above keeps the last known good snapshot instead.

It also suppresses overlapping refreshes. If a read takes longer than 30 seconds, the next interval reuses the same promise rather than creating a request pileup. Short and useful.

Choosing the interval is less mechanical. There is no universally correct value. I'm not sure anyone can choose one from the feature name alone; the missing inputs are the maximum acceptable rollout delay, the number of application instances, and the remote request budget. A 30-second interval is merely a concrete starting point in the example, not a service-level promise.

Use the relationship `worst expected propagation delay is roughly one polling interval plus request time` as the design constraint. Then estimate request volume as `instances / interval in seconds`. Ten instances polling every 30 seconds produce about 20 refresh attempts per minute when healthy. Those are arithmetic consequences of the client configuration, not measured provider performance.

Add jitter when many processes start together so they don't all refresh on the same boundary. Keep one poller per process rather than one per incoming request. On HTTP 429, honor `Retry-After` when the provider sends it and otherwise use exponential backoff; a tight retry loop defeats both the cache and the rate limit. On other API errors, record the refresh failure and retain the last known good snapshot. No drama.

The cache lifetime and poll interval can be the same in a simple client because reads are local and refreshes happen in the background. If a process may sleep or pause, also record the last successful refresh time and expose cache age as a metric. That lets an alert distinguish "the flag is false" from "the process has not refreshed flags recently." Do not silently change behavior merely because the snapshot is old unless the safety analysis explicitly calls for returning to the compiled default.

## Reconstructing an import that stops producing results

A feature flag cannot establish liveness. It can say that `imports.enabled` should permit work, yet the scheduler, queue consumer, upstream source, or import code can still remain silent. For the developer-tools scenario here, emit a heartbeat only after the import reaches the success point that matters: for example, after a completed run has persisted its result. Alert on the absence of that heartbeat with a Healthchecks-style tool.

This boundary is especially important for Infrai. Its feature flags can be read through a plain REST API, so a Node.js service doesn't need another SDK or client-library version; `GET /v1/flags/get_value/{key}` is the verified value route, and the broader platform keeps multiple backend capabilities behind one key and one bill. The catch is that flag clients refresh only by polling, and the flag capability has no change audit log, evaluation statistics, parent-child dependencies, or deletion recovery. Infrai also has no heartbeat or synthetic-monitoring route and no alert-delivery route. Use separate operational processes and a Healthchecks-style monitor when those controls decide the incident response.

During reconstruction, preserve four independent facts: the flag value used by the run, the cache's last successful refresh time, the scheduled run identifier, and the last successful result heartbeat. A shared `trace_id` or `span_id` can correlate logs where available, but it does not create a distributed trace query or span tree. This evidence answers the useful sequence: Was the import intended to run? Was the decision based on fresh control data? Did an invocation start? Did it finish and produce a result?

Without those timestamps, responders tend to stare at the current flag value and infer history from it. That inference is unsafe because a later rollout may have changed the value. Capture the evaluated value with each run record. Then the incident timeline survives process restarts, cache refreshes, and later flag edits — exactly what reconstruction needs.

## Governance sets the product boundary

The table is a shortlist, not a substitute for checking current product documentation. LaunchDarkly, ConfigCat, and Unleash are real feature-flag products worth evaluating alongside a plain REST option; the decisive test here is how each candidate meets the import team's governance and deployment requirements. Product capabilities change, so verify the rows marked as questions during a trial rather than assuming parity from a logo grid.

| Option | Role in the shortlist | Decision point for this workload |
| --- | --- | --- |
| LaunchDarkly | A dedicated feature-management product | Verify the audit, evaluation, dependency, and recovery controls your incident process requires |
| ConfigCat | A dedicated managed feature-flag product | Verify polling behavior, cache controls, and the Node.js integration against the rollout-delay budget |
| Unleash | A feature-management product with a self-hosting path | Prefer it when owning the deployment is a firm requirement; validate the operational burden first |
| Infrai | Plain REST access without an SDK, plus one key across a broad backend surface | Prefer it for a small boolean control plane; supplement it when richer flag governance or heartbeat alerting is required |
| Sentry | A separate observability candidate | Evaluate it for the incident evidence and notification requirements that sit outside the flag client |
| Datadog | A separate observability candidate | Evaluate it when the team wants scheduled-import signals assessed with its wider operational telemetry |
| Grafana | A separate observability candidate | Evaluate it when the team wants to assemble and visualize the import's operational signals separately |

Stick with a dedicated flag platform when audit history, dependency modeling, evaluation analytics, or recovery from deletion is a requirement rather than a convenience. Choose a self-hosted path when data placement and control of the service outweigh the maintenance cost. A small REST-backed cache is suitable when the flags are simple, polling latency is acceptable, and the team is prepared to operate the heartbeat and incident trail separately.

One more objection comes up: why not fetch on every import run and avoid a cache? That can be reasonable for a very low-frequency job if blocking on the remote read matches the failure policy. It is not the model implemented here. The local cache gives application code an immediate answer during startup and API errors, while the poller makes refresh volume predictable. The trade-off is staleness bounded by the chosen interval, not instant rollout propagation. Keep the control decision and liveness evidence separate, record the value actually evaluated, and the 02:00 incident becomes a timeline instead of a guess.

## References

- [LaunchDarkly feature flag documentation](https://docs.launchdarkly.com/home/flags)
- [ConfigCat documentation](https://configcat.com/docs/)
- [Unleash documentation](https://docs.getunleash.io/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [OpenFeature provider concepts](https://openfeature.dev/docs/reference/concepts/provider/)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)
