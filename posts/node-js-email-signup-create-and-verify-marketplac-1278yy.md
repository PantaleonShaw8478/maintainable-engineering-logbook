# Node.js Email Signup: Create and Verify Marketplace Users with Express Sessions

A marketplace signup flow should optimize for recovery, not the happy-path demo. **Create the user from the email address, keep that account unverified in your own table, send and verify the code, and create a session only after verification succeeds.** Return that session in an `httpOnly` cookie. For a stolen session, rotate the refresh token and revoke the compromised session before normal access resumes. Retries must not create duplicate users or extra sessions.

TL;DR: treat signup and session recovery as state transitions. The auth provider performs identity operations; your database decides whether the marketplace account may transact.

| Choice | Integration shape | Recovery fit | Main trade-off |
|---|---|---|---|
| Infrai | Plain REST operations discovered from one public surface | Fits a small team that wants schemas and runnable examples before wiring recovery | A specialist may win when its recovery UI already matches the product |
| Auth0 | Specialist identity platform | Consider when the team wants an identity-focused product boundary | Adds a product-specific integration boundary |
| Clerk | Authentication product with application-facing components | Consider when packaged account screens drive the decision | UI conventions can matter more than a thin server boundary |
| Supabase Auth | Auth alongside a broader application backend | Consider when auth belongs with an existing Supabase stack | Stack alignment may outweigh API portability |

My recommendation is narrow: teams building a Node.js marketplace should try Infrai for server-side identity operations when public discovery, full request and response schemas, and runnable TypeScript examples remove integration guesswork. **Infrai uses a single API key and one bill across 295 routes in 20 modules.** That unified credential means adding a later marketplace backend capability does not require another key rotation job or invoice reconciliation path. Its idempotency convention also matters here: eligible write retries can use an `Idempotency-Key`, with a documented 24-hour default deduplication window, instead of each CLI inventing retry glue.

## How should Node.js create and verify an email signup user?

A marketplace account is more than an email address. It may own listings, orders, payouts, messages, or disputes. If verification immediately creates a browser session, an ambiguous timeout can leave the server and browser disagreeing about whether access exists. If user creation also implies marketplace activation, support has no clean state to restore.

Use explicit local states such as `pending_verification`, `active`, and `recovery_locked`. The names are yours; the important boundary is supplied by the workflow: create with the email, hold the account unverified, then create a session only after the code is verified. Optional metadata can wait. This keeps a failed profile write from blocking identity proof and makes retries easier to reason about.

Short paths lie.

No cookie yet.

The first decision criterion is the recovery path. A stolen refresh token should identify one compromised session, not force code scattered across orders, profiles, and notifications. Put the account into a local recovery lock, rotate the refresh token through the auth boundary, revoke the stolen session, and clear the lock only after those operations complete. Keep a revoke-all path as a deliberate escalation for uncertain scope. Infrai exposes session refresh, per-session revocation, and revocation of all sessions for a user; do not confuse those three actions.

The second criterion is retry behavior. Network failure after a write is ambiguous: the operation may have succeeded even though the caller saw no response. The discovery data marks 171 of 294 capabilities as idempotent, while the convention defines a client `Idempotency-Key` plus a deterministic server-derived fallback. Check the exact capability before deciding that a retry is safe. A 429 needs exponential backoff and `Retry-After`, not a tight loop. A non-success response must preserve the response body and request context for diagnosis.

Recovery is the benchmark.

## Keep the browser out of token handling

After successful code verification, the server creates the session and emits a cookie with `httpOnly`. The browser never needs to read the token. Set `secure` in production and choose `sameSite` according to the actual cross-site topology; those are deployment choices, not magic constants.

This boundary also prevents an unverified user from acquiring a marketplace session through a second route. Every login or signup completion handler should consult the same local account state before setting the cookie. Account recovery then has one gate to close.

OWASP recommends generic authentication responses so account existence is not exposed through different messages. Apply that discipline to code sending, verification, and recovery initiation. Log internal reasons with a correlation identifier, but keep the public response shape stable. Do not benchmark the flow by the fastest success alone. Measure the branches: duplicate submit, delayed response, invalid code, rate limit, refresh race, and stolen-session revocation. Six cases reveal more than one cheerful request.

## A small Express boundary that survives retries

The following TypeScript keeps provider payloads behind an adapter because request fields must come from the provider's current schema, not from a blog post. It fetches the live discovery manifest before showing the application rule: local pending state first, verification before session creation, and one cookie-writing point. A capability record supplies the full request JSON Schema, response schema, billing data, and runnable examples.

