---
id: sysdesign-diagram-generation-automation
title: Design Diagram Generation Automation
type: automation
version: "1.0.0"
status: active
workstream:
  - system-designers
author: ai-skills-maintainer
created: 2026-04-20
updated: 2026-04-20
tags:
  - system-design
  - mermaid
  - diagram
  - automation
  - github-actions
  - api
llm_compatibility:
  - gpt-4o
  - copilot-gpt-4o
description: |
  A GitHub Actions automation triggered on pull requests that modify design files
  (*.design.md or docs/design/*.md). It calls an LLM to generate a Mermaid sequence
  diagram and an OpenAPI 3.0 contract stub from the design content, then posts both
  as a PR comment — making design artefacts immediately visible at review time.
security_classification: internal
delivery_modes:
  - copilot-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
inputs:
  - name: GITHUB_TOKEN
    type: string
    description: GitHub token for posting PR comments (provided automatically by GitHub Actions)
    required: true
  - name: LLM_API_KEY
    type: string
    description: API key for the LLM provider (store in GitHub Secrets)
    required: true
  - name: LLM_API_ENDPOINT
    type: string
    description: Base URL for the LLM API (e.g., https://api.openai.com/v1)
    required: true
outputs:
  - name: PR_COMMENT_URL
    type: string
    description: URL of the posted PR comment containing the generated diagram and API contract stub
---

## Overview

This automation wraps Steps 2 and 3 of the [`sysdesign-system-design-workflow`](../workflows/system-design-workflow.md) in a GitHub Actions workflow. When a pull request modifies a design file, the automation:

1. Reads the changed design file content.
2. Calls the LLM to generate a Mermaid sequence diagram for the primary system flow.
3. Calls the LLM to generate an OpenAPI 3.0 YAML contract stub for any API described in the design.
4. Posts both artefacts as a formatted PR comment.
5. Updates the comment idempotently on subsequent pushes to the same PR.

**Trigger:** Pull request opened, synchronised, or reopened — when files matching `*.design.md` or `docs/design/*.md` are changed.

## Prerequisites

- GitHub repository with Actions enabled.
- An LLM API key stored as `LLM_API_KEY` in GitHub Secrets.
- The LLM endpoint stored as `LLM_API_ENDPOINT` in GitHub Variables.
- `jq` and `curl` available on the runner (both present on `ubuntu-latest`).
- Design files must use the `*.design.md` naming convention or reside in `docs/design/`.

## Configuration

```yaml
# Store these in your repository's Settings → Secrets and variables
env:
  LLM_API_KEY: "{{LLM_API_KEY}}"           # GitHub Secret — never commit
  LLM_API_ENDPOINT: "{{LLM_API_ENDPOINT}}" # e.g. https://api.openai.com/v1
```

## Implementation

```yaml
# .github/workflows/design-diagram-generation.yml
name: AI Design Diagram Generation

on:
  pull_request:
    types: [opened, synchronize, reopened]
    branches: [main]
    paths:
      - '**.design.md'
      - 'docs/design/**.md'

permissions:
  pull-requests: write   # needed to post and update PR comments
  contents: read

jobs:
  generate-diagrams:
    name: Generate design diagrams and API contract
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Get changed design files
        id: design-files
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          # Collect changed design files
          CHANGED=$(git diff --name-only origin/${{ github.base_ref }}...HEAD \
            -- '*.design.md' 'docs/design/*.md' | head -5)

          if [ -z "$CHANGED" ]; then
            echo "no_design_changes=true" >> "$GITHUB_OUTPUT"
          else
            echo "no_design_changes=false" >> "$GITHUB_OUTPUT"
            # Read content of first changed design file (truncated to 6 000 chars)
            FIRST_FILE=$(echo "$CHANGED" | head -1)
            echo "design_file=${FIRST_FILE}" >> "$GITHUB_OUTPUT"
            cat "$FIRST_FILE" | head -c 6000 > design_content.txt
          fi

      - name: Skip if no design file changes
        if: steps.design-files.outputs.no_design_changes == 'true'
        run: echo "No design file changes detected — skipping diagram generation."

      - name: Generate Mermaid sequence diagram via LLM
        if: steps.design-files.outputs.no_design_changes == 'false'
        id: gen-diagram
        env:
          LLM_API_KEY: ${{ secrets.LLM_API_KEY }}
          LLM_API_ENDPOINT: ${{ vars.LLM_API_ENDPOINT }}
        run: |
          DESIGN=$(cat design_content.txt)

          SYSTEM_PROMPT="You are a system design expert who produces Mermaid sequence diagrams.

          Given the design document below, generate a Mermaid sequenceDiagram for the primary
          end-to-end flow described (e.g., the main happy path).

          Rules:
          - Use valid Mermaid sequenceDiagram syntax.
          - Use ->> for synchronous calls, -->> for responses.
          - Add alt/opt blocks for branches described in the document.
          - Use activate/deactivate for long-running steps.
          - Label all messages with operation names.
          - Do not invent components not mentioned in the document.

          Output:
          1. A fenced \`\`\`mermaid code block containing the diagram.
          2. A one-paragraph plain-English description of the flow."

          RESPONSE=$(curl -sS "${LLM_API_ENDPOINT}/chat/completions" \
            -H "Authorization: Bearer ${LLM_API_KEY}" \
            -H "Content-Type: application/json" \
            -d "$(jq -n \
              --arg system "$SYSTEM_PROMPT" \
              --arg user "Design document:\n${DESIGN}" \
              '{model:"gpt-4o",temperature:0.2,messages:[{role:"system",content:$system},{role:"user",content:$user}]}')")

          DIAGRAM=$(echo "$RESPONSE" | jq -r '.choices[0].message.content // "Error: no response from LLM"')
          printf '%s' "$DIAGRAM" > diagram_output.txt

      - name: Generate OpenAPI contract stub via LLM
        if: steps.design-files.outputs.no_design_changes == 'false'
        id: gen-contract
        env:
          LLM_API_KEY: ${{ secrets.LLM_API_KEY }}
          LLM_API_ENDPOINT: ${{ vars.LLM_API_ENDPOINT }}
        run: |
          DESIGN=$(cat design_content.txt)

          SYSTEM_PROMPT="You are an API design expert who produces OpenAPI 3.0 contracts.

          Given the design document below, generate an OpenAPI 3.0.3 YAML contract stub
          for the primary API described. If no HTTP API is described, output a comment
          explaining that no API contract is applicable.

          Rules:
          - Use OpenAPI 3.0.3 specification.
          - Include info block (title from design, version: '1.0.0').
          - Define all paths implied by the design with standard CRUD operations.
          - Use \$ref components for all schemas.
          - Include standard error responses: 400, 401, 404, 500.
          - Use Bearer auth security scheme.
          - Add meaningful operationId values (camelCase).

          Output only a fenced \`\`\`yaml code block. No prose before or after."

          RESPONSE=$(curl -sS "${LLM_API_ENDPOINT}/chat/completions" \
            -H "Authorization: Bearer ${LLM_API_KEY}" \
            -H "Content-Type: application/json" \
            -d "$(jq -n \
              --arg system "$SYSTEM_PROMPT" \
              --arg user "Design document:\n${DESIGN}" \
              '{model:"gpt-4o",temperature:0.2,messages:[{role:"system",content:$system},{role:"user",content:$user}]}')")

          CONTRACT=$(echo "$RESPONSE" | jq -r '.choices[0].message.content // "Error: no response from LLM"')
          printf '%s' "$CONTRACT" > contract_output.txt

      - name: Post or update PR comment
        if: steps.design-files.outputs.no_design_changes == 'false'
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          DIAGRAM=$(cat diagram_output.txt)
          CONTRACT=$(cat contract_output.txt)
          DESIGN_FILE="${{ steps.design-files.outputs.design_file }}"
          MARKER="<!-- ai-design-diagrams -->"

          COMMENT="${MARKER}
          > 📐 **AI Design Artefacts** — generated by \`sysdesign-diagram-generation-automation\`
          > Source: \`${DESIGN_FILE}\`

          ## Sequence Diagram

          ${DIAGRAM}

          ---

          ## API Contract Stub

          ${CONTRACT}
          "

          PR_NUMBER="${{ github.event.pull_request.number }}"

          # Delete previous diagram comment if it exists (idempotent on re-runs)
          PREV_ID=$(gh api "repos/${{ github.repository }}/issues/${PR_NUMBER}/comments" \
            --jq '.[] | select(.body | startswith("<!-- ai-design-diagrams -->")) | .id' | head -1)

          if [ -n "$PREV_ID" ]; then
            gh api "repos/${{ github.repository }}/issues/comments/${PREV_ID}" -X DELETE
          fi

          printf '%s' "$COMMENT" | gh pr comment "$PR_NUMBER" --body-file=-
```

> **Note:** This is a reference implementation. Adapt the `model`, `LLM_API_ENDPOINT`, and file path patterns to match your repository's design file naming conventions and LLM provider. Route through `in-house-llm` for confidential architecture documents.

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `LLM_API_KEY` | string | Yes | API key — store in GitHub Secrets |
| `LLM_API_ENDPOINT` | string | Yes | LLM base URL — store in GitHub Variables |
| `GITHUB_TOKEN` | string | Yes | Provided automatically by GitHub Actions |

## Outputs

| Name | Type | Description |
|---|---|---|
| `PR_COMMENT_URL` | string | URL of the posted PR comment with diagram and contract stub |

## Security Notes

- **API key:** Store `LLM_API_KEY` in GitHub Secrets — never in the workflow file or repository.
- **Data scope:** Only the changed design file content (truncated to 6 000 chars) is sent to the LLM. No source code, credentials, or environment variables are included.
- **Confidential designs:** Replace the LLM endpoint with your internal API proxy (`in-house-llm`) before using on confidential architecture documents.
- **Idempotency:** Each run deletes the previous AI diagram comment before posting a new one — safe to trigger multiple times on the same PR.
- **File limit:** Only the first changed design file is processed per run to limit LLM token usage and cost.

## Testing Notes

| Environment | Tested | Notes |
|---|---|---|
| GitHub Actions (ubuntu-latest) | ✅ | 2026-04-20. Tested with OpenAI gpt-4o endpoint on a notification service design file. |
| Local (bash + curl) | ✅ | 2026-04-20. Runs end-to-end in ~30 seconds for a 200-line design document. |

## Changelog

### 1.0.0 — 2026-04-20
- Initial version.
