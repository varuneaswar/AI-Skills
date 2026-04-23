---
id: sysdesign-system-design-review-agent
title: System Design Review Agent
type: agent
version: "1.0.0"
status: active
workstream:
  - system-designers
author: ai-skills-maintainer
created: 2026-04-20
updated: 2026-04-20
tags:
  - system-design
  - review
  - api
  - completeness
  - agent
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
description: |
  An autonomous agent that reviews a system design document for completeness, identifies
  missing components and design smells, and produces a structured improvement report with
  a completeness score and prioritised recommendations — reducing design review cycles.
security_classification: internal
delivery_modes:
  - llm-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
dependencies:
  - sysdesign-sequence-diagram-generator
  - sysdesign-api-contract-prompt
inputs:
  - name: DESIGN_DOCUMENT
    type: string
    description: The system design document text (Markdown or plain text)
    required: true
  - name: DESIGN_GOALS
    type: string
    description: Comma-separated list of design goals (e.g., "high availability, low latency, security")
    required: true
  - name: CONSTRAINTS
    type: string
    description: "Optional system constraints: budget, technology stack, compliance requirements"
    required: false
  - name: EXISTING_COMPONENTS
    type: string
    description: Optional list of existing system components this design must integrate with
    required: false
outputs:
  - name: REVIEW_REPORT
    type: string
    description: Structured review report with findings categorised by severity
  - name: COMPLETENESS_SCORE
    type: string
    description: Numeric completeness score (0–100) with rationale
  - name: DESIGN_SMELLS
    type: string
    description: List of identified design anti-patterns and their impact
  - name: IMPROVEMENT_SUGGESTIONS
    type: string
    description: Prioritised, actionable recommendations to address all findings
---

## Overview

The System Design Review Agent automates the first-pass review of a system design document. It evaluates the design against stated goals, checks for missing components (error handling, monitoring, security layers, scalability mechanisms), identifies design smells (tight coupling, single points of failure, missing APIs), and produces a scored improvement report.

Human architects use this report to focus their review effort on the highest-risk areas.

## Architecture

**Skills / Tools Used:**

| Tool / Skill | Purpose |
|---|---|
| `assess_completeness` | Score the design against a completeness checklist (0–100) |
| `identify_design_smells` | Detect anti-patterns, coupling issues, and single points of failure |
| `check_goals_alignment` | Verify each design goal has an explicit mechanism in the document |
| `generate_suggestions` | Produce prioritised, actionable improvement recommendations |

**Decision Flow:**

```
DESIGN_DOCUMENT + DESIGN_GOALS + CONSTRAINTS + EXISTING_COMPONENTS
         │
         ▼
Step 1: Assess completeness (assess_completeness)
         │
         ├─ Score < 60 ──▶ Flag as "Incomplete — major revision required"
         │
         └─ Score ≥ 60 ──▶ Continue
                   │
                   ▼
         Step 2: Identify design smells (identify_design_smells)
                   │
                   ▼
         Step 3: Check goals alignment (check_goals_alignment)
                   │
                   ▼
         Step 4: Generate improvement suggestions (generate_suggestions)
                   │
                   ▼
         REVIEW_REPORT + COMPLETENESS_SCORE + DESIGN_SMELLS + IMPROVEMENT_SUGGESTIONS
```

## System Prompt / Agent Instructions

```
[System Prompt]

You are an autonomous system design review agent for a software architecture team.

Your goals:
1. Evaluate the design document for completeness against a standard architecture checklist.
2. Identify design smells: tight coupling, single points of failure, missing error handling,
   absent monitoring/observability, undocumented API contracts, and scalability gaps.
3. Check that every stated design goal has an explicit mechanism in the document.
4. Produce a completeness score (0–100) and an actionable improvement report.

Tools available to you:
- assess_completeness: call with (design_document, goals) → returns score (int) + missing_items (list)
- identify_design_smells: call with (design_document, constraints) → returns smells (list)
- check_goals_alignment: call with (design_document, goals) → returns alignment_gaps (list)
- generate_suggestions: call with (missing_items, smells, alignment_gaps) → returns suggestions (list)

Rules:
- Never invent requirements not stated in DESIGN_GOALS or DESIGN_DOCUMENT.
- Score completeness against: components identified, data flows documented, API contracts defined,
  error handling described, security mechanisms present, monitoring/observability covered,
  scalability approach stated, deployment topology described.
- Classify findings as: Critical (blocks production) | Major (significant risk) | Minor (improvement).
- Keep improvement suggestions actionable — each must reference a specific section or gap.
- If EXISTING_COMPONENTS are provided, flag integration points that are unaddressed.

Output format:

## Completeness Score
{COMPLETENESS_SCORE} / 100 — {one-sentence rationale}

## Review Report
{REVIEW_REPORT}

## Design Smells
{DESIGN_SMELLS}

## Improvement Suggestions
{IMPROVEMENT_SUGGESTIONS}
```

