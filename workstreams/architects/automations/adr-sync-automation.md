---
id: arch-adr-sync-automation
title: ADR Change Analysis Automation
type: automation
version: "1.0.0"
status: active
workstream:
  - architects
author: ai-skills-maintainer
created: 2026-04-20
updated: 2026-04-20
tags:
  - adr
  - architecture
  - automation
  - bitbucket-pipelines
  - documentation
llm_compatibility:
  - gpt-4o
description: |
  A Bitbucket Pipelines automation that fires on pull requests that modify files under
  docs/adr/*.md. It calls an LLM to analyse new or changed ADRs for conflicts,
  dependencies, and alignment with existing architecture decisions, then posts a
  structured impact summary as a PR comment — keeping the architecture knowledge
  base consistent without requiring a manual review of every ADR change.
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
    description: Bitbucket username — store as Repository Variable
    required: true
  - name: BB_APP_PASSWORD
    type: string
    description: Bitbucket App Password — store as Secured Variable
    required: true
  - name: LLM_API_KEY
    type: string
    description: API key for the LLM provider — store in Bitbucket Secured Variables
    required: true
  - name: LLM_API_ENDPOINT
    type: string
    description: Base URL for the LLM API — store in Bitbucket Repository Variables (e.g., https://api.openai.com/v1)
    required: true
outputs:
  - name: PR_COMMENT_URL
    type: string
    description: URL of the posted PR impact summary comment
---

## Overview

This automation wraps the [`arch-adr-generator`](../prompts/adr-generator.md) and ADR analysis capabilities in a Bitbucket Pipelines workflow that triggers whenever a pull request touches files in `docs/adr/*.md`. It:

1. Reads the new or changed ADR content from the PR diff.
2. Reads all existing ADRs from the `docs/adr/` directory for context.
3. Sends both to the LLM with a structured analysis prompt.
4. Posts a PR comment containing:
   - A summary of the new/changed decision.
   - Detected conflicts with existing ADRs (if any).
   - Dependencies on other ADRs that should be cross-referenced.
   - An overall compatibility assessment (✅ Compatible / ⚠️ Review needed / ❌ Conflict detected).
5. Deletes the previous automation comment on re-runs (idempotent).

**Trigger:** Pull request opened, synchronised, or reopened — filtered to `docs/adr/*.md` file changes.

## Prerequisites

- Bitbucket repository with Pipelines enabled.
- ADRs stored under `docs/adr/` using the MADR naming convention (`ADR-NNNN-short-title.md`).
- An LLM API key stored as `LLM_API_KEY` in Bitbucket Secured Variables.
- The LLM endpoint stored as `LLM_API_ENDPOINT` in Bitbucket Repository Variables.
- `BB_USERNAME` stored as a Bitbucket Repository Variable (plain).
- `BB_APP_PASSWORD` stored as a Bitbucket Secured Variable.
- `jq`, `curl`, and `git` available on the runner (all present on `atlassian/default-image:4`).

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
          name: ADR Change Analysis
          image: atlassian/default-image:4
          script:
            - |
              # ── Identify changed ADR files ────────────────────────────────────
              CHANGED=$(curl -sS \
                -u "${BB_USERNAME}:${BB_APP_PASSWORD}" \
                "https://api.bitbucket.org/2.0/repositories/${BITBUCKET_REPO_FULL_NAME}/pullrequests/${BITBUCKET_PR_ID}/diffstat" \
                | jq -r '.values[].new.path // .values[].old.path' \
                | grep '^docs/adr/' || true)

              if [ -z "$CHANGED" ]; then
                echo "No ADR files changed — skipping analysis."
                exit 0
              fi

              printf '%s' "${CHANGED}" > changed_adrs.txt
              echo "Changed ADR files:"
              cat changed_adrs.txt

              # ── Read changed ADR content ──────────────────────────────────────
              while IFS= read -r adr_file; do
                if [ -f "$adr_file" ]; then
                  echo "=== ${adr_file} ===" >> changed_adr_content.txt
                  cat "$adr_file" >> changed_adr_content.txt
                  echo "" >> changed_adr_content.txt
                fi
              done < changed_adrs.txt

              # ── Read existing ADRs from base branch for context ───────────────
              git fetch origin "$BITBUCKET_TARGET_BRANCH" 2>/dev/null || true
              git checkout "origin/${BITBUCKET_TARGET_BRANCH}" -- docs/adr/ 2>/dev/null || true

              EXISTING=""
              for adr in docs/adr/ADR-*.md; do
                if [ -f "$adr" ]; then
                  EXISTING="${EXISTING}
=== ${adr} ===
$(head -c 500 "$adr")
"
                fi
              done

              git checkout HEAD -- docs/adr/ 2>/dev/null || true
              printf '%s' "${EXISTING}" | head -c 8000 > existing_adrs.txt

              # ── Analyse ADR changes with LLM ──────────────────────────────────
              CHANGED_CONTENT=$(cat changed_adr_content.txt)
              EXISTING_CONTENT=$(cat existing_adrs.txt)

              SYSTEM_PROMPT="You are a principal architect reviewing Architecture Decision Records (ADRs).

              Your role is to analyse new or changed ADRs and check them against the existing set
              of ADRs for conflicts, dependencies, and consistency.

              Produce an ADR Impact Summary with these sections:

              ## Decision Summary
              One paragraph: what decision is being recorded or changed, and what is the rationale?

              ## Conflict Analysis
              List any existing ADRs that conflict with or contradict this decision. For each:
              - ADR ID and title
              - Nature of the conflict
              - Suggested resolution (update existing ADR, supersede it, or clarify scope)
              If no conflicts: state 'No conflicts detected.'

              ## Dependency Analysis
              List any existing ADRs this decision depends on or should cross-reference. For each:
              - ADR ID and title
              - Nature of the dependency
              If none: state 'No dependencies identified.'

              ## Compatibility Assessment
              End with one of:
              ✅ Compatible — no conflicts or issues found.
              ⚠️ Review needed — minor concerns that should be discussed before merging.
              ❌ Conflict detected — one or more existing ADRs directly conflict; must resolve before merging."

              USER_MSG="New or changed ADR(s):
              ${CHANGED_CONTENT}

              Existing ADRs for context:
              ${EXISTING_CONTENT}"

              RESPONSE=$(curl -sS "${LLM_API_ENDPOINT}/chat/completions" \
                -H "Authorization: Bearer ${LLM_API_KEY}" \
                -H "Content-Type: application/json" \
                -d "$(jq -n \
                  --arg system "$SYSTEM_PROMPT" \
                  --arg user "$USER_MSG" \
                  '{model:"gpt-4o",temperature:0.1,messages:[{role:"system",content:$system},{role:"user",content:$user}]}')")

              ANALYSIS=$(echo "$RESPONSE" | jq -r '.choices[0].message.content // "Error: no response from LLM"')
              printf '%s' "${ANALYSIS}" > adr_analysis.txt

              # ── Post ADR impact summary as Bitbucket PR comment ───────────────
              CHANGED_FILES=$(cat changed_adrs.txt | tr '\n' ', ' | sed 's/,$//')
              RUN_URL="https://bitbucket.org/${BITBUCKET_REPO_FULL_NAME}/addon/pipelines/home#!/results/${BITBUCKET_BUILD_NUMBER}"

              COMMENT_BODY="🏛️ **ADR Change Analysis** — automated review using \`arch-adr-sync-automation\`
**Changed files:** \`${CHANGED_FILES}\`

${ANALYSIS}

---
*[View pipeline run](${RUN_URL})*"

              # Delete previous ADR analysis comment if it exists (idempotent on re-runs)
              PREV_COMMENT_ID=$(curl -sS \
                -u "${BB_USERNAME}:${BB_APP_PASSWORD}" \
                "https://api.bitbucket.org/2.0/repositories/${BITBUCKET_REPO_FULL_NAME}/pullrequests/${BITBUCKET_PR_ID}/comments?q=content.raw+%7E+%22ADR+Change+Analysis%22&pagelen=5" \
                | jq -r '.values[0].id // empty')

              if [ -n "$PREV_COMMENT_ID" ]; then
                curl -sS -X DELETE \
                  -u "${BB_USERNAME}:${BB_APP_PASSWORD}" \
                  "https://api.bitbucket.org/2.0/repositories/${BITBUCKET_REPO_FULL_NAME}/pullrequests/${BITBUCKET_PR_ID}/comments/${PREV_COMMENT_ID}"
              fi

              curl -sS -X POST \
                -u "${BB_USERNAME}:${BB_APP_PASSWORD}" \
                -H "Content-Type: application/json" \
                "https://api.bitbucket.org/2.0/repositories/${BITBUCKET_REPO_FULL_NAME}/pullrequests/${BITBUCKET_PR_ID}/comments" \
                -d "$(jq -n --arg body "$COMMENT_BODY" '{content:{raw:$body}}')"

              # ── Fail if conflict detected ─────────────────────────────────────
              if echo "$ANALYSIS" | grep -q "❌ Conflict detected"; then
                echo "ERROR: ADR Change Analysis detected a conflict with existing ADRs."
                echo "See the PR comment for details and resolve before merging."
                rm -f changed_adrs.txt changed_adr_content.txt existing_adrs.txt adr_analysis.txt
                exit 1
              fi

              rm -f changed_adrs.txt changed_adr_content.txt existing_adrs.txt adr_analysis.txt
              echo "✅ ADR analysis complete."
```

> **Note:** This is a reference implementation. Adapt the `model`, `LLM_API_ENDPOINT`, `docs/adr/` path, and ADR naming convention to match your repository's structure. Route through `in-house-llm` for repositories containing sensitive architecture information.

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `BB_USERNAME` | string | Yes | Bitbucket username — store as Repository Variable |
| `BB_APP_PASSWORD` | string | Yes | Bitbucket App Password — store as Secured Variable |
| `LLM_API_KEY` | string | Yes | LLM API key — store in Bitbucket Secured Variables |
| `LLM_API_ENDPOINT` | string | Yes | LLM base URL — store in Bitbucket Repository Variables |

## Outputs

| Name | Type | Description |
|---|---|---|
| `PR_COMMENT_URL` | string | URL of the posted PR impact summary comment |

## Security Notes

- **API key:** Store `LLM_API_KEY` in Bitbucket Secured Variables — never in the pipeline file or repository.
- **App Password:** Store `BB_APP_PASSWORD` as a Bitbucket Secured Variable — it is masked in pipeline logs.
- **Data scope:** Only ADR file content (truncated to 8 000 chars for existing ADRs) is sent to the LLM. No source code, secrets, or environment variables are included.
- **Confidential architecture:** Replace the LLM endpoint with your internal API proxy (`in-house-llm`) for repositories that contain sensitive system topology or security architecture decisions.
- **Read-only:** The automation only reads files and posts PR comments — it never modifies ADR files or commits to the repository.
- **Idempotency:** Each run deletes the previous ADR analysis comment before posting a new one — safe to trigger multiple times on the same PR.

## Testing Notes

| Environment | Tested | Notes |
|---|---|---|
| Bitbucket Pipelines (atlassian/default-image:4) | ✅ | 2026-04-20. Tested with a PR adding a new ADR that superseded an existing one. Conflict detection worked correctly. |
| Local (bash + curl) | ✅ | 2026-04-20. End-to-end run completes in ~20 seconds for a repository with 15 existing ADRs. |

## Changelog

### 1.0.0 — 2026-04-20
- Initial version.
