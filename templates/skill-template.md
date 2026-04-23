---
id: replace-with-unique-kebab-case-id
title: Replace With Human-Readable Title
type: skill
version: "1.0.0"
status: draft
workstream:
  - replace-with-workstream
author: your-username
created: YYYY-MM-DD
updated: YYYY-MM-DD
tags:
  - tag1
  - tag2
llm_compatibility:
  - gpt-4o
description: |
  One or two sentences describing what this skill does and when to use it.
security_classification: internal
delivery_modes:
  - copilot-chat          # available now — copy/paste into any LLM interface
  # - api-endpoint        # uncomment when portal REST API is available
  # - mcp-tool            # uncomment when MCP server is deployed
  # - copilot-studio      # uncomment when Copilot Studio connector is configured
  # - in-house-llm        # uncomment when internal LLM API is available
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

<!-- Describe the skill's purpose, intended users, and context in which it should be invoked. -->

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `{{INPUT_ONE}}` | string | Yes | What this input represents |

## Outputs

| Name | Type | Description |
|---|---|---|
| `OUTPUT_ONE` | string | What this output represents |

## Skill Definition

<!--
  For an LLM skill, this is typically a system prompt or a set of instructions
  that define the skill's behaviour. For an MCP tool, describe the tool definition.
-->

```
[System Prompt / Skill Instructions]

You are a specialized assistant for {{SKILL_DOMAIN}}.

When given {{INPUT_ONE}}, you will:
1. Step one
2. Step two
3. Step three

Always format your response as:
- Bullet 1
- Bullet 2
```

## Usage

### LLM Chat

```
@workspace #file:path/to/relevant-file.ts
<describe the task using this skill>
```

### Standalone (any LLM)

1. Copy the skill definition above.
2. Set it as the system prompt.
3. Provide `{{INPUT_ONE}}` as the user message.

### As an MCP Tool (future)

```json
{
  "tool": "replace-with-unique-kebab-case-id",
  "parameters": {
    "input_one": "your value"
  }
}
```

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
