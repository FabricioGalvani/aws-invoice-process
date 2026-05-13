# PyIceberg Write from Lambda

> **MCP Validated:** 2026-05-12
> **Purpose**: Direct Iceberg write from warehouse-writer Lambda using pyiceberg + GlueCatalog — no Athena query needed
> **Confidence**: 0.95
> **Sources**: [AWS blog: pyiceberg + Lambda + Glue REST](https://aws.amazon.com/blogs/big-data/accelerate-lightweight-analytics-using-pyiceberg-with-aws-lambda-and-an-aws-glue-iceberg-rest-endpoint/), [PyIceberg on AWS Lambda (DEV Community)](https://dev.to/aws-builders/pyiceberg-on-aws-lambda-comparing-gluecatalog-and-rest-catalog-access-methods-4bl8), [pyiceberg GlueCatalog reference](https://py.iceberg.apache.org/reference/pyiceberg/catalog/glue/)

## Overview

Alternative to the Athena async path. The Lambda writes directly to S3 using pyiceberg, then atomically updates the Glue catalog metadata pointer. Advantages: synchronous (no polling), lower latency for small payloads, no Athena service quota consumption.

**Write path:** `SQS batch → Lambda → pyiceberg → S3 (Parquet data + metadata files) + Glue UpdateTable`

## Dependencies (Lambda Layer or Container)

```
# requirements.txt
pyiceberg[glue,s3fs]==0.8.0   # pinned; includes pyarrow, s3fs, botocore
pyarrow==16.0.0
```

Recommended deployment: Lambda container image (pyiceberg with s3fs is ~80MB; fits Lambda 250MB layer limit but only just). Container image avoids layer size constraint entirely.

## Pattern

```python
import json
import pyarrow as pa
from pyiceberg.catalog.glue import GlueCatalog
from pyiceberg.exceptions import CommitFailedException
import time
import os
from typing import Any

REGION   = os.environ.get("AWS_REGION", "us-east-1")
DATABASE = os.environ.get("WAREHOUSE_DATABASE", "invoices_prod")
TABLE    = "extracted_invoices"
MAX_RETRIES = 3


def _get_catalog() -> GlueCatalog:
    """Initialized once per Lambda execution environment (warm start reuse)."""
    return GlueCatalog(
        "invoices",
        **{"glue.region": REGION}
        # Credentials automatically from Lambda execution role
    )


# Module-level: reused across warm invocations
_catalog = _get_catalog()


def handler(event: dict, context: Any) -> dict:
    rows = _parse_sqs_records(event["Records"])
    if not rows:
        return {"statusCode": 200}

    arrow_table = _rows_to_arrow(rows)
    _write_with_retry(arrow_table)
    return {"statusCode": 200, "body": f"wrote {len(rows)} rows"}


def _parse_sqs_records(sqs_records: list) -> list[dict]:
    rows = []
    for rec in sqs_records:
        body = json.loads(rec["body"])
        rows.append(body)
    return rows


def _rows_to_arrow(rows: list[dict]) -> pa.Table:
    """Convert list of extracted invoice dicts to PyArrow table."""
    import pyarrow as pa
    from datetime import date, datetime, timezone

    # Coerce types — invoice_date from string to date
    for r in rows:
        if isinstance(r.get("invoice_date"), str):
            r["invoice_date"] = date.fromisoformat(r["invoice_date"])
        if isinstance(r.get("period_start"), str):
            r["period_start"] = date.fromisoformat(r["period_start"])
        if isinstance(r.get("period_end"), str):
            r["period_end"] = date.fromisoformat(r["period_end"])
        r["processed_at"] = datetime.now(tz=timezone.utc)

    schema = pa.schema([
        pa.field("invoice_id",            pa.string(),  nullable=False),
        pa.field("vendor_type",           pa.string(),  nullable=False),
        pa.field("invoice_date",          pa.date32(),  nullable=False),
        pa.field("period_start",          pa.date32(),  nullable=True),
        pa.field("period_end",            pa.date32(),  nullable=True),
        pa.field("gross_sales_cents",     pa.int64(),   nullable=True),
        pa.field("commission_cents",      pa.int64(),   nullable=True),
        pa.field("delivery_fee_cents",    pa.int64(),   nullable=True),
        pa.field("adjustments_cents",     pa.int64(),   nullable=True),
        pa.field("net_payout_cents",      pa.int64(),   nullable=True),
        pa.field("restaurant_name",       pa.string(),  nullable=True),
        pa.field("restaurant_id",         pa.string(),  nullable=True),
        pa.field("extraction_confidence", pa.float64(), nullable=True),
        pa.field("llm_model",             pa.string(),  nullable=True),
        pa.field("source_s3_key",         pa.string(),  nullable=False),
        pa.field("processed_at",          pa.timestamp("us", tz="UTC"), nullable=False),
    ])
    return pa.Table.from_pylist(rows, schema=schema)


def _write_with_retry(arrow_table: pa.Table) -> None:
    table = _catalog.load_table(f"{DATABASE}.{TABLE}")
    for attempt in range(1, MAX_RETRIES + 1):
        try:
            table.append(arrow_table)
            return
        except CommitFailedException as exc:
            if attempt == MAX_RETRIES:
                raise
            wait = 0.5 * (2 ** attempt)   # 1s, 2s, 4s
            time.sleep(wait)
            # Reload table metadata (catalog pointer may have advanced)
            table = _catalog.load_table(f"{DATABASE}.{TABLE}")
```

## SQS Concurrency Control

Set `MaximumConcurrency = 1` on the event source mapping to eliminate `CommitFailedException`:

```hcl
resource "aws_lambda_event_source_mapping" "warehouse_writer_pyiceberg" {
  event_source_arn                   = aws_sqs_queue.invoice_extracted.arn
  function_name                      = aws_lambda_function.warehouse_writer.arn
  batch_size                         = 10
  maximum_batching_window_in_seconds = 30
  scaling_config {
    maximum_concurrency = 1   # serializes writes; safe at invoice pipeline volume
  }
  function_response_types = ["ReportBatchItemFailures"]
}
```

## Catalog Initialization Trade-off

| Approach | Cold start | Warm reuse | Notes |
|----------|-----------|-----------|-------|
| Module-level `_catalog` | ~200ms overhead | Free | Recommended; boto3 session cached |
| Per-invocation `GlueCatalog(...)` | Repeated boto3 init | Wasted | Avoid |

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| No retry on `CommitFailedException` | Concurrent invocations cause data loss | Add exponential backoff retry OR set MaximumConcurrency=1 |
| Writing one row per invocation | 3,500 separate commits/month; excessive metadata | Batch via SQS window |
| Using explicit AWS credentials in code | Security risk; breaks IAM rotation | Let Lambda execution role credential chain handle auth |
| Installing pyiceberg[glue,s3fs] as plain Lambda layer without Docker | Likely > 250MB unzipped | Use container image |

## Related

- [pyiceberg-alternative concept](../concepts/pyiceberg-alternative.md)
- [lambda-insert-batch pattern](lambda-insert-batch.md) — Athena SQL write path
- [create-iceberg-table pattern](create-iceberg-table.md)
- [Iceberg operations](../../../lakehouse/patterns/iceberg-operations.md)
