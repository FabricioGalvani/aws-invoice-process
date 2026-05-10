# DevOps & Infrastructure: CI/CD and Deployment Strategy

**Date:** February 10, 2026 | **Duration:** 68 minutes | **Platform:** Google Meet

**Participants:**
- Marina Santos (Product Manager)
- João Silva (Senior Data Engineer)
- Ana Costa (ML Engineer)
- Pedro Lima (Platform/DevOps Lead)
- Carlos Ferreira (Business Stakeholder - Restaurant Operations)

---

## Summary

Pedro presented the infrastructure and CI/CD strategy. Team agreed on Terraform + Terragrunt for IaC with separate dev/prod environments. GitHub Actions for CI/CD with CodeRabbit for AI-powered code reviews. Initial deployment will use gcloud CLI for rapid iteration, then full automation. Discussed secrets management with GCP Secret Manager and environment promotion workflow.

---

## Key Decisions

| # | Decision | Owner | Status |
|---|----------|-------|--------|
| 1 | Terraform + Terragrunt for all infrastructure | Pedro | Approved |
| 2 | Separate GCP projects for dev and prod | Pedro | Approved |
| 3 | GitHub Actions for CI/CD pipelines | Pedro | Approved |
| 4 | CodeRabbit for AI code review | Pedro | Approved |
| 5 | GCP Secret Manager for API keys and credentials | Pedro | Approved |
| 6 | gcloud CLI for initial deployments, then automate | João | Approved |
| 7 | Feature branch workflow with required PR reviews | Pedro | Approved |

---

## Action Items

- [ ] **Pedro**: Create Terraform module structure (Due: Feb 14, 2026)
- [ ] **Pedro**: Set up GCP dev project with basic IAM (Due: Feb 13, 2026)
- [ ] **Pedro**: Configure GitHub Actions workflow files (Due: Feb 17, 2026)
- [ ] **Pedro**: Set up CodeRabbit integration on repository (Due: Feb 14, 2026)
- [ ] **João**: Document gcloud CLI deployment commands (Due: Feb 12, 2026)
- [ ] **João**: Test initial function deployment to dev (Due: Feb 15, 2026)
- [ ] **Marina**: Define environment promotion approval process (Due: Feb 14, 2026)

---

## Next Steps

1. Pedro to demo Terraform module structure
2. Set up dev environment for team testing
3. Final meeting to discuss autonomous operations

---

## Full Transcript

**[00:00:10] Marina Santos:**
Today is all about infrastructure and deployment. Pedro, you've been working on this - show us what you've got.

**[00:00:25] Pedro Lima:**
Alright. [Shares screen] So I've designed a complete infrastructure-as-code strategy using Terraform and Terragrunt. Let me walk through it.

**[00:00:45] Pedro Lima:**
First, the project structure. We'll have two GCP projects: `invoice-pipeline-dev` and `invoice-pipeline-prod`. Complete isolation between environments.

**[00:01:05] João Silva:**
Good. Same Terraform code deploys to both?

**[00:01:12] Pedro Lima:**
Exactly. That's where Terragrunt comes in. Terragrunt handles the environment-specific configuration - project IDs, bucket names, scaling parameters. The Terraform modules are identical.

**[00:01:35] Pedro Lima:**
Here's the folder structure:

```
infrastructure/
├── modules/
│   ├── cloud-run/
│   ├── pubsub/
│   ├── gcs/
│   ├── bigquery/
│   └── iam/
├── environments/
│   ├── dev/
│   │   └── terragrunt.hcl
│   └── prod/
│       └── terragrunt.hcl
└── terragrunt.hcl (root)
```

**[00:02:10] Ana Costa:**
So if I need to add a new Cloud Run function, I just update the cloud-run module?

**[00:02:20] Pedro Lima:**
Yes. You update the module, test in dev, then promote to prod. The module itself is environment-agnostic.

**[00:02:38] Marina Santos:**
What about CI/CD?

**[00:02:45] Pedro Lima:**
GitHub Actions. I'm proposing this workflow:

