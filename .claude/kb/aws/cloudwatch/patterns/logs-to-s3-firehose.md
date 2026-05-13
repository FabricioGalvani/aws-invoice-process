# Pattern: CloudWatch Logs → Kinesis Firehose → S3 (Full Setup)

> **MCP Validated:** 2026-05-12

## Purpose

Export Lambda structured logs to S3 for two consumers:
1. **CrewAI agents** (Triage, Root Cause, Reporter) read logs from S3 — required by
   CLAUDE.md rule 10 and notes/07-aws-migration-plan.md §2.8.
2. **Long-term retention** at S3/Glacier cost (fraction of CloudWatch storage cost).

See [../concepts/firehose-export.md](../concepts/firehose-export.md) for architecture
and IAM role policies.

## Complete Terraform (modules/firehose-log-export/main.tf)

### Step 1: S3 Destination Bucket

```hcl
resource "aws_s3_bucket" "logs" {
  bucket = "invoices-logs-${var.environment}-${data.aws_caller_identity.current.account_id}"
  force_destroy = var.environment == "dev" ? true : false

  tags = {
    Environment = var.environment
    Purpose     = "pipeline-logs-crewai"
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "logs" {
  bucket = aws_s3_bucket.logs.id

  rule {
    id     = "expire-after-365-days"
    status = "Enabled"
    filter { prefix = "cloudwatch-logs/" }

    transition {
      days          = 90
      storage_class = "GLACIER_IR"
    }

    expiration {
      days = 365
    }
  }
}
```

### Step 2: IAM Role — CloudWatch Logs → Firehose

```hcl
data "aws_iam_policy_document" "cw_trust" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["logs.${var.aws_region}.amazonaws.com"]
    }
    condition {
      test     = "ArnLike"
      variable = "aws:SourceArn"
      values   = ["arn:aws:logs:${var.aws_region}:${data.aws_caller_identity.current.account_id}:log-group:*"]
    }
  }
}

data "aws_iam_policy_document" "cw_to_firehose" {
  statement {
    effect    = "Allow"
    actions   = ["firehose:PutRecord", "firehose:PutRecordBatch"]
    resources = [aws_kinesis_firehose_delivery_stream.logs.arn]
  }
}

resource "aws_iam_role" "cw_to_firehose" {
  name               = "CloudWatchToFirehose-${var.environment}"
  assume_role_policy = data.aws_iam_policy_document.cw_trust.json
}

resource "aws_iam_role_policy" "cw_to_firehose" {
  role   = aws_iam_role.cw_to_firehose.id
  policy = data.aws_iam_policy_document.cw_to_firehose.json
}
```

### Step 3: IAM Role — Firehose → S3

```hcl
data "aws_iam_policy_document" "firehose_trust" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["firehose.amazonaws.com"]
    }
  }
}

data "aws_iam_policy_document" "firehose_to_s3" {
  statement {
    effect = "Allow"
    actions = [
      "s3:AbortMultipartUpload", "s3:GetBucketLocation",
      "s3:GetObject", "s3:ListBucket",
      "s3:ListBucketMultipartUploads", "s3:PutObject",
    ]
    resources = [
      aws_s3_bucket.logs.arn,
      "${aws_s3_bucket.logs.arn}/*",
    ]
  }
}

resource "aws_iam_role" "firehose_to_s3" {
  name               = "FirehoseToS3Logs-${var.environment}"
  assume_role_policy = data.aws_iam_policy_document.firehose_trust.json
}

resource "aws_iam_role_policy" "firehose_to_s3" {
  role   = aws_iam_role.firehose_to_s3.id
  policy = data.aws_iam_policy_document.firehose_to_s3.json
}
```

### Step 4: Kinesis Firehose Delivery Stream

```hcl
resource "aws_kinesis_firehose_delivery_stream" "logs" {
  name        = "invoice-pipeline-logs-${var.environment}"
  destination = "extended_s3"

  extended_s3_configuration {
    role_arn   = aws_iam_role.firehose_to_s3.arn
    bucket_arn = aws_s3_bucket.logs.arn

    prefix              = "cloudwatch-logs/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/"
    error_output_prefix = "cloudwatch-logs-errors/!{firehose:error-output-type}/year=!{timestamp:yyyy}/"

    buffering_size     = 5    # MB
    buffering_interval = 60   # seconds
    compression_format = "GZIP"
  }

  tags = {
    Environment = var.environment
  }
}
```

### Step 5: Subscription Filters (one per Lambda Log Group)

```hcl
locals {
  lambda_functions = [
    "tiff-to-png-converter",
    "invoice-classifier",
    "data-extractor",
    "warehouse-writer",
  ]
}

resource "aws_cloudwatch_log_subscription_filter" "to_firehose" {
  for_each = toset(local.lambda_functions)

  name            = "${each.key}-to-firehose-${var.environment}"
  log_group_name  = "/aws/lambda/${each.key}-${var.environment}"
  filter_pattern  = ""  # all events; use "[severity=ERROR*]" for errors-only
  destination_arn = aws_kinesis_firehose_delivery_stream.logs.arn
  role_arn        = aws_iam_role.cw_to_firehose.arn
  distribution    = "ByLogStream"

  depends_on = [aws_cloudwatch_log_group.lambda_functions]
}
```

## CrewAI Read Note

Firehose files are gzip-compressed. CrewAI agents must: `gzip.decompress(raw)` →
parse newline-delimited JSON. Add a Firehose transformation Lambda to pre-decode
if agents need clean JSON directly. Buffering: 60s (near-real-time) / 5MB.

## Anti-Patterns

| Anti-Pattern | Impact | Fix |
|---|---|---|
| No S3 lifecycle on logs bucket | Unbounded storage cost | Add Glacier transition at 90 days |
| Filter pattern `""` in prod with high volume | High Firehose cost | Filter to WARN+ in prod |
| No error prefix for Firehose failures | Lost data on delivery errors | Always set `error_output_prefix` |
| `distribution = "Random"` | Out-of-order logs in S3 | Use `ByLogStream` |

## See Also

- [../concepts/firehose-export.md](../concepts/firehose-export.md) — architecture
- [../concepts/logs-and-retention.md](../concepts/logs-and-retention.md) — retention
- [../specs/cloudwatch-iam-policy.json](../specs/cloudwatch-iam-policy.json)
