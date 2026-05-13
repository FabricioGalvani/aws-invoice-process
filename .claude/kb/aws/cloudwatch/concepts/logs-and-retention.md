# CloudWatch Logs and Retention

> **MCP Validated:** 2026-05-12

## What It Is

Amazon CloudWatch Logs stores, monitors, and queries log output from AWS services. For
Lambda, each function gets its own **Log Group** (`/aws/lambda/{function-name}`), created
automatically on first invocation. Log Groups contain **Log Streams** — one per
Lambda execution environment (container).

## Default Retention — the Cost Trap

By default, CloudWatch Logs retention is `0` (Never Expire). At ~$0.03/GB/month, an
unmanaged production system accumulates unbounded cost. **Always set retention in
Terraform.** Omitting `retention_in_days` is an anti-pattern.

### Recommended Retention for This Pipeline

| Environment | Retention | Rationale |
|-------------|-----------|-----------|
| dev | 30 days | Debug cycle length; cost-optimized |
| prod | 90 days | Compliance window; Firehose handles long-term |

### Valid `retention_in_days` Values (Terraform)

1, 3, 5, 7, 14, **30**, 60, **90**, 120, 150, 180, 365, 400, 545, 731, 1096, 1827,
2192, 2557, 2922, 3288, 3653, and 0 (never expire — avoid).

### Terraform Resource

```hcl
resource "aws_cloudwatch_log_group" "lambda_tiff_converter" {
  name              = "/aws/lambda/tiff-to-png-converter-${var.environment}"
  retention_in_days = var.environment == "prod" ? 90 : 30

  tags = {
    Environment = var.environment
    Service     = "invoice-pipeline"
  }
}
```

Create one `aws_cloudwatch_log_group` per Lambda. Without it, Lambda auto-creates
the group with `Never Expire`.

## Log Insights — Query Syntax

CloudWatch Logs Insights uses its own SQL-like language against structured JSON logs
(as emitted by aws-lambda-powertools).

### Core Syntax

```sql
fields @timestamp, @message, @logStream
| filter level = "ERROR"
| sort @timestamp desc
| limit 100
```

Key commands: `fields`, `filter`, `stats`, `sort`, `limit`, `parse`, `display`.

### Pipeline-Specific Queries

**Error rate by Lambda function (last 1 hour):**
```sql
fields @timestamp, service, level, message
| filter level = "ERROR" or level = "CRITICAL"
| stats count(*) as error_count by service
| sort error_count desc
```

**Extraction accuracy failures:**
```sql
fields @timestamp, service, message, invoice_id, vendor_type
| filter message like /Pydantic validation failed/
| stats count(*) as failures by vendor_type
| sort failures desc
```

**P95 processing latency by invoice vendor:**
```sql
fields @timestamp, @duration, vendor_type
| filter service = "data-extractor"
| stats pct(@duration, 95) as p95_ms by vendor_type
```

**Cold start detection:**
```sql
fields @timestamp, function_name, cold_start, @duration
| filter cold_start = true
| stats avg(@duration) as avg_cold_start_ms by function_name
```

**Bedrock fallback rate (Haiku → Sonnet):**
```sql
fields @timestamp, service, message, model_id
| filter message like /Bedrock fallback/
| stats count(*) as fallbacks by bin(1h)
```

## Structured Logging Requirement

All Lambda functions in this pipeline MUST use `aws-lambda-powertools` Logger for
structured JSON output. This enables Log Insights to parse fields directly without
`parse` regex extraction.

See: `.claude/kb/aws/lambda/patterns/powertools-logging.md`

## Anti-Patterns

| Anti-Pattern | Impact | Fix |
|---|---|---|
| No `retention_in_days` in Terraform | Unbounded cost | Always set 30/90 |
| `retention_in_days = 0` explicitly | Same as never-expire | Use 30 or 90 |
| Unstructured string logs | Log Insights queries fail | Use Powertools Logger |
| Manual log group creation | Drift from IaC | Define in Terraform |

## See Also

- [metrics-and-emf.md](metrics-and-emf.md) — custom metrics from Lambda
- [firehose-export.md](firehose-export.md) — long-term S3 export for CrewAI
- [../patterns/logs-to-s3-firehose.md](../patterns/logs-to-s3-firehose.md)
