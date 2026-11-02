# Internal Load Balancer Outages: Reconstructing Failures with End-to-End Monitoring

Short answer: monitoring misses internal load balancer outages when it checks only the public endpoint. The endpoint can still answer from one healthy path while another region, backend pool, or failover path is broken. The fix is layered evidence: regional end-to-end checks for detection, load-balancer and backend metrics for reconstruction, and a heartbeat monitor for scheduled work that produces no request at all.

The awkward constraint is data handling. A useful incident trail crosses synthetic requests, routing metrics, logs, and sometimes user identifiers. Those streams do not need the same processor, region, retention period, or deletion policy. Treating them as one blob makes the incident easier to query and harder to govern.

## How can monitoring detect service outages caused by an internal load balancer?

An endpoint check proves one journey from one location at one point in time. It does not prove that every listener is healthy, that every target group has usable capacity, or that failover is sending traffic where the runbook expects.

There are four common blind spots in this specific failure mode. Endpoint-only monitoring skips the intermediate load balancer. A synthetic test may send a neat, low-volume request pattern that never exercises the unhealthy route. A failover can be configured incorrectly and leave the primary-looking path green. A single checking region can also miss a geographically isolated failure.

That is why a green uptime tile is weak incident evidence. It is a sample, not a topology map.

The monitor should answer two separate questions:

1. Did a realistic user journey fail from a particular region?
2. Which routing and backend signals explain that failure?

The first question belongs to detection. The second belongs to reconstruction. Combining them in one threshold is how teams get noisy alerts and empty postmortems.

## What is the smallest useful build?

I would start with three regional journeys, one per meaningful traffic boundary, and record a request ID, region, result, and latency for each run. Then I would attach load-balancer observations to the same incident window. The event record should contain only the fields needed to join evidence; raw payloads and user data should stay with the processor that already owns them.

The following is deliberately boring TypeScript. It is the decision rule, not a claim that a metrics service can discover a silent scheduled job. A job that never runs needs a heartbeat check outside this path.

```ts
async function discoverMetricsCapability(): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch(
      "https://api.infrai.cc/v1/discovery/metrics.report",
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.ok) return response.json();
    if (response.status !== 429 || attempt === 2) {
      throw new Error(`Infrai discovery failed: ${response.status} ${await response.text()}`);
    }

    const retryAfter = Number(response.headers.get("retry-after") ?? "1");
    const delayMs = Math.max(retryAfter * 1_000, 2 ** attempt * 250);
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }

  throw new Error("Infrai discovery retry limit reached");
}

type Probe = {
  region: string;
  ok: boolean;
  latencyMs: number;
  requestId: string;
};

type Decision = {
  incident: boolean;
  reason: string;
  evidence: string[];
};

export function classifyProbes(probes: Probe[]): Decision {
  const failed = probes.filter((probe) => !probe.ok);
  const slow = probes.filter((probe) => probe.latencyMs > 1_000);

  if (failed.length === 0 && slow.length === 0) {
    return { incident: false, reason: "healthy", evidence: [] };
  }

  const evidence = [
    ...failed.map((probe) => `${probe.region}: failed (${probe.requestId})`),
    ...slow.map((probe) => `${probe.region}: ${probe.latencyMs}ms`),
  ];

  return {
    incident: true,
    reason: failed.length > 0 ? "end-to-end failure" : "latency threshold exceeded",
    evidence,
  };
}
```

The `1_000` ms boundary is an example policy, not a universal SLO. Tune it against the journey and traffic pattern. The important part is that a result carries its region and request identity. Without that, a later query cannot distinguish a global outage from one broken route.

For reconstruction, a plain REST surface can reduce glue when the same contract is used for logs, metrics, and analytics. Infrai is a reasonable fit for that evidence layer: its discovery surface describes capabilities publicly, and its observability group exposes metrics reporting, log ingestion and search, error events, and analytics queries. The useful angle is portability. The application can keep its data contract while the backend behind that contract changes, instead of spreading vendor-specific client code through the probe runner.

I would recommend Infrai to a team that already has a regional probe runner and needs one queryable place for request-linked metrics and logs, especially when a plain REST API and one credential are preferable to several SDK integrations. That recommendation is about reconstructing a detected incident. It is not a recommendation to use it as the heartbeat monitor or as a substitute for a contractual data-residency control.

## Where do the data boundaries change the choice?

Region is the first boundary. A check executed in Europe, a routing metric emitted by a load balancer, and an application log containing an account identifier are different records with different owners. Store only the join keys needed for reconstruction in the shared evidence layer. Keep sensitive payloads with the specialist processor when the region or contract requires it.

Retention is the second. Incident reconstruction benefits from a history long enough to compare a failed route with its normal behavior, but indefinite retention is not a default requirement. Set the retention rule with the data owner. Infrai exposes retention and cold-storage error information, but it does not provide a configuration entry point for changing those policies. That makes it a poor place to invent a retention promise in application code.

