---
id: sre-incident-response-workflow
title: Incident Response Workflow
type: workflow
version: "1.0.0"
status: active
workstream:
  - sre
author: ai-skills-maintainer
created: 2026-04-20
updated: 2026-04-20
tags:
  - incident-response
  - sre
  - on-call
  - workflow
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
description: |
  A four-step workflow that takes a raw alert through triage, runbook retrieval,
  mitigation planning, and stakeholder communication — giving on-call engineers a
  complete, structured incident response in minutes rather than having to assemble
  context from scratch under pressure.
security_classification: internal
delivery_modes:
  - llm-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
dependencies:
  - sre-incident-triage-prompt
  - sre-runbook-generator
inputs:
  - name: ALERT_NAME
    type: string
    description: The name of the firing alert as it appears in the alerting system
    required: true
  - name: SERVICE_NAME
    type: string
    description: Name of the affected service
    required: true
  - name: SYSTEM_CONTEXT
    type: string
    description: Brief description of what the service does and its key dependencies
    required: true
  - name: SEVERITY
    type: string
    description: "Known or estimated severity: SEV1, SEV2, or SEV3. If omitted, Step 1 will infer it."
    required: false
outputs:
  - name: INCIDENT_REPORT
    type: string
    description: Consolidated incident report containing triage result, runbook, mitigation steps, and stakeholder communication
---

## Overview

This workflow orchestrates the four essential phases of incident response for an on-call SRE. Each step produces a well-defined output that feeds the next step. The steps are designed as self-contained, copy-pasteable LLM prompts so the workflow can be run manually in any LLM chat interface or automated via the `sre-on-call-alert-handler` automation.

**Audience:** On-call SRE engineers, incident commanders, automated alert pipelines.

## Steps

### Step 1 — Triage Alert and Assess Severity

**Asset used:** `sre-incident-triage-prompt`  
**Input:** `{{ALERT_NAME}}`, `{{SERVICE_NAME}}`, `{{SYSTEM_CONTEXT}}`, `{{SEVERITY}}` (optional)  
**Output:** `TRIAGE_RESULT` — confirmed severity with justification and initial escalation guidance.

```
[Prompt — paste into any LLM]

You are a senior SRE performing initial incident triage.

Alert name: {{ALERT_NAME}}
Service: {{SERVICE_NAME}}
Service context: {{SYSTEM_CONTEXT}}
Indicated severity (if known): {{SEVERITY}}

Assess this alert and produce a TRIAGE_RESULT with the following sections:

### Severity
State SEV1, SEV2, or SEV3 with a one-sentence justification.
Use these definitions:
- SEV1: Service is completely unavailable or customer data is at risk. Page secondary on-call immediately.
- SEV2: Service is degraded; significant customer impact likely within 15 minutes without intervention.
- SEV3: Minor degradation; no immediate customer impact; can be addressed in business hours.

### Initial Hypotheses
List 2–3 likely root causes ranked by probability, given the alert name and service context.

### Immediate Actions (before runbook)
List up to 3 actions the engineer should take in the next 2 minutes (e.g., check dashboards,
verify recent deploys, assess error rate).

### Escalation Guidance
State who should be paged and when based on the severity assessment.
```

---

### Step 2 — Retrieve / Generate Runbook

**Asset used:** `sre-runbook-generator`  
**Input:** `TRIAGE_RESULT` (from Step 1), `{{ALERT_NAME}}`, `{{SERVICE_NAME}}`, `{{SYSTEM_CONTEXT}}`  
**Output:** `RUNBOOK_REFERENCE` — a complete runbook for this alert and service.

