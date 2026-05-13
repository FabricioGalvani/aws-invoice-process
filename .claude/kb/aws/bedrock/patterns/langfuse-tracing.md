> **Web Validated:** 2026-05-12
> **Sources:** notes/07-aws-migration-plan.md §2.6 (LangFuse retained, cloud-agnostic),
>   langfuse.com/docs (generic SDK — cloud-agnostic, no web fetch needed for this)

# Pattern: LangFuse Tracing Around Bedrock Calls

## Context

LangFuse is the project's LLMOps observability layer. It is **cloud-agnostic** —
unchanged by the GCP→AWS migration. LangFuse Public Key lives in SSM Parameter
Store; LangFuse Secret Key in AWS Secrets Manager.

This pattern shows how to wrap a Bedrock `converse()` call with LangFuse
generation tracing. The wrapper is thin — it does not change retry logic or
fallback strategy; those patterns compose independently.

## LangFuse Credentials Setup

```python
import os
import boto3

def get_langfuse_config() -> dict:
    """Load LangFuse credentials at Lambda cold start."""
    ssm = boto3.client("ssm")
    secrets = boto3.client("secretsmanager")

    public_key = ssm.get_parameter(
        Name="/invoice-pipeline/langfuse/public-key"
    )["Parameter"]["Value"]

    secret_key = secrets.get_secret_value(
        SecretId="invoice-pipeline/langfuse/secret-key"
    )["SecretString"]

    host = ssm.get_parameter(
        Name="/invoice-pipeline/langfuse/host"
    )["Parameter"]["Value"]  # e.g. https://cloud.langfuse.com

    return {"public_key": public_key, "secret_key": secret_key, "host": host}
```

## Tracing Wrapper

```python
import time
import logging
from langfuse import Langfuse
from langfuse.model import TextPromptClient

logger = logging.getLogger(__name__)

# Initialise once at cold start
_langfuse: Langfuse | None = None

def get_langfuse() -> Langfuse:
    global _langfuse
    if _langfuse is None:
        cfg = get_langfuse_config()
        _langfuse = Langfuse(
            public_key=cfg["public_key"],
            secret_key=cfg["secret_key"],
            host=cfg["host"],
        )
    return _langfuse


def traced_converse(
    bedrock_client,
    *,
    model_id: str,
    vendor_type: str,
    invoice_id: str,
    trace_name: str = "invoice-extraction",
    **converse_kwargs,
) -> dict:
    """
    Calls bedrock_client.converse(**converse_kwargs) and records a LangFuse
    generation span with input tokens, output tokens, latency, and model.
    Returns the raw Bedrock response dict.
    """
    lf = get_langfuse()
    trace = lf.trace(name=trace_name, metadata={
        "invoice_id": invoice_id,
        "vendor_type": vendor_type,
    })

    generation = trace.generation(
        name="bedrock-converse",
        model=model_id,
        input=converse_kwargs.get("messages"),
    )

    start = time.monotonic()
    try:
        response = bedrock_client.converse(
            modelId=model_id, **converse_kwargs
        )
        latency_ms = (time.monotonic() - start) * 1000

        usage = response.get("usage", {})
        generation.end(
            output=response["output"]["message"]["content"],
            usage={
                "input":  usage.get("inputTokens", 0),
                "output": usage.get("outputTokens", 0),
            },
            metadata={"latency_ms": round(latency_ms, 1)},
        )
        return response

    except Exception as exc:
        latency_ms = (time.monotonic() - start) * 1000
        generation.end(
            metadata={"error": str(exc), "latency_ms": round(latency_ms, 1)},
            level="ERROR",
        )
        raise


def flush_langfuse() -> None:
    """Call at end of Lambda handler to flush buffered events."""
    if _langfuse:
        _langfuse.flush()
```

## Lambda Handler Integration

```python
def handler(event, context):
    try:
        result = traced_converse(
            bedrock_client=bedrock,
            model_id=PRIMARY_MODEL,
            vendor_type=vendor_type,
            invoice_id=invoice_id,
            messages=[...],
            toolConfig=TOOL_CONFIG,
            inferenceConfig={"maxTokens": 512, "temperature": 0.0},
        )
        # ... parse result, validate with Pydantic ...
    finally:
        flush_langfuse()  # Ensure events sent before Lambda container suspends
```

## Observability Fields Captured

| Field | Source | LangFuse location |
|-------|--------|------------------|
| Input tokens | `response["usage"]["inputTokens"]` | `generation.usage.input` |
| Output tokens | `response["usage"]["outputTokens"]` | `generation.usage.output` |
| Model ID | `model_id` parameter | `generation.model` |
| Latency | `time.monotonic()` delta | `generation.metadata.latency_ms` |
| Invoice ID | event payload | `trace.metadata.invoice_id` |
| Vendor type | event payload | `trace.metadata.vendor_type` |

## Anti-Patterns

| Anti-pattern | Fix |
|-------------|-----|
| Skip `flush_langfuse()` in Lambda | Events lost when container suspends |
| Initialise `Langfuse` on every call | Initialise once at cold start |
| Log raw PNG bytes to LangFuse | Pass only text/tool call content |
| Hard-code LangFuse keys | Read from SSM / Secrets Manager |
