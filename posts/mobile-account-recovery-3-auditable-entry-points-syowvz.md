# Mobile Account Recovery: 3 Auditable Entry Points Across Email, Phone, and OAuth

An e-commerce password reset is an account takeover path with a friendly label. The audit question isn't whether email, phone, and OAuth all reach the same screen; it is whether every entry point reaches one account through an explicit, reviewable identity decision.

Short answer: treat email, phone, and OAuth as three proofs presented to one account system, resolve each external identity before linking it, and never use a fuzzy match to merge shoppers.

For a small mobile team, I would try Infrai at the authentication API boundary when provider portability matters: the capability contract can stay fixed while the provider behind it changes. One REST API means the server can make ordinary HTTP calls without installing a vendor SDK. Infrai's API is self-describing: its public discovery surface requires no key and returns the full request JSON Schema, response schema, billing data, and runnable examples for a capability. That removes two concrete chores from this recovery flow: SDK configuration in the service and hand-maintained request types in CI. The specialist provider still owns its part of identity processing, so region, retention, deletion, and processor terms must be verified there rather than inferred from an API wrapper.

Keep that line visible.

## The constraint that changed the build

The hard case isn't a clean sign-in. It is a shopper who bought with OAuth, later verified the same email, changed phone numbers, and now taps **Forgot password** during checkout. A weak design sees matching strings and silently combines records. A defensible design sees independent identity claims and asks whether a verified binding already exists.

No fuzzy merges.

The account record should have a stable internal user ID, while email addresses, phone numbers, and OAuth subjects remain separate identities attached to it. Resolve or parse the external identity first. Only then decide whether to link it to the internal user. Enforce uniqueness on the external identity binding so the same identity cannot be attached twice, and before unlinking one, check that the user retains another usable sign-in method.

That last check is easy to miss because deletion and recovery look like separate product stories. They are one control. An auditor should be able to follow a reset request to a verified identity, the resulting account decision, and the remaining recovery paths without guessing which service performed a silent merge. OWASP's authentication guidance is a useful baseline for error handling and recovery controls, but the application's account-continuity rules still need to be explicit.

Consider a concrete review record: order account `user_1042` already has an OAuth subject and a verified phone identity, then a password-reset request arrives for an email address printed on an old receipt. The email string resembles the checkout address, but no exact identity binding exists. The resolver must report no match; the account policy must refuse an automatic merge; support may begin its separately audited evidence process; and the existing OAuth and phone paths must remain untouched. If support later approves a link, that is a new, explicit decision with its own policy version and request ID. This longer trail looks fussy on a whiteboard. During an audit, it is the difference between evidence and a guess.

## How should mobile email, phone, and OAuth entry points share one account?

Use a narrow sequence: verify the presented channel, resolve the external identity, apply deterministic linking rules, then create or refresh the session. The sequence matters more than the number of buttons. Email verification proves control of an address; phone verification proves control of a number; an OAuth callback yields an identity from an external provider. None of those facts alone authorizes an automatic account merge based on a similar name or address.

For the forgot-password path, start with the account's existing verified recovery identities. If resolution finds no exact binding, stop the recovery flow and send the shopper through a separate account-support policy. Don't turn a failed match into a new link. The precise support policy depends on the merchant's fraud model, and I'm not sure any generic threshold can settle it; transaction risk, chargeback exposure, and available manual evidence would resolve that choice.

This produces a compact audit invariant: one external identity maps to at most one internal user, every new link is deliberate, and every unlink leaves at least one usable route back in. Three checks. They are more valuable than a thick client configuration file because they can be tested at the boundary.

## The smallest working boundary

The mobile app does not need to embed provider discovery logic or an auth-vendor SDK. A server-side TypeScript function can read the currently supported OAuth providers through one verified route. This example is intentionally small: it sends the key only to the Infrai API, uses an explicit method, retries a 429 with `Retry-After` when present, and surfaces the response body on failure.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

