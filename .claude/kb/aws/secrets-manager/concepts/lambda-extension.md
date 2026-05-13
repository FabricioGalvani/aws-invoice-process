# AWS Parameters and Secrets Lambda Extension

> **Purpose**: In-memory caching layer for secrets and SSM parameters in Lambda functions
> **Confidence**: 0.95
> **MCP Validated:** 2026-05-12

## Overview

The AWS Parameters and Secrets Lambda Extension is a Lambda layer that runs as a local HTTP server (port 2773) inside the Lambda execution environment. It retrieves secrets from Secrets Manager and parameters from SSM on first request, caches them in-memory, and serves subsequent requests from cache until the TTL expires. This eliminates redundant AWS API calls across invocations within the same execution environment, reducing both latency and cost.

---

## How It Works

```
Lambda invocation
     │
     ▼
Your function code (HTTP GET localhost:2773/secretsmanager/get?secretId=...)
     │
     ▼
Extension checks in-memory cache
     ├── Cache HIT (TTL valid) → returns cached value immediately
     └── Cache MISS → calls AWS Secrets Manager API → caches result → returns value
```

The extension persists across invocations within the same warm execution environment. On cold start, the first call fetches from AWS; subsequent calls within the TTL window are served from memory.

---

## Layer ARN by Region

| Region | ARN |
|--------|-----|
| us-east-1 | `arn:aws:lambda:us-east-1:177933569100:layer:AWS-Parameters-and-Secrets-Lambda-Extension:12` |
| us-west-2 | `arn:aws:lambda:us-west-2:345057560386:layer:AWS-Parameters-and-Secrets-Lambda-Extension:12` |
| eu-west-1 | `arn:aws:lambda:eu-west-1:015030872274:layer:AWS-Parameters-and-Secrets-Lambda-Extension:12` |

Always verify the latest version at the [official docs](https://docs.aws.amazon.com/secretsmanager/latest/userguide/retrieving-secrets_lambda.html).

---

## Configuration — Environment Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `SECRETS_MANAGER_TTL` | `300` (s) | Cache TTL for Secrets Manager values |
| `SSM_PARAMETER_STORE_TTL` | `300` (s) | Cache TTL for SSM parameters |
| `PARAMETERS_SECRETS_EXTENSION_HTTP_PORT` | `2773` | Local HTTP port |
| `PARAMETERS_SECRETS_EXTENSION_CACHE_SIZE` | `1000` | Max items cached simultaneously |
| `PARAMETERS_SECRETS_EXTENSION_LOG_LEVEL` | `warn` | Extension log verbosity |

---

## Runtime Compatibility

The extension exposes a plain HTTP interface. No SDK or language-specific binding is needed. Works with Python 3.x, Node.js, Java, Go, and any custom runtime.

---

## Cache Behavior

- Default TTL: 300 seconds (5 minutes) — suitable for API keys that rarely change
- Cache survives multiple Lambda invocations within the same execution environment (warm container)
- Cache is NOT shared across concurrent Lambda instances — each container has its own cache
- On rotation: reduce TTL to 60s or less to pick up rotated values quickly

---

## Common Mistakes

### Wrong — ignoring the extension and calling boto3 on every invocation

Each invocation makes an API call, incurring latency (50–100ms) and Secrets Manager API charges.

### Wrong — setting TTL to 0 to always get fresh values

Defeats the caching purpose. For API keys, 300s is safe. For DB passwords with rotation, use 60s.

### Correct — attach the layer, set TTL, call localhost:2773

One API call per container lifetime (until TTL), minimal cost, fast cached reads.

---

## Related

- [../patterns/fetch-secret-cached.md](../patterns/fetch-secret-cached.md) — implementation pattern using this extension
- [../patterns/fetch-secret-boto3.md](../patterns/fetch-secret-boto3.md) — fallback if extension unavailable
- [secrets-vs-ssm.md](secrets-vs-ssm.md) — which service to fetch from
