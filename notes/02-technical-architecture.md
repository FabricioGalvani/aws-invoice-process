# Technical Architecture Review: Invoice Processing Pipeline

**Date:** January 22, 2026 | **Duration:** 62 minutes | **Platform:** Google Meet

**Participants:**
- Marina Santos (Product Manager)
- João Silva (Senior Data Engineer)
- Ana Costa (ML Engineer)
- Pedro Lima (Platform/DevOps Lead)
- Carlos Ferreira (Business Stakeholder - Restaurant Operations)

---

## Summary

Technical deep-dive into the proposed architecture for the invoice processing pipeline. Team aligned on GCP as primary cloud with event-driven serverless architecture. João presented the high-level flow: GCS → Pub/Sub → Cloud Run → BigQuery. Discussion about multi-cloud flexibility led to decision to implement Adapter Pattern for future AWS/Azure support. Pedro raised CI/CD and infrastructure-as-code requirements.

---

## Key Decisions

| # | Decision | Owner | Status |
|---|----------|-------|--------|
| 1 | GCP as primary cloud platform | João | Approved |
| 2 | Event-driven architecture with Pub/Sub | João | Approved |
| 3 | Cloud Run for serverless compute | João | Approved |
| 4 | BigQuery as destination data warehouse | João | Approved |
| 5 | Implement Adapter Pattern for cloud portability | João | Approved |
| 6 | Terraform for infrastructure provisioning | Pedro | Approved |

---

## Action Items

- [ ] **João**: Create architecture diagram with component details (Due: Jan 24, 2026)
- [ ] **João**: Define Adapter Pattern interface for cloud services (Due: Jan 27, 2026)
- [ ] **Ana**: Finalize LLM selection (Gemini 2.0 Flash vs OpenRouter) (Due: Jan 25, 2026)
- [ ] **Pedro**: Set up Terraform project structure (Due: Jan 29, 2026)
- [ ] **Pedro**: Configure GCP service accounts and IAM (Due: Jan 26, 2026)
- [ ] **Marina**: Schedule data pipeline deep-dive meeting (Due: Jan 23, 2026)

---

## Next Steps

1. Data pipeline process meeting to detail the Cloud Run functions
2. Ana to present LLM evaluation results
3. Pedro to share Terraform module structure

---

## Full Transcript

**[00:00:10] Marina Santos:**
Alright, let's dive into the technical architecture. João, you've been working on this - can you walk us through your proposal?

**[00:00:25] João Silva:**
Sure. So based on our requirements - 2000+ invoices per month, 90% accuracy target, variable formats - I'm proposing an event-driven serverless architecture on GCP.

**[00:00:48] João Silva:**
Let me share my screen... ok so here's the high-level flow. Invoices come in as TIFF files and land in Google Cloud Storage. That triggers a Pub/Sub event. Cloud Run picks it up, processes it through the LLM, and writes the extracted data to BigQuery.

**[00:01:22] Pedro Lima:**
Why Pub/Sub instead of just triggering Cloud Run directly from GCS?

**[00:01:35] João Silva:**
Decoupling. If we trigger directly, we lose messages if Cloud Run is down or throttled. Pub/Sub gives us retry logic, dead-letter queues, and the ability to add multiple consumers later without changing the source.

**[00:01:58] Pedro Lima:**
Makes sense. What about ordering guarantees?

**[00:02:08] João Silva:**
We don't need strict ordering for invoices. Each one is independent. Pub/Sub will handle retries if processing fails.

**[00:02:25] Ana Costa:**
On the LLM side, I've been looking at options. Gemini 2.0 Flash on Vertex AI is looking good - it's native to GCP, has excellent multimodal document understanding with 1M token context window, and the pricing is reasonable at about $0.00025 per image.

**[00:02:55] Carlos Ferreira:**
Wait, so each invoice costs... let me do the math... $0.50 per 2000 invoices? That's nothing compared to what we're paying in manual processing.

**[00:03:15] Ana Costa:**
Well, it might be a bit more because some invoices have multiple pages. But yes, order of magnitude is dollars per month, not thousands.

**[00:03:32] Marina Santos:**
That's great news for the business case. Pedro, what about infrastructure?

**[00:03:45] Pedro Lima:**
I want to use Terraform for everything. Infrastructure as code from day one. No clicking around in the console. Also, I'm thinking we should use Terragrunt for environment management - dev, staging, prod.

**[00:04:12] João Silva:**
Agreed. We should also think about... actually Marina, there's something I wanted to bring up. What if in the future the business wants to run this on AWS or Azure?

**[00:04:35] Marina Santos:**
Is that likely?

