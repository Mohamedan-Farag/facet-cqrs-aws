[← README](../README.md) | **Requirements** | [Next: Architecture →](./02-architecture.md)

# 1. Requirements

## 1.1 Problem Statement

A marketplace stores its product catalog in Amazon DynamoDB. DynamoDB serves point
reads and seller-scoped listings in single-digit milliseconds at any scale — but it
cannot answer the query buyers actually ask:

> *"Laptops · under 20,000 EGP · rating above 4 · in stock · sorted by price · show me
> how many results fall in each brand and price band."*

That query needs full-text relevance, arbitrary multi-attribute filtering, sorting on
non-key attributes, and aggregations. DynamoDB provides none of them. Adding a Global
Secondary Index buys exactly one more access pattern per index — it does not buy
arbitrary filtering, and the number of GSIs needed grows combinatorially with the
number of filter attributes.

**Facet solves this with CQRS**: DynamoDB remains the authoritative write model, and a
derived OpenSearch read model is kept in sync through Change Data Capture. The hard
part is not standing up OpenSearch — it is keeping the two stores consistent, correct,
and rebuildable when things go wrong.

## 1.2 Functional Requirements

### Command side (write)

| ID | Requirement |
|----|-------------|
| F-1 | Authenticated sellers create, update, and delete products through a REST API. |
| F-2 | Every write is validated and persisted to DynamoDB with an incrementing `version` attribute guarded by optimistic locking. |
| F-3 | A concurrent update to the same product must fail with `409 Conflict` rather than silently overwriting. |
| F-4 | Sellers list and read their own products with **strong consistency** — never through the search index. |

### Sync side (the core of this project)

| ID | Requirement |
|----|-------------|
| F-5 | Every committed write propagates to the read model automatically, with no application code writing to both stores. |
| F-6 | A stale (out-of-order) update must never overwrite a newer document in the read model. |
| F-7 | A single malformed record must not stall the sync pipeline for other records. |
| F-8 | Records that cannot be indexed after retries are captured for inspection and replay — never silently dropped. |
| F-9 | The full read model can be rebuilt from the write model with zero search downtime. |

### Query side (read)

| ID | Requirement |
|----|-------------|
| F-10 | Buyers run full-text search across product title and description. |
| F-11 | Results support filtering by category, brand, price range, rating, and availability. |
| F-12 | Results return **facet counts** (how many products match each category / brand / price band). |
| F-13 | Results support sorting by relevance, price, and rating, with stable pagination. |
| F-14 | Clients never send raw OpenSearch query DSL — the API exposes a constrained query contract only. |

## 1.3 Non-Functional Requirements

| Category | Target | How it is measured |
|---|---|---|
| **Search latency** | p95 < 300 ms, p99 < 800 ms | API Gateway `Latency` metric on `GET /search` |
| **Write latency** | p95 < 150 ms | API Gateway `Latency` on `POST/PUT /products` |
| **Sync lag** | p95 < 5 s, p99 < 30 s | Custom EMF metric `SyncLagMs`, plus `IteratorAge` on the stream |
| **Durability** | Zero tolerated loss in the write model; the read model is explicitly **disposable** | PITR enabled; reindex runbook tested |
| **Availability** | No single component failure takes the write path down | Managed multi-AZ services; read model degrades to DynamoDB-only |
| **Reindex** | Full 2 M documents in under 30 minutes with **zero** search downtime | Step Functions execution duration; alias swap |
| **Security** | No public data-store endpoints; encryption at rest and in transit throughout | See [§8 Security](../README.md#8-security) |
| **Cost** | Demo runs inside the AWS Free Tier apart from AWS WAF | See [Cost Estimate](./08-cost-estimate.md) |

### The consistency contract (stated explicitly, because it is a design output)

| Read | Store | Consistency |
|---|---|---|
| Seller reads own product (`GET /products/{id}`) | DynamoDB | **Strong** |
| Seller lists own products (`GET /products?mine`) | DynamoDB GSI1 | Eventually consistent (GSI), sub-second |
| Buyer searches the catalog (`GET /search`) | OpenSearch | **Eventually consistent, p95 < 5 s** |

This asymmetry is deliberate and is [Design Decision 11](./03-design-decisions.md#311-read-your-writes--who-is-allowed-to-see-stale-data).

## 1.4 Scale Anchor

Every decision in this document is justified against these numbers. Without them,
architecture is opinion.

**Target: a mid-size regional marketplace.**

| Signal | Number | Consequence for the design |
|---|---|---|
| Products in catalog | 2,000,000 | ~4 GB of index (≈2 KB/doc); fits one small node comfortably |
| Registered sellers | 5,000 | Cognito Lite tier is sufficient |
| Product writes (steady) | ~500 / minute | ~8 WRU/s — far below any DynamoDB partition limit |
| Product writes (burst) | ~5,000 / minute for ~10 min (bulk CSV import by a large seller) | Stream backlog must drain without alarming; drives `ParallelizationFactor` |
| Buyer searches (peak) | 200 req/s | Single `t3.small.search` node is marginal → drives the documented production topology |
| Average document size | ~2 KB | Bulk batches of 100 docs ≈ 200 KB — well under the 100 MB `_bulk` ceiling |
| Full reindex frequency | Monthly, or on any mapping change | Must not consume table read capacity → drives S3 Export over Scan |
| Stream retention | 24 hours (fixed by DynamoDB) | Pipeline outage > 24 h ⇒ reindex is the **only** recovery → drives R-7 |

## 1.5 Assumptions

1. The write volume never approaches DynamoDB's 1,000 WCU per-partition ceiling, so
   write sharding is out of scope. At 8 writes/second across 2 M distinct partition
   keys, this holds by a very wide margin.
2. Buyers tolerate a few seconds of catalog staleness. Sellers do not — hence the
   split read paths.
3. Product documents are self-contained after denormalisation; a seller renaming their
   storefront triggers a scoped partial reindex, not a full one.
4. Search traffic is read-heavy and cacheable at the edge for popular queries.

## 1.6 Out of Scope

Stated explicitly so the architecture is not judged against goals it never had.

- **Multi-region active-active.** Single region (`us-east-1`). Multi-region would
  require Global Tables plus cross-region replication of the read model, and is listed
  in [Future Work](./06-future-work.md).
- **Personalised ranking / recommendations.** Relevance is BM25 plus deterministic
  boosts; no ML ranking, no OpenSearch k-NN vector search.
- **Real-time inventory decrement.** Stock is an indexed attribute, not a transactional
  reservation system.
- **Payments, checkout, order management.** Catalog only.
- **Multi-language analysis.** English and Arabic analyzers are configured; no
  per-locale index-per-language strategy.
- **Autocomplete / type-ahead as a separate low-latency service.** Noted in Future Work.

---

[← README](../README.md) | **Requirements** | [Next: Architecture →](./02-architecture.md)
