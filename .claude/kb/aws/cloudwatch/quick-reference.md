# CloudWatch Quick Reference — Invoice Pipeline

> **MCP Validated:** 2026-05-12

## Retention (Terraform)

```hcl
resource "aws_cloudwatch_log_group" "lambda" {
  name              = "/aws/lambda/${var.function_name}-${var.environment}"
  retention_in_days = var.environment == "prod" ? 90 : 30
}
```

Default is Never Expire = cost trap. Always set retention.

---

## EMF Custom Metrics (Powertools)

```python
metrics = Metrics(namespace="InvoicePipeline", service="data-extractor")
metrics.add_dimension(name="vendor_type", value=vendor_type)
metrics.add_metric(name="InvoicesProcessed", unit=MetricUnit.Count, value=1)
metrics.add_metric(name="ExtractionAccuracy", unit=MetricUnit.Percent, value=95.5)
# Flushed automatically at handler return via @metrics.log_metrics
```

Custom metrics namespace `InvoicePipeline`: `InvoicesProcessed`, `ExtractionAccuracy`,
`BedrockFallbacks`, `PydanticValidationFailures`.

---

## Key Built-in Metrics

| Namespace | Metric | Alarm target |
|-----------|--------|--------------|
| `AWS/Lambda` | `Errors` | > 0 per 5 min |
| `AWS/Lambda` | `Duration` (p95) | > timeout × 0.8 ms |
| `AWS/Lambda` | `Throttles` | > 0 |
| `AWS/SQS` | `ApproximateNumberOfMessagesVisible` | DLQ > 0 |
| `AWS/SQS` | `ApproximateAgeOfOldestMessage` | > 300s |

---

## Alarm Skeleton (Terraform)

```hcl
resource "aws_cloudwatch_metric_alarm" "example" {
  alarm_name          = "invoice-{name}-${var.environment}"
  metric_name         = "Errors"
  namespace           = "AWS/Lambda"
  statistic           = "Sum"          # or extended_statistic = "p95"
  period              = 60
  evaluation_periods  = 2
  datapoints_to_alarm = 2
  threshold           = 0
  comparison_operator = "GreaterThanThreshold"
  treat_missing_data  = "notBreaching"
  alarm_actions       = [var.alert_sns_arn]
  ok_actions          = [var.alert_sns_arn]
  dimensions          = { FunctionName = "data-extractor-${var.environment}" }
}
```

---

## Log Insights — Useful Queries

```sql
-- Errors in last hour
fields @timestamp, service, message | filter level = "ERROR"
| sort @timestamp desc | limit 50

-- Pydantic failures by vendor
fields vendor_type | filter message like /Pydantic validation failed/
| stats count(*) as failures by vendor_type

-- p95 latency by function
fields @duration | stats pct(@duration, 95) as p95 by service

-- Cold starts
fields function_name | filter cold_start = true
| stats avg(@duration) as avg_ms by function_name
```

---

## Anti-Patterns

| Wrong | Right |
|-------|-------|
| No `retention_in_days` | Set 30d dev / 90d prod |
| `PutMetricData` from Lambda | Use EMF (Powertools) |
| Alarm without `alarm_actions` | Wire SNS ARN |
| No DLQ alarm | `ApproximateNumberOfMessagesVisible > 0` |
| Dashboard only in console | JSON in Terraform |
| High-cardinality EMF dimension | Use `vendor_type`, `environment` only |

See [index.md](index.md) for full navigation and file map.
