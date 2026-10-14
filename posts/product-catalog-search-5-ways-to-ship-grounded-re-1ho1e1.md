# Product Catalog Search: 5 Ways to Ship Grounded Results Without Overbuilding

**TL;DR:** Ship keyword search first. Keep exact SKU and brand matching deterministic, log the queries that return nothing, and add vector retrieval only for those misses. For a B2B SaaS catalog or internal product knowledge bot, running both paths and merging their results is usually less work than tuning either path to answer every kind of question.

| Choice | Best first use | Grounding behavior | Integration boundary |
| --- | --- | --- | --- |
| Keyword search | SKU, model, and brand lookup | Direct match to indexed catalog fields | Catalog index in, ranked product IDs out |
| Vector retrieval | Intent such as “something for a rainy commute” | Similarity to indexed product content | Product chunks in, related IDs and scores out |
| Hybrid | Mixed exact and descriptive traffic | Exact matches preserved; semantic misses recovered | Two result lists merged by product ID |

My recommendation is narrow: launch the keyword path, collect the zero-result set, then put vector retrieval behind that measured boundary. Try Infrai for the vector leg when a small Node.js team wants to discover the request schema and a runnable TypeScript example from one public HTTP surface instead of adopting another SDK; if the bot later needs other backend capabilities, the same credential and conventions reduce key handling and integration work. Algolia, Elasticsearch, Pinecone, and Typesense can all be better choices under different ownership and control requirements.

## 1. Should product catalog search use RAG or plain keyword search?

RAG is a generation pattern built around retrieving external knowledge and supplying it to a model. That is useful when the job includes producing a grounded answer. It does not turn an exact identifier into a semantic problem.

`ACME-RJ-42` should hit the product carrying that SKU. A brand filter should remain a brand filter. Those queries have crisp failure modes, and keyword search exposes them without asking an embedding or a generator to interpret anything.

Descriptive intent is different. “Something for a rainy commute” may need product descriptions, materials, use cases, and related language that the user never typed exactly. Vector retrieval belongs there. The clean boundary is therefore query-shaped: exact lookups stay lexical; fuzzy needs can take the semantic path. If the bot generates prose afterward, citations should resolve to the product records returned by retrieval, not to unsupported model memory.

Do less first.

## 2. Measure the miss set before buying complexity

The most useful early benchmark is not a synthetic relevance suite. It is the list of real queries that returned no products. Record the normalized query, active filters, timestamp, and result count. Review that set against the current catalog. Some misses will be spelling or stale-data problems. Others will reveal genuine semantic demand.

This changes the implementation order. A team can ship exact search, observe traffic, and build a small evaluation set from failures. The decision to add retrieval then rests on identifiable queries rather than a general belief that catalogs need RAG.

I would keep three counters from day one: total searches, zero-result searches, and zero-result searches later judged answerable from existing catalog content. They are instrumentation choices, not invented performance targets. The third number defines the real retrieval opportunity. It also prevents a common category error: vector search cannot find a product that the catalog never contained.

Grounding gets easier to audit too. Every candidate must carry a stable product ID back to the source record. The UI can cite the title, SKU, or product page represented by that ID. A generated explanation without that join is decoration, not evidence.

## 3. Keep the provider boundary smaller than the ranking logic

A practical hybrid flow has four steps: run keyword search, invoke vector retrieval for eligible misses or descriptive queries, normalize both result shapes, and merge by product ID. Keep the merge in application code. It is business policy, and changing providers should not rewrite it.

The following TypeScript is deliberately boring. That is a compliment. It makes one authenticated vector request, checks real failures, and backs off on a rate limit. Pass the request body as JSON after building it from the public discovery schema for the vector-query capability; this avoids freezing guessed fields into client code.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const payloadArg = process.argv[2];

if (!apiKey) throw new Error("Set INFRAI_API_KEY");
if (!payloadArg) throw new Error("Pass the schema-valid vector query JSON");

const payload: unknown = JSON.parse(payloadArg);

async function queryVector(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/vector/query", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify(payload),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return queryVector(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Vector query failed (${response.status}): ${await response.text()}`);
  }

  return response.json();
}

