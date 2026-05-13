# AWS S3 Invoice Bucket Terraform Module

> **MCP Validated:** 2026-05-12
> **Purpose**: Reusable module for S3 buckets with versioning, KMS/SSE-S3 encryption, lifecycle rules, EventBridge notification, and full public-access block

## When to Use

The invoice pipeline uses four S3 buckets (`invoices-input`, `invoices-processed`, `invoices-archive`, `invoices-failed`), each with different retention, encryption, and event routing requirements. This module standardises the security baseline — versioning on, public access blocked, encryption enforced — while letting each bucket override lifecycle and EventBridge notification independently. The `input` bucket must enable EventBridge so that S3 events flow to the EventBridge → SQS chain.

## Module Structure

```hcl
# modules/s3/variables.tf
variable "bucket_name"        { type = string }
variable "versioning_enabled" { type = bool; default = true }
variable "enable_eventbridge" {
  type        = bool
  default     = false
  description = "Route all S3 events to EventBridge (required for the input bucket)"
}
variable "kms_key_arn" {
  type        = string
  default     = null
  description = "CMK ARN for SSE-KMS; null falls back to SSE-S3 (AES256)"
}
variable "lifecycle_rules" {
  type = list(object({
    id                       = string
    prefix                   = optional(string, "")
    transition_days          = optional(number, null)
    transition_storage_class = optional(string, null)
    expiration_days          = optional(number, null)
  }))
  default = []
}
variable "force_destroy" { type = bool; default = false }
variable "tags"          { type = map(string); default = {} }
```

```hcl
# modules/s3/main.tf
resource "aws_s3_bucket" "bucket" {
  bucket        = var.bucket_name
  force_destroy = var.force_destroy
  tags          = var.tags
}

resource "aws_s3_bucket_public_access_block" "block" {
  bucket                  = aws_s3_bucket.bucket.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_versioning" "versioning" {
  bucket = aws_s3_bucket.bucket.id
  versioning_configuration {
    status = var.versioning_enabled ? "Enabled" : "Suspended"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "sse" {
  bucket = aws_s3_bucket.bucket.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = var.kms_key_arn != null ? "aws:kms" : "AES256"
      kms_master_key_id = var.kms_key_arn
    }
    bucket_key_enabled = var.kms_key_arn != null
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "lifecycle" {
  count  = length(var.lifecycle_rules) > 0 ? 1 : 0
  bucket = aws_s3_bucket.bucket.id

  dynamic "rule" {
    for_each = var.lifecycle_rules
    content {
      id     = rule.value.id
      status = "Enabled"
      filter { prefix = rule.value.prefix }

      dynamic "transition" {
        for_each = rule.value.transition_days != null ? [1] : []
        content {
          days          = rule.value.transition_days
          storage_class = rule.value.transition_storage_class
        }
      }

      dynamic "expiration" {
        for_each = rule.value.expiration_days != null ? [1] : []
        content { days = rule.value.expiration_days }
      }
    }
  }
}

resource "aws_s3_bucket_notification" "notification" {
  bucket      = aws_s3_bucket.bucket.id
  eventbridge = var.enable_eventbridge
}
```

```hcl
# modules/s3/outputs.tf
output "bucket_id"  { value = aws_s3_bucket.bucket.id }
output "bucket_arn" { value = aws_s3_bucket.bucket.arn }
output "bucket_name" { value = aws_s3_bucket.bucket.bucket }
```

## Example Terragrunt Invocations — All 4 Project Buckets (dev)

```hcl
# invoices-input-dev  (EventBridge enabled, 30-day lifecycle)
inputs = {
  bucket_name        = "invoices-input-dev"
  enable_eventbridge = true
  versioning_enabled = true
  lifecycle_rules = [{
    id              = "expire-raw"
    expiration_days = 30
  }]
  tags = { env = "dev", bucket = "input" }
}

# invoices-processed-dev  (no EventBridge, 90-day lifecycle)
inputs = {
  bucket_name        = "invoices-processed-dev"
  enable_eventbridge = false
  versioning_enabled = true
  lifecycle_rules = [{
    id              = "expire-processed"
    expiration_days = 90
  }]
  tags = { env = "dev", bucket = "processed" }
}

# invoices-archive-dev  (Glacier after 30 days, 7-year retention)
inputs = {
  bucket_name        = "invoices-archive-dev"
  enable_eventbridge = false
  versioning_enabled = true
  lifecycle_rules = [
    { id = "to-glacier", transition_days = 30, transition_storage_class = "GLACIER" },
    { id = "expire-after-7y", expiration_days = 2555 }
  ]
  tags = { env = "dev", bucket = "archive" }
}

# invoices-failed-dev  (manual review, no expiration rule)
inputs = {
  bucket_name        = "invoices-failed-dev"
  enable_eventbridge = false
  versioning_enabled = false
  lifecycle_rules    = []
  tags = { env = "dev", bucket = "failed" }
}
```

## Inputs Reference

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `bucket_name` | Yes | — | Globally unique bucket name |
| `versioning_enabled` | No | `true` | Enable S3 object versioning |
| `enable_eventbridge` | No | `false` | Route all events to EventBridge |
| `kms_key_arn` | No | `null` | SSE-KMS CMK; null = SSE-S3 |
| `lifecycle_rules` | No | `[]` | List of transition/expiration rules |
| `force_destroy` | No | `false` | Allow destroy with objects present |
| `tags` | No | `{}` | AWS resource tags |

## Common Pitfalls

- **`enable_eventbridge = true` on processed/archive buckets**: This floods EventBridge with irrelevant events. Only the `input` bucket needs it; the Lambda functions communicate via SQS directly.
- **Notification resource overwrites**: `aws_s3_bucket_notification` is authoritative for a bucket. Do not define it in two places or one will silently win. All notification config must live inside this module call.
- **Missing `aws_s3_bucket_public_access_block`**: S3 buckets in provider v5 do not block public access by default at the resource level. The block resource is mandatory for compliance.
- **Lifecycle and versioning order**: Apply versioning before lifecycle rules; non-current version expiration requires versioning enabled.
- **KMS key policy**: If you supply a `kms_key_arn`, the Lambda execution roles must also have `kms:GenerateDataKey` and `kms:Decrypt` on that key — not just `s3:PutObject`.

## See Also

- [EventBridge Module](./eventbridge-module.md) — consumes `bucket_arn` to filter S3 events
- [Athena Iceberg Module](./athena-iceberg-module.md) — warehouse bucket separate from these four
- [Lambda Module](./lambda-module.md) — functions that read/write these buckets
