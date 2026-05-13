# Load Non-Sensitive Config from SSM Parameter Store

> **Purpose**: Retrieve non-sensitive configuration values from SSM Parameter Store using the Lambda Extension or boto3, with caching
> **MCP Validated:** 2026-05-12

## When to Use

- The value is non-sensitive (public key, feature flag, bucket name, endpoint URL)
- The value lives in SSM Parameter Store (Standard tier, free)
- You want to keep cost at zero for configuration that does not need KMS or rotation
- Specifically: `LangFuse Public Key` → `/invoice-pipeline/prod/langfuse/public-key`

## Implementation — Via Lambda Extension (Preferred)

```python
# src/adapters/config/ssm_config_adapter.py
"""
SSM Parameter Store adapter using the AWS Parameters and Secrets Lambda Extension.

The same extension layer that caches Secrets Manager values also caches
SSM parameters — no additional layer required.

Requires:
  - Lambda layer: AWS-Parameters-and-Secrets-Lambda-Extension (already attached)
  - IAM permission: ssm:GetParameter on specific parameter ARN
  - Environment variable: AWS_SESSION_TOKEN (auto-set by Lambda runtime)
"""
from __future__ import annotations

import json
import os
import urllib.error
import urllib.parse
import urllib.request
from functools import lru_cache

_EXTENSION_PORT = os.environ.get("PARAMETERS_SECRETS_EXTENSION_HTTP_PORT", "2773")
_BASE_URL = f"http://localhost:{_EXTENSION_PORT}"


def _extension_get_parameter(name: str, with_decryption: bool = True) -> str:
    """
    Fetch an SSM parameter via the extension cache.

    The extension caches the result for SSM_PARAMETER_STORE_TTL seconds (default: 300).

    Args:
        name: The SSM parameter name (must start with '/' for path-based).
        with_decryption: True to decrypt SecureString parameters.

    Returns:
        The parameter value as a string.
    """
    token = os.environ["AWS_SESSION_TOKEN"]
    encoded_name = urllib.parse.quote(name, safe="")
    url = (
        f"{_BASE_URL}/systemsmanager/parameters/get"
        f"?name={encoded_name}&withDecryption={str(with_decryption).lower()}"
    )
    req = urllib.request.Request(url, headers={"X-Aws-Parameters-Secrets-Token": token})
    try:
        with urllib.request.urlopen(req, timeout=5) as resp:
            data = json.loads(resp.read())
            return data["Parameter"]["Value"]
    except urllib.error.HTTPError as e:
        raise RuntimeError(
            f"Extension returned HTTP {e.code} for SSM param {name}: {e.read().decode()}"
        ) from e


@lru_cache(maxsize=64)
def get_parameter(name: str) -> str:
    """
    Fetch an SSM parameter with module-level lru_cache.

    Combined with the extension's in-memory cache, this gives two layers:
    1. lru_cache: zero-cost lookup within the same process
    2. Extension cache: avoids AWS API call within TTL window across invocations

    Args:
        name: Full SSM parameter path, e.g. '/invoice-pipeline/prod/langfuse/public-key'

    Returns:
        The parameter value as a string.
    """
    return _extension_get_parameter(name)


# --- Project-specific helpers ---

def get_langfuse_public_key() -> str:
    """Retrieve the LangFuse public key from SSM (non-sensitive)."""
    param_name = os.environ.get(
        "LANGFUSE_PUBLIC_KEY_PARAM",
        "/invoice-pipeline/prod/langfuse/public-key"
    )
    return get_parameter(param_name)
```

## Fallback — Direct boto3 (No Extension Layer)

```python
import boto3, os

_ssm = boto3.client("ssm", region_name=os.environ.get("AWS_REGION", "us-east-1"))
_PARAM_CACHE: dict[str, str] = {}

def get_parameter_boto3(name: str) -> str:
    if name not in _PARAM_CACHE:
        response = _ssm.get_parameter(Name=name, WithDecryption=True)
        _PARAM_CACHE[name] = response["Parameter"]["Value"]
    return _PARAM_CACHE[name]
```

## Configuration

| Setting | Where | Value |
|---------|-------|-------|
| `SSM_PARAMETER_STORE_TTL` | Lambda env var | `300` (seconds; shared extension TTL) |
| `LANGFUSE_PUBLIC_KEY_PARAM` | Lambda env var | `/invoice-pipeline/prod/langfuse/public-key` |
| IAM permission | Lambda execution role | `ssm:GetParameter` on parameter ARN |

## Example Usage

```python
# src/lambdas/data_extractor/handler.py
from adapters.config.ssm_config_adapter import get_langfuse_public_key
from adapters.secrets.extension_secrets_adapter import get_secret_value
import os

def lambda_handler(event, context):
    # Non-sensitive: from SSM Parameter Store (free, cached)
    langfuse_public_key = get_langfuse_public_key()

    # Sensitive: from Secrets Manager (cached via extension)
    langfuse_secret_key = get_secret_value(
        os.environ["LANGFUSE_SECRET_ID"], "secret_key"
    )
    # ...
```

## Terraform — SSM Parameter (Non-Sensitive)

```hcl
resource "aws_ssm_parameter" "langfuse_public_key" {
  name  = "/invoice-pipeline/${var.env}/langfuse/public-key"
  type  = "String"   # Not SecureString — non-sensitive
  value = var.langfuse_public_key  # Set via tfvars or CI/CD variable
}
```

## See Also

- [fetch-secret-cached.md](fetch-secret-cached.md) — same extension layer for Secrets Manager
- [../concepts/secrets-vs-ssm.md](../concepts/secrets-vs-ssm.md) — why this value lives in SSM
- [../concepts/naming-and-hierarchy.md](../concepts/naming-and-hierarchy.md) — path naming rules
- [../specs/secrets-iam-policy.json](../specs/secrets-iam-policy.json) — required IAM for ssm:GetParameter