1. Developer creates feature branch
2. Opens PR when ready
3. GitHub Actions runs: lint, unit tests, security scan
4. CodeRabbit does AI code review
5. Human reviewer approves
6. Merge to main triggers deploy to dev
7. Manual approval promotes to prod

**[00:03:25] João Silva:**
Wait, CodeRabbit? What's that?

**[00:03:32] Pedro Lima:**
It's an AI code reviewer. It analyzes PRs and provides feedback - potential bugs, style issues, security concerns. Think of it as having an AI pair programmer review every PR.

**[00:03:55] Marina Santos:**
Is it any good?

**[00:04:02] Pedro Lima:**
I've used it on other projects. It catches things humans miss - like forgotten error handling or SQL injection risks. Not perfect, but it's a good first pass before human review.

**[00:04:25] Ana Costa:**
For the ML code specifically, does it understand Python and ML patterns?

**[00:04:35] Pedro Lima:**
Yes, it supports Python well. It won't validate your model accuracy, but it'll catch code quality issues, typing problems, that kind of thing.

**[00:04:55] João Silva:**
I like it. Adds another layer of review.

**[00:05:05] Marina Santos:**
How do we handle secrets? API keys for Gemini, LangFuse tokens?

**[00:05:15] Pedro Lima:**
GCP Secret Manager. All secrets stored there, accessed at runtime. Never in code, never in environment variables at deploy time.

**[00:05:35] Pedro Lima:**
The Cloud Run functions reference secrets by name. At startup they fetch from Secret Manager. If a secret is rotated, the function picks up the new value on next cold start.

**[00:05:58] Carlos Ferreira:**
What about cost? All this infrastructure sounds expensive.

**[00:06:08] Pedro Lima:**
Actually, serverless is cost-effective. We only pay when processing. Let me show you rough estimates:

- Cloud Run: ~$5-10/month for our volume
- Pub/Sub: ~$1/month
- GCS: ~$2/month
- BigQuery: ~$5/month
- Secret Manager: < $1/month

**[00:06:40] Carlos Ferreira:**
So under $20/month for infrastructure?

**[00:06:48] Pedro Lima:**
For dev environment, yes. Prod might be 2-3x depending on volume, but still very reasonable.

**[00:07:05] João Silva:**
For initial development, can we deploy directly with gcloud CLI? Terraform setup takes time and I want to iterate quickly.

**[00:07:20] Pedro Lima:**
Absolutely. For the prototype phase, use gcloud directly. I'll have Terraform ready in parallel. Once we're stable, we switch to automated deployment.

**[00:07:40] João Silva:**
Perfect. So I can do something like `gcloud run deploy tiff-converter --source .` and test immediately?

**[00:07:52] Pedro Lima:**
Exactly. Just document the commands so we can translate them to Terraform later.

**[00:08:05] Marina Santos:**
What's the rollback strategy if a deployment breaks prod?

**[00:08:15] Pedro Lima:**
Cloud Run keeps previous revisions. One command to roll back: `gcloud run services update-traffic --to-revisions=PREVIOUS=100`. We can also automate this based on error rates.

**[00:08:40] Ana Costa:**
For the ML models, if we update a prompt and accuracy drops, can we roll back the prompt too?

**[00:08:55] Pedro Lima:**
Prompts should be versioned in code, not external. When you deploy a new version, the prompt goes with it. Rollback the code, rollback the prompt.

**[00:09:15] Ana Costa:**
Makes sense. I'll treat prompts as code.

**[00:09:25] Marina Santos:**
What about monitoring? How do we know if something's wrong?

**[00:09:35] Pedro Lima:**
GCP Cloud Monitoring for infrastructure metrics - CPU, memory, error rates. Cloud Logging for application logs. We can set up alerts for anomalies.

**[00:09:58] Pedro Lima:**
Combined with LangFuse for LLM-specific metrics, we'll have full visibility.

**[00:10:12] João Silva:**
Can we have a dashboard that shows the whole pipeline? Like messages in each Pub/Sub topic, processing rates, error counts?

**[00:10:28] Pedro Lima:**
Yes, I'll build a Cloud Monitoring dashboard. We can also export to Grafana if you prefer that UI.

**[00:10:45] Marina Santos:**
Let's stick with native GCP tools for now. Less complexity.

