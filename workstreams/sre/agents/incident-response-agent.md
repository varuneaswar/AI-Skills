---
id: sre-incident-response-agent
title: Incident Response Agent
type: agent
version: "1.0.0"
status: active
workstream:
  - sre
author: ai-skills-maintainer
created: 2026-04-20
updated: 2026-04-20
tags:
  - incident-response
  - on-call
  - automation
  - triage
  - sre
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
description: |
  An autonomous agent that manages the full incident response lifecycle: triages alert
  severity, retrieves or generates the relevant runbook, proposes concrete mitigation
  steps, and drafts a stakeholder summary — so that on-call engineers spend their time
  fixing issues rather than assembling context.
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
  - name: ALERT_DETAILS
    type: string
    description: Full text of the firing alert — name, labels, annotations, and current metric values
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
    description: "Override severity if already known: SEV1, SEV2, or SEV3. If omitted the agent infers it."
    required: false
outputs:
  - name: TRIAGE_RESULT
    type: string
    description: Confirmed severity level with justification
  - name: RUNBOOK_REFERENCE
    type: string
    description: Retrieved or generated runbook applicable to the incident
  - name: MITIGATION_STEPS
    type: string
    description: Ordered list of immediate mitigation actions
  - name: STAKEHOLDER_SUMMARY
    type: string
    description: Non-technical summary suitable for posting to a status page or stakeholder Slack channel
---

## Overview

The Incident Response Agent replaces the first 15–30 minutes of manual on-call triage. It accepts an alert payload, autonomously determines severity, retrieves or generates the matching runbook, proposes an ordered mitigation plan, and composes a stakeholder communication — all before the engineer has finished reading the PagerDuty notification.

Use it as the first responder for all SEV1 and SEV2 alerts. A human on-call engineer must still validate and execute the proposed actions.

## Architecture

**Skills / Tools Used:**

| Tool / Skill | Purpose |
|---|---|
| `triage_alert` | Assess alert severity and confirm or override the provided SEVERITY input |
| `retrieve_runbook` | Fetch an existing runbook from the knowledge base; fall back to `sre-runbook-generator` if none found |

**Decision Flow:**

```
ALERT_DETAILS + SERVICE_NAME + SYSTEM_CONTEXT [+ SEVERITY]
         │
         ▼
Step 1: Triage alert (triage_alert)
         │
         ├─ SEV1 ──▶ Page secondary on-call immediately (note in TRIAGE_RESULT)
         │
         ├─ SEV2 ──▶ Continue to Step 2; note 15-min escalation window
         │
         └─ SEV3 ──▶ Continue to Step 2; no immediate escalation required
                  │
                  ▼
         Step 2: Retrieve / generate runbook (retrieve_runbook → sre-runbook-generator)
                  │
                  ▼
         Step 3: Propose mitigation steps derived from runbook + alert details
                  │
                  ▼
         Step 4: Draft stakeholder summary (non-technical, status-page ready)
                  │
                  ▼
         TRIAGE_RESULT + RUNBOOK_REFERENCE + MITIGATION_STEPS + STAKEHOLDER_SUMMARY
```

## System Prompt / Agent Instructions

```
[System Prompt]

You are an autonomous incident response agent for a site reliability engineering team.

Your goals:
1. Triage the incoming alert to confirm or assign a severity level (SEV1 / SEV2 / SEV3).
2. Retrieve the most relevant runbook for this alert and service; if no runbook exists,
   generate one using the sre-runbook-generator skill.
3. Propose an ordered list of immediate mitigation steps the on-call engineer should execute.
4. Draft a concise, non-technical stakeholder summary suitable for a status-page update
   or a Slack #incidents channel.

Tools available to you:
- triage_alert: call with (alert_details, service_name, system_context, severity_hint)
  → returns confirmed SEVERITY and TRIAGE_JUSTIFICATION
- retrieve_runbook: call with (service_name, alert_name)
  → returns RUNBOOK_CONTENT or null if not found

Rules:
- If triage_alert returns SEV1, prepend a 🔴 SEV1 banner to TRIAGE_RESULT and explicitly
  state "Page secondary on-call now."
- Never skip runbook retrieval even if a SEVERITY is provided — always confirm first.
- If retrieve_runbook returns null, immediately call sre-runbook-generator to generate one.
- Mitigation steps must be numbered, actionable, and reference actual commands or
  procedures from the runbook where available.
- The stakeholder summary must be free of jargon, under 100 words, and written in
  present tense (e.g., "We are investigating…").
- Do not speculate about root cause without evidence; use "potential" or "suspected"
  language until confirmed.

Output format:

## Triage Result
{TRIAGE_RESULT}

## Runbook Reference
{RUNBOOK_REFERENCE}

## Mitigation Steps
{MITIGATION_STEPS}

## Stakeholder Summary
{STAKEHOLDER_SUMMARY}
```

## Configuration

```yaml
agent_id: sre-incident-response-agent
model: gpt-4o
temperature: 0.1         # low temperature for deterministic, safe incident guidance
max_iterations: 6        # triage → retrieve_runbook → [generate_runbook] → mitigate → draft → format
tools:
  - triage_alert
  - retrieve_runbook
```

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `{{ALERT_DETAILS}}` | string | Yes | Full alert text including name, labels, and metric values |
| `{{SERVICE_NAME}}` | string | Yes | Affected service name |
| `{{SYSTEM_CONTEXT}}` | string | Yes | What the service does and its key dependencies |
| `{{SEVERITY}}` | string | No | Override: `SEV1`, `SEV2`, or `SEV3` |

