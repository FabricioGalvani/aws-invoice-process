# Plano de Migração para AWS: Invoice Processing Pipeline

> **Gerado:** 2026-05-10
> **Base:** Notas 01 a 06 + summary-requirements.md
> **Objetivo:** Reimplementar o pipeline originalmente desenhado para GCP usando o ecossistema AWS, mantendo todos os requisitos funcionais, não-funcionais e o cronograma do projeto.
> **Viabilidade:** ALTA — o design original já previu portabilidade via Adapter Pattern (decisão D10).

---

## 1. Resumo Executivo

| Aspecto | Detalhe |
|---|---|
| **Decisão central** | Substituir GCP por AWS como nuvem primária |
| **Habilitador chave** | Adapter Pattern já decidido (StorageAdapter, MessagingAdapter, LLMAdapter) |
| **Impacto na lógica de negócio** | Nenhum — apenas troca de implementações de interface |
| **Impacto no cronograma** | Neutro — mantém deadline de 1 de abril de 2026 (Q2 close) |
| **Impacto em ferramentas agnósticas** | Zero — Terraform/Terragrunt, GitHub Actions, CodeRabbit, LangFuse, CrewAI, Pydantic, Slack permanecem |
| **Custo estimado** | Comparável ou levemente inferior a GCP (~$20/mês dev, ~$55/mês prod) |

---

## 2. Mapeamento de Serviços GCP → AWS

### 2.1 Camada de Storage e Eventos

| GCP (original) | AWS (substituto) | Justificativa |
|---|---|---|
| Google Cloud Storage (4 buckets) | **Amazon S3** (4 buckets) | Mapeamento 1:1; eventos S3 nativos via S3 Event Notifications |
| Pub/Sub (4 tópicos) | **Amazon EventBridge + SQS** | EventBridge para roteamento de eventos S3; SQS para filas duráveis com retry e DLQ entre funções |
| GCS Buckets (input/processed/archive/failed) | S3 Buckets equivalentes | Lifecycle Policies do S3 substituem Retention Policies do GCS |

**Bucket structure proposta:**

```
s3://invoices-input-{env}        # Raw TIFF (Lifecycle: 30 days)
s3://invoices-processed-{env}    # PNGs convertidos (Lifecycle: 90 days)
s3://invoices-archive-{env}      # Originais para compliance (Lifecycle: 7 years - Glacier)
s3://invoices-failed-{env}       # Failed processing review queue
```

### 2.2 Camada de Compute

| GCP (original) | AWS (substituto) | Justificativa |
|---|---|---|
| Cloud Run (4 funções) | **AWS Lambda** (4 funções) | Equivalente serverless. Atende todas as 4 etapas do pipeline |
| Cloud Run com tempo > 15min | **AWS Fargate / ECS** ou **AWS Batch** | Fallback se TIFF→PNG ultrapassar limites de Lambda (15min, 10GB RAM) |

**Funções Lambda equivalentes:**

| Função | Runtime | Memory | Timeout | Trigger |
|---|---|---|---|---|
| `tiff-to-png-converter` | Python 3.12 + Pillow | 2048 MB | 5 min | SQS `invoice-uploaded` |
| `invoice-classifier` | Python 3.12 | 512 MB | 1 min | SQS `invoice-converted` |
| `data-extractor` | Python 3.12 + boto3 (Bedrock) | 1024 MB | 2 min | SQS `invoice-classified` |
| `bigquery-writer` → `warehouse-writer` | Python 3.12 | 512 MB | 1 min | SQS `invoice-extracted` |

### 2.3 Camada de Dados (Data Warehouse)

| GCP (original) | AWS (substituto) | Recomendação |
|---|---|---|
| BigQuery | **Athena + S3 (Iceberg/Parquet)** | **Recomendado** para o volume (2-3.5k invoices/mês). Custo mínimo, pay-per-query, lakehouse-friendly |
| BigQuery | Amazon Redshift Serverless | Alternativa se houver workload BI pesado e necessidade de SQL warehouse tradicional |

**Decisão recomendada:** Athena + Iceberg sobre S3.

