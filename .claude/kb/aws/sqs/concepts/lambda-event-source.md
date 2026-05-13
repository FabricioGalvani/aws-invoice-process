> **MCP Validated:** 2026-05-12

# Lambda Event Source Mapping for SQS

## How It Works

Lambda maintains an internal fleet of pollers that continuously poll the SQS queue using long polling (`WaitTimeSeconds=20`). When messages are available, the poller retrieves a batch and invokes the Lambda function **synchronously** with the batch as the event payload. Lambda handles the `ReceiveMessage`/`DeleteMessage` lifecycle — the handler function does not call these APIs directly.

```
SQS Queue
    │
    │  (Lambda-managed pollers, WaitTimeSeconds=20)
    ▼
Lambda Service (internal)
    │
    │  InvokeFunction (synchronous)
    ▼
Lambda Handler (your code)
    │
    ├─ All records succeed → Lambda service deletes all messages
    └─ Function throws    → No messages deleted → visibility timeout expires → redelivery
```

---

## Event Source Mapping vs Manual Polling

| Concern | Event Source Mapping (correct) | Manual polling (wrong) |
|---------|-------------------------------|----------------------|
| Who calls ReceiveMessage | Lambda service | Your code |
| Who calls DeleteMessage | Lambda service (on success) | Your code |
| Scaling | Automatic (up to 1 000 pollers) | Manual |
| Cost | No extra calls billed to you | Extra API calls |
| Error handling | Visibility timeout drives retry | Must handle manually |

**Never call `sqs.receive_message()` or `sqs.delete_message()` inside a Lambda handler triggered by SQS event source mapping.** Lambda already did the receive; it will delete on success automatically.

---

## Key Configuration Parameters

| Parameter | Description | Recommended |
|-----------|-------------|-------------|
| `BatchSize` | Max messages per invocation | 5–10 (see queue table) |
| `MaximumBatchingWindowInSeconds` | Wait up to N seconds to fill batch | 0–5 s (low latency pipeline) |
| `FunctionResponseTypes` | Enable `ReportBatchItemFailures` | Always enable |
| `ScalingConfig.MaximumConcurrency` | Cap concurrent Lambda instances | Set per environment |

---

## Concurrency Scaling (Standard Queues)

Lambda starts with 5 concurrent pollers and scales by adding 60 more per minute up to 1 000 (or the account concurrency limit). For this pipeline's throughput (≤120 msgs/hour), scaling events are rare.

---

## Terraform Resource

```hcl
resource "aws_lambda_event_source_mapping" "classified_to_extractor" {
  event_source_arn                   = aws_sqs_queue.invoice_classified.arn
  function_name                      = aws_lambda_function.data_extractor.arn
  batch_size                         = 5
  maximum_batching_window_in_seconds = 0
  function_response_types            = ["ReportBatchItemFailures"]

  scaling_config {
    maximum_concurrency = 5  # conservative for Bedrock quota
  }
}
```

---

## References

- [AWS: Using Lambda with SQS](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html)
- [AWS: Creating and configuring SQS event source mapping](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-configure.html)
- [AWS: Lambda event source mapping invocation](https://docs.aws.amazon.com/lambda/latest/dg/invocation-eventsourcemapping.html)
