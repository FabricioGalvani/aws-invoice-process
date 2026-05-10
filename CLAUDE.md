# CLAUDE.md

> Contexto arquitetural persistente do projeto **Invoice Processing Pipeline**.
> Este arquivo é carregado automaticamente em toda sessão. Mantenha-o conciso e atualizado.

---

## Visão Geral do Projeto

Pipeline serverless AI-powered para extração automatizada de dados de invoices de plataformas de delivery (UberEats, DoorDash, Grubhub) para reconciliação de parceiros restaurantes. Volume: 2.000-3.500 invoices/mês. Meta: ≥90% accuracy de extração. Deadline crítico: 1 de abril de 2026 (Q2 close).

Documentação detalhada vive em [notes/](notes/) — em especial [notes/summary-requirements.md](notes/summary-requirements.md) (SSOT consolidado de 6 reuniões).

---

## Migração GCP → AWS (decisão arquitetural ativa)

**Data da decisão:** 2026-05-10
**Status:** Aprovada — substitui o desenho original em GCP
**Documento de referência:** [notes/07-aws-migration-plan.md](notes/07-aws-migration-plan.md)

### O que era antes (GCP — desenho original em notas 01-06)

O projeto foi inicialmente desenhado com **GCP como cloud primária**:
- Google Cloud Storage (4 buckets) para arquivos
- Pub/Sub (4 tópicos) para eventos
- Cloud Run (4 funções) para compute
- BigQuery como data warehouse
- Vertex AI / Gemini 2.0 Flash como LLM primário
- GCP Secret Manager, Cloud Logging, Cloud Monitoring
- 2 GCP Projects (dev/prod)

### O que mudou

A nuvem primária passou a ser **AWS**. O Adapter Pattern já existente (decisão D10 do summary-requirements) habilita esta troca sem alterar a lógica de negócio.

| Camada | Antes (GCP) | Agora (AWS) |
|---|---|---|
| Storage | GCS | **Amazon S3** |
| Eventos | Pub/Sub | **EventBridge + SQS** (com DLQs) |
| Compute | Cloud Run | **AWS Lambda** (Fargate como fallback) |
| Data warehouse | BigQuery | **Athena + S3 Iceberg** (lakehouse) |
| LLM primário | Gemini 2.0 Flash (Vertex AI) | **Claude Haiku 4.5 (Bedrock)** com Sonnet 4.5 como fallback de qualidade |
| Secrets | GCP Secret Manager | **AWS Secrets Manager + SSM Parameter Store** |
| Logs | Cloud Logging | **CloudWatch Logs → Kinesis Firehose → S3** |
| Métricas | Cloud Monitoring | **CloudWatch Metrics + Alarms** |
| IAM | GCP IAM + Service Accounts | **AWS IAM Roles** (least-privilege por Lambda) |
| Isolamento ambientes | 2 GCP Projects | **2 AWS Accounts via Organizations** |
| CI/CD auth | Workload Identity Federation | **OIDC + IAM Role** (sem long-lived keys) |
| Deploy CLI inicial | `gcloud` | **AWS CLI / SAM CLI** |

### Decisões do summary-requirements substituídas
**D6, D7, D8, D9, D14, D15, D24, D25, D29** — agora valem as versões AWS descritas em `notes/07-aws-migration-plan.md` seção 5.

### Decisões que permanecem inalteradas
**D1-D5, D10-D13, D16-D23, D26-D28, D30** — incluindo: Adapter Pattern, 4 funções separadas, Pydantic validation, Terraform + Terragrunt, GitHub Actions, CodeRabbit, LangFuse, OpenRouter como fallback secundário, CrewAI 3-agent (Triage/Root Cause/Reporter), Slack para alertas, schema de extração de 12 campos, prompts por tipo de invoice.

### Regras que isto impõe daqui pra frente

1. **Não sugerir nem gerar código GCP-específico** (Cloud Run, Pub/Sub, BigQuery, Vertex AI, GCS, Cloud Logging, Cloud Monitoring, GCP Secret Manager, gcloud CLI) para nova implementação. Tratar referências a esses serviços nas notas 01-06 como contexto histórico, não como diretiva atual.
2. **Toda nova função serverless** deve assumir **AWS Lambda** (com Fargate como fallback se ultrapassar 15min ou 10GB RAM).
3. **Toda chamada LLM nova** deve usar **Amazon Bedrock** (Claude Haiku 4.5 padrão; Sonnet 4.5 quando validação Pydantic falhar). OpenRouter permanece como fallback secundário.
4. **Toda escrita de dados extraídos** vai para **S3 + Athena/Iceberg**, não BigQuery.
5. **Toda mensageria entre componentes** usa **EventBridge → SQS** com DLQ por fila.
6. **Toda credencial sensível** vive em **AWS Secrets Manager**; configs não-sensíveis em **SSM Parameter Store**.
7. **Toda IAM** segue least-privilege com Role dedicada por Lambda.
8. **Terraform providers**: usar `aws` (>= 5.x). Manter estrutura Terragrunt (`modules/` + `environments/dev|prod`).
9. **GitHub Actions auth**: via OIDC + IAM Role assumível, nunca long-lived AWS keys.
10. **CrewAI** lê logs do S3 (export via CloudWatch → Firehose), não de GCS.

### Itens deprecados (não usar em novo código)

- `google-cloud-*` SDKs (storage, pubsub, bigquery, aiplatform, secret-manager, logging, monitoring)
- `gcloud` CLI commands em scripts/docs novos
- Módulos Terraform com provider `google` / `google-beta`
- Workload Identity Federation no GitHub Actions
- Referências a `gs://...` URIs (substituir por `s3://...`)
- Variáveis de ambiente / nomes contendo `GCP_`, `GCS_`, `BIGQUERY_`, `VERTEX_`, `PUBSUB_` em código novo

### Estrutura de infraestrutura esperada (AWS)

```
infrastructure/
├── modules/
│   ├── lambda/
│   ├── eventbridge/
│   ├── sqs/
│   ├── s3/
│   ├── athena-iceberg/
│   ├── bedrock/
│   └── iam/
├── environments/
│   ├── dev/terragrunt.hcl
│   └── prod/terragrunt.hcl
└── terragrunt.hcl
```

### Adapters AWS esperados (interfaces já decididas em D10)

- `S3Adapter` (implementa `StorageAdapter`)
- `SqsAdapter` + `EventBridgeAdapter` (implementam `MessagingAdapter`)
- `BedrockAdapter` (implementa `LLMAdapter`)
- `AthenaAdapter` (implementa `WarehouseAdapter`)
- `AwsSecretsManagerAdapter` (implementa `SecretsAdapter`)

---

## Pasta `notes/`

- `01-business-kickoff.md` a `06-autonomous-dataops.md` — atas originais (contexto histórico, design GCP)
- `summary-requirements.md` — SSOT consolidado das 6 reuniões
- `07-aws-migration-plan.md` — **plano de migração AWS ativo (autoritativo para decisões de stack)**
- `tips/` — ignorar (não é parte do escopo do projeto)
