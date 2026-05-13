# Pattern: Emit Custom Metrics via EMF from Lambda (Powertools)

> **MCP Validated:** 2026-05-12

## When to Use

- Emitting pipeline-specific metrics (`InvoicesProcessed`, `ExtractionAccuracy`,
  `BedrockFallbacks`, `PydanticValidationFailures`) from any Lambda in this pipeline
- Zero-latency metric emission — no extra API call, no `cloudwatch:PutMetricData` needed
- See [../concepts/metrics-and-emf.md](../concepts/metrics-and-emf.md) for theory

## Prerequisites

`aws-lambda-powertools` installed (already required for structured logging).

```
aws-lambda-powertools[all]>=2.43.0
```

Environment variables (set via Terraform/SAM):

```
POWERTOOLS_SERVICE_NAME=data-extractor
POWERTOOLS_METRICS_NAMESPACE=InvoicePipeline
```

## Implementation — data-extractor Lambda

```python
"""Data extractor Lambda: emits EMF metrics via Powertools.

Metrics emitted here feed the SLO alarms defined in slo-alarms.md.
Namespace: InvoicePipeline
"""
import os
from aws_lambda_powertools import Logger, Metrics, Tracer
from aws_lambda_powertools.metrics import MetricUnit
from aws_lambda_powertools.utilities.typing import LambdaContext

logger = Logger(service=os.environ["POWERTOOLS_SERVICE_NAME"])
metrics = Metrics(
    namespace=os.environ.get("POWERTOOLS_METRICS_NAMESPACE", "InvoicePipeline"),
    service=os.environ["POWERTOOLS_SERVICE_NAME"],
)
tracer = Tracer()


@logger.inject_lambda_context
@tracer.capture_lambda_handler
@metrics.log_metrics(capture_cold_start_metric=True, raise_on_empty_metrics=False)
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    vendor_type = event.get("vendor_type", "unknown")

    # Dimensions must be set before metrics are added for that invocation
    metrics.add_dimension(name="vendor_type", value=vendor_type)
    metrics.add_dimension(name="environment", value=os.environ.get("ENVIRONMENT", "dev"))

    extraction_ok = False
    try:
        result = extract_invoice(event)
        extraction_ok = True

        metrics.add_metric(
            name="InvoicesProcessed",
            unit=MetricUnit.Count,
            value=1,
        )
        metrics.add_metric(
            name="ExtractionAccuracy",
            unit=MetricUnit.Percent,
            value=result["confidence_score"] * 100,
        )

        logger.info(
            "Extraction succeeded",
            extra={
                "invoice_id": event.get("invoice_id"),
                "vendor_type": vendor_type,
                "confidence": result["confidence_score"],
            },
        )
        return result

    except PydanticValidationError as exc:
        metrics.add_metric(
            name="PydanticValidationFailures",
            unit=MetricUnit.Count,
            value=1,
        )
        logger.warning(
            "Pydantic validation failed",
            extra={
                "invoice_id": event.get("invoice_id"),
                "vendor_type": vendor_type,
                "errors": str(exc),
            },
        )
        raise

    except BedrockFallbackEvent:
        # Haiku failed; Sonnet fallback triggered
        metrics.add_metric(
            name="BedrockFallbacks",
            unit=MetricUnit.Count,
            value=1,
        )
        logger.warning("Bedrock fallback triggered", extra={"model": "sonnet"})
        # Continue processing with Sonnet
        result = extract_invoice(event, model_override="sonnet")
        extraction_ok = True
        metrics.add_metric(name="InvoicesProcessed", unit=MetricUnit.Count, value=1)
        return result
```

## Key Powertools Behaviors

| Behavior | Detail |
|----------|--------|
| `@metrics.log_metrics` decorator | Flushes all accumulated metrics to stdout at handler return |
| `raise_on_empty_metrics=False` | Do not raise if no metrics added (e.g., in error paths that re-raise) |
| `capture_cold_start_metric=True` | Emits `ColdStart` metric automatically |
| Dimension scope | Dimensions added apply to ALL metrics in that invocation |
| 100-metric limit | Powertools auto-flushes at 100; subsequent metrics go into new EMF blob |

## Dimension Guidelines

```python
# Good: low-cardinality dimensions
metrics.add_dimension(name="vendor_type", value="ubereats")   # 3 distinct values
metrics.add_dimension(name="environment", value="prod")        # 2 distinct values

# Bad: high-cardinality — creates thousands of metric streams
# metrics.add_dimension(name="invoice_id", value=invoice_id)  # NEVER do this
```

## Single Metric Helper (for one-off situations)

```python
from aws_lambda_powertools import single_metric
from aws_lambda_powertools.metrics import MetricUnit

with single_metric(
    name="BedrockFallbacks",
    unit=MetricUnit.Count,
    value=1,
    namespace="InvoicePipeline",
) as metric:
    metric.add_dimension(name="vendor_type", value="doordash")
```

## CloudWatch Namespace

All custom metrics land in namespace `InvoicePipeline`. In dashboards and alarms,
reference this namespace explicitly to avoid confusion with `AWS/Lambda`.

## See Also

- [../concepts/metrics-and-emf.md](../concepts/metrics-and-emf.md) — EMF theory
- [../patterns/slo-alarms.md](slo-alarms.md) — alarms on these metrics
- `.claude/kb/aws/lambda/patterns/powertools-logging.md` — Logger integration
