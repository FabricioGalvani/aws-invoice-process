# AWS Bedrock IAM Access Module

> **MCP Validated:** 2026-05-12
> **Purpose**: Reusable module that creates a least-privilege IAM policy granting `bedrock:InvokeModel` and `bedrock:InvokeModelWithResponseStream` scoped to specific model ARNs; designed for attachment to Lambda execution roles

## When to Use

Amazon Bedrock itself is an account-level service — there are no per-resource Terraform resources for enabling it. What Terraform *can* and *should* manage is the IAM policy that allows a specific Lambda execution role to call specific Bedrock model endpoints. This module creates that policy document, scoped to a list of model ARN patterns (e.g. `anthropic.claude-haiku-4-5-*` and `anthropic.claude-sonnet-4-5-*`), and supports both foundation model ARNs and cross-region inference profile ARNs. Attach the output `policy_arn` to the `data-extractor` Lambda role.

> **One-time console action (not Terraformable as of 2026-05-12)**: Before any model can be invoked, you must request model access in the AWS Console under Bedrock > Model access. This step is per-region and per-model family. It is a gate that the IAM policy alone cannot bypass. See [AWS docs](https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html).

## Module Structure

```hcl
# modules/bedrock/variables.tf
variable "name" {
  type        = string
  description = "Logical name for the IAM policy (e.g. 'data-extractor-bedrock')"
}
variable "model_ids" {
  type        = list(string)
  default     = ["anthropic.claude-haiku-4-5-*", "anthropic.claude-sonnet-4-5-*"]
  description = "Model ID patterns to allow; used to construct foundation-model ARNs"
}
variable "region" {
  type        = string
  description = "AWS region where Bedrock calls will be made (e.g. 'us-east-1')"
}
variable "enable_cross_region_profiles" {
  type        = bool
  default     = true
  description = "Also grant access to us.* cross-region inference profile ARNs for resilience"
}
variable "tags" { type = map(string); default = {} }
```

```hcl
# modules/bedrock/main.tf
locals {
  # Foundation model ARNs — no account ID component (:: double-colon)
  foundation_model_arns = [
    for id in var.model_ids :
    "arn:aws:bedrock:${var.region}::foundation-model/${id}"
  ]

  # Cross-region inference profile ARNs (us.* prefix convention)
  cross_region_arns = var.enable_cross_region_profiles ? [
    for id in var.model_ids :
    "arn:aws:bedrock:${var.region}::inference-profile/us.${replace(id, "-*", "")}"
  ] : []

  all_arns = concat(local.foundation_model_arns, local.cross_region_arns)
}

data "aws_iam_policy_document" "bedrock" {
  statement {
    sid    = "BedrockInvokeModels"
    effect = "Allow"
    actions = [
      "bedrock:InvokeModel",
      "bedrock:InvokeModelWithResponseStream",
    ]
    resources = local.all_arns
  }
}

resource "aws_iam_policy" "bedrock" {
  name        = var.name
  description = "Least-privilege Bedrock invoke for ${var.name}"
  policy      = data.aws_iam_policy_document.bedrock.json
  tags        = var.tags
}
```

```hcl
# modules/bedrock/outputs.tf
output "policy_arn" { value = aws_iam_policy.bedrock.arn }
output "policy_name" { value = aws_iam_policy.bedrock.name }
```

## Example Terragrunt Invocation (dev)

```hcl
# infrastructure/environments/dev/bedrock-data-extractor/terragrunt.hcl
terraform {
  source = "../../../../modules/bedrock"
}

include "root" { path = find_in_parent_folders() }

inputs = {
  name   = "data-extractor-bedrock-dev"
  region = "us-east-1"

  model_ids = [
    "anthropic.claude-haiku-4-5-*",   # primary model (low cost/latency)
    "anthropic.claude-sonnet-4-5-*"   # fallback when Pydantic validation fails
  ]

  enable_cross_region_profiles = true  # resilience against single-region throttling (MR3)
  tags = { env = "dev", function = "data-extractor" }
}
```

Attach the output policy to the Lambda role in the IAM module invocation:

```hcl
resource "aws_iam_role_policy_attachment" "bedrock" {
  role       = aws_iam_role.data_extractor.name
  policy_arn = module.bedrock.policy_arn
}
```

## Inputs Reference

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `name` | Yes | — | IAM policy name |
| `region` | Yes | — | AWS region for model ARN construction |
| `model_ids` | No | Haiku 4.5 + Sonnet 4.5 | List of model ID patterns |
| `enable_cross_region_profiles` | No | `true` | Add cross-region inference profile ARNs |
| `tags` | No | `{}` | AWS resource tags |

## Common Pitfalls

- **Model access not requested in console**: The IAM policy grants the permission, but Bedrock will still return `AccessDeniedException` until model access is explicitly approved in the AWS Console for each model family and region. This is a one-time manual step that cannot be automated via Terraform as of 2026.
- **Foundation model ARN has double-colon**: The account ID segment is empty for Bedrock foundation models — `arn:aws:bedrock:us-east-1::foundation-model/...`. Using a single colon or inserting an account ID will make the policy invalid.
- **Wildcard in model IDs**: Using `anthropic.claude-haiku-4-5-*` covers all version suffixes of that model family. This is intentional for forward compatibility but means you must still request access to each specific model version in the console.
- **Only attach to `data-extractor`**: The other three Lambdas (`tiff-to-png-converter`, `invoice-classifier`, `warehouse-writer`) do not call Bedrock. Attaching this policy to them violates least-privilege.
- **Cross-region profile ARNs**: Inference profile ARN format is `arn:aws:bedrock:REGION::inference-profile/us.MODEL_ID`. If the model ID pattern contains `-*`, you may need to strip the wildcard when constructing profile ARNs — the `replace` call in the locals block handles this, but verify profile ARNs match the actual registered profiles in your account.

## See Also

- [Lambda Module](./lambda-module.md) — `data-extractor` function that holds this policy
- `notes/07-aws-migration-plan.md` §2.4 — LLM model selection rationale (Haiku primary, Sonnet fallback)
- `notes/07-aws-migration-plan.md` §4.5 — IAM least-privilege table for all four Lambdas
