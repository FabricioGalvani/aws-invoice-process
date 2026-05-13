# AWS EventBridge S3→SQS Routing Module

> **MCP Validated:** 2026-05-12
> **Purpose**: Reusable module that creates an EventBridge rule matching S3 Object Created events from a specific bucket and routes them to an SQS queue, including the required SQS queue policy

## When to Use

When a TIFF invoice is uploaded to `invoices-input-{env}`, the pipeline must fan the S3 event to the `invoice-uploaded` SQS queue so the `tiff-to-png-converter` Lambda picks it up. S3 event notifications support EventBridge natively (enabled via `aws_s3_bucket_notification` with `eventbridge = true` in the S3 module). This module creates the EventBridge rule that filters those events and the IAM-side SQS queue policy that permits EventBridge to enqueue messages. One module instance = one routing rule between one bucket and one queue.

## Module Structure

```hcl
# modules/eventbridge/variables.tf
variable "rule_name"         { type = string }
variable "description"       { type = string; default = "" }
variable "source_bucket_name" {
  type        = string
  description = "Name (not ARN) of the S3 bucket to match — used in event_pattern"
}
variable "target_queue_arn"  { type = string }
variable "target_queue_url"  { type = string }
variable "event_pattern_overrides" {
  type        = string
  default     = null
  description = "Full JSON event_pattern to override defaults; null uses the built-in S3 Object Created pattern"
}
variable "input_transformer_template" {
  type        = string
  default     = null
  description = "Optional JSONPath input transformer template string"
}
variable "retry_max_attempts" {
  type    = number
  default = 3
}
variable "tags" { type = map(string); default = {} }
```

```hcl
# modules/eventbridge/main.tf
locals {
  default_pattern = jsonencode({
    source      = ["aws.s3"]
    detail-type = ["Object Created"]
    detail = {
      bucket = { name = [var.source_bucket_name] }
    }
  })
  event_pattern = var.event_pattern_overrides != null ? var.event_pattern_overrides : local.default_pattern
}

resource "aws_cloudwatch_event_rule" "rule" {
  name          = var.rule_name
  description   = var.description
  event_pattern = local.event_pattern
  tags          = var.tags
}

resource "aws_cloudwatch_event_target" "sqs" {
  rule      = aws_cloudwatch_event_rule.rule.name
  target_id = "sqs-target"
  arn       = var.target_queue_arn

  retry_policy {
    maximum_retry_attempts       = var.retry_max_attempts
    maximum_event_age_in_seconds = 3600
  }

  dynamic "input_transformer" {
    for_each = var.input_transformer_template != null ? [1] : []
    content {
      input_paths = {
        bucket = "$.detail.bucket.name"
        key    = "$.detail.object.key"
        size   = "$.detail.object.size"
      }
      input_template = var.input_transformer_template
    }
  }
}

# SQS queue policy allows EventBridge to send messages
data "aws_iam_policy_document" "eb_to_sqs" {
  statement {
    sid     = "AllowEventBridgeSend"
    effect  = "Allow"
    actions = ["sqs:SendMessage"]
    principals {
      type        = "Service"
      identifiers = ["events.amazonaws.com"]
    }
    resources = [var.target_queue_arn]
    condition {
      test     = "ArnEquals"
      variable = "aws:SourceArn"
      values   = [aws_cloudwatch_event_rule.rule.arn]
    }
  }
}

resource "aws_sqs_queue_policy" "eb_to_sqs" {
  queue_url = var.target_queue_url
  policy    = data.aws_iam_policy_document.eb_to_sqs.json
}
```

```hcl
# modules/eventbridge/outputs.tf
output "rule_arn"  { value = aws_cloudwatch_event_rule.rule.arn }
output "rule_name" { value = aws_cloudwatch_event_rule.rule.name }
```

## Example Terragrunt Invocation (dev)

```hcl
# infrastructure/environments/dev/eventbridge-input-uploaded/terragrunt.hcl
terraform {
  source = "../../../../modules/eventbridge"
}

include "root" { path = find_in_parent_folders() }

dependency "sqs_uploaded" { config_path = "../sqs-invoice-uploaded" }

inputs = {
  rule_name           = "s3-input-to-uploaded-dev"
  description         = "Route invoices-input-dev Object Created events to invoice-uploaded-dev SQS"
  source_bucket_name  = "invoices-input-dev"
  target_queue_arn    = dependency.sqs_uploaded.outputs.queue_arn
  target_queue_url    = dependency.sqs_uploaded.outputs.queue_url
  retry_max_attempts  = 3

  # Optional: slim the SQS message to only bucket + key
  input_transformer_template = "{\"bucket\": \"<bucket>\", \"key\": \"<key>\", \"size\": \"<size>\"}"

  tags = { env = "dev", stage = "upload" }
}
```

## Inputs Reference

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `rule_name` | Yes | — | Unique EventBridge rule name |
| `source_bucket_name` | Yes | — | S3 bucket name to match in `detail.bucket.name` |
| `target_queue_arn` | Yes | — | ARN of the SQS target queue |
| `target_queue_url` | Yes | — | URL of the SQS target (for queue policy resource) |
| `description` | No | `""` | Human-readable rule description |
| `event_pattern_overrides` | No | `null` | Full JSON pattern override |
| `input_transformer_template` | No | `null` | Transform event shape before enqueuing |
| `retry_max_attempts` | No | `3` | EventBridge-side retry before dropping |
| `tags` | No | `{}` | AWS tags |

## Common Pitfalls

- **EventBridge not enabled on the bucket**: This module creates the rule but S3 will not emit events unless `aws_s3_bucket_notification` in the S3 module has `eventbridge = true`. The S3 module's `enable_eventbridge` variable controls this.
- **Missing SQS queue policy**: EventBridge cannot send to SQS without an explicit resource-based policy. The `aws_sqs_queue_policy` resource in this module is mandatory and scoped to the specific rule ARN via a `SourceArn` condition.
- **Default event bus only**: S3-sourced events always land on the default event bus. Do not specify a custom `event_bus_name` for this use case — it will never receive S3 events.
- **Event pattern for S3**: The `detail-type` must be `"Object Created"` (with a capital C and space) for EventBridge S3 integration. Do not use the legacy SNS-style `s3:ObjectCreated:*` syntax.
- **Applying before the S3 bucket**: Terraform/Terragrunt dependency ordering must ensure the S3 bucket and its notification resource exist before this rule is applied; otherwise the rule matches events that will never arrive.

## See Also

- [S3 Module](./s3-module.md) — must have `enable_eventbridge = true` on the source bucket
- [SQS Module](./sqs-module.md) — provides `queue_arn` and `queue_url` outputs consumed here
- [Lambda Module](./lambda-module.md) — Lambda function that polls the SQS queue as next stage
