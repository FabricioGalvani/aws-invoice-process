# Auth Model & Security

> **Purpose**: API key management, BYOK, and secure key handling in AWS Lambda environments
> **Confidence**: 0.90
> **MCP Validated**: 2026-05-13

## Overview

OpenRouter authenticates via a bearer token (`Authorization: Bearer sk-or-v1-...`). In an AWS Lambda pipeline, the key must never appear in environment variables at deploy time or in CloudWatch logs. The correct pattern is Secrets Manager retrieval inside the Lambda handler, cached for the lifetime of the execution environment (warm container reuse).

## The Concept

```python
import boto3
import json
import logging
from functools import lru_cache
from openai import OpenAI

logger = logging.getLogger(__name__)

@lru_cache(maxsize=1)
def _openrouter_client() -> OpenAI:
    """Cached per execution environment — re-fetched on cold start only."""
    sm = boto3.client("secretsmanager", region_name="us-east-1")
    secret = sm.get_secret_value(
        SecretId="prod/invoice-pipeline/openrouter-api-key"
    )
    api_key = json.loads(secret["SecretString"])["api_key"]

    # NEVER log api_key — masked in all structured log calls
    logger.info("OpenRouter client initialised", extra={"key_prefix": api_key[:8]})

    return OpenAI(
        base_url="https://openrouter.ai/api/v1",
        api_key=api_key,
        default_headers={
            "HTTP-Referer": "https://github.com/your-org/invoice-pipeline",
            "X-Title": "invoice-pipeline",
        },
    )

def handler(event, context):
    client = _openrouter_client()   # warm-cache hit after first invocation
    ...
```

## Quick Reference

| Concern | Wrong | Correct |
|---------|-------|---------|
| Key storage | `os.environ["OR_KEY"]` at deploy | Secrets Manager, fetched in handler |
| Key logging | `logger.info(api_key)` | Log only `api_key[:8]` prefix |
| Key rotation | Manual update env var | Rotate in Secrets Manager; Lambda picks up on next cold start |
| BYOK | Not applicable here | Attach provider keys in OpenRouter dashboard to use own Anthropic quota |

## Common Mistakes

### Wrong

```python
# Lambda env var — visible in console, CloudTrail, Lambda config export
api_key = os.environ["OPENROUTER_API_KEY"]
```

### Correct

```python
# Retrieved at runtime, never stored in function config
secret = boto3.client("secretsmanager").get_secret_value(
    SecretId="prod/invoice-pipeline/openrouter-api-key"
)
api_key = json.loads(secret["SecretString"])["api_key"]
```

## Related

- [unified-gateway.md](../concepts/unified-gateway.md)
- [provider-preferences.md](../concepts/provider-preferences.md)
- [bedrock-fallback-adapter.md](../patterns/bedrock-fallback-adapter.md)
