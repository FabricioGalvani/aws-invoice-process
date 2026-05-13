> **Web Validated:** 2026-05-12
> **Sources:** docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-haiku-4-5.html,
>   docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-4-5.html,
>   notes/07-aws-migration-plan.md §2.4

# Claude Models on Bedrock — Haiku 4.5 vs Sonnet 4.5

## Model Identifiers

### Claude Haiku 4.5 (primary)

| Property | Value |
|----------|-------|
| In-region ID | `anthropic.claude-haiku-4-5-20251001-v1:0` |
| US geo profile | `us.anthropic.claude-haiku-4-5-20251001-v1:0` |
| EU geo profile | `eu.anthropic.claude-haiku-4-5-20251001-v1:0` |
| Global profile | `global.anthropic.claude-haiku-4-5-20251001-v1:0` |
| Launch date | 2025-10-16 |
| EOL date | No sooner than 2026-10-01 |

### Claude Sonnet 4.5 (quality fallback)

| Property | Value |
|----------|-------|
| In-region ID | `anthropic.claude-sonnet-4-5-20250929-v1:0` |
| US geo profile | `us.anthropic.claude-sonnet-4-5-20250929-v1:0` |
| EU geo profile | `eu.anthropic.claude-sonnet-4-5-20250929-v1:0` |
| Global profile | `global.anthropic.claude-sonnet-4-5-20250929-v1:0` |
| Launch date | 2025-09-30 |
| EOL date | No sooner than 2026-09-29 |

## Capability Matrix

| Capability | Haiku 4.5 | Sonnet 4.5 |
|-----------|-----------|-----------|
| Context window | 200K tokens | 200K tokens |
| Max output tokens | 64K | 64K |
| Image input | Yes (PNG, JPEG, GIF, WebP) | Yes |
| Tool use / structured output | Yes | Yes |
| Extended thinking / reasoning | Yes | Yes |
| Prompt caching | Yes (min 4096 tokens, 4 checkpoints) | Yes (min 1024 tokens, 4 checkpoints) |
| Knowledge cutoff | Feb 2025 | Apr 2025 |
| Converse API | Yes | Yes |
| InvokeModel API | Yes | Yes |

## Cost and Latency (project §2.4)

| Model | Input $/1M tokens | Output $/1M tokens | Est. $/invoice | P50 latency |
|-------|------------------|--------------------|----------------|-------------|
| Haiku 4.5 | $1.00 | $5.00 | ~$0.001 | ~0.8s |
| Sonnet 4.5 | ~$3.00 | ~$15.00 | ~$0.005 | ~1.5s |

**Quota note:** Claude models apply a **5x burndown multiplier on output tokens**
for quota calculation. A response with 200 output tokens consumes 1000 tokens
of your tokens-per-minute (TPM) quota. Right-size `maxTokens`.

## Accuracy Estimates (benchmark — D15)

| Model | Extraction accuracy |
|-------|-------------------|
| Haiku 4.5 | ~94-95% |
| Sonnet 4.5 | ~96-97% |

Project target is ≥90%. Haiku 4.5 meets the bar at 5x lower cost; Sonnet 4.5
activates only when Pydantic validation fails on the Haiku response.

## Selection Rules

```text
1. Always call Haiku 4.5 first.
2. If Pydantic raises ValidationError → call Sonnet 4.5 with same prompt.
3. If Sonnet 4.5 also fails → raise to DLQ, alert Slack via CrewAI.
4. Never call Sonnet 4.5 first — latency + cost not justified for 94%+ success.
```

## Model ID Anti-Patterns

| Wrong | Right |
|-------|-------|
| `anthropic.claude-haiku-4-5` (no date) | `anthropic.claude-haiku-4-5-20251001-v1:0` |
| `anthropic.claude-sonnet-4-5` (no date) | `anthropic.claude-sonnet-4-5-20250929-v1:0` |
| Hardcoded string in Lambda body | Read from SSM Parameter Store |
