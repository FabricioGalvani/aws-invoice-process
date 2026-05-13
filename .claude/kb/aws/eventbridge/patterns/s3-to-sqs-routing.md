> **MCP Validated:** 2026-05-12

# Pattern: S3 Object Created → EventBridge Rule → SQS

## Pattern Overview

Route S3 `Object Created` events through the default EventBridge event bus to an
SQS queue, using an input transformer to slim the payload. This is the entry
trigger for the invoice processing pipeline.

**Flow:**

```
invoices-input-{env} bucket
    │ (S3 EventBridge notification — single setting)
    ▼
Default Event Bus (aws.s3 events land here automatically)
    │ Rule: pattern matches source=aws.s3, detail-type=Object Created, suffix=.tiff
    ▼
SQS: invoice-uploaded-{env}          ← Lambda ESM trigger
    │
    ▼ (on delivery failure, after 185 retries)
SQS: eventbridge-dlq-{env}           ← Rule-target DLQ

Lambda: tiff-to-png-converter
    │ (on processing failure, after maxReceiveCount)
    ▼
SQS: invoice-uploaded-dlq-{env}      ← SQS consumer DLQ
```

---

## Step 1 — Enable S3 EventBridge Notifications

This is a **single bucket-level setting** (not per-event-type). Once enabled,
ALL object events for the bucket are published to the default event bus.
Filtering happens on the EventBridge rule, not on S3.

```hcl
resource "aws_s3_bucket_notification" "invoices_input" {
  bucket      = aws_s3_bucket.invoices_input.id
  eventbridge = true   # ← the only required field
}
```

> This replaces the legacy `aws_s3_bucket_notification` with `lambda_function` or
> `queue` blocks. Do not mix EventBridge and direct S3→SQS notifications on the
> same bucket.

---

## Step 2 — EventBridge Rule with Pattern

```hcl
resource "aws_cloudwatch_event_rule" "invoice_uploaded" {
  name        = "invoice-uploaded-${var.env}"
  description = "Routes TIFF uploads from invoices-input to processing SQS queue"
  event_bus_name = "default"  # S3 events always go to default bus

  event_pattern = jsonencode({
    source      = ["aws.s3"]
    detail-type = ["Object Created"]
    detail = {
      bucket = { name = [{ prefix = "invoices-input-" }] }
      object = { key  = [{ suffix = ".tiff" }] }
    }
  })
}
```

---

## Step 3 — SQS Target with Input Transformer and DLQ

```hcl
resource "aws_cloudwatch_event_target" "invoice_sqs" {
  rule      = aws_cloudwatch_event_rule.invoice_uploaded.name
  target_id = "InvoiceUploadedQueue"
  arn       = aws_sqs_queue.invoice_uploaded.arn

  input_transformer {
    input_paths = {
      bucket  = "$.detail.bucket.name"
      key     = "$.detail.object.key"
      size    = "$.detail.object.size"
      eventId = "$.id"
      time    = "$.time"
    }
    input_template = <<-JSON
      {
        "bucket":  "<bucket>",
        "key":     "<key>",
        "size":    <size>,
        "eventId": "<eventId>",
        "time":    "<time>",
        "source":  "eventbridge-s3"
      }
    JSON
  }

  dead_letter_config {
    arn = aws_sqs_queue.eventbridge_dlq.arn
  }

  retry_policy {
    maximum_retry_attempts       = 185
    maximum_event_age_in_seconds = 86400
  }
}
```

---

## Step 4 — SQS Queue Resource Policy

EventBridge uses the **SQS resource-based policy** (not an IAM role) to send messages.

```hcl
data "aws_iam_policy_document" "invoice_uploaded_queue_policy" {
  statement {
    sid    = "AllowEventBridgeSendMessage"
    effect = "Allow"
    principals {
      type        = "Service"
      identifiers = ["events.amazonaws.com"]
    }
    actions   = ["sqs:SendMessage"]
    resources = [aws_sqs_queue.invoice_uploaded.arn]
    condition {
      test     = "ArnEquals"
      variable = "aws:SourceArn"
      values   = [aws_cloudwatch_event_rule.invoice_uploaded.arn]
    }
  }
}

resource "aws_sqs_queue_policy" "invoice_uploaded" {
  queue_url = aws_sqs_queue.invoice_uploaded.id
  policy    = data.aws_iam_policy_document.invoice_uploaded_queue_policy.json
}
```

---

## Step 5 — SQS Queue with Its Own DLQ

```hcl
resource "aws_sqs_queue" "invoice_uploaded_dlq" {
  name                      = "invoice-uploaded-dlq-${var.env}"
  message_retention_seconds = 1209600  # 14 days
}

resource "aws_sqs_queue" "invoice_uploaded" {
  name                       = "invoice-uploaded-${var.env}"
  visibility_timeout_seconds = 360     # >= Lambda timeout (5 min)
  message_retention_seconds  = 86400   # 1 day

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.invoice_uploaded_dlq.arn
    maxReceiveCount     = 3
  })
}
```

---

## Verification Checklist

- [ ] `aws_s3_bucket_notification.eventbridge = true` on `invoices-input-{env}`
- [ ] Rule `event_bus_name = "default"` (S3 events only go here)
- [ ] Pattern scoped to bucket prefix AND `.tiff` suffix
- [ ] Input transformer reduces payload to required fields only
- [ ] Rule-target `dead_letter_config` points to `eventbridge-dlq`
- [ ] SQS queue policy grants `sqs:SendMessage` to `events.amazonaws.com`
- [ ] SQS queue `redrive_policy` points to its own consumer DLQ
- [ ] `visibility_timeout_seconds` >= Lambda timeout

---

## References

- [S3 EventBridge notifications](https://docs.aws.amazon.com/AmazonS3/latest/userguide/EventBridge.html)
- [Create EventBridge rule for S3](https://repost.aws/knowledge-center/eventbridge-rule-monitors-s3)
- [EventBridge rule DLQ](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-rule-dlq.html)
