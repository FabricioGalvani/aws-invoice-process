> **MCP Validated:** 2026-05-12

# Visibility Timeout — Sizing and Redelivery

## What It Is

When Lambda retrieves a message from SQS, SQS hides that message from other consumers for the visibility timeout duration. If Lambda does not delete the message (by completing successfully) before the timeout expires, SQS makes the message visible again — it will be redelivered.

---

## AWS Sizing Rule

```
visibility_timeout ≥ 6 × function_timeout + MaximumBatchingWindowInSeconds
```

AWS Lambda **validates** this at event source mapping creation time and returns an error if the function timeout exceeds the queue visibility timeout. The 6× factor accounts for:

- Function execution time
- Retry time if the function is throttled
- Lambda internal polling overhead

**Do not set them equal.** A 5-minute Lambda with a 5-minute visibility timeout will produce duplicates on any throttle or cold start.

---

## Pipeline Values

| Queue | Lambda Timeout | Visibility Timeout | Margin |
|-------|---------------|-------------------|--------|
| `invoice-uploaded` | 300 s (5 min) | 1800 s (30 min) | 6× |
| `invoice-converted` | 60 s (1 min) | 360 s (6 min) | 6× |
| `invoice-classified` | 120 s (2 min) | 720 s (12 min) | 6× |
| `invoice-extracted` | 60 s (1 min) | 360 s (6 min) | 6× |

---

## Redelivery Mechanics

```
Message received by Lambda poller
    │
    ├─ Function completes OK → Lambda deletes message → gone
    │
    └─ Function fails / times out
           │
           └─ Visibility timeout expires
                  │
                  └─ Message becomes visible again → redelivered
                         │
                         └─ receiveCount increments
                                │
                                └─ receiveCount > maxReceiveCount
                                       │
                                       └─ Message sent to DLQ
```

---

## ChangeMessageVisibility

If a message requires longer processing than expected, the Lambda handler can call `sqs:ChangeMessageVisibility` to extend the timeout mid-flight. This is rarely needed if the Lambda timeout and visibility timeout are sized correctly, but is a valid escape hatch.

```python
sqs.change_message_visibility(
    QueueUrl=queue_url,
    ReceiptHandle=record["receiptHandle"],
    VisibilityTimeout=600,  # extend by 10 more minutes
)
```

---

## Common Mistake — Too Short

Setting visibility timeout equal to Lambda timeout:

```
Lambda timeout: 5 min
Visibility timeout: 5 min   ← WRONG

Result: Any Lambda throttle or cold start causes the message to
become visible before Lambda finishes. A second Lambda receives
the same message → duplicate processing.
```

---

## Terraform Setting

```hcl
resource "aws_sqs_queue" "invoice_classified" {
  name                       = "invoice-classified"
  visibility_timeout_seconds = 720  # 6 × 120s
}
```

---

## References

- [AWS: SQS visibility timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html)
- [AWS: Lambda SQS event source mapping](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-configure.html)
