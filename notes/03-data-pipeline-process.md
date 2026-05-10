# Data Pipeline Process: Cloud Run Functions Deep-Dive

**Date:** January 27, 2026 | **Duration:** 78 minutes | **Platform:** Google Meet

**Participants:**
- Marina Santos (Product Manager)
- João Silva (Senior Data Engineer)
- Ana Costa (ML Engineer)
- Pedro Lima (Platform/DevOps Lead)
- Carlos Ferreira (Business Stakeholder - Restaurant Operations)

---

## Summary

Deep-dive into the data pipeline architecture. Team agreed on 4 separate Cloud Run functions for modularity and independent scaling: (1) TIFF-to-PNG Converter, (2) Invoice Classifier, (3) Data Extractor using Gemini, and (4) BigQuery Writer. Each function communicates via Pub/Sub topics. Discussed prompt strategies for different invoice types. LLMOps (LangFuse) deferred to separate implementation phase.

---

## Key Decisions

| # | Decision | Owner | Status |
|---|----------|-------|--------|
| 1 | 4 separate Cloud Run functions (not monolith) | João | Approved |
| 2 | Function 1: tiff-to-png-converter | João | Approved |
| 3 | Function 2: invoice-classifier | Ana | Approved |
| 4 | Function 3: data-extractor (Gemini) | Ana | Approved |
| 5 | Function 4: bigquery-writer | João | Approved |
| 6 | Pub/Sub topics between each function | João | Approved |
| 7 | Different prompts for different invoice types | Ana | Approved |
| 8 | LangFuse integration deferred to Phase 2 | Ana | Approved |

---

## Action Items

- [ ] **João**: Implement tiff-to-png-converter function (Due: Feb 3, 2026)
- [ ] **João**: Implement bigquery-writer function (Due: Feb 5, 2026)
- [ ] **Ana**: Implement invoice-classifier function (Due: Feb 3, 2026)
- [ ] **Ana**: Implement data-extractor function with Gemini 2.0 Flash (Due: Feb 7, 2026)
- [ ] **Ana**: Create prompt templates for UberEats invoices (Due: Feb 5, 2026)
- [ ] **Pedro**: Set up Pub/Sub topics and subscriptions (Due: Feb 1, 2026)
- [ ] **Carlos**: Validate sample extraction outputs against ground truth (Due: Feb 10, 2026)

---

## Next Steps

1. Pedro to provision Pub/Sub infrastructure
2. Start parallel development of all 4 functions
3. Data & ML strategy meeting to discuss prompt engineering

---

## Full Transcript

**[00:00:12] Marina Santos:**
Ok everyone, today we're going deep on the data pipeline. João, you prepared the detailed architecture - take us through it.

**[00:00:28] João Silva:**
Right, so after our last meeting I thought a lot about whether to do this as a single Cloud Run function or break it into multiple steps. I'm strongly recommending we break it into 4 separate functions.

**[00:00:52] Pedro Lima:**
Why 4 specifically?

**[00:01:00] João Silva:**
Each step has different resource needs and failure modes. Let me walk through them.

**[00:01:15] João Silva:**
Function 1 is the TIFF-to-PNG converter. The TIFF files we receive are multi-page and high resolution. LLMs work better with PNG format, and we need to split multi-page TIFFs into individual page images.

**[00:01:42] Ana Costa:**
Yes, Gemini 2.0 Flash specifically handles PNG better than TIFF. Also, we might need to do some preprocessing - deskewing, contrast adjustment, that kind of thing.

**[00:02:05] João Silva:**
Exactly. So this function takes a TIFF from GCS, converts to PNG pages, saves them back to GCS in a processed bucket, and publishes a message to the next topic.

**[00:02:28] Carlos Ferreira:**
What about the original TIFF? Do we keep it?

**[00:02:35] João Silva:**
Yes, we never delete the original. It goes to an archive bucket with a retention policy.

**[00:02:48] Marina Santos:**
Good. What's Function 2?

**[00:02:55] João Silva:**
Function 2 is the invoice-classifier. Ana, do you want to explain this one?

