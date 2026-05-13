# Iceberg on Athena

> **MCP Validated:** 2026-05-12
> **Purpose**: What Athena supports for Iceberg — write paths, query patterns, limits (links to lakehouse/ for Iceberg internals)
> **Confidence**: 0.95
> **Sources**: [Athena Iceberg create tables](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg-creating-tables.html), [Athena Iceberg additional operations](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg-additional-operations.html), [OLake Athena+Iceberg overview](https://olake.io/iceberg/query-engine/athena/)

> **Iceberg internals (format-version, manifest files, hidden partitioning, deletion vectors):**
> See [lakehouse/concepts/iceberg-v3](../../../lakehouse/concepts/iceberg-v3.md)
>
> **Iceberg table operations (MERGE, compaction, time travel SQL patterns):**
> See [lakehouse/patterns/iceberg-operations](../../../lakehouse/patterns/iceberg-operations.md)

## Athena-Specific Iceberg Details

### Table Version

Athena creates **Iceberg v2 tables** via DDL (not v3). This is sufficient for the invoice pipeline: v2 supports position deletes, hidden partitioning, MERGE INTO, and time travel. v3 features (deletion vectors, Variant type) require direct Iceberg SDK writes.

### Supported Write Operations

| SQL Statement | Notes |
|---------------|-------|
| `INSERT INTO` | Appends; preferred for `warehouse-writer` Lambda batch inserts |
| `INSERT OVERWRITE` | Replaces entire partition(s); use for full partition refresh |
| `MERGE INTO` | Upsert; matches on key column, UPDATE or INSERT branch |
| `DELETE FROM` | Row-level delete; uses position delete files |
| `UPDATE` | Row-level update |
| `CREATE TABLE AS SELECT` (CTAS) | Creates new Iceberg table from query result; max 100 partitions |

### Athena INSERT Partition Limit

**Important for Lambda usage:** Athena `INSERT INTO` on a partitioned Iceberg table is limited to **100 partitions per statement**. For the invoice pipeline (partitioned by `vendor_type, day(invoice_date)`):
- 3 vendors × ~31 days = ~93 partition combinations per month — within limit for monthly batch
- Per-invocation inserts of 1-50 invoices will typically touch 1-3 partitions — safe

### Time Travel Syntax (v3 engine)

```sql
-- By timestamp (ISO 8601 string)
SELECT * FROM invoices_dev.extracted_invoices
FOR TIMESTAMP AS OF TIMESTAMP '2026-05-01 00:00:00';

-- By snapshot ID (get from information_schema or SHOW TBLPROPERTIES)
SELECT * FROM invoices_dev.extracted_invoices
FOR VERSION AS OF 8234712934812938;

-- Previous engine v2 syntax (deprecated in v3 — do NOT use)
-- FOR SYSTEM_TIME AS OF ...  ← broken in engine v3
```

### Schema Evolution

```sql
-- Add column (safe — NULL for existing rows)
ALTER TABLE invoices_dev.extracted_invoices
ADD COLUMNS (delivery_fee_cents bigint);

-- Rename column
ALTER TABLE invoices_dev.extracted_invoices
CHANGE COLUMN total_amount total_amount_cents bigint;

-- Drop column (data physically remains but hidden from queries)
ALTER TABLE invoices_dev.extracted_invoices
DROP COLUMN legacy_field;
```

### MERGE INTO (Upsert)

```sql
MERGE INTO invoices_dev.extracted_invoices AS target
USING (
  SELECT * FROM invoices_dev.extracted_invoices_staging
) AS source
ON target.invoice_id = source.invoice_id
WHEN MATCHED THEN
  UPDATE SET
    target.extraction_confidence = source.extraction_confidence,
    target.processed_at = source.processed_at
WHEN NOT MATCHED THEN
  INSERT (invoice_id, vendor_type, invoice_date, total_amount_cents,
          restaurant_name, extraction_confidence, processed_at)
  VALUES (source.invoice_id, source.vendor_type, source.invoice_date,
          source.total_amount_cents, source.restaurant_name,
          source.extraction_confidence, source.processed_at);
```

### Table Maintenance

```sql
-- Compaction (rewrite small files into larger ones)
OPTIMIZE invoices_dev.extracted_invoices
REWRITE DATA USING BIN_PACK
WHERE vendor_type = 'ubereats';   -- partition column only in WHERE

-- Vacuum (expire old snapshots + orphan files)
VACUUM invoices_dev.extracted_invoices;
```

VACUUM uses `vacuum_max_snapshot_age_seconds` (default: 5 days) and `vacuum_min_snapshots_to_keep` (default: 1).

### Athena Limits

| Limit | Value |
|-------|-------|
| Max partitions per INSERT/CTAS | 100 |
| Max columns | 1200 |
| Query timeout | 30 minutes |
| Concurrent queries per workgroup | Default 20 (Service Quota, raiseable) |
| OPTIMIZE WHERE clause | Partition columns only |

## Related

- [athena-engine-v3](athena-engine-v3.md)
- [glue-catalog](glue-catalog.md)
- [pyiceberg-alternative](pyiceberg-alternative.md)
- [create-iceberg-table pattern](../patterns/create-iceberg-table.md)
- [lambda-insert-batch pattern](../patterns/lambda-insert-batch.md)
- [Iceberg v3 internals](../../../lakehouse/concepts/iceberg-v3.md)
- [Iceberg operations](../../../lakehouse/patterns/iceberg-operations.md)
