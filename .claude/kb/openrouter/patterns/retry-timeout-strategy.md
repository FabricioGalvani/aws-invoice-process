# Retry & Timeout Strategy for Production

> **Purpose**: Tenacity-based retry pattern for OpenRouter calls in AWS Lambda with circuit-breaker semantics
> **MCP Validated**: 2026-05-13

## When to Use

- Any OpenRouter call in a Lambda function where cold-start + retry budget must fit within the function timeout
- When you need structured logs for retry events to surface in CloudWatch dashboards
- Differentiating retryable (5xx, connection) from non-retryable (4xx, validation) errors

## Implementation

```python
"""
Retry and timeout strategy for OpenRouter in Lambda.
Total worst-case time: 3 attempts × 10s timeout + 2+4 backoff ≈ 46s.
Ensure Lambda timeout > 60s when using this pattern.
"""
from __future__ import annotations

import logging
import time
from typing import Callable, TypeVar

from openai import (
    APIConnectionError,
    APIStatusError,
    RateLimitError,
)
from tenacity import (
    RetryError,
    retry,
    retry_if_exception,
    stop_after_attempt,
    wait_exponential,
    before_sleep_log,
    after_log,
)

logger = logging.getLogger(__name__)

T = TypeVar("T")

# Errors worth retrying (transient infrastructure failures)
_RETRYABLE = (APIConnectionError, RateLimitError)

def _is_retryable(exc: BaseException) -> bool:
    if isinstance(exc, _RETRYABLE):
        return True
    if isinstance(exc, APIStatusError) and exc.status_code >= 500:
        return True
    return False


def with_openrouter_retry(fn: Callable[[], T], context: str = "") -> T:
    """
    Wrap an OpenRouter call with tenacity retry + structured logging.

    Args:
        fn: Zero-arg callable that makes the OpenRouter call.
        context: Descriptive label for log messages.

    Returns:
        The result of fn() on success.

    Raises:
        The last exception after all attempts are exhausted.
    """
    @retry(
        retry=retry_if_exception(_is_retryable),
        wait=wait_exponential(multiplier=1, min=2, max=10),
        stop=stop_after_attempt(3),
        before_sleep=before_sleep_log(logger, logging.WARNING),
        after=after_log(logger, logging.DEBUG),
        reraise=True,
    )
    def _inner() -> T:
        start = time.monotonic()
        result = fn()
        elapsed = time.monotonic() - start
        logger.info(
            "OpenRouter call succeeded",
            extra={"context": context, "elapsed_ms": round(elapsed * 1000)},
        )
        return result

    try:
        return _inner()
    except RetryError as exc:
        logger.error(
            "OpenRouter retries exhausted",
            extra={"context": context, "attempts": 3},
        )
        raise exc.last_attempt.exception() from exc


# --- Convenience wrapper for adapter use ---
def invoke_with_retry(client, model: str, messages: list, **kwargs) -> str:
    def _call():
        resp = client.chat.completions.create(
            model=model,
            messages=messages,
            timeout=10.0,      # per-attempt timeout, not total
            **kwargs,
        )
        return resp.choices[0].message.content

    return with_openrouter_retry(_call, context=f"invoke/{model}")
```

## Configuration

| Setting | Value | Rationale |
|---------|-------|-----------|
| `stop_after_attempt` | 3 | 3 attempts = 2 retries; keeps total time < Lambda timeout |
| `wait_exponential(min=2, max=10)` | 2s, 4s | Conservative backoff for rate-limit recovery |
| `timeout` per attempt | 10s | P99 latency for Haiku; adjust for Sonnet (up to 30s) |
| Lambda timeout | ≥ 60s | Must exceed worst-case retry budget |

## Example Usage

```python
content = invoke_with_retry(
    client=_get_client(),
    model="anthropic/claude-haiku-4-5",
    messages=messages,
    max_tokens=512,
    temperature=0.0,
    provider={"data_collection": "deny", "require_parameters": True},
)
```

## See Also

- [bedrock-fallback-adapter.md](../patterns/bedrock-fallback-adapter.md)
- [cost-telemetry.md](../patterns/cost-telemetry.md)
