> **Web Validated:** 2026-05-12
> **Sources:** docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_InvokeModel.html,
>   boto3 bedrock-runtime reference, notes/07-aws-migration-plan.md §4.5

# Bedrock Runtime — Client, Regions, IAM

## What It Is

`bedrock-runtime` is the boto3 service client for synchronous model inference.
It is distinct from `bedrock` (the control-plane client used for provisioning).
The `data-extractor` Lambda uses only `bedrock-runtime`.

## Client Initialisation

```python
import boto3

# Minimal — picks up region from Lambda env var AWS_DEFAULT_REGION
client = boto3.client("bedrock-runtime")

# Explicit region (required when using cross-region inference profiles
# that originate from a specific source region)
client = boto3.client("bedrock-runtime", region_name="us-east-1")
```

Auth is fully IAM-based. No API key or secret is needed. The Lambda execution
role must have `bedrock:InvokeModel` on the target model ARN — see
`specs/bedrock-iam-policy.json`.

## Supported APIs

| API | boto3 method | When to use |
|-----|-------------|-------------|
| Converse | `client.converse()` | Preferred — model-agnostic, native image support |
| InvokeModel | `client.invoke_model()` | Lower-level; required if using Anthropic-specific params |
| ConverseStream | `client.converse_stream()` | Streaming; not used in batch invoice pipeline |

Use `converse()` as the default. Switch to `invoke_model()` only if you need
Anthropic-specific parameters not exposed through Converse (e.g. extended thinking
with fine-grained budget tokens).

## invoke_model Payload (Anthropic format)

When using `invoke_model`, the body must be JSON with Anthropic's schema:

```python
import json

body = json.dumps({
    "anthropic_version": "bedrock-2023-05-31",
    "max_tokens": 1024,
    "messages": [{"role": "user", "content": "..."}],
})
response = client.invoke_model(
    modelId="anthropic.claude-haiku-4-5-20251001-v1:0",
    body=body,
    contentType="application/json",
    accept="application/json",
)
result = json.loads(response["body"].read())
```

## IAM — Least Privilege

The `data-extractor` Lambda role must include (see full policy in specs/):

```json
{
  "Effect": "Allow",
  "Action": "bedrock:InvokeModel",
  "Resource": [
    "arn:aws:bedrock:*::foundation-model/anthropic.claude-haiku-4-5-20251001-v1:0",
    "arn:aws:bedrock:*::foundation-model/anthropic.claude-sonnet-4-5-20250929-v1:0"
  ]
}
```

Scope `Resource` to both primary and fallback model ARNs. The wildcard `*` on
region is intentional when using cross-region inference profiles, because the
actual inference region differs from the source region.

## Region Considerations

- The Lambda runs in its deployment region (e.g. `us-east-1`).
- When using cross-region inference profiles (recommended), the API call is still
  made to the Lambda's region endpoint; Bedrock handles the routing internally.
- Bedrock is not available in all regions. For this project, `us-east-1` is the
  primary deployment region with broadest model availability.
- All cross-region data stays on the AWS backbone — never the public internet.

## Environment Variables

| Var | Source | Example |
|-----|--------|---------|
| `BEDROCK_PRIMARY_MODEL_ID` | SSM Parameter Store | `us.anthropic.claude-haiku-4-5-20251001-v1:0` |
| `BEDROCK_FALLBACK_MODEL_ID` | SSM Parameter Store | `us.anthropic.claude-sonnet-4-5-20250929-v1:0` |
| `AWS_DEFAULT_REGION` | Lambda runtime | `us-east-1` |
