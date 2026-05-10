# Autonomous Operations Review: Self-Healing Pipeline Vision

**Date:** February 17, 2026 | **Duration:** 72 minutes | **Platform:** Google Meet

**Participants:**
- Marina Santos (Product Manager)
- João Silva (Senior Data Engineer)
- Ana Costa (ML Engineer)
- Pedro Lima (Platform/DevOps Lead)
- Carlos Ferreira (Business Stakeholder - Restaurant Operations)

---

## Summary

Final planning meeting covering the autonomous operations vision. João presented CrewAI for building AI agents that monitor and respond to pipeline issues. Team agreed on three-agent architecture: Triage Agent, Root Cause Agent, and Reporter Agent. Logs flow from Cloud Logging → GCS → CrewAI pipeline. Vision is self-healing pipeline that can identify issues, determine root cause, and either fix automatically or escalate to humans. Full end-to-end system review completed.

---

## Key Decisions

| # | Decision | Owner | Status |
|---|----------|-------|--------|
| 1 | Implement CrewAI for autonomous monitoring | João | Approved |
| 2 | Three-agent architecture (Triage, Root Cause, Reporter) | João | Approved |
| 3 | Cloud Logging → GCS export for agent consumption | Pedro | Approved |
| 4 | Slack integration for alerts and reports | Pedro | Approved |
| 5 | Start with monitoring-only, add auto-remediation later | João | Approved |
| 6 | Weekly autonomous ops review cadence | Marina | Approved |

---

## Action Items

- [ ] **João**: Set up CrewAI project structure (Due: Feb 21, 2026)
- [ ] **João**: Define agent prompts and capabilities (Due: Feb 24, 2026)
- [ ] **Pedro**: Configure Cloud Logging export to GCS (Due: Feb 20, 2026)
- [ ] **Pedro**: Set up Slack webhook for agent notifications (Due: Feb 21, 2026)
- [ ] **Ana**: Define error patterns for Triage Agent (Due: Feb 22, 2026)
- [ ] **Marina**: Create runbook for escalation procedures (Due: Feb 25, 2026)

---

## Next Steps

1. Complete MVP pipeline deployment
2. Generate baseline logs and error patterns
3. Train CrewAI agents on historical data
4. Begin autonomous ops pilot

---

## Full Transcript

**[00:00:12] Marina Santos:**
Ok everyone, this is our final planning meeting. We've covered business, architecture, data flow, ML strategy, and DevOps. Today João is going to tell us about this autonomous operations thing. João?

**[00:00:35] João Silva:**
Thanks Marina. So, we're building a pipeline that processes invoices automatically. But who monitors the pipeline itself? What happens at 3 AM when something breaks?

**[00:00:55] Carlos Ferreira:**
Today our on-call person gets paged and investigates manually.

**[00:01:05] João Silva:**
Exactly. And what if we could have AI agents do that investigation? Not just alert us, but actually analyze what went wrong and suggest fixes - or even fix it automatically.

**[00:01:25] Pedro Lima:**
You're talking about AIOps?

**[00:01:30] João Silva:**
Sort of, but more specific. I'm proposing we use CrewAI to build a team of AI agents that work together to monitor and respond to pipeline issues.

**[00:01:50] Ana Costa:**
CrewAI... that's the multi-agent framework, right?

**[00:01:58] João Silva:**
Yes. [Shares screen] CrewAI lets you define agents with specific roles, tools, and goals. They can work together, share information, and execute tasks.

**[00:02:20] João Silva:**
Here's what I'm proposing - three agents:

**Agent 1: Triage Agent**
- Monitors logs continuously
- Identifies anomalies and errors
- Classifies severity: INFO, WARNING, ERROR, CRITICAL
- Passes relevant events to Root Cause Agent

**Agent 2: Root Cause Agent**
- Receives triaged events
- Analyzes logs, metrics, and context
- Determines likely root cause
- Suggests remediation steps

**Agent 3: Reporter Agent**
- Takes analysis from Root Cause Agent
- Generates human-readable report
- Sends to Slack with recommended actions
- Tracks resolution status

**[00:03:35] Marina Santos:**
So the Triage Agent is like a filter, Root Cause is the investigator, and Reporter is the communicator?

**[00:03:48] João Silva:**
Exactly. They work as a team. Triage says "hey, something's wrong." Root Cause says "here's why and how to fix it." Reporter says "here's a summary for humans."

**[00:04:10] Pedro Lima:**
How do they access the logs?

**[00:04:18] João Silva:**
Good question. Cloud Logging exports to GCS. The agents read from GCS. We can also query Cloud Monitoring for metrics.

**[00:04:40] Pedro Lima:**
I can set up a log sink to export to GCS. We'd structure it by date and severity for easy querying.

**[00:04:58] João Silva:**
Perfect. The agents would run on Cloud Run as well - they can be triggered on a schedule or by new log files appearing.

**[00:05:18] Carlos Ferreira:**
Can you give me an example? Like, what happens when an invoice fails to process?

