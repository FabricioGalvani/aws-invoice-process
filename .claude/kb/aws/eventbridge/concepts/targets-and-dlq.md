> **MCP Validated:** 2026-05-12

# EventBridge Targets and Dead-Letter Queues

## Concept Overview

A **target** is the resource EventBridge invokes when a rule matches an event.
Each rule can have up to 5 targets. Each target has its own retry policy and
optionally its own dead-letter queue (DLQ).

---

## Target Types (commonly used)

| Target | Use case |
|---|---|
| Amazon SQS queue | Durable buffering before Lambda; fan-in with backpressure |
| AWS Lambda function | Synchronous invocation directly from EventBridge |
| Amazon SNS topic | Fan-out to email, SMS, HTTP endpoints |
| EventBridge event bus | Cross-account / cross-region routing |
| AWS Step Functions | Orchestrated workflows |
| Amazon Kinesis stream | High-throughput streaming |

**This pipeline uses SQS as the primary target** so Lambda can control its own
consumption rate and benefit from SQS visibility timeouts and its own DLQ.

---

## Retry Policy (per target)

EventBridge retries delivery to a target using exponential backoff with jitter.

| Setting | Default | Maximum |
|---|---|---|
| Maximum retry attempts | 185 | 185 |
| Maximum event age | 24 hours | 24 hours |

If the target returns an error and retries are exhausted, the event is either
**dropped** or sent to the **target-level DLQ** (if configured).

Errors that skip retries entirely (no retry is attempted):
- Missing IAM permissions on the target
- Target resource does not exist
- Invalid target ARN / DNS failure

---

## Two DLQ Layers (do not confuse them)

```
S3 → EventBridge Rule → [Rule-Target DLQ] → SQS Queue → Lambda
                                                └── [SQS DLQ]
```

| DLQ | Where configured | What it catches |
|---|---|---|
| **Rule-target DLQ** | On the EventBridge rule target | Events EventBridge cannot deliver to SQS (permissions error, queue deleted, retries exhausted) |
| **SQS DLQ** | On the SQS queue itself | Messages Lambda cannot process after `maxReceiveCount` |

Both are standard SQS queues. Both must exist before the rule or queue is created.

### Rule-Target DLQ Configuration (Terraform)

```hcl
resource "aws_cloudwatch_event_target" "invoice_sqs" {
  rule      = aws_cloudwatch_event_rule.invoice_uploaded.name
  target_id = "SendToInvoiceUploadedQueue"
  arn       = aws_sqs_queue.invoice_uploaded.arn

  dead_letter_config {
    arn = aws_sqs_queue.eventbridge_dlq.arn
  }

  retry_policy {
    maximum_retry_attempts       = 185
    maximum_event_age_in_seconds = 86400  # 24 h
  }
}
```

---

## IAM: EventBridge → SQS Permission

EventBridge does not use an IAM role to send to SQS. Instead, the **SQS queue
must have a resource-based policy** (queue policy) granting `sqs:SendMessage` to
the EventBridge service principal, scoped to the specific rule ARN.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowEventBridgeToSendMessage",
    "Effect": "Allow",
    "Principal": { "Service": "events.amazonaws.com" },
    "Action": "sqs:SendMessage",
    "Resource": "arn:aws:sqs:us-east-1:123456789012:invoice-uploaded",
    "Condition": {
      "ArnEquals": {
        "aws:SourceArn": "arn:aws:events:us-east-1:123456789012:rule/InvoiceUploaded"
      }
    }
  }]
}
```

> The `aws:SourceArn` condition is mandatory to prevent confused-deputy attacks.

---

## References

- [EventBridge targets](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-targets.html)
- [EventBridge DLQ](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-rule-dlq.html)
- [Retry policy docs](https://repost.aws/knowledge-center/eventbridge-resolve-failedinvocation-errors)