**[00:03:05] Ana Costa:**
Sure. So not everything that lands in our bucket is actually an invoice we can process. The classifier does a few things: validates it's actually an invoice, identifies the vendor type - UberEats, DoorDash, etc. - and checks image quality.

**[00:03:35] Pedro Lima:**
Does the classifier use AI too?

**[00:03:42] Ana Costa:**
It can. For the MVP we might start with rule-based checks - file size, image dimensions, maybe some OCR to look for keywords. But we can upgrade to an LLM-based classifier later.

**[00:04:05] Carlos Ferreira:**
What happens if classification fails?

**[00:04:12] Ana Costa:**
It goes to a review queue. We'll have a simple UI where your team can look at failed items and either correct the classification or mark them as invalid.

**[00:04:32] Marina Santos:**
Ok, so Function 1 converts, Function 2 classifies. What's 3?

**[00:04:45] João Silva:**
Function 3 is the heart of the system - the data-extractor. Ana, this is your domain.

**[00:04:58] Ana Costa:**
Right. This is where we call Gemini 2.0 Flash. The function receives a classified invoice, sends the image to Gemini with a carefully crafted prompt, and gets back structured data.

**[00:05:22] Ana Costa:**
The key thing here is we need different prompts for different invoice types. UberEats invoices have a specific layout. DoorDash is different. The classifier tells us which prompt to use.

**[00:05:48] Carlos Ferreira:**
Different prompts? Can you explain more?

**[00:06:00] Ana Costa:**
So imagine the UberEats prompt says "Extract the following fields from this UberEats invoice: order ID is in the top right corner, restaurant name is in the header..." and so on. Very specific instructions for that layout.

**[00:06:28] Ana Costa:**
For DoorDash, the prompt would be different because their layout is different. The order ID might be at the bottom, the date format might be different.

**[00:06:48] João Silva:**
We'll store these prompts as templates and load them based on the classification result.

**[00:07:05] Marina Santos:**
Makes sense. And Function 4?

**[00:07:12] João Silva:**
Function 4 is the bigquery-writer. Simple job - takes the structured JSON from the extractor and writes it to BigQuery. It also handles schema validation, deduplication, and error logging.

**[00:07:40] Pedro Lima:**
I have a question about the connection between functions. Are we using Pub/Sub for everything?

**[00:07:52] João Silva:**
Yes. Here's the flow:

[João sharing screen]

**[00:08:05] João Silva:**
```
File lands in GCS
  → triggers Pub/Sub topic "invoice-uploaded"
  → tiff-to-png-converter subscribes, processes, publishes to "invoice-converted"
  → invoice-classifier subscribes, processes, publishes to "invoice-classified"
  → data-extractor subscribes, processes, publishes to "invoice-extracted"
  → bigquery-writer subscribes, writes to BigQuery
```

**[00:08:45] Pedro Lima:**
So 4 topics total? invoice-uploaded, invoice-converted, invoice-classified, invoice-extracted?

**[00:09:00] João Silva:**
Exactly. Each topic can have its own dead-letter queue for failed messages.

**[00:09:15] Marina Santos:**
Ana, I want to go back to the prompts. How are we going to test that they work correctly?

**[00:09:28] Ana Costa:**
Great question. We need a test dataset with ground truth - basically, invoices where we know what the correct extraction should be. Carlos is getting us 50 invoices. We'll manually label the expected output and compare against what the LLM extracts.

**[00:09:58] Carlos Ferreira:**
I can do that. Actually, I can probably get one of my team members to label them.

**[00:10:10] Ana Costa:**
Perfect. We measure accuracy per field - invoice_id accuracy, date accuracy, total_amount accuracy, and so on. Our target is 90% on each field.

**[00:10:35] Marina Santos:**
What about observability? Last meeting someone mentioned LangFuse.

**[00:10:45] Ana Costa:**
Yes, LangFuse is great for LLM observability. It captures every call - what prompt was sent, what response came back, latency, token count, cost. Super useful for debugging.

**[00:11:10] Ana Costa:**
But I'd recommend we defer that to Phase 2. For MVP let's focus on getting the core pipeline working. We'll add basic logging and monitoring, but LangFuse integration can come after we have a working system.

