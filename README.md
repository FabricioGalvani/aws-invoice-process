# Invoice Processing Pipeline

> Serverless, AI-powered pipeline on AWS that extracts structured data from delivery-platform invoices (UberEats, DoorDash, Grubhub) for restaurant-partner reconciliation.

**Status:** Design phase — implementation starts May 16, 2026 · **Target production launch:** June 20, 2026 · **Original deadline:** April 1, 2026 (Q2 close)

---

## Overview

Three FTEs spend ~80% of their time manually keying delivery-platform invoices into spreadsheets, generating R$45,000+ in quarterly reconciliation errors. This project replaces that workflow with a cloud-native pipeline that ingests TIFF/PDF invoices from S3, classifies them by vendor, extracts 12 structured fields with an LLM, validates against a Pydantic schema, and lands the result in a queryable lakehouse.

Volume target is **2,000–3,500 invoices/month** at **≥ 90% per-field accuracy** and **< $0.01 per extraction**. A CrewAI three-agent crew (Triage → Root Cause → Reporter) watches the pipeline logs and posts incident summaries to Slack — moving the team toward autonomous data ops rather than a passive monitoring stack.

The original design targeted **GCP** (Cloud Run, Pub/Sub, BigQuery, Vertex AI). On **2026-05-10** the team approved a migration to **AWS** as the primary cloud — see [notes/07-aws-migration-plan.md](notes/07-aws-migration-plan.md). The Adapter Pattern decided in the original architecture (D10) lets the migration swap implementations without touching business logic. **All new development targets AWS.**

---

## Project Status

This repository currently contains **planning artifacts only** — meeting notes, consolidated requirements, and the AWS migration plan. No application or infrastructure code has been committed yet. Implementation kicks off with the AWS adapters and Terraform modules during the week of May 16, 2026.

| Phase | Window | Status |
| ----- | ------ | ------ |
| Planning & design (6 meetings) | Jan 15 – Feb 17, 2026 | Done |
| GCP → AWS migration plan | May 10, 2026 | Approved |
| AWS account + OIDC setup | May 11–15, 2026 | Not started |
| Adapters + Terraform modules (AWS) | May 16–25, 2026 | Not started |
| 4 Lambdas migrated to dev | May 26 – Jun 5, 2026 | Not started |
| Bedrock vs Gemini validation sprint | Jun 1–10, 2026 | Not started |
| CrewAI on AWS | Jun 8–15, 2026 | Not started |
| Production launch (AWS) | Jun 15–20, 2026 | Not started |

---

## Architecture

```text
                   INVOICE PROCESSING PIPELINE — AWS

  INGESTION         PROCESSING                                STORAGE

  ┌───────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
  │ TIFF  │──▶│ Lambda 1 │──▶│ Lambda 2 │──▶│ Lambda 3 │──▶│ Lambda 4 │──▶ S3 +
  │  S3   │   │ TIFF→PNG │   │ Classify │   │ Extract  │   │  Write   │   Athena
  └───────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘  /Iceberg
      │           │              │              │              │
      ▼           ▼              ▼              ▼              ▼
  ┌────────────────────────────────────────────────────────────────────┐
  │  EventBridge → SQS queues (each with a DLQ)                        │
  │  invoice-uploaded → converted → classified → extracted             │
  └────────────────────────────────────────────────────────────────────┘
                                ▲
                                │ Bedrock Runtime
                       ┌────────┴────────┐
                       │ Claude Haiku 4.5│ (Sonnet 4.5 fallback on
                       │   on Bedrock    │  Pydantic validation failure)
                       └─────────────────┘

  OBSERVABILITY                   AUTONOMOUS OPS (CrewAI)

  LangFuse  ─ LLM calls           Triage ──▶ Root Cause ──▶ Reporter ──▶ Slack
  CloudWatch Logs + Metrics       (reads pipeline logs from S3 via Firehose)
  Kinesis Firehose ──▶ S3

  CI/CD:  GitHub → CodeRabbit → GitHub Actions (OIDC) → Terraform → AWS
```

The pipeline is **four independent Lambdas** rather than a monolith — each owns one stage and communicates only through SQS queues, so a failure in one stage doesn't crash the others, and each can scale and be redeployed independently.

See [notes/07-aws-migration-plan.md](notes/07-aws-migration-plan.md) section 3 for the full diagram and section 4 for the event flow.

---

## Tech Stack

