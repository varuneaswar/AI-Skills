---
id: replace-with-unique-kebab-case-id
title: Replace With Human-Readable Title
type: prompt
version: "1.0.0"
status: draft
workstream:
  - replace-with-workstream   # architects | capacity-management | developers | performance-engineers | sre | system-designers | testers | shared
author: your-github-username
created: YYYY-MM-DD
updated: YYYY-MM-DD
tags:
  - tag1
  - tag2
llm_compatibility:
  - gpt-4o                    # list all models this prompt has been tested against
description: |
  One or two sentences describing what this prompt does and when to use it.
security_classification: internal   # public | internal | confidential
inputs:
  - name: PLACEHOLDER_ONE
    type: string
    description: What this input represents
    required: true
  - name: PLACEHOLDER_TWO
    type: string
    description: What this input represents
    required: false
---

## Overview

<!-- Expand on the description. What task does this prompt accomplish?
     What is the expected output format? -->

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `{{PLACEHOLDER_ONE}}` | string | Yes | What this input represents |
| `{{PLACEHOLDER_TWO}}` | string | No | What this input represents |

## Prompt

```
Replace the placeholders below with your actual values before using.

[System / Context]
You are a helpful assistant specializing in {{PLACEHOLDER_ONE}}.

[Task]
{{PLACEHOLDER_TWO}}

[Output Format]
Respond in plain text / bullet list / JSON (choose one and delete the others).
```

## Usage

1. Copy the prompt text above.
2. Replace `{{PLACEHOLDER_ONE}}` and `{{PLACEHOLDER_TWO}}` with your values.
3. Paste into your LLM interface (Copilot Chat, ChatGPT, Claude, etc.).

**GitHub Copilot Chat shortcut:**
```
@workspace /explain {{PLACEHOLDER_ONE}}
```
*(Update or remove this line if a Copilot slash-command does not apply.)*

## Examples

### Example 1

**Input:**
- `PLACEHOLDER_ONE`: *your example value*
- `PLACEHOLDER_TWO`: *your example value*

**Output:**
```
Paste a real, redacted output from a tested LLM session here.
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | Tested on YYYY-MM-DD |
| claude-3-5-sonnet | ⬜ | Not yet tested |

<!-- Known limitations, edge cases, or model-specific behaviour -->

## Changelog

### 1.0.0 — YYYY-MM-DD
- Initial version.
