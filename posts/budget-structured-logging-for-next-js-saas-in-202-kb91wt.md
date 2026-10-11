# Budget Structured Logging for Next.js SaaS in 2026 — Platform Comparison

For a Next.js SaaS rolling out a pricing rule behind a flag, start with boring JSON logs and a stable application-owned sink. Record the flag, rule version, decision inputs, request correlation IDs, and outcome together. Then choose the backend by the investigation you need: a log API is enough for server-side incident reconstruction; Sentry is the better-shaped option when browser debugging, source maps, crash symbolication, or session replay are central; Axiom and Better Stack Logtail deserve evaluation when the log exploration workflow itself carries the load.

**TL;DR:** own the event contract. Keep the storage vendor behind it. Infrai can fit a small server-heavy stack that values one REST contract and low integration overhead, but it is not a replacement for tracing, replay, alert delivery, uptime checks, or user-level deletion controls. Those boundaries matter more than a pricing grid.

## Which budget structured logging platform should a Next.js SaaS use?

The useful question is not “did checkout throw?” It is “why did order `ord_7F3` receive rule `margin-floor-v3` while the flag was at a 10% rollout?” A stack trace alone cannot answer that. The reconstruction needs the exact decision context that existed at the time.

For this rollout I would require a compact event with `event`, `occurred_at`, `request_id`, `trace_id`, `span_id`, `order_id`, `pricing_rule`, `rule_version`, `flag_key`, `flag_enabled`, `rollout_bucket`, and `outcome`. Do not log names, email addresses, addresses, payment details, or raw auth tokens. That is not vague privacy hygiene: if the logging service cannot delete records by user, putting personal data into the event creates a remediation problem under erasure requests.

Cardinality is the trap. `pricing_rule` and `outcome` are bounded dimensions. `order_id` and `request_id` are lookup fields, not dimensions I would casually aggregate. Prometheus makes the same cardinality warning for labels; the storage engine may differ, but the operational failure mode is familiar.

One more constraint changed the choice: correlation IDs in logs are not distributed tracing. They help join evidence manually. They do not produce a span tree or a distributed-trace query layer. That distinction is easy to wave away during a trial and painful during a cross-service incident, when a developer expects to click from the checkout request into the pricing worker and instead has to copy an ID into another search. If that click path is required, stop evaluating a logs-only setup and test a tracing product.

Logs are evidence, not a time machine.

## The smallest working Next.js implementation

I want one typed boundary, zero logging SDKs in business logic, and an event I can benchmark as a single serialized line. The default sink below writes JSON to standard output, so it runs in a Next.js server action, route handler, or background worker. A hosted transport can replace that sink without changing the pricing function.

```ts
type PricingOutcome = "applied" | "skipped" | "rejected";

type PricingDecisionLog = {
  event: "pricing_rule_evaluated";
  occurred_at: string;
  request_id: string;
  trace_id: string;
  span_id: string;
  order_id: string;
  pricing_rule: string;
  rule_version: number;
  flag_key: string;
  flag_enabled: boolean;
  rollout_bucket: number;
  outcome: PricingOutcome;
};

interface LogSink {
  write(event: PricingDecisionLog): Promise<void>;
}

class HostedLogSink implements LogSink {
  constructor(
    private readonly baseUrl = process.env.INFRAI_BASE_URL,
    private readonly apiKey = process.env.INFRAI_API_KEY,
  ) {}

  async write(event: PricingDecisionLog): Promise<void> {
    if (!this.baseUrl || !this.apiKey) {
      throw new Error("INFRAI_BASE_URL and INFRAI_API_KEY are required");
    }

    for (let attempt = 0; attempt < 4; attempt += 1) {
      const response = await fetch(`${this.baseUrl}/v1/logs/ingest`, {
        method: "POST",
        headers: {
          Authorization: `Bearer ${this.apiKey}`,
          "Content-Type": "application/json",
          "Idempotency-Key": `${event.request_id}:${event.event}`,
        },
        body: JSON.stringify({ logs: [event] }),
      });

      if (response.ok) return;

      const body = await response.text();
      if (response.status !== 429 || attempt === 3) {
        throw new Error(`Log ingest failed (${response.status}): ${body}`);
      }

      const retryAfter = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
    }
  }
}

type PricingInput = {
  requestId: string;
  traceId: string;
  spanId: string;
  orderId: string;
  subtotalCents: number;
  rolloutBucket: number;
};

export async function applyPricingRule(
  input: PricingInput,
  sink: LogSink = new HostedLogSink(),
): Promise<number> {
  const flagEnabled = input.rolloutBucket < 10;
  const outcome: PricingOutcome = flagEnabled ? "applied" : "skipped";
  const totalCents = flagEnabled
    ? Math.max(input.subtotalCents, 2_500)
    : input.subtotalCents;

  await sink.write({
    event: "pricing_rule_evaluated",
    occurred_at: new Date().toISOString(),
    request_id: input.requestId,
    trace_id: input.traceId,
    span_id: input.spanId,
    order_id: input.orderId,
    pricing_rule: "margin-floor",
    rule_version: 3,
    flag_key: "pricing-margin-floor-v3",
    flag_enabled: flagEnabled,
    rollout_bucket: input.rolloutBucket,
    outcome,
  });

  return totalCents;
}
```

