---
id: shared-summarize-document
title: Summarize Any Document or Text
type: skill
version: "1.0.0"
status: active
workstream:
  - shared
author: ai-skills-maintainer
created: 2024-01-01
updated: 2024-01-01
tags:
  - summarization
  - documentation
  - communication
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
  - gemini-1.5-pro
  - all
description: |
  Produces a structured summary of any document or long text block with configurable detail
  level and output format. Used across all work streams to condense reports, runbooks, PRs,
  design docs, and incident reports.
security_classification: internal
inputs:
  - name: DOCUMENT_TEXT
    type: string
    description: The full text to be summarized
    required: true
  - name: DETAIL_LEVEL
    type: string
    description: "one-liner, short (3-5 sentences), detailed (full structured summary)"
    required: true
  - name: OUTPUT_FORMAT
    type: string
    description: "prose, bullet-list, or structured (with headings)"
    required: false
  - name: FOCUS_AREA
    type: string
    description: Optional aspect to emphasize (e.g., 'risks', 'action items', 'decisions')
    required: false
outputs:
  - name: SUMMARY
    type: string
    description: The generated summary in the requested format
delivery_modes:
  - llm-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
---

## Overview

A versatile summarization skill that works for any text length and format. All work streams can use this to create executive summaries, meeting notes distillations, document TLDRs, or focused extractions of action items and risks.

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `{{DOCUMENT_TEXT}}` | string | Yes | The text to summarize |
| `{{DETAIL_LEVEL}}` | string | Yes | `one-liner`, `short`, or `detailed` |
| `{{OUTPUT_FORMAT}}` | string | No | `prose` (default), `bullet-list`, `structured` |
| `{{FOCUS_AREA}}` | string | No | Specific aspect to highlight, e.g., `risks`, `action items` |

## Outputs

| Name | Type | Description |
|---|---|---|
| `SUMMARY` | string | The generated summary |

## Skill Definition

```
[System Prompt]

You are an expert technical summarizer. You produce accurate, faithful summaries that
preserve all important information while removing redundancy and filler.

Detail levels:
- one-liner: Single sentence capturing the essence. Max 25 words.
- short: 3 to 5 sentences covering the main points.
- detailed: Full structured summary with clear sections and all key points retained.

Output formats:
- prose: Flowing paragraphs.
- bullet-list: Concise bullet points, no full sentences required.
- structured: Markdown headings (## Key Points, ## Decisions, ## Action Items, ## Risks) where applicable.

Rules:
- Never invent information not present in the source.
- If asked to focus on a specific area ({{FOCUS_AREA}}), lead with that.
- Do not include preamble or meta-commentary about the summary itself.

Document to summarize:
{{DOCUMENT_TEXT}}

Detail level: {{DETAIL_LEVEL}}
Output format: {{OUTPUT_FORMAT}}
Focus area: {{FOCUS_AREA}}
```

## Usage

### LLM Chat (current)

```
Summarize the following text at {{DETAIL_LEVEL}} detail as a {{OUTPUT_FORMAT}},
focusing on {{FOCUS_AREA}}:

{{DOCUMENT_TEXT}}
```

### Standalone (any LLM — current)

1. Copy the skill definition above.
2. Replace placeholders.
3. Paste into your LLM interface.

### API Endpoint (future portal)

```http
POST {{API_BASE_URL}}/packages/shared-utilities-pack/invoke
Content-Type: application/json

{
  "skill_id": "shared-summarize-document",
  "inputs": {
    "document_text": "...",
    "detail_level": "short",
    "output_format": "bullet-list",
    "focus_area": "action items"
  }
}
```

## Examples

### Example 1 — Incident summary

**Input:**
- `DOCUMENT_TEXT`: *(long incident report)*
- `DETAIL_LEVEL`: `short`
- `OUTPUT_FORMAT`: `bullet-list`
- `FOCUS_AREA`: `action items`

**Output:**
```
- Roll back checkout-service to v2.4.0 before next deployment window.
- Add connection pool monitoring to the SRE dashboard.
- Update runbook with steps for DB pool exhaustion scenarios.
- Schedule post-incident review for 2024-01-08.
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | 2024-01-01. Excellent. Respects all format constraints. |
| claude-3-5-sonnet | ✅ | 2024-01-01. Best for detailed structured summaries. |

## Changelog

### 1.0.0 — 2024-01-01
- Initial version.
