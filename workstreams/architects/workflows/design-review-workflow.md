---
id: arch-design-review-workflow
title: Architecture Design Review Workflow
type: workflow
version: "1.0.0"
status: active
workstream:
  - architects
author: ai-skills-maintainer
created: 2026-04-20
updated: 2026-04-20
tags:
  - architecture
  - design-review
  - workflow
  - quality-attributes
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
  - copilot-gpt-4o
description: |
  A four-step workflow that takes an architecture design document through contextualisation,
  quality-attribute evaluation, risk and anti-pattern identification, and recommendations
  synthesis — producing a consolidated design review report that architects and tech leads
  can use directly in Architecture Review Board meetings.
security_classification: internal
delivery_modes:
  - copilot-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
dependencies:
  - arch-trade-off-analysis
  - arch-adr-generator
inputs:
  - name: DESIGN_DOCUMENT
    type: string
    description: The architecture design document or description to review
    required: true
  - name: QUALITY_ATTRIBUTES
    type: string
    description: Comma-separated quality attributes to evaluate against (e.g., scalability, security, resilience, cost)
    required: true
  - name: CONSTRAINTS
    type: string
    description: Technical or business constraints the design must satisfy
    required: false
outputs:
  - name: DESIGN_REVIEW_REPORT
    type: string
    description: Consolidated design review report containing design summary, quality attribute evaluation, risk register, and recommendations
---

## Overview

This workflow orchestrates a structured first-pass review of any architecture design document. It is designed to be run before an Architecture Review Board (ARB) meeting so that human reviewers arrive with a clear picture of the design's strengths, weaknesses, and risks — rather than spending ARB time re-reading the document.

Each step produces a well-defined output that feeds the next. The prompts are self-contained and copy-pasteable into any LLM chat interface.

**Audience:** Solution architects, principal engineers, tech leads, ARB members.

## Steps

### Step 1 — Summarise and Contextualise the Design

**Asset used:** *(inline prompt)*  
**Input:** `{{DESIGN_DOCUMENT}}`, `{{QUALITY_ATTRIBUTES}}`, `{{CONSTRAINTS}}`  
**Output:** `DESIGN_SUMMARY` — a structured overview of the design that reviewers can scan in under two minutes.

```
[Prompt — paste into any LLM]

You are a principal architect preparing a design for review.

Design document: {{DESIGN_DOCUMENT}}
Quality attributes under review: {{QUALITY_ATTRIBUTES}}
Constraints: {{CONSTRAINTS}}

Produce a DESIGN_SUMMARY with the following sections:

### Intent
One paragraph: what problem does this design solve and what is the proposed approach?

### Key Components
Bulleted list: name each major component or service and its responsibility in one sentence.

### Key Design Decisions
Bulleted list: identify 3–5 significant design decisions visible in the document
(e.g., technology choices, data flow patterns, integration strategies).

### Open Questions
List any aspects of the design that are unclear, incomplete, or not addressed by the document.
If none, state "None identified."
```

---

### Step 2 — Evaluate Against Quality Attributes

**Asset used:** `arch-trade-off-analysis`  
**Input:** `DESIGN_SUMMARY` (Step 1), `{{DESIGN_DOCUMENT}}`, `{{QUALITY_ATTRIBUTES}}`, `{{CONSTRAINTS}}`  
**Output:** `ATTRIBUTE_EVALUATION` — RAG-rated table with justification per quality attribute.

```
[Prompt — paste into any LLM after Step 1]

You are a principal architect evaluating a design against quality attributes.

Design document: {{DESIGN_DOCUMENT}}
Constraints: {{CONSTRAINTS}}
Design summary (from Step 1): {{DESIGN_SUMMARY}}

For each quality attribute listed below, provide:
1. A RAG rating: 🟢 Green (fully addressed), 🟠 Amber (partially addressed or risk present), 🔴 Red (not addressed or anti-pattern present).
2. A 2–3 sentence justification citing specific aspects of the design.
3. Any trade-offs the design makes with respect to this attribute.

Quality attributes to evaluate: {{QUALITY_ATTRIBUTES}}

Format as a Markdown table followed by a brief narrative for any Amber or Red rating.

| Attribute | Rating | Justification | Trade-offs |
|---|---|---|---|
```

---

### Step 3 — Identify Risks and Anti-Patterns

**Asset used:** *(inline prompt)*  
**Input:** `DESIGN_SUMMARY` (Step 1), `ATTRIBUTE_EVALUATION` (Step 2), `{{DESIGN_DOCUMENT}}`, `{{CONSTRAINTS}}`  
**Output:** `RISK_REGISTER` — tabular risk register with likelihood, impact, and suggested mitigations.

```
[Prompt — paste into any LLM after Step 2]

You are a principal architect identifying risks in a proposed design.

Design document: {{DESIGN_DOCUMENT}}
Constraints: {{CONSTRAINTS}}
Design summary: {{DESIGN_SUMMARY}}
Quality attribute evaluation: {{ATTRIBUTE_EVALUATION}}

Identify all architectural risks and anti-patterns present in the design. For each risk:
- Assign a Risk ID (R-001, R-002, …).
- Write a one-sentence description of the risk.
- Rate Likelihood as High / Medium / Low.
- Rate Impact as Critical / High / Medium / Low.
- Propose a specific mitigation action.

Also flag any of these common architectural anti-patterns if present:
- Big ball of mud / lack of clear boundaries
- Distributed monolith (microservices with tight synchronous coupling)
- Missing circuit breakers on synchronous dependency chains
- Single points of failure with no redundancy
- Overly chatty inter-service communication
- Missing observability (no logging, metrics, or tracing strategy)

Format as a Markdown table:

| Risk ID | Description | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
```

---

### Step 4 — Produce Recommendations and Next Steps

