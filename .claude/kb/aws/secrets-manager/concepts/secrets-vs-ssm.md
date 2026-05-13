# Secrets Manager vs SSM Parameter Store — Decision Matrix

> **Purpose**: Choose the right service for each credential or config value
> **Confidence**: 0.95
> **MCP Validated:** 2026-05-12

## Overview

AWS offers two overlapping services for storing secrets and configuration values. The choice is driven by sensitivity, cost, rotation needs, and access patterns. Use Secrets Manager for any value that is a credential or sensitive token; use SSM Parameter Store Standard for everything else.

---

## Decision Matrix

| Dimension | Secrets Manager | SSM Parameter Store Standard |
|-----------|----------------|------------------------------|
| **Cost** | $0.40/secret/month + $0.05/10k API calls | Free storage, free API calls (standard TPS) |
| **Encryption** | KMS-encrypted at rest (default: `aws/secretsmanager`) | Optional KMS (SecureString) or plaintext (String) |
| **Automatic rotation** | Native managed rotation + custom Lambda rotation | No native rotation; requires custom automation |
| **Cross-region replication** | Yes, built-in | No |
| **Versioning** | Full version history with staging labels | Basic version tracking |
| **Path hierarchy** | Flat name with optional `/` convention | Native hierarchical paths (`/app/env/key`) |
| **Max value size** | 65 KB | 4 KB (Standard), 8 KB (Advanced) |
| **Use case** | Database passwords, API keys, webhook URLs | Feature flags, non-sensitive config, public keys |

---

## When to Use Secrets Manager

- The value is a credential: API key, database password, OAuth token, webhook URL
- You need automatic or scheduled rotation (DB credentials especially)
- You need cross-region replication for DR
- The value must never appear in plaintext in infrastructure state

## When to Use SSM Parameter Store

- The value is non-sensitive configuration: a public key, a feature flag, a bucket name
- You want zero storage cost (Standard tier)
- You want native hierarchical path browsing (`/invoice-pipeline/prod/*`)
- The value does not require rotation

---

## Project-Specific Assignments (§4.6)

| Secret | Classification | Service | Path |
|--------|---------------|---------|------|
| OpenRouter API Key | Sensitive — API credential | **Secrets Manager** | `prod/openrouter/api-key` |
| LangFuse Secret Key | Sensitive — API credential | **Secrets Manager** | `prod/langfuse/secret-key` |
| Slack Webhook URL | Sensitive — callable URL | **Secrets Manager** | `prod/slack/webhook-url` |
| LangFuse Public Key | Non-sensitive — public identifier | **SSM Parameter Store** | `/invoice-pipeline/prod/langfuse/public-key` |

---

## Common Mistakes

### Wrong — storing non-sensitive config in Secrets Manager

Unnecessary cost ($0.40/month × many params) when SSM Standard is free.

### Wrong — storing API keys in SSM as `String` type

`String` is unencrypted. Use `SecureString` for anything sensitive — but at that point, Secrets Manager is the better choice for its rotation and audit features.

### Correct — use Secrets Manager only for secrets; SSM for the rest

Keeps costs minimal while ensuring sensitive credentials have proper rotation support and KMS encryption.

---

## Related

- [naming-and-hierarchy.md](naming-and-hierarchy.md)
- [rotation-strategies.md](rotation-strategies.md)
- [../patterns/fetch-secret-cached.md](../patterns/fetch-secret-cached.md)
- [../specs/secrets-iam-policy.json](../specs/secrets-iam-policy.json)
