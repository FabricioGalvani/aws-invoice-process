# Invoice Processing Pipeline — Summary of Requirements (SSOT)

> **Status:** Active | **Última consolidação:** 2026-05-11
> **Fontes:** 6 atas de reunião (01–06) + plano de migração AWS (07)
> **Confiança geral:** 0.93
> **Idioma:** Português (citações de reuniões mantidas em inglês quando relevantes)
> **Convenção:** Decisões marcadas com ★ foram **substituídas** pela migração GCP→AWS (2026-05-10). Decisões marcadas com ✓ permanecem **inalteradas**.

---

## 1. Executive Summary

| Aspecto | Detalhe |
|---|---|
| **Projeto** | Invoice Processing Pipeline |
| **Problema de negócio** | 3 FTEs gastam 80% do tempo em entrada manual de dados de invoices de plataformas de delivery (UberEats, DoorDash, Grubhub) no SAP. Erros de reconciliação custaram R$45.000 no último trimestre. |
| **Solução** | Pipeline serverless AI-powered para extração automatizada de dados de invoices, com validação Pydantic e escrita em data warehouse |
| **Volume** | 2.000 invoices/mês atual → 3.500 invoices/mês até fim de ano |
| **Meta de acurácia** | ≥ 90% em cada campo extraído |
| **Deadline crítico** | **1 de abril de 2026** (Q2 close em 30 de abril de 2026) |
| **Stack original** | GCP (GCS + Pub/Sub + Cloud Run + BigQuery + Vertex AI/Gemini 2.0 Flash) |
| **Stack atual** | **AWS** (S3 + EventBridge/SQS + Lambda + Athena/Iceberg + Bedrock/Claude Haiku 4.5) |
| **Doc autoritativo de stack** | [`07-aws-migration-plan.md`](07-aws-migration-plan.md) — aprovado em **2026-05-10** |

