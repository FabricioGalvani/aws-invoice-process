# Pattern: DLQ Depth Alarm (SQS Dead-Letter Queue)

> **MCP Validated:** 2026-05-12

## Why This Matters

Every SQS queue in the invoice pipeline has a paired DLQ (Dead-Letter Queue). Messages
land in a DLQ after exhausting `maxReceiveCount` retries. A DLQ depth > 0 means at
least one invoice failed permanently — requiring human review or automated remediation.

**A DLQ with no alarm is a silent failure. Always alarm on DLQ depth > 0.**

## DLQ Architecture for This Pipeline

| Primary Queue | DLQ Name | maxReceiveCount |
|---|---|---|
| `invoice-uploaded-{env}` | `invoice-uploaded-dlq-{env}` | 3 |
| `invoice-converted-{env}` | `invoice-converted-dlq-{env}` | 3 |
| `invoice-classified-{env}` | `invoice-classified-dlq-{env}` | 3 |
| `invoice-extracted-{env}` | `invoice-extracted-dlq-{env}` | 3 |

## Terraform: DLQ + Alarm (per-queue module pattern)

```hcl
# modules/sqs/main.tf

resource "aws_sqs_queue" "dlq" {
  name                      = "${var.queue_name}-dlq-${var.environment}"
  message_retention_seconds = 1209600  # 14 days
  kms_master_key_id         = "alias/aws/sqs"

  tags = {
    Environment = var.environment
    Service     = "invoice-pipeline"
  }
}

resource "aws_sqs_queue" "main" {
  name                       = "${var.queue_name}-${var.environment}"
  visibility_timeout_seconds = var.visibility_timeout
  message_retention_seconds  = 345600  # 4 days

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.dlq.arn
    maxReceiveCount     = 3
  })

  kms_master_key_id = "alias/aws/sqs"

  tags = {
    Environment = var.environment
    Service     = "invoice-pipeline"
  }
}

resource "aws_cloudwatch_metric_alarm" "dlq_depth" {
  alarm_name          = "${var.queue_name}-dlq-depth-${var.environment}"
  alarm_description   = "DLQ ${var.queue_name}-dlq-${var.environment} has messages — invoice(s) failed permanently"
  metric_name         = "ApproximateNumberOfMessagesVisible"
  namespace           = "AWS/SQS"
  statistic           = "Sum"
  period              = 60
  evaluation_periods  = 1
  datapoints_to_alarm = 1
  threshold           = 0
  comparison_operator = "GreaterThanThreshold"
  treat_missing_data  = "notBreaching"
  alarm_actions       = [var.alert_sns_arn]
  ok_actions          = [var.alert_sns_arn]

  dimensions = {
    QueueName = aws_sqs_queue.dlq.name
  }
}
```

## Module Instantiation (environments/dev/main.tf)

```hcl
module "sqs_invoice_uploaded" {
  source             = "../../modules/sqs"
  queue_name         = "invoice-uploaded"
  environment        = var.environment
  visibility_timeout = 310  # 5s > Lambda tiff-converter timeout (300s)
  alert_sns_arn      = module.sns.pipeline_alerts_arn
}

module "sqs_invoice_converted" {
  source             = "../../modules/sqs"
  queue_name         = "invoice-converted"
  environment        = var.environment
  visibility_timeout = 65   # 5s > classifier timeout (60s)
  alert_sns_arn      = module.sns.pipeline_alerts_arn
}

module "sqs_invoice_classified" {
  source             = "../../modules/sqs"
  queue_name         = "invoice-classified"
  environment        = var.environment
  visibility_timeout = 125  # 5s > extractor timeout (120s)
  alert_sns_arn      = module.sns.pipeline_alerts_arn
}

module "sqs_invoice_extracted" {
  source             = "../../modules/sqs"
  queue_name         = "invoice-extracted"
  environment        = var.environment
  visibility_timeout = 65   # 5s > warehouse-writer timeout (60s)
  alert_sns_arn      = module.sns.pipeline_alerts_arn
}
```

## Visibility Timeout Rule

`visibility_timeout` MUST be at least 6× the Lambda function timeout to prevent
duplicate processing during retries. Minimum recommended: Lambda timeout + 5s.

## DLQ Redrive (Manual Recovery)

When a DLQ alarm fires, the Triage CrewAI agent reads the DLQ messages from S3 logs.
For manual redrive, use the AWS Console or CLI:

```bash
aws sqs start-message-move-task \
  --source-arn arn:aws:sqs:us-east-1:ACCOUNT:invoice-uploaded-dlq-prod \
  --destination-arn arn:aws:sqs:us-east-1:ACCOUNT:invoice-uploaded-prod \
  --max-number-of-messages-per-second 1
```

## Anti-Patterns

| Anti-Pattern | Impact | Fix |
|---|---|---|
| No alarm on DLQ | Silent failures | Add `ApproximateNumberOfMessagesVisible > 0` |
| `maxReceiveCount = 1` | No retry, straight to DLQ | Use 3 for transient error tolerance |
| Short `message_retention_seconds` on DLQ | Lost messages before review | Use 14 days (1209600) |
| Shared DLQ across queues | Unclear failure attribution | One DLQ per primary queue |
| `visibility_timeout` < Lambda timeout | Duplicate processing on retry | Set to Lambda timeout + 5s |

## See Also

- [../concepts/alarms-and-sns.md](../concepts/alarms-and-sns.md) — alarm anatomy
- [slo-alarms.md](slo-alarms.md) — full alarm set for the pipeline
- [../concepts/logs-and-retention.md](../concepts/logs-and-retention.md)
