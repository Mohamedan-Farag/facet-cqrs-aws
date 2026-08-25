[← README](../README.md) | [← Future Work](./06-future-work.md) | **Appendix** | [Next: Cost Estimate →](./08-cost-estimate.md)

# 7. Appendix

## 7.1 Service inventory

| Layer | Service | Role in this system |
|---|---|---|
| Edge | Amazon CloudFront | Single public entry point; serves the SPA and caches anonymous search responses |
| Edge | AWS WAF | Managed rule groups + rate-based rule, attached at the distribution |
| Edge | Amazon S3 (`web`) | Private bucket holding the storefront SPA, reachable only via OAC |
| Auth | Amazon Cognito User Pool | Seller sign-up/sign-in, MFA, JWTs; groups `sellers` and `admins` |
| API | Amazon API Gateway (REST) | `/products*` (authorised) and `/search` (public) |
| Command | AWS Lambda — `command` | Validation, ownership check, optimistic-locked writes |
| Data | Amazon DynamoDB — `facet-products` | **Source of truth.** On-demand, PITR on, stream `NEW_AND_OLD_IMAGES` |
| CDC | DynamoDB Streams | Change data capture feeding the read model |
| Sync | AWS Lambda — `indexer` | Transform + `_bulk` index with external versioning |
| Sync | Amazon SQS — `indexer-dlq` | `OnFailure` destination for exhausted stream records |
| Read model | Amazon OpenSearch Service | `t3.small.search` in private subnets; alias `products` |
| Query | AWS Lambda — `search` | Translates the public query contract into OpenSearch DSL |
| Reindex | AWS Step Functions (Standard) | Export → build → verify → alias swap |
| Reindex | Amazon S3 (`exports`) | `ExportTableToPointInTime` output |
| Network | VPC + Gateway Endpoints | Private subnets; free gateway endpoints for DynamoDB and S3; **no NAT Gateway** |
| Security | AWS KMS | Customer-managed keys for DynamoDB, S3, SQS, OpenSearch |
| Security | AWS IAM | Least-privilege role per function |
| Security | AWS CloudTrail | Multi-region trail with log-file integrity validation |
| Observability | Amazon CloudWatch | Logs, metrics, EMF custom metrics, alarms, dashboard |
| Observability | AWS X-Ray | Distributed tracing across both paths |
| IaC | AWS SAM / CloudFormation | Reproducible deploy and one-command teardown |

## 7.2 Public API contract

