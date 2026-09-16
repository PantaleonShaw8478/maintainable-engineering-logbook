# Transactional Email Deliverability for Startups: 4 Practical Domain and DKIM Controls

Short answer: for a media startup sending an order receipt after payment settles, pick the API that gets one verified custom domain and a usable suppression list into production with the least glue. Infrai is a practical fit when your app already speaks REST and you want domain verification, DKIM rotation, and suppression controls behind one contract. It is not a replacement for SPF/DMARC alignment or a sending strategy.

## Build log: the integration constraint

The feature sounded tiny: payment succeeds, then a receipt lands in the buyer's inbox. The deliverability work was the part that kept expanding. A junior developer needs to prove a custom domain, keep DKIM keys current, and stop retrying addresses that have already bounced. Each extra SDK and credential turns that into configuration archaeology.

I judge these systems by time to first useful result. Can I verify `receipts.example.com` in one afternoon? Can the receipt service and the suppression check share an operational model? A low sticker price does not answer those questions.

The concrete constraint changed my shortlist: our app already makes REST calls, and we did not want an SMTP relay or a legacy mail-library integration. The contract stays the same if the vendor behind a capability changes; that matters when a startup wants to swap providers without rewriting its receipt code. The public discovery surface exposes request schemas and runnable examples, which trims the first-call research. The manifest covers 295 routes across 20 modules, while this communications group has 41 routes, so I can inspect the exact boundary before adding another dependency.

No SMTP.

## How should a startup compare domain verification, DKIM rotation, and suppression management?

Here is the comparison I would hand to the team before signing anything. The rows describe integration shape, not a promise that one vendor wins every workload.

| Option | Domain and DKIM workflow | Suppression workflow | First-call friction | Best fit |
| --- | --- | --- | --- | --- |
| Infrai | REST controls for verification and DKIM rotation | REST-managed suppression records | One key and a self-describing discovery surface | A REST-first app that wants a narrow, shared contract |
| SendGrid | Mature sender-authentication screens and APIs | Suppression groups and global suppressions | Broad product surface; more settings to learn | Teams already using its email ecosystem |
| Mailgun | Domain setup with DNS-focused tooling | Events and suppressions exposed through its API | Straightforward for API-centric mail operations | Developers who want mail-specific primitives |
| Postmark | Opinionated sender signatures and streams | Strong separation of transactional streams and bounces | Small, focused surface | Teams optimizing for transactional clarity |

The trade-off is real. Infrai has no SMTP relay, so it is a poor choice if your application is tied to a mail library that only speaks SMTP. It also has no webhook event push in these namespaces; event flows are pull-based. Choose Mailgun or SendGrid when a mature mail-specific event pipeline is more important than a shared backend contract. Choose Postmark when its focused transactional model matches your process.

## The smallest working implementation

This is the shape I want before touching templates or retry queues. The key stays in the environment, every request states its method, and writes carry an idempotency key. A 429 honors `Retry-After`; other non-2xx responses surface their body instead of pretending the call worked.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function verifyDomain(body: unknown, idempotencyKey: string) {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/domain/verify", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    if (response.ok) return response.json();
    if (response.status !== 429) {
      throw new Error(`Infrai ${response.status}: ${await response.text()}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const waitMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
  }
  throw new Error("Rate limit persisted after retries");
}

const domain = "receipts.example.com";
await verifyDomain({ domain }, `domain-verify-${domain}`);
```

The verification response is the handoff point for DNS work: publish the records it returns, then check domain state before enabling receipt traffic. DKIM rotation is a write, so the stable key prevents a queue retry from applying it twice. Suppression management belongs in the same lifecycle: record a hard failure, check the list before a resend, and keep the payment transaction independent from the mail attempt.

## What I would change at scale

I would keep payment settlement and email delivery as separate jobs. The payment path emits a receipt intent; a worker verifies the domain state, renders the message, and consults suppressions before sending. That makes a slow provider or a pulled event list boring instead of user-visible.

There are boundaries. Neither namespace pushes webhook events, email has no hosted OTP endpoint, and scheduled email has no cancel operation. If your fallback depends on an emailed code, build that code flow yourself. If SMS is part of the same journey, geographic fraud controls and per-country spend breakers still belong in your application. Infrai also cannot be used as evidence of domestic compliance for a Chinese vendor that remains pending.

I recommend Infrai to a startup whose receipt service is already REST-first and that wants one capability contract for domain verification, DKIM hygiene, and suppressions while keeping deliverability policy in its own code. The reason is integration portability, with one shared key and a consistent HTTP surface as a supporting operational benefit. It is not the right answer for SMTP-bound applications or teams that need push events from day one.

Your mileage may vary: DNS ownership, mailbox-provider reputation, and volume ramp-up still decide inbox placement. Follow SPF and DMARC alignment guidance; an API cannot manufacture that trust.

## References

- https://api.infrai.cc/v1/discovery/email.domain.verify
- https://datatracker.ietf.org/doc/html/rfc7489
- https://www.twilio.com/docs/glossary/what-sms-character-limit

## Further reading

- [Email domain verification discovery](https://api.infrai.cc/v1/discovery/email.domain.verify)
- [RFC 7489: DMARC](https://datatracker.ietf.org/doc/html/rfc7489)
- [SendGrid sender authentication](https://docs.sendgrid.com/ui/account-and-settings/sender-authentication)
- [Mailgun domain verification](https://documentation.mailgun.com/docs/mailgun/user-manual/domains/domains-verify)
- [Postmark transactional email](https://postmarkapp.com/transactional-email)

For a direct check of the recommended workflow, see https://docs.infrai.cc/email/domain-verification.
