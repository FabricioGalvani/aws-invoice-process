> **MCP Validated:** 2026-05-12

# Pattern: DLQ + CloudWatch Alarm + Slack Alert

## Overview

Every DLQ in the pipeline must have a CloudWatch Alarm on `ApproximateNumberOfMessagesVisible > 0`. The alarm fans out to an SNS topic, which delivers to a Slack webhook via Lambda or a Chatbot integration.

The pipeline has 4 DLQs; each needs its own alarm. The Terraform module below creates alarm + SNS + subscription for a single DLQ and is called 4 times.

---

## Terraform Module

```hcl
# modules/sqs-dlq-alarm/main.tf

variable "dlq_name" {}
variable "dlq_arn" {}
variable "slack_webhook_url_secret_arn" {}
variable "environment" {}

resource "aws_cloudwatch_metric_alarm" "dlq_depth" {
  alarm_name          = "${var.dlq_name}-depth-alarm"
  alarm_description   = "Messages in DLQ ${var.dlq_name} — investigate immediately"
  namespace           = "AWS/SQS"
  metric_name         = "ApproximateNumberOfMessagesVisible"
  dimensions          = { QueueName = var.dlq_name }
  statistic           = "Average"
  period              = 300         # 5 minutes
  evaluation_periods  = 1
  threshold           = 1
  comparison_operator = "GreaterThanOrEqualToThreshold"
  treat_missing_data  = "notBreaching"
  alarm_actions       = [aws_sns_topic.dlq_alerts.arn]
  ok_actions          = [aws_sns_topic.dlq_alerts.arn]
}

resource "aws_sns_topic" "dlq_alerts" {
  name = "${var.dlq_name}-alerts"
}

resource "aws_sns_topic_subscription" "slack" {
  topic_arn = aws_sns_topic.dlq_alerts.arn
  protocol  = "lambda"
  endpoint  = aws_lambda_function.slack_notifier.arn
}
```

---

## Slack Notifier Lambda

```python
# src/functions/slack_notifier/handler.py
import json
import os
import urllib.request

import boto3

secrets = boto3.client("secretsmanager")
_WEBHOOK_URL = None


def _get_webhook() -> str:
    global _WEBHOOK_URL
    if _WEBHOOK_URL is None:
        resp = secrets.get_secret_value(SecretId=os.environ["SLACK_WEBHOOK_SECRET_ARN"])
        _WEBHOOK_URL = json.loads(resp["SecretString"])["url"]
    return _WEBHOOK_URL


def handler(event: dict, _context) -> None:
    for record in event["Records"]:
        sns_message = json.loads(record["Sns"]["Message"])
        alarm_name = sns_message.get("AlarmName", "unknown")
        state = sns_message.get("NewStateValue", "UNKNOWN")
        reason = sns_message.get("NewStateReason", "")

        text = (
            f":rotating_light: *DLQ ALARM* | `{alarm_name}`\n"
            f"*State:* {state}\n"
            f"*Reason:* {reason}\n"
            f"*Action:* Check SQS console → redrive after fix"
        )

        payload = json.dumps({"text": text}).encode()
        req = urllib.request.Request(
            _get_webhook(),
            data=payload,
            headers={"Content-Type": "application/json"},
        )
        urllib.request.urlopen(req)
```

---

## Calling the Module for All 4 DLQs

```hcl
# environments/dev/main.tf

module "alarm_uploaded_dlq" {
  source                       = "../../modules/sqs-dlq-alarm"
  dlq_name                     = "invoice-uploaded-dlq"
  dlq_arn                      = module.sqs.invoice_uploaded_dlq_arn
  slack_webhook_url_secret_arn = aws_secretsmanager_secret.slack_webhook.arn
  environment                  = "dev"
}

module "alarm_converted_dlq" {
  source                       = "../../modules/sqs-dlq-alarm"
  dlq_name                     = "invoice-converted-dlq"
  dlq_arn                      = module.sqs.invoice_converted_dlq_arn
  slack_webhook_url_secret_arn = aws_secretsmanager_secret.slack_webhook.arn
  environment                  = "dev"
}

# Repeat for invoice-classified-dlq and invoice-extracted-dlq
```

---

## Alarm Metrics Explained

| Metric | Why use it |
|--------|------------|
| `ApproximateNumberOfMessagesVisible` | Depth of DLQ — any value ≥ 1 means a message failed 3 times |
| `NumberOfMessagesSent` | Rate of new DLQ entries — useful for trend alerts |

Period 300 s, evaluation_periods 1, threshold 1 is the recommended minimum sensitivity. For prod, consider a 60-second period.

---

## References

- [AWS: Creating CloudWatch alarms for DLQs](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-configure-dead-letter-queue-alarm.html)
- [AWS: SQS dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
