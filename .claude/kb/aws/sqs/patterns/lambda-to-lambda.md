> **MCP Validated:** 2026-05-12

# Pattern: Lambda-to-Lambda via SQS

## Overview

A producer Lambda completes its work, serialises a structured JSON pointer message, and sends it to the next queue. A consumer Lambda is triggered by the SQS event source mapping on that queue.

This is the core messaging pattern for all 4 stages of the invoice pipeline.

---

## Full Example — tiff-to-png-converter (Producer)

```python
# src/functions/tiff_to_png_converter/handler.py
import boto3
import json
import os
from typing import Any

sqs = boto3.client("sqs", region_name=os.environ["AWS_REGION"])
NEXT_QUEUE_URL = os.environ["INVOICE_CONVERTED_QUEUE_URL"]


def handler(event: dict, context: Any) -> dict:
    failures = []

    for record in event["Records"]:
        message_id = record["messageId"]
        try:
            body = json.loads(record["body"])
            invoice_id = body["invoice_id"]
            source_key = body["s3_key"]

            # --- business logic ---
            png_key = convert_tiff_to_png(
                bucket=body["s3_bucket"],
                key=source_key,
                invoice_id=invoice_id,
            )
            # --- end business logic ---

            # Produce next message AFTER successful processing
            _send_next(invoice_id, png_key)

        except Exception:
            # Do not raise — return itemIdentifier so Lambda ESM handles retry
            failures.append({"itemIdentifier": message_id})

    return {"batchItemFailures": failures}


def _send_next(invoice_id: str, png_key: str) -> None:
    sqs.send_message(
        QueueUrl=NEXT_QUEUE_URL,
        MessageBody=json.dumps({
            "invoice_id": invoice_id,
            "s3_bucket": "invoices-processed-dev",
            "s3_key": png_key,
            "vendor": "ubereats",          # passed through
            "processing_stage": "converted",
        }),
    )
```

---

## Consumer Side — invoice-classifier

The consumer Lambda receives the same record structure. It does NOT call `receive_message` or `delete_message`.

```python
# src/functions/invoice_classifier/handler.py
import json
from typing import Any


def handler(event: dict, context: Any) -> dict:
    failures = []

    for record in event["Records"]:
        try:
            body = json.loads(record["body"])
            # Validate presence of required fields BEFORE processing
            _validate(body, required=["invoice_id", "s3_bucket", "s3_key"])
            result = classify_invoice(body["s3_bucket"], body["s3_key"])
            _send_classified(body, result)
        except (KeyError, ValueError):
            # Structural errors — will DLQ after maxReceiveCount retries
            failures.append({"itemIdentifier": record["messageId"]})
        except Exception:
            failures.append({"itemIdentifier": record["messageId"]})

    return {"batchItemFailures": failures}


def _validate(body: dict, required: list[str]) -> None:
    missing = [k for k in required if k not in body]
    if missing:
        raise ValueError(f"Missing fields: {missing}")
```

---

## Environment Variables Convention

| Variable | Set to | Used by |
|----------|--------|---------|
| `INVOICE_UPLOADED_QUEUE_URL` | SQS queue URL | EventBridge → Lambda 1 |
| `INVOICE_CONVERTED_QUEUE_URL` | SQS queue URL | Lambda 1 producer |
| `INVOICE_CLASSIFIED_QUEUE_URL` | SQS queue URL | Lambda 2 producer |
| `INVOICE_EXTRACTED_QUEUE_URL` | SQS queue URL | Lambda 3 producer |

Queue URLs are injected via Terraform → Lambda environment block. Never hardcode them.

---

## Sequencing Rule

**Parse and validate the message body BEFORE any side-effectful work.**
If parsing fails, mark the record as a failure immediately without attempting S3 or Bedrock calls. This prevents orphaned resources and keeps DLQ messages interpretable.

---

## References

- [AWS: Lambda SQS event source mapping](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html)
- [AWS: SQS send_message boto3](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/sqs/client/send_message.html)