Clients never send OpenSearch DSL ([DD 3.12](./03-design-decisions.md#312-search-lambda-over-direct-api-gateway--opensearch-integration)).

### `POST /products` · `PUT /products/{id}` — seller, JWT required

```jsonc
{
  "title":       "14-inch Laptop, 16 GB RAM",
  "description": "…",
  "category":    "electronics/computers/laptops",
  "brand":       "Acme",
  "price":       18500.00,
  "currency":    "EGP",
  "stock":       12,
  "version":     6            // required on PUT; omitted on POST
}
```

`409 Conflict` if `version` does not match the stored value — the client re-reads and
retries. Response echoes the new `version`.

### `GET /search` — public

| Parameter | Type | Constraint |
|---|---|---|
| `q` | string | Free text, ≤ 200 chars |
| `category` | string | Path prefix match |
| `brand` | string[] | ≤ 10 values |
| `priceMin` / `priceMax` | number | — |
| `minRating` | number | 0–5 |
| `inStock` | boolean | — |
| `sort` | enum | `relevance` \| `price_asc` \| `price_desc` \| `rating_desc` |
| `limit` | integer | 1–50, default 20 |
| `cursor` | string | Opaque base64 `search_after` cursor |

```jsonc
{
  "hits":   [ { "id": "a1b2c3", "title": "…", "price": 18500, "score": 8.42 } ],
  "facets": {
    "category":    [ { "key": "laptops", "count": 412 } ],
    "brand":       [ { "key": "Acme",    "count": 87  } ],
    "priceBucket": [ { "key": "10000-25000", "count": 233 } ]
  },
  "total":      412,
  "nextCursor": "eyJzb3J0IjpbMTg1MDAsImExYjJjMyJdfQ=="
}
```

Response header `X-Index-Lag-Ms` publishes the current read-model staleness, making the
eventual-consistency contract from [DD 3.11](./03-design-decisions.md#311-read-your-writes--who-is-allowed-to-see-stale-data)
observable to clients rather than implicit.

## 7.3 Index mapping (abridged)

```jsonc
{
  "settings": {
    "index": { "number_of_shards": 1, "number_of_replicas": 0,
               "refresh_interval": "5s" },        // demo: single node, no replicas
    "analysis": {
      "analyzer": {
        "product_en": { "type": "custom", "tokenizer": "standard",
                        "filter": ["lowercase", "asciifolding", "product_synonyms",
                                   "english_stemmer"] },
        "product_ar": { "type": "custom", "tokenizer": "standard",
                        "filter": ["lowercase", "decimal_digit", "arabic_normalization",
                                   "arabic_stemmer"] }
      }
    }
  },
  "mappings": {
    "dynamic": "strict",                          // an unmapped field is a loud error
    "properties": {
      "title":       { "type": "text", "analyzer": "product_en",
                       "fields": { "ar": { "type": "text", "analyzer": "product_ar" },
                                   "raw": { "type": "keyword" } } },
      "description": { "type": "text", "analyzer": "product_en" },
      "category":    { "type": "keyword" },
      "categoryPath":{ "type": "keyword" },       // denormalised at index time
      "brand":       { "type": "keyword" },
      "price":       { "type": "scaled_float", "scaling_factor": 100 },
      "priceBucket": { "type": "keyword" },       // derived at index time — cheap faceting
      "rating":      { "type": "half_float" },
      "inStock":     { "type": "boolean" },
      "sellerId":         { "type": "keyword" },
      "sellerDisplayName":{ "type": "keyword" },  // denormalised — OpenSearch has no joins
      "updatedAt":   { "type": "date" }
    }
  }
}
```

`"dynamic": "strict"` is deliberate. An attribute added to DynamoDB and not to this
mapping causes a **visible** indexing failure that lands in the DLQ, rather than being
silently auto-mapped with a guessed type that then breaks the next reindex. It converts a
schema-drift bug into an alarm.

Fields absent from this mapping — `costPrice`, `margin`, `sellerInternalNotes` — are
absent because the indexer projects an allow-list ([Risk R-11](./05-risks.md)).

## 7.4 Event source mapping configuration

```yaml
StartingPosition:                 TRIM_HORIZON
BatchSize:                        100
MaximumBatchingWindowInSeconds:   5
ParallelizationFactor:            10
BisectBatchOnFunctionError:       true
MaximumRetryAttempts:             3
MaximumRecordAgeInSeconds:        3600
FunctionResponseTypes:            [ ReportBatchItemFailures ]
DestinationConfig:
  OnFailure:
    Destination: !GetAtt IndexerDLQ.Arn
```

Every line is a decision, justified in [DD 3.6](./03-design-decisions.md#36-poison-records--partial-batch-response--bisect).

## 7.5 Production topology

The demo runs one Free-Tier node. Production sizing for the scale anchor, so the gap is
documented rather than discovered:

| Component | Demo | Production |
|---|---|---|
| Data nodes | 1 × `t3.small.search` | 3 × `r6g.large.search`, one per AZ |
| Dedicated master nodes | none | 3 × `m6g.large.search` across 3 AZs |
| Shards / replicas | 1 shard, 0 replicas | 3 shards, 1 replica |
| Availability zones | 1 | 3 |
| Storage | 10 GB gp3 | 100 GB gp3 per node |
| Automated snapshots | hourly (managed) | hourly + manual to S3 before each reindex |

Migrating is a domain configuration change. The indexer, reindex pipeline, and Search
Lambda are unchanged — the alias indirection and the disposable read model are what make
that true.

## 7.6 Key CloudWatch alarms

| Alarm | Threshold | Why it matters |
|---|---|---|
| `IteratorAge` (indexer ESM) | > 60 s for 5 min | **The** health metric of a CDC pipeline. Stream retention is 24 h; this alarm leaves ~23 h of margin before data loss ([R-7](./05-risks.md)). |
| DLQ `ApproximateNumberOfMessagesVisible` | ≥ 1 | A record was permanently rejected — silent drift starts here. |
| `SyncLagMs` p99 (custom EMF) | > 30 s | Measures end-to-end staleness, not just consumer position. |
| `IndexVersionConflicts` | > 3× 7-day baseline | Steady conflicts are healthy; a spike means stale data is being replayed. |
| Lambda `Errors` (any function) | > 1% of invocations | — |
| OpenSearch `ClusterStatus.red` | ≥ 1 | Shards unassigned; search results are incomplete. |
| OpenSearch `FreeStorageSpace` | < 20% | An index goes read-only when disk fills. |
| OpenSearch `JVMMemoryPressure` | > 80% | Precedes GC thrashing and node instability. |
| API Gateway `5XXError` | > 1% | — |
| AWS Budgets | \$10 / \$25 | [Cost guardrail](./08-cost-estimate.md#85-cost-guardrails-in-the-build). |

## 7.7 Runbooks

### Sync lag alarm fires

1. Check `IteratorAge` — rising steadily, or a step change?
2. Check indexer `Errors` and `Throttles`. Throttling → raise reserved concurrency.
3. Check OpenSearch `JVMMemoryPressure` and `ClusterStatus` — a struggling cluster
   backs the pipeline up.
4. Bulk import in progress? Lag is expected and self-healing; confirm it is draining.
5. If lag approaches **12 hours**, escalate: stream retention is 24 h and the recovery
   window is closing.

### DLQ has messages

1. Read the message — it carries the stream shard and sequence number, not the item body.
2. Fetch the current item from DynamoDB by key. **Current state is what should be
   indexed**, not the historical change, so a replay is always correct.
3. Reproduce the transform locally. Usual causes: a mapping conflict, a field over the
   length limit, an unmapped attribute rejected by `dynamic: strict`.
4. Fix forward — update the transform, or update the mapping and reindex.
5. Re-drive the queue, then confirm the DLQ is empty and the document is present.

### Read model is corrupt or drifted

Run the reindex pipeline. That is the whole procedure — it is the same recovery for a bad
mapping, a lost node, a transform bug, or an unexplained inconsistency. Verify the
document count, query the new index by concrete name, then let the alias swap promote it.

### Rollback a bad reindex

`POST /_aliases` swapping `products` back to `products-v{N-1}`. Seconds. The previous
index is retained for seven days precisely for this.

### Stream outage exceeded 24 hours

Records are gone; DynamoDB is unaffected. Run a full reindex. Confirm the count against
the table's item count, then resume the stream from `LATEST`.

## 7.8 Intended deployment

> **Scope note.** This repository is an **architecture deliverable** — a diagram and the
> documentation behind it. The procedure below specifies how the design is intended to be
> deployed and is written to be directly implementable; the SAM template and helper
> scripts it references are not part of this submission. Everything else in this
> repository describes the architecture as designed, not as running infrastructure.

**Prerequisites:** AWS CLI configured, AWS SAM CLI, Python 3.12, an AWS account.

```bash
# 1. Build and deploy the stack
sam build
sam deploy --guided          # creates VPC, DynamoDB, OpenSearch, Lambdas, API, WAF

# 2. Apply the index mapping and create the alias
python scripts/create_index.py --version 1
# → creates products-v1 and points the `products` alias at it

# 3. Create a seller account
aws cognito-idp sign-up --client-id <ClientId> \
  --username seller@example.com --password 'Passw0rd!'
aws cognito-idp admin-confirm-sign-up --user-pool-id <PoolId> \
  --username seller@example.com

# 4. Seed sample products
python scripts/seed.py --count 5000

# 5. Search (public — no token needed)
curl "<ApiUrl>/search?q=laptop&priceMax=20000&minRating=4&sort=price_asc"

# 6. Run a reindex
aws stepfunctions start-execution --state-machine-arn <ReindexArn>

# 7. Tear down
sam delete
```

## 7.9 Glossary

| Term | Meaning in this document |
|---|---|
| **CQRS** | Command Query Responsibility Segregation — separate models for writes and reads. |
| **CDC** | Change Data Capture — deriving a change stream from a database's commit log. |
| **Read model** | The derived, query-optimised projection. Here: the OpenSearch index. Disposable by design. |
| **Write model** | The authoritative store. Here: DynamoDB. |
| **External versioning** | Supplying a version from outside OpenSearch; writes with a non-greater version are rejected. |
| **Poison record** | A record that fails processing repeatedly and, unhandled, blocks an ordered stream shard. |
| **Partial batch response** | Returning only the failed records' sequence numbers so successful records in the batch are not reprocessed. |
| **Alias swap** | Atomically repointing a named alias from one concrete index to another. |
| **`search_after`** | Cursor pagination anchored on the previous page's sort values; constant cost at any depth. |

---

[← README](../README.md) | [← Future Work](./06-future-work.md) | **Appendix** | [Next: Cost Estimate →](./08-cost-estimate.md)
