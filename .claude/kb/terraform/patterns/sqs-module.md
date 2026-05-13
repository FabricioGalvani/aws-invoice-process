# AWS SQS Queue + DLQ Terraform Module

> **MCP Validated:** 2026-05-12
> **Purpose**: Reusable module for an SQS queue / DLQ pair with redrive policy, redrive-allow policy, and a CloudWatch alarm that fires when the DLQ is non-empty

## When to Use

The invoice pipeline requires four SQS queues — `invoice-uploaded`, `invoice-converted`, `invoice-classified`, `invoice-extracted` — each paired with a DLQ. This module provisions both queues, wires the redrive relationship with separate `aws_sqs_queue_redrive_policy` and `aws_sqs_queue_redrive_allow_policy` resources (avoiding the circular dependency that arises when using inline `redrive_policy` on the source queue), and attaches a CloudWatch alarm so an on-call engineer is notified immediately when any message lands in the DLQ.

## Module Structure

```hcl
# modules/sqs/variables.tf
variable "queue_name" { type = string }
variable "visibility_timeout_seconds" {
  type        = number
  description = "Must be >= 6x the consumer Lambda timeout_s"
}
variable "message_retention_seconds" {
  type    = number
  default = 345600  # 4 days
}
variable "max_receive_count" {
  type        = number
  default     = 5
  description = "Deliveries before message moves to DLQ"
}
variable "receive_wait_time_seconds" {
  type    = number
  default = 20  # long polling
}
variable "alarm_actions" {
  type        = list(string)
  default     = []
  description = "SNS topic ARNs to notify on DLQ depth > 0"
}
variable "tags" { type = map(string); default = {} }
```

```hcl
# modules/sqs/main.tf
resource "aws_sqs_queue" "dlq" {
  name                       = "${var.queue_name}-dlq"
  message_retention_seconds  = 1209600  # 14 days for DLQ
  receive_wait_time_seconds  = var.receive_wait_time_seconds
  tags                       = var.tags
}

resource "aws_sqs_queue" "main" {
  name                       = var.queue_name
  visibility_timeout_seconds = var.visibility_timeout_seconds
  message_retention_seconds  = var.message_retention_seconds
  receive_wait_time_seconds  = var.receive_wait_time_seconds
  tags                       = var.tags
}

# Separate resource avoids circular dependency with inline redrive_policy
resource "aws_sqs_queue_redrive_policy" "main" {
  queue_url = aws_sqs_queue.main.id
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.dlq.arn
    maxReceiveCount     = var.max_receive_count
  })
}

resource "aws_sqs_queue_redrive_allow_policy" "dlq" {
  queue_url = aws_sqs_queue.dlq.id
  redrive_allow_policy = jsonencode({
    redrivePermission  = "byQueue"
    sourceQueueArns    = [aws_sqs_queue.main.arn]
  })
}

resource "aws_cloudwatch_metric_alarm" "dlq_depth" {
  alarm_name          = "${var.queue_name}-dlq-not-empty"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "ApproximateNumberOfMessagesVisible"
  namespace           = "AWS/SQS"
  period              = 60
  statistic           = "Sum"
  threshold           = 0
  alarm_description   = "DLQ ${var.queue_name}-dlq has messages — check failed invoices"
  alarm_actions       = var.alarm_actions

  dimensions = {
    QueueName = aws_sqs_queue.dlq.name
  }
}
```

```hcl
# modules/sqs/outputs.tf
output "queue_arn"  { value = aws_sqs_queue.main.arn }
output "queue_url"  { value = aws_sqs_queue.main.url }
output "dlq_arn"    { value = aws_sqs_queue.dlq.arn }
output "dlq_url"    { value = aws_sqs_queue.dlq.url }
output "queue_name" { value = aws_sqs_queue.main.name }
output "dlq_name"   { value = aws_sqs_queue.dlq.name }
```

## Example Terragrunt Invocation (dev) — invoice-uploaded queue

```hcl
# infrastructure/environments/dev/sqs-invoice-uploaded/terragrunt.hcl
terraform {
  source = "../../../../modules/sqs"
}

include "root" { path = find_in_parent_folders() }

dependency "sns_alerts" { config_path = "../sns-alerts" }

inputs = {
  queue_name = "invoice-uploaded-dev"

  # tiff-to-png Lambda timeout = 300s → visibility must be >= 1800s
  visibility_timeout_seconds = 1800
  message_retention_seconds  = 345600   # 4 days
  max_receive_count          = 5
  receive_wait_time_seconds  = 20

  alarm_actions = [dependency.sns_alerts.outputs.topic_arn]
  tags = { env = "dev", stage = "upload" }
}
```

## Inputs Reference

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `queue_name` | Yes | — | Base name; DLQ gets `-dlq` suffix |
| `visibility_timeout_seconds` | Yes | — | Must be >= 6x consumer Lambda timeout |
| `message_retention_seconds` | No | `345600` (4d) | Main queue retention |
| `max_receive_count` | No | `5` | Before redrive to DLQ |
| `receive_wait_time_seconds` | No | `20` | Long polling (20 = max) |
| `alarm_actions` | No | `[]` | SNS ARNs for DLQ alarm |
| `tags` | No | `{}` | AWS tags |

## Common Pitfalls

- **`visibility_timeout_seconds` too short**: If the Lambda takes 300 s but visibility is 300 s, the message becomes visible again mid-processing and a second Lambda picks it up concurrently. Use 6x the Lambda timeout as a floor.
- **Inline `redrive_policy` creates a cycle**: Defining `redrive_policy` and `redrive_allow_policy` inline on the same queue pair creates a Terraform dependency cycle. Use the separate `aws_sqs_queue_redrive_policy` and `aws_sqs_queue_redrive_allow_policy` resources as shown above.
- **DLQ retention shorter than main queue**: The DLQ defaults to 14 days in this module. Never set it lower than the main queue — otherwise messages expire in the DLQ before engineers can investigate.
- **Missing CloudWatch alarm**: A DLQ with no alarm is an invisible sink. The alarm in this module uses `threshold = 0` so the first failed message pages immediately.
- **FIFO queues**: This module provisions standard queues. If you need strict ordering or exactly-once delivery, add `fifo_queue = true` and append `.fifo` to `queue_name`. The pipeline does not require FIFO.

## See Also

- [Lambda Module](./lambda-module.md) — `dlq_arn` output wired to `aws_lambda_function_event_invoke_config`
- [EventBridge Module](./eventbridge-module.md) — `queue_arn` is the EventBridge rule target
