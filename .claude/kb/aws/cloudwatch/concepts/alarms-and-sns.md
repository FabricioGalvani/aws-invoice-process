# CloudWatch Alarms and SNS Routing

> **MCP Validated:** 2026-05-12

## Alarm Anatomy

A CloudWatch Alarm monitors a single metric time series and transitions between three
states: `OK`, `ALARM`, `INSUFFICIENT_DATA`.

### Core Configuration Fields

| Field | Purpose | Example |
|-------|---------|---------|
| `metric_name` | Which metric to watch | `Errors` |
| `namespace` | Metric namespace | `AWS/Lambda` |
| `statistic` / `extended_statistic` | Aggregation function | `Sum`, `p95` |
| `period` | Evaluation window in seconds | `60` |
| `evaluation_periods` | Number of periods to evaluate | `2` |
| `datapoints_to_alarm` | Breaching datapoints to trigger | `2` |
| `threshold` | Value triggering ALARM state | `1` |
| `comparison_operator` | How to compare metric to threshold | `GreaterThanThreshold` |
| `alarm_actions` | SNS ARNs to notify | `[aws_sns_topic.alerts.arn]` |
| `ok_actions` | SNS ARNs on recovery | `[aws_sns_topic.alerts.arn]` |
| `treat_missing_data` | Behavior on no data | `notBreaching` or `breaching` |

### Terraform Skeleton

```hcl
resource "aws_cloudwatch_metric_alarm" "example" {
  alarm_name          = "invoice-pipeline-${var.alarm_name}-${var.environment}"
  metric_name         = var.metric_name
  namespace           = var.namespace
  statistic           = var.statistic
  period              = 60
  evaluation_periods  = 2
  datapoints_to_alarm = 2
  threshold           = var.threshold
  comparison_operator = "GreaterThanThreshold"
  alarm_actions       = [aws_sns_topic.pipeline_alerts.arn]
  ok_actions          = [aws_sns_topic.pipeline_alerts.arn]
  treat_missing_data  = "notBreaching"

  dimensions = var.dimensions
}
```

## SLO-Driven Alarms for This Pipeline

The project SLO is **≥ 90% extraction accuracy**. This means extraction failure rate
must stay **≤ 10%**. Every alarm below connects to a project SLO or reliability target.

### Key Alarm Targets

| Alarm | Metric | Namespace | Threshold | Rationale |
|-------|--------|-----------|-----------|-----------|
| Extraction failure rate | `PydanticValidationFailures` | `InvoicePipeline` | > 10% of `InvoicesProcessed` | SLO gate: 90% accuracy |
| DLQ depth | `ApproximateNumberOfMessagesVisible` | `AWS/SQS` | > 0 | Any DLQ message = failure requiring human review |
| Lambda duration p95 | `Duration` | `AWS/Lambda` | > (timeout × 0.8) ms | Early warning before timeout hits |
| Lambda errors | `Errors` | `AWS/Lambda` | > 0 per 5 min | Unhandled exceptions |
| Bedrock fallback rate | `BedrockFallbacks` | `InvoicePipeline` | > 50 per hour | Haiku degradation warning |
| SQS message age | `ApproximateAgeOfOldestMessage` | `AWS/SQS` | > 300s | Pipeline stuck / throughput problem |

Full Terraform for each alarm: see [../patterns/slo-alarms.md](../patterns/slo-alarms.md)
and [../patterns/dlq-depth-alarm.md](../patterns/dlq-depth-alarm.md).

## SNS Topic → Slack Webhook

Alarm actions point to an SNS topic. A Lambda subscriber forwards notifications to Slack.

```hcl
resource "aws_sns_topic" "pipeline_alerts" {
  name = "invoice-pipeline-alerts-${var.environment}"
}

resource "aws_sns_topic_subscription" "slack_webhook" {
  topic_arn = aws_sns_topic.pipeline_alerts.arn
  protocol  = "lambda"
  endpoint  = aws_lambda_function.slack_notifier.arn
}
```

The Slack notifier Lambda receives the SNS JSON payload, formats a Slack Block Kit
message, and POSTs to the Slack webhook URL stored in AWS Secrets Manager.

## Composite Alarms

For a single "pipeline health" alarm aggregating multiple sub-alarms, use
`aws_cloudwatch_composite_alarm`:

```hcl
resource "aws_cloudwatch_composite_alarm" "pipeline_health" {
  alarm_name = "invoice-pipeline-health-${var.environment}"

  alarm_rule = join(" OR ", [
    "ALARM(${aws_cloudwatch_metric_alarm.dlq_depth.alarm_name})",
    "ALARM(${aws_cloudwatch_metric_alarm.extraction_errors.alarm_name})",
    "ALARM(${aws_cloudwatch_metric_alarm.lambda_errors.alarm_name})",
  ])

  alarm_actions = [aws_sns_topic.pipeline_alerts.arn]
}
```

## treat_missing_data Guidance

| Value | Use When |
|-------|----------|
| `notBreaching` | Metric absent = healthy (most Lambda alarms) |
| `breaching` | Absent data = alarm (DLQ depth if no data means stream broken) |
| `ignore` | Keep current state (avoid for critical alarms) |
| `missing` | CloudWatch default — alarm goes INSUFFICIENT_DATA |

## Anti-Patterns

| Anti-Pattern | Impact | Fix |
|---|---|---|
| Alarms without `alarm_actions` | Silent failures | Always wire SNS ARN |
| No DLQ alarm | Messages silently dead | Add `ApproximateNumberOfMessagesVisible > 0` |
| `evaluation_periods = 1` on noisy metrics | False positives | Use 2-3 periods |
| No `ok_actions` | No recovery notification | Add same SNS ARN |
| Alarms on average Duration only | Misses tail latency | Use p95 extended_statistic |

## See Also

- [metrics-and-emf.md](metrics-and-emf.md) — custom metrics to alarm on
- [../patterns/slo-alarms.md](../patterns/slo-alarms.md) — full Terraform alarm set
- [../patterns/dlq-depth-alarm.md](../patterns/dlq-depth-alarm.md)