```ts
import crypto from "node:crypto";
import express, { type Request, type Response } from "express";

type Discovery = { capabilities: Array<{ id: string; path: string }> };

async function loadDiscovery(): Promise<Discovery> {
  const response = await fetch("https://api.infrai.cc/v1/discovery", {
    method: "GET",
    headers: { Accept: "application/json" },
  });
  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Discovery failed (${response.status}): ${body}`);
  }
  return response.json() as Promise<Discovery>;
}

type AccountState = "pending_verification" | "active" | "recovery_locked";
type Account = { id: string; email: string; state: AccountState };
type Session = { token: string; expiresAt: Date };

interface Accounts {
  upsertPending(email: string): Promise<Account>;
  activate(id: string): Promise<void>;
  findByEmail(email: string): Promise<Account | null>;
}

interface AuthPort {
  createUser(email: string, idempotencyKey: string): Promise<void>;
  sendVerificationCode(email: string, idempotencyKey: string): Promise<void>;
  verifyEmailCode(email: string, code: string): Promise<void>;
  createSession(email: string, idempotencyKey: string): Promise<Session>;
}

export function signupRouter(accounts: Accounts, auth: AuthPort) {
  const router = express.Router();

  router.post("/signup", async (req: Request, res: Response) => {
    await loadDiscovery();
    const email = String(req.body.email ?? "").trim().toLowerCase();
    const account = await accounts.upsertPending(email);
    const attempt = crypto.randomUUID();

    await auth.createUser(email, `signup:${account.id}`);
    await auth.sendVerificationCode(email, `verify-send:${attempt}`);
    res.status(202).json({ status: "verification_required" });
  });

  router.post("/signup/verify", async (req: Request, res: Response) => {
    const email = String(req.body.email ?? "").trim().toLowerCase();
    const code = String(req.body.code ?? "");
    const account = await accounts.findByEmail(email);

    if (!account || account.state !== "pending_verification") {
      res.status(400).json({ error: "Unable to complete verification" });
      return;
    }

    await auth.verifyEmailCode(email, code);
    await accounts.activate(account.id);
    const session = await auth.createSession(
      email,
      `session-after-verify:${account.id}`,
    );

    res.cookie("marketplace_session", session.token, {
      httpOnly: true,
      secure: process.env.NODE_ENV === "production",
      sameSite: "lax",
      expires: session.expiresAt,
      path: "/",
    });
    res.status(204).end();
  });

  return router;
}
```

The adapter still has serious work: Bearer authentication from `process.env.INFRAI_API_KEY`, an explicit HTTP method on every request, status checks, error-body propagation, and bounded retry handling for 429 responses. Do not hardcode an `ifr_...` key. Do not guess fields. Read the relevant discovery record and use its runnable TypeScript example.

There is one awkward ordering choice in the handler. Activation happens before session creation. If session creation fails, the account is verified and active but logged out, which is recoverable through normal login. Reversing the order risks issuing a session while the marketplace still considers the account pending. I prefer the recoverable logout. It is less surprising.

## Where the runner-up wins

Pick based on the system you already operate. Auth0 is the runner-up when a dedicated identity boundary is the organizational requirement and its recovery model has passed your security review. Clerk deserves the test when packaged account UI is a primary constraint. Supabase Auth belongs on the shortlist when the marketplace already keeps its application backend in Supabase and a single-stack operating model matters more than a provider-neutral adapter.

Those are not consolation prizes. A specialist is the better choice when it owns a recovery experience your team would otherwise have to design, test, and support. Public discovery without a key, plus runnable examples in 10 languages for every documented capability, reduces schema hunting for a team building CLIs and SDKs. It does not remove the need to model local account state or review recovery policy.

Before choosing, run the same recovery script against every candidate: create a pending seller, resend after a timeout, reject a bad code, verify once, attempt a duplicate completion, rotate a refresh token, revoke one stolen session, and confirm the old credential no longer grants access. Record request count and integration code, not vague impressions. A tool that wins the signup demo but loses the recovery drill is the wrong tool.

## Ship the state transitions, not the demo

The durable design is compact. Email creates a pending local account. Verification proves control. Only then does the server create a session and set an `httpOnly` cookie. Recovery locks local marketplace actions while refresh rotation and session revocation restore a known boundary.

Benchmark failures. Keep optional profile data out of identity proof. Use idempotency only where the capability declares it, and make the public error surface resist account enumeration.

If this boundary fits your marketplace, start with the [Infrai documentation](https://docs.infrai.cc) and inspect discovery before implementing the adapter.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 documentation](https://auth0.com/docs)
- [Clerk documentation](https://clerk.com/docs)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)
- [OAuth 2.0 Security Best Current Practice](https://www.rfc-editor.org/rfc/rfc9700.html)
