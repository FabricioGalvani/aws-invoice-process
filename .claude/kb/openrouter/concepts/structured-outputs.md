# Structured Outputs via OpenRouter

> **Purpose**: Enforcing JSON Schema on model responses for reliable Pydantic-parseable extraction
> **Confidence**: 0.92
> **MCP Validated**: 2026-05-13

## Overview

OpenRouter forwards `response_format` to providers that support it (OpenAI GPT-4o+, Anthropic Sonnet 4.5+, Google Gemini models, Fireworks-hosted open-source). For invoice extraction, combining `response_format` with `require_parameters: true` ensures OpenRouter only routes to backends that will actually enforce the schema — preventing silent plain-text fallbacks.

## The Concept

```python
from pydantic import BaseModel
from typing import Optional

class InvoiceExtraction(BaseModel):
    invoice_id: str
    vendor: str
    subtotal: float
    tax: float
    total: float
    currency: str
    issue_date: str          # ISO-8601
    due_date: Optional[str]

schema = InvoiceExtraction.model_json_schema()

response = client.chat.completions.create(
    model="anthropic/claude-haiku-4-5",
    provider={
        "require_parameters": True,   # skip backends that ignore response_format
        "data_collection": "deny",
    },
    response_format={
        "type": "json_schema",
        "json_schema": {
            "name": "InvoiceExtraction",
            "strict": True,            # fail rather than produce non-conforming output
            "schema": schema,
        },
    },
    messages=[
        {"role": "system", "content": "Extract invoice fields as JSON."},
        {"role": "user", "content": f"<invoice>{raw_text}</invoice>"},
    ],
)

# Parse and validate with Pydantic
import json
payload = json.loads(response.choices[0].message.content)
result = InvoiceExtraction.model_validate(payload)
```

## Quick Reference

| Mode | `response_format.type` | Behaviour |
|------|----------------------|-----------|
| JSON object | `"json_object"` | Forces valid JSON, no schema enforcement |
| Strict schema | `"json_schema"` | Validates against your schema; use `strict: true` |

## Common Mistakes

### Wrong

```python
# Using json_object without require_parameters — provider may return valid JSON
# that doesn't match your Pydantic model, causing a silent ValidationError downstream.
response_format={"type": "json_object"}
```

### Correct

```python
# json_schema + strict + require_parameters = maximum schema fidelity
response_format={
    "type": "json_schema",
    "json_schema": {"name": "InvoiceExtraction", "strict": True, "schema": schema},
}
# provider.require_parameters=True ensures only schema-capable backends are used
```

## Related

- [bedrock-fallback-adapter.md](../patterns/bedrock-fallback-adapter.md)
- [pydantic-validated-extraction.md](../patterns/pydantic-validated-extraction.md)
- [provider-preferences.md](../concepts/provider-preferences.md)