**[00:04:42] Carlos Ferreira:**
We've been talking about multi-cloud strategy at the executive level. Some of our partners are AWS-only.

**[00:04:58] João Silva:**
So I'm proposing we implement what's called the Adapter Pattern. Basically, we create abstract interfaces for cloud services - storage, messaging, compute - and the actual implementation is pluggable.

**[00:05:25] Pedro Lima:**
Like a Storage interface that can be implemented by GCSAdapter, S3Adapter, BlobAdapter?

**[00:05:38] João Silva:**
Exactly. On day one we only implement GCP adapters, but the architecture is ready for multi-cloud.

**[00:05:52] Marina Santos:**
How much extra work is that?

**[00:06:00] João Silva:**
Maybe 10-15% more upfront, but it saves us from a complete rewrite if we need to go multi-cloud later.

**[00:06:15] Marina Santos:**
Makes sense. Let's do it. Any objections? ... Ok, approved.

**[00:06:30] Ana Costa:**
One thing I'm not clear on - are we doing the LLM call directly in Cloud Run, or is there going to be some kind of orchestration?

**[00:06:48] João Silva:**
Good question. I'm thinking we'll have multiple Cloud Run functions, each handling a specific step. But let's dive into that in the next meeting - we should dedicate time to map out the data flow in detail.

**[00:07:12] Marina Santos:**
Agreed. I'll schedule a data pipeline process meeting. We need to understand exactly what happens from when a file lands in GCS to when data appears in BigQuery.

**[00:07:35] Pedro Lima:**
On CI/CD - I want to propose GitHub Actions. We're already using GitHub for code. And I've been looking at this tool called CodeRabbit for AI-powered code reviews.

**[00:07:58] Marina Santos:**
Interesting. Can we discuss that in the DevOps meeting?

**[00:08:08] Pedro Lima:**
Sure, just flagging it for now.

**[00:08:18] João Silva:**
One more thing on architecture - we should consider observability from the start. I don't want to build this and then scramble to figure out why things are failing.

**[00:08:38] Ana Costa:**
For the LLM specifically, there's a tool called LangFuse that I've used before. It tracks every LLM call - prompt, response, latency, cost. Super useful for debugging extraction issues.

**[00:08:58] Marina Santos:**
Let's add that to the ML strategy discussion. Ana, can you prepare a comparison of observability tools?

**[00:09:12] Ana Costa:**
Yes, I'll include that in my LLM evaluation.

**[00:09:25] Marina Santos:**
Perfect. Let me summarize what we've decided today:
1. GCP is our primary cloud
2. Event-driven architecture with Pub/Sub
3. Cloud Run for compute
4. BigQuery for data warehouse
5. Adapter Pattern for cloud portability
6. Terraform for IaC

Everyone aligned? ... Great. Next meeting we'll deep-dive into the data pipeline steps. João, can you have that architecture diagram ready?

**[00:10:05] João Silva:**
Will do.

**[Meeting ended at 01:02:15]**

---

## Architecture Diagram (from João's screen share)

```
┌─────────────────────────────────────────────────────────────────┐
│                     HIGH-LEVEL ARCHITECTURE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐ │
│   │   GCS    │───▶│ Pub/Sub  │───▶│Cloud Run │───▶│ BigQuery │ │
│   │ (input)  │    │ (events) │    │(process) │    │ (output) │ │
│   └──────────┘    └──────────┘    └──────────┘    └──────────┘ │
│        │                               │                        │
│        │                               ▼                        │
│        │                         ┌──────────┐                   │
│        │                         │   LLM    │                   │
│        │                         │ (Gemini) │                   │
│        └─────────────────────────└──────────┘                   │
│                                                                  │
│   ADAPTER PATTERN:                                               │
│   ┌────────────────────────────────────────────────────────┐   │
│   │  Interface: StorageAdapter, MessagingAdapter, LLMAdapter │   │
│   │  GCP Impl:  GCSAdapter, PubSubAdapter, VertexAdapter     │   │
│   │  AWS Impl:  S3Adapter, SNSAdapter, BedrockAdapter (TBD)  │   │
│   └────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Notes & Observations

- Team is aligned on cloud-native approach
- Adapter Pattern decision shows forward thinking - good for multi-cloud strategy
- Need to detail the Cloud Run processing steps - single function or multiple?
- LangFuse mentioned for LLM observability - Ana seems experienced with this
- Pedro pushing for strong DevOps practices - Terraform + Terragrunt
- CodeRabbit for AI code review - interesting, need to evaluate

---

*Generated by Krisp AI Meeting Assistant*
