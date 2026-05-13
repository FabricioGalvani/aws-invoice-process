# CloudWatch Metrics and Embedded Metric Format (EMF)

> **MCP Validated:** 2026-05-12

## Two Categories of Metrics

### Built-in Lambda Metrics (namespace `AWS/Lambda`)

Emitted automatically. No code required.

| Metric | Unit | Alarm Target? | Notes |
|--------|------|---------------|-------|
| `Duration` | ms | Yes (p95) | Billable compute time; alarm at p95 > timeout × 0.8 |
| `Errors` | Count | Yes | Unhandled exceptions + timeouts |
| `Throttles` | Count | Yes | Invocations rejected due to concurrency limit |
| `ConcurrentExecutions` | Count | Optional | Compare to account limit (1000 default) |
| `Invocations` | Count | No | Total calls including errors |
| `IteratorAge` | ms | Yes (SQS) | Age of oldest message; use on SQS-triggered Lambdas |

### Built-in SQS Metrics (namespace `AWS/SQS`)

| Metric | Unit | Alarm Target? |
|--------|------|---------------|
| `ApproximateAgeOfOldestMessage` | s | Yes — alarm if > 300s signals stuck processing |
| `ApproximateNumberOfMessagesVisible` | Count | Yes — DLQ depth alarm on > 0 |
| `NumberOfMessagesSent` | Count | No — throughput monitoring |
| `NumberOfMessagesDeleted` | Count | No — successful consumption |

### Custom Pipeline Metrics (namespace `InvoicePipeline`)

Emitted via EMF from Lambda code using aws-lambda-powertools.

| Metric | Unit | Source Lambda | Meaning |
|--------|------|---------------|---------|
| `InvoicesProcessed` | Count | `warehouse-writer` | Successful end-to-end pipeline completions |
| `ExtractionAccuracy` | Percent | `data-extractor` | Pydantic validation pass rate (target ≥ 90%) |
| `BedrockFallbacks` | Count | `data-extractor` | Haiku → Sonnet fallback events |
| `PydanticValidationFailures` | Count | `data-extractor` | Failed schema validations (numerator for accuracy) |

## Embedded Metric Format (EMF)

EMF is a structured JSON format that Lambda writes to **stdout**. CloudWatch Logs Agent
parses the JSON and publishes metrics asynchronously — **no extra API call, no added
latency, no `PutMetricData` permission required**.

### EMF JSON Structure (raw)

```json
{
  "_aws": {
    "Timestamp": 1715520000000,
    "CloudWatchMetrics": [
      {
        "Namespace": "InvoicePipeline",
        "Dimensions": [["service", "environment", "vendor_type"]],
        "Metrics": [
          {"Name": "InvoicesProcessed", "Unit": "Count"},
          {"Name": "ExtractionAccuracy", "Unit": "Percent"}
        ]
      }
    ]
  },
  "service": "data-extractor",
  "environment": "prod",
  "vendor_type": "ubereats",
  "InvoicesProcessed": 1,
  "ExtractionAccuracy": 95.5
}
```

### EMF via aws-lambda-powertools (preferred)

Powertools `Metrics` class generates valid EMF automatically. See
[../patterns/emf-from-lambda.md](../patterns/emf-from-lambda.md) for the complete
implementation pattern.

```python
from aws_lambda_powertools import Metrics
from aws_lambda_powertools.metrics import MetricUnit

metrics = Metrics(namespace="InvoicePipeline", service="data-extractor")

# Inside handler
metrics.add_metric(name="InvoicesProcessed", unit=MetricUnit.Count, value=1)
metrics.add_metric(name="ExtractionAccuracy", unit=MetricUnit.Percent, value=95.5)
metrics.add_dimension(name="vendor_type", value="ubereats")
```

## EMF Limits

| Limit | Value |
|-------|-------|
| Max metrics per EMF blob | 100 |
| Max dimensions per metric | 30 |
| Max dimension name/value length | 250 characters |
| Max metric name length | 1024 characters |

Powertools auto-flushes at 100 metrics and starts a new blob.

## Why EMF Over PutMetricData

| Approach | Latency | Cost | IAM Required |
|----------|---------|------|--------------|
| EMF (via stdout) | Zero added latency | No extra charge | No extra permission |
| `PutMetricData` API | +50-200ms per call | $0.01/1000 API calls | `cloudwatch:PutMetricData` |

**Always use EMF for custom metrics in Lambda. Never call PutMetricData from hot paths.**

## Dimensions Strategy

Use dimensions that have low cardinality. High-cardinality dimensions (e.g., `invoice_id`)
create too many metric time series and explode cost.

Good dimensions for this pipeline: `service`, `environment`, `vendor_type`
Bad dimensions (avoid): `invoice_id`, `request_id`, `timestamp`

## See Also

- [../patterns/emf-from-lambda.md](../patterns/emf-from-lambda.md)
- [alarms-and-sns.md](alarms-and-sns.md) — alarm on these metrics
- [../patterns/slo-alarms.md](../patterns/slo-alarms.md)
