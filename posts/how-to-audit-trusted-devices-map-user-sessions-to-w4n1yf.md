# How to Audit Trusted Devices — Map User Sessions to OTP Revocation Controls

Short answer: keep a server-side device ledger, map every session to its device record, and make revocation a state transition that the token verifier checks on every refresh. This is the least complex way to migrate phone one-time-code login away from a managed provider without losing an audit trail.

| Option | Strength | Cost | Pick it when |
| --- | --- | --- | --- |
| Device ledger plus rotating refresh tokens | Clear ownership and precise revocation | You operate a small auth data model | A fintech app needs per-device logout |
| Stateless signed sessions | Minimal storage and low latency | Revocation is coarse or delayed | Sessions are short and risk is low |
| Existing identity platform | Mature recovery and policy tooling | Migration and policy coupling | Compliance scope outweighs control |

I recommend the first row for a migration. It keeps the provider boundary narrow: the phone code proves possession, while your application owns device trust, session lifetime, and the evidence needed by support.

## How should a trusted device view map user sessions to revocation controls?

Start with two identifiers that are deliberately boring. `device_id` represents a remembered browser or handset, and `session_id` represents one login. A device can have several sessions; a session must point to exactly one device. Store a hash of the refresh-token family, never the token itself. The view shown to a user can then expose a label, last-used time, approximate location, and a “revoke” action without exposing secrets.

The important distinction is trust versus activity. A trusted device may have no active sessions, and an active session may be untrusted after a policy change. Treat those as separate fields so a support agent can answer “what is signed in?” and “what can sign in again?” with different queries.

Keep it boring.

The ledger needs an append-only event stream beside its current-state tables. Record `otp_verified`, `session_issued`, `refresh_rotated`, `session_revoked`, and `device_untrusted`, each with an actor and reason. I initially wanted one `revoked_at` column to do everything. That made incident review impossible: a second revoke erased who performed the first one. Current state is for enforcement; events are for reconstruction.

## A small data model that survives provider migration

Use database constraints to make invalid mappings hard to create. A simplified PostgreSQL shape looks like this:

```ts
type Device = {
  userId: string;
  deviceId: string;
  trustedAt: string | null;
  untrustedAt: string | null;
  lastSeenAt: string;
};

type Session = {
  sessionId: string;
  userId: string;
  deviceId: string;
  refreshFamilyHash: string;
  issuedAt: string;
  expiresAt: string;
  revokedAt: string | null;
  revokeReason: string | null;
};
```

Enforce a foreign key from `session.device_id` to `device.device_id`, scoped by `user_id`. Add a unique active-family constraint if rotation is your policy. Do not infer a device from a user-agent string at read time; user agents change, and two people can share one. Generate the device identifier after the OTP challenge succeeds, then bind it to a server-held cookie or platform credential with an explicit expiry.

## Implementing refresh checks without a revocation race

The refresh path is where a pretty device page becomes a security control. Verify the refresh token signature and family, lock the session row, and rotate the family in one transaction. If `revoked_at` or `expires_at` is set, return the same generic authentication failure as any invalid refresh. Avoid telling an attacker whether a device was revoked.

```ts
type RefreshInput = { sessionId: string; presentedFamilyHash: string };
type RefreshResult = { ok: true; newFamilyHash: string } | { ok: false };

async function refresh(input: RefreshInput, now: Date): Promise<RefreshResult> {
  return db.transaction(async (tx) => {
    const session = await tx.sessions.lockById(input.sessionId);
    if (!session || session.revokedAt || session.expiresAt <= now.toISOString()) {
      return { ok: false };
    }
    if (session.refreshFamilyHash !== input.presentedFamilyHash) {
      await tx.sessions.revoke(input.sessionId, "rotation_reuse");
      await tx.events.append({ type: "session_revoked", sessionId: input.sessionId, reason: "rotation_reuse" });
      return { ok: false };
    }
    const next = randomFamilyHash();
    await tx.sessions.rotate(input.sessionId, next, now.toISOString());
    await tx.events.append({ type: "refresh_rotated", sessionId: input.sessionId });
    return { ok: true, newFamilyHash: next };
  });
}
```

Test the ugly cases, not just the happy login: two refreshes arriving together, a revoke racing a refresh, a device untrusted between OTP and token issuance, and a replayed family hash. In a real migration test I would run those cases with a barrier that pauses both transactions after the row lock request, release the barrier in opposite orders, then assert that exactly one new family hash exists and that the losing request cannot mint a token; the event sequence should contain the winning rotation and either a revoke or a generic rejection, which gives incident response something concrete to inspect instead of a vague “logout happened” flag. The expected result is deterministic: one transaction wins, the other receives a generic failure, and the event log explains why. Your mileage may vary with transaction isolation, so verify the behavior against the database you actually run.

That is the control.

## What should the device view expose, and where is it unsuitable?

Show the minimum useful evidence: device label, first-seen and last-seen timestamps, session count, and a revoke-all action. Round IP-derived location to a broad region and state that it is approximate. Give the account owner a confirmation step for “revoke this device,” then invalidate every session mapped to that device in one transaction. Keep an internal actor field for support actions; customer-facing text does not need staff email addresses.

The catch is operational ownership. A ledger is unsuitable when the team cannot provide encrypted storage, key rotation, alerting, and a recovery path for locked-out users. In that case, stick with the managed identity platform until those controls exist; a half-migrated verifier is worse than a clear dependency. The ledger also cannot replace phishing-resistant MFA for high-risk transfers. Phone OTP is a migration step, not a universal assurance level.

Measure the migration with boring numbers: median time from OTP success to first API call, refresh rejection rate, revoke propagation latency, and the percentage of sessions with a device mapping. Alert on unmapped sessions and sudden `rotation_reuse` events. Keep the old provider's subject identifier as an immutable external reference while new sessions use your internal user ID. That lets you backfill devices without changing account ownership.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://www.rfc-editor.org/rfc/rfc9449
- https://www.rfc-editor.org/rfc/rfc6819
