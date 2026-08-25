[← README](../README.md) | [← Appendix](./07-appendix.md) | **Cost Estimate**

# 8. Cost Estimate

> **Basis.** Approximate `us-east-1` on-demand list prices, one month = 730 hours.
> Figures are rounded and intended for architectural comparison, not for budgeting.
> Verify against the [AWS Pricing Calculator](https://calculator.aws) before committing
> to any number here — AWS pricing changes, and Free Tier terms differ between the
> 12-month tier and the always-free tier.

Two columns, because they answer two different questions:

- **Demo** — what this actually costs to run as a portfolio deployment.
- **At scale anchor** — what it would cost at the production volumes defined in
  [§1.4](./01-requirements.md#14-scale-anchor), with no Free Tier applied.

---

## 8.1 Assumed monthly volumes

| | Demo | At scale anchor |
|---|---|---|
| Products in catalog | 5,000 | 2,000,000 |
| Product writes / month | ~2,000 | ~2,000,000 |
| Buyer searches / month | ~20,000 | ~50,000,000 |
| Edge cache hit ratio | — | 70% → 15 M origin requests |
| Average document size | 2 KB | 2 KB |
| Full reindex runs | 1 | 1 |

---

## 8.2 Line items

| Service | Free Tier | Demo | At scale anchor | Note |
|---|---|---:|---:|---|
| **Amazon OpenSearch Service**<br/>`t3.small.search`, 10 GB EBS | 750 hrs/mo + 10 GB, **first 12 months** | **\$0** | **\$27.50** | The only always-on resource. After month 12: ~\$26/mo. |
| **AWS WAF**<br/>web ACL + 3 managed rule groups | None | **\$8** | **\$38** | \$5 ACL + \$1/rule/mo + \$0.60 per million requests. **The one unavoidable standing charge.** |
| **Amazon CloudFront** | 1 TB out + 10 M requests/mo, **always free** | \$0 | \$30 | 40 M requests beyond the free tier. |
| **Amazon API Gateway** (REST) | 1 M requests/mo, first 12 months | \$0 | \$60 | \$3.50/M. 15 M search + 2 M write requests. |
| **AWS Lambda** (3 functions) | 1 M requests + 400,000 GB-s/mo, **always free** | \$0 | \$11 | Search Lambda dominates; indexer is ~\$0.33 thanks to batching. |
| **Amazon DynamoDB** (on-demand) | 25 GB storage, always free | \$0 | \$9 | Writes \$5, reads \$2, storage \$1, PITR \$0.80. |
| **DynamoDB Streams** | 2.5 M read request units/mo | \$0 | \$0 | 2 M records/month sits inside the free allowance. |
| **AWS KMS** (4 customer-managed keys) | 20,000 requests/mo | \$4 | \$5 | \$1/key/month — the demo pays this. |
| **Amazon CloudWatch** | 10 metrics, 5 GB logs, 10 alarms | \$0 | \$7 | Custom metrics and log volume at scale. |
| **AWS X-Ray** | 100,000 traces/mo | \$0 | \$4 | 5% sampling. |
| **Amazon S3** (SPA + exports) | 5 GB, first 12 months | \$0 | \$0.30 | Exports lifecycle to Glacier IR, then expire. |
| **Amazon SQS** (DLQ) | 1 M requests/mo, always free | \$0 | \$0 | Empty in normal operation. |
| **AWS Step Functions** (reindex) | 4,000 transitions/mo, always free | \$0 | \$0.05 | One monthly execution. |
| **Amazon Cognito** | Generous MAU allowance on the entry tier | \$0 | \$0 | 5,000 sellers. Verify the current tier — Cognito pricing was restructured in 2024. |
| **VPC Gateway Endpoints** (DynamoDB, S3) | — | **\$0** | **\$0** | Gateway endpoints are free. See below. |
| **NAT Gateway** | — | **\$0** | **\$0** | **Not provisioned.** See [DD 3.8](./03-design-decisions.md#38-opensearch-inside-a-vpc--and-avoiding-the-nat-gateway-trap). |
| | | | | |
| **Total** | | **≈ \$12 / month** | **≈ \$192 / month** | |

---

## 8.3 The three decisions that shaped this bill

Three architectural choices account for most of the difference between this estimate and
the obvious alternative build.

### No NAT Gateway — saves ~\$32/month per AZ, forever

Putting Lambda inside a VPC removes its default internet route, and the reflexive fix is
a NAT Gateway at roughly \$0.045/hour plus \$0.045 per GB processed. On the demo column
above, a single NAT Gateway would be **larger than every other line item combined** — and
it would buy nothing, because the functions need no internet access.

They need AWS service endpoints, and Gateway VPC Endpoints for DynamoDB and S3 are free.
The absence of a NAT Gateway on the architecture diagram is deliberate, and worth
noticing.

### Provisioned search instead of Serverless — the floor price problem

OpenSearch Serverless bills a minimum OCU allocation continuously, whether or not a query
is served. For a service that is idle most of the time, that floor dominates the bill and
**cannot be optimised away** — not by caching, not by scaling to zero, not by any change
to the architecture. A Free-Tier-eligible `t3.small.search` node costs \$0 for twelve
months and ~\$26/month after.

This is a scale-dependent decision, not a verdict on Serverless. At continuous production
traffic the floor stops being dead weight and the operational savings win — recorded as a
trigger in [Future Work](./06-future-work.md).

### Reindex from S3 Export instead of Scan — cost that does not scale with the table

A parallel `Scan` of 2 M items reads ~4 GB **through the table**, consuming read capacity
and competing with production traffic. `ExportTableToPointInTime` reads from continuous
backups instead: **zero table read capacity**, billed per GB of table data — cents for
this catalog, with no throttling risk to live requests.

---

## 8.4 Cost as traffic grows

| Monthly searches | Approx. total | What dominates |
|---:|---:|---|
| 20,000 (demo) | \$12 | KMS keys and the WAF web ACL — fixed costs |
| 1,000,000 | ~\$40 | WAF request charges and the search node |
| 10,000,000 | ~\$75 | API Gateway and WAF requests |
| 50,000,000 (scale anchor) | ~\$192 | API Gateway (\$60), WAF (\$38), CloudFront (\$30) |
| 200,000,000 | ~\$650 | Request-priced services scale linearly; the search node does not |

**The shape of this curve is the point.** Cost is dominated by per-request charges on the
edge and API tier, not by the datastores — which means the highest-leverage optimisation
is raising the edge cache hit ratio, not resizing the cluster. Moving from 70% to 85% cache
hit at the scale anchor removes roughly 7.5 M origin requests and takes ~\$25/month off the
bill without touching a single backend component.

Two cheaper structural options exist if request volume dominates further:

- **HTTP API instead of REST API** — roughly 70% cheaper per request (\$1.00/M vs
  \$3.50/M), at the cost of usage plans, request validation, and WAF integration on the
  API itself. Worth revisiting above ~20 M requests/month.
- **CloudFront Functions** for cache-key normalisation, so trivially-different query
  strings share a cache entry.

---

## 8.5 Cost guardrails in the build

Estimates are not controls. These are:

| Guardrail | Implementation |
|---|---|
| **Budget alerts** | AWS Budgets at \$10 and \$25, email via SNS. |
| **Invocation ceiling** | Reserved concurrency on all three Lambda functions — a retry storm cannot bill without bound. |
| **Per-request ceiling** | Maximum page size, page-depth cap, aggregation allow-list, and query `timeout` enforced by the Search Lambda ([DD 3.12](./03-design-decisions.md#312-search-lambda-over-direct-api-gateway--opensearch-integration)). |
| **Retry ceiling** | `MaximumRetryAttempts: 3` and `MaximumRecordAgeInSeconds: 3600` on the event source mapping. |
| **Storage decay** | DynamoDB TTL purges soft-deleted items; S3 export lifecycle to Glacier IR then expiry; superseded indices deleted after a seven-day rollback window. |
| **One-command teardown** | `sam delete` removes the stack. The OpenSearch domain is the only resource worth deleting between demos — and it can be rebuilt in ~30 minutes by the reindex pipeline, because the read model is disposable. |

---

[← README](../README.md) | [← Appendix](./07-appendix.md) | **Cost Estimate**
