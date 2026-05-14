# OpenRouter Quick Reference

> Fast lookup tables. For code examples, see linked files.
> **MCP Validated:** 2026-05-13

## Endpoint & Auth

| Item | Value |
|------|-------|
| Base URL | `https://openrouter.ai/api/v1` |
| Chat completions | `POST /chat/completions` |
| Generation stats | `GET /generation?id={id}` |
| Auth header | `Authorization: Bearer sk-or-v1-...` |
| SDK | `openai` Python SDK v1.x — change `base_url` only |
| Key location | AWS Secrets Manager `prod/invoice-pipeline/openrouter-api-key` |

## Model IDs (Invoice Pipeline)

| Role | Model ID | Notes |
|------|----------|-------|
| Primary (mirrors Bedrock) | `anthropic/claude-haiku-4-5` | Fast, cheap; use for all first attempts |
| Quality fallback | `anthropic/claude-sonnet-4-5` | Triggered when Haiku fails Pydantic validation |
| Auto-router | `openrouter/auto` | Let OpenRouter pick best available — NOT for production extraction |

## Provider Object Fields

| Field | Type | Default | Use in this pipeline |
|-------|------|---------|---------------------|
| `order` | string[] | load-balanced | `["Anthropic"]` for determinism |
| `allow_fallbacks` | bool | `true` | `true` (fail over within same model) |
| `require_parameters` | bool | `false` | `true` (enforce schema support) |
| `data_collection` | "allow"\|"deny" | "allow" | `"deny"` (invoice PII) |
| `zdr` | bool | `false` | `true` if compliance requires ZDR |
| `ignore` | string[] | — | Block non-compliant backends |
| `sort` | string | — | `"price"` for batch, `"latency"` for real-time |

## Response Format (Structured Outputs)

| Field | Value | Notes |
|-------|-------|-------|
| `response_format.type` | `"json_schema"` | Prefer over `"json_object"` |
| `json_schema.strict` | `true` | Fail rather than return non-conforming JSON |
| `temperature` | `0.0` | Always for extraction tasks |
| `require_parameters` | `true` | Skip backends that ignore response_format |

## Usage Object (Cost Fields)

| Field | Type | Notes |
|-------|------|-------|
| `usage.prompt_tokens` | int | Input tokens |
| `usage.completion_tokens` | int | Output tokens |
| `usage.cost` | float\|null | USD cost; null if provider omits |
| `response.model` | string | Actual model used — may differ from requested |
| `response.id` | string | Use as `generation_id` for stats endpoint |

## Decision Matrix

| Use Case | Choose |
|----------|--------|
| Bedrock unavailable | `OpenRouterAdapter` with `model=anthropic/claude-haiku-4-5` |
| Haiku output fails Pydantic | Re-invoke with `model=anthropic/claude-sonnet-4-5` |
| Need deterministic provider | `provider.order=["Anthropic"], allow_fallbacks=False` |
| PII data in prompt | Always set `data_collection="deny"` |
| Batch cost optimisation | `provider.sort="price"` |
| Latency-sensitive path | `provider.sort="latency"` |
| Schema must be enforced | `response_format.type="json_schema", strict=True, require_parameters=True` |

## Common Pitfalls

| Don't | Do |
|-------|-----|
| Store key in Lambda env vars | Fetch from Secrets Manager in handler |
| Log the full API key | Log only the first 8 characters as prefix |
| Use `models: [...]` with `allow_fallbacks: False` | Use singular `model` + `provider.order` |
| Ignore `response.model` | Always log it — silent model switches affect cost & quality |
| Default `data_collection` | Explicitly set `"deny"` for invoice data |
| Use `json_object` mode | Use `json_schema` with `strict: true` for extraction |

## Retry Budget (Lambda)

| Scenario | Per-attempt timeout | Max attempts | Worst-case total |
|----------|---------------------|--------------|-----------------|
| Haiku (fast) | 10s | 3 | ~46s |
| Sonnet (quality) | 30s | 3 | ~94s |
| Lambda timeout recommended | — | — | ≥ 120s |

## Related Documentation

| Topic | Path |
|-------|------|
| Adapter implementation | `patterns/bedrock-fallback-adapter.md` |
| Structured extraction | `patterns/pydantic-validated-extraction.md` |
| Full Index | `index.md` |
