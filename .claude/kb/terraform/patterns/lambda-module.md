# AWS Lambda Terraform Module

> **MCP Validated:** 2026-05-12
> **Purpose**: Reusable module for deploying AWS Lambda functions with CloudWatch log group, DLQ event invoke config, and optional VPC/alias support

## When to Use

The invoice pipeline runs four discrete Lambda functions (`tiff-to-png-converter`, `invoice-classifier`, `data-extractor`, `warehouse-writer`), each with distinct memory, timeout, and permission requirements. This module standardises function creation, log retention, async retry policy, and DLQ wiring so every function follows the same operational baseline without duplicating HCL across four invocations.

## Module Structure

```hcl
# modules/lambda/variables.tf
variable "function_name"       { type = string }
variable "handler"             { type = string }
variable "runtime"             { type = string; default = "python3.12" }
variable "memory_mb"           { type = number; default = 512 }
variable "timeout_s"           { type = number; default = 60 }
variable "architectures"       { type = list(string); default = ["arm64"] }
variable "s3_bucket"           { type = string; description = "S3 bucket holding the deployment package" }
variable "s3_key"              { type = string; description = "S3 key of the deployment .zip" }
variable "env_vars"            { type = map(string); default = {} }
variable "log_retention_days"  { type = number; default = 30 }
variable "role_arn"            { type = string; description = "IAM role ARN (created outside module, least-privilege per function)" }
variable "dlq_arn"             { type = string; default = null; description = "SQS DLQ ARN for async invocation failures" }
variable "max_retry_attempts"  { type = number; default = 2 }
variable "layers"              { type = list(string); default = [] }
variable "subnet_ids"          { type = list(string); default = [] }
variable "security_group_ids"  { type = list(string); default = [] }
variable "enable_alias"        { type = bool; default = false; description = "Create 'live' alias for blue/green" }
variable "tags"                { type = map(string); default = {} }
```

```hcl
# modules/lambda/main.tf
locals {
  vpc_enabled = length(var.subnet_ids) > 0
}

resource "aws_cloudwatch_log_group" "fn" {
  name              = "/aws/lambda/${var.function_name}"
  retention_in_days = var.log_retention_days
  tags              = var.tags
}

resource "aws_lambda_function" "fn" {
  function_name = var.function_name
  handler       = var.handler
  runtime       = var.runtime
  role          = var.role_arn
  architectures = var.architectures
  memory_size   = var.memory_mb
  timeout       = var.timeout_s
  layers        = var.layers

  s3_bucket = var.s3_bucket
  s3_key    = var.s3_key

  environment {
    variables = var.env_vars
  }

  dynamic "vpc_config" {
    for_each = local.vpc_enabled ? [1] : []
    content {
      subnet_ids         = var.subnet_ids
      security_group_ids = var.security_group_ids
    }
  }

  logging_config {
    log_format = "JSON"
    log_group  = aws_cloudwatch_log_group.fn.name
  }

  depends_on = [aws_cloudwatch_log_group.fn]
  tags       = var.tags
}

resource "aws_lambda_function_event_invoke_config" "async" {
  function_name                = aws_lambda_function.fn.function_name
  maximum_retry_attempts       = var.max_retry_attempts
  maximum_event_age_in_seconds = 3600

  dynamic "destination_config" {
    for_each = var.dlq_arn != null ? [1] : []
    content {
      on_failure {
        destination = var.dlq_arn
      }
    }
  }
}

resource "aws_lambda_alias" "live" {
  count            = var.enable_alias ? 1 : 0
  name             = "live"
  function_name    = aws_lambda_function.fn.function_name
  function_version = aws_lambda_function.fn.version
}
```

```hcl
# modules/lambda/outputs.tf
output "function_arn"   { value = aws_lambda_function.fn.arn }
output "function_name"  { value = aws_lambda_function.fn.function_name }
output "role_arn"       { value = aws_lambda_function.fn.role }
output "log_group_name" { value = aws_cloudwatch_log_group.fn.name }
output "alias_arn"      { value = var.enable_alias ? aws_lambda_alias.live[0].arn : null }
```

## Example Terragrunt Invocation (dev)

```hcl
# infrastructure/environments/dev/lambda-data-extractor/terragrunt.hcl
terraform {
  source = "../../../../modules/lambda"
}

include "root" { path = find_in_parent_folders() }

inputs = {
  function_name      = "data-extractor-dev"
  handler            = "src.functions.data_extractor.handler"
  runtime            = "python3.12"
  memory_mb          = 1024
  timeout_s          = 120
  architectures      = ["arm64"]
  s3_bucket          = "invoices-deploy-artifacts-dev"
  s3_key             = "data-extractor/latest.zip"
  log_retention_days = 30
  role_arn           = dependency.iam.outputs.data_extractor_role_arn
  dlq_arn            = dependency.sqs.outputs.classified_dlq_arn
  max_retry_attempts = 2
  env_vars = {
    ENVIRONMENT     = "dev"
    LOG_LEVEL       = "INFO"
    BEDROCK_MODEL   = "anthropic.claude-haiku-4-5"
    BEDROCK_REGION  = "us-east-1"
  }
  tags = { environment = "dev", function = "data-extractor" }
}
```

## Inputs Reference

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `function_name` | Yes | — | Unique Lambda function name |
| `handler` | Yes | — | Module path to handler (e.g. `src.handler.main`) |
| `runtime` | No | `python3.12` | Lambda runtime identifier |
| `memory_mb` | No | `512` | Memory in MB (128–10240) |
| `timeout_s` | No | `60` | Timeout in seconds (1–900) |
| `architectures` | No | `["arm64"]` | `arm64` ~20% cheaper than `x86_64` |
| `s3_bucket` | Yes | — | Bucket containing deployment zip |
| `s3_key` | Yes | — | Key of the deployment zip |
| `role_arn` | Yes | — | Pre-created least-privilege IAM role |
| `dlq_arn` | No | `null` | SQS DLQ for async failure routing |
| `log_retention_days` | No | `30` | CloudWatch log retention |
| `layers` | No | `[]` | Lambda Layer ARNs |
| `subnet_ids` | No | `[]` | VPC subnets (omit for public) |
| `security_group_ids` | No | `[]` | VPC security groups |
| `enable_alias` | No | `false` | Create `live` alias for blue/green |

## Common Pitfalls

- **Log group not pre-created**: If Lambda creates the log group automatically, Terraform loses retention control and destroy does not clean it up. Always let the module create `aws_cloudwatch_log_group` first and use `depends_on`.
- **IAM role outside the module**: The module accepts a `role_arn` rather than creating the role. This enforces the project rule of one dedicated IAM role per function (see `notes/07-aws-migration-plan.md` §4.5).
- **timeout_s and SQS visibility mismatch**: The consuming SQS queue's `visibility_timeout_seconds` must be at least 6x this value. Wire them via Terragrunt `dependency` blocks, not hardcoded values.
- **`archive_file` vs S3 artifact**: For local iteration, use a `data "archive_file"` source and `filename`/`source_code_hash`. For CI/CD, build via SAM CLI (`sam build`) and upload to S3; the module's `s3_bucket`/`s3_key` inputs cover this path.
- **arm64 layer compatibility**: Layers must be compiled for `arm64` when `architectures = ["arm64"]`. Mixing architectures silently fails at runtime.

## See Also

- [SQS Module](./sqs-module.md) — DLQ ARN wired via `dependency`
- [EventBridge Module](./eventbridge-module.md) — S3 events that trigger Lambda consumers
- [Bedrock Module](./bedrock-module.md) — IAM policy attached to `data-extractor` role
