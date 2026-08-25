[← README](../README.md) | [← Design Decisions](./03-design-decisions.md) | **Well-Architected** | [Next: Risks →](./05-risks.md)

# 4. AWS Well-Architected Framework

Mapped against the six pillars. Each row states what the architecture actually does, not
what it aspires to, and links to the decision that produced it.

---

## Operational Excellence

| Practice | Implementation |
|---|---|
| **Infrastructure as code** | The entire stack is defined in AWS SAM/CloudFormation — reproducible deploy, one-command teardown, no console drift. Index mappings and analyzer settings are versioned in the repository alongside it. |
| **Observability of the thing that actually matters** | Sync lag is the health metric of a CQRS system, and it is measured two ways: `IteratorAge` on the event source mapping (how far behind the consumer is) and a custom EMF metric `SyncLagMs` (`now − ApproximateCreationDateTime`, measured at index time). Latency and error rate alone would not reveal a silently drifting read model. |
| **Failures are visible, not silent** | Alarms on: `IteratorAge` > 60 s, DLQ depth ≥ 1, Lambda error rate, `IndexVersionConflicts` spike, OpenSearch `ClusterStatus` yellow/red, `FreeStorageSpace`, `JVMMemoryPressure`. |
| **Tracing** | X-Ray across API Gateway → Lambda → DynamoDB / OpenSearch, with a service map that makes the two paths visually distinct. |
| **Runbooks** | The reindex pipeline *is* the runbook for read-model corruption, drift, mapping change, and node loss — one Step Functions execution, tested, with a documented rollback ([DD 3.10](./03-design-decisions.md#310-zero-downtime-reindex--index-aliases)). |
| **Safe change** | Versioned indices mean a mapping change is a routine deploy behind an alias swap, with a seven-day rollback window, rather than a destructive in-place migration. |

---

## Security

| Practice | Implementation |
|---|---|
| **No public data stores** | The OpenSearch domain is in private subnets with **no public endpoint**; the S3 buckets are private and reachable only through CloudFront Origin Access Control ([DD 3.8](./03-design-decisions.md#38-opensearch-inside-a-vpc--and-avoiding-the-nat-gateway-trap)). |
| **Defence in depth at the edge** | AWS WAF on CloudFront with the Core Rule Set, Known Bad Inputs, and a rate-based rule — abusive traffic is dropped before it can reach a Lambda function or an expensive OpenSearch query. |
| **Authentication and authorisation** | Cognito User Pool with MFA; API Gateway validates the JWT before any function runs. Write handlers additionally verify that the caller owns the product — authentication is not authorisation. |
| **The read model is a publication surface** | The indexer applies an explicit **allow-list** of fields. `costPrice`, `margin`, and internal notes exist in DynamoDB and are structurally incapable of reaching an index that serves anonymous buyers ([DD 3.4](./03-design-decisions.md#34-custom-indexer-lambda-over-zero-etl)). |
| **No datastore query language exposed** | Clients send a constrained contract; the Search Lambda builds the DSL, capping page size, aggregations, and query timeout ([DD 3.12](./03-design-decisions.md#312-search-lambda-over-direct-api-gateway--opensearch-integration)). |
| **Encryption at rest** | KMS customer-managed keys on DynamoDB, S3, SQS, and the OpenSearch domain. |
| **Encryption in transit** | TLS on every hop; node-to-node encryption enabled on the domain; `enforce_https` on the domain endpoint. |
| **Least privilege** | A distinct IAM role per function. The Search Lambda holds `es:ESHttpGet`/`ESHttpPost` on the search path only — it cannot write to the index. The Indexer cannot read the API's tables beyond what it needs. OpenSearch fine-grained access control maps IAM roles to index-level permissions. |
| **Audit** | CloudTrail across all regions with log-file integrity validation; every KMS and Secrets Manager access recorded. |

---

## Reliability

| Practice | Implementation |
|---|---|
| **⭐ The read model is disposable** | This is the load-bearing reliability property of the whole design. OpenSearch holds no authoritative data. Node loss, index corruption, a bad mapping, or a botched deploy all collapse to the same recovery: run the reindex pipeline. It is what makes a single-node demo cluster a defensible choice rather than a hidden single point of failure ([DD 3.7](./03-design-decisions.md#37-opensearch-provisioned-over-serverless)). |
| **Graceful degradation** | If OpenSearch is unavailable, buyers lose search — but the catalog is not down. Writes continue, sellers manage products normally, product detail pages served by key from DynamoDB still work. The blast radius of a search outage is search. |
| **No dual-write** | There is exactly one component permitted to write to the read model, and it is driven by durable CDC with built-in retry. The application cannot create an inconsistency, because the application is not in that path ([DD 3.2](./03-design-decisions.md#32-change-data-capture-over-dual-write)). |
| **Poison records cannot stall the pipeline** | Partial batch response, bisect-on-error, bounded retries, and a failure destination. One bad record is isolated and preserved, not allowed to halt a shard ([DD 3.6](./03-design-decisions.md#36-poison-records--partial-batch-response--bisect)). |
| **Out-of-order writes cannot corrupt the index** | External versioning rejects stale updates atomically, server-side ([DD 3.5](./03-design-decisions.md#35-out-of-order-updates--external-versioning)). |
| **Nothing is dropped silently** | Records exhausting retries land in a DLQ with their sequence numbers, alarmed at depth ≥ 1 and replayable. |
| **Durability of the source of truth** | DynamoDB is multi-AZ by default with PITR enabled; AWS Backup holds scheduled backups. |
| **Managed multi-AZ throughout the write path** | API Gateway, Lambda, DynamoDB, SQS, and S3 are regional multi-AZ services with no single-AZ component on the write path. |

---

## Performance Efficiency

| Practice | Implementation |
|---|---|
| **Right store for each access pattern** | Keyed lookups go to DynamoDB (single-digit ms at any scale); relevance, filtering, and aggregation go to an inverted index built for them. Neither store is asked to do the other's job ([DD 3.1](./03-design-decisions.md#31-cqrs-versus-a-single-data-store)). |
| **Facets precomputed at index time** | `priceBucket` is derived by the indexer so price faceting is a cheap `terms` aggregation rather than a `range` aggregation evaluated per query — work moved from read time (200 req/s) to write time (8/s). |
| **Pagination that does not degrade with depth** | `search_after` cursors cost the same on page 500 as on page 1, where `from`/`size` degrades quadratically and then fails outright ([DD 3.14](./03-design-decisions.md#314-deep-pagination--search_after-over-fromsize)). |
| **Tuned stream consumption** | `BatchSize: 100` amortises the bulk request; `MaximumBatchingWindowInSeconds: 5` collapses invocations at low write rates; `ParallelizationFactor: 10` absorbs the 5,000/min bulk-import burst from the scale anchor without unbounded lag. |
| **Edge caching** | CloudFront caches the SPA and popular anonymous search responses on a short TTL, keyed on the normalised query — the long tail reaches OpenSearch, the head does not. |
| **Parallel reindex** | Step Functions Distributed Map fans out across S3 export objects; reindex time scales with concurrency rather than with a single sequential scan. |
| **Scale-to-zero write path** | Lambda and DynamoDB on-demand consume nothing between requests. |

---

## Cost Optimization

| Practice | Implementation |
|---|---|
| **Serverless rejected where it costs more, chosen where it costs less** | OpenSearch Serverless carries a continuous minimum-capacity charge regardless of traffic — a floor that no architectural change can optimise away, and indefensible for an intermittently-used service. `t3.small.search` is Free-Tier eligible. Conversely the write path is fully serverless and bills only on use ([DD 3.7](./03-design-decisions.md#37-opensearch-provisioned-over-serverless)). |
| **⭐ No NAT Gateway** | Attaching Lambda to a VPC normally triggers a ~\$32/month/AZ NAT Gateway. Gateway VPC Endpoints for DynamoDB and S3 are **free**, and the functions need no internet route at all — so none is provisioned. On a demo bill this single decision is larger than every other line item ([DD 3.8](./03-design-decisions.md#38-opensearch-inside-a-vpc--and-avoiding-the-nat-gateway-trap)). |
| **Reindex consumes zero table capacity** | `ExportTableToPointInTime` reads from continuous backups, not the table — a full 2 M-item reindex costs cents and does not throttle production ([DD 3.9](./03-design-decisions.md#39-reindex-source--s3-export-over-scan)). |
| **Work moved from read time to write time** | The read path runs at 200 req/s and the write path at ~8/s. Every computation pushed into the indexer is amortised across ~25× fewer executions. |
| **Edge caching cuts origin invocations** | Cached search responses never reach Lambda or OpenSearch. |
| **Guardrails, not hope** | AWS Budgets alert at \$10; Lambda reserved concurrency caps runaway invocation cost; the API enforces page-size and aggregation limits so no single request can be arbitrarily expensive. |
| **Honest accounting** | A full [cost estimate](./08-cost-estimate.md) is published, including the costs that are *not* free — AWS WAF is the one unavoidable standing charge, and it is stated rather than hidden. |

---

## Sustainability

| Practice | Implementation |
|---|---|
| **Scale to zero** | The entire write and query compute tier is Lambda; no idle servers. The only continuously-running resource is the single search node, deliberately the smallest instance that fits the workload. |
| **Right-sizing as a first-class decision** | One `t3.small.search` node is provisioned against a measured 4 GB index and the documented scale anchor — not a default three-node cluster chosen for comfort. |
| **Less compute per query** | Precomputed facet buckets, edge caching, and constant-cost pagination each reduce cluster CPU per user-visible result. |
| **Less compute per write** | Batching 100 stream records into one bulk request and one invocation, rather than one invocation per record, cuts execution count by roughly two orders of magnitude at burst. |
| **Storage discipline** | Soft-deleted items are physically purged by DynamoDB TTL; superseded indices are retained for a bounded seven-day rollback window and then deleted; S3 exports transition to Glacier Instant Retrieval and expire. |
| **No duplicated idle infrastructure** | Single region, no warm standby. Multi-region is explicitly [out of scope](./01-requirements.md#16-out-of-scope) rather than provisioned unused. |

---

[← README](../README.md) | [← Design Decisions](./03-design-decisions.md) | **Well-Architected** | [Next: Risks →](./05-risks.md)