| Layer | Technology |
| ----- | ---------- |
| Cloud | **AWS** (2 accounts via Organizations: dev, prod) |
| Compute | AWS Lambda (Fargate fallback for >15 min / >10 GB) |
| Messaging | EventBridge + SQS (per-queue DLQs) |
| File storage | Amazon S3 (4 buckets: input, processed, archive, failed) |
| Lakehouse | S3 + Iceberg, queried via Athena |
| LLM (primary) | Amazon Bedrock — **Claude Haiku 4.5** |
| LLM (quality fallback) | Bedrock — Claude Sonnet 4.5 |
| LLM (secondary fallback) | OpenRouter |
| LLM observability | LangFuse (cloud-agnostic) |
| Schema validation | Pydantic |
| Synthetic data | Custom invoice generator (Python + Faker) |
| Secrets | AWS Secrets Manager + SSM Parameter Store |
| Logging | CloudWatch Logs → Kinesis Firehose → S3 |
| Metrics & alarms | CloudWatch (EMF) + SNS → Slack |
| Tracing | AWS X-Ray |
| IaC | Terraform + Terragrunt (provider `aws` ≥ 5.x) |
| CI/CD | GitHub Actions (OIDC, no long-lived AWS keys) |
| Code review | CodeRabbit |
| Autonomous ops | CrewAI — Triage / Root Cause / Reporter agents |
| Alerting channel | Slack |

**Cost estimate:** ~$15/month dev, ~$36/month prod at full volume — see [notes/07-aws-migration-plan.md](notes/07-aws-migration-plan.md) section 6.

---

## Repository Structure

```text
aws-invoice-process/
├── CLAUDE.md                    # Persistent context for AI-assisted development
├── README.md                    # This file
└── notes/                       # Planning artifacts (SSOT)
    ├── 01-business-kickoff.md         # Problem framing, success criteria
    ├── 02-technical-architecture.md   # Original GCP architecture + Adapter Pattern
    ├── 03-data-pipeline-process.md    # 4-function pipeline, Pub/Sub flow, schema
    ├── 04-data-ml-strategy.md         # LLM selection, LangFuse, Pydantic
    ├── 05-devops-infrastructure.md    # Terraform, GitHub Actions, CodeRabbit
    ├── 06-autonomous-dataops.md       # CrewAI three-agent design
    ├── 07-aws-migration-plan.md       # ACTIVE — authoritative AWS architecture
    └── summary-requirements.md        # Consolidated SSOT from 6 meetings
```

### Planned implementation layout (not yet committed)

```text
aws-invoice-process/
├── src/
│   ├── adapters/                # AWS implementations of Adapter Pattern (D10)
│   │   ├── s3_adapter.py
│   │   ├── sqs_adapter.py
│   │   ├── eventbridge_adapter.py
│   │   ├── bedrock_adapter.py
│   │   ├── athena_adapter.py
│   │   └── secrets_adapter.py
│   ├── lambdas/
│   │   ├── tiff_to_png_converter/
│   │   ├── invoice_classifier/
│   │   ├── data_extractor/
│   │   └── warehouse_writer/
│   ├── crewai_agents/
│   │   ├── triage_agent.py
│   │   ├── root_cause_agent.py
│   │   └── reporter_agent.py
│   └── schemas/                 # Pydantic models (12-field extraction schema)
├── infrastructure/
│   ├── modules/
│   │   ├── lambda/
│   │   ├── eventbridge/
│   │   ├── sqs/
│   │   ├── s3/
│   │   ├── athena-iceberg/
│   │   ├── bedrock/
│   │   └── iam/
│   ├── environments/
│   │   ├── dev/terragrunt.hcl
│   │   └── prod/terragrunt.hcl
│   └── terragrunt.hcl
├── tests/
└── .github/workflows/
```

---

## Getting Started

Implementation hasn't started yet. To get oriented:

1. **Read the SSOT first** — [notes/summary-requirements.md](notes/summary-requirements.md) consolidates all 30 decisions, 38 action items, requirements, schema, RACI, and timeline from the 6 planning meetings.
2. **Then read the active stack decisions** — [notes/07-aws-migration-plan.md](notes/07-aws-migration-plan.md) is the authoritative source for cloud services, adapter implementations, IAM, IaC, and the migration timeline. Decisions D6, D7, D8, D9, D14, D15, D24, D25, D29 in the SSOT are **superseded** by this document.
3. **Skim notes 01–06 only for historical context** — they describe the original GCP design. Do not use them as a guide for new code.
4. **Check [CLAUDE.md](CLAUDE.md)** if you're using AI-assisted tooling — it carries the rules that prevent regression to GCP services in new code.

