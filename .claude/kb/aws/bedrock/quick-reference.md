> **Web Validated:** 2026-05-12
> **Sources:** docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-haiku-4-5.html,
>   docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-4-5.html

# Bedrock Quick Reference — Invoice Extractor

## Model IDs

```text
# In-region (single region, strictest data residency)
anthropic.claude-haiku-4-5-20251001-v1:0
anthropic.claude-sonnet-4-5-20250929-v1:0

# US cross-region geo profile (recommended default)
us.anthropic.claude-haiku-4-5-20251001-v1:0
us.anthropic.claude-sonnet-4-5-20250929-v1:0

# Global cross-region (max throughput, ~10% cheaper)
global.anthropic.claude-haiku-4-5-20251001-v1:0
global.anthropic.claude-sonnet-4-5-20250929-v1:0
```

## Client Init

```python
import boto3

client = boto3.client("bedrock-runtime", region_name="us-east-1")
```

## Minimal Converse Call

```python
response = client.converse(
    modelId="us.anthropic.claude-haiku-4-5-20251001-v1:0",
    messages=[{"role": "user", "content": [{"text": "Hello"}]}],
    inferenceConfig={"maxTokens": 1024, "temperature": 0.0},
)
text = response["output"]["message"]["content"][0]["text"]
stop = response["stopReason"]  # end_turn | max_tokens | tool_use
```

## Image Input (PNG bytes)

```python
{"image": {"format": "png", "source": {"bytes": raw_bytes}}}
```

## Structured Output (tool-use)

```python
toolConfig={
    "tools": [{"toolSpec": {"name": "extract_invoice",
                            "description": "...",
                            "inputSchema": {"json": SCHEMA}}}],
    "toolChoice": {"tool": {"name": "extract_invoice"}},
}
```

## Cost/Latency Summary

| Model | Input $/1M | Output $/1M | P50 latency | Use when |
|-------|-----------|------------|-------------|----------|
| Haiku 4.5 | $1.00 | $5.00 | ~0.8s | Default — every invoice |
| Sonnet 4.5 | ~$3.00 | ~$15.00 | ~1.5s | Pydantic validation fails |

## Anti-Pattern Checklist

- Never hardcode model ID without the date-stamp suffix
- Always check `stopReason != "max_tokens"` before parsing
- Retry `ThrottlingException` with exponential backoff + jitter
- Never set `maxTokens` arbitrarily high (consumes quota via 5x multiplier)
- Always scope `bedrock:InvokeModel` to specific model ARNs in IAM
