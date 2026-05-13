# Pattern: SLO-Driven CloudWatch Alarms (Invoice Pipeline)

> **MCP Validated:** 2026-05-12

## Context

Project SLO: **≥ 90% extraction accuracy** across 2,000-3,500 invoices/month.
All alarms below are directly traceable to this SLO or to system reliability targets.
All alarm actions route to `aws_sns_topic.pipeline_alerts` → Slack.

See [../concepts/alarms-and-sns.md](../concepts/alarms-and-sns.md) for alarm anatomy.

Required variables: `var.environment`, `var.alert_sns_arn` (SNS topic ARN → Slack).
Lambda timeout values in ms: tiff-converter 300000, classifier 60000,
data-extractor 120000, warehouse-writer 60000.

## Alarm 1: Extraction Failure Rate > 10% (SLO Gate)

Uses a metric math alarm comparing `PydanticValidationFailures` to `InvoicesProcessed`.

```hcl
resource "aws_cloudwatch_metric_alarm" "extraction_failure_rate" {
  alarm_name          = "invoice-extraction-failure-rate-${var.environment}"
  alarm_description   = "SLO BREACH: Pydantic validation failures exceed 10% of invoices processed"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  datapoints_to_alarm = 2
  threshold           = 10
  treat_missing_data  = "notBreaching"

  alarm_actions = [var.alert_sns_arn]
  ok_actions    = [var.alert_sns_arn]

  metric_query {
    id          = "failure_rate"
    expression  = "(failures / total) * 100"
    label       = "Extraction Failure Rate (%)"
    return_data = true
  }

  metric_query {
    id = "failures"
    metric {
      metric_name = "PydanticValidationFailures"
      namespace   = "InvoicePipeline"
      period      = 300
      stat        = "Sum"
      dimensions  = { environment = var.environment }
    }
  }

  metric_query {
    id = "total"
    metric {
      metric_name = "InvoicesProcessed"
      namespace   = "InvoicePipeline"
      period      = 300
      stat        = "Sum"
      dimensions  = { environment = var.environment }
    }
  }
}
```

## Alarm 2: Lambda Duration p95 > 80% of Timeout

One alarm per Lambda function. Example for `data-extractor` (120s timeout → 96,000ms).

```hcl
resource "aws_cloudwatch_metric_alarm" "data_extractor_duration" {
  alarm_name          = "data-extractor-p95-duration-${var.environment}"
  alarm_description   = "p95 duration approaching 120s timeout (threshold: 96s)"
  metric_name         = "Duration"
  namespace           = "AWS/Lambda"
  extended_statistic  = "p95"
  period              = 60
  evaluation_periods  = 3
  datapoints_to_alarm = 2
  threshold           = 96000  # 80% of 120s timeout in ms
  comparison_operator = "GreaterThanThreshold"
  treat_missing_data  = "notBreaching"
  alarm_actions       = [var.alert_sns_arn]
  ok_actions          = [var.alert_sns_arn]

  dimensions = {
    FunctionName = "data-extractor-${var.environment}"
  }
}
```

Repeat for each Lambda with its `threshold = timeout_ms * 0.8`.

## Alarm 3: Lambda Errors > 0 (5-minute window)

```hcl
resource "aws_cloudwatch_metric_alarm" "lambda_errors" {
  for_each = toset([
    "tiff-to-png-converter",
    "invoice-classifier",
    "data-extractor",
    "warehouse-writer",
  ])

  alarm_name          = "${each.key}-errors-${var.environment}"
  alarm_description   = "Unhandled errors in ${each.key}"
  metric_name         = "Errors"
  namespace           = "AWS/Lambda"
  statistic           = "Sum"
  period              = 300
  evaluation_periods  = 1
  datapoints_to_alarm = 1
  threshold           = 0
  comparison_operator = "GreaterThanThreshold"
  treat_missing_data  = "notBreaching"
  alarm_actions       = [var.alert_sns_arn]
  ok_actions          = [var.alert_sns_arn]

  dimensions = {
    FunctionName = "${each.key}-${var.environment}"
  }
}
```

## Alarm 4: Bedrock Fallback Rate (Haiku → Sonnet)

```hcl
resource "aws_cloudwatch_metric_alarm" "bedrock_fallbacks" {
  alarm_name          = "bedrock-fallbacks-${var.environment}"
  alarm_description   = "Haiku-to-Sonnet fallbacks exceeding 50/hour — check Bedrock capacity"
  metric_name         = "BedrockFallbacks"
  namespace           = "InvoicePipeline"
  statistic           = "Sum"
  period              = 3600
  evaluation_periods  = 1
  datapoints_to_alarm = 1
  threshold           = 50
  comparison_operator = "GreaterThanThreshold"
  treat_missing_data  = "notBreaching"
  alarm_actions       = [var.alert_sns_arn]

  dimensions = {
    environment = var.environment
  }
}
```

## Alarm 5: SQS Message Age > 5 Minutes (Stuck Pipeline)

```hcl
resource "aws_cloudwatch_metric_alarm" "sqs_message_age" {
  for_each = toset([
    "invoice-uploaded",
    "invoice-converted",
    "invoice-classified",
    "invoice-extracted",
  ])

  alarm_name          = "sqs-${each.key}-message-age-${var.environment}"
  alarm_description   = "Messages in ${each.key} queue are older than 5 min — pipeline stuck"
  metric_name         = "ApproximateAgeOfOldestMessage"
  namespace           = "AWS/SQS"
  statistic           = "Maximum"
  period              = 60
  evaluation_periods  = 3
  datapoints_to_alarm = 3
  threshold           = 300  # 5 minutes in seconds
  comparison_operator = "GreaterThanThreshold"
  treat_missing_data  = "notBreaching"
  alarm_actions       = [var.alert_sns_arn]
  ok_actions          = [var.alert_sns_arn]

  dimensions = {
    QueueName = "${each.key}-${var.environment}"
  }
}
```

## Composite Pipeline Health Alarm

```hcl
resource "aws_cloudwatch_composite_alarm" "pipeline_health" {
  alarm_name  = "invoice-pipeline-health-${var.environment}"
  alarm_rule  = join(" OR ", [
    "ALARM(\"invoice-extraction-failure-rate-${var.environment}\")",
    "ALARM(\"data-extractor-errors-${var.environment}\")",
    "ALARM(\"warehouse-writer-errors-${var.environment}\")",
  ])
  alarm_actions = [var.alert_sns_arn]
}
```

## See Also

- [dlq-depth-alarm.md](dlq-depth-alarm.md)
- [../concepts/alarms-and-sns.md](../concepts/alarms-and-sns.md)
- [../concepts/metrics-and-emf.md](../concepts/metrics-and-emf.md)
