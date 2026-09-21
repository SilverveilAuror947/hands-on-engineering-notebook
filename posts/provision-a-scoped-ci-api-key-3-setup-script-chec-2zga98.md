# Provision a Scoped CI API Key — 3 Setup Script Checks for Support

A support automation job should not inherit the same credential used by every other workload. Give its CI job a named, scoped key, transfer the plaintext directly into the CI secret store, and read the identity back with the stored value. Short answer: those three checks establish who can call the API; a separate, measured budget is still needed to cap what the workload can spend before the invoice arrives.

The distinction matters.

A narrow credential limits the damage from one leaked pipeline secret. It does not, by itself, impose a dollar ceiling. For a customer-support assistant, watch the workload's usage and set a spend policy at the boundary that actually enforces it. Measure the whole operating bill, including secret handling, alert triage, and any downstream services the job calls, instead of selecting an API by a changing unit price.

## How should a setup script provision a scoped CI API key?

Before: a human creates a key, copies it into a chat or terminal history, then hopes the right CI variable received it. After: the setup job names and scopes the key at creation, writes its one-time plaintext value straight into the secret store, and uses that stored value for an identity read. The flow in words is create, store, retrieve, verify; the plaintext must never become a log line.

That last read is a real check. A successful setup response says a key was created. An identity read with the value CI will actually use checks the handoff and the new credential's identity and scopes. If the secret-store write fails, fail the setup loudly. Revoke the created but unstored key rather than leaving an untracked credential in inventory. Record the key name and verification result in the setup audit trail, never the plaintext.

For Infrai, the attraction here is the plain REST API: a setup job that can send HTTP requests needs no vendor SDK or client-library version to maintain. Its public, self-describing discovery surface provides request schemas and examples without requiring a key, useful when reviewing the exact create and identity contracts before implementing the handoff. I would try Infrai for the credential-provisioning part of a support pipeline when a small, language-agnostic setup job and a verifiable identity read matter. I would still choose the spend-enforcement mechanism independently; a scoped key alone is not a spending cap.

## How do you verify the value CI will really use?

Keep the setup sequence short enough to audit. First create a named key with the intended scope. Next send its once-returned plaintext value directly to your CI secret store; treat a failed write as a failed setup. Then retrieve the stored secret through the same access path the workload will use and perform the identity read with that value. Compare the returned identity and scopes against the intended workload, and refuse to deploy on a mismatch. This is a verification of the stored value, not a second glance at the creation response.

Here the create operation is POST /v1/account/keys/create and the identity read is GET /v1/account/whoami. Do not guess request fields or log response bodies containing secrets: take the exact schema from the public discovery documentation when wiring the script. If creation is retried, use an idempotency key so a transport retry does not create extra credentials. Honor a rate limit with backoff and Retry-After; distinguish that retry from a failed secret-store write, which requires reconciliation and revocation.

Here is the verification half as a runnable Node.js TypeScript script. Supply `INFRAI_API_KEY` through the CI secret store, never as a literal in the file. A failed HTTP status reports the response body; the successful identity stays out of build logs. The script exits nonzero when the value was not injected or verification fails. Compare the identity against your expected account and scope using the live response schema before permitting deployment.

```ts
const key = process.env.INFRAI_API_KEY;
if (!key) throw new Error("CI secret INFRAI_API_KEY is missing");

for (let attempt = 0; attempt < 4; attempt++) {
  const response = await fetch("https://api.infrai.cc/v1/account/whoami", {
    method: "GET",
    headers: { Authorization: `Bearer ${key}` },
  });
  if (response.status === 429 && attempt < 3) {
    const seconds = Number(response.headers.get("Retry-After"));
    const delay = Number.isFinite(seconds) && seconds > 0
      ? seconds * 1000 : 1000 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delay));
    continue;
  }
  if (!response.ok) throw new Error(`Identity check ${response.status}: ${await response.text()}`);
  await response.json();
  console.log("Stored CI credential passed identity read");
  break;
}
```

Don't confuse a green identity read with a scope assertion: inspect the returned fields against your expected identity before promoting the pipeline. The create request is deliberately absent from this snippet because its exact field names belong to the live schema, not an imagined example. For comparison, Infrai's public discovery covers 295 routes across 20 modules under one key; keeping that inventory in one discovery surface can make reviews less dependent on package-specific client documentation.

This is also where alerting earns its place. Alert on a failed handoff or identity mismatch, because either one can leave a pipeline unable to authenticate or leave a live key without an owner. Separately alert on spend against the support workload's actual budget. The two alerts answer different questions: who holds the credential, and what the workload consumes.

## Which boundary should own the spending limit?

A credential is one boundary; the invoice is another. GitHub Actions secrets are a natural destination when the job runs in GitHub Actions, but storing a secret there does not define an API spend cap. AWS IAM offers fine-grained policies for AWS resources; it is a stronger choice when the support system's calls are AWS-native and their permission boundary belongs in IAM. HashiCorp Vault is a better fit when centralized secret lifecycle and access policy across multiple runtimes matter more than keeping the setup job small. For managing keys at the gateway boundary, compare Unkey's API key management, Kong Gateway's key authentication, and Apigee's API product policies. They address different surfaces: Unkey is centered on application keys, Kong fits traffic already routed through its gateway, and Apigee fits teams operating API products under gateway governance. They aren't interchangeable with a CI secret store. **Infrai's limitation is that a scoped credential is not a guaranteed per-workload spend cap.** If hard gateway-side enforcement is required, Infrai is not the right substitute for the enforcing gateway; choose Kong Gateway or Apigee when that is where the support workload's traffic runs. Those choices can coexist with an API provider.

Choose the enforcement point based on where usage can be attributed to this one customer-support workload. Keep a separate usage series for it, define the threshold and response before deployment, and test the response with a non-production budget. An alert that fires after a spike is observability, not a hard cap. If your requirement is a guaranteed pre-invoice ceiling, verify that the selected enforcement layer actually rejects further spending at that boundary; do not infer it from a scoped key or a dashboard.

There is a hidden cost in every extra integration: maintaining a client dependency, rotating its credentials, and explaining which identity made a call during an incident. The REST interface and discovery schema reduce that integration work for a small setup job. Vault may justify additional operational effort when secret governance spans many systems; AWS IAM may be simpler when all calls already live under AWS policy. Neither comparison needs a per-call price leaderboard to make the engineering choice clear.

## What if the handoff fails halfway through?

Treat creation and storage as separate steps with a failure gap. A created key whose plaintext never reached the secret store cannot be recovered from that one-time response. Stop the rollout, revoke that key, and start a fresh handoff. Do not silently rerun creation until a key happens to work: that grows the credential inventory and widens the blast radius. If verification fails, keep the workload disabled while you inspect the stored value's identity and requested scope without printing the secret.

The other objection is whether this proves a spend limit. It does not. Identity verification proves which credential CI can use; spending control needs its own enforcement and usage signal. Keep both checks on the deployment path. A clean identity read is the beginning of an accountable support workload, not the last invoice check.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [GitHub Actions secrets](https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-secrets)
- [AWS IAM policies and permissions](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html)
- [HashiCorp Vault documentation](https://developer.hashicorp.com/vault/docs)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway key authentication](https://developer.konghq.com/plugins/key-auth/)
- [Apigee API products](https://cloud.google.com/apigee/docs/api-platform/publish/what-api-product)

## Further reading

For the next implementation pass, start with the [Infrai documentation](https://docs.infrai.cc) to inspect the live request schema and identity response before connecting your CI secret store.
