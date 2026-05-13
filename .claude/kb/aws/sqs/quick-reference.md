> **MCP Validated:** 2026-05-12

# SQS Quick Reference — Invoice Pipeline

## Queue Names (all environments)

| Queue | Consumer Lambda | DLQ |
|-------|----------------|-----|
| `invoice-uploaded` | `tiff-to-png-converter` | `invoice-uploaded-dlq` |
| `invoice-converted` | `invoice-classifier` | `invoice-converted-dlq` |
| `invoice-classified` | `data-extractor` | `invoice-classified-dlq` |
| `invoice-extracted` | `warehouse-writer` | `invoice-extracted-dlq` |

---

## Critical Settings Per Queue

| Queue | Lambda Timeout | Visibility Timeout | BatchSize | maxReceiveCount |
|-------|---------------|-------------------|-----------|-----------------|
| `invoice-uploaded` | 5 min (300 s) | 1800 s (30 min) | 5 | 3 |
| `invoice-converted` | 1 min (60 s) | 360 s (6 min) | 10 | 3 |
| `invoice-classified` | 2 min (120 s) | 720 s (12 min) | 5 | 3 |
| `invoice-extracted` | 1 min (60 s) | 360 s (6 min) | 10 | 3 |

Visibility timeout = 6 × Lambda timeout (AWS requirement).

---

## Producer Snippet (boto3)

```python
import boto3, json

sqs = boto3.client("sqs", region_name="us-east-1")

sqs.send_message(
    QueueUrl="https://sqs.us-east-1.amazonaws.com/ACCOUNT/invoice-converted",
    MessageBody=json.dumps({
        "invoice_id": "inv-001",
        "s3_bucket": "invoices-processed-dev",
        "s3_key": "2026/05/inv-001.png",
        "vendor": "ubereats",
    }),
)
```

---

## Consumer Lambda Handler Skeleton

```python
def handler(event, context):
    failures = []
    for record in event["Records"]:
        try:
            body = json.loads(record["body"])
            process(body)
        except Exception:
            failures.append({"itemIdentifier": record["messageId"]})
    return {"batchItemFailures": failures}  # ReportBatchItemFailures
```

---

## IAM Actions Summary

| Role | Needs | On |
|------|-------|----|
| Consumer Lambda | `sqs:ReceiveMessage` `sqs:DeleteMessage` `sqs:GetQueueAttributes` `sqs:ChangeMessageVisibility` | Its own queue ARN |
| Producer Lambda | `sqs:SendMessage` | Next queue ARN only |

---

## Anti-Patterns (never do)

- Visibility timeout < 6× Lambda timeout — causes duplicate processing
- No DLQ on any queue — messages silently lost on repeated failure
- `BatchSize=1` — wastes Lambda invocations at this throughput
- Call `sqs.receive_message()` inside handler — Lambda polls automatically
- Send PNG bytes in message body — exceeds 256 KB; send S3 key instead