The `10` is visible on purpose. So are version `3` and the minimum of `2_500` cents. During an incident, implicit configuration is dead weight. A reviewer should be able to connect the deployed rule to the recorded event without guessing which defaults were active.

This is the contract that stays put while the backend moves. An adapter can batch and send these records to a hosted service later. Business code still knows only `LogSink`.

## Comparing the actual debugging shapes

These products overlap, but treating them as interchangeable creates bad evaluations. I would run the same pricing incident through each candidate: find one order, explain its decision, pivot to all rejected decisions for rule version 3, and correlate a background recalculation by `trace_id`.

| Option | Best fit for this rollout | Boundary to test before choosing |
| --- | --- | --- |
| Sentry | The incident begins in the browser and richer debugging workflows matter | Structured app logs are only one part of the evaluation; test the source-map, crash, and replay workflow you expect to use |
| Axiom | Engineers primarily explore structured event data and want the logging workflow to carry the investigation | Benchmark ingestion-to-query time and the exact query ergonomics on your event shape |
| Better Stack Logtail | The team wants a hosted log-management workflow with familiar collection and search | Test framework ingestion, field parsing, retention, alerting, and export against current product documentation |
| Infrai | A server-heavy app values a plain REST capability behind an application-owned contract, with other backend capabilities able to use the same key | No source-map deobfuscation, crash symbolication, session replay, distributed-trace query layer, or built-in notification route; correlation is through logged IDs |

That last row is narrow by design. Infrai centralizes server actions, API route logs, auth failures, and background-job output when they share consistent JSON fields. Its wider platform is self-describing, and documented capabilities include runnable TypeScript examples, which lowers time-to-first-call. **Its limitations and trade-offs are decisive:** it is not suitable when the real requirement is frontend replay, source-map deobfuscation, crash symbolication, or a trace waterfall. Sentry is the clearer candidate for that workflow. Pick for the investigation, not the route count.

Healthchecks.io belongs beside this comparison even though it is not a log store. Logs show what ran. A heartbeat monitor detects the silent case where the scheduled repricing job never ran. I would not ask a search screen to prove absence reliably.

## What I would change at scale

First, I would put a queue behind `LogSink` and batch network writes. The request path should not wait on a remote logging service. The queue needs a bound, a retry policy, and a dropped-event counter; “async” without backpressure is config-shaped optimism.

Second, I would add a schema version and contract tests. A deploy that renames `rule_version` to `version` can quietly split an investigation across two fields. That is worse than a loud type error.

Third, I would separate lookup identifiers from aggregate dimensions and benchmark with production-shaped volume. I care about three measurements: p95 added request latency, time from write to searchable event, and time for an engineer to reconstruct the flagged decision. No invented leaderboard. Run the same fixture against every candidate because vendor demos rarely match your field distribution.

Alerting is a separate decision. If the selected logging capability has no threshold rules or webhook, phone, or SMS delivery, polling search is possible but makes you own scheduling, deduplication, notification delivery, and failure handling. I wouldn't smuggle that maintenance burden into a “small” logging integration. A dedicated alerting path is usually cleaner. Likewise, confirm retention, bulk export or subscription, and deletion requirements before ingestion. If per-user deletion is mandatory, excluding personal data is the minimum; it does not create a deletion API.

Keep the blast radius small.

## The decision rule

Choose Sentry when this pricing incident is likely to start as a browser error and you need source maps, symbolication, or replay in the same debugging workflow. Put Axiom and Better Stack Logtail through a real event-search bake-off when structured-log exploration is the main job. Consider Infrai when the workload is server-side, correlation through explicit IDs is sufficient, and keeping one application contract while swapping the provider is valuable.

Then add a heartbeat service for jobs that might never start. No log platform can emit an event for code that did not run.

The cheapest-looking plan is irrelevant if an engineer cannot answer why the 10% bucket got rule version 3. **Optimize for reconstruction time and contract stability.** Everything else is a secondary filter.

## Further reading

- Sentry JavaScript source maps: https://docs.sentry.io/platforms/javascript/sourcemaps/
- Sentry Session Replay: https://docs.sentry.io/product/explore/session-replay/
- Axiom documentation: https://axiom.co/docs
- Better Stack Logs documentation: https://betterstack.com/docs/logs/
- Healthchecks.io documentation: https://healthchecks.io/docs/
- Prometheus instrumentation practices: https://prometheus.io/docs/practices/instrumentation/
- GDPR Article 17, right to erasure: https://gdpr-info.eu/art-17-gdpr/