**Asset used:** *(inline prompt)*  
**Input:** `ATTRIBUTE_EVALUATION` (Step 2), `RISK_REGISTER` (Step 3), `DESIGN_SUMMARY` (Step 1)  
**Output:** `RECOMMENDATIONS` and final `DESIGN_REVIEW_REPORT`.

```
[Prompt — paste into any LLM after Step 3]

You are a principal architect producing final recommendations for a design review.

Design summary: {{DESIGN_SUMMARY}}
Quality attribute evaluation: {{ATTRIBUTE_EVALUATION}}
Risk register: {{RISK_REGISTER}}

Produce two outputs:

### Recommendations
A prioritised list of actionable recommendations. For each:
- Priority: Critical / High / Medium / Low
- What to change (specific, not generic)
- Why (link back to the risk or quality attribute finding)
- Suggested next step (e.g., "Spike: evaluate AWS SQS vs. Kafka for async messaging")

### Overall Assessment
One paragraph: overall fitness of the design for its purpose, highlighting the two most
important changes required before the design is approved.

Then produce a final DESIGN_REVIEW_REPORT by combining the outputs of Steps 1–4 under these headings:
## Design Summary
## Quality Attribute Evaluation
## Risk Register
## Recommendations
## Overall Assessment
```

---

## Flow Diagram

```
DESIGN_DOCUMENT + QUALITY_ATTRIBUTES [+ CONSTRAINTS]
         │
         ▼
[Step 1: Summarise and Contextualise] ──▶ DESIGN_SUMMARY
         │
         ▼
[Step 2: Evaluate Quality Attributes] ──▶ ATTRIBUTE_EVALUATION
         │
         ▼
[Step 3: Identify Risks and Anti-Patterns] ──▶ RISK_REGISTER
         │
         ▼
[Step 4: Recommendations and Next Steps] ──▶ RECOMMENDATIONS
         │
         ▼
         DESIGN_REVIEW_REPORT
```

## Usage

### Manual (Copilot Chat — current)

1. Prepare your design document (or a concise description of the design).
2. Copy Step 1 prompt, substitute placeholders, and paste into GitHub Copilot Chat or your preferred LLM.
3. Save each step's output and carry it forward as input for the next step.
4. Use the final `DESIGN_REVIEW_REPORT` as the basis for your ARB meeting discussion.
5. For each Critical or High recommendation, create an ADR using `arch-adr-generator` to record the resulting decision.

### Automated (GitHub Actions — current)

See `workstreams/architects/automations/adr-sync-automation.md` for a GitHub Actions automation that runs an LLM-assisted review when architecture documents are modified in a PR.

### API Endpoint (future portal)

```http
POST {{API_BASE_URL}}/packages/architects-core-pack/invoke
Content-Type: application/json

{
  "workflow_id": "arch-design-review-workflow",
  "inputs": {
    "design_document": "## Proposed Design: Event-Driven Order Platform\n...",
    "quality_attributes": "scalability, security, resilience, cost, maintainability",
    "constraints": "Must run on AWS. Must reuse Cognito. 6-month delivery window."
  }
}
```

## Examples

### Example 1 — Event-driven e-commerce platform design review

**Input:**
- `DESIGN_DOCUMENT`: *"We propose replacing the monolithic order service with an event-driven architecture. The Order Service publishes events to an SNS topic. Downstream services (Fulfilment, Notification, Inventory) subscribe via SQS queues. Each service has its own Aurora Serverless database. Authentication via Cognito at the API Gateway layer."*
- `QUALITY_ATTRIBUTES`: `scalability, security, resilience, cost, maintainability`
- `CONSTRAINTS`: `Must run on AWS. Must reuse existing Cognito auth service. Team of 8 engineers.`

**Step 1 Output (DESIGN_SUMMARY excerpt):**
```
### Intent
This design replaces a monolithic order service with an event-driven architecture using
AWS SNS/SQS to decouple the Order Service from downstream Fulfilment, Notification, and
Inventory services — improving scalability and resilience at the cost of increased
operational complexity.

### Key Design Decisions
- Async event-driven communication via SNS fan-out to SQS queues (reduces coupling)
- Aurora Serverless per service (independent scaling; higher cost than shared cluster)
- Cognito retained at API Gateway (avoids auth re-implementation risk)
```

**Step 2 Output (ATTRIBUTE_EVALUATION excerpt):**
```
| Attribute | Rating | Justification | Trade-offs |
|---|---|---|---|
| Scalability | 🟢 Green | SNS/SQS decouples producers from consumers; each service can scale independently. | Aurora Serverless cold-start latency may affect Inventory service under burst load. |
| Resilience | 🟢 Green | SQS provides durable message buffering; a downstream service outage will not impact order acceptance. | Dead-letter queue configuration and retry policies must be explicitly defined. |
| Cost | 🟠 Amber | Four Aurora Serverless instances are cost-effective at low traffic but may become expensive at sustained high throughput. | Consider consolidating Notification and Inventory onto a shared cluster. |
```

**Step 4 Output (Overall Assessment):**
```
The design is well-suited to the stated goals of scalability and resilience. The two most
important changes before approval are: (1) explicitly define SQS dead-letter queue policies
and maximum receive counts for all queues to prevent silent message loss, and (2) evaluate
whether four Aurora Serverless instances are justified or whether two of the lower-traffic
services can share a cluster to reduce cost and operational overhead.
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | 2026-04-20. All four steps work in a single Copilot Chat session. Quality attribute ratings are well-calibrated. |
| claude-3-5-sonnet | ✅ | 2026-04-20. Step 3 risk identification is more thorough. Step 4 recommendations are better prioritised. |

## Changelog

### 1.0.0 — 2026-04-20
- Initial version.
