# Pacing Rate-Limited Weekly Customer Digests Through a Node.js Public Webhook

Use the every-minute trigger only to admit work to a durable queue; let workers enforce the downstream rate limit and make each digest delivery idempotent. A public webhook should acknowledge quickly, not become the worker.

Short answer: for an e-commerce weekly digest, the reliable shape is a small Node.js endpoint behind authenticated HTTPS, a queue containing stable customer-and-window keys, and a worker that releases messages at the permitted pace. A clock starts the run. It does not own the work.

That separation is the useful constraint. If a store has 18,000 active customers and the email or recommendation API accepts a steady rate, putting the whole batch in a cron callback creates a burst, a long request, or both. The endpoint should enqueue a bounded slice, return `202`, and leave retries, backoff, and duplicate delivery to the processing layer.

## Idempotency is the admission contract

First define what “every minute” means. It is a dispatch opportunity, not a promise that 60 seconds of work will finish inside 60 seconds. Schedules can have timing jitter. A run can also overlap the previous run unless the application makes admission idempotent.

The customer window is the record of truth.

For the weekly digest, store a durable campaign window such as `2026-W32`, then derive a key from that window and the customer ID. The scheduler can ask for the next 100 eligible customers each minute. If the worker is allowed 20 downstream calls per minute, the queue can absorb the difference without turning one schedule tick into 100 immediate calls. That record also gives an operator something concrete to inspect when a customer asks why a digest is late: eligibility, claim time, attempt count, last response, and next attempt are more useful than a green scheduler history page. A unique `(customer_id, digest_window)` constraint prevents an overlapping trigger from inventing a second logical send, while a separate delivery record lets the consumer retry transport without changing the business identity.

The exact batch size belongs to the dependency contract. Measure it. I care about queue age, release rate, duplicate side effects, and recovery after a process restart. A fast `202` from the webhook is not a throughput benchmark.

Missed schedule runs need an explicit policy. If “send this week’s digest” must eventually reach every eligible customer, maintain a cursor or eligibility table and let the next trigger resume it. If a digest is only useful within a narrow time window, expire the window instead. The scheduler's history is not a business ledger.

## Choose the public boundary before the code

The endpoint should admit work, not perform the digest. It needs authenticated HTTPS, a bounded request, and a fast response. The queue and worker own the slower part.

## How should a Node.js cron trigger pace a queue every minute?

The example below keeps the moving parts visible. It has an in-memory array, so it is a demonstration of the handoff rather than a production queue. The important parts are the stable job key, the quick admission response, and the single pacing point.

```ts
import { createServer } from "node:http";
import { timingSafeEqual } from "node:crypto";

type DigestJob = {
  customerId: string;
  window: string;
  idempotencyKey: string;
};

const token = process.env.SCHEDULER_TOKEN;
if (!token) throw new Error("SCHEDULER_TOKEN is required");

const queue: DigestJob[] = [];
const port = Number(process.env.PORT ?? "3000");
const customersPerTick = 100;
const callsPerMinute = 20;
const intervalMs = 60_000 / callsPerMinute;

function hasValidToken(value: string | undefined): boolean {
  if (!value?.startsWith("Bearer ")) return false;
  const provided = Buffer.from(value.slice("Bearer ".length));
  const expected = Buffer.from(token);
  return provided.length === expected.length && timingSafeEqual(provided, expected);
}

function enqueueWindow(window: string): number {
  const eligibleCustomerIds = loadNextEligibleCustomerIds(window, customersPerTick);
  for (const customerId of eligibleCustomerIds) {
    queue.push({
      customerId,
      window,
      idempotencyKey: `digest:${window}:${customerId}`,
    });
  }
  return eligibleCustomerIds.length;
}

function loadNextEligibleCustomerIds(window: string, limit: number): string[] {
  // Replace this with a query that advances a durable cursor atomically.
  return getEligibleCustomerIds(window).slice(0, limit);
}

async function deliver(job: DigestJob): Promise<void> {
  await sendDigest({
    customerId: job.customerId,
    window: job.window,
    idempotencyKey: job.idempotencyKey,
  });
}

setInterval(() => {
  const job = queue.shift();
  if (job) void deliver(job).catch((error: unknown) => console.error(error));
}, intervalMs);

createServer((request, response) => {
  if (request.method !== "POST" || request.url !== "/digest/enqueue") {
    response.writeHead(404).end();
    return;
  }
  if (!hasValidToken(request.headers.authorization)) {
    response.writeHead(401).end();
    return;
  }

  const window = currentDigestWindow();
  const accepted = enqueueWindow(window);
  response.writeHead(202, { "content-type": "application/json" });
  response.end(JSON.stringify({ accepted, window }));
}).listen(port);

declare function getEligibleCustomerIds(window: string): string[];
declare function sendDigest(input: DigestJob): Promise<void>;
declare function currentDigestWindow(): string;
```

