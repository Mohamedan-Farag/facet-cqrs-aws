[← README](../README.md) | [← Requirements](./01-requirements.md) | **Architecture** | [Next: Design Decisions →](./03-design-decisions.md)

# 2. Architecture

![Architecture](../diagram/architecture.png)

## 2.1 The shape of the system

Facet is a **CQRS** architecture. The system is split down the middle:

```
                    ┌──────────────── COMMAND SIDE ────────────────┐
   Sellers ────────►│  API → Lambda → DynamoDB (source of truth)   │
                    └──────────────────────┬───────────────────────┘
                                           │
                                  DynamoDB Streams (CDC)
                                           │
                    ┌──────────────────────▼───────────────────────┐
                    │  SYNC PIPELINE — Indexer Lambda → OpenSearch  │
                    └──────────────────────┬───────────────────────┘
                                           │
                    ┌──────────────────────▼───────────────────────┐
   Buyers  ────────►│  API → Lambda → OpenSearch (read model)      │
                    └──────────────── QUERY SIDE ──────────────────┘
```

Two properties follow from this shape, and everything else in the design defends them:

1. **DynamoDB is the only source of truth.** Nothing writes to OpenSearch except the
   sync pipeline. There is no code path that writes to both stores.
2. **The read model is disposable.** OpenSearch can be deleted entirely and rebuilt
   from DynamoDB. That single property is what turns "we have two databases" from a
   liability into a manageable design — it collapses a whole class of failure modes
   into one runbook.

## 2.2 Components

### ① Edge — CloudFront + AWS WAF + S3

The single public entry point. The storefront SPA is served from a private S3 bucket
reachable only through CloudFront Origin Access Control; `/api/*` behaviours forward to
API Gateway. AWS WAF is attached at the CloudFront distribution with managed rule
groups (Core Rule Set, Known Bad Inputs) plus a rate-based rule, so abusive traffic is
rejected at the edge before it can reach a Lambda function or, worse, an expensive
OpenSearch query.

Search responses for anonymous traffic are cached at the edge with a short TTL keyed on
the normalised query string.

### ② Auth — Amazon Cognito

A Cognito User Pool handles seller sign-up, sign-in, and MFA, and issues JWTs that API
Gateway validates on every write request. Two groups exist: `sellers` and `admins`
(admins may trigger a reindex). Buyer search is public — an unauthenticated route
protected by WAF rate limiting rather than by tokens.

### ③ Command side — API Gateway + Command Lambda + DynamoDB

`POST /products`, `PUT /products/{id}`, `DELETE /products/{id}` and the seller-scoped
reads are backed by a single zip Lambda. It validates input, enforces that the caller
owns the product, and writes to DynamoDB with an optimistic-locking condition:

```
UpdateExpression:      SET #v = :newVersion, ...
ConditionExpression:   attribute_not_exists(PK) OR #v = :expectedVersion
```

