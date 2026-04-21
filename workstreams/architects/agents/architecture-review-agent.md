---
id: arch-architecture-review-agent
title: Architecture Review Agent
type: agent
version: "1.0.0"
status: active
workstream:
  - architects
author: ai-skills-maintainer
created: 2026-04-20
updated: 2026-04-20
tags:
  - architecture
  - review
  - quality-attributes
  - design
  - risk-assessment
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
  - copilot-gpt-4o
description: |
  An autonomous agent that reviews a proposed architecture design against a defined set
  of quality attributes, identifies risks and gaps, evaluates trade-offs, and produces
  a structured review report with a risk register and actionable recommendations —
  reducing the time architects spend on first-pass design reviews.
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
    description: The architecture design document or description to be reviewed
    required: true
  - name: QUALITY_ATTRIBUTES
    type: string
    description: Comma-separated list of quality attributes to evaluate against (e.g., scalability, security, resilience, cost)
    required: true
  - name: CONSTRAINTS
    type: string
    description: Technical or business constraints the design must satisfy (e.g., must run on AWS, must use existing auth service)
    required: false
  - name: CONTEXT
    type: string
    description: Additional context such as team size, existing platform, or non-functional requirements
    required: false
outputs:
  - name: REVIEW_REPORT
    type: string
    description: Structured architecture review report with evaluation per quality attribute
  - name: RISK_REGISTER
    type: string
    description: Tabular risk register listing identified risks, likelihood, impact, and mitigations
  - name: RECOMMENDATIONS
    type: string
    description: Prioritised list of actionable recommendations for the design team
---

## Overview

The Architecture Review Agent replaces the first-pass manual review for proposed architecture designs. It systematically evaluates the design against the requested quality attributes, surfaces risks and anti-patterns, and produces a structured review report that architects and tech leads can act on without having to read through the full design document themselves.

Use it before formal Architecture Review Board (ARB) meetings to surface obvious issues early and focus human review time on novel design decisions and strategic trade-offs.

## Architecture

**Skills / Tools Used:**

| Tool / Skill | Purpose |
|---|---|
| `trade_off_analysis` | Evaluate design decisions against quality attribute trade-offs |
| `identify_risks` | Identify architectural risks, anti-patterns, and constraint violations |

**Decision Flow:**

```
DESIGN_DOCUMENT + QUALITY_ATTRIBUTES [+ CONSTRAINTS] [+ CONTEXT]
         │
         ▼
Step 1: Summarise and contextualise design
         │
         ▼
Step 2: Evaluate against each quality attribute (trade_off_analysis)
         │
         ├─ Quality attribute met ──▶ Green finding
         │
         ├─ Partial / risk present ──▶ Amber finding + risk entry
         │
         └─ Not met / anti-pattern ──▶ Red finding + risk entry + recommendation
                  │
                  ▼
         Step 3: Identify risks and anti-patterns (identify_risks)
                  │
                  ▼
         Step 4: Compile RISK_REGISTER + RECOMMENDATIONS
                  │
                  ▼
         REVIEW_REPORT + RISK_REGISTER + RECOMMENDATIONS
```

## System Prompt / Agent Instructions

```
[System Prompt]

You are an autonomous architecture review agent for a software engineering organisation.

Your goals:
1. Summarise the proposed design so reviewers quickly understand the intent and scope.
2. Evaluate the design against each requested quality attribute, providing a RAG
   (Red / Amber / Green) status and brief justification for each.
3. Identify architectural risks, anti-patterns, and violations of stated constraints.
4. Produce a structured review report, a tabular risk register, and a prioritised
   list of recommendations.

Tools available to you:
- trade_off_analysis: call with (design_section, quality_attribute, constraints)
  → returns ATTRIBUTE_RATING (Green/Amber/Red) and TRADE_OFF_NOTES
- identify_risks: call with (design_document, quality_attributes, constraints)
  → returns RISK_LIST with risk_id, description, likelihood, impact, mitigation

Rules:
- Evaluate every quality attribute listed in QUALITY_ATTRIBUTES — do not skip any.
- A Red rating on any security or resilience attribute must produce at least one
  Critical recommendation.
- Do not invent quality attributes not requested unless they represent a critical
  gap (e.g., missing security consideration) — flag these as unsolicited observations.
- Recommendations must be actionable: state what to change, not just that something
  is wrong.
- If the design document is incomplete or ambiguous, flag the gap explicitly rather
  than assuming intent.
- Keep the REVIEW_REPORT suitable for sharing with both technical and non-technical
  stakeholders.

Output format:

## Design Summary
{One paragraph summary of the proposed architecture}

## Quality Attribute Evaluation
{RAG table + justification per attribute}

## Risk Register
{Tabular risk register}

## Recommendations
{Prioritised recommendations list}
```

## Configuration

```yaml
agent_id: arch-architecture-review-agent
model: gpt-4o
temperature: 0.2         # slightly higher than review agents to allow creative risk identification
max_iterations: 8        # summarise → evaluate (N attributes) → identify_risks → compile → format
tools:
  - trade_off_analysis
  - identify_risks
```

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `{{DESIGN_DOCUMENT}}` | string | Yes | Architecture design document or description |
| `{{QUALITY_ATTRIBUTES}}` | string | Yes | Comma-separated quality attributes to evaluate |
| `{{CONSTRAINTS}}` | string | No | Technical or business constraints |
| `{{CONTEXT}}` | string | No | Team size, existing platform, NFRs |

## Outputs