```
s3://invoices-warehouse-{env}/
  └── extracted_invoices/    # Iceberg table partitioned by vendor_type, invoice_date
```

### 2.4 Camada de LLM / IA

| GCP (original) | AWS (substituto) | Justificativa |
|---|---|---|
| Gemini 2.0 Flash (Vertex AI) | **Amazon Bedrock — Claude Sonnet 4.5 ou Haiku 4.5** | Equivalente nativo AWS. Claude 3.5 Sonnet já testado em 95.8% no benchmark da Ana (D15 reavaliado) |
| OpenRouter (fallback) | **OpenRouter mantido** ou **Bedrock multi-model** | Fallback continua via OpenRouter; Bedrock também oferece Nova, Llama, Mistral como alternativas |
| Vertex AI SDK | **boto3 (`bedrock-runtime`)** ou **Anthropic SDK direto** | SDK nativo AWS via IAM Role da Lambda |

**Reavaliação do LLM primário:**

| Modelo | Provedor | Acurácia (benchmark Ana) | Latência P50 | Custo/Invoice |
|---|---|---|---|---|
| Claude Sonnet 4.5 | Bedrock | ~96-97% (estimado) | ~1.5s | ~$0.005 |
| Claude Haiku 4.5 | Bedrock | ~94-95% (estimado) | ~0.8s | ~$0.001 |
| Amazon Nova Pro | Bedrock | TBD (testar) | ~1.0s | ~$0.003 |

**Recomendação:** Iniciar com **Claude Haiku 4.5** (custo/latência) e cair para **Claude Sonnet 4.5** apenas em casos onde o validator Pydantic falhar.

### 2.5 Camada de Segurança e Configuração

| GCP (original) | AWS (substituto) | Justificativa |
|---|---|---|
| GCP Secret Manager | **AWS Secrets Manager** | Para credenciais sensíveis (OpenRouter API key, LangFuse token) |
| GCP Secret Manager (config simples) | **AWS Systems Manager Parameter Store** | Mais barato para parâmetros não-sensíveis e versionamento |
| GCP IAM + Service Accounts | **AWS IAM Roles** (least-privilege por Lambda) | Cada Lambda recebe role dedicada com permissões mínimas |
| 2 GCP Projects (dev/prod) | **2 AWS Accounts** (via AWS Organizations) | Isolamento mais forte que projects; controle de billing por conta |

### 2.6 Camada de Observabilidade

| GCP (original) | AWS (substituto) | Justificativa |
|---|---|---|
| Cloud Logging | **Amazon CloudWatch Logs** | Log Groups por Lambda, structured logging via aws-lambda-powertools |
| Cloud Monitoring | **Amazon CloudWatch Metrics + Alarms** | Métricas customizadas + alarmes para SLOs |
| Cloud Monitoring Dashboard | **CloudWatch Dashboard** | Painel unificado do pipeline |
| Cloud Logging → GCS sink (CrewAI) | **CloudWatch Logs → Kinesis Firehose → S3** | Mesmo padrão, agentes CrewAI leem logs do S3 |
| LangFuse | **LangFuse mantido** | Agnóstico de cloud — continua igual |

### 2.7 Camada de DevOps e CI/CD

| GCP (original) | AWS (substituto) | Justificativa |
|---|---|---|
| Terraform + Terragrunt | **Terraform + Terragrunt mantidos** | Mesma estrutura, troca de provider e módulos |
| GitHub Actions | **GitHub Actions mantido** | Auth via `aws-actions/configure-aws-credentials` com OIDC |
| CodeRabbit | **CodeRabbit mantido** | Agnóstico |
| Workload Identity Federation (GCP) | **OIDC + IAM Role** (sem long-lived keys) | Padrão equivalente AWS para auth GitHub→Cloud |
| `gcloud` CLI (deploy inicial) | **AWS CLI / SAM CLI** | Para iteração rápida antes de migrar para Terraform |

### 2.8 Camada de Autonomous Ops (CrewAI)

