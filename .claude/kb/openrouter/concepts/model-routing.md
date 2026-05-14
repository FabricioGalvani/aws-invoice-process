# Model Routing & Fallbacks

> **Purpose**: How OpenRouter selects providers and models; model-level vs provider-level fallback mechanics
> **Confidence**: 0.92
> **MCP Validated**: 2026-05-13

## Overview

OpenRouter supports two independent fallback layers: **provider-level** (same model, different hosting backend) and **model-level** (different model IDs tried in sequence). For an extraction pipeline where output schema fidelity matters more than speed, understanding which layer to use prevents silent quality regressions.

## The Concept

```python
# --- Model-level fallback array ---
# If claude-haiku-4-5 errors, try sonnet-4-5, then gpt-4o-mini.
# response.model tells you which one actually ran.
response = client.chat.completions.create(
    models=[                                  # note: plural "models"
        "anthropic/claude-haiku-4-5",
        "anthropic/claude-sonnet-4-5",
        "openai/gpt-4o-mini",
    ],
    messages=messages,
)
used_model = response.model   # ALWAYS inspect — cost and quality differ

# --- Provider-level routing (same model, different backends) ---
# Restrict to a single provider for determinism; disable fallbacks.
response = client.chat.completions.create(
    model="anthropic/claude-haiku-4-5",
    provider={
        "order": ["Anthropic"],               # try Anthropic-hosted first
        "allow_fallbacks": False,             # fail fast if unavailable
    },
    messages=messages,
)
```

## Quick Reference

| Routing strategy | Field | When to use |
|-----------------|-------|-------------|
| Model waterfall | `models: [...]` | Accept quality trade-offs for availability |
| Provider order | `provider.order: [...]` | Same model, prefer specific infra |
| Block providers | `provider.ignore: [...]` | Exclude non-compliant backends |
| Allow-list only | `provider.only: [...]` | Strict compliance requirement |
| Disable fallbacks | `provider.allow_fallbacks: false` | Fail-fast; no silent degradation |
| Sort by latency | `provider.sort: "latency"` | Minimize P50 response time |
| Sort by price | `provider.sort: "price"` | Cost-optimise batch jobs |

## Common Mistakes

### Wrong

```python
# Using "models" (plural) with allow_fallbacks: false — they conflict.
# model-level fallback fires before provider routing is even evaluated.
response = client.chat.completions.create(
    models=["anthropic/claude-haiku-4-5", "openai/gpt-4o"],
    provider={"allow_fallbacks": False},
    messages=messages,
)
```

### Correct

```python
# For strict fallback control, use singular "model" + provider routing.
response = client.chat.completions.create(
    model="anthropic/claude-haiku-4-5",
    provider={
        "order": ["Anthropic", "AWS Bedrock"],
        "allow_fallbacks": True,
        "require_parameters": True,   # skip providers that don't support tools/schema
    },
    messages=messages,
)
```

## Related

- [unified-gateway.md](../concepts/unified-gateway.md)
- [provider-preferences.md](../concepts/provider-preferences.md)
- [bedrock-fallback-adapter.md](../patterns/bedrock-fallback-adapter.md)
