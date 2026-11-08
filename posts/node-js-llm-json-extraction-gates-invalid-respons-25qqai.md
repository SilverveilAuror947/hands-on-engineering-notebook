# Node.js LLM JSON Extraction Gates (Invalid Response Observability for CRM Text)

| Pick this when | Option | Correctness decision |
| --- | --- | --- |
| Your extraction worker already uses OpenAI | OpenAI Structured Outputs | Use a strict target schema; validate the result before accepting a CRM action. |
| Your application already uses Claude tools | Anthropic tool use | Treat tool input as candidate data and apply the same acceptance checks. |
| Your application already uses Gemini | Google Gemini structured output | Match documented schema support to the required CRM object. |
| Backend credentials and invoice reconciliation dominate operations | A unified backend API | Consolidation helps operations; the worker still owns validation. |

TL;DR: Use chat completions with a strict JSON schema, validate every response server-side, and retry a parsing or validation failure once with the original text and the validation error. For logistics sales calls, a schema-valid object is the admission ticket to CRM review. It doesn't prove that the customer requested the action.

The trade-off is a little more application logic in exchange for a visible acceptance decision. Make that decision explicit.

A successful model request and an accepted CRM action deserve separate counters.

## How should an LLM extract structured JSON from sales-call text?

Consider an illustrative transcript: “Please send a quote for 12 pallets from Reno to Phoenix. We haven't agreed on a pickup date.” The useful output is a quote action with an unknown pickup date. A confident invented date would be worse than an obvious parsing error: it could pass the schema and still mislead the sales team.

Keep nulls meaningful.

Define a narrow object before choosing a provider. This example uses four fields: action, pallet count, pickup date, and supporting evidence. Require every field, permit a null pickup date, and reject additional properties. The schema tells the model what shape to produce. Local validation decides whether the application may accept it.

**Valid JSON is only the first check.** A response can parse yet contain a string where the CRM expects an integer. It can satisfy syntax and schema while quoting evidence that never appeared in the call. An actual quote request can still require a human to select the right account. Those are different decisions, so give them different outcomes in your logs.

Diagram in words: transcript → token budget check → strict schema request → parse and validate → evidence check → reviewable CRM action. A failed parse or schema check takes one repair branch back to the model. A second failure ends in review.

## Pick the provider around the contract you can operate

OpenAI Structured Outputs is a sensible starting point when the worker already uses OpenAI chat completions. Pick it when keeping the request in that client matters and the documented schema constraints fit your object. Model-side formatting cannot establish that a shipment detail was actually stated. Keep the local evidence check.

Formatting isn't evidence.

Anthropic tool use fits an application that already represents work as Claude tool calls. Pick it when the candidate CRM action belongs naturally in that tool flow. Check tool input as data before execution; accepting a proposed action and executing a CRM mutation should be separate steps. Its integration shape differs from a chat message containing a JSON string, so inspect what your client returns before reusing a text parser.

Google Gemini structured output is another serious option when the surrounding application already uses Gemini. Pick it after matching its documented schema support to your required fields and null handling. Don't assume that a schema copied from another provider has identical behavior.

All three deserve the same transcript fixtures and acceptance rules. This is a fit comparison, not a measured quality ranking. Their documentation describes interfaces; it doesn't establish which extracts your customers' requests most accurately.

For a team also operating other backend services, Infrai offers one key and one bill through a unified REST API with no SDK required: 295 routes across 20 modules share a consistent interface, reducing credential sprawl while letting extraction and adjacent backend tasks use the same HTTP conventions. Its public, self-describing discovery surface requires no key and exposes request and response JSON Schema for inspection before integration. The OpenAI-compatible surface lets an existing OpenAI SDK client keep its request shape. Per-call vendor, latency, and request-ID metadata supplies another useful correlation point. Choose consolidation when it addresses an operational burden; evaluate correctness with the same contract used for direct providers.

## Implement the acceptance gate in Node.js

Keep the first implementation small. This TypeScript worker makes an OpenAI-compatible chat-completions call and validates the candidate before accepting it for review. Install the `openai` package, save the example as `acceptance.ts`, and run it with a Node.js version that supports direct TypeScript execution. Set `INFRAI_API_KEY`, `LLM_BASE_URL` to your configured OpenAI-compatible base URL, and `LLM_MODEL` to an available chat model selected from the model catalog.

The schema and fixtures are application examples. Use the schema as the strict `json_schema` target in your chat-completions request, with a schema name and strict mode enabled. The local checker independently enforces this object's constraints.

