> **MCP Validated:** 2026-05-12

# Event Bus (Rules) vs EventBridge Scheduler

## Concept Overview

EventBridge provides two distinct primitives for triggering targets. Choosing the
wrong one is a common architectural mistake.

---

## Event Bus + Rules

**What it is:** A reactive mechanism. A rule continuously watches an event bus for
events matching a pattern and routes matching events to one or more targets.

**Trigger:** An event arrives on the bus (push-based, source-initiated).

**Use when:**
- Reacting to state changes in AWS services (e.g., S3 `Object Created`)
- Fan-out: one event must reach multiple targets simultaneously
- Filtering: only specific events (by bucket, prefix, metadata) should be forwarded
- Decoupling producers from consumers across services

**Pipeline usage:** S3 upload on `invoices-input-*` → default event bus → rule →
SQS `invoice-uploaded`. This is the primary trigger for Lambda 1 (`tiff-to-png-converter`).

---

## EventBridge Scheduler

**What it is:** A proactive mechanism. A schedule fires at a defined time (one-time
or recurring), independent of any event, and invokes a target directly.

**Trigger:** Wall-clock time (cron, rate, or one-time timestamp).

**Use when:**
- Running periodic jobs regardless of whether events occurred
- Replacing Cloud Scheduler / cron-based triggers
- Controlling invocation time (e.g., daily digest, weekly cleanup)
- One-time delayed invocations (fire once at a future timestamp)

**Pipeline usage:** EventBridge Scheduler triggers CrewAI Lambda (or Fargate task)
on a recurring schedule to read S3 logs and run Triage → Root Cause → Reporter.

---

## Comparison Table

| Dimension | Event Bus Rule | EventBridge Scheduler |
|---|---|---|
| Trigger | Event pattern match | Time (cron / rate / one-time) |
| Fan-out | Yes (multiple targets per rule) | No (single target per schedule) |
| Filtering | Deep JSON pattern matching | N/A |
| DLQ support | Per-target DLQ | Built-in DLQ per schedule |
| Retry | Configurable (up to 185, 24 h) | Configurable (up to 185 retries) |
| Flexible time window | No | Yes (0 min – 4 h) |
| Cross-account | Yes (cross-account bus) | Yes (cross-account targets) |
| Terraform resource | `aws_cloudwatch_event_rule` | `aws_scheduler_schedule` |

---

## Decision Rule for This Pipeline

```
Is the trigger an S3 upload or other AWS service event?
  YES → Event Bus Rule (default bus, aws.s3 source)

Is the trigger time-based (daily, weekly, cron)?
  YES → EventBridge Scheduler

Do you need multiple targets from one trigger?
  YES → Event Bus Rule (supports multiple targets per rule)
```

---

## References

- [EventBridge Rules docs](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-create-rule.html)
- [EventBridge Scheduler docs](https://docs.aws.amazon.com/scheduler/latest/UserGuide/what-is-scheduler.html)
- [S3 EventBridge notifications](https://docs.aws.amazon.com/AmazonS3/latest/userguide/EventBridge.html)
