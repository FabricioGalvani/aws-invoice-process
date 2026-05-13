# Create Secrets Manager Secret Without Value in Terraform

> **Purpose**: Provision the secret container and metadata in Terraform while setting the actual secret value out-of-band to avoid plaintext exposure in Terraform state
> **MCP Validated:** 2026-05-12

## When to Use

- Any time a Secrets Manager secret holds a sensitive credential that must never appear in Terraform state files
- CI/CD pipelines where the secret value is injected separately (AWS CLI, manual console entry, or a secrets bootstrapping tool)
- All three project secrets: OpenRouter API key, LangFuse secret key, Slack webhook URL

## Why This Matters

Terraform state files (`.tfstate`) are stored in S3 and contain all resource attributes. If you use `aws_secretsmanager_secret_version` with a literal `secret_string`, that value appears in plaintext in state. The correct pattern separates secret container creation (Terraform) from secret value injection (out-of-band).

## Implementation

```hcl
# infrastructure/modules/secrets/main.tf

# 1. Create the secret container — Terraform manages metadata only
resource "aws_secretsmanager_secret" "openrouter_api_key" {
  name        = "${var.env}/openrouter/api-key"
  description = "OpenRouter API key for LLM fallback calls"
  kms_key_id  = "aws/secretsmanager"  # Default CMK; swap for CMK ARN if needed

  recovery_window_in_days = 7  # 0 = immediate delete (use in dev only)

  tags = {
    Environment = var.env
    Project     = "invoice-pipeline"
    ManagedBy   = "terraform"
  }
}

resource "aws_secretsmanager_secret" "langfuse_secret_key" {
  name        = "${var.env}/langfuse/secret-key"
  description = "LangFuse secret key for LLMOps tracking"
  kms_key_id  = "aws/secretsmanager"
  recovery_window_in_days = 7
  tags = local.common_tags
}

resource "aws_secretsmanager_secret" "slack_webhook_url" {
  name        = "${var.env}/slack/webhook-url"
  description = "Slack webhook URL for pipeline alerts"
  kms_key_id  = "aws/secretsmanager"
  recovery_window_in_days = 7
  tags = local.common_tags
}

# 2. SSM Parameter (non-sensitive — value IS managed by Terraform)
resource "aws_ssm_parameter" "langfuse_public_key" {
  name  = "/invoice-pipeline/${var.env}/langfuse/public-key"
  type  = "String"    # Non-sensitive; no KMS needed
  value = var.langfuse_public_key  # Set via CI/CD variable, NOT hardcoded

  tags = local.common_tags
}

# 3. Output the secret ARNs (not values) for use by Lambda IAM policies
output "openrouter_secret_arn" {
  value = aws_secretsmanager_secret.openrouter_api_key.arn
}

output "langfuse_secret_arn" {
  value = aws_secretsmanager_secret.langfuse_secret_key.arn
}

output "slack_secret_arn" {
  value = aws_secretsmanager_secret.slack_webhook_url.arn
}
```

## Set Value Out-of-Band (After Terraform Apply)

```bash
# Run AFTER terraform apply — secret container exists, now inject value
# This command does NOT appear in Terraform state

aws secretsmanager put-secret-value \
  --secret-id "prod/openrouter/api-key" \
  --secret-string '{"api_key": "sk-or-..."}' \
  --region us-east-1

aws secretsmanager put-secret-value \
  --secret-id "prod/langfuse/secret-key" \
  --secret-string '{"secret_key": "lf-sk-..."}' \
  --region us-east-1

aws secretsmanager put-secret-value \
  --secret-id "prod/slack/webhook-url" \
  --secret-string '{"webhook_url": "https://hooks.slack.com/..."}' \
  --region us-east-1
```

In CI/CD (GitHub Actions), store the values as GitHub Secrets and inject via `aws secretsmanager put-secret-value` in a bootstrap workflow.

## Anti-Pattern — Never Do This

```hcl
# WRONG: secret value in Terraform state
resource "aws_secretsmanager_secret_version" "openrouter" {
  secret_id     = aws_secretsmanager_secret.openrouter_api_key.id
  secret_string = "sk-or-my-actual-key"  # Appears in .tfstate in plaintext
}
```

## Configuration

| Variable | Description | Example |
|----------|-------------|---------|
| `var.env` | Deployment environment | `dev` or `prod` |
| `var.langfuse_public_key` | Non-sensitive LangFuse public key | `pk-lf-abc123` |
| `local.common_tags` | Standard resource tags | `{Environment, Project, ManagedBy}` |

## Lifecycle for `aws_secretsmanager_secret_version` (If Used)

If you must create an initial placeholder version via Terraform (e.g., to satisfy downstream dependencies), use `lifecycle.ignore_changes` to prevent Terraform from overwriting out-of-band updates:

```hcl
resource "aws_secretsmanager_secret_version" "openrouter_placeholder" {
  secret_id     = aws_secretsmanager_secret.openrouter_api_key.id
  secret_string = jsonencode({ api_key = "PLACEHOLDER_SET_OUT_OF_BAND" })

  lifecycle {
    ignore_changes = [secret_string]  # Terraform will not touch the value after creation
  }
}
```

## See Also

- [../concepts/naming-and-hierarchy.md](../concepts/naming-and-hierarchy.md) — secret path naming rules
- [../concepts/rotation-strategies.md](../concepts/rotation-strategies.md) — managing secret versions after rotation
- [../specs/secrets-iam-policy.json](../specs/secrets-iam-policy.json) — IAM permissions for the secrets
- [fetch-secret-cached.md](fetch-secret-cached.md) — how Lambdas consume these secrets
