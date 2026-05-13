# Kinesis Data Firehose Log Export to S3

> **MCP Validated:** 2026-05-12

## Purpose

CloudWatch Logs → Kinesis Data Firehose → S3 serves two goals in this pipeline:

1. **CrewAI agent input**: The 3-agent system (Triage, Root Cause, Reporter) reads
   pipeline logs from S3 rather than CloudWatch Logs directly. Firehose delivers the
   continuous export (see CLAUDE.md rule 10 and notes/07-aws-migration-plan.md §2.8).
2. **Cost reduction**: Long-term log storage in S3 (+ Glacier lifecycle) is far cheaper
   than CloudWatch Logs at $0.03/GB/month. After 90 days, logs move to S3 at ~$0.023/GB.

## Architecture

```
Lambda stdout
    │
    ▼
CloudWatch Log Group (/aws/lambda/{fn}-{env})
    │
    │ [Subscription Filter]
    ▼
CloudWatch Logs service (logs.{region}.amazonaws.com)
    │  Role: CloudWatchLogsToFirehoseRole
    │  Action: firehose:PutRecord
    ▼
Kinesis Data Firehose Delivery Stream
    │  Buffering: 60s / 5MB (whichever first)
    │  Compression: GZIP
    │  Role: FirehoseToS3Role
    ▼
s3://invoices-logs-{env}/cloudwatch-logs/
    ├── year=2026/month=05/day=12/
    │   └── invoice-pipeline-1-2026-05-12-10-00-00-{uuid}.gz
    └── ...
```

## Data Format Note

Logs delivered by a subscription filter to Firehose are **base64-encoded** and
**gzip-compressed** before delivery. The Firehose stream adds an optional Lambda
transformation step to decode and reformat (e.g., to NDJSON) before writing to S3.

Without transformation, each S3 file contains raw gzip'd base64 CloudWatch log records.
CrewAI agents must decode: `base64 decode → gzip decompress → parse CloudWatch records`.

**Recommended**: add a Firehose transformation Lambda to output clean JSON per line.

## IAM Roles Required

### Role 1: CloudWatch Logs → Firehose (`CloudWatchLogsToFirehoseRole`)

```json
Trust policy principal: "logs.{region}.amazonaws.com"
Condition: ArnLike SourceArn = "arn:aws:logs:{region}:{account}:log-group:*"

Permission:
  firehose:PutRecord on the delivery stream ARN
```

### Role 2: Firehose → S3 (`FirehoseToS3Role`)

```json
Trust policy principal: "firehose.amazonaws.com"

Permissions:
  s3:AbortMultipartUpload
  s3:GetBucketLocation
  s3:GetObject
  s3:ListBucket
  s3:ListBucketMultipartUploads
  s3:PutObject
  on the target S3 bucket ARN and ARN/*
```

See [../specs/cloudwatch-iam-policy.json](../specs/cloudwatch-iam-policy.json) for the
least-privilege Lambda → CloudWatch policy.

## Subscription Filter

A subscription filter links a Log Group to the Firehose stream. Pattern `""` (empty)
matches all log events. You can filter to only forward `ERROR` logs by using a pattern.

```hcl
resource "aws_cloudwatch_log_subscription_filter" "to_firehose" {
  name            = "${var.function_name}-to-firehose"
  log_group_name  = aws_cloudwatch_log_group.lambda_fn.name
  filter_pattern  = ""  # empty = all events; use "ERROR" for errors-only
  destination_arn = aws_kinesis_firehose_delivery_stream.logs.arn
  role_arn        = aws_iam_role.cw_to_firehose.arn
  distribution    = "ByLogStream"
}
```

## Buffering Hints

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Buffer interval | 60 seconds | Low enough for near-real-time CrewAI reads |
| Buffer size | 5 MB | Below 128MB max; suits log volume at 3.5k invoices/month |
| Compression | GZIP | Reduces S3 storage cost by ~70% |

## S3 Prefix Pattern

```
s3://invoices-logs-{env}/cloudwatch-logs/{function-name}/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/
```

Hive-compatible partitioning enables Athena to query logs by date without scanning all files.

## Limits

| Limit | Value |
|-------|-------|
| Subscription filters per log group | 2 |
| Max Firehose buffer interval | 900 seconds |
| Max Firehose buffer size | 128 MB |
| Max record size (before base64) | 1 MB |

## Anti-Patterns

| Anti-Pattern | Impact | Fix |
|---|---|---|
| No transformation Lambda | CrewAI must decode base64+gzip | Add transformation or decode in agent |
| Buffer interval > 300s | CrewAI sees stale data | Use 60s |
| No S3 prefix partitioning | Full bucket scan for queries | Add date-based prefix |
| Sending all logs unfiltered to prod | Cost spike | Consider filtering to WARN+ in prod |

## See Also

- [logs-and-retention.md](logs-and-retention.md) — retention before Firehose
- [../patterns/logs-to-s3-firehose.md](../patterns/logs-to-s3-firehose.md) — full Terraform
- [../specs/cloudwatch-iam-policy.json](../specs/cloudwatch-iam-policy.json)
