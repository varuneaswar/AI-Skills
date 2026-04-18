---
id: replace-with-unique-kebab-case-id
title: Replace With Human-Readable Title
type: automation
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
  One or two sentences describing what this automation does and what it replaces.
security_classification: internal
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

<!-- Describe the automation's purpose, trigger conditions, and expected outcomes.
     What repetitive task does it eliminate? -->

## Prerequisites

- Tool / dependency version X or higher
- Environment variable `{{ENV_VAR_ONE}}` set to the appropriate value
- Access to [service / system name]

## Configuration

```yaml
# Replace placeholder values — never commit real credentials
env:
  API_ENDPOINT: "{{API_ENDPOINT}}"   # e.g., https://api.example.com
  API_KEY: "{{API_KEY}}"             # store in your secrets manager
  INPUT_ONE: "{{INPUT_ONE}}"
```

## Implementation

```bash
#!/usr/bin/env bash
# automation: replace-with-unique-kebab-case-id
# description: Replace With Human-Readable Title
# version: 1.0.0
#
# Usage:
#   INPUT_ONE="<value>" ./automation-name.sh
#
# Environment variables:
#   API_ENDPOINT  - Base URL of the target service (required)
#   API_KEY       - Authentication token (required; use a secrets manager)
#   INPUT_ONE     - Description of this variable

set -euo pipefail

: "${API_ENDPOINT:?API_ENDPOINT is required}"
: "${API_KEY:?API_KEY is required}"
: "${INPUT_ONE:?INPUT_ONE is required}"

# Step 1 — describe what this step does
echo "Starting automation with INPUT_ONE=${INPUT_ONE}"

# Step 2 — main logic
RESPONSE=$(curl -sS -X POST "${API_ENDPOINT}/endpoint" \
  -H "Authorization: Bearer ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d "{\"input\": \"${INPUT_ONE}\"}")

echo "Response: ${RESPONSE}"

# Step 3 — post-process output
OUTPUT_ONE=$(echo "${RESPONSE}" | jq -r '.result')
echo "OUTPUT_ONE=${OUTPUT_ONE}"
```

> **Note:** This is a reference implementation. Adapt it to your environment, secrets management solution, and execution platform (GitHub Actions, Jenkins, Azure DevOps, etc.).

## GitHub Actions Example

```yaml
name: Run Automation
on:
  workflow_dispatch:
    inputs:
      input_one:
        description: INPUT_ONE value
        required: true

jobs:
  run:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Execute automation
        env:
          API_ENDPOINT: ${{ vars.API_ENDPOINT }}
          API_KEY: ${{ secrets.API_KEY }}
          INPUT_ONE: ${{ inputs.input_one }}
        run: bash workstreams/<work-stream>/automations/automation-name.sh
```

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `INPUT_ONE` | string | Yes | What this input represents |

## Outputs

| Name | Type | Description |
|---|---|---|
| `OUTPUT_ONE` | string | What this output represents |

## Security Notes

<!-- Required for automations:
     - What systems does this access?
     - What credentials are required and how should they be managed?
     - Is the automation idempotent? -->

## Testing Notes

| Environment | Tested | Notes |
|---|---|---|
| Local (bash) | ✅ | Tested on YYYY-MM-DD |
| GitHub Actions | ⬜ | Not yet tested |

## Changelog

### 1.0.0 — YYYY-MM-DD
- Initial version.
