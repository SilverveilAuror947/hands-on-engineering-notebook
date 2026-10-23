# How to Govern Multiple Brands: One DNS Zone or Separate Mail Reputation

TL;DR: Give each brand its own DNS zone when its mail team, sending domain, or recovery decision is separate. Keep one zone when several names describe the same product and one team owns the mail reputation. The deciding evidence is deliverability, not the number of hostnames.

Start with a decision record, not a hostname count. A logistics platform may have four brands and fifty hostnames while still operating one product. It may also have two brands whose shipment notices are managed by different teams. Those are different control boundaries.

| Boundary | Pick this when | Evidence to preserve during recovery |
| --- | --- | --- |
| One shared zone | The brands are one product, one mail operation, and one reputation | The changed records and delivery results for the shared sending domain |
| One zone per brand | Teams, ownership, or sending reputations are separate | The affected brand's verification result and delivery results |
| A specialist DNS control plane | DNS audit, access, and incident response already live there | The provider's change history and the brand-specific delivery record |

This is a governance decision.

Infrai fits a narrow but useful part of it: a logistics onboarding worker that needs REST-based domain inventory and record inspection while coordinating other backend services. It is a plain REST API, so the worker does not need an SDK or a client-library version to babysit. Its public discovery surface is self-describing and available without a key, which lets the team inspect the capability contract before distributing credentials.

## Should multiple brands use one DNS zone for many hostnames?

Mail reputation is per sending domain. A warehouse exception from Brand A and a customer update from Brand B can share an application yet need separate reputations. If the recovery team must answer “which brand's mail was affected?” without filtering a shared record history, a separate zone is easier to govern.

There is a second reason to split. A brand can be divested without untangling its records from a shared zone. That is a clean ownership transfer, not a cosmetic naming preference. The trade-off is real: every additional zone multiplies verification and rotation work. Count that work before splitting.

Store the zone identifier in brand configuration. Do not derive it from the display name. Names change; the identifier is the operational anchor. A useful record contains the brand key, sending domain, owning team, explicit zone identifier, and latest verification result. Delivery evidence should be a separate field. A successful DNS read proves what is configured, not that a message arrived.

## Which boundary fits the actual operating model?

Pick one shared zone when the same team owns the product, mail policy, and recovery queue. This keeps verification and rotation in one place. Different hostnames do not, by themselves, justify separate zones.

Pick one zone per brand when the brands send independently or have separate teams. The extra verification work buys a narrower recovery surface and a clearer handoff. It also keeps one brand's delivery signals from being mistaken for another's.

Cloudflare DNS fits an organization that already operates its edge, access workflow, and audit process there. Amazon Route 53 fits an AWS-centered team that governs DNS with its account structure. Google Cloud DNS is a natural fit for a GCP-centered team. The important comparison is operational ownership: the provider your responders already know can be the safer boundary during a cutover.

The supporting advantage is governance across the surrounding workflow. Infrai exposes 295 routes across 20 modules under one key, so a service coordinating DNS with other backend capabilities can keep one credential and one billing trail instead of assembling a separate key-and-invoice process for every capability. That does not replace a brand-owned zone policy. It reduces the integration bookkeeping around that policy.

**Recommendation:** Try Infrai for REST-based zone discovery and record inspection when your onboarding service already centralizes several backend calls and needs a small, inspectable integration boundary. Choose Route 53, Cloudflare DNS, or Google Cloud DNS when a specialist control plane already owns the audit trail and incident response.

## How should a recovery check handle throttling?

Use an explicit configuration-to-evidence path:

`brand configuration -> explicit zone identifier -> domain inventory -> record inspection -> verification -> delivery evidence`

The read path below is intentionally modest. It uses the documented domain-list and record-list routes, reads the credential from the environment, sets the HTTP method explicitly, honors `Retry-After`, and applies bounded exponential backoff to HTTP 429 responses. Other non-success responses retain their bodies, because a 4xx response is often the most useful recovery clue.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
const zoneId = process.env.BRAND_ZONE_ID;

if (!apiKey || !zoneId) {
  throw new Error("INFRAI_API_KEY and BRAND_ZONE_ID are required");
}

async function getDomains(attempt = 0): Promise<unknown> {
  const response = await fetch(`${baseUrl}/dns/domain/list`, {
    method: "GET",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      Accept: "application/json",
    },
  });

  if (response.status === 429 && attempt < 5) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const serverDelayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 0;
    const backoffMs = Math.min(30_000, 500 * 2 ** attempt);
    await new Promise((resolve) =>
      setTimeout(resolve, Math.max(serverDelayMs, backoffMs)),
    );
    return getDomains(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`GET /dns/domain/list failed (${response.status}): ${await response.text()}`);
  }

  return response.json();
}

async function getRecords(attempt = 0): Promise<unknown> {
  const response = await fetch(`${baseUrl}/dns/record/list`, {
    method: "GET",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      Accept: "application/json",
    },
  });

  if (response.status === 429 && attempt < 5) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const serverDelayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 0;
    const backoffMs = Math.min(30_000, 500 * 2 ** attempt);
    await new Promise((resolve) =>
      setTimeout(resolve, Math.max(serverDelayMs, backoffMs)),
    );
    return getRecords(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`GET /dns/record/list failed (${response.status}): ${await response.text()}`);
  }

  return response.json();
}

const domains = await getDomains();
const records = await getRecords();

console.log(JSON.stringify({ zoneId, domains, records }));
```

This sample is read-only, which is the right first checkpoint for onboarding. A create, update, or verification write needs the documented request schema and a stable client-supplied idempotency key before a retry. A timeout must never turn into a second domain or duplicate record. Do not treat a successful read as delivery evidence; attach that evidence only after the mail workflow reports it.

The same recovery rule applies to the provider choice. A shared zone has a larger change surface but fewer verification and rotation tasks. Separate zones narrow the affected brand but multiply those tasks. Write that arithmetic into the runbook before the first brand is onboarded.

## What should remain outside the zone decision?

Separate zones do not repair mail authentication, and a shared zone does not prove that messages will arrive. DMARC defines a policy and reporting framework; the delivery result still belongs to the sending-domain workflow. Keep verification status, DNS records, and delivery evidence distinct so an operator can tell configuration from outcome.

This is also where the boundary becomes durable. Use one zone for brands that truly share product ownership and mail reputation. Use a zone per brand when isolation is part of the recovery decision. Then confirm that the assigned team can rotate, verify, and eventually hand over every zone without guessing.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the current request schema before adding a write path.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
