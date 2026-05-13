> **MCP Validated:** 2026-05-12

# Pattern: Partial Batch Failure (ReportBatchItemFailures)

## Problem

By default, if a Lambda function throws an exception while processing an SQS batch, **all messages in the batch** become visible again and are redelivered — including the ones that processed successfully. This causes duplicate processing of already-handled invoices.

## Solution

Enable `ReportBatchItemFailures` on the event source mapping. The handler returns a `batchItemFailures` list containing only the `messageId` values that failed. Lambda deletes the successful messages and returns only the failed ones to the queue.

---

## Event Source Mapping Configuration

```hcl
# Terraform
resource "aws_lambda_event_source_mapping" "invoice_classified" {
  event_source_arn        = aws_sqs_queue.invoice_classified.arn
  function_name           = aws_lambda_function.data_extractor.arn
  batch_size              = 5
  function_response_types = ["ReportBatchItemFailures"]  # required
}
```

```python
# SAM / boto3 create_event_source_mapping
FunctionResponseTypes=["ReportBatchItemFailures"]
```

---

## Handler Contract

The function MUST return this shape (even on full success):

```python
# Full success
return {"batchItemFailures": []}

# Partial failure
return {
    "batchItemFailures": [
        {"itemIdentifier": "msg-id-that-failed-1"},
        {"itemIdentifier": "msg-id-that-failed-2"},
    ]
}

# All failed
return {
    "batchItemFailures": [
        {"itemIdentifier": r["messageId"]} for r in event["Records"]
    ]
}
```

**If the handler raises an unhandled exception, all messages in the batch are returned to the queue** regardless of this setting. Catch all exceptions inside the loop.

---

## Reference Implementation

```python
import json
import logging
from typing import Any

logger = logging.getLogger(__name__)


def handler(event: dict, context: Any) -> dict:
    failures: list[dict] = []

    for record in event["Records"]:
        message_id = record["messageId"]
        try:
            body = json.loads(record["body"])
            # Validate FIRST — before any I/O
            _validate_body(body)
            process_record(body)
            logger.info("processed", extra={"invoice_id": body["invoice_id"]})
        except json.JSONDecodeError:
            # Malformed message — will DLQ after maxReceiveCount, do not retry infinitely
            logger.error("json_decode_error", extra={"message_id": message_id})
            failures.append({"itemIdentifier": message_id})
        except ValidationError as exc:
            logger.error("validation_error", extra={"error": str(exc), "message_id": message_id})
            failures.append({"itemIdentifier": message_id})
        except Exception as exc:
            logger.error("processing_error", extra={"error": str(exc), "message_id": message_id})
            failures.append({"itemIdentifier": message_id})

    return {"batchItemFailures": failures}
```

---

## Failure Routing

```
Batch of 5 records
  ├─ record-1: success  → deleted by Lambda ESM
  ├─ record-2: failure  → returned to queue, receiveCount + 1
  ├─ record-3: success  → deleted
  ├─ record-4: failure  → returned to queue, receiveCount + 1
  └─ record-5: success  → deleted

If receiveCount for record-2 exceeds maxReceiveCount (3):
  └─ record-2 → DLQ
```

---

## Standard Queue Advantage

On Standard queues, failed records in a batch are returned **independently**. On FIFO queues, if record-2 fails, records 3–5 that share the same MessageGroupId must also be returned as failures (to preserve order), even if they succeeded. Standard queues avoid this complexity.

---

## References

- [AWS: Handling SQS errors in Lambda](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-errorhandling.html)
- [AWS: Partial batch failure best practices](https://docs.aws.amazon.com/prescriptive-guidance/latest/lambda-event-filtering-partial-batch-responses-for-sqs/best-practices-partial-batch-responses.html)
- [AWS: Reporting batch item failures (code example)](https://docs.aws.amazon.com/lambda/latest/dg/example_serverless_SQS_Lambda_batch_item_failures_section.html)