| GCP (original) | AWS (substituto) | Justificativa |
|---|---|---|
| CrewAI no Cloud Run | **CrewAI no AWS Lambda ou Fargate** | Lambda para execuções agendadas via EventBridge Scheduler |
| Logs em GCS para agentes lerem | **Logs em S3 para agentes lerem** | Export via CloudWatch → Firehose → S3 |
| Slack webhook | **Slack webhook mantido** | Agnóstico |

**Comportamento dos 3 agentes (Triage, Root Cause, Reporter) permanece idêntico.**

---

## 3. Nova Arquitetura End-to-End (AWS)

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│              INVOICE PROCESSING PIPELINE - AWS ARCHITECTURE                       │
├──────────────────────────────────────────────────────────────────────────────────┤
│                                                                                   │
│  INGESTION         PROCESSING                                STORAGE              │
│  ─────────         ──────────                                ───────              │
│                                                                                   │
│  ┌───────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐         │
│  │ TIFF  │──▶│ Lambda 1 │──▶│ Lambda 2 │──▶│ Lambda 3 │──▶│ Lambda 4 │──▶ S3+  │
│  │  S3   │   │ TIFF→PNG │   │ Classify │   │ Extract  │   │  Write   │  Athena │
│  └───────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘  /Iceberg│
│      │           │              │              │              │                  │
│      ▼           ▼              ▼              ▼              ▼                  │
│  ┌────────────────────────────────────────────────────────────────────┐         │
│  │   EventBridge → SQS Queues (com DLQ por fila)                      │         │
│  │   uploaded → converted → classified → extracted                    │         │
│  └────────────────────────────────────────────────────────────────────┘         │
│                                                                                   │
│                              ▲                                                    │
│                              │ Bedrock Runtime                                   │
│                       ┌──────┴───────┐                                           │
│                       │  Bedrock     │                                           │
│                       │  Claude 4.5  │                                           │
│                       └──────────────┘                                           │
│                                                                                   │
│  ──────────────────────────────────────────────────────────────────────────────  │
│                                                                                   │
│  OBSERVABILITY                                                                    │
│  ─────────────                                                                    │
│                                                                                   │
│  ┌───────────────┐    ┌───────────────┐    ┌───────────────┐                    │
│  │   LangFuse    │    │ CloudWatch    │    │  CloudWatch   │                    │
│  │  (LLM calls)  │    │     Logs      │    │   Metrics     │                    │
│  └───────────────┘    └───────┬───────┘    └───────────────┘                    │
│                               │                                                   │
│                               ▼                                                   │
│                     ┌─────────────────┐                                          │
│                     │ Kinesis Firehose│──▶ S3 (logs bucket)                     │
│                     └─────────────────┘                                          │
│                                                                                   │
│  ──────────────────────────────────────────────────────────────────────────────  │
│                                                                                   │
│  AUTONOMOUS OPS (CrewAI on Lambda/Fargate)                                       │
│  ──────────────────────────────────────────                                      │
│                                                                                   │
│  ┌─────────────┐    ┌─────────────────┐    ┌─────────────────┐                  │
│  │   TRIAGE    │───▶│   ROOT CAUSE    │───▶│    REPORTER     │──▶ Slack         │
│  │   AGENT     │    │     AGENT       │    │     AGENT       │                  │
│  └─────────────┘    └─────────────────┘    └─────────────────┘                  │
│       (lê de S3)                                                                 │
│                                                                                   │
│  ──────────────────────────────────────────────────────────────────────────────  │
│                                                                                   │
│  CI/CD                                                                            │
│  ─────                                                                            │
│                                                                                   │
│  GitHub → CodeRabbit → GitHub Actions (OIDC) → Terraform → AWS                  │
│                                                                                   │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Plano de Trabalho (O Que Precisa Ser Feito)

### 4.1 Adapters (já previstos no D10)

