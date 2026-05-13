# Rotation Strategies

> **Purpose**: When and how to rotate secrets; monitoring rotation health
> **Confidence**: 0.95
> **MCP Validated:** 2026-05-12

## Overview

AWS Secrets Manager supports two rotation models: native managed rotation (for AWS-native database credentials) and custom Lambda-backed rotation (for API keys and other third-party credentials). The project's three Secrets Manager secrets (OpenRouter, LangFuse, Slack) are third-party API keys and require the custom Lambda approach. Monitoring rotation freshness via CloudWatch alarms on `LastRotationDate` is a mandatory security control.

---

## Rotation Models

| Model | Supported Secrets | Setup | Project Applicability |
|-------|------------------|-------|----------------------|
| **Managed rotation** | Amazon RDS, Redshift, DocumentDB, ElastiCache | Zero-code; AWS manages the Lambda | Not applicable (no DB secrets) |
| **Custom Lambda rotation** | Any secret (API keys, webhooks, tokens) | Deploy a rotation Lambda; configure rotation schedule | Use for OpenRouter, LangFuse, Slack when vendors support key rollover |
| **Manual (no rotation)** | Static credentials with external lifecycle | Human process; monitor staleness via alarm | Initial state for this project |

---

## Native Rotation (DB Credentials — Reference Only)

For projects adding RDS credentials later, enable managed rotation with a single CLI call:

```bash
aws secretsmanager rotate-secret \
  --secret-id prod/rds/master-password \
  --rotation-rules AutomaticallyAfterDays=30
```

AWS creates and manages the rotation Lambda automatically. No custom code needed.

---

## Custom Lambda Rotation (API Keys)

API keys from third-party vendors (OpenRouter, LangFuse) require:

1. Vendor must support issuing a new key while the old one remains valid (dual-key window)
2. Deploy a rotation Lambda implementing the four-step rotation lifecycle:
   - `createSecret` — generate and store new key candidate
   - `setSecret` — activate the new key with the vendor
   - `testSecret` — verify the new key works
   - `finishSecret` — mark new key as AWSCURRENT; old key as AWSPREVIOUS

```python
# rotation_lambda/handler.py — skeleton
def lambda_handler(event, context):
    step = event["Step"]
    secret_id = event["SecretId"]
    token = event["ClientRequestToken"]

    if step == "createSecret":
        create_secret(secret_id, token)
    elif step == "setSecret":
        set_secret(secret_id, token)
    elif step == "testSecret":
        test_secret(secret_id, token)
    elif step == "finishSecret":
        finish_secret(secret_id, token)
```

---

## Monitoring Rotation Health

Set a CloudWatch alarm on the `LastRotationDate` metadata. If the secret has not been rotated within the expected window, the alarm fires and notifies via SNS → Slack.

### CloudWatch Metric Math Alarm (Terraform)

```hcl
resource "aws_cloudwatch_metric_alarm" "secret_rotation_stale" {
  alarm_name          = "secret-rotation-stale-prod-openrouter"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  threshold           = 0

  metric_query {
    id          = "days_since_rotation"
    expression  = "DATEDIFF(NOW(), last_rotation_timestamp)"
    label       = "Days Since Last Rotation"
    return_data = true
  }

  alarm_actions = [aws_sns_topic.security_alerts.arn]
}
```

In practice, use EventBridge rule to trigger a Lambda that checks `describe_secret()["LastRotatedDate"]` and publishes a custom CloudWatch metric, then alarm on that metric.

---

## Current Project Posture (Initial State)

| Secret | Rotation Strategy | Frequency | Alarm Threshold |
|--------|------------------|-----------|-----------------|
| `prod/openrouter/api-key` | Manual (no vendor rotation API yet) | On compromise or quarterly | 90 days |
| `prod/langfuse/secret-key` | Manual | On compromise or quarterly | 90 days |
| `prod/slack/webhook-url` | Manual (Slack supports regenerating webhook) | On compromise | 180 days |

---

## KMS Key Selection

| Option | When to Use |
|--------|-------------|
| `aws/secretsmanager` (default) | Sufficient for API keys; no extra cost |
| Customer Managed Key (CMK) | Required if compliance mandates key rotation control or cross-account access to the key policy |

The project uses the default `aws/secretsmanager` key. CMK is not required unless a compliance framework (PCI-DSS, HIPAA) mandates it.

---

## Related

- [secrets-vs-ssm.md](secrets-vs-ssm.md) — why these secrets live in Secrets Manager
- [../patterns/fetch-secret-cached.md](../patterns/fetch-secret-cached.md) — reduce TTL after rotation
- [../specs/secrets-iam-policy.json](../specs/secrets-iam-policy.json) — rotation Lambda IAM permissions
