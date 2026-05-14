# Bedrock Fallback Adapter (OpenRouter)

> **Purpose**: Implement OpenRouter as the secondary fallback when Amazon Bedrock is unavailable, following the project's Adapter Pattern (D10)
> **MCP Validated**: 2026-05-13

## When to Use

- Bedrock returns HTTP 5xx or throttling errors after retries
- Bedrock service health check fails during Lambda cold start
- Evaluating alternative models without changing business logic

## Implementation

```python
"""
OpenRouterAdapter — implements LLMAdapter interface (D10 Adapter Pattern).
Drop-in replacement for BedrockAdapter when Bedrock is unavailable.
"""
from __future__ import annotations

import json
import logging
from functools import lru_cache
from typing import Any

import boto3
from openai import OpenAI, APIStatusError, APIConnectionError
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type

logger = logging.getLogger(__name__)

OPENROUTER_SECRET_ID = "prod/invoice-pipeline/openrouter-api-key"
OPENROUTER_BASE_URL = "https://openrouter.ai/api/v1"

# Model priority: keep same family as primary Bedrock path
PRIMARY_MODEL = "anthropic/claude-haiku-4-5"
QUALITY_FALLBACK_MODEL = "anthropic/claude-sonnet-4-5"


@lru_cache(maxsize=1)
def _get_client() -> OpenAI:
    sm = boto3.client("secretsmanager")
    secret = sm.get_secret_value(SecretId=OPENROUTER_SECRET_ID)
    key = json.loads(secret["SecretString"])["api_key"]
    return OpenAI(
        base_url=OPENROUTER_BASE_URL,
        api_key=key,
        timeout=30.0,
        max_retries=0,   # tenacity handles retries
        default_headers={
            "HTTP-Referer": "https://github.com/your-org/invoice-pipeline",
            "X-Title": "invoice-pipeline",
        },
    )


class OpenRouterAdapter:
    """LLMAdapter implementation using OpenRouter as fallback gateway."""

    def __init__(self, use_quality_model: bool = False):
        self.model = QUALITY_FALLBACK_MODEL if use_quality_model else PRIMARY_MODEL
        self._client = _get_client()

    @retry(
        retry=retry_if_exception_type((APIStatusError, APIConnectionError)),
        wait=wait_exponential(multiplier=1, min=2, max=10),
        stop=stop_after_attempt(3),
        reraise=True,
    )
    def invoke(
        self,
        messages: list[dict],
        response_format: dict | None = None,
        max_tokens: int = 1024,
        temperature: float = 0.0,
    ) -> str:
        kwargs: dict[str, Any] = dict(
            model=self.model,
            messages=messages,
            max_tokens=max_tokens,
            temperature=temperature,
            provider={
                "require_parameters": True,
                "data_collection": "deny",
                "allow_fallbacks": True,
            },
        )
        if response_format:
            kwargs["response_format"] = response_format

        response = self._client.chat.completions.create(**kwargs)

        # Log actual model used — may differ from requested if fallback fired
        actual = response.model
        if actual != self.model:
            logger.warning(
                "OpenRouter fallback model used",
                extra={"requested": self.model, "actual": actual},
            )
        return response.choices[0].message.content
```

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `OPENROUTER_SECRET_ID` | `prod/invoice-pipeline/openrouter-api-key` | Secrets Manager path |
| `timeout` | `30.0` s | Per-request timeout |
| `stop_after_attempt` | `3` | Max retries before raising |
| `data_collection` | `"deny"` | Suppress provider data retention |
| `require_parameters` | `True` | Skip backends missing schema support |

## Example Usage

```python
# In your extraction Lambda — wrap Bedrock call with OpenRouter fallback
from adapters.bedrock import BedrockAdapter
from adapters.openrouter import OpenRouterAdapter

def extract_invoice(raw_text: str, use_quality: bool = False) -> dict:
    for AdapterClass in [BedrockAdapter, OpenRouterAdapter]:
        try:
            adapter = AdapterClass(use_quality_model=use_quality)
            return adapter.invoke(messages=[...])
        except Exception as exc:
            logger.warning("Adapter failed, trying next", extra={"adapter": AdapterClass.__name__, "error": str(exc)})
    raise RuntimeError("All LLM adapters exhausted")
```

## See Also

- [pydantic-validated-extraction.md](../patterns/pydantic-validated-extraction.md)
- [retry-timeout-strategy.md](../patterns/retry-timeout-strategy.md)
- [unified-gateway.md](../concepts/unified-gateway.md)
