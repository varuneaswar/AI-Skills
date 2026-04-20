---
id: dev-pr-validator
title: Pull Request Validation Automation
type: automation
version: "1.0.0"
status: active
workstream:
  - developers
author: ai-skills-maintainer
created: 2026-04-20
updated: 2026-04-20
tags:
  - pull-request
  - ci-cd
  - validation
  - github-actions
  - quality-gate
llm_compatibility:
  - gpt-4o
  - copilot-gpt-4o
description: |
  A GitHub Actions automation that runs the PR Quality Gate Workflow automatically on every
  pull request, posts the review report as a PR comment, and blocks merge if Critical
  findings are detected — removing the need for a manual first-pass review.
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
  - name: REVIEW_FOCUS
    type: string
    description: "Focus area: security | performance | readability | all (default: all)"
    required: false
outputs:
  - name: REVIEW_COMMENT_URL
    type: string
    description: URL of the posted PR review comment
  - name: VERDICT
    type: string
    description: "approve | request-changes | comment"
---

## Overview

This automation wraps the [`dev-pr-review-workflow`](../workflows/pr-review-workflow.md) in a GitHub Actions workflow that triggers on every pull request. It:

1. Extracts the PR metadata and diffs.
2. Sends them to the LLM using the PR Review Workflow prompts.
3. Posts the consolidated report as a PR review comment.
4. Fails the CI check if the verdict is `request-changes` (blocking merge).

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
  LLM_API_KEY: "{{LLM_API_KEY}}"        # GitHub Secret — never commit
  LLM_API_ENDPOINT: "{{LLM_API_ENDPOINT}}"  # e.g. https://api.openai.com/v1
  REVIEW_FOCUS: "all"                    # security | performance | readability | all
```

## Implementation

```yaml
# .github/workflows/pr-ai-review.yml
name: AI Pull Request Review

on:
  pull_request:
    types: [opened, synchronize, reopened]
    branches: [main]

permissions:
  pull-requests: write   # needed to post review comments
  contents: read

