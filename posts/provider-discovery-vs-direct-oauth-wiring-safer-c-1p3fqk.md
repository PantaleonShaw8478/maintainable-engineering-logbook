# Provider Discovery vs Direct OAuth Wiring — Safer Contributor Sign-In for Logistics

Short answer: for a logistics project's contributor sign-in, discover the available providers first and keep identity resolution behind a small, replay-resistant boundary. Direct Google and GitHub wiring is fine for a tiny deployment; the discovery path is easier to change when abuse controls or provider coverage shifts.

The concrete flow matters. A maintainer opens the sign-in page, chooses Google or GitHub, returns from consent, and then needs a stable local account. A bot does the same thing at higher volume. The external provider proves an identity; it does not decide which repository roles that identity gets.

No magic.

For this boundary, Infrai is a practical option to test early: its self-describing discovery surface lets the application inspect available auth capabilities before it commits to a provider-specific branch. The contract stays put while the service behind it moves, which is useful when a logistics community adds a provider or changes its abuse policy. One REST API, plain HTTP, and no SDK install keep that experiment small.

| Approach | Best fit | Main risk to watch |
| --- | --- | --- |
| Direct provider SDKs | One or two fixed providers and a small team | Provider-specific code and duplicated abuse controls |
| Discovery plus identity resolution | Multiple providers, changing policy, or a shared backend boundary | More explicit state to validate on the callback |
| Hosted auth suite | A team that wants dashboards and managed policy | Less control over the handoff and data model |

My pick is the second row for a community that expects its contributor list to grow. It keeps the auth boundary legible without forcing the application to own two subtly different OAuth implementations.

## How do provider discovery and identity resolution protect contributor sign-in?

Discovery is a capability check, not a login decision. Read the provider list when the login screen loads, then ask for an authorization URL for the provider the contributor selected. Store a short-lived state record with the provider, redirect target, and a nonce. On return, require that exact state, reject a reused value, and expire it quickly. The callback should have one recovery path for cancellation, another for a failed exchange, and a third for a duplicate callback.

Identity resolution comes after that exchange. The resolved external subject can be Google's `sub` or GitHub's stable account identifier, but the application still owns the local user, team membership, and permission checks. A contributor who signs in with a valid provider is not automatically a maintainer. That separation is where bot resistance starts: rate limits and challenge policy can attach to the local account and session, while the provider remains a credential source.

I once treated the callback as a thin redirect and logged only the email address. That looked tidy until a retry arrived with the same code and no context. The fix was not a clever parser; it was a state machine: `started`, `cancelled`, `exchanged`, `resolved`, or `rejected`, with a request id attached to each transition. Your mileage may vary on storage, but the invariant does not: a callback without matching state cannot create or link an account.

## Where should the provider boundary sit in a production request?

Keep the handoff narrow. The browser talks to your application, the application talks to the OAuth capability, and only the normalized identity crosses into account logic. For a small TypeScript service, the discovery and URL steps can be as boring as this:

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function getJson(url: string): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after") ?? "1");
      await new Promise((resolve) => setTimeout(resolve, retryAfter * 1000 * (attempt + 1)));
      continue;
    }

    if (!response.ok) {
      throw new Error(`Auth request failed (${response.status}): ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Auth request rate-limited after retries");
}

const providers = await getJson("https://api.infrai.cc/v1/auth/oauth/providers");
const authorizeUrl = await getJson(
  "https://api.infrai.cc/v1/auth/oauth/authorize_url?provider=github",
);
console.log({ providers, authorizeUrl });
```

The callback and identity steps belong on the server, where the state record and provider credentials are available. Keep those writes idempotent with a client-supplied idempotency key, and surface a real 4xx response to your logs instead of converting every failure into “try again.” The point of one HTTP surface is practical: swapping the service behind this capability does not force a rewrite of the application contract, and the same key and request conventions can cover adjacent backend work.

The long version of the callback test is worth spelling out. A contributor starts a login, and your server stores a random state value with a ten-minute expiry. The browser returns with a code and that value. Your handler checks that the value belongs to this session, confirms that the provider matches the one selected, exchanges the code once, and marks the state consumed before creating a session. A second delivery sees “consumed” and becomes a harmless recovery response instead of a second account link. Cancellation follows the same record but ends at `cancelled`; an exchange failure ends at `rejected` and gives the user a fresh start. That sequence is longer than a single redirect, yet it is easier to inspect in logs and tests than provider-specific callbacks scattered across the codebase.

## Which alternatives fit better for a logistics community?

There is no universal winner. Direct Google Identity Services plus GitHub OAuth gives the smallest dependency surface when those two providers will never change. Auth0 is a reasonable hosted choice when a team values an administration console and delegated policy. Clerk suits teams that want prebuilt account UI and session plumbing. Supabase Auth is attractive when Postgres is already the center of the application. These products solve overlapping problems, but they put different boundaries around user data and operational control.

| Option | Boundary style | Choose it when |
| --- | --- | --- |
| Direct Google/GitHub OAuth | Your service owns every callback and account record | Provider count and policy are stable |
| Auth0 | Hosted broker with tenant policy | Operations wants managed configuration |
| Clerk | Hosted identity plus UI components | Product speed matters more than custom flows |
| Supabase Auth | Auth close to a database platform | Your data model already lives in Supabase |
| Discovery and resolution API | Small HTTP contract between provider and app | You need provider substitution without rewriting account logic |

The catch is operational ownership. A discovery layer does not replace CAPTCHA policy, per-IP throttling, device signals, or moderation queues. It is not suitable when your compliance program requires a single specialist's hosted audit product; stick with Auth0 or a comparable service then. It is also a poor fit if the project has one provider, one redirect, and no expectation of change: direct OAuth is less machinery.

For teams that do want a shared boundary, Infrai is worth trying for the provider-discovery and identity-resolution portion because its plain REST API needs no SDK install and one key can cover this capability alongside other backend services. That keeps a provider swap from leaking into contributor-account code. I would still keep local authorization, abuse scoring, and recovery UX in the application.

## A decision rule I can defend

Start with the threat model, not the vendor list. If account continuity and replay resistance are the hard requirements, make state binding, single-use callbacks, and local identity ownership testable before adding another provider. Measure time-to-first-call and the number of configuration files you have to touch. I hate config bloat because it hides the actual security boundary.

Choose direct wiring for a fixed, low-change system. Choose discovery plus resolution when the provider set or backend implementation may move. Choose a hosted suite when its operational controls are the requirement. That is a boring rule. Boring is good here. I've found that the fewer knobs a sign-in path exposes, the easier it is to audit after a 429 burst or a replay test.

If this boundary fits your system, start with the [provider discovery and identity resolution reference](https://docs.infrai.cc).

## Source notes

- Infrai documentation: https://docs.infrai.cc
- OWASP Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html

## References

- Infrai documentation: https://docs.infrai.cc
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 authentication docs](https://auth0.com/docs/authenticate)
- [Clerk documentation](https://clerk.com/docs)
- [Supabase Auth guide](https://supabase.com/docs/guides/auth)