When implementation starts (week of May 16, 2026), this section will be replaced with `git clone`, `terragrunt apply`, and `sam local invoke` instructions.

---

## Extraction Schema (v1)

The pipeline extracts 12 fields per invoice, validated against a Pydantic model:

| Field | Type | Required | Example |
| ----- | ---- | -------- | ------- |
| `invoice_id` | string | yes | `UE-2025-001234` |
| `vendor_name` | string | yes | `Restaurant ABC` |
| `vendor_type` | enum | yes | `ubereats` / `doordash` / `grubhub` / `other` |
| `invoice_date` | date | yes | `2026-01-15` |
| `due_date` | date | yes | `2026-02-15` |
| `subtotal` | float | yes | `1234.56` |
| `tax_amount` | float | yes | `123.45` |
| `commission_rate` | float | yes | `0.15` |
| `commission_amount` | float | yes | `185.18` |
| `total_amount` | float | yes | `1358.01` |
| `currency` | string | yes | `BRL` |
| `line_items` | array | yes | `[{description, quantity, unit_price, amount}]` |

If validation fails, the message is routed to the `invoice-failed` SQS queue and the file moves to `s3://invoices-failed-{env}` for human review.

---

## Success Metrics

| Metric | Target |
| ------ | ------ |
| Per-field extraction accuracy | ≥ 90% |
| Pipeline P95 latency | < 30 s |
| LLM call P95 latency | < 3 s |
| Pipeline availability | > 99% |
| Cost per invoice | < $0.01 |
| Manual processing reduction | > 80% |
| Pydantic validation failure rate | < 5% |
| Time to detect issues (CrewAI) | < 5 min |

---

## Team

| Name | Role | Owns |
| ---- | ---- | ---- |
| Marina Santos | Product Manager | Project leadership, requirements, stakeholder management |
| João Silva | Senior Data Engineer | Architecture, Lambdas, adapter implementations, CrewAI |
| Ana Costa | ML Engineer | LLM selection, prompts, extraction logic, LangFuse |
| Pedro Lima | Platform / DevOps Lead | AWS accounts, Terraform, CI/CD, security |
| Carlos Ferreira | Business Stakeholder | Test data, ground truth, business acceptance |

Full RACI matrix in [notes/summary-requirements.md](notes/summary-requirements.md) section 9.

---

## Documentation

| Document | Purpose |
| -------- | ------- |
| [CLAUDE.md](CLAUDE.md) | Persistent context for AI tooling — enforces AWS-only guidance |
| [notes/summary-requirements.md](notes/summary-requirements.md) | Consolidated SSOT (30 decisions, 38 actions, requirements, schema, timeline) |
| [notes/07-aws-migration-plan.md](notes/07-aws-migration-plan.md) | **Authoritative** AWS architecture, adapters, IaC, IAM, timeline |
| [notes/01-business-kickoff.md](notes/01-business-kickoff.md) | Original problem framing (historical) |
| [notes/02-technical-architecture.md](notes/02-technical-architecture.md) | Original GCP architecture + Adapter Pattern (historical) |
| [notes/03-data-pipeline-process.md](notes/03-data-pipeline-process.md) | 4-function pipeline design (historical) |
| [notes/04-data-ml-strategy.md](notes/04-data-ml-strategy.md) | Original LLM selection — Gemini (historical) |
| [notes/05-devops-infrastructure.md](notes/05-devops-infrastructure.md) | Terraform, GitHub Actions, CodeRabbit setup |
| [notes/06-autonomous-dataops.md](notes/06-autonomous-dataops.md) | CrewAI three-agent autonomous ops design |

---

## Contributing

Until implementation starts, contributions are limited to refining the planning notes. Once code lands:

- Branch off `develop`, target `develop` for PRs (`main` is the production line).
- Every PR is reviewed by **CodeRabbit** (AI) and at least one human reviewer (D26).
- All new infrastructure goes through Terraform/Terragrunt — no manual AWS console changes (D13).
- All new compute goes to **AWS Lambda** (D8 superseded by AWS migration); fall back to Fargate only when Lambda limits are exceeded.
- All new LLM calls go through **Bedrock** (Claude Haiku 4.5 default; Sonnet 4.5 on validation failure).
- No GCP-specific code, SDKs, or `gs://` URIs in new commits — see [CLAUDE.md](CLAUDE.md) for the full deprecation list.

---

## License

No license has been declared yet. Treat this repository as proprietary until a `LICENSE` file is committed.