```
[Prompt — paste into any LLM after Step 1]

You are a senior SRE generating an operational runbook.

Alert name: {{ALERT_NAME}}
Service: {{SERVICE_NAME}}
Service context: {{SYSTEM_CONTEXT}}
Triage result (from Step 1): {{TRIAGE_RESULT}}

Generate a complete runbook using this exact Markdown structure:

# Runbook: {{ALERT_NAME}} — {{SERVICE_NAME}}

## Overview
What this runbook covers and when it applies.

## Default Severity
{severity from TRIAGE_RESULT} — brief justification.

## Impact
Who is affected and how when this alert fires.

## Escalation Path
| Step | Who | How | When |
|---|---|---|---|
| 1 | On-call engineer | PagerDuty / Slack | Immediately |
| 2 | Service owner | Slack / Phone | After 15 min without resolution |
| 3 | Engineering manager | Phone | SEV1 only, after 30 min |

## Pre-Investigation Checklist
- [ ] Check the status page for any ongoing platform incidents.
- [ ] Verify recent deployments in the last 2 hours.
- [ ] Check dependent services for degradation.

## Diagnostic Steps
Number each step. Use realistic placeholder CLI commands in code blocks.

## Resolution Procedures
### Scenario A: [Most common cause]
Steps to resolve.

### Scenario B: [Second most common cause]
Steps to resolve.

## Rollback Procedure
Steps to safely roll back if resolution fails.

## Post-Incident Actions
- [ ] Update this runbook with any new findings.
- [ ] File a post-incident review if SEV1 or SEV2.
- [ ] Update monitoring/alerting if the alert misfired.
```

---

### Step 3 — Propose Mitigation Steps

**Asset used:** *(inline prompt)*  
**Input:** `RUNBOOK_REFERENCE` (Step 2), `TRIAGE_RESULT` (Step 1), `{{ALERT_NAME}}`, `{{SERVICE_NAME}}`  
**Output:** `MITIGATION_STEPS` — a numbered, immediately actionable mitigation plan.

```
[Prompt — paste into any LLM after Step 2]

You are a senior SRE. Based on the triage result and runbook below, produce a
MITIGATION_STEPS list for the on-call engineer to follow right now.

Alert: {{ALERT_NAME}}
Service: {{SERVICE_NAME}}
Triage result: {{TRIAGE_RESULT}}
Runbook: {{RUNBOOK_REFERENCE}}

Rules for MITIGATION_STEPS:
- Number each step starting at 1.
- Each step must be a single, concrete action (not a category or heading).
- Include the exact CLI command or UI action where applicable.
- Add a "(verify)" note after each step that has a measurable outcome the engineer can check.
- If severity is SEV1, include "Page secondary on-call" as step 1.
- Limit to 10 steps maximum. Prioritise actions that restore service fastest.
- Flag any step that requires a change-freeze approval or a second engineer to sign off.
```

---

### Step 4 — Draft Stakeholder Communication

**Asset used:** *(inline prompt)*  
**Input:** `TRIAGE_RESULT` (Step 1), `MITIGATION_STEPS` (Step 3), `{{SERVICE_NAME}}`  
**Output:** `STAKEHOLDER_COMMUNICATION` — non-technical status update and final `INCIDENT_REPORT`.

```
[Prompt — paste into any LLM after Step 3]

You are an SRE drafting stakeholder communications during an active incident.

Service: {{SERVICE_NAME}}
Triage result: {{TRIAGE_RESULT}}
Mitigation steps (summary): {{MITIGATION_STEPS}}

Produce two outputs:

### Status Page Update (under 75 words, present tense, no jargon)
Suitable for posting publicly or to #incidents Slack channel. State what is affected,
current status, and expected next update time.

### Internal Incident Summary (for engineering leadership and on-call handoff)
Include: severity, affected service, customer impact, current mitigation status,
next steps, and incident commander name (use "TBD" if not yet assigned).

Then produce a final INCIDENT_REPORT by combining the outputs of Steps 1–4 under these headings:
## Triage Result
## Runbook Reference
## Mitigation Steps
## Stakeholder Communication
```

---

## Flow Diagram

```
ALERT_NAME + SERVICE_NAME + SYSTEM_CONTEXT [+ SEVERITY]
         │
         ▼
[Step 1: Triage Alert] ──▶ TRIAGE_RESULT
         │
         ▼
[Step 2: Retrieve / Generate Runbook] ──▶ RUNBOOK_REFERENCE
         │
         ▼
[Step 3: Propose Mitigation Steps] ──▶ MITIGATION_STEPS
         │
         ▼
[Step 4: Draft Stakeholder Communication] ──▶ STAKEHOLDER_COMMUNICATION
         │
         ▼
         INCIDENT_REPORT
```