> ⚠️ **Aviso de migração:** O projeto foi inicialmente desenhado em GCP nas atas 01–06. Em 2026-05-10, a equipe aprovou a migração para AWS, viabilizada pelo Adapter Pattern previsto em [D10](#decisões-d1d30). Esta SSOT preserva as decisões originais como registro histórico **e** apresenta as substituições AWS lado a lado.

---

## 2. Stakeholders & Roles (RACI)

| Pessoa | Cargo | R | A | C | I | Notas |
|---|---|---|---|---|---|---|
| **Marina Santos** | Product Manager | — | ✅ Decisões finais de escopo, timeline e go/no-go | Stakeholders executivos | Equipe técnica | Owner do produto; agenda reuniões; aprova promoções dev→prod |
| **João Silva** | Senior Data Engineer | ✅ Pipeline, adapters, infra de dados, warehouse-writer, tiff-to-png-converter | — | Pedro, Ana | Marina | Architect informal; propôs Adapter Pattern e CrewAI |
| **Ana Costa** | ML Engineer | ✅ Classifier, data-extractor, prompts, LLM eval, LangFuse | — | João | Marina | Owner do ML/LLM; experiência prévia com document extraction |
| **Pedro Lima** | Platform/DevOps Lead | ✅ IaC (Terraform/Terragrunt), CI/CD, IAM, secrets, observabilidade infra | — | João, Ana | Marina | Pushing strong DevOps practices desde o início |
| **Carlos Ferreira** | Business Stakeholder (Restaurant Operations) | ✅ Sample data, ground truth labeling, validação business | Negócio | — | Marina | "Internal champion"; coordena com legal para dados de produção |
| **Security Team** | (TBD) | — | — | Pedro, Marina | Marina | Security review obrigatório antes do production launch ([M6](06-autonomous-dataops.md)) |

---

## 3. Business Requirements

### 3.1 Funcionais (FR)

| # | Requisito | Origem | Prioridade |
|---|-----------|--------|------------|
| FR1 | Sistema deve extrair invoice_id, vendor, datas, valores, line items e commission de invoices em TIFF/PDF | [M1](01-business-kickoff.md), [M3](03-data-pipeline-process.md), [M4](04-data-ml-strategy.md) | MVP |
| FR2 | Sistema deve identificar automaticamente o vendor type (UberEats, DoorDash, Grubhub) | [M3](03-data-pipeline-process.md) | MVP |
| FR3 | Sistema deve aplicar prompt específico por vendor type | [M3](03-data-pipeline-process.md), [M4](04-data-ml-strategy.md) | MVP |
| FR4 | Sistema deve validar saída do LLM contra schema Pydantic estrito | [M4](04-data-ml-strategy.md) | MVP |
| FR5 | Sistema deve enviar invoices que falham validação para review queue | [M3](03-data-pipeline-process.md) | MVP |
| FR6 | Sistema deve preservar arquivo original sem nunca deletar (compliance) | [M3](03-data-pipeline-process.md) | MVP |
| FR7 | Sistema deve registrar todas as chamadas LLM (prompt, response, latência, custo) | [M4](04-data-ml-strategy.md) | Phase 2 (MVP usa logs básicos) |
| FR8 | Sistema deve detectar anomalias e classificar severidade automaticamente | [M6](06-autonomous-dataops.md) | Phase 2 |
| FR9 | Sistema deve gerar relatório human-readable de incidentes via Slack | [M6](06-autonomous-dataops.md) | Phase 2 |
| FR10 | Sistema deve permitir rollback rápido de deployments e prompts | [M5](05-devops-infrastructure.md) | MVP |

### 3.2 Não-funcionais (NFR)

| # | Requisito | Target | Origem |
|---|-----------|--------|--------|
| NFR1 | Acurácia de extração por campo | ≥ 90% (97% atual com Gemini, ~95% projetado com Claude Haiku) | [M1](01-business-kickoff.md), [M4](04-data-ml-strategy.md) |
| NFR2 | Throughput | 2.000–3.500 invoices/mês | [M1](01-business-kickoff.md) |
| NFR3 | Latência P95 ponta-a-ponta | < 30s | [M6](06-autonomous-dataops.md) |
| NFR4 | Disponibilidade do pipeline | > 99% | [M6](06-autonomous-dataops.md) |
| NFR5 | Custo por invoice | < $0.01 | [M6](06-autonomous-dataops.md) |
| NFR6 | Custo total de infraestrutura | < $20/mês dev, < $60/mês prod | [M5](05-devops-infrastructure.md), [07](07-aws-migration-plan.md) |
| NFR7 | Tempo para detectar incidentes | < 5 min | [M6](06-autonomous-dataops.md) |
| NFR8 | Redução de processamento manual | > 80% | [M6](06-autonomous-dataops.md) |
| NFR9 | Segurança: secrets nunca em código ou env vars | 100% | [M5](05-devops-infrastructure.md) |
| NFR10 | Resiliência: retry + DLQ por etapa do pipeline | obrigatório | [M2](02-technical-architecture.md), [M3](03-data-pipeline-process.md) |

### 3.3 Constraints

| # | Constraint | Origem |
|---|-----------|--------|
| C1 | Não pode usar GCP-specific services para novo código (pós migração) | [CLAUDE.md](../CLAUDE.md), [07](07-aws-migration-plan.md) |
| C2 | Toda credencial sensível deve viver em AWS Secrets Manager | [CLAUDE.md](../CLAUDE.md), [07](07-aws-migration-plan.md) |
| C3 | Arquivos originais não podem ser deletados (compliance) | [M3](03-data-pipeline-process.md) |
| C4 | Authentication GitHub→AWS via OIDC (sem long-lived keys) | [CLAUDE.md](../CLAUDE.md), [07](07-aws-migration-plan.md) |
| C5 | Feature branch workflow obrigatório com PR review humano | [M5](05-devops-infrastructure.md) |
| C6 | Security review obrigatório antes do production launch | [M6](06-autonomous-dataops.md) |
| C7 | Deadline imutável: 1 de abril de 2026 (Q2 close) | [M1](01-business-kickoff.md), [M6](06-autonomous-dataops.md) |

---

## 4. Architecture Overview

### 4.1 Original design (GCP — histórico, atas 01–06)

```
TIFF Upload → GCS (input bucket)
    ↓ trigger
Pub/Sub topic "invoice-uploaded"
    ↓
Cloud Run Function 1: tiff-to-png-converter
    ↓ (PNG → GCS processed bucket)
Pub/Sub topic "invoice-converted"
    ↓
Cloud Run Function 2: invoice-classifier
    ↓
Pub/Sub topic "invoice-classified"
    ↓
Cloud Run Function 3: data-extractor (Vertex AI / Gemini 2.0 Flash)
    ↓
Pub/Sub topic "invoice-extracted"
    ↓
Cloud Run Function 4: bigquery-writer
    ↓
BigQuery (data warehouse)

Observability: LangFuse + Cloud Logging + Cloud Monitoring
Secrets:       GCP Secret Manager
IaC:           Terraform + Terragrunt (provider google)
CI/CD:         GitHub Actions + CodeRabbit + Workload Identity Federation
AutoOps:       CrewAI (3 agentes) lendo Cloud Logging → GCS export
```

Buckets GCS originais:
- `gs://invoices-input` — raw TIFF
- `gs://invoices-processed` — PNGs convertidos
- `gs://invoices-archive` — originais retidos
- `gs://invoices-failed` — review queue

### 4.2 Current design (AWS — ativo, autoritativo)

> **Fonte autoritativa:** [`07-aws-migration-plan.md`](07-aws-migration-plan.md) seções 2–4

```
TIFF Upload → S3 (invoices-input-{env})
    ↓ S3 Event Notification
EventBridge → SQS "invoice-uploaded" (+ DLQ)
    ↓
Lambda 1: tiff-to-png-converter (Python 3.12 + Pillow, 2GB, 5min)
    ↓ (PNG → S3 processed bucket; original → archive bucket com Glacier lifecycle)
SQS "invoice-converted" (+ DLQ)
    ↓
Lambda 2: invoice-classifier (Python 3.12, 512MB, 1min)
    ↓
SQS "invoice-classified" (+ DLQ)
    ↓
Lambda 3: data-extractor (Python 3.12 + boto3 Bedrock, 1GB, 2min)
    ↓ (chama Bedrock Claude Haiku 4.5; fallback Sonnet 4.5)
SQS "invoice-extracted" (+ DLQ)
    ↓
Lambda 4: warehouse-writer (Python 3.12, 512MB, 1min)
    ↓
S3 + Athena/Iceberg (extracted_invoices, particionado por vendor_type, invoice_date)

Observability: LangFuse (mantido) + CloudWatch Logs + CloudWatch Metrics
               + Kinesis Firehose → S3 (para CrewAI)
               + AWS X-Ray (distributed tracing)
Secrets:       AWS Secrets Manager (sensíveis) + SSM Parameter Store (config)
IaC:           Terraform + Terragrunt (provider aws >= 5.x)
CI/CD:         GitHub Actions + CodeRabbit + OIDC + IAM Role assumível
AutoOps:       CrewAI (3 agentes) lendo S3 (logs export)
Isolamento:    2 AWS Accounts via AWS Organizations (dev / prod)
```

Buckets S3 atuais:
- `s3://invoices-input-{env}` — raw TIFF (Lifecycle: 30 days)
- `s3://invoices-processed-{env}` — PNGs convertidos (Lifecycle: 90 days)
- `s3://invoices-archive-{env}` — originais (Lifecycle: 7 anos → Glacier)
- `s3://invoices-failed-{env}` — failed processing review queue
- `s3://invoices-warehouse-{env}` — Iceberg tables

### 4.3 O que permanece igual (cloud-agnostic)

| Item | Inalterado | Origem |
|---|---|---|
| **Adapter Pattern** (StorageAdapter, MessagingAdapter, LLMAdapter, WarehouseAdapter, SecretsAdapter) | ✓ | [D10](#decisões-d1d30) — habilitou a migração |
| **4 funções separadas** (modular, não monolito) | ✓ | [D12](#decisões-d1d30) |
| **Schema de extração de 12 campos com Pydantic** | ✓ | [D18](#decisões-d1d30) |
| **Prompts por tipo de invoice** | ✓ | [D23](#decisões-d1d30) |
| **Pydantic validation strict** | ✓ | [D18](#decisões-d1d30) |
| **OpenRouter como fallback** | ✓ | [D16](#decisões-d1d30) |
| **LangFuse para LLMOps** | ✓ | [D17](#decisões-d1d30) |
| **CrewAI 3-agent (Triage / Root Cause / Reporter)** | ✓ | [D30](#decisões-d1d30) |
| **Slack para alertas** | ✓ | M6 |
| **GitHub Actions + CodeRabbit + Terraform + Terragrunt** | ✓ | [D11, D26, D27](#decisões-d1d30) |
| **Feature branch workflow + required PR reviews** | ✓ | [D28](#decisões-d1d30) |
| **Métricas de sucesso (90% accuracy, etc.)** | ✓ | M1, M4, M6 |
| **Equipe e RACI** | ✓ | — |

---

## 5. Decisões (D1–D30)

### 5.1 Tabela mestre

> ★ = substituída pela migração AWS (ver detalhe em §5.2)
> ✓ = inalterada

| # | Decisão | Owner | Reunião / Source | Data | Status | AWS replacement |
|---|---------|-------|------------------|------|--------|-----------------|
| **D1** | Construir pipeline automatizado de extração de invoices | Marina | [M1](01-business-kickoff.md) | 2026-01-15 | ✓ Aprovada | — |
| **D2** | Adotar 90% de acurácia como threshold MVP | Ana | [M1](01-business-kickoff.md) | 2026-01-15 | ✓ Aprovada | — |
| **D3** | Iniciar somente com UberEats (60% do volume) | Carlos | [M1](01-business-kickoff.md) | 2026-01-15 | ✓ Aprovada | — |
| **D4** | Arquitetura cloud-native serverless | João | [M1](01-business-kickoff.md) | 2026-01-15 | ✓ Aprovada | — |
| **D5** | Deadline de produção: 1 de abril de 2026 (Q2 close 30/abr) | Marina | [M1](01-business-kickoff.md) (notas observacionais) | 2026-01-15 | ✓ Aprovada | — |
| **D6** ★ | **GCP** como nuvem primária | João | [M2](02-technical-architecture.md) | 2026-01-22 | ★ Substituída | **AWS** como nuvem primária ([07 §5](07-aws-migration-plan.md#5-decisões-atualizadas-substituem-gcp-specific)) |
| **D7** ★ | Event-driven com **Pub/Sub** (4 tópicos com DLQ) | João | [M2](02-technical-architecture.md), [M3](03-data-pipeline-process.md) | 2026-01-22 | ★ Substituída | **EventBridge + SQS** (com DLQ por fila) |
| **D8** ★ | **Cloud Run** para compute serverless | João | [M2](02-technical-architecture.md) | 2026-01-22 | ★ Substituída | **AWS Lambda** (Fargate como fallback se >15min ou >10GB) |
| **D9** ★ | **BigQuery** como destination data warehouse | João | [M2](02-technical-architecture.md) | 2026-01-22 | ★ Substituída | **Athena + S3 Iceberg** (lakehouse) |
| **D10** | **Adapter Pattern** para portabilidade multi-cloud | João | [M2](02-technical-architecture.md) | 2026-01-22 | ✓ Mantida (HABILITOU a migração) | Implementação primária agora em AWS |
| **D11** | **Terraform + Terragrunt** para IaC | Pedro | [M2](02-technical-architecture.md), [M5](05-devops-infrastructure.md) | 2026-01-22 / 2026-02-10 | ✓ Mantida | Provider trocado de `google` para `aws >= 5.x` |
| **D12** | **4 funções separadas** (não monolito), comunicando via mensageria | João | [M3](03-data-pipeline-process.md) | 2026-01-27 | ✓ Mantida | Funções viram Lambdas |
| **D13** | Função 1: **tiff-to-png-converter** (split multi-page, otimização para LLM) | João | [M3](03-data-pipeline-process.md) | 2026-01-27 | ✓ Mantida | Roda em Lambda 2GB / 5min |
| **D14** ★ | **2 GCP Projects** separados (dev/prod) para isolamento | Pedro | [M5](05-devops-infrastructure.md) | 2026-02-10 | ★ Substituída | **2 AWS Accounts via AWS Organizations** |
| **D15** ★ | **Gemini 2.0 Flash** (Vertex AI) como LLM primário | Ana | [M4](04-data-ml-strategy.md) | 2026-02-03 | ★ Substituída | **Claude Haiku 4.5 (Bedrock)** primário; **Sonnet 4.5** como fallback de qualidade |
| **D16** | **OpenRouter** como fallback provider de LLM | Ana | [M4](04-data-ml-strategy.md) | 2026-02-03 | ✓ Mantida (agora secundário) | Bedrock multi-model é fallback primário; OpenRouter é secundário |
| **D17** | **LangFuse** para LLMOps observability (traces, custo, latência, qualidade) | Ana | [M4](04-data-ml-strategy.md) | 2026-02-03 | ✓ Mantida | Cloud-agnostic |
| **D18** | **Structured JSON output com Pydantic validation** (strict) | João | [M4](04-data-ml-strategy.md) | 2026-02-03 | ✓ Mantida | Cloud-agnostic |
| **D19** | Construir **invoice generator sintético** para test data com ground truth | João | [M4](04-data-ml-strategy.md) | 2026-02-03 | ✓ Mantida | Cloud-agnostic |
| **D20** | Trackear **4 métricas-chave**: cost per extraction, latency P95, accuracy per field, token usage | Ana | [M4](04-data-ml-strategy.md) | 2026-02-03 | ✓ Mantida | Métricas iguais; instrumentação muda |
| **D21** | Função 3: **data-extractor** chamando LLM com prompt template carregado por classification | Ana | [M3](03-data-pipeline-process.md) | 2026-01-27 | ✓ Mantida | Agora chama Bedrock via boto3 |
| **D22** | Função 4: **warehouse-writer** (originalmente bigquery-writer) com schema validation, dedup e error logging | João | [M3](03-data-pipeline-process.md) | 2026-01-27 | ✓ Mantida (renomeada) | Escreve em S3 Iceberg via Athena |
| **D23** | **Diferentes prompts por tipo de invoice** (UberEats / DoorDash / Grubhub) versionados em código | Ana | [M3](03-data-pipeline-process.md), [M5](05-devops-infrastructure.md) | 2026-01-27 / 2026-02-10 | ✓ Mantida | Prompts são agnósticos de provider LLM |
| **D24** ★ | **GCP Secret Manager** para API keys e credenciais | Pedro | [M5](05-devops-infrastructure.md) | 2026-02-10 | ★ Substituída | **AWS Secrets Manager** (sensíveis) + **SSM Parameter Store** (config não-sensível) |
| **D25** ★ | Deploy inicial via **gcloud CLI** (rapid iteration), depois automatizar | João | [M5](05-devops-infrastructure.md) | 2026-02-10 | ★ Substituída | **AWS CLI / SAM CLI** inicial, depois Terraform/Terragrunt |
| **D26** | **GitHub Actions** para CI/CD pipelines | Pedro | [M5](05-devops-infrastructure.md) | 2026-02-10 | ✓ Mantida | Auth via OIDC + IAM Role (era Workload Identity Federation) |
| **D27** | **CodeRabbit** para AI-powered code review em PRs | Pedro | [M5](05-devops-infrastructure.md) | 2026-02-10 | ✓ Mantida | Cloud-agnostic |
| **D28** | **Feature branch workflow** com required PR reviews (humano + CodeRabbit) | Pedro | [M5](05-devops-infrastructure.md) | 2026-02-10 | ✓ Mantida | — |
| **D29** ★ | **Cloud Logging → GCS** export para CrewAI consumir | Pedro | [M6](06-autonomous-dataops.md) | 2026-02-17 | ★ Substituída | **CloudWatch Logs → Kinesis Firehose → S3** |
| **D30** | **CrewAI 3-agent architecture** (Triage / Root Cause / Reporter); **monitoring-only first**, auto-remediation Phase 2; alertas via **Slack #alerts-ops**; **weekly autonomous ops review** | João, Pedro, Marina | [M6](06-autonomous-dataops.md) | 2026-02-17 | ✓ Mantida | Comportamento dos agentes inalterado; agentes leem de S3 |

> **Nota de reconstrução:** A numeração D1–D30 foi reconstruída a partir das atas e está alinhada com as referências do [`07-aws-migration-plan.md`](07-aws-migration-plan.md) seção 5 (D6, D7, D8, D9, D10, D14, D15, D24, D25, D29). Validar com a equipe antes de citar em comunicação externa.

### 5.2 Lado a lado: original GCP vs. AWS atual

| # | Original (GCP) | Atual (AWS) | Documento |
|---|----------------|-------------|-----------|
| **D6** | GCP como nuvem primária; já uso BigQuery; Cloud Run nativo | AWS como nuvem primária; ecossistema serverless completo; isolamento via Organizations | [07 §2.1](07-aws-migration-plan.md#21-camada-de-storage-e-eventos), [07 §5](07-aws-migration-plan.md#5-decisões-atualizadas-substituem-gcp-specific) |
| **D7** | Pub/Sub com 4 tópicos (`invoice-uploaded`, `-converted`, `-classified`, `-extracted`) e DLQ por tópico | EventBridge para roteamento de eventos S3 + 4 SQS queues com DLQ por fila | [07 §2.1](07-aws-migration-plan.md#21-camada-de-storage-e-eventos), [07 §4.3](07-aws-migration-plan.md#43-triggers-e-fluxo-de-eventos) |
| **D8** | Cloud Run para todas as 4 funções; auto-scale to zero | AWS Lambda para todas as 4 funções; cold start mitigado com Provisioned Concurrency em prod (data-extractor); Fargate como fallback se TIFF→PNG ultrapassar 15min/10GB | [07 §2.2](07-aws-migration-plan.md#22-camada-de-compute), [07 §7 MR1, MR2](07-aws-migration-plan.md#7-riscos-e-mitigações-da-migração) |
| **D9** | BigQuery como warehouse SQL serverless | Athena + Iceberg sobre S3 (lakehouse). Tabela `extracted_invoices` particionada por `vendor_type` e `invoice_date`. Redshift Serverless é alternativa só se workload BI pesado | [07 §2.3](07-aws-migration-plan.md#23-camada-de-dados-data-warehouse) |
| **D14** | 2 GCP Projects (`invoice-pipeline-dev`, `invoice-pipeline-prod`) com IAM por projeto | 2 AWS Accounts via AWS Organizations (controle de billing por conta, isolamento mais forte) | [07 §2.5](07-aws-migration-plan.md#25-camada-de-segurança-e-configuração) |
| **D15** | Gemini 2.0 Flash via Vertex AI: 96.5% accuracy / 1.2s latency / $0.002 por invoice; 1M token context; native tool use | Claude Haiku 4.5 via Bedrock como primário (~94-95% projetado, ~0.8s, ~$0.001); Claude Sonnet 4.5 como fallback de qualidade quando Pydantic falhar (~96-97%, ~1.5s, ~$0.005). **Validação obrigatória sprint março/abril** | [07 §2.4](07-aws-migration-plan.md#24-camada-de-llm--ia), [07 §7 MR4](07-aws-migration-plan.md#7-riscos-e-mitigações-da-migração) |
| **D24** | GCP Secret Manager para tudo (API keys, tokens, webhooks) | AWS Secrets Manager (OpenRouter API key, LangFuse Secret Key, Slack Webhook URL) + SSM Parameter Store (LangFuse Public Key, configs não-sensíveis). **Bedrock NÃO precisa de API key** — usa IAM Role | [07 §2.5](07-aws-migration-plan.md#25-camada-de-segurança-e-configuração), [07 §4.6](07-aws-migration-plan.md#46-secrets) |
| **D25** | `gcloud run deploy ...` para iteração rápida no MVP | `aws lambda deploy` / `sam deploy` para iteração rápida no MVP | [07 §2.7](07-aws-migration-plan.md#27-camada-de-devops-e-cicd) |
| **D29** | Cloud Logging Sink → GCS → CrewAI lê de GCS | CloudWatch Logs → Kinesis Firehose → S3 → CrewAI lê de S3 | [07 §2.8](07-aws-migration-plan.md#28-camada-de-autonomous-ops-crewai), [07 §4.8](07-aws-migration-plan.md#48-crewai-autonomous-ops) |

---

## 6. Data Pipeline & Process

### 6.1 Fluxo end-to-end (AWS atual)

```
┌─────────┐
│ Vendor  │ uploads TIFF (multi-page) via API/SFTP
└────┬────┘
     ▼
┌──────────────────────────────────────┐
│  s3://invoices-input-{env}           │ ← raw TIFF (lifecycle 30d)
└────┬─────────────────────────────────┘
     │ S3 Event Notification
     ▼
┌──────────────┐
│ EventBridge  │
└────┬─────────┘
     ▼
┌──────────────┐                    ┌──────────────┐
│ SQS          │ ─── on failure ──▶ │ DLQ          │
│ uploaded     │                    │ uploaded-dlq │
└────┬─────────┘                    └──────────────┘
     ▼
┌────────────────────────────┐
│ Lambda 1: tiff-to-png      │ ← 2GB / 5min / Pillow
│ - converte TIFF → PNG/page │
│ - escreve em processed/    │
│ - copia original p/ archive│
│ - publica msg em SQS       │
└────┬───────────────────────┘
     ▼
┌──────────────┐
│ SQS          │ ─── DLQ ───▶ ...
│ converted    │
└────┬─────────┘
     ▼
┌────────────────────────────┐
│ Lambda 2: classifier       │ ← 512MB / 1min
│ - valida que é invoice     │
│ - identifica vendor_type   │
│ - checa qualidade da img   │
│ - se falha → failed/ + msg │
└────┬───────────────────────┘
     ▼
┌──────────────┐
│ SQS          │ ─── DLQ ───▶ ...
│ classified   │
└────┬─────────┘
     ▼
┌─────────────────────────────────────┐
│ Lambda 3: data-extractor            │ ← 1GB / 2min / boto3 Bedrock
│ - carrega prompt por vendor_type    │
│ - chama Bedrock Claude Haiku 4.5    │
│ - valida com Pydantic               │
│ - se valida falha → Sonnet 4.5      │
│ - se Sonnet falha → OpenRouter      │
│ - se tudo falha → failed/ + alert   │
│ - registra trace no LangFuse        │
└────┬────────────────────────────────┘
     ▼
┌──────────────┐
│ SQS          │ ─── DLQ ───▶ ...
│ extracted    │
└────┬─────────┘
     ▼
┌────────────────────────────────┐
│ Lambda 4: warehouse-writer     │ ← 512MB / 1min
│ - schema validation (final)    │
│ - dedup por invoice_id         │
│ - escreve em Iceberg via Athena│
│ - log de sucesso/erro          │
└────┬───────────────────────────┘
     ▼
┌──────────────────────────────┐
│ s3://invoices-warehouse/     │
│   extracted_invoices/        │ ← Iceberg, partitioned by
│   (queryable via Athena)     │   vendor_type, invoice_date
└──────────────────────────────┘
```

### 6.2 Schema de extração (12 campos)

> Schema final em [M4](04-data-ml-strategy.md) após Carlos pedir os campos de commission.

```json
{
  "invoice_id": "string",
  "vendor_name": "string",
  "vendor_type": "enum: ubereats|doordash|grubhub|other",
  "invoice_date": "date (ISO 8601)",
  "due_date": "date (ISO 8601)",
  "subtotal": "float",
  "tax_amount": "float",
  "commission_rate": "float",
  "commission_amount": "float",
  "total_amount": "float",
  "currency": "string (BRL|USD|...)",
  "line_items": [
    {
      "description": "string",
      "quantity": "int",
      "unit_price": "float",
      "amount": "float"
    }
  ]
}
```

> ⚠️ **Conflito flagged:** [M3](03-data-pipeline-process.md) listou 9 campos no schema inicial; [M4](04-data-ml-strategy.md) expandiu para 12 (adicionou `currency`, `commission_rate`, `commission_amount` por pedido do Carlos). A versão de 12 campos é a oficial.

### 6.3 Estratégia de prompts por tipo de invoice

| Vendor | Prompt template (resumo) | Owner | Status MVP |
|--------|-------------------------|-------|------------|
| UberEats | "Extract from UberEats invoice: order ID is in top right, restaurant name in header, ..." | Ana | Em desenvolvimento (due 2026-02-08, [M4](04-data-ml-strategy.md) AI) |
| DoorDash | "Extract from DoorDash invoice: order ID is at bottom, date format MM/DD/YYYY, ..." | Ana | Phase 2 |
| Grubhub | (TBD após validação UberEats e DoorDash) | Ana | Phase 3 |

**Princípio:** prompts são tratados como código, versionados no repo, deployados junto com a Lambda. Rollback de prompt = rollback de código ([D23](#decisões-d1d30), [M5](05-devops-infrastructure.md)).

### 6.4 Estratégia de erro, retry e DLQ

| Etapa | Falha esperada | Tratamento |
|---|---|---|
| Upload S3 | Arquivo corrompido | Validação básica no S3 Event handler (size > 0, extensão válida) |
| TIFF→PNG | Arquivo não-TIFF / multipage gigante | Move para `failed/`; Lambda timeout → SQS retry → DLQ após 3 tentativas |
| Classifier | Não é invoice / qualidade ruim | Move para `failed/`; envia notificação para review queue |
| Data extractor | LLM retorna garbage | **Pydantic rejeita** → retry com Sonnet 4.5 → retry com OpenRouter → SQS retry → DLQ |
| Warehouse writer | Schema validation final / dedup conflict | Log e move para `failed/`; Iceberg upserts evitam duplicação |
| Toda Lambda | Throttling / quota | Backoff exponencial via SQS visibility timeout |

---

## 7. Data & ML Strategy

### 7.1 Seleção do LLM (histórico e atual)

| Modelo | Provedor | Acurácia | P50 / P95 latência | Custo/invoice | Status |
|--------|----------|----------|--------------------|----|--------|
| Gemini 2.0 Flash | Vertex AI | 96.5% | 0.9s / 1.8s | $0.002 | ★ Substituído (era D15) |
| GPT-4o Vision | Azure OpenAI | 97.1% | 1.8s / 3.5s | $0.012 | Rejeitado (custo) |
| Claude 3.5 Sonnet | OpenRouter | 95.8% | 1.4s / 2.6s | $0.006 | Mantido como referência |
| **Claude Haiku 4.5** | **Bedrock** | ~94-95% (projetado) | ~0.8s | ~$0.001 | ✅ **Primário (atual)** |
| **Claude Sonnet 4.5** | **Bedrock** | ~96-97% (projetado) | ~1.5s | ~$0.005 | ✅ **Fallback de qualidade** |
| Amazon Nova Pro | Bedrock | TBD | ~1.0s | ~$0.003 | Em avaliação |

> ⚠️ **Risco MR4** ([07 §7](07-aws-migration-plan.md#7-riscos-e-mitigações-da-migração)): As estimativas de Claude Haiku/Sonnet são **projeções**, não benchmarks validados. **Sprint de validação obrigatória** entre 1–10 junho 2026 com dataset real do Carlos.

### 7.2 Fluxo de validação Pydantic ([D18](#decisões-d1d30))

```python
# pseudocódigo
response = bedrock.invoke_model(
    model="anthropic.claude-haiku-4-5",
    prompt=prompt_template[vendor_type],
    image=png_bytes
)

try:
    extracted = ExtractedInvoice.model_validate_json(response.content)
except ValidationError as e:
    # fallback de qualidade
    response = bedrock.invoke_model(
        model="anthropic.claude-sonnet-4-5",
        prompt=prompt_template[vendor_type],
        image=png_bytes
    )
    try:
        extracted = ExtractedInvoice.model_validate_json(response.content)
    except ValidationError:
        # último recurso
        response = openrouter.invoke(...)
        extracted = ExtractedInvoice.model_validate_json(response.content)

langfuse.log(trace=trace_id, prompt=..., response=..., latency=..., cost=...)
```

### 7.3 Observabilidade LLM (LangFuse, [D17](#decisões-d1d30))

| Métrica | Target | Alert threshold |
|---------|--------|-----------------|
| Cost per extraction | < $0.005 | > $0.01 |
| Latency P95 | < 3s | > 5s |
| Overall accuracy | > 90% | < 85% |
| Validation failures (Pydantic) | < 5% | > 10% |

### 7.4 Dados de teste

| Tipo | Origem | Owner | Tamanho inicial |
|------|--------|-------|-----------------|
| Real (50 invoices) | Carlos provê de produção | Carlos (M1 AI, due 2026-01-17; ground truth labeling due 2026-02-10) | 50 |
| Sintético | João implementa generator com formatos variados, fontes, ruído de scan | João (M4 AI, due 2026-02-10) | 1.000+ |
| Validação cross-cloud (Bedrock vs Gemini) | Sprint específica | Ana | TBD (sprint junho 2026) |

---

## 8. DevOps & Infrastructure

### 8.1 IaC — Terraform + Terragrunt ([D11](#decisões-d1d30))

**Estrutura esperada (AWS, conforme [CLAUDE.md](../CLAUDE.md)):**

```
infrastructure/
├── modules/
│   ├── lambda/          # (era cloud-run/)
│   ├── eventbridge/     # (era pubsub/ — roteamento)
│   ├── sqs/             # (era pubsub/ — filas)
│   ├── s3/              # (era gcs/)
│   ├── athena-iceberg/  # (era bigquery/)
│   ├── bedrock/         # (era vertex-ai/)
│   └── iam/             # (mantido, nova implementação)
├── environments/
│   ├── dev/terragrunt.hcl
│   └── prod/terragrunt.hcl
└── terragrunt.hcl
```

**Provider:** `aws >= 5.x`

### 8.2 CI/CD ([D26, D27, D28](#decisões-d1d30))

```
Feature branch
   ↓ open PR
GitHub Actions: lint + test + security scan
   ↓
CodeRabbit AI review
   ↓
Human reviewer approval
   ↓
Merge to main
   ↓
Deploy to dev (auto, via OIDC + IAM Role)
   ↓
Manual approval gate
   ↓
Deploy to prod (via OIDC + IAM Role)
```

**Auth:** OIDC + IAM Role (substituiu Workload Identity Federation). Secrets `GCP_SA_KEY` removidos.

### 8.3 Secrets ([D24](#decisões-d1d30))

| Secret | Tipo | Storage | Acesso |
|--------|------|---------|--------|
| OpenRouter API Key | Sensível | AWS Secrets Manager | Lambda data-extractor (IAM Role) |
| LangFuse Public Key | Não-sensível | SSM Parameter Store | Todas Lambdas (IAM Role) |
| LangFuse Secret Key | Sensível | AWS Secrets Manager | Todas Lambdas (IAM Role) |
| Slack Webhook URL | Sensível | AWS Secrets Manager | CrewAI Reporter Agent (IAM Role) |
| Bedrock | (não há key) | — | Lambda data-extractor via IAM Role `bedrock:InvokeModel` |

**Princípio:** zero secrets em código, zero secrets em env vars de deploy. Função busca em runtime na primeira cold start.

### 8.4 Observabilidade

| Camada | Ferramenta AWS | Substitui (GCP) |
|--------|----------------|-----------------|
| Logs estruturados | CloudWatch Logs + `aws-lambda-powertools` (Python) | Cloud Logging |
| Métricas custom | CloudWatch Embedded Metric Format (EMF) | Cloud Monitoring |
| Distributed tracing | AWS X-Ray | Cloud Trace |
| Dashboard | CloudWatch Dashboard (via Terraform) | Cloud Monitoring Dashboard |
| Alertas | CloudWatch Alarms → SNS → Slack | Cloud Monitoring Alerts |
| Logs para CrewAI | CloudWatch Logs → Kinesis Firehose → S3 | Log Sink → GCS |
| LLM observability | LangFuse (cloud-agnostic, mantido) | (mesmo) |

### 8.5 IAM least-privilege por Lambda ([07 §4.5](07-aws-migration-plan.md#45-permissões-iam-least-privilege))

| Lambda | Permissões mínimas |
|---|---|
| `tiff-to-png-converter` | `s3:GetObject` (input), `s3:PutObject` (processed/archive), `sqs:SendMessage` (converted) |
| `invoice-classifier` | `s3:GetObject` (processed), `sqs:ReceiveMessage` (converted), `sqs:SendMessage` (classified) |
| `data-extractor` | `s3:GetObject` (processed), `sqs:*` (classified/extracted), `bedrock:InvokeModel`, `secretsmanager:GetSecretValue` (OpenRouter) |
| `warehouse-writer` | `sqs:ReceiveMessage` (extracted), `s3:PutObject` (warehouse), `glue:*` (Iceberg metadata), `athena:*` |

### 8.6 Multi-account isolation ([D14](#decisões-d1d30))

- 2 AWS Accounts via AWS Organizations: **invoice-pipeline-dev** e **invoice-pipeline-prod**
- Billing separado, blast radius isolado
- Promotion gate: aprovação manual no GitHub Actions

### 8.7 Estimativa de custos AWS ([07 §6](07-aws-migration-plan.md#6-estimativa-de-custos-aws))

| Componente | Dev (~$/mês) | Prod (~$/mês) |
|------------|--------------|----------------|
| Lambda | $2 | $5 |
| SQS + DLQ | $1 | $2 |
| S3 (com Glacier para archive em prod) | $1 | $5 |
| Athena | $2 | $5 |
| EventBridge | <$1 | $1 |
| Bedrock Haiku | $2 | $4 |
| Bedrock Sonnet (fallback ~5%) | — | $2 |
| Secrets Manager | $2 | $2 |
| CloudWatch + Firehose + X-Ray | $3 | $10 |
| **Total** | **~$15** | **~$36** |

> Comparação: estimativas GCP eram ~$20 dev / ~$55 prod. AWS é levemente mais barato no volume atual.

---

## 9. Autonomous DataOps (CrewAI) — [D30](#decisões-d1d30)

### 9.1 Arquitetura de 3 agentes

```
CloudWatch Logs (todas Lambdas)
    ↓ Log Sink
Kinesis Firehose
    ↓
S3 (logs bucket)
    ↓
┌─────────────────────────────────────────────────────────────────┐
│                       CrewAI Pipeline                            │
│                                                                  │
│  ┌─────────────┐    ┌─────────────────┐    ┌─────────────────┐ │
│  │   TRIAGE    │───▶│   ROOT CAUSE    │───▶│    REPORTER     │ │
│  │   AGENT     │    │     AGENT       │    │     AGENT       │ │
│  ├─────────────┤    ├─────────────────┤    ├─────────────────┤ │
│  │ Lê logs do  │    │ Analisa logs +  │    │ Format report   │ │
│  │ S3          │    │ métricas CW     │    │ humano-readable │ │
│  │ Classifica  │    │ Detecta padrão  │    │ Envia p/ Slack  │ │
│  │ severidade: │    │ (ex: 5 erros    │    │ Track resolução │ │
│  │ INFO/WARN/  │    │  similares em   │    │                 │ │
│  │ ERROR/CRIT  │    │  1h)            │    │                 │ │
│  │ Filtra      │    │ Sugere fix      │    │                 │ │
│  └─────────────┘    └─────────────────┘    └─────────────────┘ │
│                                                                  │
│  Tools: S3 Reader, Log Parser, CW Metrics Query, Pattern Match,  │
│         LLM Analysis, Slack Sender                               │
└─────────────────────────────────────────────────────────────────┘
    ↓
Slack #alerts-ops
    Ex: "🔴 ERROR: Bedrock extraction failures
         Root Cause: Multi-page invoice handling bug
         Affected: 5 invoices in last hour
         Suggested Fix: Check prompt template for page handling
         Escalation: @ana @joao"
```

### 9.2 Princípios

1. **Phase 1: monitoring-only** — agentes só sugerem, não fixam (build trust)
2. **Phase 2: auto-remediation** — apenas para issues conhecidos (ex: retry com backoff)
3. **Guardrails:** maximum retries, circuit breaker, human-in-the-loop para qualquer ação que modifique dados, audit log de todas as ações
4. **Hosting:** Lambda para execuções curtas (scheduled via EventBridge Scheduler) ou Fargate para longas
5. **Cadence:** weekly autonomous ops review (Marina)

### 9.3 Exemplo de fluxo (cenário real)

> Citado verbatim de [M6](06-autonomous-dataops.md):

1. Bedrock retorna JSON malformado em invoice multi-page
2. Lambda data-extractor loga erro no CloudWatch
3. Firehose exporta para S3 em ~1 minuto
4. Triage Agent: classifica como ERROR, encaminha
5. Root Cause Agent: detecta 5 erros similares na última hora, conclui "Bedrock retorna JSON malformado para invoices multi-page"
6. Reporter Agent: envia para Slack com sugestão de fix e escalação para Ana e João

---

## 10. Action Items (consolidado, abertos e pendentes)

> Itens marcados como **(Original)** vêm das atas 01–06; itens **(Migração)** são novos do plano AWS. Status presumido "open" exceto onde o documento original explicita conclusão.

### 10.1 Setup inicial e adapters (Pedro / João)

- [ ] **Pedro Lima** — Provisionar AWS Organizations + 2 contas (dev/prod) + OIDC GitHub Identity Provider (Due: ~2026-05-15, Source: [07 §10](07-aws-migration-plan.md#10-próximos-passos)) **(Migração)**
- [ ] **Pedro Lima** — Configurar IAM baseline + Role assumível por GitHub Actions com trust policy (Due: 2026-05-15, Source: [07](07-aws-migration-plan.md)) **(Migração)**
- [ ] **João Silva** — Implementar `S3Adapter`, `SqsAdapter`, `EventBridgeAdapter`, `BedrockAdapter`, `AthenaAdapter`, `AwsSecretsManagerAdapter` (Due: 2026-05-22, Source: [07 §4.1](07-aws-migration-plan.md#41-adapters-já-previstos-no-d10)) **(Migração)**
- [ ] **Pedro Lima** — Reescrever módulos Terraform: `lambda/`, `sqs/`, `eventbridge/`, `s3/`, `athena-iceberg/`, `bedrock/`, `iam/` (Due: 2026-05-25, Source: [07 §4.2](07-aws-migration-plan.md#42-infraestrutura-como-código)) **(Migração)**

### 10.2 Migração de funções

- [ ] **João Silva** — Migrar `tiff-to-png-converter` para Lambda (2GB / 5min / Pillow), trigger SQS (Due: 2026-06-05, Source: [07 §8](07-aws-migration-plan.md#8-cronograma-de-migração-sem-impacto-no-deadline)) **(Migração)**
- [ ] **Ana Costa** — Migrar `invoice-classifier` para Lambda (Due: 2026-06-05) **(Migração)**
- [ ] **Ana Costa** — Migrar `data-extractor` para Lambda + boto3 Bedrock (Claude Haiku 4.5 primary, Sonnet 4.5 fallback, OpenRouter secondary) (Due: 2026-06-05) **(Migração)**
- [ ] **João Silva** — Migrar `bigquery-writer` → `warehouse-writer` (Lambda + Athena Iceberg) (Due: 2026-06-05) **(Migração)**

### 10.3 Validação ML

- [ ] **Ana Costa** — Sprint de validação Bedrock Claude Haiku/Sonnet 4.5 vs Gemini 2.0 Flash usando dataset real do Carlos (Due: 2026-06-10, Source: [07 §8 + §10](07-aws-migration-plan.md#10-próximos-passos)) **(Migração — crítico para MR4)**
- [ ] **Ana Costa** — Escrever prompt templates para UberEats invoices (Due: 2026-02-08, Source: [M4](04-data-ml-strategy.md)) **(Original — provavelmente concluído, validar)**
- [ ] **João Silva** — Construir invoice generator sintético + ground truth CSV format (Due: 2026-02-10, Source: [M4](04-data-ml-strategy.md)) **(Original — validar status)**
- [ ] **Carlos Ferreira** — Aprovar extraction schema com 12 campos incluindo commission (Due: 2026-02-06, Source: [M4](04-data-ml-strategy.md)) **(Original — provavelmente concluído)**
- [ ] **Marina Santos** — Definir thresholds de acurácia por campo (go/no-go) (Due: 2026-02-07, Source: [M4](04-data-ml-strategy.md)) **(Original — validar)**

### 10.4 Observabilidade

- [ ] **Ana Costa** — Configurar projeto LangFuse e integração nas Lambdas (Due: 2026-02-07 originalmente; revalidar para nova stack, Source: [M4](04-data-ml-strategy.md)) **(Original)**
- [ ] **Pedro Lima** — Configurar CloudWatch → Kinesis Firehose → S3 export (substitui Cloud Logging → GCS) (Due: 2026-06-15, Source: [07](07-aws-migration-plan.md)) **(Migração)**
- [ ] **Pedro Lima** — Configurar CloudWatch Dashboard via Terraform + CloudWatch Alarms → SNS → Slack (Due: ~2026-06-15) **(Migração)**

### 10.5 CrewAI

- [ ] **João Silva** — Migrar CrewAI para ler logs de S3 (substituir leitor GCS) (Due: 2026-06-15, Source: [07 §10](07-aws-migration-plan.md#10-próximos-passos)) **(Migração)**
- [ ] **João Silva** — Setup CrewAI project structure (Due: 2026-02-21, Source: [M6](06-autonomous-dataops.md)) **(Original)**
- [ ] **João Silva** — Definir prompts e capabilities dos 3 agentes (Due: 2026-02-24, Source: [M6](06-autonomous-dataops.md)) **(Original)**
- [ ] **Ana Costa** — Definir error patterns para Triage Agent (Due: 2026-02-22, Source: [M6](06-autonomous-dataops.md)) **(Original)**
- [ ] **Marina Santos** — Atualizar runbook de escalação com endpoints AWS (Due: ~2026-06-15, Source: [07 §10](07-aws-migration-plan.md#10-próximos-passos)) **(Migração; original era runbook de escalação genérico, M6)**

### 10.6 Segurança e go-live

- [ ] **Pedro Lima** — Security review específico para AWS antes do production launch (Due: pré-cutover ~2026-06-18, Source: [07 §10](07-aws-migration-plan.md#10-próximos-passos)) **(Migração — substitui security review GCP)**
- [ ] **Carlos Ferreira** — Coordenar com legal sobre data handling para invoices de produção (Due: TBD, Source: [M6](06-autonomous-dataops.md)) **(Original)**
- [ ] **Marina Santos** — Production cutover para AWS (Due: 2026-06-15 a 2026-06-20, Source: [07 §8](07-aws-migration-plan.md#8-cronograma-de-migração-sem-impacto-no-deadline)) **(Migração)**

### 10.7 Itens originais cuja conclusão deve ser validada

| Item | Owner | Due original | Validação |
|---|---|---|---|
| Initial GCP project setup | João | 2026-01-17 | **N/A — supersedeed pela migração** |
| Sample dataset 50 invoices | Carlos | 2026-01-17 | Status TBD |
| Success criteria document | Marina | 2026-01-18 | Status TBD |
| LLM options research | Ana | 2026-01-20 | Concluído (M4 trouxe resultados) |
| Infrastructure requirements eval | Pedro | 2026-01-22 | Concluído (M2/M5) |
| Architecture diagram | João | 2026-01-24 | Concluído (M2 contém diagrama) |
| Adapter Pattern interface | João | 2026-01-27 | Concluído (D10) |
| Pub/Sub setup | Pedro | 2026-02-01 | **N/A — supersedeed (será SQS)** |
| All 4 functions implementations | João/Ana | 2026-02-03 a 2026-02-07 | Status TBD; deve ser revalidado para Lambda |
| GCP dev project | Pedro | 2026-02-13 | **N/A — supersedeed** |
| GCP Secret Manager setup | Pedro | 2026-02-14 | **N/A — supersedeed** |
| GitHub Actions config | Pedro | 2026-02-17 | Em revisão para OIDC |
| CodeRabbit setup | Pedro | 2026-02-14 | Status TBD |
| Cloud Logging → GCS export | Pedro | 2026-02-20 | **N/A — supersedeed** |
| Slack webhook setup | Pedro | 2026-02-21 | Cloud-agnostic, validar |

---

## 11. Open Questions

| # | Questão | Origem | Owner sugerido |
|---|---------|--------|----------------|
| OQ1 | O que acontece quando classificação falha? Tem UI de review queue? | [M3](03-data-pipeline-process.md) | Marina + Carlos |
| OQ2 | Qual o budget total do projeto (não foi discutido em M1)? | [M1](01-business-kickoff.md) (notas) | Marina |
| OQ3 | Quanto da migração já foi executado vs. apenas planejado? | [07](07-aws-migration-plan.md) | Pedro |
| OQ4 | Qual a estratégia para dados de produção (PII / compliance)? | [M6](06-autonomous-dataops.md) | Carlos + Legal |
| OQ5 | Numeração D1–D30 desta SSOT bate com a que existia anteriormente? | Reconstruído | Marina |
| OQ6 | Em qual região AWS será deployado (us-east-1?)? | [07](07-aws-migration-plan.md) (não especifica) | Pedro |
| OQ7 | Como exatamente será o gate de promotion dev→prod (manual approval no GH Actions com qual aprovador)? | [M5](05-devops-infrastructure.md) | Marina + Pedro |
| OQ8 | Acurácia projetada de Claude Haiku 4.5 (~94-95%) é validada ou estimada? | [07 §2.4](07-aws-migration-plan.md#24-camada-de-llm--ia) | Ana (sprint junho 2026) |
| OQ9 | Há SLA contratual com restaurantes parceiros que afete NFR1/NFR3? | Não discutido | Carlos |
| OQ10 | Como tratar invoices em moeda não-BRL (currency field foi adicionado mas não há regra de conversão)? | [M4](04-data-ml-strategy.md) | Carlos + Ana |

---

## 12. Risks & Blockers

### 12.1 Riscos técnicos

| # | Risco | Probabilidade | Impacto | Mitigação | Origem |
|---|-------|---------------|---------|-----------|--------|
| R1 | Acurácia <90% em production data | Média | Alto | Sprint de validação março 2026 com dataset real; fallback Sonnet 4.5; revisão de prompts | [M6](06-autonomous-dataops.md), [M4](04-data-ml-strategy.md) |
| R2 | Lambda timeout em TIFFs grandes multi-page | Média | Médio | 15min + 10GB RAM; fallback Fargate Tasks | [07 MR1](07-aws-migration-plan.md#7-riscos-e-mitigações-da-migração) |
| R3 | Lambda cold start elevando latência P95 | Baixa | Médio | Provisioned Concurrency em prod para data-extractor | [07 MR2](07-aws-migration-plan.md#7-riscos-e-mitigações-da-migração) |
| R4 | Bedrock quota/throttling em região | Média | Alto | Cross-region inference profiles; fallback OpenRouter | [07 MR3](07-aws-migration-plan.md#7-riscos-e-mitigações-da-migração) |
| R5 | Diferenças de qualidade Claude vs Gemini | Média | Médio | Sprint dedicada de validação | [07 MR4](07-aws-migration-plan.md#7-riscos-e-mitigações-da-migração) |
| R6 | Curva de aprendizado AWS (time vinha de GCP) | Média | Médio | Pareamento Pedro+João nas primeiras Lambdas; documentação interna | [07 MR5](07-aws-migration-plan.md#7-riscos-e-mitigações-da-migração) |
| R7 | Athena/Iceberg vs BigQuery — gap em features SQL | Baixa | Baixo | Validar queries do BI antes de cutover | [07 MR6](07-aws-migration-plan.md#7-riscos-e-mitigações-da-migração) |
| R8 | Egress costs se algum dado ficar fora da AWS (ex: LangFuse) | Baixa | Baixo | Manter LangFuse em região próxima; minimizar transferência | [07 MR7](07-aws-migration-plan.md#7-riscos-e-mitigações-da-migração) |

### 12.2 Riscos de cronograma e organizacionais

| # | Risco | Mitigação |
|---|-------|-----------|
| R9 | **Conflito de timeline:** [D5](#decisões-d1d30) define production launch em 1 de abril de 2026 (Q2 close); [07 §8](07-aws-migration-plan.md#8-cronograma-de-migração-sem-impacto-no-deadline) prevê cutover AWS para 15–20 de junho de 2026 — **gap de 2,5 meses** | ⚠️ **Resolução obrigatória** com Marina/Carlos: rodar GCP MVP até Q2 close + migrar AWS depois? OU reescrever migration timeline? |
| R10 | Carlos Ferreira é único business champion — bus factor 1 | Engajar segundo stakeholder do Restaurant Operations |
| R11 | Security review pode bloquear go-live | Agendar antecipadamente com security team |

### 12.3 Blockers ativos

- 🔴 **B1:** Aprovação executiva da migração para AWS pendente ([07 §10 step 1](07-aws-migration-plan.md#10-próximos-passos))
- 🔴 **B2:** Conflito timeline R9 não resolvido — afeta planejamento de todos os streams
- 🟡 **B3:** OQ4 (compliance de dados PII) não tem owner formal de Legal designado

---

## 13. Timeline & Milestones

### 13.1 Cronograma original (planejado em GCP)

| Fase | Datas | Deliverable | Status |
|------|-------|-------------|--------|
| MVP | Feb 14–28, 2026 | Pipeline básico (4 Cloud Run functions), dev deploy | TBD (validar) |
| Testing | Mar 1–15, 2026 | LangFuse integration, accuracy validation | TBD |
| Launch | Mar 16–Apr 1, 2026 | **Production launch GCP** | ⚠️ Em conflito com R9 |
| AutoOps | Apr 1–30, 2026 | CrewAI agentes em produção | TBD |

### 13.2 Cronograma de migração AWS ([07 §8](07-aws-migration-plan.md#8-cronograma-de-migração-sem-impacto-no-deadline))

| Fase | Datas | Deliverable |
|------|-------|-------------|
| Setup AWS | 11–15 maio 2026 | AWS Organizations, OIDC, IAM baseline |
| Adapters AWS | 16–22 maio 2026 | S3/Sqs/Bedrock adapters implementados e testados |
| Terraform AWS | 16–25 maio 2026 | Módulos reescritos, dev environment provisionado |
| Migração 4 Lambdas | 26 maio – 5 junho 2026 | E2E test passando em dev |
| Validação Bedrock vs Gemini | 1–10 junho 2026 | Sprint Ana — comparativo de acurácia |
| CrewAI no AWS | 8–15 junho 2026 | Agentes lendo de S3, alertas no Slack |
| **Production launch (AWS)** | **15–20 junho 2026** | Cutover prod AWS |

### 13.3 Marcos críticos

- 🚨 **2026-04-01** — Production launch (deadline original Q2 close) — conflito ativo com R9
- 🚨 **2026-04-30** — Q2 close — não pode escapar
- 🟢 **2026-06-20** — Cutover AWS final (per migration plan)

---

## 14. Glossário & Referências

### 14.1 Termos

| Termo | Significado |
|-------|-------------|
| **Adapter Pattern** | Padrão de design que abstrai serviços de cloud atrás de interfaces, permitindo trocar implementações sem alterar lógica de negócio |
| **DLQ** | Dead Letter Queue — fila para mensagens que falharam após N retentativas |
| **Iceberg** | Open table format (Apache) que dá ACID, schema evolution e time travel sobre arquivos Parquet em object storage |
| **Lakehouse** | Arquitetura que combina data lake (S3) com features de data warehouse (transactions via Iceberg) |
| **OIDC** | OpenID Connect — protocolo de federação que permite GitHub Actions assumir IAM Role sem long-lived AWS keys |
| **Pydantic** | Library Python para validação de dados via type hints; usada para validar saída do LLM contra schema estrito |
| **SSOT** | Single Source of Truth |
| **Terragrunt** | Wrapper sobre Terraform para gestão de múltiplos environments com DRY |
| **Workload Identity Federation** | Equivalente OIDC do GCP — substituído por OIDC + IAM Role na migração |

### 14.2 Documentos de referência

| Documento | Tipo | Status |
|-----------|------|--------|
| [01-business-kickoff.md](01-business-kickoff.md) | Ata original (2026-01-15) | Histórico — design GCP |
| [02-technical-architecture.md](02-technical-architecture.md) | Ata original (2026-01-22) | Histórico — design GCP |
| [03-data-pipeline-process.md](03-data-pipeline-process.md) | Ata original (2026-01-27) | Histórico — design GCP |
| [04-data-ml-strategy.md](04-data-ml-strategy.md) | Ata original (2026-02-03) | Histórico — design GCP |
| [05-devops-infrastructure.md](05-devops-infrastructure.md) | Ata original (2026-02-10) | Histórico — design GCP |
| [06-autonomous-dataops.md](06-autonomous-dataops.md) | Ata original (2026-02-17) | Histórico — design GCP |
| [07-aws-migration-plan.md](07-aws-migration-plan.md) | **Plano de migração ativo** (2026-05-10) | **AUTORITATIVO para decisões de stack** |
| [../CLAUDE.md](../CLAUDE.md) | Contexto persistente do projeto | Autoritativo para regras do dia-a-dia |

### 14.3 Itens deprecated (não usar em novo código)

> Conforme [CLAUDE.md](../CLAUDE.md):

- SDKs `google-cloud-*` (storage, pubsub, bigquery, aiplatform, secret-manager, logging, monitoring)
- `gcloud` CLI commands em scripts/docs novos
- Módulos Terraform com provider `google` / `google-beta`
- Workload Identity Federation no GitHub Actions
- URIs `gs://...` (substituir por `s3://...`)
- Variáveis com `GCP_`, `GCS_`, `BIGQUERY_`, `VERTEX_`, `PUBSUB_` em código novo

---

## 15. Confidence & Provenance

### 15.1 Confiança por seção

| Seção | Confiança | Justificativa |
|-------|-----------|---------------|
| 1. Executive Summary | 0.95 | Dados explícitos em todas as atas |
| 2. Stakeholders & RACI | 0.95 | Todos nomeados em todas as atas |
| 3. Business Requirements | 0.92 | FRs/NFRs explicitamente discutidos; alguns inferidos |
| 4. Architecture Overview | 0.95 | Diagramas explícitos em M2, M3, M6 + 07 |
| 5. Decisões D1–D30 | 0.85 | Numeração reconstruída — alinhada com 07 mas não com SSOT anterior; validar com equipe (OQ5) |
| 6. Data Pipeline & Process | 0.93 | Fluxo explícito em M3 + 07; conflito de campos (9 vs 12) flagged e resolvido |
| 7. Data & ML Strategy | 0.88 | Benchmarks Gemini validados (M4); Bedrock projeções não validadas (R5/MR4) |
| 8. DevOps & Infrastructure | 0.95 | Estrutura explícita em M5 + 07 + CLAUDE.md |
| 9. Autonomous DataOps | 0.95 | Diagrama e fluxo de exemplo verbatim em M6 |
| 10. Action Items | 0.85 | Originais possivelmente desatualizados (não há tracking de conclusão) |
| 11. Open Questions | 0.90 | Questões reais identificadas durante consolidação |
| 12. Risks & Blockers | 0.92 | Riscos do plano AWS explícitos; conflito de cronograma R9 inferido mas robusto |
| 13. Timeline | 0.80 | Conflito ativo entre cronogramas original e migração — requer resolução |
| 14. Glossário | 0.95 | Termos definidos a partir das atas |
| **Geral** | **0.93** | |

### 15.2 Conflitos flagged

| # | Conflito | Resolução proposta |
|---|----------|--------------------|
| C1 | Schema 9 campos (M3) vs 12 campos (M4) | Resolvido: 12 campos é a versão final; ver §6.2 |
| C2 | Production launch 1 abr 2026 (D5) vs cutover AWS 15–20 jun 2026 ([07 §8](07-aws-migration-plan.md#8-cronograma-de-migração-sem-impacto-no-deadline)) | **Não resolvido** — requer decisão Marina+Carlos (R9, B2) |
| C3 | Numeração D1–D30 reconstruída desta SSOT pode divergir da numeração da SSOT anterior referenciada em [07 §5](07-aws-migration-plan.md#5-decisões-atualizadas-substituem-gcp-specific) | Validar com equipe (OQ5); a referência crítica é a substituição, não o número |
| C4 | "Implementation status" da maioria dos AIs originais é desconhecido | Recomenda-se uma sessão de status review antes de iniciar a Sprint AWS |

### 15.3 Provenance

Esta SSOT foi **reconstruída** em 2026-05-11 a partir de:
- Atas brutas: 6 documentos em `notes/01-*.md` a `notes/06-*.md`
- Plano de migração: `notes/07-aws-migration-plan.md`
- Contexto do projeto: `CLAUDE.md` (raiz)

Não havia uma `summary-requirements.md` prévia disponível no checkout (apesar de [07](07-aws-migration-plan.md) referenciá-la como base). Esta versão é a **primeira persistida** sob esse nome.

**Próxima revisão recomendada:** após resolução de B1 (aprovação executiva), B2 (conflito de timeline) e sprint de validação Bedrock (Ana, 1–10 junho 2026).

---

> **"Every meeting contains decisions waiting to be discovered."**
> — Meeting Analyst Mission

*Última atualização: 2026-05-11 por meeting-analyst (consolidação inicial)*
