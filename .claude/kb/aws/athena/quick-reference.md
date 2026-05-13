# Athena + Iceberg Quick Reference

> **MCP Validated:** 2026-05-12

## DDL Cheat Sheet

```sql
-- Create Iceberg table (Athena v3, Glue catalog)
CREATE TABLE IF NOT EXISTS invoices_dev.extracted_invoices (
  invoice_id   VARCHAR NOT NULL,
  vendor_type  VARCHAR NOT NULL,
  invoice_date DATE    NOT NULL,
  ...
)
PARTITIONED BY (vendor_type, day(invoice_date))
LOCATION 's3://invoices-warehouse-dev/extracted_invoices/'
TBLPROPERTIES ('table_type'='ICEBERG', 'format'='parquet', 'write_compression'='snappy');

-- Insert batch
INSERT INTO invoices_dev.extracted_invoices (invoice_id, vendor_type, ...) VALUES (...), (...);

-- Upsert
MERGE INTO invoices_dev.extracted_invoices AS t USING staging AS s ON t.invoice_id = s.invoice_id
WHEN MATCHED THEN UPDATE SET ... WHEN NOT MATCHED THEN INSERT (...) VALUES (...);

-- Time travel (engine v3 syntax)
SELECT * FROM invoices_dev.extracted_invoices FOR TIMESTAMP AS OF TIMESTAMP '2026-05-01 00:00:00';
SELECT * FROM invoices_dev.extracted_invoices FOR VERSION AS OF 8234712934812938;

-- Compaction
OPTIMIZE invoices_dev.extracted_invoices REWRITE DATA USING BIN_PACK WHERE vendor_type = 'ubereats';

-- Vacuum snapshots
VACUUM invoices_dev.extracted_invoices;
```

## Key Limits

| Limit | Value |
|-------|-------|
| Cost | $5.00 / TB scanned |
| Max partitions per INSERT | 100 |
| OPTIMIZE WHERE clause | Partition columns only |
| Iceberg table version (Athena DDL) | v2 (not v3) |
| Time travel syntax (engine v3) | `FOR TIMESTAMP AS OF` / `FOR VERSION AS OF` |
| Timestamp precision in Iceberg CTAS | Use `CAST(dt AS timestamp(6))` |

## boto3 Insert Pattern (Athena)

```python
import boto3, time
athena = boto3.client("athena", region_name="us-east-1")

qid = athena.start_query_execution(
    QueryString="INSERT INTO invoices_prod.extracted_invoices ...",
    WorkGroup="warehouse-writer-prod",
    QueryExecutionContext={"Database": "invoices_prod"},
)["QueryExecutionId"]

# Poll
for _ in range(30):
    state = athena.get_query_execution(QueryExecutionId=qid)["QueryExecution"]["Status"]["State"]
    if state == "SUCCEEDED": break
    if state in ("FAILED", "CANCELLED"): raise RuntimeError(state)
    time.sleep(2)
```

## pyiceberg Write Pattern

```python
from pyiceberg.catalog.glue import GlueCatalog
import pyarrow as pa

catalog = GlueCatalog("invoices", **{"glue.region": "us-east-1"})
table   = catalog.load_table("invoices_prod.extracted_invoices")
arrow_table = pa.Table.from_pylist(rows, schema=SCHEMA)
table.append(arrow_table)
```

## IAM Actions Required

| Scope | Actions |
|-------|---------|
| Athena | `athena:StartQueryExecution`, `athena:GetQueryExecution`, `athena:GetWorkGroup` |
| Glue | `glue:GetTable`, `glue:UpdateTable`, `glue:GetDatabase`, `glue:GetPartitions` |
| S3 warehouse | `s3:PutObject`, `s3:GetObject`, `s3:ListBucket`, `s3:DeleteObject` |
| S3 query results | `s3:PutObject`, `s3:GetObject`, `s3:ListBucket` |

## Partition Transforms

`year(ts)`, `month(ts)`, `day(ts)`, `hour(ts)` (timestamp only), `bucket(N, col)`, `truncate(L, col)`, identity (no transform).

## Anti-Patterns (Do Not)

- One Athena query per row — batch minimum 10 rows
- No workgroup set — always pass `WorkGroup=`
- `FOR SYSTEM_TIME AS OF` — broken in engine v3; use `FOR TIMESTAMP AS OF`
- Function on partition column in WHERE (`MONTH(invoice_date)`) — use explicit date range
- OPTIMIZE WHERE on non-partition column — Athena rejects it
