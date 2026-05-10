# Business Kickoff: UberEats Invoice Processing

**Date:** January 15, 2026 | **Duration:** 47 minutes | **Platform:** Google Meet

**Participants:**
- Marina Santos (Product Manager)
- João Silva (Senior Data Engineer)
- Ana Costa (ML Engineer)
- Pedro Lima (Platform/DevOps Lead)
- Carlos Ferreira (Business Stakeholder - Restaurant Operations)

---

## Summary

Business kickoff meeting to discuss the manual invoice processing problem affecting restaurant partner reconciliation. Carlos presented the current pain points: 3 FTEs spending 80% of their time on manual data entry from delivery platform invoices. Team agreed to build an AI-powered extraction pipeline targeting 90%+ accuracy. Timeline pressure due to Q2 financial close requirements.

---

## Key Decisions

| # | Decision | Owner | Status |
|---|----------|-------|--------|
| 1 | Build automated invoice extraction pipeline | Marina | Approved |
| 2 | Target 90% extraction accuracy as MVP threshold | Ana | Approved |
| 3 | Start with UberEats invoices only (largest volume) | Carlos | Approved |
| 4 | Use cloud-native architecture (serverless) | João | Approved |

---

## Action Items

- [ ] **João**: Set up initial GCP project and access permissions (Due: Jan 17, 2026)
- [ ] **Ana**: Research LLM options for document extraction (Due: Jan 20, 2026)
- [ ] **Marina**: Define detailed success criteria document (Due: Jan 18, 2026)
- [ ] **Pedro**: Evaluate infrastructure requirements (Due: Jan 22, 2026)
- [ ] **Carlos**: Provide sample invoice dataset (minimum 50 invoices) (Due: Jan 17, 2026)

---

## Next Steps

1. Technical architecture review meeting scheduled for Jan 22, 2026
2. Carlos to share access to invoice repository
3. Marina to circulate success criteria for async review

---

## Full Transcript

**[00:00:15] Marina Santos:**
Ok everyone, thanks for joining. Today we're kicking off the invoice processing project. Carlos, do you want to start by walking us through the problem?

**[00:00:35] Carlos Ferreira:**
Sure. So basically, we have this massive problem with invoice reconciliation. Every month we receive thousands of invoices from UberEats, DoorDash, Grubhub... and right now we have three full-time people just entering this data manually into our systems.

**[00:01:12] João Silva:**
Three FTEs? That's... that seems like a lot. What exactly are they doing?

**[00:01:22] Carlos Ferreira:**
They open each PDF or TIFF file, look for the invoice number, vendor name, date, line items, totals... and type it all into SAP. It's tedious and error-prone. We had two significant reconciliation errors last quarter that cost us about R$45,000 to fix.

**[00:01:58] Ana Costa:**
And what's the volume we're talking about here?

**[00:02:05] Carlos Ferreira:**
About 2,000 invoices per month currently. But that's growing. We expect maybe 3,500 by end of year with the new restaurant partnerships.

**[00:02:25] Pedro Lima:**
Are all these invoices in the same format?

**[00:02:32] Carlos Ferreira:**
No, that's part of the problem. UberEats has one format, DoorDash another... even UberEats changes their format sometimes. It's a mess honestly.

**[00:02:55] Marina Santos:**
Ok so let me summarize. We have high volume, manual process, variable formats, and it's causing real financial impact. João, from a technical perspective, is this something we can automate?

**[00:03:18] João Silva:**
Definitely. This is exactly the kind of problem that modern LLMs are good at. We can build a pipeline that receives the invoice, extracts the data using AI, and pushes it to our systems. The question is accuracy - Carlos, what accuracy would be acceptable?

**[00:03:48] Carlos Ferreira:**
We need at least 90%. Below that and we're still spending too much time on corrections.

**[00:04:02] Ana Costa:**
90% is achievable with the current state of vision-language models. I've seen benchmarks where Gemini and GPT-4V hit 95%+ on structured document extraction. But we'll need to test with your specific invoice formats.

**[00:04:28] Marina Santos:**
Perfect. So our success criteria is 90% accuracy. Ana, can you research what LLM would be best for this?

**[00:04:42] Ana Costa:**
Yes, I'll look at Gemini, GPT-4, and maybe some of the open source options through OpenRouter. Cost is going to be a factor with 2000+ invoices per month.

**[00:05:05] Pedro Lima:**
On the infrastructure side, are we thinking cloud or on-prem?

**[00:05:15] João Silva:**
Cloud for sure. Serverless specifically. We don't want to manage servers for this. I'm thinking GCP since we're already using BigQuery for analytics.

**[00:05:35] Marina Santos:**
Makes sense. Let's keep it in the GCP ecosystem. João, can you set up the project?

**[00:05:48] João Silva:**
Yeah, I'll create the GCP project and set up access for everyone by end of week.

**[00:06:02] Marina Santos:**
Carlos, one more thing - can we start with just UberEats invoices? They're the largest volume right?

**[00:06:15] Carlos Ferreira:**
Yes, about 60% of our invoices are UberEats. Makes sense to start there. I can get you a sample dataset... maybe 50 invoices to start testing?

**[00:06:35] Marina Santos:**
Perfect. Get those to João by Wednesday if possible.

**[00:06:45] Carlos Ferreira:**
Will do.

**[00:07:00] Marina Santos:**
Ok I think we have a solid foundation. Let me summarize the decisions:
1. We're building an automated extraction pipeline
2. 90% accuracy is our target
3. Starting with UberEats only
4. Using GCP serverless architecture

Any objections? ... Ok great. I'll schedule the technical architecture review for next week. Thanks everyone!

**[Meeting ended at 00:47:22]**

---

## Notes & Observations

- Carlos seems very frustrated with current process - good internal champion
- Ana has experience with document extraction - she mentioned previous project at her old company
- Need to clarify: what happens when extraction fails? Manual review queue?
- Budget not discussed yet - need to address in next meeting
- Timeline pressure: Q2 close is April 30, 2026, need MVP by April 1, 2026

---

*Generated by Krisp AI Meeting Assistant*
