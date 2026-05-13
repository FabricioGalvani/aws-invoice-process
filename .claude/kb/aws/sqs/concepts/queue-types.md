> **MCP Validated:** 2026-05-12

# SQS Queue Types — Standard vs FIFO

## Decision for This Pipeline

**Use Standard queues for all 4 pipeline queues.**

---

## Comparison

| Attribute | Standard | FIFO |
|-----------|----------|------|
| Throughput | Unlimited | 3 000 msg/s (300 msg/s without batching) |
| Ordering | Best-effort (not guaranteed) | Strict per MessageGroupId |
| Delivery | At-least-once (rare duplicates) | Exactly-once (deduplication ID) |
| Lambda concurrency model | Scales to 1 000 pollers | 1 poller per active MessageGroupId |
| Cost | Lower | ~10% higher |
| Name suffix | none | `.fifo` required |

---

## Why Standard Fits This Pipeline

### Throughput is not a concern

Volume: 2 000–3 500 invoices/month ≈ 117 invoices/hour at peak.
Standard queues handle millions per second. FIFO's 3 000 msg/s limit is irrelevant here.

### Ordering is enforced at the application level, not queue level

Each invoice flows through a 4-stage pipeline independently. The pipeline does not require one invoice to complete before the next starts. S3 keys carry enough identity (invoice_id) to correlate events without FIFO ordering.

### Duplicates are handled by idempotency

The data-extractor Lambda writes to S3/Iceberg using `invoice_id` as the partition key. A duplicate SQS delivery results in an idempotent upsert, not data corruption.

### ReportBatchItemFailures is simpler on Standard

FIFO queues require stopping processing at the first failure within a MessageGroup and returning all subsequent messages as failures too (to preserve order). Standard queues allow independent failure reporting per message within a batch, which is simpler and more efficient.

---

## When You Would Use FIFO Instead

- Invoice updates must be processed in strict arrival order (e.g., void after payment)
- Exactly-once semantics are legally required and idempotency is insufficient
- A downstream system cannot handle the same logical event twice under any circumstance

None of these conditions apply to this pipeline.

---

## Terraform Resource

```hcl
# Standard queue (no .fifo suffix, no ContentBasedDeduplication)
resource "aws_sqs_queue" "invoice_uploaded" {
  name                       = "invoice-uploaded"
  visibility_timeout_seconds = 1800   # 6 × 300s Lambda timeout
  message_retention_seconds  = 86400  # 1 day
  receive_wait_time_seconds  = 20     # long polling
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.invoice_uploaded_dlq.arn
    maxReceiveCount     = 3
  })
}
```

---

## References

- [AWS: Choosing between SQS queue types](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-queue-types.html)
- [AWS: FIFO queue Lambda concurrency behavior](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/fifo-queue-lambda-behavior.html)
