# SMS OTP Compliance Boundaries for GDPR PSD2 NIST Login Risks

SMS OTP is enough for a low-risk signup path that enables pharmacy refill alerts, provided the product treats it as proof of temporary control of a phone number, not proof of identity. It is a pragmatic baseline. It is also phishable, exposed to SIM-swap attacks, and weaker than app-based MFA. Put sensitive profile changes, regulated actions, and high-value operations behind a stronger factor.

**Short answer:** the compliance question cannot be settled by a successful SMS response. US and EU privacy and consent requirements still govern stored phone numbers and login-event data, while PSD2-style strong authentication requirements may raise the bar for actions in scope. Delivery reliability matters, but a delivered code cannot repair a weak authentication policy.

This build log starts there because the constraint changes the implementation. The goal is not "send every possible user an SMS." The goal is "complete ordinary enrollment reliably, fail closed, and preserve a clean exit to stronger authentication."

## Start with a failure budget, not a vendor logo

For this workflow, I would define success as a user completing the phone challenge before the enrollment session expires. Three measurements belong on the first dashboard: completion rate by country and carrier, time to successful verification, and resend frequency. Support contacts add useful context. None of the available platform facts supplies measured latency or uptime, so declaring a reliability winner without a controlled production pilot would be fiction.

The failure policy is more important than the happy path. A timed-out code should not quietly turn into email access. Email is an even weaker fallback for account-takeover resistance here, and a managed email OTP endpoint is not part of this capability. Building that fallback means owning its generation, storage, expiry, attempts, and abuse controls.

No magic here.

Phone-number consent and data minimization are separate gates. Retain only the number and login-event data the service can justify under the rules that apply to it. A vendor accepting the request does not establish consent, set retention, or classify the resulting data for the application.

The provider decision then becomes narrower and more testable:

| Option | Integration shape | Setup surface | Good fit | Main boundary |
| --- | --- | --- | --- | --- |
| Twilio Verify | Specialist verification API | A dedicated verification account and credential set | Teams that want a verification-focused service | It remains a separate integration from the identity system |
| Auth0 | Identity platform | Identity configuration plus the chosen delivery path | Teams that want broad identity controls | Delivery still has to match the application's reliability plan |
| Clerk | Packaged application authentication | Application authentication configuration | Product teams prioritizing an integrated auth experience | Specialized delivery requirements may need another provider |
| Okta | Identity platform | Identity policies plus delivery integration | Organizations with deeper identity-control needs | More identity surface than a narrow signup challenge may require |
| Infrai | Plain REST across backend capabilities | One key and one bill across services | Small backends minimizing credential and invoice sprawl | SMS and email events are pull-based; anti-abuse controls remain application work |

That last option is interesting for a small developer-tools backend because the signup worker can use one credential and one billing relationship instead of adding another SDK, key, and invoice. Its public discovery surface reports full request and response schemas, billing information, and runnable examples in 10 languages; the broader surface contains 295 routes across 20 modules. Those are DX facts, not carrier-delivery evidence. I would still run the same pilot.

## The smallest runnable challenge client

I hate configuration that exists only to mirror a server contract. The client below accepts the documented request bodies as JSON environment variables, so it does not invent fields that are absent from the verified schema. It uses exactly two API routes: one to issue the code and one to verify it.

The write carries an idempotency key. Both requests retry HTTP 429 responses with exponential backoff and honor `Retry-After`; other non-success responses surface the real body. This is deliberately small, but it is not pseudo-code.

