[← README](../README.md) | [← Well-Architected](./04-well-architected.md) | **Risks** | [Next: Future Work →](./06-future-work.md)

# 5. Risk Register

Risks are rated for **likelihood** and **impact** as the system is actually built — not
as it would be with unlimited budget. Where a risk is *accepted* rather than mitigated,
that is stated plainly.

| ID | Risk | Likelihood | Impact | Mitigation |
|----|------|:---:|:---:|------------|
| **R-1** | **Sync lag spike.** A bulk seller import floods the stream and the read model falls minutes behind the write model. Buyers see a stale catalog. | High | Medium | `ParallelizationFactor: 10` gives ten concurrent batches per shard. `IteratorAge` alarm at 60 s and a `SyncLagMs` EMF metric make it visible within one alarm period. The lag is bounded and self-healing — the pipeline drains once the burst ends, and nothing is lost, only delayed. |
| **R-2** | **Poison record stalls a shard.** A malformed item — a mapping conflict, an oversized field — fails indexing repeatedly and blocks every later record on that shard. | Medium | **High** | Fully mitigated by design: `ReportBatchItemFailures` commits the other records in the batch, `BisectBatchOnFunctionError` isolates the offender, `MaximumRetryAttempts: 3` and `MaximumRecordAgeInSeconds: 3600` bound the retry loop, and the `OnFailure` DLQ preserves the record with its sequence number. DLQ depth ≥ 1 alarms. See [DD 3.6](./03-design-decisions.md#36-poison-records--partial-batch-response--bisect). |
| **R-3** | **Out-of-order update corrupts a document.** A stale v6 lands after v7 and silently reverts a price in the search index. | Medium | **High** | External versioning rejects it atomically server-side. The failure mode is converted from silent corruption into a counted `409`. `IndexVersionConflicts` is tracked; a *spike* alarms, a steady trickle is evidence the mechanism works. See [DD 3.5](./03-design-decisions.md#35-out-of-order-updates--external-versioning). |
| **R-4** | **OpenSearch node loss.** The demo runs a single-AZ, single-node domain. The node dies and search returns nothing. | Medium | Medium | **Partially accepted.** Search degrades; the catalog does not. DynamoDB continues to serve writes and all keyed reads, so sellers and product pages are unaffected — only discovery is lost. Recovery is one reindex run (~30 min for 2 M docs), because the read model holds no authoritative data. Production topology (3 masters + 3 data nodes across 3 AZs) is specified in the [Appendix](./07-appendix.md); the demo accepts the risk to stay inside the Free Tier, and this is an explicit trade, not an oversight. |
| **R-5** | **Index/mapping drift.** The mapping in the cluster diverges from the mapping in the repository after a manual change, and the next reindex behaves differently from production. | Medium | Medium | Mappings and analyzer settings live in version control and are applied by the reindex pipeline, never by hand. Versioned indices (`products-v{N}`) mean a mapping change is always a rebuild behind an alias swap, so the deployed mapping and the committed mapping cannot silently diverge for long. |
| **R-6** | **Deep-pagination or aggregation abuse.** A crawler walks to page 50,000, or a crafted request triggers a high-cardinality aggregation, and CPU on a single-node cluster spikes for every user. | Medium | Medium | Clients cannot send raw DSL. The Search Lambda enforces `search_after` cursors, a maximum page size, a hard page-depth cap, an aggregation allow-list, and a per-query `timeout`. WAF rate-based rules throttle at the edge before the request reaches Lambda. See [DD 3.12](./03-design-decisions.md#312-search-lambda-over-direct-api-gateway--opensearch-integration) and [DD 3.14](./03-design-decisions.md#314-deep-pagination--search_after-over-fromsize). |
| **R-7** | **⭐ Pipeline outage exceeding 24 hours.** DynamoDB Streams retention is fixed at 24 h and cannot be extended. If the indexer is broken for longer, those change records are gone permanently. | Low | **High** | This is the one failure the stream cannot recover from, and it is precisely why the reindex pipeline is a first-class component rather than a maintenance script. Recovery is a full rebuild from `ExportTableToPointInTime` — no data is lost, because DynamoDB is the source of truth. The `IteratorAge` alarm fires at 60 s, giving roughly 23 hours of margin before the deadline. Documented and rehearsed in the [Appendix](./07-appendix.md) runbook. |
| **R-8** | **Reindex builds a bad index and it is promoted.** A transform bug produces 2 M subtly wrong documents and the alias is swapped onto them. | Low | **High** | The pipeline verifies document count against the export manifest before swapping, and the new index can be queried by its concrete name for validation before any traffic reaches it. The previous index is retained for seven days, so rollback is a single alias action taking seconds. |
| **R-9** | **Concurrent seller edits lose an update.** Two dashboard tabs save the same product; the second silently overwrites the first. | Medium | Low | Optimistic locking on `version` via `ConditionExpression`. The second write fails with `409 Conflict` and the client re-reads and retries rather than clobbering. The same `version` attribute then guarantees ordering downstream — one mechanism, two problems. |
| **R-10** | **Read-your-writes confusion.** A seller updates a price, does not see it reflected, and edits again — generating duplicate writes and distrust. | High | Low | Structurally removed rather than mitigated: seller-facing reads never touch the read model. `GET /products/{id}` and the "my products" listing are served from DynamoDB. Only anonymous buyer search is eventually consistent, and the staleness bound is published in an `X-Index-Lag-Ms` response header. See [DD 3.11](./03-design-decisions.md#311-read-your-writes--who-is-allowed-to-see-stale-data). |
| **R-11** | **Internal fields leak into the public index.** `costPrice` or `margin` reaches a read model queried by anonymous buyers. | Low | **High** | The indexer projects an explicit **allow-list**, so a new sensitive attribute in DynamoDB is excluded by default rather than included by default. `_source` filtering in the Search Lambda is a second layer. A schema test in CI asserts the indexed field set matches the allow-list. |
| **R-12** | **Cost overrun.** A retry loop, an accidental provisioned-capacity change, or a forgotten Serverless collection produces a surprise bill. | Low | Medium | AWS Budgets alerts at \$10 and \$25. No NAT Gateway and no Serverless collection are provisioned — the two largest silent cost sources are structurally absent. Lambda reserved concurrency caps invocation cost. The demo teardown is a single `sam delete`. |
| **R-13** | **Free Tier expiry.** The OpenSearch Free Tier covers 750 hours/month for 12 months only; after that the node bills continuously. | High *(certain, at 12 months)* | Low | Known and dated, not a surprise. Post-Free-Tier cost is roughly \$26/month for `t3.small.search`, stated in the [cost estimate](./08-cost-estimate.md). For a portfolio deployment the domain can be deleted and rebuilt on demand from the reindex pipeline in ~30 minutes — again a consequence of the read model being disposable. |

## Risks explicitly accepted

Recorded so the reader does not mistake an omission for an oversight:

1. **Single-AZ search (R-4).** Accepted for the demo in exchange for Free Tier
   eligibility. The production topology is specified and the failure is non-authoritative.
2. **Eventual consistency for buyer search (R-10).** Accepted deliberately; it is the
   point of the architecture. Bounded by an SLO, measured, and published to clients.
3. **Single region.** No multi-region failover. A regional outage takes the service
   down. Out of scope by [§1.6](./01-requirements.md#16-out-of-scope), with the migration
   path in [Future Work](./06-future-work.md).
4. **At-least-once indexing.** The pipeline may index the same change twice. This is
   harmless — indexing is idempotent by document `_id` and guarded by external
   versioning — but it is a property, not an accident.

---

[← README](../README.md) | [← Well-Architected](./04-well-architected.md) | **Risks** | [Next: Future Work →](./06-future-work.md)