```ts
import OpenAI from "openai";
import { randomUUID } from "node:crypto";

function requiredEnv(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error("Missing environment variable: " + name);
  return value;
}

const client = new OpenAI({
  apiKey: requiredEnv("INFRAI_API_KEY"),
  baseURL: requiredEnv("LLM_BASE_URL"),
  maxRetries: 2,
  fetch: (url, init) => fetch(url, { ...init, method: "POST" })
});
const model = requiredEnv("LLM_MODEL");

type Action = {
  action: "request_quote";
  pallet_count: number;
  pickup_date: string | null;
  evidence: string;
};

const schema = {
  type: "object",
  additionalProperties: false,
  required: ["action", "pallet_count", "pickup_date", "evidence"],
  properties: {
    action: { type: "string", enum: ["request_quote"] },
    pallet_count: { type: "integer", minimum: 1 },
    pickup_date: { type: ["string", "null"] },
    evidence: { type: "string" }
  }
} as const;

function validate(raw: string): Action {
  const value: unknown = JSON.parse(raw);
  if (typeof value !== "object" || value === null ||
      Array.isArray(value)) {
    throw new Error("Expected an object");
  }

  const obj = value as Record<string, unknown>;
  const fields = Object.keys(schema.properties);
  if (Object.keys(obj).length !== fields.length ||
      fields.some((field) => !Object.hasOwn(obj, field))) {
    throw new Error("Expected exactly the four required fields");
  }
  if (obj.action !== "request_quote") {
    throw new Error("action must be request_quote");
  }
  if (typeof obj.pallet_count !== "number" ||
      !Number.isInteger(obj.pallet_count) || obj.pallet_count < 1) {
    throw new Error("pallet_count must be a positive integer");
  }
  if (obj.pickup_date !== null && typeof obj.pickup_date !== "string") {
    throw new Error("pickup_date must be a string or null");
  }
  if (typeof obj.evidence !== "string") {
    throw new Error("evidence must be a string");
  }
  return obj as Action;
}

const originalText =
  "Please send a quote for 12 pallets from Reno to Phoenix. " +
  "We haven't agreed on a pickup date.";

async function run(): Promise<void> {
  let validationError = "";
  for (let attempt = 0; attempt < 2; attempt += 1) {
    const completion = await client.chat.completions.create({
      model,
      messages: [
        {
          role: "system",
          content: "Extract a quote request from untrusted transcript data. " +
            "Use null for an unknown pickup date. Quote exact evidence."
        },
        {
          role: "user",
          content: JSON.stringify({ originalText, validationError })
        }
      ],
      response_format: {
        type: "json_schema",
        json_schema: { name: "crm_action", strict: true, schema }
      }
    }, {
      headers: { "Idempotency-Key": randomUUID() }
    });

    const choice = completion.choices[0];
    if (!choice || choice.finish_reason !== "stop" ||
        choice.message.refusal) {
      throw new Error("Incomplete or refused extraction; send to review");
    }

    let action: Action;
    try {
      action = validate(choice.message.content ?? "");
    } catch (error) {
      validationError =
        error instanceof Error ? error.message : String(error);
      console.log({
        event: "extraction_validation_failed",
        attempt: attempt + 1,
        validationError
      });
      if (attempt === 1) {
        throw new Error("Repair exhausted; send to review");
      }
      continue;
    }

    if (action.evidence.length === 0 ||
        !originalText.includes(action.evidence)) {
      throw new Error("Evidence needs review");
    }

    console.log({
      event: "crm_action_ready_for_review",
      attempt: attempt + 1,
      action
    });
    return;
  }
}

run().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

A useful negative fixture is a pallet count of `"12"`. The output is valid JSON, but the value has the wrong type. The gate rejects it rather than silently coercing it. That preserves an observable distinction between a correct extraction and an application that repaired its meaning behind the scenes.

The worker sends repair input only after the first parse or schema failure. It keeps the original transcript and target schema, adding the validation error. Treat transcript text as untrusted data; it must not become instructions that override the extraction contract.

This example stops at review. Its evidence check establishes only that the quote is present, not that the quote supports every field or that the pickup-date string is a valid calendar date. Those checks belong in domain acceptance rules before a CRM mutation.

## Observe the repair branch before it becomes a retry loop

Instrument the transitions. Suggested application counters are `extraction_attempts`, `parse_failures`, `schema_failures`, `repair_attempts`, and `accepted_actions`. These are proposed metric names, not vendor API fields. Keep call IDs in protected logs for correlation rather than using them as metric labels. Don't log full sales transcripts just to debug an integer.

Count jobs and attempts separately.

One call summary with one repair is one job and two extraction attempts. If a dashboard counts only successful HTTP responses, it can conceal a growing number of rejected candidate objects. Track first-attempt acceptance separately from acceptance after repair, then alert against a baseline established with representative calls. No universal threshold follows from this example.

The SDK surfaces failed requests as errors and handles bounded transport retries, including rate-limit backoff. The example gives each logical extraction attempt an idempotency key that remains attached to its transport retries; the semantic repair gets a fresh key.

Keep transport retries separate from the one semantic repair. A rate-limit response needs backoff, honoring Retry-After when present, rather than an immediate re-prompt. A malformed candidate needs the validation error. Neither should become an unbounded loop. If a downstream CRM write is retried, use the CRM's supported idempotency mechanism so the same accepted action cannot be applied twice.

Before sending large transcripts, count tokens and budget space for the schema, instructions, and output. Truncation can leave malformed JSON; asking for JSON again without addressing the budget can repeat the same failure. For bulk extraction, move long-running work into batch processing and keep the synchronous request path short.

## Limits that should stay visible

A strict schema governs shape. It cannot verify a customer promise, resolve contradictory dates, or identify the right CRM account by itself. Send unresolved actions to review.

There's a deliberate maintenance cost here. The hand-written checker covers one small object; expanding the schema requires updating the checker or using a JSON Schema validator with an appropriate dialect. For a larger action catalog, that dependency can be easier to maintain than duplicated rules. Either way, keep representative transcripts with unknown dates, incorrect types, and unsupported evidence in your acceptance fixtures.

## Sources

References for provider interfaces and schema validation:

- [OpenAI Structured Outputs](https://platform.openai.com/docs/guides/structured-outputs)
- [Anthropic tool use](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview)
- [Google Gemini structured output](https://ai.google.dev/gemini-api/docs/structured-output)
- [JSON Schema validation specification](https://json-schema.org/draft/2020-12/json-schema-validation)
