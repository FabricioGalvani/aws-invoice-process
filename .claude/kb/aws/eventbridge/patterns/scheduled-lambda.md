> **MCP Validated:** 2026-05-12

# Pattern: EventBridge Scheduler → Lambda (CrewAI)

## Pattern Overview

Use **EventBridge Scheduler** (not a rule) to invoke a Lambda function on a
recurring or one-time schedule. This replaces Cloud Scheduler from the original
GCP design and is used to trigger the CrewAI autonomous ops pipeline.

**Flow:**

```
EventBridge Scheduler
    │ (cron or rate expression)
    ▼
Lambda: crewai-autonomous-ops (or Fargate task for longer runs)
    │
    ├── Triage Agent → reads S3 logs (CloudWatch → Firehose → S3)
    ├── Root Cause Agent
    └── Reporter Agent → Slack webhook
```

---

## Scheduler vs Rule: Why Scheduler Here

The CrewAI pipeline must run on a fixed schedule (e.g., daily at 06:00 UTC),
independent of whether any invoice was processed. EventBridge Scheduler is the
correct primitive:

- No event must occur to trigger it
- Supports flexible time windows to avoid thundering-herd at midnight
- Has its own built-in DLQ and retry separate from the event bus
- Supports one-time schedules for on-demand backfill runs

---

## Recurring Schedule (cron)

```hcl
resource "aws_scheduler_schedule" "crewai_daily" {
  name       = "crewai-daily-ops-${var.env}"
  group_name = "invoice-pipeline"

  schedule_expression          = "cron(0 6 * * ? *)"  # 06:00 UTC daily
  schedule_expression_timezone = "UTC"

  flexible_time_window {
    mode                      = "FLEXIBLE"
    maximum_window_in_minutes = 30  # fire any time in 06:00–06:30 window
  }

  target {
    arn      = aws_lambda_function.crewai_ops.arn
    role_arn = aws_iam_role.scheduler_invoke_crewai.arn

    input = jsonencode({
      mode        = "full-run"
      environment = var.env
      log_bucket  = "invoices-logs-${var.env}"
      log_prefix  = "cloudwatch-export/"
    })

    retry_policy {
      maximum_retry_attempts       = 3
      maximum_event_age_in_seconds = 3600  # 1 hour
    }

    dead_letter_config {
      arn = aws_sqs_queue.scheduler_dlq.arn
    }
  }
}
```

---

## One-Time Schedule (backfill / on-demand)

```hcl
resource "aws_scheduler_schedule" "crewai_onetime" {
  name = "crewai-backfill-${var.env}-${formatdate("YYYYMMDD", timestamp())}"

  schedule_expression = "at(2026-06-01T06:00:00)"  # ISO 8601 UTC

  flexible_time_window {
    mode = "OFF"  # fire exactly at the specified time
  }

  target {
    arn      = aws_lambda_function.crewai_ops.arn
    role_arn = aws_iam_role.scheduler_invoke_crewai.arn
    input    = jsonencode({ mode = "backfill", date = "2026-06-01" })
  }
}
```

---

## Rate Expression (simpler alternative to cron)

```hcl
schedule_expression = "rate(1 day)"    # every 24 h from creation time
schedule_expression = "rate(6 hours)"  # every 6 h
```

Rate expressions do not support a fixed start time. Use cron for time-of-day precision.

---

## IAM: Scheduler → Lambda

The Scheduler **requires an IAM execution role** (unlike EventBridge rules → SQS,
which use resource-based policies). The role must trust the Scheduler service
principal and have `lambda:InvokeFunction`.

```hcl
resource "aws_iam_role" "scheduler_invoke_crewai" {
  name = "scheduler-invoke-crewai-${var.env}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "scheduler.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy" "scheduler_invoke_crewai" {
  role = aws_iam_role.scheduler_invoke_crewai.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = "lambda:InvokeFunction"
      Resource = aws_lambda_function.crewai_ops.arn
    }]
  })
}
```

---

## Flexible Time Window Guidance

| Window | Use when |
|---|---|
| `OFF` | Exact time required (one-time, SLA-bound) |
| `FLEXIBLE` 15 min | Light smoothing, low concurrency |
| `FLEXIBLE` 30 min | Default for CrewAI daily ops |
| `FLEXIBLE` 4 h (max) | Batch jobs with wide tolerance |

Flexible windows distribute invocations randomly within the window, reducing
Lambda cold-start burst pressure when many schedules fire near the same time.

---

## Verification Checklist

- [ ] `schedule_expression_timezone` set to `"UTC"` explicitly
- [ ] `flexible_time_window.mode = "FLEXIBLE"` for daily CrewAI ops
- [ ] `target.role_arn` set (Scheduler requires IAM role — not optional)
- [ ] IAM role trust policy allows `scheduler.amazonaws.com`
- [ ] `retry_policy.maximum_retry_attempts` set (default is 185 — reduce for ops jobs)
- [ ] `dead_letter_config.arn` set to a standard SQS DLQ
- [ ] Lambda function has sufficient timeout for full CrewAI run

---

## References

- [EventBridge Scheduler docs](https://docs.aws.amazon.com/scheduler/latest/UserGuide/what-is-scheduler.html)
- [Invoke Lambda on schedule](https://docs.aws.amazon.com/lambda/latest/dg/with-eventbridge-scheduler.html)
- [Flexible time windows](https://docs.aws.amazon.com/scheduler/latest/UserGuide/managing-schedule-flexible-time-windows.html)
- [Introducing EventBridge Scheduler (AWS blog)](https://aws.amazon.com/blogs/compute/introducing-amazon-eventbridge-scheduler/)
