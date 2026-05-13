# Fetch Secret via boto3 with Module-Level Cache (Fallback)

> **Purpose**: Retrieve Secrets Manager values using boto3 when the Lambda Extension layer is not available, with module-level caching to avoid per-invocation API calls
> **MCP Validated:** 2026-05-12

## When to Use

- The Lambda Extension layer cannot be attached (e.g., container image with strict size budget)
- Local development or testing where the extension HTTP server is not running
- Unit testing with mocked boto3 clients
- As a reference for understanding what the extension replaces

**Prefer the extension layer pattern in production.** This is the fallback.

## Implementation

```python
# src/adapters/secrets/boto3_secrets_adapter.py
"""
Secrets adapter using boto3 with module-level caching.

Module-level caching (via _SECRET_CACHE dict) persists across invocations
within the same warm Lambda execution environment — equivalent to the
extension layer's in-memory cache, but implemented manually.

Requires:
  - boto3 >= 1.26 (available in Lambda Python 3.12 runtime)
  - IAM permission: secretsmanager:GetSecretValue on specific secret ARN
"""
from __future__ import annotations

import json
import os
import time
from typing import Any

import boto3
from botocore.exceptions import ClientError

# Module-level client — reused across warm invocations (important: do NOT
# create boto3 clients inside the handler function).
_client = boto3.client("secretsmanager", region_name=os.environ.get("AWS_REGION", "us-east-1"))

# Simple TTL cache: {secret_id: (value_dict, expiry_timestamp)}
_SECRET_CACHE: dict[str, tuple[dict, float]] = {}
_CACHE_TTL_SECONDS = int(os.environ.get("SECRET_CACHE_TTL", "300"))


def get_secret(secret_id: str, *, ttl: int = _CACHE_TTL_SECONDS) -> dict:
    """
    Fetch a JSON secret from Secrets Manager with TTL-based module-level caching.

    Args:
        secret_id: The secret name or full ARN.
        ttl: Cache TTL in seconds (default: SECRET_CACHE_TTL env var or 300).

    Returns:
        Parsed dict from the secret's JSON string value.

    Raises:
        RuntimeError: If the secret cannot be retrieved.
    """
    now = time.monotonic()
    cached_value, expiry = _SECRET_CACHE.get(secret_id, ({}, 0.0))

    if cached_value and now < expiry:
        return cached_value  # Cache hit — no AWS API call

    try:
        response = _client.get_secret_value(SecretId=secret_id)
    except ClientError as e:
        error_code = e.response["Error"]["Code"]
        if error_code == "ResourceNotFoundException":
            raise RuntimeError(f"Secret not found: {secret_id}") from e
        if error_code == "AccessDeniedException":
            raise RuntimeError(f"Access denied to secret: {secret_id}") from e
        raise RuntimeError(f"Failed to retrieve secret {secret_id}: {e}") from e

    secret_string = response.get("SecretString")
    if not secret_string:
        raise RuntimeError(f"Secret {secret_id} has no SecretString (binary secrets not supported)")

    value = json.loads(secret_string)
    _SECRET_CACHE[secret_id] = (value, now + ttl)
    return value


def get_secret_value(secret_id: str, key: str) -> str:
    """Fetch a single key from a JSON secret."""
    return get_secret(secret_id)[key]


def invalidate_cache(secret_id: str | None = None) -> None:
    """
    Invalidate cache after a rotation event.

    Args:
        secret_id: Specific secret to invalidate. Pass None to clear all.
    """
    if secret_id is None:
        _SECRET_CACHE.clear()
    else:
        _SECRET_CACHE.pop(secret_id, None)


# --- Usage helpers for this project ---

def get_openrouter_key() -> str:
    secret_id = os.environ["OPENROUTER_SECRET_ID"]
    return get_secret_value(secret_id, "api_key")


def get_langfuse_secret_key() -> str:
    secret_id = os.environ["LANGFUSE_SECRET_ID"]
    return get_secret_value(secret_id, "secret_key")


def get_slack_webhook_url() -> str:
    secret_id = os.environ["SLACK_SECRET_ID"]
    return get_secret_value(secret_id, "webhook_url")
```

## Configuration

| Setting | Where | Value |
|---------|-------|-------|
| `SECRET_CACHE_TTL` | Lambda env var | `300` (seconds) |
| `OPENROUTER_SECRET_ID` | Lambda env var | `prod/openrouter/api-key` |
| `LANGFUSE_SECRET_ID` | Lambda env var | `prod/langfuse/secret-key` |
| `SLACK_SECRET_ID` | Lambda env var | `prod/slack/webhook-url` |
| IAM permission | Lambda execution role | `secretsmanager:GetSecretValue` scoped to ARN |

## Example Usage

```python
# src/lambdas/data_extractor/handler.py
from adapters.secrets.boto3_secrets_adapter import get_openrouter_key

def lambda_handler(event, context):
    # First warm call: hits Secrets Manager API, caches result
    # Subsequent warm calls within TTL: served from _SECRET_CACHE dict
    api_key = get_openrouter_key()
    # ...
```

## Anti-Pattern — Never Do This

```python
# WRONG: boto3 client created inside handler (no connection reuse)
# WRONG: no caching (API call on every invocation)
def lambda_handler(event, context):
    client = boto3.client("secretsmanager")  # new client every call
    secret = client.get_secret_value(SecretId="prod/openrouter/api-key")
    # ...
```

Cost impact: at 3,500 invoices/month × 3 secrets each = 10,500 API calls/month = $0.05 extra/month. Latency impact: ~50–100ms per cold API call added to every invocation.

## See Also

- [fetch-secret-cached.md](fetch-secret-cached.md) — preferred extension layer approach
- [../concepts/lambda-extension.md](../concepts/lambda-extension.md) — why the extension is preferred
- [../specs/secrets-iam-policy.json](../specs/secrets-iam-policy.json) — IAM permissions
