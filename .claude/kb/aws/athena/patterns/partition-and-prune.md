# Partition Design and Cost Optimization

> **MCP Validated:** 2026-05-12
> **Purpose**: Partition strategy for extracted_invoices, cost impact of partition pruning, query patterns for the invoice pipeline
> **Confidence**: 0.95
> **Sources**: [Athena Iceberg hidden partitioning](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg-creating-tables.html), [OLake Athena+Iceberg overview](https://olake.io/iceberg/query-engine/athena/), [AWS Athena pricing](https://aws.amazon.com/athena/pricing/)

## Cost Model

Athena charges **$5.00 per TB scanned**. For the invoice pipeline:

| Scenario | Data scanned | Cost/query |
|----------|-------------|-----------|
| Full table scan (no partition pruning) | 100% of all data | ~$0.0001 at current volume |
| With vendor_type filter | ~33% scanned | ~$0.00003 |
| With vendor_type + date range filter | <1% scanned | <$0.000001 |

At 3,500 invoices/month, total table size is tiny (~50MB Parquet). Partition pruning matters more when:
- Historical queries span years of data
- BI tools run repeated dashboard queries without date filters
- Compliance/audit queries scan full history

Design for the future, not just current volume.

## Partition Spec for extracted_invoices

```sql
PARTITIONED BY (
  vendor_type,       -- identity transform: ubereats | doordash | grubhub
  day(invoice_date)  -- day transform on DATE column
)
```

### Physical Layout on S3

```
s3://invoices-warehouse-prod/extracted_invoices/
  data/
    vendor_type=ubereats/
      invoice_date_day=2026-05-01/
        00001-abc123.parquet
      invoice_date_day=2026-05-02/
        00001-def456.parquet
    vendor_type=doordash/
      invoice_date_day=2026-05-01/
        00001-ghi789.parquet
```

Iceberg tracks this layout in manifest files — users never write `vendor_type=ubereats` in queries. The engine reads the partition spec from Glue metadata and translates predicates automatically.

## Writing Partition-Aware Queries

```sql
-- GOOD: predicate on partition columns → Athena prunes to 1 vendor, 30 day-partitions
SELECT
  invoice_id,
  restaurant_name,
  net_payout_cents / 100.0 AS net_payout_usd
FROM invoices_prod.extracted_invoices
WHERE vendor_type = 'ubereats'                     -- hits vendor_type partition
  AND invoice_date BETWEEN DATE '2026-04-01'
                       AND DATE '2026-04-30';       -- hits day() partitions

-- GOOD: GROUP BY on partition columns → full pruning applies
SELECT
  vendor_type,
  DATE_TRUNC('month', invoice_date) AS month,
  SUM(net_payout_cents) / 100.0 AS total_payout_usd,
  COUNT(*) AS invoice_count
FROM invoices_prod.extracted_invoices
WHERE invoice_date >= DATE '2026-01-01'
GROUP BY vendor_type, DATE_TRUNC('month', invoice_date)
ORDER BY month, vendor_type;

-- BAD: function on partition column breaks pruning
-- day(invoice_date) partition not hit when filtering on MONTH(invoice_date)
SELECT * FROM invoices_prod.extracted_invoices
WHERE MONTH(invoice_date) = 4;  -- full scan!
-- FIX: use explicit date range
WHERE invoice_date BETWEEN DATE '2026-04-01' AND DATE '2026-04-30'

-- BAD: no vendor_type filter when you know it
SELECT * FROM invoices_prod.extracted_invoices
WHERE invoice_date = DATE '2026-05-01';  -- scans all 3 vendor partitions
-- FIX: add vendor_type = '...' if known
```

## Partition Evolution (future-proofing)

If invoice volume grows significantly (e.g. 50,000+/month) and day partitions get too large, add a `month()` transform instead of `day()`:

```sql
ALTER TABLE invoices_prod.extracted_invoices
  DROP PARTITION FIELD day(invoice_date);
ALTER TABLE invoices_prod.extracted_invoices
  ADD PARTITION FIELD month(invoice_date);
-- Old data stays in day partitions; new data in month partitions
-- Athena queries both seamlessly via Iceberg metadata
```

This is a metadata-only operation — no data rewrite needed. This is the key advantage of Iceberg hidden partitioning over Hive-style partitioning.

## Compaction Schedule

Small files (each Lambda invocation writes 1 small Parquet file) accumulate and increase metadata overhead. Run OPTIMIZE periodically:

```sql
-- Compact all small files in a specific vendor partition
OPTIMIZE invoices_prod.extracted_invoices
REWRITE DATA USING BIN_PACK
WHERE vendor_type = 'ubereats';   -- partition column only

-- Or schedule per-vendor compaction via EventBridge Scheduler
```

Recommended cadence: weekly compaction per vendor partition. At current volume (2-3k/month), monthly is sufficient.

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| No partitioning at all | Full table scan on every query | Use `vendor_type + day(invoice_date)` |
| Hive-style `PARTITIONED BY (vendor_type STRING, invoice_date STRING)` | Partition column in schema; predicate must match exactly; no transforms | Use Iceberg hidden transforms |
| High-cardinality identity partition (e.g. `restaurant_id`) | Too many partitions; Iceberg metadata explodes | Use low-cardinality columns for identity; use transforms for dates |
| Writing one row per Athena INSERT | 3,500 files/month; many tiny files; slow compaction | Batch minimum 10-50 rows per INSERT |
| OPTIMIZE without WHERE on large table | Full table rewrite; expensive | Always scope to specific partition in WHERE |

## Related

- [create-iceberg-table pattern](create-iceberg-table.md)
- [lambda-insert-batch pattern](lambda-insert-batch.md)
- [athena-engine-v3 concept](../concepts/athena-engine-v3.md)
- [Iceberg v3 internals](../../../lakehouse/concepts/iceberg-v3.md)
