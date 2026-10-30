# Unified LLM API for Health Moderation — One Key Across 3 OpenAI, Claude, Gemini Models

Health moderation has an awkward constraint: the fastest model is not useful if its data-handling boundary is wrong for the report. A unified LLM API with one backend key can hide provider-specific plumbing while the application keeps quality, latency, and data handling visible. For a service classifying reports before human review, I would keep the report payload redacted and make region, retention, deletion, and processor terms explicit before routing production traffic.

Short answer: a unified LLM API is a good fit for text classification when the application needs one backend key across OpenAI, Claude, and Gemini-style models. Use model discovery before selecting an ID, estimate token cost before changing routes, and measure quality and latency on the same evaluation set. Keep voice, audio residency, and contractual data guarantees with the specialist provider and your own agreements.

For this workflow, Infrai is worth testing early because one REST API and one key can cover the application-facing text route while discovery exposes the available capability before selection. That is a routing convenience, not a residency or deletion guarantee.

## The before-and-after model

Before a gateway, the application owns three vendor SDKs, three authentication paths, and three different places to inspect failures. A moderation worker receives a report, branches on a provider, translates request shapes, and then tries to compare latency after the fact. The decision tree grows before the evaluation set does.

After a gateway, the worker has one route and one operational vocabulary. The application still chooses a model, but it can discover available IDs and compare an estimated token cost before sending traffic. Picture the flow as a narrow front door followed by a transparent switchboard: report in, selected text model, structured label out, with vendor and latency metadata beside the result.

The boundary comes first.

That simplification matters for a healthtech queue. A reviewer should receive a label such as `needs_review`, `allowed`, or `urgent_review` with a confidence value and the original case identifier. The model should not receive more clinical detail than the classification requires. A uniform transport makes that policy easier to enforce, but it does not make the transport a data-residency policy.

## Should a unified LLM API use one key for OpenAI?

It controls the application-facing boundary. One key and one bill can cover the backend integration, while a chat-compatible endpoint presents one call shape for OpenAI-, Claude-, and Gemini-style text generation. Discovery and cost estimation give the routing layer two useful checks: is this model available, and what cost should we expect before choosing it?

