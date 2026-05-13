> **MCP Validated:** 2026-05-12

# Amazon SQS — Invoice Pipeline KB

Knowledge base for SQS usage between the 4 Lambda functions in the invoice processing pipeline. Replaces GCP Pub/Sub (decision D7 → AWS).

---

## Pipeline Queue Map

```
S3 Upload
    └─► EventBridge ─► invoice-uploaded        ─► Lambda: tiff-to-png-converter
                            └─► invoice-uploaded-dlq

tiff-to-png-converter ─► invoice-converted      ─► Lambda: invoice-classifier
                              └─► invoice-converted-dlq

invoice-classifier ─► invoice-classified        ─► Lambda: data-extractor
                          └─► invoice-classified-dlq

data-extractor ─► invoice-extracted             ─► Lambda: warehouse-writer
                      └─► invoice-extracted-dlq
```

4 main queues + 4 DLQs = 8 SQS queues total per environment.

---

## Navigation

### Concepts

| File | Covers |
|------|--------|
| [concepts/queue-types.md](concepts/queue-types.md) | Standard vs FIFO, why Standard fits this pipeline |
| [concepts/visibility-timeout.md](concepts/visibility-timeout.md) | Sizing rule: ≥ 6× Lambda timeout |
| [concepts/dlq-redrive.md](concepts/dlq-redrive.md) | DLQ wiring, maxReceiveCount, redrive |
| [concepts/lambda-event-source.md](concepts/lambda-event-source.md) | Event source mapping vs manual polling |

### Patterns

| File | Covers |
|------|--------|
| [patterns/lambda-to-lambda.md](patterns/lambda-to-lambda.md) | Producer Lambda → SQS → consumer Lambda |
| [patterns/partial-batch-failure.md](patterns/partial-batch-failure.md) | ReportBatchItemFailures |
| [patterns/dlq-with-alarm.md](patterns/dlq-with-alarm.md) | DLQ + CloudWatch alarm + Slack |
| [patterns/s3-pointer-message.md](patterns/s3-pointer-message.md) | Send S3 key reference, not payload |

### Specs

| File | Covers |
|------|--------|
| [specs/sqs-iam-policy.json](specs/sqs-iam-policy.json) | Least-privilege IAM per Lambda role |

---

## Key Numbers (memorise these)

| Parameter | Value | Source |
|-----------|-------|--------|
| Message size limit | 256 KB | AWS hard limit |
| Long poll wait | 20 s | `WaitTimeSeconds=20` |
| Visibility timeout rule | ≥ 6× function timeout | AWS guidance |
| Recommended maxReceiveCount | 3–5 | AWS best practice |
| Default BatchSize | 10 | Event source mapping default |
| Max BatchSize (Standard) | 10 000 | AWS limit |

---

## Quick Links

- [Quick Reference](quick-reference.md) — fast lookup card
- AWS Docs: [Using Lambda with SQS](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html)
- AWS Docs: [SQS Dead-Letter Queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