| # | Adapter | Implementação AWS | Esforço |
|---|---|---|---|
| 1 | `StorageAdapter` | `S3Adapter` usando boto3 | Baixo |
| 2 | `MessagingAdapter` | `SqsAdapter` + `EventBridgeAdapter` | Baixo |
| 3 | `LLMAdapter` | `BedrockAdapter` (Claude/Nova) | Médio |
| 4 | `WarehouseAdapter` (novo) | `AthenaAdapter` ou `RedshiftAdapter` | Médio |
| 5 | `SecretsAdapter` (novo) | `AwsSecretsManagerAdapter` | Baixo |

### 4.2 Infraestrutura como Código

Reescrita dos módulos Terraform mantendo a estrutura Terragrunt:

```
infrastructure/
├── modules/
│   ├── lambda/          # (era cloud-run/)
│   ├── eventbridge/     # (era pubsub/ — parte 1)
│   ├── sqs/             # (era pubsub/ — parte 2)
│   ├── s3/              # (era gcs/)
│   ├── athena-iceberg/  # (era bigquery/)
│   ├── bedrock/         # (era vertex-ai/ — novo)
│   └── iam/             # (era iam/)
├── environments/
│   ├── dev/
│   │   └── terragrunt.hcl
│   └── prod/
│       └── terragrunt.hcl
└── terragrunt.hcl
```

**Provider:** trocar `google` / `google-beta` por `aws` (versão >= 5.x).

### 4.3 Triggers e Fluxo de Eventos

Novo fluxo de eventos:

```
1. Upload de TIFF para s3://invoices-input
2. S3 Event Notification → EventBridge
3. EventBridge → SQS "invoice-uploaded"
4. Lambda tiff-to-png-converter consome SQS
5. Lambda escreve PNG em s3://invoices-processed
6. Lambda envia mensagem para SQS "invoice-converted"
7. Lambda invoice-classifier consome SQS
8. Lambda envia para SQS "invoice-classified"
9. Lambda data-extractor consome, chama Bedrock
10. Lambda envia para SQS "invoice-extracted"
11. Lambda warehouse-writer consome, escreve em S3 (Iceberg) via Athena
```

Cada SQS tem DLQ associada (substitui o DLQ do Pub/Sub).

### 4.4 CI/CD (GitHub Actions)

Mudanças necessárias no workflow:

```yaml
# Trecho atualizado de .github/workflows/deploy.yml
permissions:
  id-token: write
  contents: read

jobs:
  deploy-dev:
    steps:
      - uses: actions/checkout@v4
      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::DEV_ACCOUNT_ID:role/GitHubActionsDeployRole
          aws-region: us-east-1
      - name: Terragrunt Apply
        run: terragrunt apply -auto-approve
        working-directory: infrastructure/environments/dev
```

**Pré-requisitos:**
- Configurar OIDC Identity Provider no IAM (GitHub)
- Criar IAM Role assumível por GitHub Actions com trust policy correta
- Substituir secrets `GCP_SA_KEY` por roles IAM via OIDC (sem long-lived credentials)

### 4.5 Permissões IAM (least-privilege)

Cada Lambda recebe role dedicada:

| Lambda | Permissões mínimas |
|---|---|
| `tiff-to-png-converter` | `s3:GetObject` (input), `s3:PutObject` (processed/archive), `sqs:SendMessage` (converted) |
| `invoice-classifier` | `s3:GetObject` (processed), `sqs:ReceiveMessage` (converted), `sqs:SendMessage` (classified) |
| `data-extractor` | `s3:GetObject` (processed), `sqs:*` (classified/extracted), `bedrock:InvokeModel`, `secretsmanager:GetSecretValue` (OpenRouter) |
| `warehouse-writer` | `sqs:ReceiveMessage` (extracted), `s3:PutObject` (warehouse), `glue:*` (Iceberg metadata), `athena:*` |

### 4.6 Secrets

Migração de secrets:

| Secret | Origem (GCP) | Destino (AWS) |
|---|---|---|
| OpenRouter API Key | Secret Manager | AWS Secrets Manager |
| LangFuse Public Key | Secret Manager | SSM Parameter Store (não-sensível) |
| LangFuse Secret Key | Secret Manager | AWS Secrets Manager |
| Slack Webhook URL | Secret Manager | AWS Secrets Manager |

