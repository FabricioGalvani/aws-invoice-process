# OpenRouter: Unified LLM Gateway

> **Purpose**: What OpenRouter is, how it works, and why it matters as a production fallback gateway
> **Confidence**: 0.92
> **MCP Validated**: 2026-05-13

## Overview

OpenRouter is a provider-agnostic API gateway that exposes 300+ models from Anthropic, OpenAI, Google, Meta, Mistral, DeepSeek, and others under a single OpenAI-compatible endpoint. It normalises request/response shapes, handles provider failover, and tracks per-call cost transparently — making it the natural secondary fallback when Amazon Bedrock is unavailable in an extraction pipeline.

## The Concept

```python
# Drop-in replacement for any OpenAI-compatible SDK call.
# Only the base_url and api_key change; everything else stays identical.
from openai import OpenAI

client = OpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key="sk-or-v1-...",          # from AWS Secrets Manager
    default_headers={
        "HTTP-Referer": "https://github.com/your-org/invoice-pipeline",
        "X-Title": "invoice-pipeline",
    },
)

response = client.chat.completions.create(
    model="anthropic/claude-haiku-4-5",   # same model family as primary Bedrock path
    messages=[{"role": "user", "content": "Extract invoice fields."}],
    max_tokens=1024,
)
print(response.choices[0].message.content)
# response.model contains the ACTUAL model used (may differ if fallback fired)
```

## Quick Reference

| Attribute | Value |
|-----------|-------|
| Base URL | `https://openrouter.ai/api/v1` |
| Auth header | `Authorization: Bearer <key>` |
| SDK compat | OpenAI Python SDK v1.x |
| Model format | `{provider}/{model-slug}` e.g. `anthropic/claude-haiku-4-5` |
| Rate limit info | Returned in standard `X-RateLimit-*` headers |
| Cost tracking | `usage.cost` field in response + `/api/v1/generation?id=` endpoint |

## Common Mistakes

### Wrong

```python
# Hardcoding the OpenRouter key in source — gets committed and leaked
client = OpenAI(api_key="sk-or-v1-abc123...")
```

### Correct

```python
import boto3, json

def _get_openrouter_key() -> str:
    sm = boto3.client("secretsmanager")
    secret = sm.get_secret_value(SecretId="prod/invoice-pipeline/openrouter-api-key")
    return json.loads(secret["SecretString"])["api_key"]

client = OpenAI(base_url="https://openrouter.ai/api/v1", api_key=_get_openrouter_key())
```

## Related

- [model-routing.md](../concepts/model-routing.md)
- [provider-preferences.md](../concepts/provider-preferences.md)
- [bedrock-fallback-adapter.md](../patterns/bedrock-fallback-adapter.md)
