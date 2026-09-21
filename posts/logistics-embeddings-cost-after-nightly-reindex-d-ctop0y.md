# Logistics Embeddings Cost After Nightly Reindex: Debug Duplicate Calls

Short answer: Keep the nightly refresh, but embed only changed, uniquely identified chunks. For logistics listings collected from several feeds, compare normalized chunk fingerprints before making an embedding call; record attempted and completed calls separately. If embedding cost exploded after a nightly reindex, a rising bill is a symptom, not proof that the corpus grew.

| Refresh choice | Freshness | Duplicate-call risk | Use when |
| --- | --- | --- | --- |
| Re-embed every listing nightly | Predictable refresh window | High when feeds repeat records | Chunk rules or embedding model changed globally |
| Fingerprint chunks and embed changes | Tracks actual content changes | Low with durable claims and retries | Most recurring multi-feed imports |
| Skip scheduled imports entirely | Stale until another trigger runs | Low | Feeds provide a trustworthy change signal and missed updates are acceptable |

The default is the middle row. It adds a small amount of state, so measure its payoff: count unique fingerprints, embedding attempts, successful writes, and replays per run. If the numbers cannot be reconciled, do not treat a lower request count as evidence that listings are current.

## Where did the duplicate embedding calls enter?

Start with the unit of work, not the invoice. A nightly import may fetch the same freight listing from two sources, read yesterday's unchanged description under a new crawl timestamp, split it into several chunks, then retry one batch after a timeout. Those are distinct paths to repeat calls. An embedding request made twice for the same chunk is different from two legitimate chunks that happen to describe similar routes.

Assign each ingestion run an ID and log the source record ID, canonical listing ID, chunk index, content fingerprint, chunking revision, model identifier, attempt ID, and final write status. Avoid putting raw listing text in logs. Reconcile four counts: records fetched, unique canonical listings, unique chunk keys needing vectors, and embedding requests actually sent. If requests exceed the unique keys, debug retries and concurrent workers first. If unique keys themselves jump while listing count holds steady, inspect normalization and chunk boundaries. A crawl timestamp or feed-specific tracking URL must not become part of the text fingerprint unless the reader should retrieve it.

Two attempts are not two changes.

Do one controlled comparison: run the same frozen input twice against a disposable index, with the same chunking and model identifiers. The second pass should request no embeddings for unchanged chunks. This is a test design, not a claimed benchmark. Record both runs' counters before changing production scheduling.

## What should define a changed chunk?

The first criterion is identity. Map each feed's record to a stable listing key before chunking; otherwise one shipment advertised by two sources can create two independent vector histories. Do not merge merely because descriptions look alike: route, availability, and source-specific terms can matter. Keep source provenance alongside the canonical key so a later update or removal can be traced.

The second criterion is freshness. Separate fields that affect retrieval meaning from volatile ingestion metadata. For example, a pickup window or load status may change the answer, while a crawler's observed-at timestamp alone should not trigger re-embedding. Pick a deterministic field order and text normalization policy; changing either is a chunking revision, not an invisible cache miss. A shorter chunk can be cheaper to recompute but increases the number of calls and makes small boundary shifts more disruptive. Measure chunks per listing and changed chunks per refresh, not just total records.

Consider a listing whose pickup date moves but whose route does not: a fingerprint of route alone would skip the update, while a fingerprint containing every crawl timestamp would trigger a vector call at every import. The useful boundary lies in fields that alter the retrieval answer. Keep the original pickup date in the record even if the embedding text summarizes it differently. Compare a changed listing's before-and-after chunk keys during a trial run; a shift in all indices after editing one opening sentence signals a chunk-boundary problem, not a sudden surge of new freight. This diagnostic matters more than tuning a nominal chunk size in isolation.

This is the awkward trade-off: a stale vector can survive a perfectly deduplicated pipeline. Store a last-seen marker and expire missing listings according to the feed's deletion semantics. If the source only supplies periodic snapshots, absence may mean deletion; if it supplies partial pages, absence proves nothing. Keep the original listing fields available for freshness checks at answer time, since retrieval-augmented generation depends on the retrieved evidence, not on an index's claim to be current.

Freshness wins over a tidy cache statistic.

## How can a worker enforce the boundary?

The example keys a vector to canonical identity, relevant text, and processing revisions. The backing store must implement an atomic claim with a lease or equivalent compare-and-set operation; an in-memory set alone will not protect two workers. It also needs a completed state that survives restarts. The snippet intentionally leaves storage and embedding transport behind interfaces.

```ts
import { createHash } from 'node:crypto';

type Chunk = { listingId: string; index: number; text: string };
type ClaimStore = {
  claim(key: string): Promise<boolean>;
  complete(key: string, vector: number[]): Promise<void>;
  release(key: string): Promise<void>;
};
type Embed = (text: string) => Promise<number[]>;

async function embedChanged(
  chunk: Chunk,
  revisions: { chunker: string; model: string },
  store: ClaimStore,
  embed: Embed,
): Promise<'unchanged' | 'embedded'> {
  const digest = createHash('sha256').update(chunk.text, 'utf8').digest('hex');
  const key = JSON.stringify([chunk.listingId, chunk.index, revisions.chunker, revisions.model, digest]);
  if (!(await store.claim(key))) return 'unchanged';
  try {
    const vector = await embed(chunk.text);
    await store.complete(key, vector);
    return 'embedded';
  } catch (error) {
    await store.release(key);
    throw error;
  }
}
```

A retry after the embedding service succeeds but before `complete` persists can still make another paid call. No local key removes that ambiguity. Use a provider-supported idempotency mechanism if one exists and its documented semantics fit; otherwise count and alert on retried attempts, and design the claim lease so abandoned work can resume. Test simultaneous claims, lease expiry, partial batches, and a failure between embedding and persistence.

## When is a full rebuild the better choice?

Choose the first row when a model or chunking revision invalidates every stored vector, or when the canonical mapping was wrong and the existing index cannot be trusted. Rebuild into a separate index generation, check record and chunk counts against the source snapshot, then switch reads after validation. Keep the old generation until the new one passes retrieval checks against representative route and availability queries. An immediate overwrite makes rollback and comparison harder.

A feed with reliable change events can justify the third row, but a periodic reconciliation pass still catches missed events. The relevant operating metric is not merely calls saved: compare changed chunks, request attempts per changed chunk, indexing lag, and retrieval of recently updated listings. If freshness fails, tighten the feed reconciliation policy before adding more embedding capacity. Small glue code is worthwhile here only if it produces an auditable answer to one question: why was this particular chunk embedded tonight?

## References

- Lewis et al., Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks: https://arxiv.org/abs/2005.11401
- HTTP Semantics, conditional requests and validators: https://www.rfc-editor.org/rfc/rfc9110.html

## Further reading

- https://arxiv.org/abs/2005.11401
- https://www.rfc-editor.org/rfc/rfc9110.html