**[00:11:35] Marina Santos:**
Agreed. Let's not over-engineer the MVP. Pedro, are you comfortable setting up the Pub/Sub infrastructure?

**[00:11:48] Pedro Lima:**
Yes, I'll have the topics and subscriptions ready by end of week. I'll also set up the GCS buckets - input, processed, archive, and failed.

**[00:12:10] Marina Santos:**
Great. Let me assign the functions:
- João: tiff-to-png-converter and bigquery-writer
- Ana: invoice-classifier and data-extractor
- Pedro: Infrastructure - Pub/Sub, GCS buckets

Everyone ok with that split?

**[00:12:40] João Silva:**
Makes sense. The converter and writer are more data engineering. The classifier and extractor are more ML.

**[00:12:55] Ana Costa:**
Agreed. I'll start with the classifier since it's simpler, then move to the extractor.

**[00:13:10] Carlos Ferreira:**
When can I see it working?

**[00:13:18] Marina Santos:**
We're targeting end of first week of February for a demo with sample invoices. Carlos, if you can have those labeled invoices ready by then, we can show you accuracy metrics.

**[00:13:38] Carlos Ferreira:**
I'll have them by February 10th.

**[00:13:45] Marina Santos:**
Perfect. Any other questions? ... Ok, next meeting we'll discuss the ML strategy in more detail - prompt engineering, model selection, observability. Thanks everyone!

**[Meeting ended at 01:18:22]**

---

## Pipeline Architecture (from João's screen share)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           DATA PIPELINE FLOW                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐│
│   │  .TIFF   │───▶│ FUNCTION │───▶│ FUNCTION │───▶│ FUNCTION │───▶│ FUNCTION ││
│   │  Upload  │    │    1     │    │    2     │    │    3     │    │    4     ││
│   │   GCS    │    │ TIFF→PNG │    │ Classify │    │ Extract  │    │  Write   ││
│   └──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘│
│        │               │               │               │               │       │
│        ▼               ▼               ▼               ▼               ▼       │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐│
│   │ Pub/Sub  │    │ Pub/Sub  │    │ Pub/Sub  │    │ Pub/Sub  │    │ BigQuery ││
│   │  Topic   │    │  Topic   │    │  Topic   │    │  Topic   │    │  Table   ││
│   │ uploaded │    │ converted│    │ classified│   │ extracted│    │ invoices ││
│   └──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘│
│                                                                                 │
│   FUNCTION DETAILS:                                                             │
│   1. tiff-to-png-converter    │ Convert .tiff to .png for LLM consumption      │
│   2. invoice-classifier       │ Validate structure, detect invoice type        │
│   3. data-extractor           │ Gemini 2.0 Flash extraction with prompts       │
│   4. bigquery-writer          │ Write structured data to BigQuery              │
│                                                                                 │
│   GCS BUCKETS:                                                                  │
│   - gs://invoices-input       │ Raw TIFF files land here                       │
│   - gs://invoices-processed   │ Converted PNG files                            │
│   - gs://invoices-archive     │ Original files for retention                   │
│   - gs://invoices-failed      │ Failed processing for review                   │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Extraction Schema (discussed)

| Field | Type | Example |
|-------|------|---------|
| invoice_id | String | "UE-2025-001234" |
| vendor_name | String | "Restaurant ABC" |
| vendor_type | String | "UberEats" |
| invoice_date | Date | "2026-01-15" |
| due_date | Date | "2026-02-15" |
| subtotal | Float | 1234.56 |
| tax_amount | Float | 123.45 |
| total_amount | Float | 1358.01 |
| line_items | Array | [{description, quantity, unit_price, amount}] |

---

## Notes & Observations

- Team aligned on modular function approach - good for debugging and scaling
- Different prompts for different invoice types - need to build prompt template library
- LangFuse deferred to Phase 2 - pragmatic MVP approach
- Carlos engaged - willing to have team label ground truth data
- 4 Pub/Sub topics creates good observability points
- Need to think about retry logic and DLQ strategy per topic

---

*Generated by Krisp AI Meeting Assistant*
