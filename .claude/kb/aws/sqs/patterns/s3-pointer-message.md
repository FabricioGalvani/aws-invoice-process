> **MCP Validated:** 2026-05-12

# Pattern: S3 Pointer Message

## Problem

SQS hard limit: **256 KB per message**. Invoice files (TIFF, PNG) can be several megabytes. Sending file content directly in the message body will raise `InvalidMessageContents` from the SQS API.

## Solution

Store the binary payload in S3. Send only a lightweight JSON pointer (S3 bucket + key + metadata) in the SQS message. The consumer Lambda fetches the object from S3 directly.

This is the canonical pattern for this pipeline — all 4 stages exchange S3 references, never file bytes.

---

## Message Schema (all queues)

```json
{
  "invoice_id": "inv-2026-05-0042",
  "s3_bucket": "invoices-processed-dev",
  "s3_key": "2026/05/inv-2026-05-0042.png",
  "vendor": "ubereats",
  "processing_stage": "converted",
  "uploaded_at": "2026-05-12T14:32:00Z"
}
```

Typical size: ~250 bytes. Well within the 256 KB limit. The message body will never approach the limit as long as it contains only identifiers and metadata.

---

## Producer — Sending the Pointer

```python
import boto3
import json
import os
from datetime import datetime, timezone

s3 = boto3.client("s3", region_name=os.environ["AWS_REGION"])
sqs = boto3.client("sqs", region_name=os.environ["AWS_REGION"])


def upload_and_enqueue(
    file_bytes: bytes,
    invoice_id: str,
    vendor: str,
    dest_bucket: str,
    dest_key: str,
    next_queue_url: str,
) -> None:
    # 1. Write binary to S3 FIRST
    s3.put_object(
        Bucket=dest_bucket,
        Key=dest_key,
        Body=file_bytes,
        ContentType="image/png",
    )

    # 2. Send pointer to SQS AFTER S3 write succeeds
    sqs.send_message(
        QueueUrl=next_queue_url,
        MessageBody=json.dumps({
            "invoice_id": invoice_id,
            "s3_bucket": dest_bucket,
            "s3_key": dest_key,
            "vendor": vendor,
            "processing_stage": "converted",
            "uploaded_at": datetime.now(timezone.utc).isoformat(),
        }),
    )
```

Order matters: write to S3 before sending the SQS message. If S3 write fails, no message is enqueued and the item stays in the source queue for retry. If SQS send fails after a successful S3 write, the message is retried; the consumer will find the S3 object already present (idempotent read).

---

## Consumer — Fetching from S3

```python
import boto3
import json
import os
from typing import Any

s3 = boto3.client("s3", region_name=os.environ["AWS_REGION"])


def handler(event: dict, context: Any) -> dict:
    failures = []

    for record in event["Records"]:
        try:
            body = json.loads(record["body"])
            invoice_id = body["invoice_id"]

            # Fetch binary from S3 using the pointer
            response = s3.get_object(
                Bucket=body["s3_bucket"],
                Key=body["s3_key"],
            )
            file_bytes = response["Body"].read()

            process(invoice_id, file_bytes, body["vendor"])

        except Exception:
            failures.append({"itemIdentifier": record["messageId"]})

    return {"batchItemFailures": failures}
```

---

## IAM Requirements for This Pattern

The producer Lambda needs `s3:PutObject` on the destination bucket.
The consumer Lambda needs `s3:GetObject` on the source bucket for its stage.

```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject"],
  "Resource": "arn:aws:s3:::invoices-processed-dev/*"
}
```

See [specs/sqs-iam-policy.json](../specs/sqs-iam-policy.json) for complete per-Lambda policy examples.

---

## When the S3 Extended Client Library Is Needed

The manual pointer pattern above is sufficient for this pipeline. The `amazon-sqs-python-extended-client-lib` (AWS Labs) automates pointer serialisation for payloads between 256 KB and 2 GB and is needed only if:

- Message metadata itself approaches 256 KB (not the case here)
- The team wants automatic transparent handling without manual S3 calls

For this pipeline, manual pointer sending is preferred for transparency and reduced dependencies.

---

## References

- [AWS: SQS message quotas (256 KB)](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/quotas-messages.html)
- [AWS: Managing large SQS messages using S3](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-s3-messages.html)
- [AWS Labs: amazon-sqs-python-extended-client-lib](https://github.com/awslabs/amazon-sqs-python-extended-client-lib)
