# Secrets Manager + SSM — Quick Reference

> **MCP Validated:** 2026-05-12

## Service Decision (One Line)

**Sensitive credential** → Secrets Manager. **Non-sensitive config** → SSM Parameter Store Standard.

---

## Cost at a Glance

| Service | Storage | API Calls |
|---------|---------|-----------|
| Secrets Manager | $0.40/secret/month | $0.05/10k calls |
| SSM Standard | Free | Free (up to 40 TPS) |
| SSM Advanced | $0.05/param/month | $0.05/10k calls |

Project cost: 3 secrets × $0.40 = **$1.20/month** (Secrets Manager) + $0 (SSM).

---

## Project Secrets Map

| Secret | Path | Service |
|--------|------|---------|
| OpenRouter API Key | `prod/openrouter/api-key` | Secrets Manager |
| LangFuse Secret Key | `prod/langfuse/secret-key` | Secrets Manager |
| Slack Webhook URL | `prod/slack/webhook-url` | Secrets Manager |
| LangFuse Public Key | `/invoice-pipeline/prod/langfuse/public-key` | SSM Parameter Store |

---

## Lambda Extension Layer ARN (us-east-1)

```
arn:aws:lambda:us-east-1:177933569100:layer:AWS-Parameters-and-Secrets-Lambda-Extension:12
```

Check current version at: https://docs.aws.amazon.com/secretsmanager/latest/userguide/retrieving-secrets_lambda.html

---

## Extension — Fetch Secret (curl from Lambda)

```python
import os, urllib.request, json

def get_secret(secret_id: str) -> dict:
    port = os.environ.get("PARAMETERS_SECRETS_EXTENSION_HTTP_PORT", "2773")
    token = os.environ["AWS_SESSION_TOKEN"]
    url = f"http://localhost:{port}/secretsmanager/get?secretId={secret_id}"
    req = urllib.request.Request(url, headers={"X-Aws-Parameters-Secrets-Token": token})
    with urllib.request.urlopen(req) as resp:
        return json.loads(json.loads(resp.read())["SecretString"])
```

---

## Extension — Fetch SSM Parameter

```python
def get_parameter(name: str) -> str:
    port = os.environ.get("PARAMETERS_SECRETS_EXTENSION_HTTP_PORT", "2773")
    token = os.environ["AWS_SESSION_TOKEN"]
    url = f"http://localhost:{port}/systemsmanager/parameters/get?name={name}&withDecryption=true"
    req = urllib.request.Request(url, headers={"X-Aws-Parameters-Secrets-Token": token})
    with urllib.request.urlopen(req) as resp:
        return json.loads(resp.read())["Parameter"]["Value"]
```

---

## IAM Minimum Permissions

```json
{
  "Effect": "Allow",
  "Action": "secretsmanager:GetSecretValue",
  "Resource": "arn:aws:secretsmanager:us-east-1:ACCOUNT:secret:prod/openrouter/api-key-*"
}
```

---

## Environment Variables for Extension

| Variable | Default | Effect |
|----------|---------|--------|
| `SECRETS_MANAGER_TTL` | `300` (5 min) | Cache TTL for secrets (seconds) |
| `SSM_PARAMETER_STORE_TTL` | `300` | Cache TTL for parameters (seconds) |
| `PARAMETERS_SECRETS_EXTENSION_HTTP_PORT` | `2773` | Local HTTP port |
| `PARAMETERS_SECRETS_EXTENSION_CACHE_SIZE` | `1000` | Max cached items |
