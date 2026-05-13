# Athena Engine Version 3

> **MCP Validated:** 2026-05-12
> **Purpose**: Athena engine v3 — Trino base, Iceberg support, workgroups, cost model
> **Confidence**: 0.95
> **Sources**: [AWS Athena engine v3 docs](https://docs.aws.amazon.com/athena/latest/ug/engine-versions-reference-0003.html), [AWS big-data blog upgrade post](https://aws.amazon.com/blogs/big-data/upgrade-to-athena-engine-version-3-to-increase-query-performance-and-access-more-analytics-features/)

## Overview

Athena engine version 3 is the current default (GA since Oct 2022; new workgroups default to v3). It is based on the **Trino** open-source engine with continuous-integration tracking of the Trino project — meaning community improvements land in Athena faster than before. v3 supersedes the Presto-based v2.

Key characteristics:
- Trino-based SQL dialect (not Presto-compatible in all edge cases; see breaking changes)
- Native Apache Iceberg support: `CREATE TABLE`, `INSERT INTO`, `MERGE INTO`, time travel, OPTIMIZE, VACUUM
- 50+ new SQL functions, 30+ new features, 90+ performance improvements over v2
- Pay-per-query: **$5.00 per TB scanned** (compressed Parquet with partition pruning drastically reduces cost)
- Serverless — no cluster to provision or manage

## Iceberg Support in Engine v3

| Operation | Support |
|-----------|---------|
| `CREATE TABLE` with `TBLPROPERTIES ('table_type'='ICEBERG')` | Full |
| `INSERT INTO` (append) | Full |
| `MERGE INTO` (upsert) | Full |
| `DELETE FROM` | Full (row-level, v2 position deletes) |
| `UPDATE` | Full |
| Time travel (`FOR TIMESTAMP AS OF`, `FOR VERSION AS OF`) | Full — note: v2 `FOR SYSTEM_TIME AS OF` syntax deprecated |
| `OPTIMIZE` (compaction) | Full — WHERE clause must use partition columns only |
| `VACUUM` (snapshot expiry) | Full |
| Schema evolution (`ALTER TABLE ... ADD/DROP/RENAME COLUMN`) | Full |
| Hidden partitioning (`day()`, `month()`, `year()`, `hour()`, `bucket()`, `truncate()`) | Full |
| Format | **Parquet** (default, recommended); ORC supported; Avro read-only |
| Iceberg table version created by Athena DDL | **v2** (not v3) |

## Workgroups

A workgroup groups users/queries for access control, cost management, and result location enforcement.

```sql
-- Check current workgroup engine version
SELECT current_workgroup_name;
```

Key workgroup settings:

| Setting | Purpose | Recommendation |
|---------|---------|---------------|
| Engine version | Must be set to `AUTO` or `Athena engine version 3` | Always use v3 |
| Query result location | S3 URI where `SELECT` results land | Enforce via workgroup; never leave unconfigured |
| Encrypt query results | SSE-S3, SSE-KMS, or CSE-KMS | SSE-S3 minimum; SSE-KMS for regulated data |
| Data usage control (per-query limit) | Cancel queries exceeding N bytes scanned | Set to ~1 TB as safety guard |
| Data usage control (workgroup limit) | SNS alarm when total exceeds threshold | Set per billing period |
| Override client settings | Force workgroup settings regardless of client config | Enable in prod |

Terraform snippet (module `infrastructure/modules/athena-iceberg/`):

```hcl
resource "aws_athena_workgroup" "warehouse_writer" {
  name = "warehouse-writer-${var.env}"

  configuration {
    enforce_workgroup_configuration    = true
    publish_cloudwatch_metrics_enabled = true
    engine_version {
      selected_engine_version = "Athena engine version 3"
    }
    result_configuration {
      output_location = "s3://${var.query_results_bucket}/athena-results/"
      encryption_configuration {
        encryption_option = "SSE_S3"
      }
    }
    bytes_scanned_cutoff_per_query = 1099511627776  # 1 TB
  }
}
```

## Cost Model

- **$5.00 per TB scanned** (us-east-1; other regions may differ slightly)
- DDL statements (`CREATE TABLE`, `ALTER TABLE`) — **free**
- Failed queries — **free** (no data scanned)
- Partition pruning + Parquet compression reduces scanned bytes by 90-99% vs raw CSV
- At 3,500 invoices/month each ~10 KB extracted: ~35 MB uncompressed → negligible cost per run
- Query result storage on S3: standard S3 pricing per GB; clean up results bucket with lifecycle policy

## Breaking Changes from Engine v2 (Relevant Subset)

| Change | v2 Behavior | v3 Behavior |
|--------|------------|------------|
| Time travel syntax | `FOR SYSTEM_TIME AS OF` | `FOR TIMESTAMP AS OF` |
| `log()` argument order | `log(value, base)` | `log(base, value)` — SQL standard |
| `uuid()` return type | `varchar` | `uuid` (cannot use in CTAS directly; cast to `varchar`) |
| Nested cols in GROUP BY | unquoted OK | must double-quote |
| Timestamp precision in Iceberg CTAS | 3 ms | cast to `timestamp(6)` required for Iceberg targets |

## Related

- [glue-catalog](glue-catalog.md)
- [iceberg-on-athena](iceberg-on-athena.md)
- [create-iceberg-table pattern](../patterns/create-iceberg-table.md)
- [partition-and-prune pattern](../patterns/partition-and-prune.md)
