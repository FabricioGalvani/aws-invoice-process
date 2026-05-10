# Data & ML Strategy: LLM Selection and Observability

**Date:** February 3, 2026 | **Duration:** 55 minutes | **Platform:** Google Meet

**Participants:**
- Marina Santos (Product Manager)
- João Silva (Senior Data Engineer)
- Ana Costa (ML Engineer)
- Pedro Lima (Platform/DevOps Lead)
- Carlos Ferreira (Business Stakeholder - Restaurant Operations)

---

## Summary

Ana presented LLM evaluation results. Team decided on Gemini 2.0 Flash on Vertex AI as primary model with OpenRouter as fallback. Gemini 2.0 Flash offers superior multimodal capabilities with 1M token context window, native tool use, and excellent document understanding - ideal for invoice extraction. Detailed discussion on prompt engineering strategies and extraction schema. LangFuse selected for LLMOps observability - will track traces, cost, latency, and extraction quality. Also discussed need for synthetic data generator to create test invoices with ground truth.

---

## Key Decisions

| # | Decision | Owner | Status |
|---|----------|-------|--------|
| 1 | Gemini 2.0 Flash as primary LLM | Ana | Approved |
| 2 | OpenRouter as fallback provider | Ana | Approved |
| 3 | LangFuse for LLMOps observability | Ana | Approved |
| 4 | Structured JSON output with Pydantic validation | João | Approved |
| 5 | Build synthetic data generator for testing | João | Approved |
| 6 | Track 4 key metrics: cost, latency, accuracy, token usage | Ana | Approved |

---

## Action Items

- [ ] **Ana**: Set up Vertex AI project and enable Gemini 2.0 Flash API (Due: Feb 5, 2026)
- [ ] **Ana**: Create LangFuse project and configure integration (Due: Feb 7, 2026)
- [ ] **Ana**: Write prompt templates for UberEats invoice types (Due: Feb 8, 2026)
- [ ] **João**: Build invoice generator for synthetic test data (Due: Feb 10, 2026)
- [ ] **João**: Create ground truth CSV format and sample dataset (Due: Feb 10, 2026)
- [ ] **Carlos**: Review and approve extraction schema fields (Due: Feb 6, 2026)
- [ ] **Marina**: Define accuracy thresholds per field for go/no-go (Due: Feb 7, 2026)

---

## Next Steps

1. Ana to demonstrate LangFuse dashboard with sample traces
2. João to present synthetic data generator design
3. DevOps meeting to discuss CI/CD pipeline

---

## Full Transcript

**[00:00:15] Marina Santos:**
Today we're focusing on the ML side - model selection and how we're going to monitor everything. Ana, you did the evaluation. Walk us through it.

**[00:00:32] Ana Costa:**
Sure. So I tested three options: Gemini 2.0 Flash on Vertex AI, GPT-4o Vision through Azure OpenAI, and Claude 3.5 Sonnet through OpenRouter.

**[00:00:52] Ana Costa:**
[Sharing screen with benchmark results]

Here are my results on 20 sample invoices from Carlos:

| Model | Accuracy | Latency | Cost per Invoice |
|-------|----------|---------|------------------|
| Gemini 2.0 Flash | 96.5% | 1.2s | $0.002 |
| GPT-4o Vision | 97.1% | 2.8s | $0.012 |
| Claude 3.5 Sonnet | 95.8% | 1.9s | $0.006 |

**[00:01:35] João Silva:**
Interesting. GPT-4 is more accurate but 5x more expensive and slower.

**[00:01:45] Ana Costa:**
Exactly. For 2000+ invoices per month, cost adds up. Gemini 2.0 Flash at $0.002 is about $4/month. GPT-4o would be $24/month. Not huge, but Gemini is also native to GCP which makes integration easier. Plus Gemini 2.0 Flash has a 1M token context window and native tool use capabilities.

**[00:02:10] Carlos Ferreira:**
96.5% accuracy - is that good enough?

**[00:02:18] Ana Costa:**
It's above our 90% target. And remember, this is on the first iteration of prompts. With prompt engineering we can likely push it higher.

**[00:02:35] Marina Santos:**
What about reliability? What happens when the model returns garbage?

**[00:02:48] Ana Costa:**
Good question. We'll use Pydantic for schema validation. The LLM output must match our expected structure - if it doesn't, we reject it and either retry or send to manual review.

**[00:03:12] João Silva:**
I like that. We define a strict Pydantic model for the extraction output. If the LLM returns something weird, it fails validation immediately.

**[00:03:30] Ana Costa:**
Exactly. And we can set up OpenRouter as a fallback. If Gemini 2.0 Flash fails or is unavailable, we can route to Claude 3.5 or GPT-4o through OpenRouter.

**[00:03:50] Marina Santos:**
That sounds like good redundancy. What about observability? You mentioned LangFuse before.

**[00:04:05] Ana Costa:**
Yes, let me explain LangFuse. [Pulls up LangFuse documentation]

LangFuse is an open-source LLM observability platform. It captures:
- **Traces**: Every LLM call with full prompt and response
- **Cost**: Token usage and actual cost per call
- **Latency**: Time to first token, total generation time
- **Quality**: We can log human feedback and accuracy scores

**[00:04:48] Pedro Lima:**
Is it self-hosted or SaaS?

**[00:04:55] Ana Costa:**
Both options. For now I recommend their cloud version - it's free up to 50k observations per month. We can self-host later if needed.

**[00:05:15] João Silva:**
How do we integrate it?

