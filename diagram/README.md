# Architecture Diagram

## Files

| File | Purpose |
|---|---|
| `architecture.drawio` | Editable source |
| `architecture.png` | Exported image embedded in the [README](../README.md) — **you must generate this** |

## Editing

1. Open [app.diagrams.net](https://app.diagrams.net) (or the desktop app / VS Code
   extension) and open `architecture.drawio`.
2. Make changes.
3. Export: **File → Export as → PNG…**
   - **Zoom:** 200%
   - **Border width:** 10
   - **Transparent background:** off
   - **Appearance:** Light
4. Save as `architecture.png` in this folder, overwriting the previous export.

The README references `./diagram/architecture.png`, so the filename must not change.

## If an AWS icon renders as a blank square

The file uses the **AWS 2019 (aws4)** shape library. If a stencil name is missing in your
draw.io version, the shape renders empty. Fix it by selecting the shape, pressing
`Ctrl+Shift+E` (Edit Style), and correcting the `resIcon=` value — or delete it and drag
the correct icon in from the shape panel (**More Shapes → Networking → AWS 2019**).

The one most likely to need this is **Amazon OpenSearch Service**
(`resIcon=mxgraph.aws4.opensearch_service`). Older draw.io builds still carry it under its
former name, `mxgraph.aws4.elasticsearch_service`.

## What the diagram shows

| Group | Contents |
|---|---|
| ① Edge & CDN | CloudFront, WAF, private S3 bucket for the SPA |
| ② Auth | Cognito User Pool |
| ③ API Tier | API Gateway — authorised `/products*` and public `/search` |
| ④ Command side | Command Lambda → DynamoDB (source of truth) |
| ⑤ **Sync pipeline** ⭐ | DynamoDB Streams → Indexer Lambda → OpenSearch, with the DLQ and the full event-source-mapping configuration |
| ⑥ Query side | Search Lambda translating a safe contract into OpenSearch DSL |
| ⑦ Read model | The OpenSearch domain, addressed through the `products` alias |
| ⑧ Reindex pipeline | Step Functions → S3 export → Distributed Map → atomic alias swap |
| ⑨ Observability & security | CloudWatch, X-Ray, KMS, IAM, CloudTrail |

Numbered arrows follow the flows documented in
[02-architecture.md](../docs/02-architecture.md):
**1–5** write · **6–8** sync · **9** the strongly-consistent seller read ·
**10–12** buyer search · **13–15** reindex.

Three callouts carry the arguments that the boxes alone cannot:

- **"The read model is disposable"** — the reliability property the whole design rests on.
- **The VPC note** — private subnets, free gateway endpoints, and the deliberate absence
  of a NAT Gateway.
- **The consistency contract** — who reads strongly and who reads eventually.

## Style conventions

Colours follow the AWS 2019 category palette, so a reader who knows the icon set can
identify service categories at a glance:

| Category | Colour |
|---|---|
| Networking & Content Delivery / Analytics | `#8C4FFF` |
| Compute | `#ED7100` |
| Database | `#C925D1` |
| Storage | `#7AA116` |
| Application Integration / Management | `#E7157B` |
| Security, Identity & Compliance | `#DD344C` |

Solid arrows are request/data flow; dashed arrows are secondary or failure paths. The
sync pipeline group is drawn with a heavier border because it is the subject of the
project.
