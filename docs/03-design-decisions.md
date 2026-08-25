[← README](../README.md) | [← Architecture](./02-architecture.md) | **Design Decisions** | [Next: Well-Architected →](./04-well-architected.md)

# 3. Design Decisions

Every decision below follows the same structure: the **options** that were genuinely on
the table, the **choice**, and the **rationale** — including why the rejected options
were rejected. A rejected option with a reason is worth more than an unmentioned one.

**Contents**

| # | Decision | Category |
|---|---|---|
| [3.1](#31-cqrs-versus-a-single-data-store) | CQRS versus a single data store | Foundational |
| [3.2](#32-change-data-capture-over-dual-write) | Change Data Capture over dual-write | ⭐ Foundational |
| [3.3](#33-dynamodb-streams-over-kinesis-data-streams) | DynamoDB Streams over Kinesis Data Streams | Integration |
| [3.4](#34-custom-indexer-lambda-over-zero-etl) | Custom indexer Lambda over Zero-ETL | Integration |
| [3.5](#35-out-of-order-updates--external-versioning) | Out-of-order updates — external versioning | ⭐ Correctness |
| [3.6](#36-poison-records--partial-batch-response--bisect) | Poison records — partial batch response + bisect | ⭐ Reliability |
| [3.7](#37-opensearch-provisioned-over-serverless) | OpenSearch provisioned over Serverless | ⭐ Cost |
| [3.8](#38-opensearch-inside-a-vpc--and-avoiding-the-nat-gateway-trap) | OpenSearch inside a VPC — and the NAT trap | Security / Cost |
| [3.9](#39-reindex-source--s3-export-over-scan) | Reindex source — S3 Export over Scan | ⭐ Performance |
| [3.10](#310-zero-downtime-reindex--index-aliases) | Zero-downtime reindex — index aliases | Availability |
| [3.11](#311-read-your-writes--who-is-allowed-to-see-stale-data) | Read-your-writes — who may see stale data | ⭐ Consistency |
| [3.12](#312-search-lambda-over-direct-api-gateway--opensearch-integration) | Search Lambda over direct integration | Security |
| [3.13](#313-deletes--soft-delete-over-hard-delete) | Deletes — soft delete over hard delete | Correctness |
| [3.14](#314-deep-pagination--search_after-over-fromsize) | Deep pagination — `search_after` over `from`/`size` | Performance |

---

## 3.1 CQRS versus a single data store

**Options**
1. DynamoDB alone, adding a GSI per filter combination.
2. Aurora PostgreSQL alone, using `tsvector` full-text search and B-tree indexes.
3. CQRS — DynamoDB as the write model, OpenSearch as a derived read model.

**Choice: CQRS (option 3).**

**Rationale.**
Option 1 fails on arithmetic, not preference. A DynamoDB `Query` requires a known
partition key; `Scan` with `FilterExpression` reads the entire table and filters
afterwards, so it pays for every item examined regardless of how few match. Each GSI
buys exactly one additional access pattern. With five filterable attributes that users
may combine freely, the number of useful index permutations grows far faster than the
20-GSI-per-table quota allows — and none of them deliver relevance ranking or facet
counts, which are aggregations, not lookups.

Option 2 works technically. Postgres full-text search over 2 M rows is perfectly
capable. It was rejected on operational grounds: it means running a database instance
24/7 for a workload whose write path already scales to zero, it reintroduces connection
pooling and failover management to a stack that had neither, and its facet
aggregations over multi-million-row tables are markedly slower than an inverted index
designed for exactly that.

Option 3 keeps each store doing what it is good at: DynamoDB for authoritative,
single-digit-millisecond keyed access at unlimited scale; OpenSearch for relevance,
arbitrary filtering, and aggregation. The cost of CQRS is the synchronisation problem —
and that problem is the substance of this project, addressed in 3.2 through 3.11.

---

## 3.2 Change Data Capture over dual-write

**Options**
1. **Dual-write** — the Command Lambda writes to DynamoDB, then writes to OpenSearch.
2. **Transactional outbox** — write the item and an outbox event in one
   `TransactWriteItems`, then a poller ships outbox rows to OpenSearch.
3. **CDC via DynamoDB Streams** — DynamoDB emits the change; a Lambda indexes it.

**Choice: CDC via DynamoDB Streams (option 3).**

**Rationale.**
Option 1 is the default instinct and it is unsafe. The two writes are not atomic. If
the DynamoDB write commits and the OpenSearch write then fails — a timeout, a throttle,
a Lambda `OutOfMemory`, the container being frozen mid-execution — the index is
permanently wrong, and worse, *nothing in the system records that it happened*. There
is no retry queue, because the thing that would have retried is the process that died.
Making the API call synchronous also couples the seller's write latency to the health
of the search cluster: OpenSearch being slow makes product creation slow, which is
exactly backwards.

Option 2 is the correct pattern **when the platform does not give you CDC**. It buys
atomicity by writing the event into the same transaction as the data. Here it is
redundant: DynamoDB Streams *is* a durable, ordered, at-least-once outbox, maintained
by the service, with no poller to write, no outbox table to garbage-collect, and no
extra write capacity consumed. Choosing option 2 on DynamoDB is re-implementing a
feature you are already paying for.

Option 3 gives, out of the box: at-least-once delivery, ordering guaranteed per item
key within a shard, automatic retry, a failure destination, and 24 hours of replay. It
also enforces the property that makes the whole architecture tractable — **only one
component in the system may write to OpenSearch**. The application code cannot get this
wrong, because the application code is not involved.

**Accepted trade-off.** Stream retention is fixed at 24 hours and cannot be extended.
A pipeline outage longer than that loses records permanently. This is not mitigated by
the stream; it is mitigated by the reindex pipeline (3.9), and it is recorded as
[Risk R-7](./05-risks.md).

---

## 3.3 DynamoDB Streams over Kinesis Data Streams

**Options**
1. **DynamoDB Streams** — native, 24 h retention, max 2 consumers per shard.
2. **Kinesis Data Streams for DynamoDB** — up to 365 days retention, many consumers,
   integrates with Firehose and Managed Service for Apache Flink.

**Choice: DynamoDB Streams (option 1).**

**Rationale.**
The deciding factor is delivery semantics, not retention. Kinesis Data Streams for
DynamoDB does not guarantee that records arrive in the order the changes were made, and
may deliver duplicates — consumers are expected to handle both. DynamoDB Streams
guarantees that changes to a **given item key** appear in the correct order within a
shard, which is precisely the guarantee an indexer wants.

This system has exactly one consumer, so the two-consumer ceiling is not binding, and
24 hours of retention is sufficient given that a full reindex from S3 export exists as
the recovery path for anything longer.

**When this decision flips:** if a second and third consumer appear — an analytics
pipeline, an audit log, a cache invalidator — the two-consumer limit becomes real and
Kinesis Data Streams becomes correct. That migration is a table-level configuration
change plus per-consumer idempotency and reordering logic, not a redesign. Recorded in
[Future Work](./06-future-work.md).

---

## 3.4 Custom indexer Lambda over Zero-ETL

**Options**
1. **DynamoDB Zero-ETL integration with Amazon OpenSearch Service** — a managed
   OpenSearch Ingestion pipeline replicates the table into an index with no code.
2. **Custom indexer Lambda** on the stream.

**Choice: Custom indexer Lambda (option 2), with Zero-ETL as a documented alternative.**

**Rationale.**
Zero-ETL is the better answer when the search document is essentially the table item.
It is not, here. The indexer performs work that has to live somewhere:

- **Projection.** Internal attributes — `sellerInternalNotes`, `costPrice`, `margin` —
  must never reach a read model that serves anonymous buyers. A search index is a
  publication surface, and an explicit allow-list of fields is a security control.
- **Derivation.** `priceBucket` (`0-500`, `500-2000`, …) is computed at index time so
  that price faceting is a cheap `terms` aggregation instead of an expensive
  `range` aggregation on every query.
- **Denormalisation.** `sellerDisplayName` and `categoryPath` are joined in at index
  time, because the read model must be self-contained — OpenSearch has no joins.
- **Versioning.** External versioning (3.5) requires setting `version` and
  `version_type` per document, which needs control of the bulk request.
- **Soft-delete translation.** `status: DELETED` in the table must become a document
  *deletion* in the index (3.13), not a document with a status field.

Zero-ETL also runs on OpenSearch Ingestion Compute Units, which carry a floor cost
whether or not there is traffic — the same objection raised against Serverless in 3.7.

The Lambda costs a few hundred lines of code and gives complete control of the mapping
between the two models. That mapping is the interesting part of a CQRS system, and it
is not something to delegate.

---

## 3.5 Out-of-order updates — external versioning

**Options**
1. **Last-write-wins by arrival order** — index whatever arrives, whenever it arrives.
2. **Read-then-compare in the indexer** — `GET` the document, compare timestamps, write
   only if newer.
3. **OpenSearch external versioning** — attach the DynamoDB `version` to every write and
   let OpenSearch reject non-monotonic updates.

**Choice: External versioning (option 3).**

**Rationale — and first, why the problem exists at all.**
DynamoDB Streams guarantee ordering per item key *within a shard*. That guarantee is
weaker than it sounds once the pipeline is configured for throughput:

- `ParallelizationFactor: 10` runs ten concurrent batches per shard. Two updates to the
  same product landing in different concurrent batches have **no ordering guarantee**
  between them.
- `BisectBatchOnFunctionError` splits and retries a failing batch. The retried half can
  be indexed *after* a batch that was created later.
- A table partition split changes the shard topology; records for one key may span the
  parent and child shards.

So a stale v6 landing after a fresh v7 is not a theoretical race — it is a normal
consequence of a correctly-tuned pipeline. Option 1 therefore silently corrupts the read
model, and the corruption is invisible: no error, no alarm, just a wrong price on a
product page until the next write or reindex.

Option 2 is a check-then-act race. Between the `GET` and the `PUT`, a concurrent
invocation can write. It also doubles the request count against the cluster and adds a
round-trip to every document.

Option 3 pushes the comparison to where it can be made atomically — inside OpenSearch,
on the shard that owns the document:

```json
{ "index": { "_index": "products", "_id": "a1b2c3",
             "version": 7, "version_type": "external" } }
```

OpenSearch stores the version and rejects any subsequent write whose external version is
**not strictly greater**. A late v6 returns `409 version_conflict_engine_exception` and
changes nothing. This works because DynamoDB's optimistic-locking `version` attribute
(3.1 of the data model) is already monotonic per item — the write model's concurrency
control is reused as the read model's ordering key, at zero additional cost.

**Operational consequence.** A `409` is a **success**, not a failure. The indexer counts
`IndexVersionConflicts` as a metric and must not route these records to the DLQ.
A steady low rate is healthy evidence the mechanism is working; a sudden spike means
something is replaying stale data and is worth alarming on.

---

## 3.6 Poison records — partial batch response + bisect

**Options**
1. **Fail the whole batch** — throw, let the event source mapping retry.
2. **Catch and swallow** — `try/except`, log, continue.
3. **`ReportBatchItemFailures` + `BisectBatchOnFunctionError` + bounded retries +
   `OnFailure` DLQ.**

**Choice: Option 3.**

**Rationale.**
Option 1 produces the worst failure mode in stream processing. A stream shard is
processed **in order**; if a batch fails, the same batch is retried, and the shard makes
no progress. One malformed record — a `NaN` price, an attribute the mapping rejects, a
document over the field limit — halts **every subsequent update on that shard**. Sync
lag climbs without bound, `IteratorAge` grows until it exceeds the 24-hour retention, and
records are lost. A single bad record takes out the pipeline.

Option 2 avoids the stall by discarding data. The record is gone, the index is
permanently inconsistent for that product, and the only signal is a log line nobody
reads. It converts a loud failure into a silent one, which is strictly worse.

Option 3 handles each concern with the mechanism designed for it:

| Mechanism | What it prevents |
|---|---|
| `ReportBatchItemFailures` | The 99 good records in a batch of 100 are committed; only the failing sequence numbers are retried. No redundant reprocessing, no duplicate side effects. |
| `BisectBatchOnFunctionError` | Binary-searches the batch to isolate exactly which record is poison, instead of retrying all 100 forever. |
| `MaximumRetryAttempts: 3` | Bounds the retry loop so a permanently-bad record cannot consume the pipeline. |
| `MaximumRecordAgeInSeconds: 3600` | An age ceiling — even a record that keeps failing for other reasons is eventually discarded from the retry loop rather than blocking. |
| `OnFailure` → SQS DLQ | The exhausted record is **preserved with its sequence number** for inspection and replay. Nothing is silently lost. |
| Alarm on DLQ depth ≥ 1 | The failure is loud within one alarm period. |

The DLQ message contains stream metadata rather than the record body, so replay reads
the item back from DynamoDB — which is correct, because by replay time the current item
state is the state that should be indexed anyway.

---

## 3.7 OpenSearch provisioned over Serverless

**Options**
1. **Amazon OpenSearch Serverless** — capacity in OpenSearch Compute Units, auto-scaled.
2. **Provisioned `t3.small.search`, single node** — Free Tier eligible.
3. **Provisioned production topology** — 3 dedicated master nodes + 3 data nodes across
   three Availability Zones.

**Choice: `t3.small.search` for the demo (option 2); option 3 documented as the
production topology.**

**Rationale.**
Serverless is architecturally attractive — no node sizing, no shard rebalancing, no
version upgrades — and for a bursty workload it is the right default. It is rejected
here on cost, and the reasoning matters more than the number: **Serverless bills a
minimum OCU allocation continuously, whether or not a single query is served.** For a
graduation project with intermittent demo traffic, the floor price dominates the bill
entirely, and it is a floor that cannot be optimised away by caching, by scaling to
zero, or by any other change to the architecture. Paying a standing monthly charge for a
service that is idle 99% of the time fails the Cost Optimization pillar on its own terms.

`t3.small.search` is covered by the AWS Free Tier at 750 instance-hours per month for
the first 12 months, plus 10 GB of EBS storage — enough for one node running
continuously, and enough for a ~4 GB index.

**What is knowingly given up, and why it is acceptable:** a single node is single-AZ,
has no replica shards, and is a single point of failure. That is tolerable *specifically
because of the CQRS shape*: OpenSearch holds no authoritative data. Losing the node
loses search, not the catalog. Writes continue, sellers continue to manage their
products, and the read model is rebuilt by the reindex pipeline. This is the concrete
payoff of the "read model is disposable" property asserted in §2.1 — the availability
argument is architectural, not wishful.

The production topology is specified in the [Appendix](./07-appendix.md) so the
migration path is a domain configuration change, not a redesign.

---

## 3.8 OpenSearch inside a VPC — and avoiding the NAT Gateway trap

**Options**
1. **Public domain endpoint** secured by an IAM access policy and fine-grained access
   control.
2. **VPC domain in private subnets**, Lambda attached to the VPC.

**Choice: VPC in private subnets (option 2).**

**Rationale.**
Publicly-reachable search clusters are one of the most reliably exploited misconfigurations
in cloud infrastructure. An IAM access policy is a real control, but it is a *single*
control on an endpoint that is resolvable and reachable from the entire internet; a
policy mistake, an over-broad principal, or a leaked credential is immediately
exploitable. A VPC endpoint removes network reachability altogether, so the IAM policy
becomes the second layer of defence rather than the only one.

**The cost trap this creates, and how it is avoided.** Attaching a Lambda to a VPC
removes its default internet route. The reflexive fix is a NAT Gateway, which costs
roughly \$32/month per AZ plus data processing — for a demo, the NAT Gateway would be
larger than every other line item on the bill combined, and it is pure waste here
because the function needs no internet access at all. It needs AWS service endpoints.
So:

| Destination | Mechanism | Cost |
|---|---|---|
| OpenSearch | Same VPC, private subnet | Free |
| DynamoDB | **Gateway VPC Endpoint** | **Free** |
| S3 | **Gateway VPC Endpoint** | **Free** |
| Secrets Manager, KMS (if reached from in-VPC functions) | Interface VPC Endpoints | ~\$7.20/month each |
| Internet | **None — deliberately no route** | \$0 |

The Indexer and Search Lambdas need only OpenSearch, DynamoDB, and S3, all of which are
reachable at zero networking cost. **No NAT Gateway is provisioned**, and the absence is
deliberate rather than accidental — a security posture and a cost decision that happen
to point the same way.

---

## 3.9 Reindex source — S3 Export over Scan

**Options**
1. **Parallel `Scan`** of the DynamoDB table.
2. **`ExportTableToPointInTime`** to S3, then read the export.

**Choice: S3 Export (option 2).**

**Rationale.**
A `Scan` reads every item and **consumes read capacity for all of them**. Scanning 2 M
items of ~2 KB is roughly 4 GB of reads. On on-demand billing that is a real charge; on
provisioned capacity it competes directly with production traffic and will throttle live
requests. Reindexing is an operational task that must never degrade the customer-facing
write path, and a mechanism whose cost scales with table size and whose side effect is
throttling the production table fails that requirement structurally — the pagination,
segment tuning, and backoff logic it also demands are secondary annoyances.

`ExportTableToPointInTime` reads from the continuous backup maintained by
point-in-time recovery. It consumes **zero table read capacity**, has no effect on live
throughput, and writes DynamoDB-JSON objects to S3 that can be processed in parallel by
as many workers as desired. The export itself is billed per GB of table data, which for
this catalog is cents.

The output shape is also better suited to the job: many independent S3 objects map
directly onto a Step Functions **Distributed Map**, giving parallel bulk indexing with
per-item failure isolation and no custom sharding logic.

**Prerequisite:** PITR must be enabled on the table. It is — and it is worth noting that
this decision makes a durability feature do double duty as a performance feature.

---

## 3.10 Zero-downtime reindex — index aliases

**Options**
1. **Reindex in place** — delete and rebuild the `products` index.
2. **Build `products-v{N}` alongside, then atomically swap the `products` alias.**

**Choice: Alias swap (option 2).**

**Rationale.**
Option 1 makes search return partial results for the duration of the rebuild and no
results at the moment of deletion — for 2 M documents that is minutes of a visibly
broken catalog, and there is no way to abort once started.

Under option 2, all application traffic addresses the alias `products`, never a concrete
index. The new index is built, verified against the export manifest's document count,
and only then does a single `POST /_aliases` call remove the old index from the alias
and add the new one. That call is **atomic** — no request ever sees zero indices or two.

It also delivers three things that were not the primary goal:

- **Instant rollback.** The previous index is retained for seven days; reverting is one
  alias action, seconds rather than another full reindex.
- **Mapping changes become routine.** Analyzers and field types are largely immutable in
  an existing index. Versioned indices make a mapping change an ordinary deployment
  instead of an outage.
- **Verification before exposure.** The new index can be queried directly by its concrete
  name and validated before any user traffic reaches it.

---

## 3.11 Read-your-writes — who is allowed to see stale data

**Options**
1. **Block the write** until the document is confirmed in the index.
2. **Client polls** the search API until its change appears.
3. **Route reads by audience** — owner reads hit DynamoDB, public search hits OpenSearch.

**Choice: Route reads by audience (option 3).**

**Rationale.**
This is the question every CQRS design has to answer explicitly, and answering it with
"eventual consistency, the user will cope" is how CQRS systems acquire their reputation.
The concrete failure: a seller edits a price, is redirected to their dashboard, and sees
the old price. They edit again. Now there are two writes, possible version conflicts, and
a user who does not trust the product.

Option 1 destroys the decoupling that motivated the architecture. Write latency becomes
a function of search cluster health; an OpenSearch incident becomes a product-creation
outage. It also cannot be implemented honestly — confirming a document is *searchable*
requires waiting for a refresh, so "blocking until indexed" means blocking for the
refresh interval on every write.

Option 2 pushes an internal implementation detail into every client and burns request
volume on polling.

Option 3 observes that the two audiences have genuinely different requirements:

| Audience | Query | Store | Consistency | Justification |
|---|---|---|---|---|
| Seller | "my products", "this product" | DynamoDB (`GetItem` / GSI1) | Strong | Sees their own writes immediately. Bounded by key — DynamoDB's strength. |
| Buyer | full-text, faceted search | OpenSearch | Eventual, p95 < 5 s | Cannot tell a 3-second-old catalog from a current one. |

The seller's access pattern is a keyed lookup, which is exactly what DynamoDB does best —
so this costs nothing and adds no component. It is not a workaround; it is routing each
query to the store that suits it, which is what CQRS is for.

The staleness bound is published rather than hidden: search responses carry an
`X-Index-Lag-Ms` header, and the contract (p95 < 5 s) is an [SLO in §1.3](./01-requirements.md#13-non-functional-requirements)
with an alarm behind it.

---

## 3.12 Search Lambda over direct API Gateway → OpenSearch integration

**Options**
1. **API Gateway AWS-service integration** proxying requests straight to the domain.
2. **Search Lambda** translating a constrained public contract into OpenSearch DSL.

**Choice: Search Lambda (option 2).**

**Rationale.**
Option 1 removes a component and a few milliseconds, and exposes the query language of
the datastore to the internet. OpenSearch DSL is expressive enough to be an attack
surface in three separate ways:

- **Denial of service.** A deeply-nested `bool` query, an unbounded `terms` aggregation
  with high cardinality, a wildcard leading with `*`, or `from: 999999` can each consume
  disproportionate cluster CPU and memory. On a single-node domain, one crafted query
  can degrade search for everyone.
- **Data exfiltration.** Unless every field is explicitly excluded, a client that
  controls `_source` and can express arbitrary queries can enumerate the index — including
  any field whose exclusion was forgotten. The Lambda's allow-list makes exposure opt-in.
- **Coupling.** The public API contract becomes the OpenSearch version's DSL. Upgrading
  the cluster or changing the mapping becomes a breaking API change.

The Lambda accepts a small, validated contract — `q`, `category`, `brand`, `priceMin`,
`priceMax`, `minRating`, `inStock`, `sort`, `cursor` — and constructs the DSL itself,
enforcing a maximum page size, an aggregation allow-list, `_source` field filtering, and
a `timeout` on every query. It also emits per-query metrics and shapes the response so
the storefront is not coupled to OpenSearch's hit envelope.

The added latency is a Lambda invocation on a warm container: single-digit milliseconds
against a p95 budget of 300 ms.

---

## 3.13 Deletes — soft delete over hard delete

**Options**
1. **Hard delete** — `DeleteItem`, and the `REMOVE` stream record drives an index delete.
2. **Soft delete** — set `status: DELETED`, and the indexer issues an index delete.

**Choice: Soft delete (option 2).**

**Rationale.**
A hard delete is a `REMOVE` stream event, which carries `OldImage` but no new image and
no `version` — so the external-versioning mechanism of 3.5 has nothing to compare
against, and the ordering guarantee that protects every other write does not protect
deletes. A delete racing an in-flight update can be applied first and then undone by the
late update, resurrecting a deleted product in the search index with no error anywhere.

A soft delete is an ordinary `MODIFY` with an incremented `version`, so it participates
in the same monotonic ordering as every other change: the indexer sees
`status: DELETED`, version 8, and issues a document delete tagged with version 8. A late
version 7 update is rejected by the same mechanism that rejects any stale write. The
product cannot come back.

It also buys recoverability — an accidental deletion is an attribute change, reversible
by a normal versioned update — and it keeps a row for audit. DynamoDB TTL performs the
physical purge after a retention window, and the resulting `REMOVE` event is ignored by
the indexer because the document is already gone from the index.

---

## 3.14 Deep pagination — `search_after` over `from`/`size`

**Options**
1. **`from` / `size`** — classic offset pagination.
2. **`search_after`** — cursor pagination on a stable sort key.

**Choice: `search_after` (option 2), with a hard page-depth cap.**

**Rationale.**
`from`/`size` is not merely slow at depth, it fails: OpenSearch enforces
`index.max_result_window` (10,000 by default), and requests past it are rejected. Raising
the window makes it worse rather than better, because serving `from: 50000` requires every
shard to collect and sort 50,000 + `size` hits and the coordinating node to merge them —
memory and CPU that scale with the offset, on every request. It is a documented way to
destabilise a cluster, and it is trivially reachable by a crawler incrementing a page
parameter.

`search_after` carries the sort values of the last hit as an opaque cursor, so each page
costs the same as the first regardless of depth. It also fixes a correctness bug that
offset pagination has and users notice: with a live index, a document inserted while the
user is paging shifts every subsequent offset, so items are skipped or repeated between
pages. A cursor anchored to sort values does not drift.

The sort must be **total** for the cursor to be stable, so every sort specification is
suffixed with the document `_id` as a tie-breaker. The cursor is base64-encoded and
opaque, and the API additionally caps page depth — beyond a few hundred pages the correct
answer is a better filter, not more pages.

---

[← README](../README.md) | [← Architecture](./02-architecture.md) | **Design Decisions** | [Next: Well-Architected →](./04-well-architected.md)
