# CloudWatch Observability — Invoice Pipeline

> **MCP Validated:** 2026-05-12

Observability stack for the AWS invoice processing pipeline. Covers CloudWatch Logs,
Metrics (EMF), Alarms, Dashboards, Kinesis Firehose log export to S3, and X-Ray tracing.

**Decision reference:** `notes/07-aws-migration-plan.md` §2.6 and §4.7.

---

## Stack at a Glance

```
Lambda stdout (structured JSON via Powertools)
    │
    ├─▶ CloudWatch Log Group /aws/lambda/{fn}-{env}
    │       Retention: 30d (dev) / 90d (prod) — set in Terraform
    │       │
    │       ├─▶ Log Insights — ad-hoc queries
    │       │
    │       └─▶ Subscription Filter
    │               │
    │               └─▶ Kinesis Firehose → S3 (invoices-logs-{env}/)
    │                       └─▶ CrewAI agents read logs from S3
    │
    ├─▶ CloudWatch Metrics (EMF — parsed from stdout, zero latency)
    │       Namespace: InvoicePipeline (custom)
    │       Namespace: AWS/Lambda + AWS/SQS (built-in)
    │       │
    │       └─▶ CloudWatch Alarms → SNS → Slack
    │               SLO alarm: ExtractionAccuracy < 90%
    │               DLQ alarm: any dead-lettered message
    │               Duration alarm: p95 > timeout × 0.8
    │
    └─▶ AWS X-Ray (distributed tracing across 4-Lambda flow)
```

---

## Navigation

### Concepts (what and why)

| File | Covers |
|------|--------|
| [concepts/logs-and-retention.md](concepts/logs-and-retention.md) | Log Groups, retention policy, Log Insights query syntax |
| [concepts/metrics-and-emf.md](concepts/metrics-and-emf.md) | Built-in Lambda/SQS metrics, EMF, custom pipeline metrics |
| [concepts/alarms-and-sns.md](concepts/alarms-and-sns.md) | Alarm anatomy, SLO-driven targets, SNS → Slack routing |
| [concepts/firehose-export.md](concepts/firehose-export.md) | CW Logs → Firehose → S3 architecture and IAM |

### Patterns (how to implement)

| File | Use When |
|------|----------|
| [patterns/emf-from-lambda.md](patterns/emf-from-lambda.md) | Adding `InvoicesProcessed`, `ExtractionAccuracy`, etc. to a Lambda |
| [patterns/slo-alarms.md](patterns/slo-alarms.md) | Creating the full Terraform alarm set for the pipeline |
| [patterns/dlq-depth-alarm.md](patterns/dlq-depth-alarm.md) | Creating a DLQ + alarm pair for an SQS queue |
| [patterns/pipeline-dashboard.md](patterns/pipeline-dashboard.md) | CloudWatch Dashboard Terraform + layout |
| [patterns/logs-to-s3-firehose.md](patterns/logs-to-s3-firehose.md) | Full Terraform for Firehose export (CrewAI log feed) |

### Specs (machine-readable)

| File | Contents |
|------|----------|
| [specs/cloudwatch-iam-policy.json](specs/cloudwatch-iam-policy.json) | Least-privilege IAM for Lambda → CW, CW → Firehose, Firehose → S3 |
| [specs/pipeline-dashboard.json](specs/pipeline-dashboard.json) | Dashboard body JSON (parameterized with {env}, {region}) |

---

## Key Decisions

| Decision | Value |
|----------|-------|
| Log retention | 30d dev / 90d prod |
| EMF namespace | `InvoicePipeline` |
| Custom metrics | `InvoicesProcessed`, `ExtractionAccuracy`, `BedrockFallbacks`, `PydanticValidationFailures` |
| Firehose buffer | 60s / 5MB / GZIP |
| Alert path | SNS → Slack (webhook in Secrets Manager) |
| Long-term logs | S3 Glacier IR at 90d, expire 365d |
| X-Ray | `tracing_config { mode = "Active" }` + `AWSXRayDaemonWriteAccess` on each Lambda role |

## Anti-Patterns

| Never Do | Correct Approach |
|----------|-----------------|
| No `retention_in_days` | Set 30d dev / 90d prod |
| `PutMetricData` from Lambda | Use EMF via Powertools Metrics |
| Alarm without `alarm_actions` | Wire SNS ARN |
| No DLQ alarm | `ApproximateNumberOfMessagesVisible > 0` |
| Dashboard only in console | JSON in Terraform |

## See Also

- `.claude/kb/aws/lambda/patterns/powertools-logging.md` — Logger + Tracer + Metrics
- `notes/07-aws-migration-plan.md` §4.7 — observability decisions table