**Bedrock NÃO precisa de API key** — autenticação via IAM Role da Lambda.

### 4.7 Observabilidade

| Item | Implementação AWS |
|---|---|
| Structured logging | `aws-lambda-powertools` (Python) — substitui Cloud Logging client |
| Métricas custom | CloudWatch Embedded Metric Format (EMF) |
| Distributed tracing | AWS X-Ray (substitui Cloud Trace) |
| Dashboard | CloudWatch Dashboard via Terraform |
| Alertas | CloudWatch Alarms → SNS → Slack |
| LLM metrics | LangFuse (mantido) |

### 4.8 CrewAI (Autonomous Ops)

| Item | Mudança |
|---|---|
| Agentes lendo logs | De GCS → de S3 (export via CloudWatch → Firehose) |
| Execução agendada | EventBridge Scheduler (substitui Cloud Scheduler) |
| Hosting dos agentes | Lambda (curtos) ou Fargate (longos) |
| Comportamento dos agentes | **Inalterado** — Triage, Root Cause, Reporter |

---

## 5. Decisões Atualizadas (substituem GCP-specific)

| # Original | Decisão Original (GCP) | Decisão Nova (AWS) |
|---|---|---|
| D6 | GCP como nuvem primária | **AWS como nuvem primária** |
| D7 | Event-driven com Pub/Sub | **Event-driven com EventBridge + SQS** |
| D8 | Cloud Run para compute | **AWS Lambda para compute** (Fargate como fallback) |
| D9 | BigQuery como data warehouse | **Athena + S3 Iceberg como lakehouse** |
| D10 | Adapter Pattern (mantida) | **Adapter Pattern (mantida — agora com implementação AWS primária)** |
| D14 | Projetos GCP separados | **Contas AWS separadas (via Organizations)** |
| D15 | Gemini 2.0 Flash como LLM primário | **Claude Haiku 4.5 (Bedrock) como primário, Sonnet 4.5 como fallback de qualidade** |
| D16 | OpenRouter como fallback | **OpenRouter mantido como fallback secundário** |
| D24 | GCP Secret Manager | **AWS Secrets Manager + SSM Parameter Store** |
| D25 | gcloud CLI inicial | **AWS CLI / SAM CLI inicial** |
| D29 | Cloud Logging → GCS export | **CloudWatch Logs → Kinesis Firehose → S3** |

Demais decisões (D1-D5, D11-D13, D17-D23, D26-D28, D30) **permanecem inalteradas** pois são agnósticas de nuvem.

---

## 6. Estimativa de Custos (AWS)

### Dev Environment (~2.000 invoices/mês simulados)

| Componente | Custo Mensal Estimado |
|---|---|
| Lambda (4 funções, ~8k invocações) | ~$2 |
| SQS (4 filas + DLQs) | ~$1 |
| S3 (4 buckets, ~5GB) | ~$1 |
| Athena (queries de validação) | ~$2 |
| EventBridge | <$1 |
| Bedrock Claude Haiku 4.5 | ~$2 |
| Secrets Manager (4 secrets) | ~$2 |
| CloudWatch Logs + Metrics | ~$3 |
| **Total Dev** | **~$15/mês** |

### Prod Environment (~3.500 invoices/mês)

| Componente | Custo Mensal Estimado |
|---|---|
| Lambda | ~$5 |
| SQS | ~$2 |
| S3 (com archive Glacier) | ~$5 |
| Athena | ~$5 |
| EventBridge | ~$1 |
| Bedrock Claude Haiku 4.5 | ~$4 |
| Bedrock Claude Sonnet 4.5 (fallback ~5%) | ~$2 |
| Secrets Manager | ~$2 |
| CloudWatch + Firehose + X-Ray | ~$10 |
| **Total Prod** | **~$36/mês** |

**Comparação:** versão GCP estimava ~$20 dev / ~$55 prod. AWS Lambda + Athena tende a ser ligeiramente mais barato no volume atual.

---

## 7. Riscos e Mitigações da Migração

| # | Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|---|
| MR1 | Lambda timeout em TIFFs grandes (multi-page) | Média | Médio | Configurar 15min + 10GB RAM; fallback para Fargate Tasks |
| MR2 | Cold start de Lambda elevando latência P95 | Baixa | Médio | Provisioned Concurrency em prod para `data-extractor` |
| MR3 | Bedrock quota/throttling em região | Média | Alto | Cross-region inference profiles; fallback OpenRouter |
| MR4 | Diferenças de qualidade entre Claude e Gemini | Média | Médio | Sprint de validação em março com dataset real do Carlos |
| MR5 | Curva de aprendizado AWS (time já aprendendo GCP) | Média | Médio | Pareamento Pedro+João nas primeiras Lambdas; documentação interna |
| MR6 | Athena/Iceberg vs BigQuery — gap de features SQL | Baixa | Baixo | Validar queries do BI antes de cutover |
| MR7 | Custos de egress se algum dado ficar fora da AWS | Baixa | Baixo | Manter LangFuse em região próxima; minimizar transferência |

---

## 8. Cronograma de Migração (sem impacto no deadline)

| Fase | Datas | Entregáveis |
|---|---|---|
| **Setup AWS** | 11-15 maio 2026 | Contas AWS Organizations criadas, OIDC GitHub configurado, IAM baseline |
| **Adapters AWS** | 16-22 maio 2026 | S3Adapter, SqsAdapter, BedrockAdapter implementados e testados |
| **Terraform AWS** | 16-25 maio 2026 | Módulos reescritos, dev environment provisionado |
| **Migração das 4 Lambdas** | 26 maio - 5 junho 2026 | Funções deployadas em dev, end-to-end teste passando |
| **Validação Bedrock vs Gemini** | 1-10 junho 2026 | Sprint Ana — comparativo de acurácia em dataset real |
| **CrewAI no AWS** | 8-15 junho 2026 | Agentes lendo de S3, alertas no Slack |
| **Production launch (AWS)** | 15-20 junho 2026 | Cutover para prod AWS |

---

## 9. O Que NÃO Muda

| Item | Permanece |
|---|---|
| Lógica de negócio das 4 funções | Inalterada (extração + validação Pydantic) |
| Schema de extração (12 campos) | Inalterado |
| Prompts por tipo de invoice | Inalterados (são agnósticos de provedor LLM) |
| LangFuse para LLMOps | Inalterado |
| CrewAI (3 agentes) | Inalterado em comportamento |
| GitHub + CodeRabbit | Inalterados |
| Terraform + Terragrunt (estrutura) | Inalterada (apenas providers e módulos) |
| Pydantic validation | Inalterada |
| Slack alerts | Inalterado |
| Equipe e RACI | Inalterados |
| Métricas de sucesso (90% accuracy, etc.) | Inalteradas |

---

## 10. Próximos Passos

1. **Aprovar a migração** com Marina + executivos (decisão de negócio)
2. **Pedro Lima** — provisionar AWS Organizations, contas dev/prod, OIDC GitHub
3. **João Silva** — adaptar interfaces existentes (StorageAdapter, MessagingAdapter, LLMAdapter) e implementar versões AWS
4. **Ana Costa** — sprint de validação Bedrock Claude 4.5 vs Gemini 2.0 Flash usando dataset do Carlos
5. **Pedro Lima** — reescrever módulos Terraform (lambda/, sqs/, eventbridge/, s3/, athena-iceberg/, bedrock/)
6. **João Silva** — migrar CrewAI para ler logs de S3 (substituir leitor GCS)
7. **Marina Santos** — atualizar runbook de escalação com endpoints AWS
8. **Pedro Lima** — security review específico para AWS antes do go-live

---

## Document Metadata

| Campo | Valor |
|---|---|
| **Versão** | 1.0.0 |
| **Criado** | 2026-05-10 |
| **Base** | summary-requirements.md + notas 01-06 |
| **Status** | Proposta para aprovação |
| **Substitui decisões** | D6, D7, D8, D9, D14, D15, D24, D25, D29 |
| **Mantém decisões** | D1-D5, D10-D13, D16-D23, D26-D28, D30 |
