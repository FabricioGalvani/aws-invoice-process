# Cost Telemetry & Observability

> **Purpose**: Log per-call spend from OpenRouter responses and the generation stats endpoint; emit to CloudWatch for budget alerting
> **MCP Validated**: 2026-05-13

## When to Use

- Tracking fallback path cost separately from primary Bedrock cost
- Populating a CloudWatch custom metric for monthly budget alarms
- Reconciling OpenRouter credit consumption against invoice processing volume

## Implementation

```python
"""
Cost telemetry for OpenRouter calls in Lambda.
Usage object returned in every response; generation endpoint for async audit.
"""
from __future__ import annotations

import logging
import time
from dataclasses import dataclass, asdict

import boto3
import httpx
from openai.types.chat import ChatCompletion

logger = logging.getLogger(__name__)
cw = boto3.client("cloudwatch", region_name="us-east-1")


@dataclass
class CallTelemetry:
    model: str
    prompt_tokens: int
    completion_tokens: int
    cost_usd: float | None          # None if provider doesn't return cost
    generation_id: str | None       # OpenRouter generation ID for async lookup
    elapsed_ms: int


def extract_telemetry(response: ChatCompletion, elapsed_ms: int) -> CallTelemetry:
    """Pull cost data directly from the response object."""
    usage = response.usage
    cost = getattr(usage, "cost", None)                    # optional field
    gen_id = getattr(response, "id", None)                 # use as generation ID

    tel = CallTelemetry(
        model=response.model,
        prompt_tokens=usage.prompt_tokens,
        completion_tokens=usage.completion_tokens,
        cost_usd=cost,
        generation_id=gen_id,
        elapsed_ms=elapsed_ms,
    )
    logger.info("openrouter_call", extra=asdict(tel))
    return tel


def emit_cost_metric(tel: CallTelemetry, namespace: str = "InvoicePipeline") -> None:
    """Publish to CloudWatch for budget alarm integration."""
    if tel.cost_usd is None:
        return
    cw.put_metric_data(
        Namespace=namespace,
        MetricData=[{
            "MetricName": "OpenRouterCostUSD",
            "Value": tel.cost_usd,
            "Unit": "None",
            "Dimensions": [{"Name": "Model", "Value": tel.model}],
        }],
    )


def fetch_generation_stats(api_key: str, generation_id: str) -> dict:
    """
    Async cost lookup — call after response received.
    Useful when usage.cost is None (some providers omit it).
    Endpoint: GET https://openrouter.ai/api/v1/generation?id={generation_id}
    """
    resp = httpx.get(
        f"https://openrouter.ai/api/v1/generation?id={generation_id}",
        headers={"Authorization": f"Bearer {api_key}"},
        timeout=5.0,
    )
    resp.raise_for_status()
    return resp.json()


# --- Integrated usage in adapter ---
def invoke_and_track(client, model: str, messages: list, api_key: str, **kwargs) -> tuple[str, CallTelemetry]:
    t0 = time.monotonic()
    response = client.chat.completions.create(model=model, messages=messages, **kwargs)
    elapsed = int((time.monotonic() - t0) * 1000)

    tel = extract_telemetry(response, elapsed)
    emit_cost_metric(tel)

    content = response.choices[0].message.content
    return content, tel
```

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| CloudWatch namespace | `"InvoicePipeline"` | Change per environment |
| Metric name | `"OpenRouterCostUSD"` | Dimension: `Model` |
| Generation endpoint | `GET /api/v1/generation?id=` | Async fallback when `usage.cost` is None |

## Example Usage

```python
content, tel = invoke_and_track(
    client=_get_client(),
    model="anthropic/claude-haiku-4-5",
    messages=messages,
    api_key=_get_api_key(),
    max_tokens=512,
    temperature=0.0,
)
# tel.cost_usd available for budget reporting
# TODO: wire tel into your existing LangFuse trace if active
```

## See Also

- [retry-timeout-strategy.md](../patterns/retry-timeout-strategy.md)
- [bedrock-fallback-adapter.md](../patterns/bedrock-fallback-adapter.md)
- [auth-and-security.md](../concepts/auth-and-security.md)
