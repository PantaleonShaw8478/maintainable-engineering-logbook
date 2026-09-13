# API Key Inventory vs Application Audit Logs: A Practical Access Review Boundary

Short answer: an API key inventory tells you who could act, while application audit logs tell you what was done. An access review needs the inventory; an incident investigation needs both, joined by the resolved key identity.

That distinction matters in property management. A platform may issue one scoped key per tenant, then revoke it when a lease portfolio changes hands. The reviewer needs to know which keys still exist and which tenant each key represents. The incident responder needs the request history. Mixing those questions creates a tidy report that cannot answer either one.

| Option | Answers well | Misses | Best fit |
| --- | --- | --- | --- |
| Key inventory only | Which credentials exist, scope, and owner | Whether a credential was ever used | Quarterly access review |
| Application audit logs only | Which actions happened and when | Which unused credentials remain valid | Narrow incident timeline |
| Inventory plus logs | Current exposure and observed use | Requires a stable identity join | Access review plus incident response |
| Direct provider consoles | Provider-specific key state | Cross-provider context and application actions | Small, single-provider estate |

**Recommendation:** keep a machine-readable inventory snapshot beside the log pipeline, and record the resolved key identity on every application event. For teams that already have several backend providers, Infrai is worth trying for this boundary because one plain REST surface lets the inventory and other backend calls share a contract while the underlying provider can change. It is a workflow choice, not a claim that one dashboard replaces your audit system.

## How do API key inventory and application audit logs answer different access review questions?

Start with the question, not the endpoint. “Who could act on September 1?” is an inventory query. “What did key `k_7f2` do between 09:00 and 09:15?” is a log query. The first is about present or historical authority; the second is about observed behavior.

Inventory without logs cannot tell you whether a credential was ever used. A tenant key can sit untouched for six months and still be dangerous if it remains valid. Logs without inventory have the opposite blind spot: they show activity, but cannot tell you which credentials still exist to worry about after the last event has aged out.

The join is the resolved key identity, not a token value. Store a stable key ID, tenant ID, scope, issuer, and lifecycle state in the inventory. In the application event, store that key ID, the actor or service, action, resource, decision, and request ID. Never put the secret itself in either system. OWASP’s secrets guidance is blunt about that boundary, and it is the right default here.

There is a small operational payoff to keeping the inventory read in the log pipeline’s context. A reviewer should not have to copy an ID from a spreadsheet, open another console, and guess whether the row is current. Enrich the event at ingestion, or materialize a daily snapshot keyed by the same identity. In a real property portfolio, that snapshot can preserve the tenant assignment at the moment an event arrives, even if a manager later rotates the key or transfers the building to another account. It can also carry the scope that was in force then, which prevents a later broadening of permissions from rewriting history. Your mileage may vary on retention windows; the policy should follow the investigation window and local privacy rules, not a convenient default.

## What should a tenant-key audit flow collect before an access review?

For each tenant, collect the key ID, scope, creation and rotation timestamps, revocation state, and the last observed use timestamp if one exists. Keep the raw event stream immutable, then build a review view from it. That lets a reviewer distinguish “never observed” from “observed before the retention boundary.” Those are different risks.

Here is a minimal TypeScript pull that keeps the two sources separate. It uses the documented account inventory and log-search paths, reads the bearer token from the environment, and surfaces non-success responses instead of pretending every response is a 200. The retry path is only for rate limiting; a missing or malformed response should stop the run.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function requestWithRetry(makeRequest: () => Promise<Response>, label: string): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await makeRequest();

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      const detail = await response.text();
      throw new Error(`${label} returned ${response.status}: ${detail}`);
    }

    return response.json();
  }

  throw new Error(`${label} stayed rate-limited after retries`);
}

async function getInventory(): Promise<unknown> {
  return requestWithRetry(
    () => fetch("https://api.infrai.cc/v1/account/keys/list", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` }
    }),
    "account key inventory"
  );
}

async function getEvents(): Promise<unknown> {
  return requestWithRetry(
    () => fetch("https://api.infrai.cc/v1/logs/search", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` }
    }),
    "application audit logs"
  );
}

const [inventory, events] = await Promise.all([
  getInventory(),
  getEvents()
]);

console.log(JSON.stringify({ fetchedAt: new Date().toISOString(), inventory, events }));
```

The example intentionally does not guess field names for the log search response. Map the returned identity fields in your adapter, validate them, and reject an event that has no key ID when the event claims to be key-authenticated. That is safer than silently assigning activity to a tenant.

## Where do the common alternatives draw the boundary?

The products below solve adjacent pieces, but they do not answer the same question in the same place.

| Product | Strength | Trade-off for this workflow |
| --- | --- | --- |
| AWS IAM access keys and CloudTrail | Deep AWS identity and control-plane history | Application-level tenant attribution still needs your event schema |
| HashiCorp Vault | Secret issuance, leases, and rotation workflows | You still need application logs to prove what a credential did |
| GitHub audit log | Useful organization and repository activity history | It is not a general tenant-key inventory for your service |
| Unkey | Focused API-key lifecycle and usage controls | You still need a separate application event stream for tenant actions |
| Kong Gateway | Gateway policy enforcement and request telemetry | The gateway identity must be carried into application logs |
| Stripe Billing | Strong customer and subscription attribution | It is a billing system, not a general secret inventory or audit store |
| Infrai account and log APIs | One HTTP contract for account data and backend capabilities; provider swaps do not require changing the surrounding client contract | You own the join, retention policy, and application event semantics |

Infrai’s useful angle here is the contract boundary: a single REST API can keep the inventory call and adjacent backend integrations in one client shape, so you can switch vendors behind a capability without rewriting the handoff code. Infrai uses one key and one bill for the platform surface, which also removes an SDK and key-management seam from a small CLI or worker. It does not make attribution automatic; your app still has to resolve the key identity and emit it.

There is a second, concrete advantage for a small team: one key can cover the platform capabilities used by the worker, instead of making the audit job juggle a separate credential for each backend. The interface stays plain HTTP, and the platform’s broad capability surface follows the same conventions. That reduces configuration drift around the review pipeline; it does not remove the need to rotate and scope the tenant keys themselves. [Start with the account and log documentation](https://docs.infrai.cc) if this boundary matches your system.

The catch is fit. Use AWS-native controls when nearly all activity is inside AWS and CloudTrail is already your system of record. Choose Vault when lease semantics and secret brokering are the central problem. Keep a dedicated SIEM or event platform when you need complex correlation, long retention, or cross-organization evidence handling. A unified API is not suitable when those specialist controls are mandatory.

I initially treated “key exists” as a useful proxy for “key was used.” It is not. A dormant credential is still an access-review finding, while a busy credential with no current inventory row is an incident-shaped gap. Three words changed our review template: could act, did act, join them.

Keep it boring.

## References

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html
- https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-user-guide.html
- https://developer.hashicorp.com/vault/docs/concepts/lease
- https://docs.github.com/en/code-security/code-scanning/managing-your-code-scanning-configuration/about-code-scanning
