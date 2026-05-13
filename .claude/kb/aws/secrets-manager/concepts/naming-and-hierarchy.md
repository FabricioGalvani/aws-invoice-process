# Naming Conventions and Path Hierarchy

> **Purpose**: Consistent secret and parameter naming for the Invoice Processing Pipeline
> **Confidence**: 0.95
> **MCP Validated:** 2026-05-12

## Overview

Consistent naming is critical for IAM scoping, secret discovery, and multi-environment management. The project uses a `{env}/{service}/{key}` hierarchy for Secrets Manager and a `/invoice-pipeline/{env}/{service}/{key}` hierarchy for SSM Parameter Store. The difference reflects SSM's native path-browsing capability (using `/` prefix) vs Secrets Manager's flat namespace.

---

## Secrets Manager Naming Pattern

```
{env}/{service}/{key}
```

| Segment | Values | Example |
|---------|--------|---------|
| `{env}` | `dev`, `prod` | `prod` |
| `{service}` | The external service name | `openrouter`, `langfuse`, `slack` |
| `{key}` | The credential type | `api-key`, `secret-key`, `webhook-url` |

### Project Secrets

| Full Path | Description |
|-----------|-------------|
| `dev/openrouter/api-key` | OpenRouter API key (dev environment) |
| `prod/openrouter/api-key` | OpenRouter API key (production) |
| `dev/langfuse/secret-key` | LangFuse secret key (dev) |
| `prod/langfuse/secret-key` | LangFuse secret key (prod) |
| `dev/slack/webhook-url` | Slack webhook (dev) |
| `prod/slack/webhook-url` | Slack webhook (prod) |

---

## SSM Parameter Store Naming Pattern

```
/invoice-pipeline/{env}/{service}/{key}
```

Leading `/` is required for SSM path-based GetParametersByPath API calls. The `invoice-pipeline` prefix namespaces all project parameters and allows IAM scoping with a single path prefix.

### Project Parameters

| Full Path | Type | Description |
|-----------|------|-------------|
| `/invoice-pipeline/dev/langfuse/public-key` | String | LangFuse public key (dev) |
| `/invoice-pipeline/prod/langfuse/public-key` | String | LangFuse public key (prod) |

---

## IAM Path Scoping

Using consistent prefixes enables least-privilege IAM with minimal policy statements.

### Secrets Manager — scope by env prefix

```json
{
  "Resource": "arn:aws:secretsmanager:us-east-1:ACCOUNT:secret:prod/*"
}
```

Allows reading all `prod/` secrets. For tighter scoping, list individual ARNs per Lambda.

### SSM Parameter Store — scope by path prefix

```json
{
  "Resource": "arn:aws:ssm:us-east-1:ACCOUNT:parameter/invoice-pipeline/prod/*"
}
```

---

## Naming Rules

1. Use lowercase kebab-case for all segments (`api-key`, not `ApiKey` or `API_KEY`)
2. Do NOT include `prod` or `dev` in the `{key}` segment — environment is always in the first segment
3. Secrets Manager names are case-sensitive — be consistent
4. Avoid embedding account IDs or region names in secret names (they are implicit from the ARN)
5. SSM paths must start with `/` for GetParametersByPath to work

---

## Common Mistakes

### Wrong — environment suffix at the end

```
openrouter/api-key-prod   # hard to IAM-scope by env prefix
```

### Wrong — uppercase or underscores

```
prod/OpenRouter/API_KEY   # inconsistent, harder to query
```

### Correct — env always first, lowercase kebab

```
prod/openrouter/api-key
```

---

## Related

- [secrets-vs-ssm.md](secrets-vs-ssm.md) — which service to use
- [../specs/secrets-iam-policy.json](../specs/secrets-iam-policy.json) — IAM policy using these ARN patterns
- [../patterns/terraform-secret-without-value.md](../patterns/terraform-secret-without-value.md) — how names are declared in Terraform
