# Provider Preferences Object

> **Purpose**: Full reference for the `provider` request field — routing, data compliance, and cost controls
> **Confidence**: 0.92
> **MCP Validated**: 2026-05-13

## Overview

The `provider` object in every OpenRouter request controls which hosting backends serve the model, how they are prioritised, and what compliance constraints apply. In a pipeline that handles restaurant invoice data, `data_collection: "deny"` and `zdr: true` are the critical fields for preventing data retention by third-party backends.

## The Concept

```python
# Full provider object — all fields are optional
provider_config = {
    # Routing
    "order": ["Anthropic", "AWS Bedrock"],   # try these providers in order
    "only": None,                             # or allowlist: ["Anthropic"]
    "ignore": ["Hyperbolic"],                # blocklist specific backends
    "allow_fallbacks": True,                 # True = try next on error

    # Compliance
    "data_collection": "deny",              # "allow" | "deny" — suppress training data use
    "zdr": False,                           # Zero Data Retention endpoints only

    # Quality filter
    "require_parameters": True,             # skip providers missing tools/response_format support
    "quantizations": ["fp8", "fp16"],       # filter by quantization level

    # Performance
    "sort": "latency",                      # "price" | "throughput" | "latency"
    "preferred_min_throughput": 40,         # tokens/sec minimum
    "preferred_max_latency": 5,             # seconds P90 max
    "max_price": {
        "prompt": "0.000001",               # $/token ceiling
        "completion": "0.000005",
    },
}
```

## Quick Reference

| Field | Type | Default | Production recommendation |
|-------|------|---------|--------------------------|
| `order` | string[] | load-balanced | Set for deterministic routing |
| `allow_fallbacks` | boolean | `true` | `false` for fail-fast in critical paths |
| `require_parameters` | boolean | `false` | `true` when using tools or response_format |
| `data_collection` | "allow"\|"deny" | `"allow"` | `"deny"` for PII/invoice data |
| `zdr` | boolean | `false` | `true` for strictest compliance |
| `ignore` | string[] | — | Exclude non-compliant or slow backends |
| `sort` | string | — | `"price"` for batch, `"latency"` for real-time |

## Common Mistakes

### Wrong

```python
# Default data_collection="allow" means provider may use your data for training.
response = client.chat.completions.create(
    model="anthropic/claude-haiku-4-5",
    messages=messages,   # contains PII invoice data — no provider constraints set
)
```

### Correct

```python
response = client.chat.completions.create(
    model="anthropic/claude-haiku-4-5",
    provider={
        "data_collection": "deny",
        "require_parameters": True,
    },
    messages=messages,
)
```

## Related

- [model-routing.md](../concepts/model-routing.md)
- [auth-and-security.md](../concepts/auth-and-security.md)
- [cost-telemetry.md](../patterns/cost-telemetry.md)
