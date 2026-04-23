---
id: sre-incident-triage-prompt
title: Incident Triage First-Responder Guide
type: prompt
version: "1.0.0"
status: active
workstream:
  - sre
author: ai-skills-maintainer
created: 2024-01-01
updated: 2024-01-01
tags:
  - incident-response
  - triage
  - on-call
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
description: |
  Guides the on-call engineer through structured incident triage by analysing alert details
  and recent changes to suggest probable root causes and immediate mitigation steps.
security_classification: internal
inputs:
  - name: ALERT_TITLE
    type: string
    description: The alert or incident title
    required: true
  - name: ALERT_DETAILS
    type: string
    description: Alert body, metric values, log snippets, or error messages
    required: true
  - name: RECENT_CHANGES
    type: string
    description: Recent deployments, config changes, or infrastructure changes (last 24 h)
    required: false
---

## Overview

Use this prompt during the first five minutes of an incident to quickly orient yourself, generate hypotheses about the root cause, and identify immediate steps to mitigate customer impact.

**Important:** Always sanitize or redact PII, credentials, and sensitive internal hostnames before pasting into an external LLM.

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `{{ALERT_TITLE}}` | string | Yes | The alert or incident title |
| `{{ALERT_DETAILS}}` | string | Yes | Alert body, metric values, log snippets (redacted) |
| `{{RECENT_CHANGES}}` | string | No | Recent deployments or config changes in the last 24 h |

## Prompt

```
You are an experienced site reliability engineer helping triage a production incident.

Incident Alert: {{ALERT_TITLE}}

Alert Details:
{{ALERT_DETAILS}}

Recent Changes (last 24 hours):
{{RECENT_CHANGES}}

Provide a structured triage response using the following format:

## Severity Assessment
Estimate the severity (SEV1 / SEV2 / SEV3) and briefly justify your reasoning based on
likely customer impact.

## Probable Root Causes
List up to three hypotheses for the root cause, ranked by likelihood. For each, explain
the reasoning and what evidence would confirm or rule it out.

## Immediate Mitigation Steps
Provide up to five concrete, ordered steps the on-call engineer should take right now to
reduce customer impact. Be specific (e.g., "Roll back deployment X" or "Increase replica
count from 2 to 4").

## Diagnostic Commands / Queries
Suggest up to three CLI commands, log queries (generic format), or dashboard checks that
would help confirm the root cause. Use placeholder values for hostnames and credentials.

## Escalation Recommendation
State whether escalation is needed, and if so, which team or individual should be paged.

Keep the response concise and actionable. The on-call engineer is under time pressure.
```

## Usage

### LLM Chat

```
@workspace
I'm responding to an incident. Alert: <paste alert title>
Details: <paste redacted alert body>
Recent changes: <paste change log>
Help me triage this.
```

### Standalone

1. Copy the prompt above.
2. Replace placeholders with real (redacted) values.
3. Paste into your LLM interface.

## Examples

### Example 1

**Input:**
- `ALERT_TITLE`: `HIGH ERROR RATE: checkout-service 5xx > 5% for 10 minutes`
- `ALERT_DETAILS`: `HTTP 500 error rate: 8.3%. P99 latency: 12 s (normal: 400 ms). Errors concentrated in /api/checkout/confirm endpoint. DB connection pool exhausted errors in logs.`
- `RECENT_CHANGES`: `2024-01-01 14:30 UTC — checkout-service v2.4.1 deployed (adds loyalty points calculation)`

**Output (abbreviated):**
```
## Severity Assessment
SEV1 — Checkout failures directly block revenue. 8% error rate over 10 minutes
indicates significant customer impact.

## Probable Root Causes
1. **DB connection pool leak in v2.4.1** (HIGH) — The new loyalty points calculation
   may be opening DB connections without releasing them. Evidence: pool exhausted errors
   correlate with deployment time.
2. **Increased query load from loyalty calculation** (MEDIUM) — New queries may be
   slower than expected, holding connections longer.
3. **Downstream dependency timeout** (LOW) — A loyalty points service may be timing out,
   cascading into the checkout flow.

## Immediate Mitigation Steps
1. Roll back checkout-service to v2.4.0 immediately.
2. If rollback is not possible, reduce traffic to checkout-service by 50% via load balancer.
3. Increase DB connection pool size as a temporary measure.
4. Monitor error rate after rollback to confirm recovery.
5. Page the checkout team lead for awareness.
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | Tested 2024-01-01. Structured output, actionable steps. |
| claude-3-5-sonnet | ✅ | Tested 2024-01-01. Slightly longer but thorough. |

## Changelog

### 1.0.0 — 2024-01-01
- Initial version.