The declarations are the boundary, not magical Node.js APIs. In a real service, `loadNextEligibleCustomerIds` must claim rows or job keys atomically. Otherwise two overlapping trigger calls can select the same customers. The downstream write must also treat a repeated `idempotencyKey` as the same logical delivery.

Do not let the timer callback become an accidental queue implementation.

The queue array is intentionally disposable. Process restarts erase it, multiple instances do not share it, and a slow `sendDigest` call can make the simple timer misleading. Replace it with durable storage before the first customer-facing send. That is a hard boundary.

## Test duplicate delivery before throughput

The obvious failure is a burst: every scheduled tick pushes a full batch and every worker starts immediately. A less obvious one is an acknowledgement made too early. If a worker removes a message before persisting the delivery result, a process exit can turn a successful admission into a lost digest. If it persists the result and then exits before acknowledgement, the queue may deliver the message again. At-least-once processing makes that second outcome normal. Picture one concrete run: customer `c-1842` is claimed for `2026-W32`, the dependency accepts the request, and the process dies before the local delivery record is committed. The retry must be allowed to send again because the system cannot prove the first side effect was durable. Now invert it: the local record commits, then acknowledgement is lost. The retry sees the same key and becomes a no-op. Those two cases look identical to a naive timer, but they demand different handling at the application boundary. This is why idempotency is not a nice property to add after rate limiting; it is what makes a rate-limited retry safe to operate.

Handle it as a state transition. Claim a stable key, call the dependency with the same idempotency key, record the result, then acknowledge. A `429` should delay the attempt according to the dependency's contract, often using `Retry-After` or bounded exponential backoff. The retry counter and next-attempt time belong in durable state, so a restart does not reset the pressure on the dependency.

Keep failed messages available for inspection. A dead-letter queue gives operators a place to examine messages that exceed the retry policy rather than letting one poisonous customer record block ordinary work. The queue's retention period, visibility behavior, message size, and acknowledgement semantics should be checked against the actual digest payload; do not assume a task queue is a replay log.

Security is part of the scheduling design. The endpoint is public by definition, so require HTTPS, authenticate the scheduler, validate the method and body, cap request size, and return no customer data in the response. A secret in a header is a starting point. Signature verification, replay protection, and network controls may be required by the deployment environment.

## The operating ledger at scale

I would move eligibility into a table with a unique constraint on `(customer_id, digest_window)`. The trigger would claim a page of rows, publish compact references, and record the claim. Workers would expose an internal metric for permitted calls, actual calls, retry delay, queue age, and dead-letter count. Logs would carry the customer ID only when policy allows it; the idempotency key is usually the safer correlation field.

Here is the decision table I use before choosing more infrastructure:

| Constraint | Smallest useful boundary | Move up a category when |
| --- | --- | --- |
| One recurring handoff | Scheduler, durable queue, idempotent worker | A missed run needs a business-specific recovery path |
| A downstream quota | One pacing point and retry state | Several consumers must coordinate one quota |
| Duplicate delivery | Stable key plus idempotent write | The dependency cannot accept repeated requests |
| Private network target | Private scheduler or internal worker | Public HTTPS ingress is prohibited |
| Multi-step dependency graph | Workflow state with explicit transitions | The job needs joins, approvals, or human recovery |

I would test time as data. Freeze the digest window, invoke the webhook twice, kill a worker after the external call, return a throttling response, and pause the scheduler for several intervals. The assertions are practical: no duplicate customer-facing side effect, no silent loss, bounded release rate, and a clear outcome for missed windows.

A small team should resist adding a workflow engine just because the word “schedule” appears in the requirement. A cron trigger, queue, and idempotent consumer are enough for one recurring handoff. Add workflow state when the digest has real dependencies, approvals, fan-out and join behavior, or human recovery steps.

## A practical fit test

This pattern is suitable when a public HTTPS ingress is acceptable and eventual processing is more valuable than an exact execution timestamp. The catch is that the queue adds another operational boundary. It is not suitable when the endpoint must remain private, when the downstream operation cannot tolerate duplicate calls and has no idempotency mechanism, or when the job is a multi-step workflow with a required join. Stick with a private scheduler for a private target, a transactional outbox for a local database handoff, and a workflow system for joins or human recovery.

The queue adds operational work: capacity, retention, poison-message handling, visibility timeouts, metrics, and credentials. The benefit is a failure model you can inspect. A one-line timer has less configuration, but it hides the backlog and makes restart behavior part of the business outcome.

Your mileage may vary on the batch size. It depends on the dependency's real quota, not on the scheduler's label. Start with the smallest slice that makes queue age visible, then benchmark duplicate delivery and recovery before increasing it.

## References

- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html
- https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/
