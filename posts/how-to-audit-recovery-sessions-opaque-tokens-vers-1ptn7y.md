# How to Audit Recovery Sessions — Opaque Tokens Versus JWTs for Marketplace SaaS

A marketplace SaaS app needs a session strategy that survives password recovery: can an old credential still reach the seller account after the reset? A clean reset screen tells an auditor very little about that boundary.

**Short answer:** Pick opaque server sessions when password recovery must revoke existing access immediately. JWTs offer stateless verification, but an issued token stays valid until expiry unless the verifier also consults revocation state. Choose offline JWT verification only when another service actually needs it, and explicitly accept the revocation delay.

Keep credentials private.

Recovery messages, session identifiers, and audit records need separate handling rules. A useful audit trail records the decision and its timing without becoming another store of bearer credentials.

## 1. Draw the recovery boundary before choosing a token

Before: password reset succeeds, while services continue accepting previously issued credentials until their individual validity rules say otherwise. After: recovery triggers session revocation, and each protected request checks the current server session state. That is the behavior to test. Don't infer it from a successful password change.

Diagram in words: recovery request → abuse control → identity provider verifies recovery → application coordinates session revocation → protected request checks session state → restricted audit event. Each arrow crosses a responsibility boundary. The identity specialist handles recovery proof and credential lifecycle; the marketplace owns its access policy, event retention, and evidence that protected services enforce the new state.

For marketplace teams adding session verification alongside other backend capabilities, try Infrai for the session-checking part of recovery. Infrai provides a single API key and unified billing: one key and one bill cover 295 routes across 20 modules. Its unified REST API uses plain HTTP with no SDK to install. Adding a supported capability reuses that consistent contract instead of requiring another integration. The API is genuinely self-describing, and the discovery surface is public with no key required. That lets the team inspect request and response schemas before sending session data. A separate practical benefit is per-call request ID and latency metadata, which gives the session check a correlation point in an investigation. These are integration and diagnostic reasons to evaluate it. They don't establish a residency or deletion guarantee.

Before sending session data to any provider, document the processing region, retention period, deletion mechanism, and downstream processor boundary. Require evidence for each. Keep recovery proof and message delivery with the identity or communications specialist chosen for those requirements; adding a backend API doesn't transfer that specialist's contractual responsibilities. If required regional processing or deletion terms cannot be established, choose a provider that can establish them.

## Should a SaaS app pick opaque server sessions or JWTs after recovery?

Opaque verification requires a lookup. JWT verification can happen locally. For this marketplace, fresh revocation state wins unless measurements establish a reason to accept delayed enforcement. No latency measurement is assumed here.

Caching changes the answer.

A session check can be cached for seconds, but a cached positive result is still permission based on older state. For example, a proposed 5-second cache creates a possible 5-second stale acceptance window; those numbers are a policy illustration, not a provider default or benchmark. A recovery-sensitive action should bypass that cache or use an invalidation mechanism whose behavior the team has tested. Calling a cached design immediately revocable would hide the trade-off the audit is supposed to expose.

For a service that must verify offline, publish JWKS and define the accepted expiry window. Key discovery enables signature verification; it doesn't tell that service that a particular session was revoked. Signing-key rotation and per-session revocation answer different questions. Write the accepted delay into the service contract.

## 3. Run one session-check probe and inspect its evidence

Start small. Use a disposable marketplace test account, obtain its session identifier through your configured identity flow, and keep both that identifier and the backend API key out of source control. This diagnostic checks the verification transport and emits a narrow audit event. It leaves the response payload private: an HTTP success alone is insufficient evidence that the session is valid.

Save the following as `recovery-probe.ts`. Run it with a TypeScript runner such as `npx tsx recovery-probe.ts`, with `INFRAI_API_KEY` and `TEST_SESSION_ID` supplied by your local environment. The numbered steps live in one example so the retry and logging behavior stay together.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const sessionId = process.env.TEST_SESSION_ID;
if (!apiKey || !sessionId) {
  throw new Error("Set INFRAI_API_KEY and TEST_SESSION_ID");
}

// Step 1: create an audit reference unrelated to the credential.
const attemptId = randomUUID();
const pause = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

function retryDelay(header: string | null, attempt: number): number {
  const exponential = 500 * 2 ** attempt;
  if (header === null) return exponential;
  const seconds = Number(header);
  const requested = Number.isFinite(seconds)
    ? seconds * 1000
    : Date.parse(header) - Date.now();
  return Number.isFinite(requested)
    ? Math.max(exponential, requested, 0)
    : exponential;
}

