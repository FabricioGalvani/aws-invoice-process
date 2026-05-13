# Lambda Insert Batch (Athena SQL Path)

> **MCP Validated:** 2026-05-12
> **Purpose**: warehouse-writer Lambda pattern — batch INSERT INTO Iceberg via Athena start_query_execution with async polling
> **Confidence**: 0.95
> **Sources**: [Athena boto3 start_query_execution](https://docs.aws.amazon.com/athena/latest/ug/querying-with-the-api.html), [AWS prescriptive guidance pyiceberg on AWS](https://docs.aws.amazon.com/prescriptive-guidance/latest/apache-iceberg-on-aws/iceberg-pyiceberg.html)

## Overview

The `warehouse-writer` Lambda consumes SQS messages from the `invoice-extracted` queue (one message per extracted invoice). It batches messages from a single Lambda invocation (SQS batch size: up to 10) and writes them to the Iceberg table via a single `INSERT INTO` Athena query.

**Write path:** `SQS batch → Lambda → Athena start_query_execution (INSERT INTO) → poll → S3 Iceberg table`

This pattern keeps the Lambda simple (no pyiceberg dependency layer needed) but requires async polling.

## Pattern

```python
import json
import time
import boto3
from typing import Any

REGION       = "us-east-1"
WORKGROUP    = "warehouse-writer-prod"
DATABASE     = "invoices_prod"
TABLE        = "extracted_invoices"
POLL_INTERVAL = 2   # seconds
MAX_POLLS     = 30  # 60s max wait

athena = boto3.client("athena", region_name=REGION)


def handler(event: dict, context: Any) -> dict:
    records = _parse_sqs_records(event["Records"])
    if not records:
        return {"statusCode": 200, "body": "no records"}

    sql = _build_insert_sql(records)
    query_id = _start_query(sql)
    _poll_until_complete(query_id)

    return {"statusCode": 200, "body": f"inserted {len(records)} rows"}


def _parse_sqs_records(sqs_records: list) -> list[dict]:
    rows = []
    for rec in sqs_records:
        body = json.loads(rec["body"])
        rows.append({
            "invoice_id":            body["invoice_id"],
            "vendor_type":           body["vendor_type"],
            "invoice_date":          body["invoice_date"],           # 'YYYY-MM-DD'
            "period_start":          body.get("period_start"),
            "period_end":            body.get("period_end"),
            "gross_sales_cents":     body.get("gross_sales_cents"),
            "commission_cents":      body.get("commission_cents"),
            "delivery_fee_cents":    body.get("delivery_fee_cents"),
            "adjustments_cents":     body.get("adjustments_cents"),
            "net_payout_cents":      body.get("net_payout_cents"),
            "restaurant_name":       body.get("restaurant_name"),
            "restaurant_id":         body.get("restaurant_id"),
            "extraction_confidence": body.get("extraction_confidence"),
            "llm_model":             body.get("llm_model"),
            "source_s3_key":         body["source_s3_key"],
        })
    return rows


def _escape(val: Any) -> str:
    """Produce SQL literal for a Python value."""
    if val is None:
        return "NULL"
    if isinstance(val, bool):
        return "TRUE" if val else "FALSE"
    if isinstance(val, (int, float)):
        return str(val)
    # String: escape single quotes
    return "'" + str(val).replace("'", "''") + "'"


def _build_insert_sql(rows: list[dict]) -> str:
    values_clauses = []
    for r in rows:
        values_clauses.append(
            f"({_escape(r['invoice_id'])}, {_escape(r['vendor_type'])}, "
            f"DATE {_escape(r['invoice_date'])}, "
            f"DATE {_escape(r.get('period_start'))}, DATE {_escape(r.get('period_end'))}, "
            f"{_escape(r.get('gross_sales_cents'))}, {_escape(r.get('commission_cents'))}, "
            f"{_escape(r.get('delivery_fee_cents'))}, {_escape(r.get('adjustments_cents'))}, "
            f"{_escape(r.get('net_payout_cents'))}, "
            f"{_escape(r.get('restaurant_name'))}, {_escape(r.get('restaurant_id'))}, "
            f"{_escape(r.get('extraction_confidence'))}, {_escape(r.get('llm_model'))}, "
            f"{_escape(r['source_s3_key'])}, current_timestamp)"
        )
    return (
        f"INSERT INTO {DATABASE}.{TABLE} "
        "(invoice_id, vendor_type, invoice_date, period_start, period_end, "
        "gross_sales_cents, commission_cents, delivery_fee_cents, adjustments_cents, "
        "net_payout_cents, restaurant_name, restaurant_id, extraction_confidence, "
        "llm_model, source_s3_key, processed_at) VALUES\n"
        + ",\n".join(values_clauses)
    )


def _start_query(sql: str) -> str:
    response = athena.start_query_execution(
        QueryString=sql,
        WorkGroup=WORKGROUP,
        QueryExecutionContext={"Database": DATABASE},
    )
    return response["QueryExecutionId"]


def _poll_until_complete(query_id: str) -> None:
    for _ in range(MAX_POLLS):
        result = athena.get_query_execution(QueryExecutionId=query_id)
        state = result["QueryExecution"]["Status"]["State"]
        if state == "SUCCEEDED":
            return
        if state in ("FAILED", "CANCELLED"):
            reason = result["QueryExecution"]["Status"].get("StateChangeReason", "")
            raise RuntimeError(f"Athena query {query_id} {state}: {reason}")
        time.sleep(POLL_INTERVAL)
    raise TimeoutError(f"Athena query {query_id} did not complete in time")
```

## SQS Event Source Configuration

```hcl
resource "aws_lambda_event_source_mapping" "warehouse_writer" {
  event_source_arn                   = aws_sqs_queue.invoice_extracted.arn
  function_name                      = aws_lambda_function.warehouse_writer.arn
  batch_size                         = 10        # up to 10 invoices per invocation
  maximum_batching_window_in_seconds = 30        # wait up to 30s to fill batch
  function_response_types            = ["ReportBatchItemFailures"]
}
```

`ReportBatchItemFailures` lets Lambda report partial batch failures — rows that failed to parse go back to SQS for retry without failing successfully-written rows.

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| One Athena query per row | 1 query × 3,500 invoices/month = 3,500 Athena queries; each has startup overhead | Batch via SQS window; batch_size=10 |
| Synchronous Lambda waiting on Athena (no timeout guard) | Lambda timeout (max 15min) if Athena is slow | Always poll with MAX_POLLS guard; raise TimeoutError |
| No workgroup set | Results go to default location; no cost controls | Always set `WorkGroup=` |
| Building SQL by f-string concatenation without escaping | SQL injection via malformed invoice data | Use `_escape()` helper or parameterized approach |

## Related

- [pyiceberg-write pattern](pyiceberg-write.md) — alternative synchronous write path
- [create-iceberg-table pattern](create-iceberg-table.md)
- [partition-and-prune pattern](partition-and-prune.md)
- [athena-iam-policy spec](../specs/athena-iam-policy.json)