## Usage

### Manual (Copilot Chat — current)

1. Receive the alert from PagerDuty or Alertmanager.
2. Copy Step 1 prompt, substitute placeholders, and paste into LLM Chat or your preferred LLM.
3. Save the `TRIAGE_RESULT` output and use it as input for Step 2.
4. Continue through Steps 3 and 4 in order, carrying forward each step's output.
5. Post the `STAKEHOLDER_COMMUNICATION` to the #incidents Slack channel.
6. Save the `INCIDENT_REPORT` to your incident management system.

### Automated (Bitbucket Pipelines — current)

See `workstreams/sre/automations/on-call-alert-handler.md` for a ready-to-use Bitbucket Pipelines implementation of this workflow.

### API Endpoint (future portal)

```http
POST {{API_BASE_URL}}/packages/sre-incident-response-pack/invoke
Content-Type: application/json

{
  "workflow_id": "sre-incident-response-workflow",
  "inputs": {
    "alert_name": "DBConnectionPoolExhausted",
    "service_name": "payment-service",
    "system_context": "Handles all card payments. Depends on Postgres, Redis, and Stripe.",
    "severity": "SEV1"
  }
}
```

## Examples

### Example 1 — payment-service DB connection pool exhaustion

**Input:**
- `ALERT_NAME`: `DBConnectionPoolExhausted`
- `SERVICE_NAME`: `payment-service`
- `SYSTEM_CONTEXT`: `Java Spring Boot service handling all card payment authorisations. Depends on Postgres (RDS, max_connections=200), Redis session cache, and the Stripe API. Traffic: ~2 000 req/min peak.`
- `SEVERITY`: *(omitted)*

**Step 1 Output (TRIAGE_RESULT):**
```
### Severity
SEV1 — DB connection pool exhaustion on the payment service will cause all payment
authorisations to fail with 500 errors, resulting in immediate revenue loss.

### Initial Hypotheses
1. Traffic spike exceeding pool capacity (most likely — peak trading period).
2. Long-running queries holding connections (check pg_stat_activity).
3. Connection leak after a recent deployment change.

### Immediate Actions
1. Open the payment-service RDS dashboard and check active connection count vs. max_connections.
2. Run `kubectl rollout history deployment/payment-service -n payments` to check for recent deploys.
3. Check the Stripe API status page for downstream errors that might be stalling connections.

### Escalation Guidance
SEV1 — page secondary on-call engineer and notify the payment service owner immediately.
```

**Step 3 Output (MITIGATION_STEPS excerpt):**
```
1. Page secondary on-call engineer via PagerDuty now (SEV1 protocol).
2. Check active connections: `psql -h $DB_HOST -U $DB_USER -c "SELECT count(*) FROM pg_stat_activity WHERE datname='payments';"` (verify: count < 200)
3. Kill longest-running idle connections: `psql -h $DB_HOST -U $DB_USER -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE state='idle' AND query_start < NOW() - INTERVAL '5 minutes';"` (verify: active connection count drops)
4. If a deploy occurred in the last 2 hours, roll back: `kubectl rollout undo deployment/payment-service -n payments` (verify: new pods start healthy)
5. Scale the deployment to distribute load: `kubectl scale deployment/payment-service --replicas=8 -n payments` (verify: error rate drops in Datadog)
```

**Step 4 Output (Status Page Update):**
```
We are currently experiencing issues with our payment service. Some payment
transactions may fail. Our engineering team is actively investigating and working
to restore full service. Next update in 15 minutes.
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | 2026-04-20. All four steps work in a single Copilot Chat session. Step 1 severity inference is accurate for well-named alerts. |
| claude-3-5-sonnet | ✅ | 2026-04-20. Step 2 runbook generation is more thorough. Step 4 stakeholder language occasionally too technical — enforce the 75-word limit explicitly. |

## Changelog

### 1.0.0 — 2026-04-20
- Initial version.