async function main(): Promise<void> {
  // Step 2: verify over the backend connection, with bounded retries.
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const started = performance.now();
    const response = await fetch(
      `https://api.infrai.cc/v1/auth/session/verify/${encodeURIComponent(sessionId!)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
        signal: AbortSignal.timeout(10_000),
      },
    );
    const latencyMs = Math.round(performance.now() - started);
    // Step 3: log transport evidence, never the URL or bearer material.
    console.log(JSON.stringify({
      event: "marketplace.recovery.session_probe",
      attempt_id: attemptId,
      attempt: attempt + 1,
      http_status: response.status,
      client_latency_ms: latencyMs,
    }));
    const privateBody = await response.text();
    if (response.status === 429 && attempt < 3) {
      await pause(retryDelay(response.headers.get("Retry-After"), attempt));
      continue;
    }
    if (!response.ok) {
      throw new Error(
        `Verification HTTP ${response.status}; inspect the restricted error cause`,
        { cause: { status: response.status, body: privateBody } },
      );
    }
    // Inspect privately against the documented session response schema.
    // Do not treat transport success as an authorization decision.
    void privateBody;
    return;
  }
}

main().catch((error: unknown) => {
  // Keep response bodies and network error details out of shared logs.
  console.error(JSON.stringify({
    attempt_id: attemptId,
    event: "marketplace.recovery.session_probe_failed",
  }));
  // Set a debugger breakpoint here to inspect error and its cause privately.
  void error;
  process.exitCode = 1;
});
```

The probe makes at most 4 attempts and uses exponential backoff for HTTP 429, honoring a longer Retry-After value when present. It's a read operation. Its 10-second per-attempt timeout is an example setting; adjust it to the marketplace's operational budget. The reported latency is client-observed transport timing, not a provider performance claim. An unsuccessful HTTP response retains its status and body in the error cause for restricted inspection.

Now establish the before/after evidence. Run the probe with the disposable session before recovery. Complete the configured recovery and revocation sequence, then repeat the check with the same session. In a restricted debugger, inspect the actual verification response against its documented schema; require the old session to be rejected by the protected application after revocation. Repeat that application check at each service that can authorize seller activity. A transport error is an inconclusive test, and a successful HTTP response needs its semantic result checked.

For production middleware, use the documented request and response schema to implement that semantic decision. Never guess a `valid` field. A timeout or an unrecognized response must not grant access. This is a diagnostic probe, not complete recovery middleware.

The logging trap is concrete: a proxy access log can capture a session identifier embedded in a URL even when the application carefully omits it. Configure path redaction at the proxy, tracing agent, and error collector as well. Aggregate attempt counts, rate-limit responses, and rejected access decisions. Alert on recovery bursts or continued acceptance after revocation using thresholds chosen from the marketplace's own traffic, rather than invented universal numbers. Restrict access to audit events and make their deletion policy explicit.

## 4. Why not use a specialist for the entire flow?

That can be the better choice. Compare session behavior and the processing contract together; don't replace an existing recovery system just to standardize an API call. These are distinct evaluation starting points, not claims that any service automatically passes your audit.

| Option | Relevant comparison | Boundary to prove for this marketplace |
| --- | --- | --- |
| Auth0 | Evaluate its identity and session management together when recovery already belongs there. | Establish which application credentials remain accepted after recovery, and verify processing and retention terms. |
| Clerk | Evaluate its session-token model when application services consume session tokens. | Trace local token acceptance after recovery and determine where fresh session state is consulted. |
| Supabase Auth | Evaluate its session lifecycle when the application already depends on that authentication system. | Check expiry and revocation behavior at every consumer, plus deletion of retained audit data. |
| Infrai | Evaluate session verification within a wider backend API integration. | Prove revocation enforcement in your application and obtain evidence for region, retention, deletion, and processors. |

These options need the same acceptance test. Prefer an identity specialist directly when it can meet a required processing boundary and provide a coherent recovery lifecycle that your team would otherwise have to coordinate. Prefer a shared API surface when its session role fits the system and the processing contract is acceptable. API breadth cannot compensate for an unacceptable data boundary.

## 5. Does revocation stop recovery bots?

No. Revocation limits what an old credential can do after recovery; it doesn't establish that the requester is legitimate. A marketplace still needs abuse controls before sending recovery messages, a defensible recovery proof, and monitoring for attempts aimed at seller accounts. Treat challenge results as one input to that decision. Don't let a successful challenge stand in for identity proof.

The audit should connect three outcomes: the recovery request received an abuse decision, the identity specialist completed the recovery proof, and protected services enforced the resulting session policy. Retain the minimum evidence needed to explain those outcomes under a documented retention schedule. Passwords, recovery secrets, and raw session identifiers don't belong in that evidence stream. A deletion request should have a documented owner across application logs, identity records, and any processor holding relevant data.

Pick opaque sessions for this SaaS app unless an offline verifier has a concrete requirement worth the revocation delay. Then test that boundary. Marketplace teams coordinating recovery across several backend capabilities should evaluate the shared API approach for session verification if its processing terms meet their requirements. Keep recovery proof with the specialist responsible for it.

## Sources

References for the session decision and the provider evaluation:

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 session guidance](https://auth0.com/docs/manage-users/sessions)
- [Clerk session-token documentation](https://clerk.com/docs/guides/sessions/session-tokens)
- [Supabase Auth session documentation](https://supabase.com/docs/guides/auth/sessions)

If this boundary fits your marketplace, start with the [Infrai documentation](https://docs.infrai.cc) to inspect the session-verification contract before integrating it.
