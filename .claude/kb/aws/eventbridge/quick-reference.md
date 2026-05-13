> **MCP Validated:** 2026-05-12

# EventBridge Quick Reference

## Enable S3 → EventBridge (one setting)

```hcl
resource "aws_s3_bucket_notification" "invoices_input" {
  bucket      = aws_s3_bucket.invoices_input.id
  eventbridge = true
}
```

## Event Pattern for invoices-input-* TIFF uploads

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": {
    "bucket": { "name": [{ "prefix": "invoices-input-" }] },
    "object": { "key":  [{ "suffix": ".tiff" }] }
  }
}
```

## Key JSONPath variables for input transformer

| Variable | JSONPath |
|---|---|
| `bucket` | `$.detail.bucket.name` |
| `key` | `$.detail.object.key` |
| `size` | `$.detail.object.size` |
| `eventId` | `$.id` |
| `time` | `$.time` |

## SQS queue policy (required for EventBridge → SQS)

```json
{
  "Principal": { "Service": "events.amazonaws.com" },
  "Action": "sqs:SendMessage",
  "Condition": { "ArnEquals": { "aws:SourceArn": "<rule-arn>" } }
}
```

## Two DLQs — do not confuse

| DLQ | Config location | Catches |
|---|---|---|
| Rule-target DLQ | `aws_cloudwatch_event_target.dead_letter_config` | EventBridge cannot deliver to SQS |
| SQS DLQ | `aws_sqs_queue.redrive_policy` | Lambda cannot process message |

## Retry policy defaults (rules)

- Max retry attempts: **185**
- Max event age: **86400 s (24 h)**
- Strategy: exponential backoff with jitter

## EventBridge Scheduler schedule expressions

| Type | Expression | When |
|---|---|---|
| Cron | `cron(0 6 * * ? *)` | 06:00 UTC daily |
| Rate | `rate(1 day)` | Every 24 h from creation |
| One-time | `at(2026-06-01T06:00:00)` | Exactly once |

## Scheduler vs Rule — decision

| Trigger | Use |
|---|---|
| S3 upload / AWS service event | EventBridge Rule |
| Time-based / periodic | EventBridge Scheduler |
| Fan-out to multiple targets | EventBridge Rule (up to 5 targets) |

## IAM model differences

| Primitive | Auth mechanism |
|---|---|
| Rule → SQS | SQS resource-based policy (`events.amazonaws.com`) |
| Rule → Lambda | Lambda resource-based policy |
| Scheduler → Lambda | IAM execution role on the schedule |
## Terraform resources

| Resource | Purpose |
|---|---|
| `aws_cloudwatch_event_rule` | Rule + event pattern |
| `aws_cloudwatch_event_target` | Target binding |
| `aws_sqs_queue_policy` | Resource policy on SQS |
| `aws_s3_bucket_notification` | Enable EventBridge on S3 |
| `aws_scheduler_schedule` | Scheduler schedule |

## Anti-patterns at a glance

| Never do | Why |
|---|---|
| S3 → Lambda direct trigger for fan-out | Cannot filter or add DLQ at rule level |
| Missing `dead_letter_config` on rule target | Silent event loss |
| Overly broad pattern (no bucket/key filter) | Triggers on all S3 events in account |
| Skipping `aws:SourceArn` condition on SQS policy | Confused-deputy vulnerability |
