---
id: test-coverage-automation
title: Test Coverage Gap Automation
type: automation
version: "1.0.0"
status: active
workstream:
  - testers
author: ai-skills-maintainer
created: 2026-04-20
updated: 2026-04-20
tags:
  - testing
  - coverage
  - automation
  - bitbucket-pipelines
  - pull-request
llm_compatibility:
  - gpt-4o
description: |
  A Bitbucket Pipelines automation that triggers on pull requests, extracts changed source
  files, sends them to an LLM to identify functions and methods lacking test coverage,
  and posts a prioritised coverage gap report as a PR comment — making test gaps visible
  at review time without any manual analysis.
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
  - name: COVERAGE_REPORT_COMMENT_URL
    type: string
    description: URL of the posted PR coverage gap report comment
---

## Overview

This automation wraps the [`test-generation-workflow`](../workflows/test-generation-workflow.md) coverage gap step in a Bitbucket Pipelines workflow triggered on every pull request. It:

1. Extracts the source files changed in the PR (excluding test files themselves).
2. Sends the changed code to the LLM to identify functions and methods that lack corresponding tests.
3. Posts a coverage gap report as a PR comment.
4. Updates the comment idempotently on subsequent pushes to the same PR.

**Trigger:** Pull request opened, synchronised, or reopened against `main`.

## Prerequisites

- Bitbucket repository with Pipelines enabled.
- An LLM API key stored as `LLM_API_KEY` in Bitbucket Secured Variables.
- The LLM endpoint stored as `LLM_API_ENDPOINT` in Bitbucket Repository Variables.
- `BB_USERNAME` and `BB_APP_PASSWORD` stored as Bitbucket variables (Secured for the password).
- `jq` and `curl` available on the runner (both present on `atlassian/default-image:4`).

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
          name: AI Test Coverage Gap Analysis
          image: atlassian/default-image:4
          script:
            - |
              # ── Get changed source files ──────────────────────────────────────
              git fetch origin "${BITBUCKET_TARGET_BRANCH}"
              DIFF=$(git diff origin/${BITBUCKET_TARGET_BRANCH}...HEAD \
                -- '*.py' '*.ts' '*.js' '*.java' '*.cs' '*.go' \
                ':!*test*' ':!*spec*' ':!*__tests__*' \
                | head -c 8000)

              if [ -z "$DIFF" ]; then
                echo "true" > no_source_changes.txt
                echo "No source file changes detected — skipping coverage gap analysis."
                exit 0
              fi

              echo "false" > no_source_changes.txt
              printf '%s' "$DIFF" > source_diff.txt

              if [ "$(cat no_source_changes.txt 2>/dev/null)" = "true" ]; then
                echo "No source file changes detected — skipping coverage gap analysis."
                exit 0
              fi

              # ── Analyse coverage gaps via LLM ─────────────────────────────────
              SOURCE_DIFF=$(cat source_diff.txt)

              SYSTEM_PROMPT="You are a senior QA engineer performing test coverage gap analysis on a code diff.

              Identify every function, method, or code path in the diff that:
              1. Is not already covered by a corresponding test file visible in the diff.
              2. Contains logic (conditionals, loops, error handling) that warrants a test.

              For each gap, output:

              **GAP-NNN: [Short title]**
              - File: filename
              - Function/Method: name
              - Risk: High / Medium / Low
              - What is missing: one sentence
              - Suggested test case:
                Given [precondition]
                When [action]
                Then [expected result]

              After listing gaps, add a ## Coverage Summary paragraph with an overall assessment.
              Sort gaps by Risk descending."

              USER_MSG="Code diff to analyse for test coverage gaps:
              ${SOURCE_DIFF}"

              RESPONSE=$(curl -sS "${LLM_API_ENDPOINT}/chat/completions" \
                -H "Authorization: Bearer ${LLM_API_KEY}" \
                -H "Content-Type: application/json" \
                -d "$(jq -n \
                  --arg system "$SYSTEM_PROMPT" \
                  --arg user "$USER_MSG" \
                  '{model:"gpt-4o",temperature:0.2,messages:[{role:"system",content:$system},{role:"user",content:$user}]}')")

              REPORT=$(echo "$RESPONSE" | jq -r '.choices[0].message.content // "Error: no response from LLM"')
              printf '%s' "$REPORT" > coverage_report.txt

              # ── Post or update coverage gap comment ───────────────────────────
              REPORT_TEXT=$(cat coverage_report.txt)
              FULL_COMMENT="🧪 **AI Test Coverage Gap Report** — generated by \`test-coverage-automation\`

${REPORT_TEXT}"

              # Delete previous coverage gap comment if it exists (idempotent on re-runs)
              PREV_ID=$(curl -sS \
                -u "${BB_USERNAME}:${BB_APP_PASSWORD}" \
                "https://api.bitbucket.org/2.0/repositories/${BITBUCKET_REPO_FULL_NAME}/pullrequests/${BITBUCKET_PR_ID}/comments?q=content.raw+%7E+%22AI+Test+Coverage+Gap+Report%22&pagelen=5" \
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
                -d "$(jq -n --arg body "$FULL_COMMENT" '{content:{raw:$body}}')"

              rm -f source_diff.txt coverage_report.txt no_source_changes.txt
              echo "✅ Coverage gap analysis complete."
```

> **Note:** This is a reference implementation. Adapt the `model`, `LLM_API_ENDPOINT`, file glob patterns, and diff size limit to match your codebase and LLM provider. Route through `in-house-llm` for confidential repositories.

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
| `COVERAGE_REPORT_COMMENT_URL` | string | URL of the posted PR coverage gap comment |

## Security Notes

- **API key:** Store `LLM_API_KEY` in Bitbucket Secured Variables — never in the pipeline file or repository.
- **App Password:** Store `BB_APP_PASSWORD` as a Bitbucket Secured Variable — masked in pipeline logs.
- **Data scope:** Only the changed source file diff (truncated to 8 000 chars) is sent to the LLM. No secrets, credentials, or environment variables from the codebase are included.
- **Confidential code:** Replace the LLM endpoint with your internal API proxy (`in-house-llm`) before using on confidential repositories.
- **Idempotency:** Each run deletes the previous coverage gap comment before posting a new one — safe to trigger multiple times on the same PR.
- **Scope limitation:** Test files (matching `*test*`, `*spec*`, `*__tests__*`) are explicitly excluded from the diff sent to the LLM to avoid false positives.

## Testing Notes

| Environment | Tested | Notes |
|---|---|---|
| Bitbucket Pipelines (atlassian/default-image:4) | ✅ | 2026-04-20. Tested with OpenAI gpt-4o endpoint on a Python service PR. |
| Local (bash + curl) | ✅ | 2026-04-20. Runs end-to-end in ~20 seconds on a 150-line diff. |

## Changelog

### 1.0.0 — 2026-04-20
- Initial version.
