> **Web Validated:** 2026-05-12
> **Sources:** docs.aws.amazon.com/boto3/latest/reference/services/bedrock-runtime/client/converse.html,
>   docs.aws.amazon.com/bedrock/latest/userguide/bedrock-runtime_example_bedrock-runtime_Converse_AnthropicClaude_section.html

# Pattern: Invoke Claude Vision — PNG Bytes to JSON Extraction

## Context

The `data-extractor` Lambda receives an SQS message pointing to a PNG in
`s3://invoices-processed-{env}`. It downloads the PNG, sends it to Bedrock
via the Converse API with a tool definition, and parses the structured response.

## Complete Pattern

```python
import boto3
import json
from pydantic import ValidationError

from myapp.models import InvoiceExtraction  # Pydantic model, 12 fields
from myapp.prompts import get_system_prompt  # prompt per vendor_type

PRIMARY_MODEL = "us.anthropic.claude-haiku-4-5-20251001-v1:0"

TOOL_CONFIG = {
    "tools": [{
        "toolSpec": {
            "name": "extract_invoice",
            "description": "Extract all 12 fields from delivery platform invoice.",
            "inputSchema": {"json": InvoiceExtraction.model_json_schema()},
        }
    }],
    "toolChoice": {"tool": {"name": "extract_invoice"}},
}


def extract_invoice(
    png_bytes: bytes,
    vendor_type: str,
    bedrock_client=None,
) -> InvoiceExtraction:
    """
    Send a PNG invoice to Claude Haiku 4.5 and return validated extraction.
    Raises ValidationError if the model output fails Pydantic validation
    (caller should retry with Sonnet 4.5).
    """
    if bedrock_client is None:
        bedrock_client = boto3.client("bedrock-runtime")

    system_prompt = get_system_prompt(vendor_type)

    response = bedrock_client.converse(
        modelId=PRIMARY_MODEL,
        system=[{"text": system_prompt}],
        messages=[{
            "role": "user",
            "content": [
                {
                    "image": {
                        "format": "png",
                        "source": {"bytes": png_bytes},  # Raw bytes — no base64
                    }
                },
                {"text": "Extract all invoice data using the extract_invoice tool."},
            ],
        }],
        toolConfig=TOOL_CONFIG,
        inferenceConfig={
            "maxTokens": 512,   # Right-sized: 12-field JSON fits in ~200 tokens
            "temperature": 0.0,  # Deterministic extraction
        },
    )

    _assert_stop_reason(response)
    raw = _parse_tool_call(response)
    return InvoiceExtraction(**raw)  # Raises ValidationError if invalid


def _assert_stop_reason(response: dict) -> None:
    stop = response.get("stopReason")
    if stop == "max_tokens":
        raise RuntimeError(
            "Bedrock stopped at max_tokens — increase maxTokens or reduce prompt size"
        )
    if stop != "tool_use":
        raise RuntimeError(f"Unexpected stopReason: {stop!r}")


def _parse_tool_call(response: dict) -> dict:
    content = response["output"]["message"]["content"]
    for block in content:
        tool_use = block.get("toolUse", {})
        if tool_use.get("name") == "extract_invoice":
            return tool_use["input"]  # Already a dict — do NOT json.loads()
    raise RuntimeError("extract_invoice tool call missing from response")
```

## Image Constraints (Converse API)

| Constraint | Limit |
|-----------|-------|
| Formats | png, jpeg, gif, webp |
| Max images per message | 20 |
| Max size per image | 3.75 MB |
| Max dimensions | 8000 x 8000 px |

## S3 Download Pattern

```python
import boto3

def get_png_bytes(bucket: str, key: str) -> bytes:
    s3 = boto3.client("s3")
    obj = s3.get_object(Bucket=bucket, Key=key)
    return obj["Body"].read()
```

## Prompt Design Notes

- Use a `system` block with vendor-specific instructions (UberEats, DoorDash, Grubhub
  have different invoice layouts).
- Keep the user turn short — the image carries the content.
- `temperature=0.0` is essential for deterministic extraction.
- Set `maxTokens` to 512 or less for this pipeline. 12 fields of JSON is ~150-200
  tokens. A high ceiling (e.g. 4096) wastes quota due to the 5x output multiplier.

## Anti-Patterns

| Anti-pattern | Fix |
|-------------|-----|
| Pass base64-encoded string | Pass raw bytes; SDK handles encoding |
| `toolChoice: {auto: {}}` | Use `{tool: {name: "extract_invoice"}}` |
| Parse `toolUse.input` with `json.loads()` | `input` is already a dict |
| Skip `stopReason` check | Always validate before parsing |
| `maxTokens=4096` for JSON output | Use 512; reduces quota burn |