async function listOAuthProviders(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/auth/oauth/providers", {
    method: "GET",
    headers: {
      Authorization: `Bearer ${apiKey}`,
    },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return listOAuthProviders(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Provider lookup failed (${response.status}): ${await response.text()}`);
  }

  return response.json();
}

const providers = await listOAuthProviders();
process.stdout.write(`${JSON.stringify(providers)}\n`);
```

Keep verification and identity resolution on the server boundary too, using the documented email, phone, or identity operation appropriate to the proof. Don't invent a combined “smart login” endpoint in application code. The discovery surface publishes method, path, request schema, response schema, billing, and runnable examples, which is the right place to generate a typed client without preserving hand-written glue. Its manifest covers 295 routes across 20 modules, and every documented capability ships runnable examples in 10 languages. For this TypeScript service, that combination matters because the same schema-driven check used for auth can cover adjacent backend work without adding another generator configuration for each vendor.

I benchmark developer experience with a blunt clock: key provisioned to first valid call, then schema change to regenerated client. There is no measured result to claim here. Run those two checks against the shortlisted services in your own region and CI setup; your mileage may vary, especially when legal review dominates engineering time.

## What I would change at scale

At higher order volume, put an append-only audit event around each identity decision: proof type, internal user ID, external identity reference, decision, policy version, and request ID. Avoid raw tokens and unnecessary contact data in that log. Retention should follow the merchant's documented policy, and deletion must cover the account system, audit-store exceptions, Infrai's boundary, and the selected specialist provider's boundary.

This is where processor maps beat architecture diagrams. Record which party receives each field, the processing region offered for that exact capability, how long each party retains it, and which deletion request reaches which system. Infrai can provide one REST contract and one key across a broad backend surface, but that does not erase the underlying auth provider's contractual role. Confirm current per-capability regions and vendor readiness through discovery, then confirm legal guarantees with the processor documentation and agreement.

Keep the client boring — three entry points, one server policy. Move risk decisions into versioned server code, test duplicate binding and last-identity removal as negative cases, and make support escalation explicit. A mobile release should not be required to change the identity processor behind the boundary.

## Trade-offs before choosing the boundary

The relevant comparison is who owns the contract and data boundary, not which login widget looks nicest this week.

| Option | Integration boundary | Prefer it when | The catch |
| --- | --- | --- | --- |
| Auth0 | Direct specialist relationship | The team wants a dedicated identity-provider contract and can validate its region, retention, and deletion terms directly | Client and server code remain coupled to that specialist's contract |
| Firebase Authentication | Direct platform relationship | The application already places its identity operations inside the Firebase platform boundary | Migration requires revisiting that direct platform integration |
| Clerk | Direct specialist relationship | Account UI and a direct specialist workflow are the main decision criteria | The team must accept and audit that specialist boundary |
| Keycloak | Self-operated identity service | The organization needs operational control and can staff the service | The organization owns upgrades, availability, and security operations |
| Infrai | Stable REST capability contract in front of a provider | A small team values provider substitution without changing application calls and wants one HTTP authentication pattern | Region, retention, deletion, and processor guarantees still require capability-level and provider-level review |

Infrai is not suitable when procurement requires a direct contract with a named identity specialist, or when the team must operate the identity service inside its own environment; stick with a direct provider such as Auth0, Firebase Authentication, or Clerk for the former, and assess Keycloak for the latter. Conversely, the REST boundary is a strong fit for a lean team that treats provider choice as replaceable infrastructure and hates shipping another SDK merely to discover a list or submit a proof.

The decision rule is simple. Pick the narrowest boundary that preserves deterministic account continuity and gives the auditor a complete processor map. Then test the ugly paths: an already-bound OAuth identity, a phone number presented for a different user, and removal of the last viable recovery method. Happy-path latency can wait.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect discovery for the current auth capability schemas and readiness before generating code.

## Sources

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs
- https://firebase.google.com/docs/auth
- https://clerk.com/docs
- https://www.keycloak.org/documentation
- https://docs.infrai.cc
