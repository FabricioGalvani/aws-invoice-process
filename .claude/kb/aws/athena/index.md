# Athena + Glue + Iceberg Knowledge Base

> **MCP Validated:** 2026-05-12
> **Purpose**: Amazon Athena engine v3, Glue Data Catalog, and Iceberg as the data warehouse layer for the invoice pipeline
> **Domain**: `aws/athena/`
> **Project context**: `warehouse-writer` Lambda writes extracted invoices to `extracted_invoices` Iceberg table; Athena v3 used for reconciliation queries and time travel

## Navigation

### Concepts

| File | What it covers |
|------|---------------|
| [athena-engine-v3](concepts/athena-engine-v3.md) | Trino-based engine, Iceberg support matrix, workgroups, cost model ($5/TB), breaking changes from v2 |
| [glue-catalog](concepts/glue-catalog.md) | Glue as the only supported Iceberg catalog, database/table hierarchy, IAM, Glue REST endpoint |
| [iceberg-on-athena](concepts/iceberg-on-athena.md) | Athena-specific Iceberg: write ops, time travel syntax, schema evolution, MERGE INTO, OPTIMIZE, limits |
| [pyiceberg-alternative](concepts/pyiceberg-alternative.md) | Direct S3+Glue writes from Lambda via pyiceberg — no Athena query required for write path |

### Patterns

| File | What it covers |
|------|---------------|
| [create-iceberg-table](patterns/create-iceberg-table.md) | Full DDL for `extracted_invoices` with hidden partition spec, TBLPROPERTIES, bootstrap |
| [lambda-insert-batch](patterns/lambda-insert-batch.md) | `warehouse-writer` Lambda using `start_query_execution` INSERT INTO with polling |
| [pyiceberg-write](patterns/pyiceberg-write.md) | Direct write from Lambda via pyiceberg GlueCatalog + PyArrow, concurrency control |
| [partition-and-prune](patterns/partition-and-prune.md) | Partition design rationale, cost-efficient query patterns, compaction schedule |

### Specs

| File | What it covers |
|------|---------------|
| [athena-iam-policy.json](specs/athena-iam-policy.json) | Least-privilege IAM policy for `warehouse-writer` (Athena + Glue + S3) |

## External References (do not duplicate here)

- Iceberg format internals (manifest files, deletion vectors, format-version): [lakehouse/concepts/iceberg-v3](../../lakehouse/concepts/iceberg-v3.md)
- MERGE, compaction, snapshot expiry SQL patterns: [lakehouse/patterns/iceberg-operations](../../lakehouse/patterns/iceberg-operations.md)

## Decision Context

From `notes/07-aws-migration-plan.md` §2.3 + §4.5:
- **Decision D9 (AWS):** Athena + S3 Iceberg replaces BigQuery
- **Table:** `invoices_{env}.extracted_invoices` partitioned by `vendor_type, day(invoice_date)`
- **Writer:** `warehouse-writer` Lambda (512 MB, 1 min timeout, SQS trigger)
- **Write path choice:** Athena SQL (default) or pyiceberg direct (lower latency alternative)
- **IAM:** least-privilege role per Lambda (see `notes/07-aws-migration-plan.md` §4.5)

## Quick Decision: Athena SQL vs pyiceberg Write Path

| Criterion | Athena SQL path | pyiceberg path |
|-----------|----------------|---------------|
| Dependency complexity | Only boto3 (built-in) | pyiceberg + pyarrow layer/container |
| Latency per write | 2-10s (async + polling) | <1s (synchronous) |
| Athena service quota impact | Yes (concurrent queries) | No |
| Concurrency safety | Athena handles it | Requires MaxConcurrency=1 or retry |
| Recommended for this pipeline | Default choice | Use if Athena latency is a concern |