**[00:10:55] Pedro Lima:**
Agreed.

**[00:11:05] Marina Santos:**
Ok, let me summarize:
1. Terraform + Terragrunt for IaC
2. Separate dev and prod GCP projects
3. GitHub Actions for CI/CD
4. CodeRabbit for AI code review
5. GCP Secret Manager for secrets
6. gcloud CLI for prototyping, then automate
7. Cloud Monitoring for observability

Everyone aligned?

**[00:11:45] All:**
Yes.

**[00:11:50] Marina Santos:**
Great. Pedro, can you have the dev environment ready by end of week?

**[00:11:58] Pedro Lima:**
Yes, I'll have the GCP project set up with basic IAM. Team can start deploying functions by Friday.

**[00:12:12] Marina Santos:**
Perfect. One more meeting to go - we need to discuss the autonomous operations vision. João, you've been talking about that CrewAI thing?

**[00:12:30] João Silva:**
Yes, it's pretty exciting. I'll prepare a presentation for next week.

**[00:12:40] Marina Santos:**
Great. Thanks everyone!

**[Meeting ended at 01:08:33]**

---

## Infrastructure Diagram (from Pedro's slides)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           DEPLOYMENT ARCHITECTURE                                │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────────┐  │
│   │                         GitHub Repository                                 │  │
│   ├─────────────────────────────────────────────────────────────────────────┤  │
│   │  Feature Branch → PR → CodeRabbit Review → Human Review → Merge → Main  │  │
│   └──────────────────────────────────┬──────────────────────────────────────┘  │
│                                      │                                          │
│                                      ▼                                          │
│   ┌─────────────────────────────────────────────────────────────────────────┐  │
│   │                        GitHub Actions CI/CD                              │  │
│   ├─────────────────────────────────────────────────────────────────────────┤  │
│   │  Lint → Test → Security Scan → Build → Deploy Dev → [Approval] → Prod   │  │
│   └──────────────────────────────────┬──────────────────────────────────────┘  │
│                                      │                                          │
│                    ┌─────────────────┴─────────────────┐                       │
│                    ▼                                   ▼                        │
│   ┌────────────────────────────┐    ┌────────────────────────────┐            │
│   │     DEV Environment        │    │     PROD Environment       │            │
│   │  invoice-pipeline-dev      │    │  invoice-pipeline-prod     │            │
│   ├────────────────────────────┤    ├────────────────────────────┤            │
│   │  - Cloud Run (4 functions) │    │  - Cloud Run (4 functions) │            │
│   │  - Pub/Sub (4 topics)      │    │  - Pub/Sub (4 topics)      │            │
│   │  - GCS (4 buckets)         │    │  - GCS (4 buckets)         │            │
│   │  - BigQuery dataset        │    │  - BigQuery dataset        │            │
│   │  - Secret Manager          │    │  - Secret Manager          │            │
│   └────────────────────────────┘    └────────────────────────────┘            │
│                                                                                  │
│   TERRAGRUNT STRUCTURE:                                                         │
│   infrastructure/                                                                │
│   ├── modules/           # Reusable Terraform modules                           │
│   └── environments/      # Dev/Prod specific configs                            │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## GitHub Actions Workflow (planned)

```yaml
# .github/workflows/deploy.yml
name: Deploy Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run linters
        run: |
          pip install ruff
          ruff check .

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: pytest tests/

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run security scan
        uses: aquasecurity/trivy-action@master

  deploy-dev:
    needs: [lint, test, security]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to dev
        run: terragrunt apply -auto-approve

  deploy-prod:
    needs: [deploy-dev]
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Deploy to prod
        run: terragrunt apply -auto-approve
```

---

## Notes & Observations

- Strong IaC foundation with Terraform + Terragrunt
- CodeRabbit is interesting - AI reviewing AI code
- gcloud CLI for prototyping is pragmatic - don't over-engineer early
- Cost estimates are very reasonable for serverless
- Need to set up Cloud Monitoring dashboard
- Next meeting is about autonomous ops - João has CrewAI idea

---

*Generated by Krisp AI Meeting Assistant*
