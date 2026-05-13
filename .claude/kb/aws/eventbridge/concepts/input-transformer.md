> **MCP Validated:** 2026-05-12

# EventBridge Input Transformer

## Concept Overview

An **input transformer** reshapes an EventBridge event before it is delivered to
the target. Use it to extract only the fields the target needs, rename keys, or
construct a custom payload — without writing any Lambda glue code.

---

## How It Works

Two components work together:

1. **Input Paths Map** — JSONPath expressions that extract values from the event
   into named variables (up to 100 variables)
2. **Input Template** — A string template that uses `<variable-name>` placeholders
   to compose the final payload delivered to the target

EventBridge evaluates the paths first, then substitutes variables into the template.

---

## JSONPath Reference (EventBridge subset)

| Expression | Extracts |
|---|---|
| `$.source` | Top-level `source` field |
| `$.detail.bucket.name` | Nested field in `detail` |
| `$.detail.object.key` | Object key from S3 event |
| `$.detail.object.size` | Object size in bytes |
| `$.id` | EventBridge event UUID |
| `$.time` | Event timestamp |
| `$.account` | AWS account ID |
| `$.region` | AWS region |

> If a JSONPath does not exist in the event, the variable is omitted. The template
> renders with the placeholder removed (resulting in `null` or an empty string,
> depending on context).

---

## Example: S3 Event → Trimmed SQS Payload

The raw S3 event contains ~20+ fields. The downstream `tiff-to-png-converter`
Lambda only needs `bucket`, `key`, and `eventId`.

### Input Paths Map

```json
{
  "bucket": "$.detail.bucket.name",
  "key":    "$.detail.object.key",
  "size":   "$.detail.object.size",
  "eventId": "$.id"
}
```

### Input Template

```json
{
  "bucket":  "<bucket>",
  "key":     "<key>",
  "size":    <size>,
  "eventId": "<eventId>",
  "source":  "eventbridge-s3"
}
```

> String variables must use `"<var>"` (with quotes). Numeric/boolean/JSON-object
> variables use `<var>` (without quotes). EventBridge adds quotes automatically
> for string variables — do not double-quote them.

### Terraform Configuration

```hcl
resource "aws_cloudwatch_event_target" "invoice_sqs" {
  rule      = aws_cloudwatch_event_rule.invoice_uploaded.name
  target_id = "SendToInvoiceUploadedQueue"
  arn       = aws_sqs_queue.invoice_uploaded.arn

  input_transformer {
    input_paths = {
      bucket  = "$.detail.bucket.name"
      key     = "$.detail.object.key"
      size    = "$.detail.object.size"
      eventId = "$.id"
    }
    input_template = <<-JSON
      {
        "bucket":  "<bucket>",
        "key":     "<key>",
        "size":    <size>,
        "eventId": "<eventId>",
        "source":  "eventbridge-s3"
      }
    JSON
  }
}
```

---

## What Arrives in SQS

The SQS message body will be the rendered template string (a valid JSON object).
Lambda's event handler reads it via `json.loads(record["body"])`.

---

## When NOT to Use an Input Transformer

| Scenario | Better approach |
|---|---|
| Complex conditional logic | Lambda enrichment function before SQS |
| Aggregating multiple events | EventBridge Pipes with enrichment |
| Calling external API for data | EventBridge Pipes enrichment step |
| Template exceeds 8,192 characters | Split payload; store large data in S3 |

---

## References

- [EventBridge input transformation](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-transform-target-input.html)
- [Input transformer tutorial](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-input-transformer-tutorial.html)
- [EventBridge transformer tool](https://eventbridge-transformer.vercel.app/)