## Configuration

```yaml
agent_id: sysdesign-system-design-review-agent
model: gpt-4o
temperature: 0.1         # low for consistent, deterministic review output
max_iterations: 6        # assess → smells → alignment → suggestions → synthesise → format
tools:
  - assess_completeness
  - identify_design_smells
  - check_goals_alignment
  - generate_suggestions
```

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `{{DESIGN_DOCUMENT}}` | string | Yes | System design document (Markdown or plain text) |
| `{{DESIGN_GOALS}}` | string | Yes | Comma-separated design goals |
| `{{CONSTRAINTS}}` | string | No | Budget, stack, or compliance constraints |
| `{{EXISTING_COMPONENTS}}` | string | No | Components this design must integrate with |

## Outputs

| Name | Type | Description |
|---|---|---|
| `REVIEW_REPORT` | string | Findings by severity (Critical / Major / Minor) |
| `COMPLETENESS_SCORE` | string | Numeric score 0–100 with rationale |
| `DESIGN_SMELLS` | string | Identified anti-patterns and their impact |
| `IMPROVEMENT_SUGGESTIONS` | string | Prioritised, actionable recommendations |

## Security Considerations

- The agent reads design documents but never writes to the repository or modifies designs.
- Design documents may contain sensitive architecture details — route through `in-house-llm` for confidential system designs.
- The agent must not recommend removing existing security mechanisms even if they appear to add complexity.
- API contract details (endpoints, auth methods) in the design are treated as internal and are not shared externally.

## Usage

### LLM Chat (current)

Open the design document in your IDE, then:

```
@workspace #file:<design-doc.md>
You are a system design review agent. Review this design document.
Goals: {{DESIGN_GOALS}}
Constraints: {{CONSTRAINTS}}
```

### API Endpoint (future portal)

```http
POST {{API_BASE_URL}}/packages/system-designers-design-pack/invoke
Content-Type: application/json

{
  "agent_id": "sysdesign-system-design-review-agent",
  "inputs": {
    "design_document": "## Notification Service Design\n...",
    "design_goals": "high availability, low latency, at-least-once delivery",
    "constraints": "AWS infrastructure, Node.js, GDPR compliance",
    "existing_components": "User Service, Order Service, API Gateway"
  }
}
```

### MCP Tool (future)

```json
{
  "tool": "sysdesign-system-design-review-agent",
  "parameters": {
    "design_document": "...",
    "design_goals": "high availability, low latency",
    "constraints": "AWS, Node.js",
    "existing_components": "User Service, API Gateway"
  }
}
```

## Examples

### Example 1 — Notification service design review

**Input:**
- `DESIGN_DOCUMENT`: *(Draft design for a push notification service covering fan-out queues and delivery workers)*
- `DESIGN_GOALS`: `high availability, at-least-once delivery, sub-500ms p95 latency`
- `CONSTRAINTS`: `AWS infrastructure, SQS + Lambda, GDPR`
- `EXISTING_COMPONENTS`: `User Service, API Gateway`

**Output:**

```
## Completeness Score
62 / 100 — Core components are identified but error handling, observability,
and dead-letter queue strategy are absent.

## Review Report

### Critical (must address before production)
- Dead-letter queue (DLQ) handling is not described. Failed deliveries will be silently
  dropped — violating the "at-least-once delivery" goal.
  Reference: Delivery Workers section.

### Major (significant risk)
- No monitoring or alerting strategy is defined. Queue depth and delivery failure rates
  are not tracked, making SLA compliance unverifiable.
- GDPR data retention for notification payloads is not addressed. Personal data in
  notification bodies must have a defined retention and deletion policy.

### Minor (improvement)
- API Gateway integration point with the User Service is mentioned but the contract
  (endpoint, auth, payload schema) is not documented.

## Design Smells

- **Single point of failure:** The fan-out Lambda has no retry configuration. A transient
  AWS outage would drop all in-flight notifications.
- **Missing circuit breaker:** No backpressure or throttling mechanism is described for
  downstream notification providers (FCM, APNS), risking cascade failure.

## Improvement Suggestions

1. [Critical] Add a DLQ configuration to the SQS queue with a max-receive-count of 3.
   Document the DLQ consumer and alerting threshold.
2. [Major] Add an Observability section covering: CloudWatch alarms on DLQ depth,
   Lambda error rate, and p95 delivery latency.
3. [Major] Add a Data Retention section: notification payloads containing PII must be
   purged after 30 days per GDPR Article 17.
4. [Minor] Document the User Service API contract using `sysdesign-api-contract-prompt`.
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | 2026-04-20. Completeness scoring is consistent. Occasionally flags minor items as Major. |
| claude-3-5-sonnet | ✅ | 2026-04-20. More thorough design smell detection, especially around coupling and SPoFs. |

## Changelog

### 1.0.0 — 2026-04-20
- Initial version.
