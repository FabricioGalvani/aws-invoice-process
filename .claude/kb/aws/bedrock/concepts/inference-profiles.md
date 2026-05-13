> **Web Validated:** 2026-05-12
> **Sources:** docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html,
>   docs.aws.amazon.com/bedrock/latest/userguide/inference-profiles-support.html,
>   notes/07-aws-migration-plan.md §7 risk MR3

# Cross-Region Inference Profiles

## Purpose

Cross-region inference profiles let Bedrock automatically route a request to
another AWS region when the source region is throttled or at quota. This is the
primary mitigation for risk **MR3** (Bedrock quota/throttling).

There is **no additional routing cost**. Price is calculated based on the source
region. All inter-region data travels on the AWS backbone, never the public internet.

## Profile Types

| Type | Data residency | Throughput | Cost |
|------|---------------|-----------|------|
| In-region | Single region only | Baseline | Standard |
| Geo (US/EU/AU/JP) | Within geography | Higher | Standard |
| Global | Any commercial region | Highest | ~10% savings |

**Project recommendation:** Use **US geo profile** in dev and prod (us-east-1 source).
This keeps data in US regions while absorbing throttle spikes across us-east-1,
us-east-2, and us-west-2.

## Profile IDs for This Project

```text
# Haiku 4.5 — US geo (recommended)
us.anthropic.claude-haiku-4-5-20251001-v1:0

# Haiku 4.5 — Global (max throughput, use if US geo exhausted)
global.anthropic.claude-haiku-4-5-20251001-v1:0

# Sonnet 4.5 — US geo (fallback model)
us.anthropic.claude-sonnet-4-5-20250929-v1:0

# Sonnet 4.5 — Global
global.anthropic.claude-sonnet-4-5-20250929-v1:0
```

## US Geo Routing (Haiku 4.5, source us-east-1)

Destination regions: `us-east-1`, `us-east-2`, `us-west-2`

The model ID used in the API call is the geo profile ID. No code change is
required; Bedrock handles the routing internally before the Lambda sees a response.

## IAM Considerations

When using cross-region inference, `bedrock:InvokeModel` must be allowed for
**all destination regions**. Use a wildcard on the region segment of the ARN:

```json
"Resource": [
  "arn:aws:bedrock:*::foundation-model/anthropic.claude-haiku-4-5-20251001-v1:0",
  "arn:aws:bedrock:*::foundation-model/anthropic.claude-sonnet-4-5-20250929-v1:0"
]
```

If your AWS Organization has Service Control Policies (SCPs) that restrict regions,
ensure all destination regions (`us-east-1`, `us-east-2`, `us-west-2`) are
allowed for Bedrock inference actions.

## CloudTrail Logging

Cross-region inference requests are logged in CloudTrail in the **source region**.
The `additionalEventData.inferenceRegion` field shows where the request was processed.
Use this for cost attribution and debugging throttle events.

## Quota Monitoring

```python
# CloudWatch alarm recommended at 70-80% of TPM quota
# Haiku 4.5 applies 5x burndown multiplier on output tokens:
# 100 output tokens → 500 tokens consumed from TPM quota
# Right-size maxTokens to avoid unexpected throttling
```

Set CloudWatch alarm on `EstimatedTotalTokensConsumed` metric at 75% of
your account's TPM limit. Request quota increase proactively via AWS console.

## Provisioned Throughput

Inference profiles **do not support** Provisioned Throughput. If deterministic
SLA is required, switch to a standalone in-region model ID and purchase
Provisioned Throughput separately. For this pipeline's volume (2k-3.5k
invoices/month), on-demand + cross-region is sufficient.
