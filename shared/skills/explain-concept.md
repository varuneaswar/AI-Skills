---
id: shared-explain-concept
title: Explain Technical Concept in Plain Language
type: skill
version: "1.0.0"
status: active
workstream:
  - shared
author: ai-skills-maintainer
created: 2024-01-01
updated: 2024-01-01
tags:
  - explanation
  - communication
  - knowledge-sharing
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
  - gemini-1.5-pro
  - all
description: |
  Translates any technical concept into clear, jargon-free language tailored to a specified
  audience level. Useful across all work streams for documentation, presentations, and onboarding.
security_classification: public
inputs:
  - name: CONCEPT
    type: string
    description: The technical concept, term, or topic to explain
    required: true
  - name: AUDIENCE
    type: string
    description: "Target audience level: executive, non-technical, junior-engineer, or expert"
    required: true
  - name: CONTEXT
    type: string
    description: Optional domain context (e.g., 'cloud infrastructure', 'software testing')
    required: false
outputs:
  - name: EXPLANATION
    type: string
    description: Plain-language explanation tailored to the specified audience
delivery_modes:
  - llm-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
---

## Overview

This skill converts dense technical content into audience-appropriate language. It is intentionally cross-cutting — any work stream can use it to explain architecture decisions to stakeholders, describe incidents to management, or onboard new team members.

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `{{CONCEPT}}` | string | Yes | The technical concept or term to explain |
| `{{AUDIENCE}}` | string | Yes | `executive`, `non-technical`, `junior-engineer`, `expert` |
| `{{CONTEXT}}` | string | No | Domain context to make the explanation more relevant |

## Outputs

| Name | Type | Description |
|---|---|---|
| `EXPLANATION` | string | Plain-language explanation tailored to the audience |

## Skill Definition

```
[System Prompt]

You are a technical writer who excels at explaining complex concepts clearly and concisely.
Your explanations are always accurate, never condescending, and precisely matched to the
audience's knowledge level.

Audience levels:
- executive: Focus on business impact, avoid all technical jargon. Max 3 sentences.
- non-technical: Use everyday analogies. Avoid acronyms unless explained. Max 1 short paragraph.
- junior-engineer: Assume basic programming knowledge, no domain expertise. Use analogies and simple examples.
- expert: Peer-level explanation. Use correct terminology, reference standards where relevant.

Concept to explain: {{CONCEPT}}
Audience: {{AUDIENCE}}
Domain context: {{CONTEXT}}

Provide only the explanation. Do not add preamble like "Sure, here is..." or "Great question!".
```

## Usage

### LLM Chat (current)

```
Explain "{{CONCEPT}}" to a {{AUDIENCE}} audience in the context of {{CONTEXT}}.
```

### Standalone (any LLM — current)

1. Copy the skill definition above.
2. Set it as the system prompt.
3. Send `{{CONCEPT}}`, `{{AUDIENCE}}`, and optionally `{{CONTEXT}}` as the user message.

### API Endpoint (future portal)

```http
POST {{API_BASE_URL}}/packages/shared-utilities-pack/invoke
Content-Type: application/json

{
  "skill_id": "shared-explain-concept",
  "inputs": {
    "concept": "Kubernetes pod scheduling",
    "audience": "executive",
    "context": "cloud infrastructure"
  }
}
```

### MCP Tool (future)

```json
{
  "tool": "shared-explain-concept",
  "parameters": {
    "concept": "Kubernetes pod scheduling",
    "audience": "executive",
    "context": "cloud infrastructure"
  }
}
```

## Examples

### Example 1 — Executive audience

**Input:** `CONCEPT` = `Kubernetes pod scheduling`, `AUDIENCE` = `executive`, `CONTEXT` = `cloud cost optimisation`

**Output:**
```
Kubernetes automatically decides which server runs each piece of your application,
optimising for cost and reliability. This reduces idle server spend and ensures your
app keeps running even if individual servers fail.
```

### Example 2 — Junior-engineer audience

**Input:** `CONCEPT` = `eventual consistency`, `AUDIENCE` = `junior-engineer`, `CONTEXT` = `distributed databases`

**Output:**
```
Imagine you update your profile picture on a social network. For a few seconds, some
users might still see your old picture while others see the new one — the system is
"eventually" consistent once all servers have synced. In distributed databases, this
trade-off lets the system stay fast and available even when servers can't instantly
agree, at the cost of brief windows where different nodes show different data.
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | 2024-01-01. Excellent audience calibration. |
| claude-3-5-sonnet | ✅ | 2024-01-01. Slightly longer but well-structured. |
| all | ✅ | Works well across all models due to simple, clear instructions. |

## Changelog

### 1.0.0 — 2024-01-01
- Initial version.
