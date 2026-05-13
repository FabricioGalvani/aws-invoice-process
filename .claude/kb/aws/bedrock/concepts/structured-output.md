> **Web Validated:** 2026-05-12
> **Sources:** docs.aws.amazon.com/boto3/latest/reference/services/bedrock-runtime/client/converse.html,
>   docs.aws.amazon.com/bedrock/latest/userguide/tool-use.html

# Structured Output with Claude — Tool Use / JSON Schema

## Why Tool Use for Extraction

Claude's tool-use mechanism is the reliable way to get structured JSON from the
model. When `toolChoice` is set to `{"tool": {"name": "..."}}`, Claude is
**forced** to call that tool, making the response always parseable as JSON.
This is the correct approach for invoice extraction — do not rely on asking
the model to "return JSON" in free text.

## Converse API Tool Configuration

```python
INVOICE_SCHEMA = {
    "type": "object",
    "properties": {
        "invoice_number":   {"type": "string"},
        "invoice_date":     {"type": "string", "format": "date"},
        "vendor_type":      {"type": "string", "enum": ["ubereats", "doordash", "grubhub"]},
        "restaurant_id":    {"type": "string"},
        "gross_sales":      {"type": "number"},
        "commission_rate":  {"type": "number"},
        "commission_amount":{"type": "number"},
        "adjustments":      {"type": "number"},
        "taxes":            {"type": "number"},
        "net_payout":       {"type": "number"},
        "period_start":     {"type": "string", "format": "date"},
        "period_end":       {"type": "string", "format": "date"},
    },
    "required": [
        "invoice_number", "invoice_date", "vendor_type", "restaurant_id",
        "gross_sales", "commission_rate", "commission_amount",
        "adjustments", "taxes", "net_payout", "period_start", "period_end",
    ],
}

tool_config = {
    "tools": [
        {
            "toolSpec": {
                "name": "extract_invoice",
                "description": (
                    "Extract structured data from a delivery platform invoice image. "
                    "Return all 12 fields. Use null for fields not found."
                ),
                "inputSchema": {"json": INVOICE_SCHEMA},
            }
        }
    ],
    "toolChoice": {"tool": {"name": "extract_invoice"}},
}
```

## Parsing the Tool Use Response

```python
def parse_tool_result(response: dict) -> dict:
    stop_reason = response["stopReason"]
    if stop_reason != "tool_use":
        raise ValueError(f"Unexpected stopReason: {stop_reason}")

    content_blocks = response["output"]["message"]["content"]
    for block in content_blocks:
        if block.get("toolUse", {}).get("name") == "extract_invoice":
            return block["toolUse"]["input"]  # already a dict, not a string

    raise ValueError("No extract_invoice tool call found in response")
```

## Stop Reason Reference

| `stopReason` | Meaning | Action |
|-------------|---------|--------|
| `tool_use` | Model called a tool — expected | Parse `toolUse.input` |
| `end_turn` | Model finished without tool call | Error — check toolChoice config |
| `max_tokens` | Hit token limit | Increase maxTokens or shrink prompt |
| `stop_sequence` | Hit a stop sequence | Check system prompt for accidental stops |

## Pydantic Validation Integration

```python
from pydantic import BaseModel, ValidationError
from decimal import Decimal

class InvoiceExtraction(BaseModel):
    invoice_number: str
    invoice_date: str
    vendor_type: str
    restaurant_id: str
    gross_sales: Decimal
    commission_rate: Decimal
    commission_amount: Decimal
    adjustments: Decimal
    taxes: Decimal
    net_payout: Decimal
    period_start: str
    period_end: str

def validate_extraction(raw: dict) -> InvoiceExtraction:
    try:
        return InvoiceExtraction(**raw)
    except ValidationError as exc:
        # Caller should retry with Sonnet 4.5
        raise exc
```

## Anti-Patterns

| Anti-pattern | Problem | Fix |
|-------------|---------|-----|
| Ask model to "return JSON" in text | Model may add prose, hallucinate format | Use `toolChoice: {tool: ...}` |
| `toolChoice: {auto: {}}` | Model may skip the tool call | Force with `{tool: {name: ...}}` |
| Trust `end_turn` as success | Missing extraction | Assert `stopReason == "tool_use"` |
| Parse `toolUse.input` as string | `input` is already a dict | Use directly, no `json.loads()` |