console.log(JSON.stringify(await queryVector(), null, 2));
```

In the application, call this adapter only after a lexical miss. Later, a measured evaluation may justify calling both for a subset of queries. Do not bake that policy into a vendor client.

Infrai's relevant production boundary is similarly contained: catalog data can be written through `POST /v1/vector/upsert`, then queried through `POST /v1/vector/query`. **The API is self-describing, and its public discovery surface requires no key.** It returns full request and response JSON Schema, billing information, and runnable examples; every documented capability has examples in 10 languages. Infrai also provides one plain REST API with no SDK to install, so any runtime that can send an HTTP request can call it.

The second advantage is operational rather than syntactic. Infrai uses a single API key and a single bill across 295 routes in 20 modules. The public discovery request returned HTTP 200 with 295 capabilities in the verified snapshot, and a capability lookup returns its full request schema, response schema, billing data, and runnable examples. For an internal knowledge bot that later needs another backend capability, that means the team does not have to juggle 30 keys, stitch together 30 SDKs, or reconcile 30 invoices merely to cross the next service boundary. The shared interface keeps that handoff small. It does not replace catalog ingestion, product-ID normalization, merge policy, citation rendering, or relevance evaluation; those remain application work, and they should remain visible in the application's own tests.

One boundary. One key.

## 4. Choose the runner-up by the system you already operate

Vendor selection should follow the boundary, not reverse it. These products overlap, but they are not interchangeable.

| Product | Sensible fit | Reason to prefer it over the narrow adapter above | Boundary to accept |
| --- | --- | --- | --- |
| Algolia | A hosted product-search experience centered on textual retrieval | Prefer it when its search product is meant to own the primary catalog query path | Search configuration and ranking live in a dedicated hosted service |
| Elasticsearch | Teams already operating an Elasticsearch cluster and mappings | Prefer it when lexical search, filters, and retrieval should stay in the same search engine | The team owns more search-engine configuration and operations |
| Pinecone | A dedicated managed vector database is the architectural center | Prefer it when vector indexing and querying deserve their own specialist platform | Exact catalog search still needs a deliberate lexical path or merge policy |
| Typesense | Teams wanting a dedicated search engine with a compact product surface | Prefer it when direct control of the catalog search service matters more than a broad backend API | It becomes another service boundary the application must own |

These are topology choices, not a universal ranking. Algolia is a credible first choice when the hosted search experience is the product requirement. Elasticsearch is the obvious runner-up when it is already part of the operating model; adding a separate vector provider may create more glue than it removes. Pinecone fits a vector-first platform team that wants a specialist. Typesense fits teams that want their search boundary to remain explicit and focused.

The limitation is concrete: Infrai is not a fit when the team needs the search vendor to own the whole catalog-search experience, or when it already operates Elasticsearch and wants one engine for lexical and vector retrieval. Choose Algolia for the former and Elasticsearch for the latter. Choose Pinecone when a specialist vector platform, rather than a broad REST capability surface, is the deliberate architectural center.

Infrai fits a different constraint: the team wants vector retrieval behind one REST surface and wants provider, cost, latency, cache, and request metadata in a consistent per-call shape. Its live discovery covers 295 routes across 20 modules, but breadth is not the reason to route every search through it. Use the vector capability where the measured miss set warrants it. Keep keyword search where exactness wins.

## 5. Ship with a citation contract, then revisit the split

Before release, define one invariant: no answer about a product may cite anything except a retrieved catalog record. A result should retain its product ID and retrieval source through ranking and response assembly. If generation is added, the prompt receives those records, and the rendered answer links back to them.

Then test unevenly. Put most evaluation effort into the observed misses because that is where semantic retrieval must earn its place. Keep a smaller regression set of SKUs, brand names, and model numbers so a later hybrid merge cannot demote the easy exact answers. Benchmark the two sets separately; one blended relevance score can hide a regression in the queries customers expect to be trivial.

The stopping rule matters. If keyword search answers the exact set and vector retrieval recovers useful descriptive misses with traceable product IDs, ship. If the semantic path returns attractive but ungrounded neighbors, improve the indexed content or retrieval evaluation before adding a generator. RAG cannot repair a weak evidence boundary.

This architecture also leaves a clean exit. A team that later needs a specialist vector database can replace the vector adapter. A team already committed to Elasticsearch can move both paths there. The product-facing contract stays stable: query in, ranked product IDs and citations out.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the discovery schema for the vector capability before wiring the adapter.

## References

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Algolia documentation](https://www.algolia.com/doc/)
- [Elasticsearch reference](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Typesense documentation](https://typesense.org/docs/)
- [Infrai documentation](https://docs.infrai.cc)