**[00:05:22] Ana Costa:**
Python SDK. It's literally a few lines of code:

```python
from langfuse import Langfuse

langfuse = Langfuse()

trace = langfuse.trace(name="invoice-extraction")
generation = trace.generation(
    name="gemini-call",
    model="gemini-2.0-flash",
    input=prompt,
    output=response
)
```

**[00:05:55] Marina Santos:**
What metrics do we care about most?

**[00:06:05] Ana Costa:**
I'd say four key metrics:
1. **Cost per extraction** - are we within budget?
2. **Latency (P95)** - are we processing fast enough?
3. **Accuracy per field** - are we hitting 90%+ on each field?
4. **Token usage** - are our prompts efficient?

**[00:06:40] Carlos Ferreira:**
How do we measure accuracy automatically?

**[00:06:50] Ana Costa:**
That's the tricky part. We need ground truth data. João, have you thought about this?

**[00:07:00] João Silva:**
Actually, yes. I've been thinking we should build a synthetic invoice generator. Instead of relying only on real invoices, we generate fake invoices where we control exactly what the data should be.

**[00:07:25] João Silva:**
So we create an invoice that says "Total: $1,234.56" and we know the LLM should extract exactly that value. We can generate thousands of these with known ground truth.

**[00:07:50] Marina Santos:**
But will synthetic invoices look like real ones?

**[00:08:00] João Silva:**
We'll make them realistic - different layouts, fonts, some noise and scanning artifacts. But yes, we'll also test on real invoices from Carlos.

**[00:08:20] Carlos Ferreira:**
I can also have my team label real invoices. Create a spreadsheet with the correct values for each invoice.

**[00:08:35] Ana Costa:**
Perfect. We'll have two test sets: synthetic with automatic ground truth, and real with human-labeled ground truth.

**[00:08:52] Marina Santos:**
Let's talk about the extraction schema. What fields are we extracting?

**[00:09:05] Ana Costa:**
[Shows schema diagram]

Core fields:
- `invoice_id`: The unique invoice number
- `vendor_name`: Restaurant or delivery partner name
- `vendor_type`: UberEats, DoorDash, Grubhub
- `invoice_date`: When invoice was issued
- `due_date`: Payment due date
- `subtotal`: Amount before tax
- `tax_amount`: Tax
- `total_amount`: Final total
- `currency`: BRL, USD, etc.
- `line_items`: Array of individual charges

**[00:09:55] Carlos Ferreira:**
We also need the commission rate. UberEats takes different percentages.

**[00:10:08] Ana Costa:**
Good catch. I'll add `commission_rate` and `commission_amount` to the schema.

**[00:10:22] João Silva:**
For line items, what fields?

**[00:10:28] Ana Costa:**
`description`, `quantity`, `unit_price`, `amount`. Pretty standard.

**[00:10:42] Marina Santos:**
Ok, I think we have a solid plan. Let me summarize:
1. Gemini 2.0 Flash as primary model
2. OpenRouter as fallback
3. LangFuse for observability
4. Pydantic for schema validation
5. João builds the invoice generator
6. Track cost, latency, accuracy, and tokens

Anything else?

**[00:11:20] Pedro Lima:**
When do we need this for the CI/CD pipeline?

**[00:11:28] Marina Santos:**
Let's discuss that in the DevOps meeting. I'll schedule it for next week.

**[00:11:40] Marina Santos:**
Great work everyone. Ana, can you have LangFuse set up by Friday so we can see it in action?

**[00:11:52] Ana Costa:**
Yes, I'll have a demo ready.

**[Meeting ended at 00:55:18]**

---

## LLM Evaluation Results (from Ana's slides)

| Model | Provider | Accuracy | P50 Latency | P95 Latency | Cost/Invoice |
|-------|----------|----------|-------------|-------------|--------------|
| Gemini 2.0 Flash | Vertex AI | 96.5% | 0.9s | 1.8s | $0.002 |
| GPT-4o Vision | Azure OpenAI | 97.1% | 1.8s | 3.5s | $0.012 |
| Claude 3.5 Sonnet | OpenRouter | 95.8% | 1.4s | 2.6s | $0.006 |

**Recommendation:** Gemini 2.0 Flash for cost efficiency, 1M token context, native tool use, and GCP native integration.

---

## LangFuse Metrics Dashboard (planned)

| Metric | Target | Alert Threshold |
|--------|--------|-----------------|
| Cost per extraction | < $0.005 | > $0.01 |
| Latency P95 | < 3s | > 5s |
| Overall accuracy | > 90% | < 85% |
| Validation failures | < 5% | > 10% |

---

## Extraction Schema v1

```json
{
  "invoice_id": "string",
  "vendor_name": "string",
  "vendor_type": "enum: ubereats|doordash|grubhub|other",
  "invoice_date": "date",
  "due_date": "date",
  "subtotal": "float",
  "tax_amount": "float",
  "commission_rate": "float",
  "commission_amount": "float",
  "total_amount": "float",
  "currency": "string",
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

---

## Notes & Observations

- Gemini 2.0 Flash selected for cost, 1M context, native tool use, and GCP integration - excellent choice
- OpenRouter as fallback adds resilience
- LangFuse is new to Pedro - may need training
- Invoice generator is great idea - synthetic data with known ground truth
- Schema needs Carlos approval - added commission fields
- Need to define go/no-go accuracy thresholds per field

---

*Generated by Krisp AI Meeting Assistant*
