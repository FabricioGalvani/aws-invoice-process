# Pattern: CloudWatch Dashboard for the 4-Lambda Pipeline

> **MCP Validated:** 2026-05-12

## What This Dashboard Shows

One dashboard per environment visualizing the full invoice pipeline health:
- 4 Lambda functions: invocations, errors, duration p95
- 4 SQS queues: visible message count, oldest message age
- 4 DLQs: depth (should always be 0)
- Custom pipeline metrics: InvoicesProcessed, ExtractionAccuracy, BedrockFallbacks

## Terraform Resource

```hcl
resource "aws_cloudwatch_dashboard" "invoice_pipeline" {
  dashboard_name = "invoice-pipeline-${var.environment}"
  dashboard_body = templatefile(
    "${path.module}/dashboard.json.tpl",
    {
      environment = var.environment
      region      = var.aws_region
    }
  )
}
```

The dashboard body is defined in `specs/pipeline-dashboard.json` (example static form)
or via a `templatefile` with environment substitution.

## Dashboard Layout (7 rows)

| Row | Widget | Metric(s) |
|-----|--------|-----------|
| 1 | Pipeline SLO: Extraction Accuracy | `ExtractionAccuracy` (line, InvoicePipeline ns) |
| 1 | Invoices Processed (24h) | `InvoicesProcessed` (number, sum) |
| 2 | Lambda Invocations | All 4 functions, `Invocations` (line) |
| 2 | Lambda Errors | All 4 functions, `Errors` (line) |
| 3 | Lambda Duration p95 | All 4 functions, `Duration` p95 (line) |
| 3 | Bedrock Fallbacks | `BedrockFallbacks` (bar, InvoicePipeline ns) |
| 4 | SQS Queue Depths | `ApproximateNumberOfMessagesVisible` all queues |
| 4 | SQS Message Age | `ApproximateAgeOfOldestMessage` all queues |
| 5 | DLQ Depths (must be 0) | All 4 DLQs `ApproximateNumberOfMessagesVisible` |
| 6 | Pydantic Validation Failures | `PydanticValidationFailures` (bar) |
| 7 | Alarm Status Panel | All pipeline alarms health at a glance |

## Minimal JSON Widget Structure (reference)

```json
{
  "type": "metric",
  "properties": {
    "title": "Lambda Duration p95",
    "view": "timeSeries",
    "stat": "p95",
    "period": 60,
    "metrics": [
      ["AWS/Lambda", "Duration", "FunctionName", "tiff-to-png-converter-prod"],
      ["AWS/Lambda", "Duration", "FunctionName", "invoice-classifier-prod"],
      ["AWS/Lambda", "Duration", "FunctionName", "data-extractor-prod"],
      ["AWS/Lambda", "Duration", "FunctionName", "warehouse-writer-prod"]
    ],
    "yAxis": { "left": { "label": "ms" } }
  }
}
```

Full dashboard JSON: see [../specs/pipeline-dashboard.json](../specs/pipeline-dashboard.json).

## Exporting Dashboard JSON from Console

1. Build the dashboard visually in CloudWatch Console.
2. Choose Actions → View/edit source.
3. Copy JSON → paste into `specs/pipeline-dashboard.json`.
4. Convert placeholders to `templatefile` variables for environment-specific names.

## Alarm Status Widget

Renders all alarms as a colored grid (green/red/grey). Add as the last widget:

```json
{
  "type": "alarm",
  "properties": {
    "title": "Pipeline Alarm Status",
    "alarms": [
      "arn:aws:cloudwatch:${region}:${account_id}:alarm:invoice-extraction-failure-rate-prod",
      "arn:aws:cloudwatch:${region}:${account_id}:alarm:invoice-uploaded-dlq-depth-prod"
    ]
  }
}
```

## Cost Note

Dashboards are **$3/month per dashboard** (first 3 are free). For dev, consider
reducing to a single shared dashboard or using the free tier.

## Anti-Patterns

| Anti-Pattern | Impact | Fix |
|---|---|---|
| Dashboard defined in console only | Destroyed on account recreation | Store JSON in Terraform |
| No DLQ widgets | Failures invisible | Always show all 4 DLQ depths |
| Using average Duration | Hides tail latency | Always use p95 for Duration |
| Hardcoded account IDs in JSON | Non-portable | Use templatefile with variables |

## See Also

- [../specs/pipeline-dashboard.json](../specs/pipeline-dashboard.json) — full body
- [../concepts/metrics-and-emf.md](../concepts/metrics-and-emf.md) — custom metrics
- [../concepts/alarms-and-sns.md](../concepts/alarms-and-sns.md) — alarm status widget
