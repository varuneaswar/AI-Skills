---
id: dev-jira-pr-linkage-check
title: Jira PR Linkage Check
type: skill
version: "1.0.0"
status: active
workstream:
  - developers
author: ai-skills-maintainer
created: 2026-04-22
updated: 2026-04-22
tags:
  - jira
  - pull-request
  - bitbucket
  - validation
  - quality-gate
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
  - gemini-1.5-pro
description: |
  Validates that a pull request references a Jira issue key (e.g., PROJ-1234) in its
  title or description. Returns a pass/fail result with the extracted Jira key when
  found, or a suggested fix when missing. Designed to be composed into CI pipelines
  (Bitbucket Pipelines, Bitbucket Pipelines, Jenkins) as a lightweight quality gate step.
security_classification: internal
inputs:
  - name: PR_TITLE
    type: string
    description: The pull request title
    required: true
  - name: PR_DESCRIPTION
    type: string
    description: The pull request body / description text
    required: false
  - name: JIRA_PROJECT_KEYS
    type: string
    description: >
      Comma-separated list of valid Jira project key prefixes to accept
      (e.g., "PROJ,PLAT,OPS"). Leave blank to accept any key matching the
      standard pattern ([A-Z]{2,10}-[0-9]+).
    required: false
outputs:
  - name: LINKAGE_RESULT
    type: string
    description: >
      "PASS" if a valid Jira issue key is found, "FAIL" otherwise — always
      accompanied by a one-sentence explanation and, on PASS, the extracted key.
delivery_modes:
  - llm-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
---

## Overview

Enforces the team convention that every pull request must be traceable to a Jira ticket.
Run this skill as the first step in a PR quality gate so that unlinked pull requests are
caught immediately, before the more expensive AI code review runs.

**Typical use cases:**

- Bitbucket Pipelines step that fails the build when a PR has no Jira reference.
- Bitbucket Pipelines check that adds a comment asking the author to add a ticket link.
- Manual check: paste the PR title and description into any LLM to verify traceability.

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `{{PR_TITLE}}` | string | Yes | The pull request title |
| `{{PR_DESCRIPTION}}` | string | No | The full PR body / description |
| `{{JIRA_PROJECT_KEYS}}` | string | No | Accepted project key prefixes, comma-separated (e.g., `PROJ,PLAT`) |

## Outputs

| Name | Type | Description |
|---|---|---|
| `LINKAGE_RESULT` | string | `PASS` or `FAIL` with explanation and extracted key on pass |

## Skill Definition

```
[System Prompt]

You are a PR compliance checker for a software engineering team that uses Jira for
issue tracking.

Your task is to verify that the pull request references at least one Jira issue key.

A Jira issue key matches the pattern: one or more uppercase letters (the project key),
a hyphen, and one or more digits — for example: PROJ-123, PLAT-4567, OPS-9.

Accepted project key prefixes (if provided): {{JIRA_PROJECT_KEYS}}
If no prefixes are provided, accept any key matching the standard pattern.

PR Title: {{PR_TITLE}}
PR Description: {{PR_DESCRIPTION}}

Instructions:
1. Search both the title and description for a Jira issue key.
2. If one or more keys are found and (where prefixes are specified) at least one matches
   an accepted prefix, respond with:

   RESULT: PASS
   KEY: <first matching key>
   EXPLANATION: Pull request is linked to Jira issue <key>.

3. If no valid key is found, respond with:

   RESULT: FAIL
   KEY: none
   EXPLANATION: No Jira issue key was found in the PR title or description.
   SUGGESTED FIX: Add the Jira issue key to the PR title, for example:
   "<KEY>-<NUMBER>: <original title>" or reference it in the description as
   "Closes <KEY>-<NUMBER>".

Output only the structured block above — no additional commentary.
```

## Usage

### Standalone (any LLM — current)

