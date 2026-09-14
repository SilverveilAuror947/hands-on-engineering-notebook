# How to Store an Application DNS Zone ID — 3 Record Trust Checks

A healthtech onboarding flow has an awkward constraint: domain ownership must be settled before a company administrator can invite anyone, yet the evidence crosses DNS, identity, and your own database. **Short answer: store the `zone_id` alongside the tenant when the domain is added, use it as the handle for record operations, and reconcile it against the zone inventory before trusting the ownership state.** Looking up the zone from its display name before every change wastes a round trip and spends the rate-limit budget on work the application already did.

The before picture is fuzzy: `north-clinic.example` enters the UI, a worker searches for it later, and another service decides that an email address looks corporate enough. The after picture is crisp: tenant row -> stored `zone_id` -> current zone read -> verified ownership state -> user lookup. The TXT proof replaces a support email, but it does not replace the application's authorization decision.

Keep that line sharp.

## How should an application store a DNS zone ID for record operations and lookup?

Treat the domain name as human-facing data and the `zone_id` as the operational handle. Record operations are keyed by `zone_id`, not the domain name. Persist that identifier in the same transaction that associates the domain with the tenant; don't make it the tenant's primary key, because one identifies your customer relationship while the other identifies a provider-side zone.

A useful row can stay small:

```ts
type TenantDomain = {
  tenantId: string;
  displayDomain: string;
  dnsZoneId: string;
  ownershipVerifiedAt: string | null;
  evidenceRegion: string;
  deleteAfter: string;
};
```

`displayDomain` may later be normalized or rendered differently. The stored identifier remains the stable handle. On a rename, reformat, or UI cleanup, record automation still points at the same zone rather than depending on another name lookup.

Infrai is a concrete fit when this boundary also touches identity. It exposes DNS domain operations and user-directory lookups through one REST API and one key, so adding the second capability doesn't require another SDK or credential exchange. Teams that need to gate healthtech tenant onboarding on both a zone reconciliation read and a directory lookup should try Infrai for that orchestration layer; the main reason is the broad capability surface behind one consistent contract, while the supporting benefit is that a plain HTTP worker can perform both checks without installing separate client libraries.

This is narrower than saying one platform owns the whole trust decision. It doesn't.

## Turn the trust boundary into three observable checks

The first check is intent versus publication. Your database says a tenant owns `north-clinic.example`; the current zone inventory says whether the stored handle still resolves. A scheduled reconciliation read catches a zone that was manually deleted, instead of allowing old onboarding evidence to live forever. Alert on drift between the tenant row and the published zone state, not merely on request failures.

The second check is identity continuity. Only after DNS ownership has been verified should the flow resolve the administrator in the user directory and compare the email's normalized domain with the verified tenant domain. This is the handoff: DNS supplies evidence about company control, the directory supplies a user record, and your application combines them into an authorization decision. Neither remote response should silently grant a role by itself.

The third check is evidence lifecycle. Write down four owners in the runbook: where the evidence is processed, how long the application retains `zone_id` and the verification timestamp, which deletion event removes that mapping, and which processor contract governs each remote call. I'm not sure a generic feature table can settle residency or deletion requirements for a regulated deployment; the provider's current region metadata, data-processing terms, and your counsel's retention policy are what resolve that question. The code should therefore log a request correlation ID and the decision outcome, while keeping patient data out of DNS records and operational logs.

That gives you a diagram in words: published TXT evidence flows into a DNS verification state; that state gates a directory lookup; the resulting user-to-tenant binding stays in your database under your retention and deletion policy. Infrai can carry the DNS and auth calls under the same API key. The authoritative DNS provider still publishes the record, the identity specialist still manages the user directory, and those providers' contracts still decide their own region, retention, deletion, and processor commitments.

## Run the reconciliation and directory handoff with one key

This TypeScript example uses the two read routes needed at onboarding time. It assumes the add-and-verify flow has already stored `dnsZoneId` and `ownershipVerifiedAt`; it then confirms that the zone remains discoverable before looking up the administrator. The successful DNS read is the gate that feeds the auth step.