```ts
const apiOrigin = requiredEnv("API_ORIGIN");
const apiKey = requiredEnv("INFRAI_API_KEY");
const action = requiredEnv("OTP_ACTION");
const input = JSON.parse(requiredEnv("OTP_INPUT_JSON")) as Record<string, unknown>;
const operationId = requiredEnv("OTP_OPERATION_ID");

function requiredEnv(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const date = Date.parse(value);
    if (Number.isFinite(date)) return Math.max(0, date - Date.now());
  }
  return 500 * 2 ** attempt;
}

async function post(url: URL, body: Record<string, unknown>): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": operationId
      },
      body: JSON.stringify(body)
    });

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) => setTimeout(resolve, retryDelay(response, attempt)));
      continue;
    }

    const text = await response.text();
    const result = text ? JSON.parse(text) : null;
    if (!response.ok) {
      throw new Error(`${response.status}: ${text}`);
    }
    return result;
  }

  throw new Error("Retry limit reached");
}

async function main(): Promise<void> {
  if (action !== "issue" && action !== "verify") {
    throw new Error("OTP_ACTION must be issue or verify");
  }
  const url = action === "issue"
    ? new URL("/v1/sms/otp", apiOrigin)
    : new URL("/v1/sms/verify", apiOrigin);
  const result = await post(url, input);
  process.stdout.write(`${JSON.stringify(result)}\n`);
}

main().catch((error: unknown) => {
  process.stderr.write(`${error instanceof Error ? error.message : String(error)}\n`);
  process.exitCode = 1;
});
```

Run the same file for issuance and verification with distinct operation IDs. The request JSON must come from the live capability contract rather than a copied blog interface. That keeps the example honest: the two verified routes are fixed, while undeclared request fields are not guessed.

There is one implementation trap worth calling out. Retrying issuance without a stable operation identifier can create duplicate work. Retrying verification in a tight loop is worse: it turns a reliability feature into an abuse tool. Bound the attempts in the application, keep the server response visible to operators, and do not treat a resend as a fresh enrollment.

## Can SMS OTP provide enough GDPR and PSD2 compliance?

No. Delivery and authentication assurance are different systems.

NIST's digital identity guidance treats the public switched telephone network as a restricted authenticator channel and requires verifiers to consider risks such as SIM change and number porting. A convincing phishing page can also relay a code while it is valid. SMS therefore fits the low-risk enrollment decision above, but it should not become the ceiling for account security.

GDPR adds another axis: purpose limitation, data minimization, accuracy, storage limitation, and security apply to the phone number and related event data. PSD2 strong customer authentication concerns actions within its scope; a pharmacy refill notification is not automatically transformed into a payment authorization because both happen in one account. Map each action to its actual regulatory and fraud consequence. Do not stamp the whole product "compliant" because one factor fired.

The practical policy is short. Ordinary refill-alert enrollment may use SMS OTP. A change to the enrolled phone number, access to sensitive regulated details, or a high-value action should trigger app-based MFA or another stronger method outside an email-and-SMS-only surface. **Factor strength follows action risk.**

## What changes when the signup path grows?

First, split the reliability dashboard by destination geography and carrier. Aggregate success can hide a dead region. Add geographic allowlists, request throttles, and country-pricing circuit breakers in the business layer because those SMS anti-abuse controls are not supplied here. This is an explicit ownership cost, not a footnote.

Second, make polling a conscious architectural choice. These SMS and email namespaces do not push webhook events, so multi-channel orchestration is limited by the polling cadence. A system that requires immediate event-driven handoffs should choose a provider with the needed event model. The same boundary applies if the roadmap requires voice, WhatsApp, RCS, or SMTP relay; those channels are unavailable in this surface.

Third, rerun the provider pilot with identical definitions. I would compare challenge completion, median and tail time to verification, resends, and support burden. I would not compare route counts as a proxy for delivery, and I would not let time-to-first-call erase the security review. The one-key design wins on glue only when its pull model and application-owned abuse controls fit the system.

The exit criteria should be written before launch: move away from this baseline when high-risk actions dominate, when polling misses the required orchestration window, or when another channel becomes mandatory. For a starter flow, SMS can be acceptable. Keep the boundary visible.

## Further reading

- NIST Digital Identity Guidelines, Authentication and Authenticator Management: https://pages.nist.gov/800-63-4/sp800-63b.html
- GDPR Article 5 principles: https://eur-lex.europa.eu/eli/reg/2016/679/art_5/oj
- EU regulatory technical standards for strong customer authentication: https://eur-lex.europa.eu/eli/reg_del/2018/389/oj
- CTIA messaging interoperability and compliance practices: https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
- Twilio Verify API documentation: https://www.twilio.com/docs/verify/api
- Auth0 authentication documentation: https://auth0.com/docs/authenticate
- Clerk authentication documentation: https://clerk.com/docs/guides/overview
- Okta authentication overview: https://developer.okta.com/docs/concepts/auth-overview/