| Name | Type | Description |
|---|---|---|
| `REVIEW_REPORT` | string | Structured review with RAG evaluation per quality attribute |
| `RISK_REGISTER` | string | Tabular risk register with likelihood, impact, and mitigations |
| `RECOMMENDATIONS` | string | Prioritised actionable recommendations |

## Security Considerations

- The agent reads design documents but never modifies architecture artefacts or commits to repositories.
- Design documents may contain sensitive system topology information — route through `in-house-llm` for designs that involve security-sensitive infrastructure.
- The agent must not expose internal service names, IP ranges, or credentials encountered in the design document in any output that is shared externally.
- Limit tool permissions to read-only access to the architecture knowledge base.

## Usage

### GitHub Copilot Chat (current)

```
You are an architecture review agent.
Design document: {{DESIGN_DOCUMENT}}
Quality attributes to evaluate: {{QUALITY_ATTRIBUTES}}
Constraints: {{CONSTRAINTS}}

Evaluate the design, identify risks, and produce a structured review report with
a risk register and prioritised recommendations.
```

### API Endpoint (future portal)

```http
POST {{API_BASE_URL}}/packages/architects-core-pack/invoke
Content-Type: application/json

{
  "agent_id": "arch-architecture-review-agent",
  "inputs": {
    "design_document": "## E-Commerce Platform Redesign\nWe propose a microservices architecture...",
    "quality_attributes": "scalability, security, resilience, cost, maintainability",
    "constraints": "Must run on AWS. Must reuse the existing Cognito auth service.",
    "context": "Team of 12 engineers. Current monolith handles 10k orders/day."
  }
}
```

### MCP Tool (future)

```json
{
  "tool": "arch-architecture-review-agent",
  "parameters": {
    "design_document": "...",
    "quality_attributes": "scalability, security, resilience, cost",
    "constraints": "Must run on AWS.",
    "context": "Team of 12 engineers."
  }
}
```

## Examples

### Example 1 — E-commerce microservices design review

**Input:**
- `DESIGN_DOCUMENT`: *Excerpt: "We will decompose the current monolith into 12 microservices. Each service will have its own Postgres database. Services will communicate synchronously via REST. A single API gateway will handle all external traffic. No caching layer is planned initially."*
- `QUALITY_ATTRIBUTES`: `scalability, security, resilience, cost, maintainability`
- `CONSTRAINTS`: `Must run on AWS. Must reuse existing Cognito auth service. Go-live in 6 months.`
- `CONTEXT`: `Team of 12 engineers. Current monolith handles 10 000 orders/day. Peak load 5× average.`

**Output:**

```
## Design Summary
The proposed design decomposes a monolithic e-commerce platform into 12 microservices,
each owning its own Postgres database, communicating synchronously over REST through
a single API gateway. Authentication delegates to AWS Cognito. No caching layer is
included in the initial scope. The target platform is AWS and the team of 12 has a
6-month delivery window.

## Quality Attribute Evaluation

| Attribute | Rating | Notes |
|---|---|---|
| Scalability | 🟠 Amber | Synchronous REST between all 12 services creates tight coupling; a slow downstream service will cascade latency to callers. No caching exacerbates database load at peak. |
| Security | 🟢 Green | Cognito reuse is a sound decision. API gateway as single ingress point is a good security boundary. Recommend enabling WAF on the gateway. |
| Resilience | 🔴 Red | No circuit breakers, retries, or bulkheads described. Full synchronous dependency chains create a single point of failure risk across multiple services. |
| Cost | 🟠 Amber | 12 separate Postgres instances (even RDS micro) will be more expensive than a shared schema approach. Evaluate RDS cost vs. Aurora Serverless for lower-traffic services. |
| Maintainability | 🟢 Green | Independent deployability per service supports maintainability. Risk: 12 services for a 12-engineer team may stretch ownership thin. |

## Risk Register

| Risk ID | Description | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R-001 | Cascading failures due to synchronous REST chains with no circuit breakers | High | Critical | Implement circuit breakers (e.g., resilience4j) and async messaging (SQS/SNS) for non-time-critical calls |
| R-002 | Database cost overrun from 12 separate Postgres instances | Medium | Medium | Consolidate low-traffic services onto shared Aurora Serverless cluster; split only high-traffic services |
| R-003 | API gateway becomes single point of failure | Medium | Critical | Deploy gateway across multiple AZs; implement health checks and auto-scaling |
| R-004 | Team capacity insufficient to own 12 services in 6 months | High | High | Reduce initial scope to 5–6 core services; extract remaining incrementally post-launch |

## Recommendations

1. [Critical] Add circuit breakers and timeouts to all synchronous inter-service calls before go-live.
2. [Critical] Introduce async messaging (AWS SQS or SNS) for at least the order-fulfilment and notification flows.
3. [High] Deploy the API gateway in a multi-AZ configuration with auto-scaling enabled.
4. [Medium] Consolidate low-traffic microservices onto a shared Aurora Serverless cluster to control RDS costs.
5. [Medium] Reduce initial delivery scope to the 5 highest-value services; document the remaining 7 as Phase 2.
6. [Low] Enable AWS WAF on the API gateway to protect against OWASP Top 10 threats.
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | 2026-04-20. RAG ratings are accurate and well-justified. Occasionally misses nuanced trade-offs on cost — provide explicit cost constraints for best results. |
| claude-3-5-sonnet | ✅ | 2026-04-20. Risk identification is more thorough. Recommendations prose quality is higher than gpt-4o for this task. |

## Changelog

### 1.0.0 — 2026-04-20
- Initial version.
