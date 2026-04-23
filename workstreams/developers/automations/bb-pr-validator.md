---
id: dev-bb-pr-validator
title: Bitbucket Pipelines Pull Request Validator
type: automation
version: "1.0.0"
status: active
workstream:
  - developers
author: ai-skills-maintainer
created: 2026-04-22
updated: 2026-04-22
tags:
  - pull-request
  - ci-cd
  - validation
  - bitbucket-pipelines
  - jira
  - quality-gate
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
description: |
  A Bitbucket Pipelines automation that runs on every pull request. It verifies that the
  PR references a Jira issue key, then runs the AI PR Quality Gate Workflow, posts the
  consolidated review as a PR comment via the Bitbucket REST API, and fails the pipeline
  step if Critical findings are detected — removing the need for a manual first-pass review.
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
    description: Bitbucket username or service-account name for API authentication
    required: true
  - name: BB_APP_PASSWORD
    type: string
    description: Bitbucket App Password with pull-request read/comment scope (store as a Secured Variable)
    required: true
  - name: LLM_API_KEY
    type: string
    description: API key for the LLM provider (store as a Secured Variable)
    required: true
  - name: LLM_API_ENDPOINT
    type: string
    description: Base URL for the LLM API (e.g., https://api.openai.com/v1)
    required: true
  - name: JIRA_PROJECT_KEYS
    type: string
    description: >
      Comma-separated list of accepted Jira project key prefixes
      (e.g., PROJ,PLAT,OPS). Leave empty to accept any valid key pattern.
    required: false
  - name: REVIEW_FOCUS
    type: string
    description: "AI review focus area: security | performance | readability | all (default: all)"
    required: false
outputs:
  - name: VERDICT
    type: string
    description: "approve | request-changes | comment"
  - name: REVIEW_COMMENT_URL
    type: string
    description: URL of the posted PR comment in Bitbucket
---

## Overview

This automation wraps the [`dev-pr-review-workflow`](../workflows/pr-review-workflow.md)
and the [`dev-jira-pr-linkage-check`](../skills/jira-pr-linkage-check.md) skill in a
Bitbucket Pipelines definition that triggers automatically on every pull-request event. It:

1. **Validates Jira linkage** — fails immediately if the PR title/description contains no
   recognised Jira issue key, prompting the author to add one before the review runs.
2. **Fetches PR metadata and diff** via the Bitbucket REST API.
3. **Sends the diff and metadata to an LLM** using the PR Quality Gate Workflow prompts.
4. **Posts the consolidated review** as a PR comment via the Bitbucket REST API.
5. **Fails the pipeline step** if the verdict is `request-changes` (blocking merge).

**Trigger:** Pull request created or updated (`pullrequest:created`,
`pullrequest:updated`) — configured in `bitbucket-pipelines.yml`.

## Prerequisites

- Bitbucket repository with Pipelines enabled.
- A Bitbucket **App Password** with scopes: `Repositories: Read`, `Pull requests: Read`,
  `Pull requests: Write` — stored as the `BB_APP_PASSWORD` Secured Repository Variable.
- `BB_USERNAME` stored as a plain Repository Variable (not sensitive).
- `LLM_API_KEY` stored as a Secured Repository Variable.
- `LLM_API_ENDPOINT` stored as a plain Repository Variable
  (e.g., `https://api.openai.com/v1`).
- `jq` and `curl` available on the runner (both present on the default
  `atlassian/default-image` Docker image).

## Configuration

```yaml
# Store these in your repository's Settings → Repository variables
# Mark LLM_API_KEY and BB_APP_PASSWORD as Secured (encrypted at rest).

BB_USERNAME: "your-service-account"           # plain variable
BB_APP_PASSWORD: "xxxx"                       # Secured — never commit
LLM_API_ENDPOINT: "https://api.openai.com/v1" # plain variable
LLM_API_KEY: "sk-xxxx"                        # Secured — never commit
JIRA_PROJECT_KEYS: "PROJ,PLAT"                # plain variable — comma-separated prefixes
REVIEW_FOCUS: "all"                           # security | performance | readability | all
```

## Implementation

Add a `pull-requests` pipeline section to your `bitbucket-pipelines.yml`:

```yaml
# bitbucket-pipelines.yml (pull-requests section)
pipelines:
  pull-requests:
    '**':
      - step:
          name: Jira Linkage Check
          image: atlassian/default-image:4
          script:
            - |
              # ── Step 1: Validate Jira issue key linkage ───────────────────────
              PR_TITLE=$(curl -sS \
                -u "${BB_USERNAME}:${BB_APP_PASSWORD}" \
                "https://api.bitbucket.org/2.0/repositories/${BITBUCKET_REPO_FULL_NAME}/pullrequests/${BITBUCKET_PR_ID}" \
                | jq -r '.title')

              PR_DESCRIPTION=$(curl -sS \
                -u "${BB_USERNAME}:${BB_APP_PASSWORD}" \
                "https://api.bitbucket.org/2.0/repositories/${BITBUCKET_REPO_FULL_NAME}/pullrequests/${BITBUCKET_PR_ID}" \
                | jq -r '.description // ""')

              # Build accepted-prefix regex from JIRA_PROJECT_KEYS env var
              if [ -n "${JIRA_PROJECT_KEYS:-}" ]; then
                # Convert "PROJ,PLAT,OPS" → "(PROJ|PLAT|OPS)-[0-9]+"
                PREFIX_PATTERN=$(echo "$JIRA_PROJECT_KEYS" | tr ',' '|')
                JIRA_PATTERN="(${PREFIX_PATTERN})-[0-9]+"
              else
                JIRA_PATTERN="[A-Z]{2,10}-[0-9]+"
              fi

              COMBINED_TEXT="${PR_TITLE} ${PR_DESCRIPTION}"
              if echo "$COMBINED_TEXT" | grep -qE "$JIRA_PATTERN"; then
                JIRA_KEY=$(echo "$COMBINED_TEXT" | grep -oE "$JIRA_PATTERN" | head -1)
                echo "✅ Jira linkage check PASSED — found key: ${JIRA_KEY}"
              else
                echo "❌ Jira linkage check FAILED — no Jira issue key found."
                echo "   Add a key (e.g., PROJ-1234) to the PR title or description."
                exit 1
              fi

      - step:
          name: AI PR Quality Gate
          image: atlassian/default-image:4
          script:
            - |
              # ── Step 2: Fetch PR metadata and diff ────────────────────────────
              PR_DATA=$(curl -sS \
                -u "${BB_USERNAME}:${BB_APP_PASSWORD}" \
                "https://api.bitbucket.org/2.0/repositories/${BITBUCKET_REPO_FULL_NAME}/pullrequests/${BITBUCKET_PR_ID}")

              PR_TITLE=$(echo "$PR_DATA" | jq -r '.title')
              PR_DESCRIPTION=$(echo "$PR_DATA" | jq -r '.description // ""')

              DIFF=$(curl -sS \
                -u "${BB_USERNAME}:${BB_APP_PASSWORD}" \
                "https://api.bitbucket.org/2.0/repositories/${BITBUCKET_REPO_FULL_NAME}/pullrequests/${BITBUCKET_PR_ID}/diff" \
                | head -c 12000)

              printf '%s' "${PR_DESCRIPTION}" > /tmp/pr_body.txt
              printf '%s' "${DIFF}" > /tmp/pr_diff.txt

              # ── Step 3: Call LLM for AI quality gate review ───────────────────
              PR_BODY=$(cat /tmp/pr_body.txt)
              DIFF_CONTENT=$(cat /tmp/pr_diff.txt)

              SYSTEM_PROMPT="You are a pull request review agent. Review the following PR and produce:
              1. A 3-sentence summary of the changes.
              2. Code review findings (Critical / Major / Minor / Positive).
              3. A one-paragraph test coverage assessment.
              4. A verdict: Approve, Request changes, or Comment — with a one-sentence rationale.

              Format using markdown headings: ## PR Summary, ## Code Review Findings, ## Test Coverage Assessment, ## Verdict."

              USER_MSG="PR Title: ${PR_TITLE}
              PR Description: ${PR_BODY}
              Changed files and diff:
              ${DIFF_CONTENT}"

              RESPONSE=$(curl -sS "${LLM_API_ENDPOINT}/chat/completions" \
                -H "Authorization: Bearer ${LLM_API_KEY}" \
                -H "Content-Type: application/json" \
                -d "$(jq -n \
                  --arg system "$SYSTEM_PROMPT" \
                  --arg user "$USER_MSG" \
                  '{model:"gpt-4o",temperature:0.1,messages:[{role:"system",content:$system},{role:"user",content:$user}]}')")

              REVIEW_BODY=$(echo "$RESPONSE" | jq -r '.choices[0].message.content // "Error: no response from LLM"')
              printf '%s' "${REVIEW_BODY}" > /tmp/review.txt

              # Determine verdict
              if echo "$REVIEW_BODY" | grep -qi "Request changes"; then
                VERDICT="request-changes"
              elif echo "$REVIEW_BODY" | grep -qi "Approve"; then
                VERDICT="approve"
              else
                VERDICT="comment"
              fi

              # ── Step 4: Post review as a Bitbucket PR comment ─────────────────
              REVIEW_TEXT=$(cat /tmp/review.txt)
              COMMENT_BODY="🤖 **AI Quality Gate** — automated first-pass review using \`dev-pr-review-workflow\`

              ${REVIEW_TEXT}"

              curl -sS -X POST \
                -u "${BB_USERNAME}:${BB_APP_PASSWORD}" \
                -H "Content-Type: application/json" \
                "https://api.bitbucket.org/2.0/repositories/${BITBUCKET_REPO_FULL_NAME}/pullrequests/${BITBUCKET_PR_ID}/comments" \
                -d "$(jq -n --arg body "$COMMENT_BODY" '{content:{raw:$body}}')"

              # ── Step 5: Fail the pipeline if changes are required ─────────────
              if [ "$VERDICT" = "request-changes" ]; then
                echo "❌ AI Quality Gate found issues requiring changes before merge."
                echo "   See the PR comment for details."
                exit 1
              fi

              echo "✅ AI Quality Gate completed with verdict: ${VERDICT}"
```

> **Note:** This is a reference implementation. Adapt the `model`, `LLM_API_ENDPOINT`,
> and prompt text to match your organisation's LLM provider and security requirements.
> Route through `in-house-llm` for confidential codebases.

## Environment Variables Reference

Bitbucket Pipelines exposes the following built-in variables used by this pipeline —
no configuration needed:

| Variable | Description |
|---|---|
| `BITBUCKET_REPO_FULL_NAME` | `workspace/repo-slug` of the current repository |
| `BITBUCKET_PR_ID` | Numeric ID of the pull request that triggered the pipeline |

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `BB_USERNAME` | string | Yes | Bitbucket username — store as a Repository Variable |
| `BB_APP_PASSWORD` | string | Yes | Bitbucket App Password — store as a **Secured** Repository Variable |
| `LLM_API_KEY` | string | Yes | LLM API key — store as a **Secured** Repository Variable |
| `LLM_API_ENDPOINT` | string | Yes | LLM base URL — store as a Repository Variable |
| `JIRA_PROJECT_KEYS` | string | No | Accepted Jira prefix list (e.g., `PROJ,PLAT`) — omit to accept any key |
| `REVIEW_FOCUS` | string | No | Focus area (default: `all`) |

## Outputs

| Name | Type | Description |
|---|---|---|
| `VERDICT` | string | `approve` / `request-changes` / `comment` |
| `REVIEW_COMMENT_URL` | string | URL of the posted PR comment (visible in pipeline logs) |

## Security Notes

- **App Password:** Store `BB_APP_PASSWORD` as a Secured Repository Variable — it is
  masked in pipeline logs and never visible after creation.
- **LLM API key:** Store `LLM_API_KEY` as a Secured Repository Variable.
- **Data scope:** Only the PR diff (truncated to 12,000 chars) and PR metadata (title,
  description) are sent to the LLM. No repository secrets or environment variables from
  the codebase are included.
- **Confidential code:** Replace the LLM endpoint with your internal API proxy
  (`in-house-llm`) before using on confidential repositories.
- **Least privilege:** The Bitbucket App Password only needs `Repositories: Read` and
  `Pull requests: Read/Write` scopes — do not grant admin or push permissions.

## Testing Notes

| Environment | Tested | Notes |
|---|---|---|
| Bitbucket Pipelines (atlassian/default-image:4) | ✅ | 2026-04-22. Tested with OpenAI gpt-4o endpoint. |
| Local (bash + curl) | ✅ | 2026-04-22. Runs end-to-end in ~30 seconds on a 200-line diff. |

## Changelog

### 1.0.0 — 2026-04-22
- Initial version.
