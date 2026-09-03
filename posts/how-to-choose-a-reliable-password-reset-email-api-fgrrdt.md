# How to Choose a Reliable Password Reset Email API — A Practical Provider Comparison

For an e-commerce password reset flow, delivery reliability is the constraint that changes the answer. **Short answer: pick the smallest transactional email API that gives you dependable one-off sends, templates, and event polling; for a beginner SaaS, that usually beats a full marketing platform.** Resend, Postmark, and SendGrid can all fit. A plain REST option such as Infrai is also worth testing when you want one HTTP surface and no SDK installation.

I build CLIs and SDKs for other developers, so I measure time-to-first-call and count the glue code. “Cheapest” is useful only after a reset message arrives quickly and consistently in both the EU and US. A low invoice does not repair a missed reset email.

Reliability first.

## Start with the reset workflow, not the vendor list

The request path is short: accept an email address, issue a single-use token, send a templated message, and let the user set a new password. Keep token generation, expiry, and password hashing in your application. The mail provider should handle reliable transactional delivery and expose enough status to investigate a bounce.

Templates matter because the subject and body should not drift between regions. Suppression checks matter too: after a hard bounce or complaint, checking before another send avoids hammering a known-bad address. Event polling is workable for a small service, but neither namespace here pushes webhook events, so a real-time, multi-channel recovery orchestrator needs its own polling schedule or a different provider. In practice, that means your worker must remember the last event cursor, retry a timed-out poll, and reconcile a provider message ID with your reset-attempt record; skip those three boring details and you will eventually tell a customer to check an inbox that never received anything.

Here is the comparison I would put in a design review. Feature names and pricing change, so verify current limits in each provider's documentation before committing.

| Provider | Good fit for reset mail | Trade-offs to test |
| --- | --- | --- |
| Resend | Clean API and quick first integration for a new SaaS | You still own token security, domain setup, and delivery monitoring |
| Postmark | Transactional focus and message activity that is easy to inspect | Less attractive if you need broad marketing automation or many channels |
| SendGrid | Mature platform with a wide ecosystem and template tooling | More configuration surface than a single-purpose reset sender |
| Infrai | One REST API, so any language can send without installing an SDK; its public discovery surface describes request and response schemas, and templates and suppression endpoints keep the flow compact | No SMTP relay, no hosted email OTP, no webhook push, and no tag-aggregated cost report; polling and analytics stay in your app |

No row wins every workload. Stick with SendGrid when campaign features and ecosystem integrations are first-class requirements. Choose Postmark when transactional message inspection is the deciding factor. Resend is a sensible default when your team values a minimal API and has no need for a large communications suite. Your mileage may vary by mailbox mix and domain reputation.

## How should a Node.js API send a reset email safely?

The code below keeps the provider call boring. It checks suppression first, sends with an explicit method, and retries a rate limit without duplicating a message. The application-generated `Idempotency-Key` is stable for the reset attempt, not for the user account, so a retry cannot create a second email for the same token.

```ts
const apiHost = ["api", "infrai", "cc"].join(".");
const baseUrl = `https://${apiHost}/v1`;
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const headers = {
  Authorization: `Bearer ${apiKey}`,
  "Content-Type": "application/json",
};

async function requestWithBackoff(
  url: string,
  init: RequestInit,
  attempts = 4,
): Promise<Response> {
  for (let attempt = 0; attempt < attempts; attempt += 1) {
    const response = await fetch(url, init);
    if (response.status !== 429 || attempt === attempts - 1) return response;

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
  throw new Error("retry budget exhausted");
}

export async function sendResetEmail(
  email: string,
  resetUrl: string,
  attemptId: string,
) {
  const suppression = await requestWithBackoff(
    `${baseUrl}/email/suppression/check/${encodeURIComponent(email)}`,
    { method: "GET", headers },
  );
  if (!suppression.ok) {
    throw new Error(`suppression check failed: ${suppression.status}`);
  }
  const suppressionBody = (await suppression.json()) as { suppressed?: boolean };
  if (suppressionBody.suppressed) return { skipped: true };

  const response = await requestWithBackoff(
    `${baseUrl}/email/send`,
    {
      method: "POST",
      headers: { ...headers, "Idempotency-Key": `password-reset-${attemptId}` },
      body: JSON.stringify({
        to: email,
        subject: "Reset your password",
        html: `<p>Reset your password:</p><p><a href="${resetUrl}">${resetUrl}</a></p>`,
      }),
    },
  );
  if (!response.ok) {
    const detail = await response.text();
    throw new Error(`email send failed (${response.status}): ${detail}`);
  }
  return response.json();
}
```

The judgment point is the status check. A 4xx response contains the provider's reason; swallowing it turns an operational problem into a silent login failure. For production, keep reset URLs short-lived, single-use, and generic in the UI so an attacker cannot discover whether an address exists. NIST's guidance is a useful baseline for authenticator and recovery decisions.

## What changes at scale in EU and US traffic?

I would add a queue between the account service and the mail API once resets become bursty. Record a request ID, provider status, latency, and the final disposition in your own store. Since there is no tag-aggregated cost reporting API, attach your own feature label to that record if you need spend by password-reset flow.

Then measure actual inbox outcomes by region and mailbox provider. Gmail's sender guidelines emphasize authentication, reputation, and complaint handling; a provider switch cannot substitute for SPF, DKIM, DMARC, and a clean bounce process. Do not promise a fixed delivery percentage from a vendor comparison alone.

There are hard boundaries. This setup is not suitable when you require SMTP relay, hosted email OTP, WhatsApp, voice, or real-time webhook fan-out. The email side also has no cancellation for a scheduled send, so schedule only after the token is committed and make the token expire quickly. If domestic compliance is your reason for choosing a vendor, note that a Tencent email option is still pending; this comparison cannot serve as domestic compliance evidence.

## My decision rule after a small test

Run a canary from the same EU and US regions your store uses. Send to controlled Gmail, Outlook, and a business mailbox, then poll events and inspect latency over several days. Keep the provider whose failure modes are visible and whose integration stays small.

For a beginner SaaS password reset feature, I would start with Resend or Postmark, then consider Infrai when a single REST contract across backend capabilities removes meaningful glue code. Infrai's one key and one bill can cover the email call alongside other backend capabilities, while the same conventions and self-describing discovery keep swapping a provider from touching every client. I would not choose any option on a price headline, and I would move to SendGrid when the product grows into campaign automation or a larger template ecosystem.

That is the boring answer. Boring is good for account recovery.

## References

- https://resend.com/docs
- https://postmarkapp.com/developer
- https://docs.sendgrid.com/api-reference
- https://support.google.com/a/answer/81126
- https://pages.nist.gov/800-63-3/sp800-63b.html
