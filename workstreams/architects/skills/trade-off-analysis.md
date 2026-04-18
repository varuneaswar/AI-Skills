---
id: arch-trade-off-analysis
title: Architecture Trade-Off Analysis
type: skill
version: "1.0.0"
status: active
workstream:
  - architects
  - system-designers
author: ai-skills-maintainer
created: 2024-01-01
updated: 2024-01-01
tags:
  - trade-off
  - architecture
  - decision-support
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
  - gemini-1.5-pro
description: |
  Produces a structured trade-off analysis comparing two or more architecture or technology
  options across configurable quality attributes, helping teams reach faster, better-documented
  decisions.
security_classification: internal
inputs:
  - name: DECISION_CONTEXT
    type: string
    description: What decision needs to be made and why
    required: true
  - name: OPTIONS
    type: string
    description: Comma-separated list of options to compare
    required: true
  - name: QUALITY_ATTRIBUTES
    type: string
    description: "Comma-separated quality attributes to evaluate against (e.g., scalability, cost, team-expertise, time-to-market)"
    required: true
  - name: CONSTRAINTS
    type: string
    description: Hard constraints that must be satisfied regardless of option chosen
    required: false
outputs:
  - name: TRADE_OFF_MATRIX
    type: string
    description: Structured trade-off analysis with recommendation
delivery_modes:
  - copilot-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
---

## Overview

Accelerates architecture decision-making by producing a consistent, structured comparison. Output can be pasted directly into an ADR (see `arch-adr-generator` prompt) or a design document. Works best when the architect reviews and calibrates the scoring.

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `{{DECISION_CONTEXT}}` | string | Yes | What is being decided and why |
| `{{OPTIONS}}` | string | Yes | Options to compare, comma-separated |
| `{{QUALITY_ATTRIBUTES}}` | string | Yes | Attributes to evaluate, comma-separated |
| `{{CONSTRAINTS}}` | string | No | Hard constraints (e.g., "must run on-premises", "no new vendor") |

## Outputs

| Name | Type | Description |
|---|---|---|
| `TRADE_OFF_MATRIX` | string | Structured comparison with scores and recommendation |

## Skill Definition

```
[System Prompt]

You are a principal architect helping a team evaluate technical options systematically.

Decision context: {{DECISION_CONTEXT}}
Options to compare: {{OPTIONS}}
Quality attributes: {{QUALITY_ATTRIBUTES}}
Hard constraints: {{CONSTRAINTS}}

Produce a structured trade-off analysis using this format:

## Decision Context
Restate the decision in one paragraph, including key constraints.

## Evaluation Criteria
List each quality attribute with a brief definition and its relative weight (High / Medium / Low).

## Comparison Matrix

| Quality Attribute | Weight | [Option A] | [Option B] | [Option C] |
|---|---|---|---|---|
| attribute 1 | High | ⭐⭐⭐ (reason) | ⭐⭐ (reason) | ⭐ (reason) |
...

(Score: ⭐⭐⭐ = Strong, ⭐⭐ = Adequate, ⭐ = Weak, ✗ = Disqualified by constraint)

## Constraint Check
Explicitly state whether each option satisfies every hard constraint.

## Recommendation
State the recommended option and explain the reasoning, acknowledging the main trade-offs
that were accepted.

## Risks of the Recommended Option
Bulleted list of the top 3 risks and how they could be mitigated.
```

## Usage

### GitHub Copilot Chat (current)

```
Compare {{OPTIONS}} for {{DECISION_CONTEXT}}.
Evaluate against: {{QUALITY_ATTRIBUTES}}.
Constraints: {{CONSTRAINTS}}.
```

### API Endpoint (future portal)

```http
POST {{API_BASE_URL}}/packages/architects-design-pack/invoke
Content-Type: application/json

{
  "skill_id": "arch-trade-off-analysis",
  "inputs": {
    "decision_context": "Choose a message broker for our event-driven microservices",
    "options": "Apache Kafka, RabbitMQ, AWS SQS/SNS",
    "quality_attributes": "throughput, operational-complexity, cost, team-expertise",
    "constraints": "must support event replay"
  }
}
```

## Examples

### Example 1

**Input:**
- `DECISION_CONTEXT`: `Choose a caching layer for our read-heavy API (100k req/s peak)`
- `OPTIONS`: `Redis, Memcached, In-process cache`
- `QUALITY_ATTRIBUTES`: `throughput, data-structure-richness, operational-complexity, cost`
- `CONSTRAINTS`: `must support TTL-based expiry`

**Output (excerpt):**
```markdown
## Comparison Matrix

| Quality Attribute | Weight | Redis | Memcached | In-process |
|---|---|---|---|---|
| Throughput | High | ⭐⭐⭐ (sub-ms, clustering) | ⭐⭐⭐ (sub-ms, simpler model) | ⭐⭐ (no network overhead but limited by single-node memory) |
| Data-structure richness | Medium | ⭐⭐⭐ (strings, hashes, lists, sorted sets) | ⭐ (strings/bytes only) | ⭐⭐ (language-native structures) |
| Operational complexity | Medium | ⭐⭐ (cluster mode adds complexity) | ⭐⭐⭐ (simpler to operate) | ⭐⭐⭐ (no separate infrastructure) |
| Cost | Low | ⭐⭐ (managed service costs) | ⭐⭐⭐ (lower managed cost) | ⭐⭐⭐ (no additional infra) |

## Recommendation
**Redis** — best overall fit given the need for TTL expiry, future flexibility with richer
data structures, and acceptable operational overhead at this scale.
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | 2024-01-01. Good matrix structure. Scores need architect calibration. |
| claude-3-5-sonnet | ✅ | 2024-01-01. More verbose explanations per cell — useful for documentation. |

## Changelog

### 1.0.0 — 2024-01-01
- Initial version.
