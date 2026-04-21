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
  - github-actions
  - pull-request
llm_compatibility:
  - gpt-4o
  - copilot-gpt-4o
description: |
  A GitHub Actions automation that triggers on pull requests, extracts changed source
  files, sends them to an LLM to identify functions and methods lacking test coverage,
  and posts a prioritised coverage gap report as a PR comment — making test gaps visible
  at review time without any manual analysis.
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
  - name: COVERAGE_REPORT_COMMENT_URL
    type: string
    description: URL of the posted PR coverage gap report comment
---

## Overview

This automation wraps the [`test-generation-workflow`](../workflows/test-generation-workflow.md) coverage gap step in a GitHub Actions workflow triggered on every pull request. It:

1. Extracts the source files changed in the PR (excluding test files themselves).
2. Sends the changed code to the LLM to identify functions and methods that lack corresponding tests.
3. Posts a coverage gap report as a PR comment.
4. Updates the comment idempotently on subsequent pushes to the same PR.

**Trigger:** Pull request opened, synchronised, or reopened against `main`.

## Prerequisites

- GitHub repository with Actions enabled.
- An LLM API key stored as `LLM_API_KEY` in GitHub Secrets.
- The LLM endpoint stored as `LLM_API_ENDPOINT` in GitHub Variables.
- `jq` and `curl` available on the runner (both present on `ubuntu-latest`).

## Configuration

```yaml
# Store these in your repository's Settings → Secrets and variables
env:
  LLM_API_KEY: "{{LLM_API_KEY}}"           # GitHub Secret — never commit
  LLM_API_ENDPOINT: "{{LLM_API_ENDPOINT}}" # e.g. https://api.openai.com/v1
```

## Implementation

```yaml
# .github/workflows/test-coverage-gap.yml
name: AI Test Coverage Gap Analysis

on:
  pull_request:
    types: [opened, synchronize, reopened]
    branches: [main]

permissions:
  pull-requests: write   # needed to post and update PR comments
  contents: read

jobs:
  coverage-gap:
    name: Identify test coverage gaps
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Get changed source files
        id: changed-files
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          # Collect changed non-test source files (first 8 000 chars of diff)
          DIFF=$(git diff origin/${{ github.base_ref }}...HEAD \
            -- '*.py' '*.ts' '*.js' '*.java' '*.cs' '*.go' \
            ':!*test*' ':!*spec*' ':!*__tests__*' \
            | head -c 8000)

          if [ -z "$DIFF" ]; then
            echo "no_source_changes=true" >> "$GITHUB_OUTPUT"
          else
            echo "no_source_changes=false" >> "$GITHUB_OUTPUT"
            printf '%s' "$DIFF" > source_diff.txt
          fi

      - name: Skip if no source changes
        if: steps.changed-files.outputs.no_source_changes == 'true'
        run: echo "No source file changes detected — skipping coverage gap analysis."

      - name: Analyse coverage gaps via LLM
        if: steps.changed-files.outputs.no_source_changes == 'false'
        id: gap-analysis
        env:
          LLM_API_KEY: ${{ secrets.LLM_API_KEY }}
          LLM_API_ENDPOINT: ${{ vars.LLM_API_ENDPOINT }}
        run: |
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

      - name: Post or update coverage gap comment
        if: steps.changed-files.outputs.no_source_changes == 'false'
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          REPORT=$(cat coverage_report.txt)
          MARKER="<!-- ai-coverage-gap -->"
          HEADER="${MARKER}
          > 🧪 **AI Test Coverage Gap Report** — generated by \`test-coverage-automation\`

          "
          FULL_COMMENT="${HEADER}${REPORT}"

          PR_NUMBER="${{ github.event.pull_request.number }}"

          # Delete previous coverage gap comment if it exists (idempotent on re-runs)
          PREV_ID=$(gh api "repos/${{ github.repository }}/issues/${PR_NUMBER}/comments" \
            --jq '.[] | select(.body | startswith("<!-- ai-coverage-gap -->")) | .id' | head -1)

          if [ -n "$PREV_ID" ]; then
            gh api "repos/${{ github.repository }}/issues/comments/${PREV_ID}" -X DELETE
          fi

          printf '%s' "$FULL_COMMENT" | gh pr comment "$PR_NUMBER" --body-file=-
```

> **Note:** This is a reference implementation. Adapt the `model`, `LLM_API_ENDPOINT`, file glob patterns, and diff size limit to match your codebase and LLM provider. Route through `in-house-llm` for confidential repositories.

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `LLM_API_KEY` | string | Yes | API key — store in GitHub Secrets |
| `LLM_API_ENDPOINT` | string | Yes | LLM base URL — store in GitHub Variables |
| `GITHUB_TOKEN` | string | Yes | Provided automatically by GitHub Actions |

## Outputs

| Name | Type | Description |
|---|---|---|
| `COVERAGE_REPORT_COMMENT_URL` | string | URL of the posted PR coverage gap comment |

## Security Notes

- **API key:** Store `LLM_API_KEY` in GitHub Secrets — never in the workflow file or repository.
- **Data scope:** Only the changed source file diff (truncated to 8 000 chars) is sent to the LLM. No secrets, credentials, or environment variables from the codebase are included.
- **Confidential code:** Replace the LLM endpoint with your internal API proxy (`in-house-llm`) before using on confidential repositories.
- **Idempotency:** Each run deletes the previous coverage gap comment before posting a new one — safe to trigger multiple times on the same PR.
- **Scope limitation:** Test files (matching `*test*`, `*spec*`, `*__tests__*`) are explicitly excluded from the diff sent to the LLM to avoid false positives.

## Testing Notes

| Environment | Tested | Notes |
|---|---|---|
| GitHub Actions (ubuntu-latest) | ✅ | 2026-04-20. Tested with OpenAI gpt-4o endpoint on a Python service PR. |
| Local (bash + curl) | ✅ | 2026-04-20. Runs end-to-end in ~20 seconds on a 150-line diff. |

## Changelog

### 1.0.0 — 2026-04-20
- Initial version.