**[00:05:30] João Silva:**
Sure. Let's say the data-extractor function throws an error because Gemini returns an invalid response.

[João walks through the flow]

1. Error logged to Cloud Logging
2. Exported to GCS within 1 minute
3. Triage Agent sees the log, classifies as ERROR
4. Root Cause Agent analyzes:
   - Looks at the Gemini response
   - Checks if it's a one-off or pattern
   - Sees 5 similar errors in last hour
   - Determines: "Gemini is returning malformed JSON for multi-page invoices"
5. Reporter Agent sends to Slack:
   "🔴 ERROR: Gemini extraction failures detected
   Root Cause: Multi-page invoice handling bug
   Affected: 5 invoices in last hour
   Suggested Fix: Check prompt template for page handling
   Escalation: @ana @joao"

**[00:06:45] Carlos Ferreira:**
That's pretty impressive. So instead of getting "Error 500" alerts, we get actual analysis?

**[00:06:58] João Silva:**
Exactly. The agents do the first level of investigation that a human would do.

**[00:07:10] Ana Costa:**
Can they actually fix things? Or just suggest fixes?

**[00:07:18] João Silva:**
For Phase 1, just suggest. We need to build trust first. But Phase 2 can include auto-remediation for known issues. Like, if the fix is "retry with backoff," the agent can do that automatically.

**[00:07:45] Pedro Lima:**
What about runaway agents? Like, what if the agent starts retrying endlessly?

**[00:07:58] João Silva:**
Good concern. We implement guardrails:
- Maximum retries
- Circuit breaker pattern
- Human-in-the-loop for any action that modifies data
- Audit log of all agent actions

**[00:08:25] Marina Santos:**
I like the phased approach. Monitor first, automate later.

**[00:08:35] João Silva:**
Right. We start conservative. The agents earn trust by being accurate and helpful. Then we give them more capabilities.

**[00:08:55] Marina Santos:**
Ok, let's zoom out. We've now planned the entire system across six meetings. Can someone summarize the end-to-end flow?

**[00:09:10] João Silva:**
I'll give it a shot.

[João pulls up architecture diagram]

**END-TO-END FLOW:**

1. **Ingestion**: TIFF invoice uploaded to GCS
2. **Event**: Pub/Sub triggers pipeline
3. **Processing**:
   - Function 1: TIFF → PNG conversion
   - Function 2: Invoice classification
   - Function 3: Gemini 2.0 Flash extraction
   - Function 4: BigQuery write
4. **Observability**:
   - LangFuse for LLM metrics
   - Cloud Logging for system logs
   - Cloud Monitoring for infrastructure
5. **CI/CD**:
   - GitHub Actions + CodeRabbit for review
   - Terraform + Terragrunt for IaC
   - Separate dev/prod environments
6. **Autonomous Ops**:
   - CrewAI agents for monitoring
   - Triage → Root Cause → Reporter
   - Slack integration for alerts

**[00:10:25] Carlos Ferreira:**
And all of this processes 2000+ invoices per month automatically?

**[00:10:35] João Silva:**
Yes. Humans only get involved when something fails validation or the agents detect issues they can't resolve.

**[00:10:50] Marina Santos:**
What's our target timeline again?

**[00:11:00] Pedro Lima:**
- MVP by end of February 2026: basic pipeline working
- March 2026: LangFuse integration, testing
- April 1, 2026: Production launch for Q2 close
- April-May 2026: CrewAI autonomous ops

**[00:11:25] Marina Santos:**
Perfect. Any risks or concerns we haven't addressed?

**[00:11:35] Ana Costa:**
Model accuracy is still my concern. We're targeting 90% but haven't validated on real production data yet.

**[00:11:50] Marina Santos:**
Let's plan a validation sprint in March. Carlos, can we get production invoices for testing?

**[00:12:02] Carlos Ferreira:**
Yes, I'll coordinate with legal on data handling.

**[00:12:12] Pedro Lima:**
Security review - we should have one before production launch.

**[00:12:22] Marina Santos:**
Good point. I'll schedule that with the security team.

**[00:12:32] Marina Santos:**
Anything else? ... Ok, I think we have a solid plan. Great work everyone over these six meetings. Let's build this thing!

**[00:12:50] All:**
[Applause and celebration]

**[Meeting ended at 01:12:45]**

---