A failed condition returns `409 Conflict`. The `version` attribute that this maintains
is not incidental bookkeeping — it becomes the document version in OpenSearch and is
what makes out-of-order indexing safe ([DD 5](./03-design-decisions.md#35-out-of-order-updates--external-versioning)).

**Table: `facet-products`** (single table, on-demand billing, PITR enabled)

| Attribute | Example | Purpose |
|---|---|---|
| `PK` (HASH) | `PRODUCT#a1b2c3` | Item identity |
| `SK` (RANGE) | `META` | Room for future child items (variants, media) |
| `GSI1PK` | `SELLER#42` | Seller-scoped listing |
| `GSI1SK` | `UPDATED#2026-08-25T10:00:00Z` | Most-recently-updated first |
| `version` | `7` | Monotonic; drives optimistic locking **and** external versioning |
| `status` | `ACTIVE` / `DRAFT` / `DELETED` | Soft delete — see [DD 13](./03-design-decisions.md#313-deletes--soft-delete-over-hard-delete) |
| `title`, `description`, `category`, `brand`, `price`, `rating`, `stock` | — | Indexed product attributes |

**Stream:** enabled with `NEW_AND_OLD_IMAGES`.

### ④ Sync pipeline — DynamoDB Streams + Indexer Lambda

This is the part of the system worth studying. The event source mapping is configured
deliberately, and every parameter defends a specific failure mode:

| Setting | Value | Why |
|---|---|---|
| `StartingPosition` | `TRIM_HORIZON` | Pick up everything present when the pipeline first starts |
| `BatchSize` | `100` | One `_bulk` request per invocation; ~200 KB payload |
| `MaximumBatchingWindowInSeconds` | `5` | Trades a little latency for far fewer invocations at low write rates |
| `ParallelizationFactor` | `10` | Ten concurrent batches per shard — drains the bulk-import burst |
| `BisectBatchOnFunctionError` | `true` | Binary-searches a failing batch to isolate the poison record |
| `MaximumRetryAttempts` | `3` | Bounded retries |
| `MaximumRecordAgeInSeconds` | `3600` | Stops a record being retried forever |
| `FunctionResponseTypes` | `[ReportBatchItemFailures]` | Partial batch response — only failed records are retried |
| `DestinationConfig.OnFailure` | Indexer DLQ (SQS) | Exhausted records are preserved, not dropped |

The Indexer Lambda transforms each DynamoDB item into a search document — flattening
nested attributes, computing a `priceBucket` for faceting, denormalising the seller
display name, and stripping internal fields that must never reach the read model — then
issues one `_bulk` request with **external versioning**:

```json
{ "index": { "_index": "products", "_id": "a1b2c3",
             "version": 7, "version_type": "external" } }
```

OpenSearch rejects any document whose external version is not greater than the stored
one, atomically and server-side. A `409` in the bulk response therefore means *"a newer
version already landed"* — an expected outcome, counted as a metric, not an error.

### ⑤ Query side — Search Lambda + OpenSearch

`GET /search?q=...&category=...&priceMin=...&sort=...` is handled by a Lambda deployed
into the VPC. It translates a **constrained public query contract** into OpenSearch
DSL — clients never send raw DSL ([DD 12](./03-design-decisions.md#312-search-lambda-over-direct-api-gateway--opensearch-integration)).
Every response carries facet aggregations alongside the hits.

The OpenSearch domain sits in private subnets with fine-grained access control and
SigV4-signed requests. Reads and writes target the **`products` alias**, never a
concrete index name — which is what makes the zero-downtime reindex possible.

### ⑥ Reindex pipeline — Step Functions + S3 Export

A Standard workflow, triggered manually by an admin or by EventBridge Scheduler:

1. `DynamoDB:ExportTableToPointInTime` → S3, in DynamoDB JSON. **Consumes zero table
   read capacity** — production traffic is untouched.
2. Create `products-v{N}` with the current mapping and analyzer configuration.
3. A **Distributed Map** state fans out across the exported S3 objects; each child
   invocation bulk-indexes its slice.
4. Verify the document count against the export manifest.
5. **Atomically swap the `products` alias** from `v{N-1}` to `v{N}` in a single
   OpenSearch alias action. Readers never observe a half-built index.
6. Retain the previous index for a grace period so rollback is one alias action away.

### ⑦ Observability & security

| Concern | Mechanism |
|---|---|
| Sync lag | Custom EMF metric `SyncLagMs` (`now − ApproximateCreationDateTime`) + `IteratorAge` alarm at 60 s |
| Poison records | DLQ depth alarm; every DLQ message carries the failing sequence number |
| Version conflicts | `IndexVersionConflicts` counter — a healthy non-zero number, alarmed only on a sharp spike |
| Traces | X-Ray across API Gateway → Lambda → DynamoDB / OpenSearch |
| Cluster health | OpenSearch `ClusterStatus`, `FreeStorageSpace`, `JVMMemoryPressure` alarms |
| Encryption at rest | KMS CMK on DynamoDB, S3, SQS, and the OpenSearch domain |
| Encryption in transit | TLS everywhere; node-to-node encryption enabled on the domain |
| Access | Least-privilege IAM role per Lambda; OpenSearch fine-grained access control |

## 2.3 Request flows

### Flow A — Write and propagate (the happy path)

```
1. Seller  → CloudFront → WAF → API Gateway (Cognito JWT)
2.         → Command Lambda
3.         → DynamoDB conditional write (version 6 → 7)   ── 201/200 returned here
   ─────────────────────────── seller's request ends ──────────────────────────
4. DynamoDB Streams emits the NEW_AND_OLD_IMAGES record
5.         → Indexer Lambda (batch of ≤100, ≤5 s window)
6.         → transform → OpenSearch _bulk, version=7, version_type=external
7.         → emit SyncLagMs
   Buyer-visible within p95 5 s.
```

### Flow B — Search

```
1. Buyer   → CloudFront (edge cache) → WAF → API Gateway
2.         → Search Lambda (in VPC)
3.         → validate + translate to OpenSearch DSL (bounded page size, search_after)
4.         → OpenSearch query against the `products` alias
5.         → hits + facet aggregations → response (Cache-Control: max-age=30)
```

### Flow C — Seller reads their own catalog (bypasses the read model)

```
1. Seller  → API Gateway (Cognito JWT) → Command Lambda
2.         → DynamoDB GetItem (strongly consistent) or GSI1 Query
   No OpenSearch involvement — sellers never see their own stale data.
```

### Flow D — Poison record

```
5. Indexer Lambda fails on one malformed record in a batch of 100
6. ReportBatchItemFailures returns only that record's sequence number
   → the other 99 are committed and never retried
7. BisectBatchOnFunctionError narrows the retry batch
8. After MaximumRetryAttempts=3 → record delivered to the Indexer DLQ
9. CloudWatch alarm on DLQ depth → operator inspects and replays
   The shard never stalls.
```

### Flow E — Zero-downtime reindex

```
1. Admin (or schedule) → Step Functions
2. ExportTableToPointInTime → S3            (0 table RCU consumed)
3. Create index products-v4 with mapping
4. Distributed Map over export shards → _bulk index
5. Verify count against the manifest
6. POST /_aliases  { remove: products-v3, add: products-v4 }   ← atomic
7. Keep v3 for 7 days for one-action rollback
```

---

[← README](../README.md) | [← Requirements](./01-requirements.md) | **Architecture** | [Next: Design Decisions →](./03-design-decisions.md)
