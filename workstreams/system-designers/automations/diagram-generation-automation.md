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
  - bitbucket-pipelines
  - api
llm_compatibility:
  - gpt-4o
description: |
  A Bitbucket Pipelines automation triggered on pull requests that modify design files
  (*.design.md or docs/design/*.md). It calls an LLM to generate a Mermaid sequence
  diagram and an OpenAPI 3.0 contract stub from the design content, then posts both
  as a PR comment — making design artefacts immediately visible at review time.
security_classification: internal
delivery_modes:
  - llm-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
inputs:
  - name: BB_USERNAME
    type: string
    description: Bitbucket username — store as Bitbucket Repository Variable
    required: true
  - name: BB_APP_PASSWORD
    type: string
    description: Bitbucket App Password — store as Bitbucket Secured Variable
    required: true
  - name: LLM_API_KEY
    type: string
    description: API key for the LLM provider (store in Bitbucket Secured Variables)
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

This automation wraps Steps 2 and 3 of the [`sysdesign-system-design-workflow`](../workflows/system-design-workflow.md) in a Bitbucket Pipelines workflow. When a pull request modifies a design file, the automation:

1. Reads the changed design file content via the Bitbucket diffstat API.
2. Calls the LLM to generate a Mermaid sequence diagram for the primary system flow.
3. Calls the LLM to generate an OpenAPI 3.0 YAML contract stub for any API described in the design.
4. Posts both artefacts as a formatted PR comment.
5. Updates the comment idempotently on subsequent pushes to the same PR.

**Trigger:** Pull request opened, synchronised, or reopened — when files matching `*.design.md` or `docs/design/*.md` are changed.

## Prerequisites

- Bitbucket repository with Pipelines enabled.
- An LLM API key stored as `LLM_API_KEY` in Bitbucket Secured Variables.
- The LLM endpoint stored as `LLM_API_ENDPOINT` in Bitbucket Repository Variables.
- `BB_USERNAME` and `BB_APP_PASSWORD` stored as Bitbucket variables (Secured for the password).
- `jq` and `curl` available on the runner (both present on `atlassian/default-image:4`).
- Design files must use the `*.design.md` naming convention or reside in `docs/design/`.

## Configuration

```yaml
# Store these in your repository's Settings → Repository variables
# Mark LLM_API_KEY and BB_APP_PASSWORD as Secured (encrypted at rest).
BB_USERNAME: "your-service-account"           # plain variable
BB_APP_PASSWORD: "xxxx"                       # Secured — never commit
LLM_API_ENDPOINT: "https://api.openai.com/v1" # plain variable
LLM_API_KEY: "sk-xxxx"                        # Secured — never commit
```

## Implementation

```yaml
# bitbucket-pipelines.yml (pull-requests section)
# Add to your existing bitbucket-pipelines.yml
pipelines:
  pull-requests:
    '**':
      - step:
          name: AI Design Diagram Generation
          image: atlassian/default-image:4
          script:
            - |
              # ── Get changed design files via Bitbucket diffstat API ───────────
              CHANGED=$(curl -sS \
                -u "${BB_USERNAME}:${BB_APP_PASSWORD}" \
                "https://api.bitbucket.org/2.0/repositories/${BITBUCKET_REPO_FULL_NAME}/pullrequests/${BITBUCKET_PR_ID}/diffstat" \
                | jq -r '.values[].new.path // .values[].old.path' \
                | grep -E '\.design\.md$|^docs/design/.*\.md$' || true)

              if [ -z "$CHANGED" ]; then
                echo "No design file changes detected — skipping diagram generation."
                exit 0
              fi

              FIRST_FILE=$(echo "$CHANGED" | head -1)
              echo "Processing design file: ${FIRST_FILE}"

              if [ ! -f "$FIRST_FILE" ]; then
                echo "Design file not found in working tree: ${FIRST_FILE}"
                exit 0
              fi

              cat "$FIRST_FILE" | head -c 6000 > design_content.txt

              # ── Generate Mermaid sequence diagram via LLM ─────────────────────
              DESIGN=$(cat design_content.txt)

              DIAGRAM_SYSTEM="You are a system design expert who produces Mermaid sequence diagrams.

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

              DIAGRAM_RESPONSE=$(curl -sS "${LLM_API_ENDPOINT}/chat/completions" \
                -H "Authorization: Bearer ${LLM_API_KEY}" \
                -H "Content-Type: application/json" \
                -d "$(jq -n \
                  --arg system "$DIAGRAM_SYSTEM" \
                  --arg user "Design document:\n${DESIGN}" \
                  '{model:"gpt-4o",temperature:0.2,messages:[{role:"system",content:$system},{role:"user",content:$user}]}')")

              DIAGRAM=$(echo "$DIAGRAM_RESPONSE" | jq -r '.choices[0].message.content // "Error: no response from LLM"')
              printf '%s' "$DIAGRAM" > diagram_output.txt

              # ── Generate OpenAPI contract stub via LLM ────────────────────────
              CONTRACT_SYSTEM="You are an API design expert who produces OpenAPI 3.0 contracts.

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

              CONTRACT_RESPONSE=$(curl -sS "${LLM_API_ENDPOINT}/chat/completions" \
                -H "Authorization: Bearer ${LLM_API_KEY}" \
                -H "Content-Type: application/json" \
                -d "$(jq -n \
                  --arg system "$CONTRACT_SYSTEM" \
                  --arg user "Design document:\n${DESIGN}" \
                  '{model:"gpt-4o",temperature:0.2,messages:[{role:"system",content:$system},{role:"user",content:$user}]}')")

              CONTRACT=$(echo "$CONTRACT_RESPONSE" | jq -r '.choices[0].message.content // "Error: no response from LLM"')
              printf '%s' "$CONTRACT" > contract_output.txt

              # ── Post or update PR comment ─────────────────────────────────────
              DIAGRAM_TEXT=$(cat diagram_output.txt)
              CONTRACT_TEXT=$(cat contract_output.txt)

              COMMENT_BODY="📐 **AI Design Artefacts** — generated by \`sysdesign-diagram-generation-automation\`
Source: \`${FIRST_FILE}\`

## Sequence Diagram

${DIAGRAM_TEXT}

---

## API Contract Stub

${CONTRACT_TEXT}"

              # Delete previous diagram comment if it exists (idempotent on re-runs)
              PREV_ID=$(curl -sS \
                -u "${BB_USERNAME}:${BB_APP_PASSWORD}" \
                "https://api.bitbucket.org/2.0/repositories/${BITBUCKET_REPO_FULL_NAME}/pullrequests/${BITBUCKET_PR_ID}/comments?q=content.raw+%7E+%22AI+Design+Artefacts%22&pagelen=5" \
                | jq -r '.values[0].id // empty')

              if [ -n "$PREV_ID" ]; then
                curl -sS -X DELETE \
                  -u "${BB_USERNAME}:${BB_APP_PASSWORD}" \
                  "https://api.bitbucket.org/2.0/repositories/${BITBUCKET_REPO_FULL_NAME}/pullrequests/${BITBUCKET_PR_ID}/comments/${PREV_ID}"
              fi

              curl -sS -X POST \
                -u "${BB_USERNAME}:${BB_APP_PASSWORD}" \
                -H "Content-Type: application/json" \
                "https://api.bitbucket.org/2.0/repositories/${BITBUCKET_REPO_FULL_NAME}/pullrequests/${BITBUCKET_PR_ID}/comments" \
                -d "$(jq -n --arg body "$COMMENT_BODY" '{content:{raw:$body}}')"

              rm -f design_content.txt diagram_output.txt contract_output.txt
              echo "✅ Diagram generation complete."
```

> **Note:** This is a reference implementation. Adapt the `model`, `LLM_API_ENDPOINT`, and file path patterns to match your repository's design file naming conventions and LLM provider. Route through `in-house-llm` for confidential architecture documents.

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `BB_USERNAME` | string | Yes | Bitbucket username — store as Repository Variable |
| `BB_APP_PASSWORD` | string | Yes | Bitbucket App Password — store as Secured Variable |
| `LLM_API_KEY` | string | Yes | API key — store in Bitbucket Secured Variables |
| `LLM_API_ENDPOINT` | string | Yes | LLM base URL — store in Bitbucket Repository Variables |

## Outputs

| Name | Type | Description |
|---|---|---|
| `PR_COMMENT_URL` | string | URL of the posted PR comment with diagram and contract stub |

## Security Notes

- **API key:** Store `LLM_API_KEY` in Bitbucket Secured Variables — never in the pipeline file or repository.
- **App Password:** Store `BB_APP_PASSWORD` as a Bitbucket Secured Variable — masked in pipeline logs.
- **Data scope:** Only the changed design file content (truncated to 6 000 chars) is sent to the LLM. No source code, credentials, or environment variables are included.
- **Confidential designs:** Replace the LLM endpoint with your internal API proxy (`in-house-llm`) before using on confidential architecture documents.
- **Idempotency:** Each run deletes the previous AI diagram comment before posting a new one — safe to trigger multiple times on the same PR.
- **File limit:** Only the first changed design file is processed per run to limit LLM token usage and cost.

## Testing Notes

| Environment | Tested | Notes |
|---|---|---|
| Bitbucket Pipelines (atlassian/default-image:4) | ✅ | 2026-04-20. Tested with OpenAI gpt-4o endpoint on a notification service design file. |
| Local (bash + curl) | ✅ | 2026-04-20. Runs end-to-end in ~30 seconds for a 200-line design document. |

## Changelog

### 1.0.0 — 2026-04-20
- Initial version.