## CrewAI Agent Architecture (from João's slides)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         AUTONOMOUS DATAOPS ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   ┌──────────────────┐                                                          │
│   │   Cloud Logging  │                                                          │
│   │   (all services) │                                                          │
│   └────────┬─────────┘                                                          │
│            │ Log Sink                                                            │
│            ▼                                                                     │
│   ┌──────────────────┐                                                          │
│   │      GCS         │                                                          │
│   │  (logs bucket)   │                                                          │
│   └────────┬─────────┘                                                          │
│            │                                                                     │
│            ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────────┐  │
│   │                           CrewAI Pipeline                                 │  │
│   ├─────────────────────────────────────────────────────────────────────────┤  │
│   │                                                                          │  │
│   │  ┌─────────────┐    ┌─────────────────┐    ┌─────────────────┐         │  │
│   │  │   TRIAGE    │───▶│   ROOT CAUSE    │───▶│    REPORTER     │         │  │
│   │  │   AGENT     │    │     AGENT       │    │     AGENT       │         │  │
│   │  ├─────────────┤    ├─────────────────┤    ├─────────────────┤         │  │
│   │  │ • Read logs │    │ • Analyze error │    │ • Format report │         │  │
│   │  │ • Classify  │    │ • Find pattern  │    │ • Send to Slack │         │  │
│   │  │ • Filter    │    │ • Suggest fix   │    │ • Track status  │         │  │
│   │  └─────────────┘    └─────────────────┘    └─────────────────┘         │  │
│   │                                                                          │  │
│   │  TOOLS:                                                                  │  │
│   │  • GCS Reader     • Log Parser      • Metrics Query                     │  │
│   │  • Pattern Match  • LLM Analysis    • Slack Sender                      │  │
│   │                                                                          │  │
│   └─────────────────────────────────────────────────────────────────────────┘  │
│            │                                                                     │
│            ▼                                                                     │
│   ┌──────────────────┐                                                          │
│   │      Slack       │                                                          │
│   │   #alerts-ops    │                                                          │
│   └──────────────────┘                                                          │
│                                                                                  │
│   FUTURE (Phase 2):                                                             │
│   • Auto-remediation agent                                                       │
│   • Self-healing capabilities                                                    │
│   • Proactive optimization                                                       │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Complete System Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    INVOICE PROCESSING PIPELINE - COMPLETE ARCHITECTURE           │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  INGESTION          PROCESSING                              STORAGE              │
│  ─────────          ──────────                              ───────              │
│                                                                                  │
│  ┌───────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐        │
│  │ TIFF  │──▶│ TIFF→PNG │──▶│ CLASSIFY │──▶│ EXTRACT  │──▶│  WRITE   │──▶ BQ  │
│  │ (GCS) │   │          │   │          │   │ (Gemini) │   │          │        │
│  └───────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘        │
│      │           │              │              │              │                 │
│      └───────────┴──────────────┴──────────────┴──────────────┘                 │
│                              │                                                   │
│                         Pub/Sub (events)                                        │
│                                                                                  │
│  ─────────────────────────────────────────────────────────────────────────────  │
│                                                                                  │
│  OBSERVABILITY                                                                   │
│  ─────────────                                                                   │
│                                                                                  │
│  ┌───────────────┐    ┌───────────────┐    ┌───────────────┐                   │
│  │   LangFuse    │    │ Cloud Logging │    │Cloud Monitor. │                   │
│  │  (LLM calls)  │    │  (all logs)   │    │  (metrics)    │                   │
│  └───────────────┘    └───────────────┘    └───────────────┘                   │
│                               │                                                  │
│                               ▼                                                  │
│  ─────────────────────────────────────────────────────────────────────────────  │
│                                                                                  │
│  AUTONOMOUS OPS (CrewAI)                                                         │
│  ─────────────────────────                                                       │
│                                                                                  │
│  ┌─────────────┐    ┌─────────────────┐    ┌─────────────────┐                 │
│  │   TRIAGE    │───▶│   ROOT CAUSE    │───▶│    REPORTER     │──▶ Slack       │
│  │   AGENT     │    │     AGENT       │    │     AGENT       │                 │
│  └─────────────┘    └─────────────────┘    └─────────────────┘                 │
│                                                                                  │
│  ─────────────────────────────────────────────────────────────────────────────  │
│                                                                                  │
│  CI/CD                                                                           │
│  ─────                                                                           │
│                                                                                  │
│  GitHub → CodeRabbit → GitHub Actions → Terraform → GCP                        │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Project Timeline

| Phase | Dates | Deliverables |
|-------|-------|--------------|
| MVP | Feb 14-28, 2026 | Basic pipeline (4 functions), dev deployment |
| Testing | Mar 1-15, 2026 | LangFuse integration, accuracy validation |
| Launch | Mar 16-Apr 1, 2026 | Production deployment, monitoring |
| AutoOps | Apr 1-30, 2026 | CrewAI agents, autonomous monitoring |

---

## Success Metrics (Final)

| Metric | Target | Owner |
|--------|--------|-------|
| Extraction accuracy | ≥ 90% | Ana |
| Processing latency P95 | < 30s | João |
| Pipeline availability | > 99% | Pedro |
| Cost per invoice | < $0.01 | João |
| Time to detect issues | < 5 min | João (CrewAI) |
| Manual processing reduction | > 80% | Carlos |

---

## Notes & Observations

- Full end-to-end architecture now documented
- CrewAI approach is innovative - positions us as AI-first team
- Phased rollout is smart - earn trust before automation
- Security review needed before production
- Team alignment is strong - good collaboration across meetings
- Carlos is engaged champion - critical for business success

---

*Generated by Krisp AI Meeting Assistant*
