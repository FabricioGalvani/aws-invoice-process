# OpenRouter Knowledge Base

> **Purpose**: OpenRouter as a production secondary fallback LLM gateway in the aws-invoice-process pipeline (after Amazon Bedrock)
> **MCP Validated**: 2026-05-13

## Quick Navigation

### Concepts (< 150 lines each)

| File | Purpose |
|------|---------|
| [concepts/unified-gateway.md](concepts/unified-gateway.md) | What OpenRouter is; base URL, auth, OpenAI SDK compatibility |
| [concepts/model-routing.md](concepts/model-routing.md) | Model-level vs provider-level fallbacks; `models` array vs `provider.order` |
| [concepts/provider-preferences.md](concepts/provider-preferences.md) | Full `provider` object reference: order, allow_fallbacks, data_collection, zdr |
| [concepts/structured-outputs.md](concepts/structured-outputs.md) | `response_format` + json_schema enforcement + Pydantic integration |
| [concepts/auth-and-security.md](concepts/auth-and-security.md) | API key via AWS Secrets Manager; never in env vars or logs |

### Patterns (< 200 lines each)

| File | Purpose |
|------|---------|
| [patterns/bedrock-fallback-adapter.md](patterns/bedrock-fallback-adapter.md) | `OpenRouterAdapter` class implementing the D10 Adapter Pattern |
| [patterns/pydantic-validated-extraction.md](patterns/pydantic-validated-extraction.md) | Structured extraction with json_schema + Pydantic validation |
| [patterns/retry-timeout-strategy.md](patterns/retry-timeout-strategy.md) | Tenacity retry with structured CloudWatch-compatible logging |
| [patterns/cost-telemetry.md](patterns/cost-telemetry.md) | Per-call cost logging from `usage.cost` + generation stats endpoint |

### Specs (Machine-Readable)

| File | Purpose |
|------|---------|
| [specs/client-config.yaml](specs/client-config.yaml) | Minimum viable client config: models, timeouts, provider defaults |
| [specs/fallback-health-criteria.yaml](specs/fallback-health-criteria.yaml) | Acceptance criteria for "OpenRouter fallback healthy" in this pipeline |

---

## Quick Reference

- [quick-reference.md](quick-reference.md) - Fast lookup tables

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Unified Gateway** | Single OpenAI-compatible endpoint for 300+ models; swap base_url only |
| **Provider Object** | Controls routing, compliance (`data_collection: deny`), and performance sort |
| **Model Fallback Array** | `models: [...]` tries IDs in order; `response.model` reveals what ran |
| **Structured Outputs** | `response_format.type: json_schema` + `strict: true` for schema fidelity |
| **Adapter Pattern** | `OpenRouterAdapter` wraps the gateway behind `LLMAdapter` interface (D10) |

---

## Learning Path

| Level | Files |
|-------|-------|
| **Start here** | concepts/unified-gateway.md, concepts/provider-preferences.md |
| **Integration** | patterns/bedrock-fallback-adapter.md, patterns/retry-timeout-strategy.md |
| **Production** | patterns/cost-telemetry.md, specs/fallback-health-criteria.yaml |

---

## Agent Usage

| Agent | Primary Files | Use Case |
|-------|---------------|----------|
| invoice-extractor | patterns/bedrock-fallback-adapter.md, patterns/pydantic-validated-extraction.md | Fallback extraction when Bedrock unavailable |
| devops-agent | specs/fallback-health-criteria.yaml, patterns/cost-telemetry.md | Monitor OpenRouter budget and health |
