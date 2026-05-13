# Fetch Secret via Lambda Extension (Preferred)

> **Purpose**: Retrieve Secrets Manager values with zero-cost in-memory caching via the AWS Parameters and Secrets Lambda Extension layer
> **MCP Validated:** 2026-05-12

## When to Use

- Any Lambda function that needs a Secrets Manager secret
- When you want to minimize API calls across warm invocations
- When the extension layer is available (all standard Lambda runtimes)
- Production default — this pattern is always preferred over direct boto3 calls

## Implementation

```python
# src/adapters/secrets/extension_secrets_adapter.py
"""
Secrets adapter using the AWS Parameters and Secrets Lambda Extension.
The extension runs as a local HTTP server on port 2773 and caches
secrets in-memory for the configured TTL (default: 300 seconds).

Requires:
  - Lambda layer: AWS-Parameters-and-Secrets-Lambda-Extension
  - IAM permission: secretsmanager:GetSecretValue on specific secret ARN
  - Environment variable: AWS_SESSION_TOKEN (auto-set by Lambda runtime)
"""
from __future__ import annotations

import json
import os
import urllib.error
import urllib.request
from functools import lru_cache

_EXTENSION_PORT = os.environ.get("PARAMETERS_SECRETS_EXTENSION_HTTP_PORT", "2773")
_BASE_URL = f"http://localhost:{_EXTENSION_PORT}"


def _extension_get(path: str) -> dict:
    """Make a GET request to the local extension HTTP server."""
    token = os.environ["AWS_SESSION_TOKEN"]
    url = f"{_BASE_URL}{path}"
    req = urllib.request.Request(url, headers={"X-Aws-Parameters-Secrets-Token": token})
    try:
        with urllib.request.urlopen(req, timeout=5) as resp:
            return json.loads(resp.read())
    except urllib.error.HTTPError as e:
        raise RuntimeError(
            f"Extension returned HTTP {e.code} for {path}: {e.read().decode()}"
        ) from e


def get_secret(secret_id: str) -> dict:
    """
    Fetch a JSON secret from Secrets Manager via the extension cache.

    The extension caches the result in-memory for SECRETS_MANAGER_TTL seconds
    (default 300). Subsequent calls within the TTL window are served from
    memory without an AWS API call.

    Args:
        secret_id: The secret name or full ARN.

    Returns:
        Parsed dict from the secret's JSON string value.
    """
    import urllib.parse
    encoded_id = urllib.parse.quote(secret_id, safe="")
    data = _extension_get(f"/secretsmanager/get?secretId={encoded_id}")
    return json.loads(data["SecretString"])


def get_secret_value(secret_id: str, key: str) -> str:
    """
    Fetch a single key from a JSON secret.

    Example:
        api_key = get_secret_value("prod/openrouter/api-key", "api_key")
    """
    return get_secret(secret_id)[key]


# Module-level cache for the secret dict itself (avoids repeated JSON parsing)
# This is a second caching layer on top of the extension's own cache.
@lru_cache(maxsize=32)
def _cached_secret(secret_id: str) -> str:
    """Returns JSON string; lru_cache holds it for the process lifetime."""
    import urllib.parse
    encoded_id = urllib.parse.quote(secret_id, safe="")
    data = _extension_get(f"/secretsmanager/get?secretId={encoded_id}")
    return data["SecretString"]


# --- Usage in Lambda handler ---

# OPENROUTER_SECRET_ID is set as a (non-sensitive) Lambda env var
# pointing to the secret name, NOT the secret value.

def get_openrouter_key() -> str:
    secret_id = os.environ["OPENROUTER_SECRET_ID"]  # e.g. "prod/openrouter/api-key"
    return json.loads(_cached_secret(secret_id))["api_key"]
```

## Configuration

| Setting | Where | Value |
|---------|-------|-------|
| Layer ARN | Lambda function definition | `arn:aws:lambda:us-east-1:177933569100:layer:AWS-Parameters-and-Secrets-Lambda-Extension:12` |
| `SECRETS_MANAGER_TTL` | Lambda env var | `300` (default) or `60` if rotation is active |
| `OPENROUTER_SECRET_ID` | Lambda env var | `prod/openrouter/api-key` |
| IAM permission | Lambda execution role | `secretsmanager:GetSecretValue` on secret ARN |

## Example Usage in a Lambda Handler

```python
# src/lambdas/data_extractor/handler.py
import os
from adapters.secrets.extension_secrets_adapter import get_openrouter_key


def lambda_handler(event, context):
    # First invocation: extension fetches from Secrets Manager
    # Subsequent invocations (warm): served from extension in-memory cache
    api_key = get_openrouter_key()

    # Use api_key for OpenRouter calls...
    response = call_openrouter(api_key, event["invoice_data"])
    return response
```

## Terraform — Attach Layer

```hcl
resource "aws_lambda_function" "data_extractor" {
  # ...existing config...

  layers = [
    "arn:aws:lambda:${var.aws_region}:177933569100:layer:AWS-Parameters-and-Secrets-Lambda-Extension:12"
  ]

  environment {
    variables = {
      SECRETS_MANAGER_TTL  = "300"
      OPENROUTER_SECRET_ID = "prod/openrouter/api-key"
      LANGFUSE_SECRET_ID   = "prod/langfuse/secret-key"
      SLACK_SECRET_ID      = "prod/slack/webhook-url"
    }
  }
}
```

## See Also

- [../concepts/lambda-extension.md](../concepts/lambda-extension.md) — how the extension works
- [fetch-secret-boto3.md](fetch-secret-boto3.md) — fallback if extension not available
- [ssm-config-load.md](ssm-config-load.md) — same extension for SSM parameters
- [../specs/secrets-iam-policy.json](../specs/secrets-iam-policy.json) — required IAM permissions
