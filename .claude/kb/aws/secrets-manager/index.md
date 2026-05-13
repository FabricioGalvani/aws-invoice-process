# AWS Secrets Manager + SSM Parameter Store

> **Purpose**: Secrets and config management for Lambda functions in the Invoice Processing Pipeline
> **MCP Validated:** 2026-05-12

## Quick Navigation

### Concepts (< 150 lines each)

| File | Purpose |
|------|---------|
| [concepts/secrets-vs-ssm.md](concepts/secrets-vs-ssm.md) | Decision matrix: when to use each service |
| [concepts/lambda-extension.md](concepts/lambda-extension.md) | AWS Parameters and Secrets Lambda Extension (cached fetch) |
| [concepts/naming-and-hierarchy.md](concepts/naming-and-hierarchy.md) | `{env}/{service}/{key}` naming convention |
| [concepts/rotation-strategies.md](concepts/rotation-strategies.md) | Native rotation vs manual; alarms on LastRotationDate |

### Patterns (< 200 lines each)

| File | Purpose |
|------|---------|
| [patterns/fetch-secret-cached.md](patterns/fetch-secret-cached.md) | Preferred: extension layer HTTP fetch with in-memory cache |
| [patterns/fetch-secret-boto3.md](patterns/fetch-secret-boto3.md) | Fallback: boto3 with module-level cache |
| [patterns/ssm-config-load.md](patterns/ssm-config-load.md) | Non-sensitive config retrieval from SSM Parameter Store |
| [patterns/terraform-secret-without-value.md](patterns/terraform-secret-without-value.md) | Create secret container in Terraform; set value out-of-band |

### Specs (Machine-Readable)

| File | Purpose |
|------|---------|
| [specs/secrets-iam-policy.json](specs/secrets-iam-policy.json) | Least-privilege IAM policy examples |

---

## Project Secrets (§4.6 of 07-aws-migration-plan.md)

| Secret Name | Service | Reason |
|-------------|---------|--------|
| `prod/openrouter/api-key` | Secrets Manager | Sensitive API credential |
| `prod/langfuse/secret-key` | Secrets Manager | Sensitive API credential |
| `prod/slack/webhook-url` | Secrets Manager | Sensitive webhook |
| `/invoice-pipeline/prod/langfuse/public-key` | SSM Parameter Store | Non-sensitive public key |

---

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Sensitive credentials | Secrets Manager | KMS encryption, native rotation, $0.40/secret/month |
| Non-sensitive config | SSM Parameter Store | Free Standard tier, hierarchical paths |
| Lambda fetch strategy | Extension layer (preferred) | In-memory cache, reduces API calls and cold-start cost |
| KMS key | Default `aws/secretsmanager` | Sufficient for API keys; CMK only if compliance requires |
| IAM scope | Per-secret ARN | Least-privilege; never wildcard |

---

## Anti-Patterns (Never Do)

- Store secrets in Lambda environment variables (visible in AWS console)
- Fetch secrets on every Lambda invocation without caching
- Use broad `secretsmanager:*` permissions
- Store plaintext secret values in Terraform state
- Commit API keys or webhook URLs to source control

---

## Quick Reference

- [quick-reference.md](quick-reference.md) - Decision tables and code snippets at a glance