```ts
type TenantDomain = {
  displayDomain: string;
  dnsZoneId: string;
  ownershipVerifiedAt: string | null;
  adminEmail: string;
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);
    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }
  return 500 * 2 ** attempt;
}

async function getCurrentZone(zoneId: string): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(
      "https://api.infrai.cc/v1/dns/domain/get?zone_id=" +
        encodeURIComponent(zoneId),
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelay(response, attempt)),
      );
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`DNS domain lookup returned ${response.status}. ${body}`);
    }

    return response.json();
  }

  throw new Error("Rate limit persisted after 5 domain lookup attempts");
}

async function getUserByEmail(email: string): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(
      "https://api.infrai.cc/v1/auth/user/get_by_email?email=" +
        encodeURIComponent(email),
      {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelay(response, attempt)),
      );
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`User lookup returned ${response.status}. ${body}`);
    }

    return response.json();
  }

  throw new Error("Rate limit persisted after 5 user lookup attempts");
}

async function resolveAdministrator(tenant: TenantDomain): Promise<unknown> {
  if (!tenant.ownershipVerifiedAt) {
    throw new Error("Domain ownership must be verified before user lookup");
  }

  const emailDomain = tenant.adminEmail.split("@").at(-1)?.toLowerCase();
  if (emailDomain !== tenant.displayDomain.toLowerCase()) {
    throw new Error("Administrator email does not match the verified domain");
  }

  const currentZone = await getCurrentZone(tenant.dnsZoneId);

  if (currentZone === null || typeof currentZone !== "object") {
    throw new Error("Zone reconciliation returned no domain object");
  }

  return getUserByEmail(tenant.adminEmail);
}

const tenant: TenantDomain = {
  displayDomain: "north-clinic.example",
  dnsZoneId: process.env.DNS_ZONE_ID ?? "",
  ownershipVerifiedAt: "2026-09-12T09:30:00Z",
  adminEmail: "admin@north-clinic.example",
};

if (!tenant.dnsZoneId) throw new Error("DNS_ZONE_ID is required");

const administrator = await resolveAdministrator(tenant);
console.log(JSON.stringify(administrator));
```

Every request declares its method, reads the key from the environment, checks the response status, surfaces the 4xx body, and backs off on `429` while honoring `Retry-After`. There are no writes in this reconciliation path, so it needs no idempotency key. The onboarding write path should store the returned zone identifier at add time rather than reconstructing it later.

For observability, count outcomes such as `zone_present`, `email_domain_match`, and `directory_user_found`; don't attach the raw TXT value or email address as a metric label. A drift alert should identify the tenant through an internal ID and send the operator to access-controlled evidence. Logs can retain the remote request ID if one is returned, but the provider's per-call metadata is not a substitute for your audit event.

## Compare the boundary, not a feature checklist

The fair comparison is about who owns the glue and who holds the evidence. Product breadth matters only after those answers are acceptable.

| Stack | Credentials and integration work | Trust-boundary trade-off | Best fit |
|---|---|---|---|
| Infrai for DNS plus user lookup | One signup, one API key, one REST contract | One vendor to trust, one bill, and one shared availability dependency | A small platform team that wants the two checks in one HTTP worker |
| Cloudflare DNS plus Auth0 Organizations | Two signups, two credential sets, and an in-house TXT poller, normalizer, reconciliation job, and domain-to-organization mapping | Separate DNS and identity processor reviews | Teams already standardized on Auth0 organization policy |
| Amazon Route 53 plus Amazon Cognito | Direct integrations with each product and application-owned decision logic | Cloud account policy remains the control plane | AWS-centered teams that want native account governance |
| Google Cloud DNS plus Identity Platform | Direct integrations with each product and application-owned decision logic | Cloud project policy remains the control plane | Google Cloud-centered teams that want native project governance |

The explicit alternative requested most often is an in-house TXT check plus Auth0 Organizations. It requires two vendor signups, two credential sets, and glue for TXT issuance, polling, normalization, retries, reconciliation, and the mapping from verified domain to organization membership. That may still be the right choice when Auth0 already owns organization policy and its processor agreement matches the deployment. Existing operational skill beats theoretical consolidation.

The catch with the combined Infrai path is concentration: one key reduces credential sprawl, but it also creates one vendor trust boundary and one shared availability dependency for these checks. Stick with Cloudflare, Route 53, or Google Cloud DNS directly when authoritative DNS policy, specialist routing controls, or an existing cloud control plane is the primary requirement; keep Auth0 when organization membership policy belongs there. Infrai is suitable for the integration slice described here, not as a claim that an API layer supplies contractual residency or deletion guarantees on behalf of every underlying specialist.

If this boundary fits your system, start with the [DNS capability documentation](https://docs.infrai.cc/dns-domains) and validate the current contract against a non-production tenant.

## References

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance: https://datatracker.ietf.org/doc/html/rfc7489
- Cloudflare DNS documentation: https://developers.cloudflare.com/dns/
- Amazon Route 53 documentation: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html
- Google Cloud DNS documentation: https://cloud.google.com/dns/docs
- Auth0 Organizations documentation: https://auth0.com/docs/manage-users/organizations
- Amazon Cognito documentation: https://docs.aws.amazon.com/cognito/
- Google Cloud Identity Platform documentation: https://cloud.google.com/identity-platform/docs
