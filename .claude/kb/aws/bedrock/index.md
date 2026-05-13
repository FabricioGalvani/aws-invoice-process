> **Web Validated:** 2026-05-12
> **Sources:** docs.aws.amazon.com/bedrock, boto3 bedrock-runtime reference

# Amazon Bedrock KB — Invoice Pipeline

Knowledge base for Amazon Bedrock usage inside the `data-extractor` Lambda.
The extractor calls Claude models via `bedrock-runtime` to read invoice PNGs
and return structured JSON matching the 12-field extraction schema.

---

## Navigation

### Concepts (what things are)

| File | Topic |
|------|-------|
| [bedrock-runtime.md](concepts/bedrock-runtime.md) | Client init, regions, IAM trust |
| [claude-models.md](concepts/claude-models.md) | Haiku 4.5 vs Sonnet 4.5 — IDs, pricing, context |
| [structured-output.md](concepts/structured-output.md) | Tool-use / JSON schema enforcement |
| [inference-profiles.md](concepts/inference-profiles.md) | Cross-region routing for quota/throttle |

### Patterns (how to do things)

| File | Topic |
|------|-------|
| [invoke-claude-vision.md](patterns/invoke-claude-vision.md) | PNG bytes → JSON extraction |
| [fallback-strategy.md](patterns/fallback-strategy.md) | Haiku → Sonnet on Pydantic failure |
| [retry-throttling.md](patterns/retry-throttling.md) | Exponential backoff with jitter |
| [langfuse-tracing.md](patterns/langfuse-tracing.md) | Wrap calls for LangFuse observability |

### Specs (machine-readable)

| File | Topic |
|------|-------|
| [bedrock-iam-policy.json](specs/bedrock-iam-policy.json) | Least-privilege IAM for data-extractor |

---

## Key Facts (fast lookup)

| Property | Value |
|----------|-------|
| Primary model | `anthropic.claude-haiku-4-5-20251001-v1:0` |
| Fallback model | `anthropic.claude-sonnet-4-5-20250929-v1:0` |
| boto3 client | `boto3.client("bedrock-runtime", region_name=...)` |
| Preferred API | `converse()` — model-agnostic, handles images natively |
| Auth | IAM Role on Lambda — no API key needed |
| Fallback trigger | Pydantic `ValidationError` after Haiku response |

---

## Decision Context

- **D15 (updated):** Haiku 4.5 primary; Sonnet 4.5 quality fallback
- **D10 (active):** `BedrockAdapter` implements `LLMAdapter` interface
- **MR3 mitigation:** Cross-region inference profiles for throttle resilience
- See `notes/07-aws-migration-plan.md` §2.4 and §4.5 for full rationale
