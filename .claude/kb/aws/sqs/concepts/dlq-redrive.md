> **MCP Validated:** 2026-05-12

# DLQ and Redrive Policy

## What a DLQ Is

A Dead-Letter Queue (DLQ) is a standard SQS queue that receives messages from a source queue after the message has been received (and not deleted) more than `maxReceiveCount` times. It is the poison-message quarantine for the pipeline.

---

## Redrive Policy

Attach a redrive policy to the **source** queue, pointing at the DLQ:

```json
{
  "deadLetterTargetArn": "arn:aws:sqs:us-east-1:ACCOUNT:invoice-uploaded-dlq",
  "maxReceiveCount": 3
}
```

The DLQ itself must have **no** redrive policy — it is the final stop.

---

## maxReceiveCount Guidance

| Value | When to use |
|-------|-------------|
| 1–2 | Almost never — one transient network error sends to DLQ |
| **3–5** | **Recommended for this pipeline** — tolerates transient failures, cold starts, Bedrock throttles |
| 6+ | Only for idempotent operations that must exhaust retries before investigation |

**Use 3 across all 4 queues.** Bedrock throttles are transient; 3 attempts is enough to distinguish a real failure from a spike.

---

## Pipeline DLQ Map

| Source Queue | DLQ | maxReceiveCount |
|-------------|-----|-----------------|
| `invoice-uploaded` | `invoice-uploaded-dlq` | 3 |
| `invoice-converted` | `invoice-converted-dlq` | 3 |
| `invoice-classified` | `invoice-classified-dlq` | 3 |
| `invoice-extracted` | `invoice-extracted-dlq` | 3 |

---

## Terraform Wiring

```hcl
resource "aws_sqs_queue" "invoice_uploaded_dlq" {
  name                      = "invoice-uploaded-dlq"
  message_retention_seconds = 1209600  # 14 days (max) for investigation
}

resource "aws_sqs_queue" "invoice_uploaded" {
  name                       = "invoice-uploaded"
  visibility_timeout_seconds = 1800
  message_retention_seconds  = 86400
  receive_wait_time_seconds  = 20
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.invoice_uploaded_dlq.arn
    maxReceiveCount     = 3
  })
}
```

---

## Redrive (Replay from DLQ)

To replay messages from the DLQ back to the source queue:

**Console:** SQS → DLQ → Start DLQ redrive → select source queue.

**CLI:**
```bash
aws sqs start-message-move-task \
  --source-arn arn:aws:sqs:us-east-1:ACCOUNT:invoice-uploaded-dlq \
  --destination-arn arn:aws:sqs:us-east-1:ACCOUNT:invoice-uploaded \
  --max-number-of-messages-per-second 5
```

Fix the root cause before redrive. Redriving before fixing sends messages back into the same failure loop.

---

## References

- [AWS: SQS dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
- [AWS: Troubleshoot DLQ redrive](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/troubleshooting-dlq-redrive.html)
