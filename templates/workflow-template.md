---
id: replace-with-unique-kebab-case-id
title: Replace With Human-Readable Title
type: workflow
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
  One or two sentences describing what this workflow orchestrates and the end-to-end outcome.
security_classification: internal
delivery_modes:
  - copilot-chat          # available now — copy/paste into any LLM interface
  # - api-endpoint        # uncomment when portal REST API is available
  # - mcp-tool            # uncomment when MCP server is deployed
  # - copilot-studio      # uncomment when Copilot Studio connector is configured
  # - in-house-llm        # uncomment when internal LLM API is available
dependencies:
  - dependent-skill-or-prompt-id
inputs:
  - name: INPUT_ONE
    type: string
    description: What this input represents
    required: true
outputs:
  - name: OUTPUT_ONE
    type: string
    description: The final output of the workflow
---

## Overview

<!-- Describe the workflow's purpose, who runs it, and under what circumstances. -->

## Steps

### Step 1 — [Step Name]

**Asset used:** `linked-prompt-or-skill-id`  
**Input:** `{{INPUT_ONE}}`  
**Output:** Intermediate result fed into Step 2.

```
[Prompt / instruction for this step]
```

---

### Step 2 — [Step Name]

**Asset used:** `another-skill-id`  
**Input:** Output from Step 1.  
**Output:** Final result.

```
[Prompt / instruction for this step]
```

---

## Flow Diagram

```
[INPUT_ONE]
     │
     ▼
[Step 1: action] ──▶ intermediate_result
                              │
                              ▼
                   [Step 2: action] ──▶ [OUTPUT_ONE]
```

## Usage

1. Gather all required inputs.
2. Execute Step 1 — paste the Step 1 prompt into your LLM interface and provide `{{INPUT_ONE}}`.
3. Take the output of Step 1 and use it as input to Step 2.
4. The output of Step 2 is your final result.

> **Automation tip:** If you have access to a workflow engine (e.g., Bitbucket Pipelines, Azure Logic Apps, n8n), you can automate these steps — see `automations/` for reference implementations.

## Examples

### Example 1

**Input:** `INPUT_ONE` = *your example value*

**Step 1 Output:**
```
Paste example intermediate output here.
```

**Final Output:**
```
Paste example final output here.
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | Tested on YYYY-MM-DD |

## Changelog

### 1.0.0 — YYYY-MM-DD
- Initial version.