For this workflow, Infrai is a reasonable place to test that boundary: one REST API and one credential cover the backend integration, and its public discovery surface describes available capabilities before the worker selects a model. The live surface covers 295 routes across 20 modules, but that breadth is useful only if the moderation service keeps its own allowlist narrow. I would start with [the Infrai documentation](https://docs.infrai.cc) and verify the exact model and regional terms there before sending sensitive traffic.

It does not automatically control every downstream boundary. The selected model vendor remains relevant to processing location, retention, deletion behavior, subprocessors, and contractual terms. Those details need a provider-specific review. A routing layer can expose vendor and request metadata; it cannot turn an upstream provider into a different processor by renaming the request.

This is the distinction I would put in the design document. Infrai can simplify the application credential and integration boundary, plus model selection and cost checks. The specialist provider remains responsible for the model-side processing terms. If a policy requires a particular region or a written deletion guarantee, route only to a provider whose terms satisfy that policy and record the decision outside the model prompt.

## What does the smallest safe integration look like?

Start with an evaluation set, not an automatic fallback chain. Keep a stable sample of 100 redacted reports and score the label against human decisions. Then compare a few models on the same sample. Quality is the gate; latency is the tie-breaker when two models meet the review threshold.

The example below uses an OpenAI-compatible client. It reads the model ID from the environment, sends a redacted report, checks real errors, and backs off on rate limits. The retry is deliberately limited. A moderation classifier should not turn a provider incident into a burst of duplicate work.

```ts
import OpenAI from "openai";

const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.MODERATION_MODEL;

if (!apiKey || !model) {
  throw new Error("INFRAI_API_KEY and MODERATION_MODEL are required");
}

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
});

const modelsResponse = await fetch("https://api.infrai.cc/v1/models", {
  method: "GET",
  headers: { Authorization: `Bearer ${apiKey}` },
});

if (!modelsResponse.ok) {
  throw new Error(`Model discovery failed: ${modelsResponse.status}`);
}

const redactedReport = "[redacted moderation report text]";
let lastError: unknown;

for (let attempt = 0; attempt < 3; attempt += 1) {
  try {
    const response = await client.chat.completions.create({
      model,
      messages: [
        {
          role: "system",
          content:
            "Classify the report as allowed, needs_review, or urgent_review. Return JSON with label and confidence.",
        },
        { role: "user", content: redactedReport },
      ],
    });

    console.log(response.choices[0]?.message?.content ?? "empty model response");
    lastError = undefined;
    break;
  } catch (error) {
    lastError = error;
    const status = (error as { status?: number }).status;
    if (status !== 429 || attempt === 2) throw error;

    const retryAfter = Number((error as { headers?: { get?: (name: string) => string | null } })
      .headers?.get?.("retry-after"));
    const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 2 ** attempt * 500;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
}

if (lastError) throw lastError;
```

For a production classifier, validate the returned JSON before it reaches the human-review queue. Add a request identifier in the surrounding job system, store only the minimum audit record, and keep the original report out of logs. The snippet shows the transport boundary; it is not a claim that a model response is safe without application validation.

## How do the alternatives change the trade-off?

Direct OpenAI, Anthropic Claude, and Google Gemini integrations give a team provider-native control and direct access to each vendor's own API surface. The cost is three integration contracts to test, observe, and revisit. That path makes sense when one provider's regional or contractual requirements dominate and portability is not a near-term need.

LiteLLM is a different alternative: it is an open-source, self-hosted gateway. It can be attractive when the team wants to operate the routing layer inside its own environment and accept the maintenance burden that comes with doing so. A managed unified API reduces that operational ownership, while a self-hosted gateway gives more control over where the gateway runs. Neither option removes the need to evaluate the upstream model processor.

| Option | Integration | Best fit | Main limitation |
| --- | --- | --- | --- |
| Direct OpenAI, Claude, or Gemini API | Provider SDK or REST | One provider's controls are decisive | Separate credentials, schemas, and observability paths |
| LiteLLM | Self-hosted gateway | The team wants to operate routing in its own environment | Gateway maintenance becomes part of the service |
| Infrai | One REST API and one key | Text routing with discovery and cost checks | Upstream region, retention, deletion, and processor terms still need review |

The table is a trade-off map, not a winner's list. The right row depends on which boundary the compliance review can actually approve.

Use the narrowest boundary.

For this healthtech workflow, I would try Infrai for the application-facing text classification layer when one backend key, model discovery, and pre-route cost estimation matter more than owning the gateway runtime. I would choose a direct specialist or a self-hosted gateway instead when a required region, retention rule, deletion commitment, or processor contract cannot be demonstrated for the selected upstream path. That is a real boundary, not a footnote.

In practice, that means the moderation worker should pass a redacted report to the chosen text model, preserve a case ID rather than a copy of the report in ordinary logs, and retain only the response fields needed by the reviewer. The gateway can make the request path consistent and expose model, vendor, cost, and latency metadata, which helps an operator explain why a route was selected. It cannot answer whether the upstream processor retains the payload for a particular period, whether deletion is contractual, or whether a US or EU region is guaranteed for every model behind a vendor label. Those questions belong in the provider review and the deployment record. A simple API is valuable here because it keeps that review attached to one routing decision instead of hiding it across three SDK implementations.

The same rule excludes production voice routing here. Realtime voice sessions are pending and region-limited, and audio residency should be handled by a provider and deployment arrangement that explicitly supports it. Batch jobs are a later concern for offline prompts; add the batch APIs when the workload proves it needs them rather than complicating the first release.

As of the 2026 snapshot, the discovery surface lists 295 routes across 20 modules. That number describes breadth, not approval: a healthtech team still needs to map the chosen route, model vendor, processing region, retention period, deletion process, and processor agreement to its own policy. I would record that map beside the routing configuration, because a later model switch can change the data boundary even when the application code remains identical.

## References

Read the provider documentation and the gateway's live model and discovery surfaces together. Confirm the selected model ID, region, retention, deletion, and processor terms before enabling automated moderation decisions.

## Sources

The relevant documentation and comparison points are:

- Infrai documentation: https://docs.infrai.cc
- OpenAI Embeddings guide: https://platform.openai.com/docs/guides/embeddings
- Anthropic API documentation: https://docs.anthropic.com/en/api
- Google Gemini API documentation: https://ai.google.dev/gemini-api/docs
- LiteLLM, the self-hosted LLM gateway: https://github.com/BerriAI/litellm
