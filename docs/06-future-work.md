[← README](../README.md) | [← Risks](./05-risks.md) | **Future Work** | [Next: Appendix →](./07-appendix.md)

# 6. Future Work

Each item states the trigger that would make it worth doing. A v2 list without triggers
is a wish list.

---

### Kinesis Data Streams when a third consumer appears

**Trigger:** a second and third stream consumer — an analytics sink, an audit log, a
cache invalidator — pushing past DynamoDB Streams' two-consumer-per-shard ceiling, or a
requirement to replay further back than 24 hours.

Switch the table to Kinesis Data Streams for DynamoDB: up to 365 days of retention, many
independent consumers, and native fan-out to Firehose and Managed Service for Apache
Flink. The cost is that Kinesis for DynamoDB does not guarantee ordering across records
or exclude duplicates, so the indexer must lean harder on external versioning and become
explicitly idempotent — which, having already built 3.5, it largely is. This is a table
configuration change plus consumer hardening, not a redesign
([DD 3.3](./03-design-decisions.md#33-dynamodb-streams-over-kinesis-data-streams)).

### OpenSearch Serverless once traffic is continuous

**Trigger:** sustained traffic that keeps the cluster genuinely busy, at which point the
minimum-OCU floor stops being dead weight — or an operational preference to stop managing
shards and version upgrades.

The rejection in [DD 3.7](./03-design-decisions.md#37-opensearch-provisioned-over-serverless)
was a cost argument at demo scale, not an architectural one. Nothing outside the domain
configuration changes: the indexer, the alias-swap reindex, and the Search Lambda all work
unmodified.

### Zero-ETL for the subset of fields that map one-to-one

**Trigger:** the transform logic stabilising, or a second index (e.g. a lightweight
autocomplete index) whose documents genuinely are the raw item.

DynamoDB's Zero-ETL integration with OpenSearch removes the indexer for
straight-through replication. It remains unsuitable for the main product index while
field projection, derived facets, and denormalisation are required
([DD 3.4](./03-design-decisions.md#34-custom-indexer-lambda-over-zero-etl)).

### Autocomplete as a separate index

**Trigger:** type-ahead latency budgets below ~50 ms, which the main index cannot meet
while also serving faceted search.

A small `products-suggest` index using a completion suggester or edge n-grams, fed by the
same stream. Keeping it separate means suggestion tuning cannot destabilise the main
index's relevance or its mapping.

### Semantic search with k-NN vectors

**Trigger:** measurable recall loss on natural-language queries — buyers searching
"something to keep coffee warm" and getting nothing, because BM25 matches tokens, not
meaning.

Add a `knn_vector` field, generate embeddings in the indexer (Amazon Bedrock), and run a
hybrid query combining BM25 with vector similarity. This meaningfully increases index
size and indexing cost per document, so it should follow evidence, not fashion.

### Scoped partial reindex

**Trigger:** the first time a seller renames their storefront and a full 2 M-document
rebuild is triggered to update a denormalised field on 400 documents.

A Step Functions workflow accepting a query predicate — by seller, by category, by date
range — that reindexes only matching documents through the same versioned write path.
Minutes instead of half an hour, and no alias swap needed.

### Multi-region read replicas

**Trigger:** a second geography where search latency from a single region is
unacceptable, or an RTO that a single-region outage would breach.

DynamoDB Global Tables replicate the write model. The read model is *not* replicated —
each region runs its own indexer against its own regional stream and builds its own
index. That is cheaper and simpler than cross-region index replication, and it follows
directly from the read model being disposable: a derived store should be derived locally,
not shipped.

### Change-data-capture-driven cache invalidation

**Trigger:** edge caching of search responses causing visible staleness beyond the sync
lag SLO.

Reuse the existing stream: a second consumer issues targeted CloudFront invalidations for
affected query patterns. Requires the Kinesis migration above if the indexer already
occupies a consumer slot.

### Relevance tuning and search analytics

**Trigger:** the first "I searched for X and the right product was on page 3" report.

Log queries, zero-result rates, and click-through position to S3 via Firehose; analyse
with Athena. Feed the results into field boosts, synonym lists, and Arabic/English
analyzer tuning. Zero-result rate is the single most actionable search metric and costs
almost nothing to capture.

### Blue/green index promotion gated on quality

**Trigger:** a relevance change shipping to production and measurably making search
worse.

Extend the alias swap to run a golden-query set against the newly-built index and compare
top-N results against expected output before promoting. The alias mechanism already
supports this — the new index is queryable by concrete name before it takes traffic; only
the verification step is missing.

---

[← README](../README.md) | [← Risks](./05-risks.md) | **Future Work** | [Next: Appendix →](./07-appendix.md)
