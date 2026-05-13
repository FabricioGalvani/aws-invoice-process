> **Web Validated:** 2026-05-12
> **Sources:** notes/07-aws-migration-plan.md §2.4 (D15 updated decision),
>   docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-4-5.html

# Pattern: Haiku → Sonnet Fallback Strategy

## Decision Context (D15)

Primary: **Claude Haiku 4.5** — ~94-95% accuracy, ~$0.001/invoice, ~0.8s P50  
Fallback: **Claude Sonnet 4.5** — ~96-97% accuracy, ~$0.005/invoice, ~1.5s P50

Sonnet 4.5 activates **only** when Pydantic validation of the Haiku response
raises `ValidationError`. This keeps median cost at Haiku levels while recovering
borderline cases with the more capable model.

## Fallback Orchestrator

```python
import logging
from pydantic import ValidationError

from myapp.bedrock import extract_invoice   # uses Haiku 4.5
from myapp.models import InvoiceExtraction

PRIMARY_MODEL   = "us.anthropic.claude-haiku-4-5-20251001-v1:0"
FALLBACK_MODEL  = "us.anthropic.claude-sonnet-4-5-20250929-v1:0"

logger = logging.getLogger(__name__)


def extract_with_fallback(
    png_bytes: bytes,
    vendor_type: str,
    bedrock_client,
) -> tuple[InvoiceExtraction, str]:
    """
    Returns (extraction, model_used).
    Raises RuntimeError if both models fail — caller routes to DLQ.
    """
    for model_id, label in [
        (PRIMARY_MODEL,  "haiku_4_5"),
        (FALLBACK_MODEL, "sonnet_4_5"),
    ]:
        try:
            result = _call_model(
                png_bytes, vendor_type, model_id, bedrock_client
            )
            if label == "sonnet_4_5":
                logger.warning(
                    "fallback_triggered",
                    extra={"model": label, "vendor": vendor_type},
                )
            return result, label

        except ValidationError as exc:
            logger.warning(
                "pydantic_validation_failed",
                extra={"model": label, "errors": exc.error_count()},
            )
            if label == "sonnet_4_5":
                raise RuntimeError(
                    f"Both models failed validation: {exc}"
                ) from exc
            # continue to fallback

        except Exception as exc:
            logger.error(
                "bedrock_call_failed",
                extra={"model": label, "error": str(exc)},
            )
            if label == "sonnet_4_5":
                raise
            # continue to fallback


def _call_model(
    png_bytes: bytes,
    vendor_type: str,
    model_id: str,
    bedrock_client,
) -> InvoiceExtraction:
    """Single model call — see invoke-claude-vision.md for full implementation."""
    from myapp.bedrock import extract_invoice
    return extract_invoice(png_bytes, vendor_type, bedrock_client, model_id=model_id)
```

## Metrics to Emit

```python
import boto3

cloudwatch = boto3.client("cloudwatch")

def emit_model_metric(model_label: str, success: bool) -> None:
    cloudwatch.put_metric_data(
        Namespace="InvoicePipeline",
        MetricData=[{
            "MetricName": "ExtractionAttempt",
            "Dimensions": [
                {"Name": "Model", "Value": model_label},
                {"Name": "Outcome", "Value": "success" if success else "failure"},
            ],
            "Value": 1,
            "Unit": "Count",
        }],
    )
```

Track `sonnet_4_5` invocations as a KPI. If >15% of invoices trigger the
fallback, investigate prompt quality or a shift in invoice format.

## Fallback Decision Rules

| Trigger | Action |
|---------|--------|
| Pydantic `ValidationError` | Retry with Sonnet 4.5 |
| `ThrottlingException` | Retry with backoff (same model) — see retry-throttling.md |
| `stopReason == "max_tokens"` | Increase `maxTokens`, retry same model |
| Both models fail validation | Send to SQS DLQ, emit alert to Slack |

## Cost Impact Calculation

At 3,500 invoices/month with 5% fallback rate:
- 3,325 Haiku calls × $0.001 = **$3.33**
- 175 Sonnet calls × $0.005 = **$0.88**
- Total LLM cost: **~$4.21/month** (matches §6 prod estimate)

## Anti-Patterns

| Anti-pattern | Fix |
|-------------|-----|
| Always call Sonnet first for safety | Costs 5x more; Haiku meets ≥90% target |
| Fallback on any exception | Only fallback on `ValidationError`; throttle = retry |
| Infinite fallback loop | Limit to 1 fallback model; hard-fail to DLQ after |
| Suppress fallback metric | Emit CloudWatch metric for every Sonnet invocation |
