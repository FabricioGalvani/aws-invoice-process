> **Web Validated:** 2026-05-12
> **Sources:** docs.aws.amazon.com/bedrock/latest/userguide (quota/throttling),
>   aws.amazon.com/blogs/machine-learning/optimize-your-applications-for-scale-and-reliability-on-amazon-bedrock

# Pattern: Retry on ThrottlingException — Exponential Backoff with Jitter

## Why This Matters

Bedrock throttles on two dimensions: **Requests Per Minute (RPM)** and
**Tokens Per Minute (TPM)**. Claude models apply a **5x output-token burndown
multiplier** for TPM quota. A single request with 200 output tokens consumes
1,000 of your TPM quota. Bursts in the SQS queue can spike into throttle territory.

`ThrottlingException` (HTTP 429) must be retried — it is transient. Do **not**
route throttled requests to the DLQ; that wastes invoices.

## Full Retry Implementation

```python
import time
import random
import logging
from botocore.exceptions import ClientError

logger = logging.getLogger(__name__)

MAX_RETRIES = 4
BASE_DELAY_S = 1.0
MAX_DELAY_S  = 30.0
JITTER_FACTOR = 0.25  # ±25% random jitter


def bedrock_converse_with_retry(client, **kwargs) -> dict:
    """
    Wraps client.converse() with exponential backoff + full jitter for
    ThrottlingException and ServiceUnavailableException.
    Propagates all other exceptions immediately.
    """
    last_exc = None

    for attempt in range(MAX_RETRIES + 1):
        try:
            return client.converse(**kwargs)

        except ClientError as exc:
            code = exc.response["Error"]["Code"]

            if code in ("ThrottlingException", "ServiceUnavailableException"):
                if attempt == MAX_RETRIES:
                    logger.error(
                        "bedrock_max_retries_exceeded",
                        extra={"attempts": attempt + 1, "error_code": code},
                    )
                    raise

                delay = _backoff_delay(attempt)
                logger.warning(
                    "bedrock_throttled_retrying",
                    extra={"attempt": attempt + 1, "wait_s": round(delay, 2)},
                )
                time.sleep(delay)
                last_exc = exc

            else:
                # ModelErrorException, ValidationException, etc. — do not retry
                raise

    raise last_exc  # unreachable but satisfies type checkers


def _backoff_delay(attempt: int) -> float:
    """Full-jitter exponential backoff — capped at MAX_DELAY_S."""
    cap = min(MAX_DELAY_S, BASE_DELAY_S * (2 ** attempt))
    jitter = random.uniform(cap * (1 - JITTER_FACTOR), cap)
    return jitter
```

## boto3 Adaptive Retry Mode (Alternative)

boto3 supports a built-in adaptive retry mode that adjusts retry rate based
on observed throttle responses. Activate it via the session config:

```python
from botocore.config import Config

config = Config(retries={"mode": "adaptive", "max_attempts": 5})
client = boto3.client("bedrock-runtime", config=config)
```

The adaptive mode is appropriate for **load tests** and steady-state production.
However, the manual implementation above gives finer control over logging and
metrics — preferred for this pipeline.

## Retry-able vs Non-Retry-able Errors

| Error Code | Retry? | Action |
|-----------|--------|--------|
| `ThrottlingException` | Yes | Exponential backoff |
| `ServiceUnavailableException` | Yes | Exponential backoff |
| `ModelErrorException` | No | Log + route to DLQ |
| `ValidationException` | No | Fix request payload |
| `ResourceNotFoundException` | No | Fix model ID |
| Pydantic `ValidationError` | No | Trigger Sonnet fallback |

## CloudWatch Alarm for Throttling

```hcl
# Terraform — alert when throttle rate is elevated
resource "aws_cloudwatch_metric_alarm" "bedrock_throttle" {
  alarm_name          = "bedrock-throttle-rate-high"
  metric_name         = "ThrottledRequests"
  namespace           = "AWS/Bedrock"
  statistic           = "Sum"
  period              = 300
  evaluation_periods  = 2
  threshold           = 10
  comparison_operator = "GreaterThanThreshold"
  alarm_actions       = [var.sns_topic_arn]
}
```

## Anti-Patterns

| Anti-pattern | Fix |
|-------------|-----|
| Route `ThrottlingException` to DLQ | Retry with backoff — it's transient |
| Fixed sleep between retries | Add jitter to spread retry storms |
| Retry `ModelErrorException` | Non-transient — fix the request |
| Infinite retry loop | Cap at MAX_RETRIES (4), then raise |
| `time.sleep(60)` flat | Exponential + jitter is faster on average |
