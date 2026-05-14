# Pydantic-Validated Structured Extraction via OpenRouter

> **Purpose**: End-to-end pattern for invoice field extraction with JSON Schema enforcement and Pydantic validation through OpenRouter
> **MCP Validated**: 2026-05-13

## When to Use

- Primary Bedrock path failed Pydantic validation and OpenRouter is the secondary fallback
- Evaluating an alternative model for structured extraction accuracy
- Needing strict schema enforcement across heterogeneous provider backends

## Implementation

```python
"""
Structured extraction with json_schema + Pydantic validation.
Compatible with any OpenRouter-routed model that supports response_format.
"""
from __future__ import annotations

import json
import logging
from typing import Optional, TypeVar

from openai import OpenAI
from pydantic import BaseModel, ValidationError

logger = logging.getLogger(__name__)

T = TypeVar("T", bound=BaseModel)

# --- Schema definition (project-specific 12-field invoice schema) ---
class InvoiceExtraction(BaseModel):
    invoice_id: str
    vendor_name: str
    vendor_platform: str                    # "UberEats" | "DoorDash" | "Grubhub"
    restaurant_id: str
    subtotal: float
    tax: float
    fees: float
    total: float
    currency: str                           # ISO-4217
    issue_date: str                         # ISO-8601
    due_date: Optional[str] = None
    line_items_count: int


def extract_with_schema(
    client: OpenAI,
    model: str,
    raw_text: str,
    schema_class: type[T],
    max_retries: int = 1,
) -> T:
    """
    Call OpenRouter with json_schema enforcement, parse and validate with Pydantic.
    Raises ValidationError if model output doesn't conform after max_retries.
    """
    schema = schema_class.model_json_schema()
    schema_name = schema_class.__name__

    for attempt in range(max_retries + 1):
        response = client.chat.completions.create(
            model=model,
            provider={
                "require_parameters": True,   # skip backends without schema support
                "data_collection": "deny",
            },
            response_format={
                "type": "json_schema",
                "json_schema": {
                    "name": schema_name,
                    "strict": True,
                    "schema": schema,
                },
            },
            messages=[
                {
                    "role": "system",
                    "content": (
                        f"Extract invoice data as JSON matching the {schema_name} schema. "
                        "Return ONLY valid JSON. No explanation, no markdown."
                    ),
                },
                {"role": "user", "content": f"<invoice>\n{raw_text}\n</invoice>"},
            ],
            max_tokens=512,
            temperature=0.0,                  # deterministic for extraction
        )

        raw_json = response.choices[0].message.content
        try:
            payload = json.loads(raw_json)
            result = schema_class.model_validate(payload)
            logger.info(
                "Extraction success",
                extra={
                    "model": response.model,
                    "attempt": attempt,
                    "input_tokens": response.usage.prompt_tokens,
                    "output_tokens": response.usage.completion_tokens,
                },
            )
            return result
        except (json.JSONDecodeError, ValidationError) as exc:
            logger.warning(
                "Extraction validation failure",
                extra={"attempt": attempt, "error": str(exc)},
            )
            if attempt >= max_retries:
                raise

    raise RuntimeError("Unreachable")  # for type checker
```

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `strict` | `True` | Enforce schema exactly; reject non-conforming output |
| `temperature` | `0.0` | Deterministic extraction |
| `require_parameters` | `True` | Only route to schema-capable backends |
| `max_retries` | `1` | One retry on ValidationError before propagating |

## Example Usage

```python
client = _get_client()                          # from auth-and-security pattern
result = extract_with_schema(
    client=client,
    model="anthropic/claude-haiku-4-5",
    raw_text=invoice_text,
    schema_class=InvoiceExtraction,
)
print(result.total, result.vendor_platform)
```

## See Also

- [bedrock-fallback-adapter.md](../patterns/bedrock-fallback-adapter.md)
- [retry-timeout-strategy.md](../patterns/retry-timeout-strategy.md)
- [structured-outputs.md](../concepts/structured-outputs.md)
