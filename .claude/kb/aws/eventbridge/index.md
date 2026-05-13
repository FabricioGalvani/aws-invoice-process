> **MCP Validated:** 2026-05-12

# Amazon EventBridge — KB Index

Knowledge base for EventBridge usage in the Invoice Processing Pipeline.
Covers the full S3 → EventBridge → SQS → Lambda flow and the EventBridge
Scheduler trigger for CrewAI autonomous ops.

---

## Pipeline Role

EventBridge is the **event routing layer** between S3 and SQS queues:

```
invoices-input-{env}  (S3)
      │ EventBridge notification (single bucket setting)
      ▼
Default Event Bus  (aws.s3 events land here — no custom bus needed)
      │ Rule: source=aws.s3, detail-type=Object Created, suffix=.tiff
      ▼
SQS: invoice-uploaded-{env}   →  Lambda: tiff-to-png-converter
      └── DLQ (eventbridge-dlq + sqs consumer DLQ)

EventBridge Scheduler (separate, time-based)
      │ cron(0 6 * * ? *)  — daily at 06:00 UTC
      ▼
Lambda: crewai-autonomous-ops  →  Triage → Root Cause → Reporter → Slack
```

---

## Concepts

| File | What it explains |
|---|---|
| [event-bus-vs-scheduler.md](concepts/event-bus-vs-scheduler.md) | Rules (reactive) vs Scheduler (time-based) — when to use which |
| [event-patterns.md](concepts/event-patterns.md) | JSON pattern matching; S3 event structure; suffix/prefix filters |
| [targets-and-dlq.md](concepts/targets-and-dlq.md) | Target types; retry policy; two-layer DLQ model; SQS resource policy |
| [input-transformer.md](concepts/input-transformer.md) | JSONPath variables; input template syntax; Terraform HCL |

---

## Patterns

| File | What it implements |
|---|---|
| [s3-to-sqs-routing.md](patterns/s3-to-sqs-routing.md) | Complete S3 → EventBridge Rule → SQS with input transformer and DLQ |
| [scheduled-lambda.md](patterns/scheduled-lambda.md) | EventBridge Scheduler → Lambda (cron, rate, one-time; flexible window) |
| [multi-target-fanout.md](patterns/multi-target-fanout.md) | One rule → multiple targets; when to use SNS instead |

---

## Specs

| File | Contents |
|---|---|
| [eventbridge-rule.json](specs/eventbridge-rule.json) | Machine-readable rule, target, SQS policy, S3 notification, Scheduler spec |

---

## Fast Lookup

See [quick-reference.md](quick-reference.md) for patterns, expressions, and
anti-patterns on a single page.

---

## Key Decisions (from notes/07-aws-migration-plan.md)

- **D7 (AWS):** Event-driven via EventBridge + SQS — replaces Pub/Sub
- S3 → EventBridge uses the **default event bus** (not a custom bus)
- S3 EventBridge notification is a **single bucket-level boolean** (`eventbridge = true`)
- Every SQS queue has **two DLQs**: one at the rule-target level, one at the consumer level
- EventBridge Scheduler replaces Cloud Scheduler for CrewAI cron jobs
- SQS queue policy (resource-based) grants `sqs:SendMessage` — no IAM role needed for rule→SQS

---

## Schema Registry (Optional)

EventBridge can auto-discover schemas from events flowing through the default bus.
Enable a **Discoverer** on the default bus to populate the `discovered-schemas`
registry. This is optional for this pipeline but useful for generating typed event
models. See `specs/eventbridge-rule.json` for the discoverer configuration shape.
