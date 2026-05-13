# Create Iceberg Table (DDL)

> **MCP Validated:** 2026-05-12
> **Purpose**: Athena DDL to create the `extracted_invoices` Iceberg table with correct partition spec for the invoice pipeline
> **Confidence**: 0.95
> **Sources**: [Athena create Iceberg tables](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg-creating-tables.html)

## Overview

The `extracted_invoices` table is the authoritative warehouse table for the invoice pipeline. It is created once (via `warehouse-writer` Lambda bootstrap or Terraform) and written to on every invocation. Partitioned by `vendor_type` (identity) and `day(invoice_date)` (Iceberg hidden transform).

## DDL

```sql
CREATE TABLE IF NOT EXISTS invoices_${env}.extracted_invoices (
  -- Business keys
  invoice_id          VARCHAR     NOT NULL COMMENT 'Unique invoice identifier',
  vendor_type         VARCHAR     NOT NULL COMMENT 'ubereats | doordash | grubhub',
  -- Date fields
  invoice_date        DATE        NOT NULL COMMENT 'Invoice issue date (partition key)',
  period_start        DATE        COMMENT 'Billing period start',
  period_end          DATE        COMMENT 'Billing period end',
  -- Financial fields (cents to avoid float precision issues)
  gross_sales_cents   BIGINT      COMMENT 'Gross sales in USD cents',
  commission_cents    BIGINT      COMMENT 'Platform commission in USD cents',
  delivery_fee_cents  BIGINT      COMMENT 'Delivery fees in USD cents',
  adjustments_cents   BIGINT      COMMENT 'Adjustments / credits in USD cents',
  net_payout_cents    BIGINT      COMMENT 'Net payout = gross - commission - fees + adjustments',
  -- Restaurant metadata
  restaurant_name     VARCHAR     COMMENT 'Restaurant name as printed on invoice',
  restaurant_id       VARCHAR     COMMENT 'Platform-assigned restaurant identifier',
  -- Pipeline metadata
  extraction_confidence DOUBLE    COMMENT 'LLM extraction confidence score [0.0, 1.0]',
  llm_model           VARCHAR     COMMENT 'Model used: haiku-4-5 | sonnet-4-5',
  source_s3_key       VARCHAR     NOT NULL COMMENT 'S3 key of source TIFF file',
  processed_at        TIMESTAMP   NOT NULL COMMENT 'UTC timestamp of warehouse write'
)
PARTITIONED BY (
  vendor_type,               -- identity transform (low cardinality: 3 values)
  day(invoice_date)          -- hidden day transform on DATE column
)
LOCATION 's3://invoices-warehouse-${env}/extracted_invoices/'
TBLPROPERTIES (
  'table_type'                        = 'ICEBERG',
  'format'                            = 'parquet',
  'write_compression'                 = 'snappy',
  'optimize_rewrite_delete_file_threshold' = '5',
  'vacuum_max_snapshot_age_seconds'   = '604800',  -- 7 days
  'vacuum_min_snapshots_to_keep'      = '3'
);
```

## Partition Design Rationale

| Partition field | Transform | Cardinality | Purpose |
|----------------|-----------|-------------|---------|
| `vendor_type` | identity | 3 (ubereats, doordash, grubhub) | Filter by vendor in reconciliation queries |
| `invoice_date` | `day()` | ~31/month | Filter by date range; each day ~100 invoices max |

**Why hidden partitioning:** Queries filter on `vendor_type = 'ubereats'` and `invoice_date BETWEEN '2026-04-01' AND '2026-04-30'` without knowing the physical partition layout. The Iceberg engine prunes partitions automatically. Do NOT use Hive-style `PARTITIONED BY (vendor_type STRING, invoice_date STRING)` which forces query predicates to match partition column names and types exactly.

## Verifying the Table

```sql
-- Show table properties including metadata_location
SHOW TBLPROPERTIES invoices_dev.extracted_invoices;

-- Show partition spec
DESCRIBE invoices_dev.extracted_invoices;

-- Check table history (snapshots)
SELECT * FROM "invoices_dev"."extracted_invoices$history";

-- Check files per partition
SELECT * FROM "invoices_dev"."extracted_invoices$partitions";
```

## One-time Bootstrap in Lambda (Python)

```python
import boto3

def bootstrap_table(env: str, workgroup: str) -> None:
    """Run CREATE TABLE IF NOT EXISTS on cold start or deploy."""
    athena = boto3.client("athena", region_name="us-east-1")
    ddl = open("sql/create_extracted_invoices.sql").read().replace("${env}", env)
    response = athena.start_query_execution(
        QueryString=ddl,
        WorkGroup=workgroup,
        QueryExecutionContext={"Database": f"invoices_{env}"},
    )
    # Poll for completion (DDL is fast, typically < 5s)
    _wait_for_query(athena, response["QueryExecutionId"])
```

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Hive-style `PARTITIONED BY (invoice_date STRING)` | Partition column duplicated; no transform; loses hidden partitioning | Use `day(invoice_date)` in PARTITIONED BY |
| No `LOCATION` | Athena uses random temp location; table lost after recreation | Always specify `LOCATION` pointing to permanent S3 prefix |
| `format='csv'` | No Iceberg Parquet compression; 10-100x more data scanned | Use `format='parquet'` |
| Creating table per vendor | Fragments data; harder to query cross-vendor | One table, partition by `vendor_type` |

## Related

- [partition-and-prune pattern](partition-and-prune.md)
- [lambda-insert-batch pattern](lambda-insert-batch.md)
- [glue-catalog concept](../concepts/glue-catalog.md)
- [athena-engine-v3 concept](../concepts/athena-engine-v3.md)