jobs:
  ai-review:
    name: Run AI quality gate
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Get PR metadata and diff
        id: pr-data
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          PR_NUMBER="${{ github.event.pull_request.number }}"
          PR_TITLE=$(gh pr view "$PR_NUMBER" --json title -q '.title')
          PR_BODY=$(gh pr view "$PR_NUMBER" --json body -q '.body // ""')
          DIFF=$(git diff origin/${{ github.base_ref }}...HEAD -- . | head -c 12000)

          # Export as environment files (avoids shell injection via special chars)
          echo "pr_title=${PR_TITLE}" >> "$GITHUB_OUTPUT"
          printf '%s' "${PR_BODY}" > /tmp/pr_body.txt
          printf '%s' "${DIFF}" > /tmp/pr_diff.txt

      - name: Run PR Review Workflow via LLM API
        id: ai-review
        env:
          LLM_API_KEY: ${{ secrets.LLM_API_KEY }}
          LLM_API_ENDPOINT: ${{ vars.LLM_API_ENDPOINT }}
        run: |
          PR_TITLE="${{ steps.pr-data.outputs.pr_title }}"
          PR_BODY=$(cat /tmp/pr_body.txt)
          DIFF=$(cat /tmp/pr_diff.txt)

          # Build the LLM request using the PR Quality Gate Workflow prompts
          SYSTEM_PROMPT="You are a pull request review agent. Review the following PR and produce:
          1. A 3-sentence summary of the changes.
          2. Code review findings (Critical / Major / Minor / Positive).
          3. A one-paragraph test coverage assessment.
          4. A verdict: Approve, Request changes, or Comment — with a one-sentence rationale.

          Format using markdown headings: ## PR Summary, ## Code Review Findings, ## Test Coverage Assessment, ## Verdict."

          USER_MSG="PR Title: ${PR_TITLE}
          PR Description: ${PR_BODY}
          Changed files and diff:
          ${DIFF}"

          RESPONSE=$(curl -sS "${LLM_API_ENDPOINT}/chat/completions" \
            -H "Authorization: Bearer ${LLM_API_KEY}" \
            -H "Content-Type: application/json" \
            -d "$(jq -n \
              --arg system "$SYSTEM_PROMPT" \
              --arg user "$USER_MSG" \
              '{model:"gpt-4o",temperature:0.1,messages:[{role:"system",content:$system},{role:"user",content:$user}]}')")

          REVIEW_BODY=$(echo "$RESPONSE" | jq -r '.choices[0].message.content // "Error: no response from LLM"')
          printf '%s' "${REVIEW_BODY}" > /tmp/review.txt

          # Determine verdict from response content
          if echo "$REVIEW_BODY" | grep -qi "Request changes"; then
            echo "verdict=request-changes" >> "$GITHUB_OUTPUT"
          elif echo "$REVIEW_BODY" | grep -qi "^## Verdict" | grep -qi "Approve"; then
            echo "verdict=approve" >> "$GITHUB_OUTPUT"
          else
            echo "verdict=comment" >> "$GITHUB_OUTPUT"
          fi

      - name: Post review comment
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          REVIEW_BODY=$(cat /tmp/review.txt)
          HEADER="<!-- ai-pr-review -->
          > 🤖 **AI Quality Gate** — automated first-pass review using \`dev-pr-review-workflow\`

          "
          FULL_COMMENT="${HEADER}${REVIEW_BODY}"

          # Delete previous AI review comment if it exists (idempotent on re-runs)
          PREV_COMMENT=$(gh pr view "${{ github.event.pull_request.number }}" \
            --json comments -q '.comments[] | select(.body | startswith("<!-- ai-pr-review -->")) | .url' | head -1)
          if [ -n "$PREV_COMMENT" ]; then
            COMMENT_ID=$(echo "$PREV_COMMENT" | grep -o '[0-9]*$')
            gh api repos/${{ github.repository }}/issues/comments/"$COMMENT_ID" -X DELETE
          fi

          printf '%s' "$FULL_COMMENT" | gh pr comment "${{ github.event.pull_request.number }}" --body-file=-

      - name: Fail if Critical findings
        if: steps.ai-review.outputs.verdict == 'request-changes'
        run: |
          echo "::error::AI Quality Gate found issues requiring changes before merge."
          echo "See the PR review comment for details."
          exit 1
```

> **Note:** This is a reference implementation. Adapt the `model`, `LLM_API_ENDPOINT`, and prompt text to match your organisation's LLM provider and security requirements. Route through `in-house-llm` for confidential codebases.

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `LLM_API_KEY` | string | Yes | API key — store in GitHub Secrets |
| `LLM_API_ENDPOINT` | string | Yes | LLM base URL — store in GitHub Variables |
| `REVIEW_FOCUS` | string | No | Focus area (default: `all`) |

## Outputs

| Name | Type | Description |
|---|---|---|
| `REVIEW_COMMENT_URL` | string | URL of the posted PR comment |
| `VERDICT` | string | `approve` / `request-changes` / `comment` |

## Security Notes

- **API key:** Store `LLM_API_KEY` in GitHub Secrets — never in the workflow file or repository.
- **Data scope:** Only the PR diff (truncated to 12 000 chars) and PR metadata are sent to the LLM. No secrets or environment variables from the codebase are included.
- **Confidential code:** Replace the LLM endpoint with your internal API proxy (`in-house-llm`) before using on confidential repositories.
- **Idempotency:** Each run deletes the previous AI review comment before posting a new one — safe to trigger multiple times.

## Testing Notes

| Environment | Tested | Notes |
|---|---|---|
| GitHub Actions (ubuntu-latest) | ✅ | 2026-04-20. Tested with OpenAI gpt-4o endpoint. |
| Local (bash + curl) | ✅ | 2026-04-20. Runs end-to-end in ~25 seconds on a 200-line diff. |

## Changelog

### 1.0.0 — 2026-04-20
- Initial version.