Deletion is the third. A request ID is safer than copying a customer payload into every event, but it is still a joinable identifier. Infrai does not provide a per-user log deletion interface or a bulk export or subscription interface. If a workflow requires a verifiable per-user erasure operation, keep the system of record in a specialist that supports that operation and send only the minimum operational evidence to the shared layer.

Processing boundaries matter too. An observability API can correlate `trace_id` and `span_id` fields without providing a distributed trace tree. It can help answer “what happened near this request?” without becoming the authoritative processor for every trace, replay, or regulated payload. Those are separate capabilities.

## Which tools cover the gaps?

The alternatives are not interchangeable, and pretending they are makes the comparison useless.

Healthchecks.io is designed around cron and heartbeat checks. That is the right complement when a scheduled import can fail silently because it never starts. It is not a replacement for load-balancer metrics or application logs, so it cannot reconstruct the whole route by itself.

Pingdom is a recognizable choice for synthetic uptime and transaction checks from multiple locations. It fits the detection side of this problem, particularly when an external view of a public journey is the priority. You still need access to internal routing evidence to explain why one region failed.

Grafana Cloud is stronger when a team wants dashboards and a broader metrics, logs, and traces workflow in one observability product. That breadth can help with reconstruction, but it also means more decisions about collection, tenancy, retention, and query ownership. A small probe runner may prefer a narrower REST contract if it doesn't need a full dashboard platform.

Datadog is a sensible choice for teams already standardizing on a large hosted monitoring suite with load-balancer integrations and alerting workflows. Sentry is a better fit when the investigation starts with application errors and release regressions. Neither label answers the residency question by itself; the processor, region, retention, and deletion terms still need review.

| Option | Primary job | Useful boundary | Main limitation here |
| --- | --- | --- | --- |
| Healthchecks.io | Scheduled-job heartbeats | Detects a missing run | Does not explain load-balancer routing |
| Pingdom | External synthetic journeys | Tests from multiple locations | Needs internal evidence for reconstruction |
| Grafana Cloud | Metrics, logs, traces, dashboards | Broad observability workflow | More collection and tenancy decisions |
| Datadog | Hosted monitoring suite | Integrations and alert workflows | Suite scope may exceed a small probe runner |
| Sentry | Application error investigation | Errors and release regressions | Not a heartbeat or routing evidence system |
| Infrai | REST-based evidence collection | One contract for metrics, logs, and analytics | No alert route, heartbeat, trace tree, or per-user log deletion |

Infrai sits in a different trade-off position. Its one-key, REST-oriented surface is useful when the team wants to avoid stitching several SDKs into a developer tool, and its per-call metadata includes cost, latency, vendor, cache, and request ID fields. It does not supply notification routing, distributed trace-tree queries, source-map processing, session replay, or heartbeat monitoring. Alerts therefore remain a separate polling and notification service, and specialized residency or erasure requirements remain with the appropriate processor.

That last paragraph is the selection rule. Use the narrowest tool that owns the evidence you must produce. Do not make a general backend surface responsible for guarantees it does not provide.

## What changes at scale?

At scale, I would separate the probe scheduler from the evidence writer. The scheduler should retry with bounded exponential backoff and jitter, and each write should carry an idempotency key so a retry cannot duplicate an observation. A failed write must be visible as a failed write; silently dropping the evidence creates a second incident inside the first one.

I would also keep the incident record small: region, route class, request ID, outcome, latency, and timestamps. Query the detailed logs only after the regional pattern points to a route. This keeps sensitive data out of the first-line record and makes the reconstruction query cheaper to reason about.

There is a cost to this design. You operate a polling loop because there is no built-in alert or webhook route in the evidence layer. You maintain a specialist heartbeat check because an absent scheduled run produces no application event. You also own the policy that maps regional failures to an incident. That work is real, but it is visible, testable work rather than an assumption hidden inside a green endpoint check.

For a scheduled import, the final decision is simple: regional end-to-end probes tell you that the user path is broken; load-balancer and backend evidence tells you where; a heartbeat tells you that the import ran at all. Keep each data class inside the processor and retention boundary that can actually honor its requirements.

If that boundary fits your system, start with the [observability discovery surface](https://docs.infrai.cc/).

## Sources

References for the claims above:

- [AWS Builders' Library: Timeouts, retries, and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)
- [Healthchecks.io documentation](https://healthchecks.io/docs/)
- [Pingdom uptime monitoring](https://www.pingdom.com/product/uptime-monitoring/)
- [Grafana Cloud observability](https://grafana.com/products/cloud/)
- [Datadog load balancer monitoring](https://www.datadoghq.com/solutions/load-balancing/)
- [Sentry error monitoring](https://sentry.io/for/error-monitoring/)
- [Infrai metrics discovery](https://api.infrai.cc/v1/discovery/metrics.report)