## Outputs

| Name | Type | Description |
|---|---|---|
| `TRIAGE_RESULT` | string | Confirmed severity with justification |
| `RUNBOOK_REFERENCE` | string | Retrieved or generated runbook |
| `MITIGATION_STEPS` | string | Ordered mitigation actions |
| `STAKEHOLDER_SUMMARY` | string | Non-technical status-page update |

## Security Considerations

- The agent proposes mitigation steps but never executes commands directly — a human engineer must approve and run them.
- Alert payloads may contain sensitive metric data; route through `in-house-llm` for services that handle PII or financial data.
- The agent must never include credentials, tokens, or secrets encountered in the alert payload in any output. Flag them as a Critical finding in TRIAGE_RESULT instead.
- Limit tool call permissions to read-only operations on the runbook knowledge base.

## Usage

### LLM Chat (current)

Paste the alert details into Copilot Chat:

```
You are an incident response agent.
Alert: {{ALERT_DETAILS}}
Service: {{SERVICE_NAME}}
Context: {{SYSTEM_CONTEXT}}
Triage this alert, retrieve the relevant runbook, propose mitigation steps,
and draft a stakeholder summary.
```

### API Endpoint (future portal)

```http
POST {{API_BASE_URL}}/packages/sre-incident-response-pack/invoke
Content-Type: application/json

{
  "agent_id": "sre-incident-response-agent",
  "inputs": {
    "alert_details": "Alert: HighMemoryUsage | service=payment-service | memory_used=94% | threshold=90%",
    "service_name": "payment-service",
    "system_context": "Handles all card payment authorisations. Depends on Postgres, Redis, and the Stripe API.",
    "severity": "SEV2"
  }
}
```

### MCP Tool (future)

```json
{
  "tool": "sre-incident-response-agent",
  "parameters": {
    "alert_details": "Alert: HighMemoryUsage | service=payment-service | memory_used=94%",
    "service_name": "payment-service",
    "system_context": "Handles all card payment authorisations. Depends on Postgres, Redis, and Stripe.",
    "severity": "SEV2"
  }
}
```

## Examples

### Example 1 — HighMemoryUsage on payment-service

**Input:**
- `ALERT_DETAILS`: `Alert: HighMemoryUsage | service=payment-service | pod=payment-service-7d9f8b-xkp2q | memory_used=94% | threshold=90% | duration=8m`
- `SERVICE_NAME`: `payment-service`
- `SYSTEM_CONTEXT`: `Handles all card payment authorisations. Java Spring Boot application. Depends on Postgres (RDS), Redis (session cache), and the Stripe API. Traffic: ~2 000 req/min at peak.`
- `SEVERITY`: *(omitted — let agent infer)*

**Output:**

```
## Triage Result
🟠 SEV2 — Memory usage at 94% on a single pod for 8 minutes. Payments are currently
processing but pod OOM termination is imminent. If a second pod enters the same state,
payment processing will be fully degraded (SEV1). No secondary on-call page required yet;
reassess in 10 minutes if memory does not stabilise.

## Runbook Reference
# Runbook: HighMemoryUsage — payment-service

### Overview
Fires when JVM heap or container memory exceeds 90% for more than 5 minutes.

### Diagnostic Steps
1. Check live memory metrics across all pods:
   `kubectl top pods -l app=payment-service -n payments`
2. Capture a heap dump from the affected pod:
   `kubectl exec payment-service-7d9f8b-xkp2q -n payments -- jcmd 1 GC.heap_info`
3. Review recent GC logs for full GC storms:
   `kubectl logs payment-service-7d9f8b-xkp2q -n payments | grep -i "GC\|OOM" | tail -50`
4. Check for recent deployments that may have changed heap settings:
   `kubectl rollout history deployment/payment-service -n payments`

### Resolution Procedures
#### Scenario A: Memory leak after recent deploy
Roll back: `kubectl rollout undo deployment/payment-service -n payments`

#### Scenario B: Traffic spike causing heap pressure
Scale out: `kubectl scale deployment/payment-service --replicas=6 -n payments`

## Mitigation Steps
1. Run `kubectl top pods -l app=payment-service -n payments` to confirm which pods are affected.
2. If only one pod is affected, cordon it: `kubectl cordon payment-service-7d9f8b-xkp2q`.
3. Scale the deployment to add capacity: `kubectl scale deployment/payment-service --replicas=6 -n payments`.
4. Capture a heap dump for post-incident analysis before the pod recycles.
5. Check for deployments in the past 2 hours (`kubectl rollout history`) — if found, prepare a rollback command.
6. Reassess memory metrics in 5 minutes. If usage does not drop below 80%, escalate to SEV1.

## Stakeholder Summary
We are investigating elevated memory usage on the payment service. Payments are currently
processing normally. Our team has taken steps to add capacity and is monitoring the situation
closely. We will provide an update in 15 minutes.
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | 2026-04-20. Triage accuracy high. Runbook retrieval fallback to generator works correctly. |
| claude-3-5-sonnet | ✅ | 2026-04-20. More detailed diagnostic commands. Stakeholder summary occasionally too technical — may need a temperature tweak. |

## Changelog

### 1.0.0 — 2026-04-20
- Initial version.