1. Copy the skill definition above.
2. Replace `{{PR_TITLE}}`, `{{PR_DESCRIPTION}}`, and `{{JIRA_PROJECT_KEYS}}` with real values.
3. Paste into your LLM interface.

### Bitbucket Pipelines (current)

This skill is embedded as Step 1 in the
[Bitbucket Pipelines PR Validator](../automations/bb-pr-validator.md).
See that automation for a complete, ready-to-use pipeline definition.

### LLM Chat (current)

```
I have a PR with the following details. Check whether it references a Jira issue key.

PR Title: {{PR_TITLE}}
PR Description: {{PR_DESCRIPTION}}
Accepted Jira project prefixes: {{JIRA_PROJECT_KEYS}}
```

### API Endpoint (future portal)

```http
POST {{API_BASE_URL}}/packages/developers-core-pack/invoke
Content-Type: application/json

{
  "skill_id": "dev-jira-pr-linkage-check",
  "inputs": {
    "pr_title": "Add rate limiting to /api/orders",
    "pr_description": "Implements token-bucket rate limiting.",
    "jira_project_keys": "PROJ,PLAT"
  }
}
```

### MCP Tool (future)

```json
{
  "tool": "dev-jira-pr-linkage-check",
  "parameters": {
    "pr_title": "Add rate limiting to /api/orders",
    "pr_description": "Implements token-bucket rate limiting.",
    "jira_project_keys": "PROJ,PLAT"
  }
}
```

## Examples

### Example 1 — Key present in title (PASS)

**Input:**
- `PR_TITLE`: `PROJ-1042: Add token-bucket rate limiting to /api/orders`
- `PR_DESCRIPTION`: `Adds configurable rate limiting middleware.`
- `JIRA_PROJECT_KEYS`: `PROJ,PLAT`

**Output:**
```
RESULT: PASS
KEY: PROJ-1042
EXPLANATION: Pull request is linked to Jira issue PROJ-1042.
```

### Example 2 — Key present in description only (PASS)

**Input:**
- `PR_TITLE`: `Add token-bucket rate limiting to /api/orders`
- `PR_DESCRIPTION`: `Implements rate limiting. Closes PROJ-1042.`
- `JIRA_PROJECT_KEYS`: `PROJ`

**Output:**
```
RESULT: PASS
KEY: PROJ-1042
EXPLANATION: Pull request is linked to Jira issue PROJ-1042.
```

### Example 3 — No key found (FAIL)

**Input:**
- `PR_TITLE`: `Add token-bucket rate limiting to /api/orders`
- `PR_DESCRIPTION`: `Adds configurable rate limiting middleware.`
- `JIRA_PROJECT_KEYS`: `PROJ,PLAT`

**Output:**
```
RESULT: FAIL
KEY: none
EXPLANATION: No Jira issue key was found in the PR title or description.
SUGGESTED FIX: Add the Jira issue key to the PR title, for example:
"PROJ-<NUMBER>: Add token-bucket rate limiting to /api/orders" or reference it
in the description as "Closes PROJ-<NUMBER>".
```

### Example 4 — Key present but wrong project prefix (FAIL)

**Input:**
- `PR_TITLE`: `OLD-55: Add token-bucket rate limiting to /api/orders`
- `PR_DESCRIPTION`: `Adds configurable rate limiting middleware.`
- `JIRA_PROJECT_KEYS`: `PROJ,PLAT`

**Output:**
```
RESULT: FAIL
KEY: none
EXPLANATION: Issue key OLD-55 was found but does not match any accepted project
prefix (PROJ, PLAT).
SUGGESTED FIX: Reference a ticket from the PROJ or PLAT project instead, or update
the accepted prefix list if OLD is a valid project.
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | 2026-04-22. Reliable pattern matching. Handles keys in title and description. |
| claude-3-5-sonnet | ✅ | 2026-04-22. Correctly rejects wrong-prefix keys. |

## Changelog

### 1.0.0 — 2026-04-22
- Initial version.
