> **MCP Validated:** 2026-05-12

# Pattern: Multi-Target Fan-Out (One Event → Multiple Targets)

## Pattern Overview

A single EventBridge rule can route one matching event to **up to 5 targets**
simultaneously. Each target receives the event (or a transformed version of it)
independently and in parallel. This enables fan-out without any custom routing code.

**Flow:**

```
S3 Object Created event
    │
    ▼  (single rule matches)
EventBridge Rule: invoice-uploaded
    ├──▶ Target 1: SQS invoice-uploaded (primary processing queue)
    ├──▶ Target 2: CloudWatch Logs (audit trail)
    └──▶ Target 3: SNS topic (ops notification on large file upload)
```

---

## When to Use Fan-Out

| Scenario | Fan-out appropriate? |
|---|---|
| Primary processing queue + audit log | Yes |
| Notify ops team AND queue for processing | Yes |
| Route same event to dev AND prod | No — use separate rules per env |
| 6+ consumers need the same event | No — use SNS as intermediary |
| Consumers have different filters | No — separate rules per consumer |

---

## Terraform: Multiple Targets on One Rule

Each target is a separate `aws_cloudwatch_event_target` resource sharing the
same rule name.

```hcl
# Target 1: Primary SQS processing queue
resource "aws_cloudwatch_event_target" "invoice_sqs" {
  rule      = aws_cloudwatch_event_rule.invoice_uploaded.name
  target_id = "InvoiceUploadedQueue"
  arn       = aws_sqs_queue.invoice_uploaded.arn

  input_transformer {
    input_paths = {
      bucket  = "$.detail.bucket.name"
      key     = "$.detail.object.key"
      eventId = "$.id"
    }
    input_template = <<-JSON
      {"bucket":"<bucket>","key":"<key>","eventId":"<eventId>","source":"eventbridge-s3"}
    JSON
  }

  dead_letter_config {
    arn = aws_sqs_queue.eventbridge_dlq.arn
  }
}

# Target 2: CloudWatch Logs for audit (raw event, no transform)
resource "aws_cloudwatch_event_target" "invoice_audit_log" {
  rule      = aws_cloudwatch_event_rule.invoice_uploaded.name
  target_id = "InvoiceAuditLog"
  arn       = "/aws/events/invoice-audit"  # CloudWatch Log Group ARN
}

# Target 3: SNS for ops notification (conditional on file size)
resource "aws_cloudwatch_event_target" "invoice_ops_notify" {
  rule      = aws_cloudwatch_event_rule.invoice_uploaded.name
  target_id = "InvoiceOpsNotify"
  arn       = aws_sns_topic.invoice_ops.arn

  input_transformer {
    input_paths = {
      bucket = "$.detail.bucket.name"
      key    = "$.detail.object.key"
      size   = "$.detail.object.size"
    }
    input_template = "New invoice uploaded: <key> (<size> bytes) in <bucket>"
  }
}
```

---

## Fan-Out vs SNS Fan-Out

| Approach | Max consumers | Filter per consumer | DLQ | Recommended when |
|---|---|---|---|---|
| EventBridge multi-target | 5 | Shared rule filter | Per-target | Small number of fixed targets |
| EventBridge → SNS → SQS | Unlimited | SNS filter policy | Per SQS | Many consumers, dynamic subscriptions |
| EventBridge → multiple rules | Unlimited | Per-rule filter | Per-target | Consumers need different event patterns |

For the invoice pipeline, the **single-rule → single-SQS-target** pattern (no fan-out)
is correct for the happy path. Fan-out is appropriate when adding observability
targets (audit log, metrics) without changing the primary processing target.

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| >5 targets on a single rule | Not supported — hard limit | Use SNS intermediary or multiple rules |
| Same SQS queue as two targets on same rule | Duplicate messages | One target per queue |
| Skipping DLQ on secondary targets | Silent audit log failures | Add `dead_letter_config` to each target |
| Using fan-out as a replacement for SQS ordering | EventBridge does not guarantee target invocation order | Use SQS FIFO if order matters |

---

## Resource Policy Note for Each Target

Every non-Lambda target added to a rule requires its own resource-based policy.

- **SQS:** `sqs:SendMessage` for `events.amazonaws.com` scoped to rule ARN
- **SNS:** `sns:Publish` for `events.amazonaws.com` scoped to rule ARN
- **CloudWatch Logs:** `logs:CreateLogDelivery` + resource policy on log group

---

## References

- [EventBridge targets reference](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-targets.html)
- [EventBridge rules with multiple targets](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-create-rule.html)
- [S3 EventBridge fan-out blog](https://aws.amazon.com/blogs/aws/new-use-amazon-s3-event-notifications-with-amazon-eventbridge/)
