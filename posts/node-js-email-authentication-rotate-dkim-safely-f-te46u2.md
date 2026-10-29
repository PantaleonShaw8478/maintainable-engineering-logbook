# Node.js Email Authentication: Rotate DKIM Safely for Expiring Domain Reset Links

TL;DR: To rotate DKIM in a Node.js email domain, keep two selectors live for a controlled handoff. For a password-reset message whose link expires quickly, publish the new public key first, wait for DNS visibility, send with the new selector, then retire the old selector only after the longest plausible queue and retry window has passed. The least complex reliable design is a small, observable state machine around that handoff, not a cron job that swaps one environment variable. Keep it boring.

| Choice | Delivery risk | Operational burden | Use it when |
| --- | --- | --- | --- |
| Replace one selector in place | High | Low at first | Almost never |
| Overlap old and new selectors | Lower | Moderate | Normal production rotation |
| Delegate signing to a managed mail edge | Depends on the contract | Low in application code | The team can verify key, DNS, and incident controls externally |

My decision rule is blunt: if a reset email can be delayed long enough for its link to expire, preserve verification during the delay. A two-selector overlap gives queued messages a chance to authenticate while new messages use the replacement key. It also gives the operator a rollback that does not require rebuilding the Node.js process.

## How should Node.js rotate DKIM for email domain authentication?

DKIM is a signature over selected message data, verified with a public key found through DNS. The selector is part of the lookup name, so changing selectors creates a new verification path. SPF answers a different question: whether a sending IP or designated mechanism is permitted for the envelope domain. DMARC evaluates alignment between the authenticated identifiers and the visible From domain. These checks are related, but a passing SPF record does not repair a broken DKIM lookup.

For the reset workflow, four invariants matter:

- Every production message has a signing selector whose public key is already published.
- The selector in the DKIM-Signature matches the key record being monitored.
- The visible From domain and the authenticated domain satisfy the alignment policy.
- The reset URL expires independently of mail delivery time and is rejected after use.

The last item is easy to miss. Deliverability cannot make an expired token valid. Keep the token lifetime short, but do not use that lifetime as the DNS retirement timer. DNS caches and mail retries have different clocks.

## Why is replacement safer than a one-line swap?

A one-line swap couples three events: publishing a key, signing with it, and removing the old key. That is a brittle change boundary. A resolver may still have an old answer, a receiving system may retry later, and a queued message may carry the previous selector. Removing the old record immediately turns a temporary propagation difference into an authentication failure.

The safer sequence is staged:

1. Generate a new key pair in a restricted signing boundary. Store the private key outside the repository and record its selector, creation time, and owner.
2. Publish the new selector's TXT record. Confirm that authoritative DNS servers return the expected public key.
3. Start signing new reset messages with the new selector while retaining the old public key.
4. Watch DKIM results, DMARC alignment, bounce categories, and reset completion for the full retirement window.
5. Remove the old public key only after the window has ended and the change record has a concrete verification result.

The window is a policy value, not a magic number. Set it from your maximum queue delay, retry policy, DNS TTL, and incident procedure. A 10-minute TTL is not proof that every resolver refreshes in 10 minutes. This is where a benchmark helps: measure lookup visibility from several recursive resolvers and compare it with the message retry horizon. That is the limitation of a TTL-only plan.

## A small Node.js boundary keeps the risky state visible

The application should receive a signer, a selector, and a domain as explicit inputs. It should not discover rotation state by reading whichever secret happens to be mounted. This example uses a generic mail transport interface so the rotation policy remains testable without binding the article to a provider.

```ts
type ResetMail = {
  to: string;
  token: string;
  expiresAt: Date;
};

type MailTransport = {
  send(message: {
    from: string;
    to: string;
    subject: string;
    text: string;
    dkim: { domainName: string; selector: string; privateKey: string };
  }): Promise<{ messageId: string }>;
};

export async function sendPasswordReset(
  transport: MailTransport,
  input: ResetMail,
  signing: { domainName: string; selector: string; privateKey: string },
): Promise<string> {
  const remainingMs = input.expiresAt.getTime() - Date.now();
  if (remainingMs <= 0) throw new Error("reset token already expired");

  const result = await transport.send({
    from: `Security <no-reply@${signing.domainName}>`,
    to: input.to,
    subject: "Reset your password",
    text: `Use this link before ${input.expiresAt.toISOString()}: /reset/${input.token}`,
    dkim: signing,
  });

  return result.messageId;
}
```

The code does not decide when a key is retired. That belongs to deployment configuration and an operator-visible change record. Test the boundary with a fake transport that captures the selector, then test the received message through a real DKIM verifier in staging. A unit test that only checks a resolved promise proves almost nothing about authentication.

A useful production metric is the ratio of reset messages with a DKIM pass to total reset messages, segmented by selector and receiving domain. Add message ID, selector, signing-domain, and send timestamp to structured logs. Do not log the reset token. Alert on a selector disappearing from successful traffic before the retirement event, and on DMARC failures after the new selector becomes active.

## When should the runner-up win?

The two-key overlap is the default for a team that owns DNS and signing. A delegated mail edge can be the better operational choice when the application team cannot safely hold private keys or cannot operate DNS changes during an incident. That choice moves the boundary; it does not remove the checklist. Require documented selector ownership, key-roll timing, alignment behavior, event visibility, and a tested exit path.

Direct SMTP is a different runner-up when the system needs full control over queueing and retries. It also creates more work: connection handling, reputation controls, bounce processing, TLS policy, and delivery telemetry become your problem. An HTTP mail API can reduce transport code, but its abstraction may hide the exact DKIM selector or retry state needed for a forensic check. Compare those boundaries with a test message and captured headers, not a feature grid.

For a short-lived reset, I would choose the path with the clearest evidence after a failed delivery. The winning setup is the one where an engineer can answer, in one trace, which selector signed the message, which DNS answer was expected, whether alignment passed, and whether the token was still valid. The trade-off is operational effort: overlap needs a retirement record, while a one-line swap is faster only until it fails.

## Further reading

- RFC 6376, DomainKeys Identified Mail (DKIM) Signatures: https://datatracker.ietf.org/doc/html/rfc6376
- RFC 7208, Sender Policy Framework (SPF): https://datatracker.ietf.org/doc/html/rfc7208
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- Node.js Crypto API: https://nodejs.org/api/crypto.html
- MDN, DNS record types: https://developer.mozilla.org/en-US/docs/Glossary/DNS_record_types
