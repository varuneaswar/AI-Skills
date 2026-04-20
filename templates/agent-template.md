---
id: replace-with-unique-kebab-case-id
title: Replace With Human-Readable Title
type: agent
version: "1.0.0"
status: draft
workstream:
  - replace-with-workstream
author: your-github-username
created: YYYY-MM-DD
updated: YYYY-MM-DD
tags:
  - tag1
  - tag2
llm_compatibility:
  - gpt-4o
description: |
  One or two sentences describing what this agent does and when to deploy it.
security_classification: internal
delivery_modes:
  - copilot-chat          # available now — copy/paste into any LLM interface
  # - api-endpoint        # uncomment when portal REST API is available
  # - mcp-tool            # uncomment when MCP server is deployed
  # - copilot-studio      # uncomment when Copilot Studio connector is configured
  # - in-house-llm        # uncomment when internal LLM API is available
dependencies: []
inputs:
  - name: INPUT_ONE
    type: string
    description: What this input represents
    required: true
outputs:
  - name: OUTPUT_ONE
    type: string
    description: What this output represents
---

## Overview

<!-- Describe the agent's purpose, autonomy level, and typical trigger conditions.
     What decisions can it make independently? What requires human approval? -->

## Architecture

<!-- List the skills, tools, or external services this agent uses.
     Describe how they are orchestrated. -->

**Skills / Tools Used:**

| Tool / Skill | Purpose |
|---|---|
| `skill-id-one` | Describe what it does in this agent |
| `external-tool` | Describe what it does |

**Decision Flow:**

```
Input received
  │
  ▼
Step 1: [action]
  │
  ├─ [condition A] ──▶ Step 2a
  │
  └─ [condition B] ──▶ Step 2b
                          │
                          ▼
                       Output produced
```

## System Prompt / Agent Instructions

```
[System Prompt]

You are an autonomous agent responsible for {{AGENT_RESPONSIBILITY}}.

Your goals:
1. Goal one
2. Goal two

Tools available to you:
- tool_one: description
- tool_two: description

Rules:
- Never take destructive actions without explicit user confirmation.
- Always explain your reasoning before acting.
- If uncertain, ask for clarification rather than guessing.
```

## Configuration

<!-- Any configuration required before deploying this agent -->

```yaml
agent_id: replace-with-unique-kebab-case-id
model: gpt-4o          # recommended model
temperature: 0.2       # lower for more deterministic behaviour
max_iterations: 10     # safety cap on autonomous steps
tools:
  - tool_one
  - tool_two
```

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `{{INPUT_ONE}}` | string | Yes | What this input represents |

## Outputs

| Name | Type | Description |
|---|---|---|
| `OUTPUT_ONE` | string | What this output represents |

## Security Considerations

<!-- Required for all agents.
     - What external systems does it access?
     - What is the blast radius of a mis-step?
     - What guardrails are in place? -->

## Examples

### Example 1

**Input:** `INPUT_ONE` = *your example value*

**Output:**
```
Paste a real, redacted output here.
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | Tested on YYYY-MM-DD |

## Changelog

### 1.0.0 — YYYY-MM-DD
- Initial version.
