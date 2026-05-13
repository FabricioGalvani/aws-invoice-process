> **MCP Validated:** 2026-05-12

# EventBridge Event Patterns

## Concept Overview

An event pattern is a JSON object that defines which events a rule matches.
EventBridge compares each incoming event against the pattern; a rule fires only
when ALL specified fields match.

---

## Pattern Matching Rules

- Fields not mentioned in the pattern are ignored (implicit wildcard)
- String values match exactly (case-sensitive)
- Arrays act as OR: `["pdf", "tiff"]` matches either value
- `prefix` matches the start of a string value
- `exists` tests for field presence / absence
- `numeric` supports range comparisons (`<`, `<=`, `>`, `>=`, `=`)
- Nested fields use standard JSON dot-path notation in the pattern object

---

## S3 Event Structure (aws.s3)

When S3 EventBridge notifications are enabled, every object operation emits an
event to the **default event bus** with this envelope:

```json
{
  "version": "0",
  "id": "<uuid>",
  "source": "aws.s3",
  "detail-type": "Object Created",
  "account": "123456789012",
  "region": "us-east-1",
  "detail": {
    "version": "0",
    "bucket": { "name": "invoices-input-dev" },
    "object": {
      "key": "2026/05/invoice-0001.tiff",
      "size": 204800,
      "etag": "abc123",
      "sequencer": "00..."
    },
    "request-id": "<uuid>",
    "requester": "123456789012",
    "source-ip-address": "1.2.3.4",
    "reason": "PutObject"
  }
}
```

`detail-type` values for S3: `Object Created`, `Object Deleted`,
`Object Restore Initiated`, `Object Restore Completed`, `Object Restore Expired`,
`Object Tags Added`, `Object Tags Deleted`, `Object ACL Updated`.

---

## Pattern for This Pipeline

Match only `Object Created` events on the `invoices-input-*` buckets:

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": {
    "bucket": {
      "name": [{ "prefix": "invoices-input-" }]
    },
    "object": {
      "key": [{ "suffix": ".tiff" }]
    }
  }
}
```

> The `prefix` and `suffix` matchers narrow the rule to TIFF files only, avoiding
> spurious triggers from any other object type written to the bucket.

---

## Additional Pattern Examples

### Match specific object key prefix (folder)

```json
{
  "detail": {
    "object": { "key": [{ "prefix": "2026/05/" }] }
  }
}
```

### Match multiple bucket names exactly

```json
{
  "detail": {
    "bucket": { "name": ["invoices-input-dev", "invoices-input-prod"] }
  }
}
```

### Exclude a field value (anything-but)

```json
{
  "detail": {
    "object": { "key": [{ "anything-but": { "suffix": ".png" } }] }
  }
}
```

---

## Anti-Patterns

| Anti-Pattern | Risk | Fix |
|---|---|---|
| Omitting `detail.bucket.name` | Rule fires for ALL S3 buckets in account | Always scope to specific bucket prefix |
| No `detail.object.key` filter | Processes converted PNGs that land in same bucket | Filter by `.tiff` suffix |
| Using `source: ["aws.s3"]` alone | Too broad — matches deletions, restores, tag changes | Add `detail-type: ["Object Created"]` |

---

## References

- [EventBridge event patterns](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-event-patterns.html)
- [S3 EventBridge event structure](https://docs.aws.amazon.com/AmazonS3/latest/userguide/EventBridge.html)
- [Pattern matching reference](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-event-patterns-content-based-filtering.html)
