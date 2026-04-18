---
id: sre-runbook-generator
title: Runbook Generator
type: skill
version: "1.0.0"
status: active
workstream:
  - sre
author: ai-skills-maintainer
created: 2024-01-01
updated: 2024-01-01
tags:
  - runbook
  - incident-response
  - operations
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
  - copilot-gpt-4o
description: |
  Generates a structured operational runbook for a service or alert, including
  description, escalation path, diagnostic steps, and resolution procedures.
security_classification: internal
inputs:
  - name: SERVICE_NAME
    type: string
    description: Name of the service this runbook covers
    required: true
  - name: ALERT_OR_SCENARIO
    type: string
    description: The alert name or operational scenario the runbook addresses
    required: true
  - name: SYSTEM_CONTEXT
    type: string
    description: Brief description of what the service does and its dependencies
    required: true
  - name: SEVERITY
    type: string
    description: "Default severity level: SEV1, SEV2, or SEV3"
    required: false
outputs:
  - name: RUNBOOK
    type: string
    description: Completed runbook in Markdown format
delivery_modes:
  - copilot-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
---

## Overview

Generates a first-draft runbook that the team can then refine and add real diagnostic commands to. Saves significant time compared to writing from scratch. The output follows a standard SRE runbook structure.

**Important:** Replace all placeholder commands and endpoints with real values before using in production.

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `{{SERVICE_NAME}}` | string | Yes | The service name |
| `{{ALERT_OR_SCENARIO}}` | string | Yes | The alert or scenario |
| `{{SYSTEM_CONTEXT}}` | string | Yes | What the service does and its dependencies |
| `{{SEVERITY}}` | string | No | Default severity: `SEV1`, `SEV2`, `SEV3` |

## Outputs

| Name | Type | Description |
|---|---|---|
| `RUNBOOK` | string | Complete Markdown runbook |

## Skill Definition

```
[System Prompt]

You are a senior SRE helping your team build operational runbooks. Your runbooks are
clear, actionable, and structured consistently so on-call engineers can follow them
under pressure.

Generate a complete runbook for the following:
Service: {{SERVICE_NAME}}
Alert / Scenario: {{ALERT_OR_SCENARIO}}
Service context: {{SYSTEM_CONTEXT}}
Default severity: {{SEVERITY}}

Use this exact Markdown structure:

# Runbook: {{ALERT_OR_SCENARIO}} — {{SERVICE_NAME}}

## Overview
What this runbook covers and when it applies.

## Default Severity
{{SEVERITY}} — brief justification.

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
Number each step. Use placeholder CLI commands in backtick code blocks.

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

## Related Resources
- Link to service dashboard: {{DASHBOARD_URL}}
- Link to architecture doc: {{ARCH_DOC_URL}}
- Slack channel: #{{SLACK_CHANNEL}}
```

## Usage

### GitHub Copilot Chat (current)

```
Generate a runbook for the "{{ALERT_OR_SCENARIO}}" alert on {{SERVICE_NAME}}.
Service context: {{SYSTEM_CONTEXT}}
```

### API Endpoint (future portal)

```http
POST {{API_BASE_URL}}/packages/sre-incident-response-pack/invoke
Content-Type: application/json

{
  "skill_id": "sre-runbook-generator",
  "inputs": {
    "service_name": "checkout-service",
    "alert_or_scenario": "DB connection pool exhausted",
    "system_context": "Handles all payment processing. Depends on Postgres and Stripe API.",
    "severity": "SEV1"
  }
}
```

## Examples

### Example 1

**Input:**
- `SERVICE_NAME`: `checkout-service`
- `ALERT_OR_SCENARIO`: `DB connection pool exhausted`
- `SYSTEM_CONTEXT`: `Handles all payment flows. PostgreSQL backend. Upstream: API gateway. Downstream: Stripe, loyalty-service.`
- `SEVERITY`: `SEV1`

**Output (excerpt):**
```markdown
# Runbook: DB Connection Pool Exhausted — checkout-service

## Default Severity
SEV1 — Payment processing is fully blocked when the DB pool is exhausted.

## Impact
All checkout requests fail with 500. Revenue impact is immediate.

## Diagnostic Steps
1. Check current connection count:
   `psql -c "SELECT count(*) FROM pg_stat_activity WHERE datname='checkout';"`
2. Identify queries holding connections longest:
   `psql -c "SELECT pid, query, state, query_start FROM pg_stat_activity ORDER BY query_start ASC LIMIT 20;"`
3. Check for recent deployments:
   `kubectl rollout history deployment/checkout-service`
...
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | 2024-01-01. Good structure. Diagnostic commands are illustrative — team must replace with real ones. |
| claude-3-5-sonnet | ✅ | 2024-01-01. Thorough escalation paths. |

## Changelog

### 1.0.0 — 2024-01-01
- Initial version.
