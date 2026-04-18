---
id: dev-summarize-pull-request
title: Summarize Pull Request for Reviewers
type: prompt
version: "1.0.0"
status: active
workstream:
  - developers
author: ai-skills-maintainer
created: 2024-01-01
updated: 2024-01-01
tags:
  - pull-request
  - code-review
  - documentation
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
  - gemini-1.5-pro
  - copilot-gpt-4o
description: |
  Generates a concise, reviewer-friendly summary of a pull request from its title, description,
  and list of changed files. Reduces review time by surfacing intent and risk areas up front.
security_classification: internal
inputs:
  - name: PR_TITLE
    type: string
    description: The pull request title
    required: true
  - name: PR_DESCRIPTION
    type: string
    description: The pull request body / description text
    required: true
  - name: CHANGED_FILES
    type: string
    description: Newline-separated list of changed file paths
    required: true
  - name: AUDIENCE
    type: string
    description: "Intended audience for the summary: 'reviewer', 'manager', or 'release-notes'"
    required: false
---

## Overview

Use this prompt when you want to produce a structured summary of a pull request that helps reviewers quickly understand:

1. **What** changed and why.
2. **Which areas** of the codebase are affected.
3. **What risks** the reviewer should pay attention to.

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `{{PR_TITLE}}` | string | Yes | The pull request title |
| `{{PR_DESCRIPTION}}` | string | Yes | The full PR body / description |
| `{{CHANGED_FILES}}` | string | Yes | Newline-separated list of changed file paths |
| `{{AUDIENCE}}` | string | No | `reviewer` (default), `manager`, or `release-notes` |

## Prompt

```
You are a senior software engineer helping a team conduct efficient code reviews.

Given the following pull request information, produce a structured summary.

---
PR Title: {{PR_TITLE}}

PR Description:
{{PR_DESCRIPTION}}

Changed Files:
{{CHANGED_FILES}}

Intended Audience: {{AUDIENCE}}
---

Write the summary using this structure:

## What Changed
One paragraph explaining the purpose and scope of the change.

## Files Affected
Group the changed files by area (e.g., "API layer", "Database", "Tests") and briefly state what changed in each group.

## Key Risks / Review Focus
A bulleted list of up to five things the reviewer should pay close attention to (e.g., edge cases, security implications, breaking changes, missing tests).

## Testing Coverage
State what tests were added or modified and whether you believe coverage is sufficient.

Keep the tone professional and concise. Do not repeat the raw file list verbatim.
```

## Usage

### GitHub Copilot Chat

Open Copilot Chat in your IDE and type:

```
Summarize this PR for reviewers.
PR title: <paste title>
PR description: <paste description>
Changed files: <paste file list>
```

### Standalone (any LLM)

1. Copy the prompt above.
2. Replace `{{PR_TITLE}}`, `{{PR_DESCRIPTION}}`, `{{CHANGED_FILES}}`, and `{{AUDIENCE}}` with real values.
3. Paste into your LLM interface.

## Examples

### Example 1

**Input:**
- `PR_TITLE`: `feat: add rate limiting to public API endpoints`
- `PR_DESCRIPTION`: `Adds token-bucket rate limiting to all public REST endpoints to prevent abuse. Configurable limits per environment via environment variables.`
- `CHANGED_FILES`: `src/middleware/rate-limit.ts\nsrc/app.ts\ntest/middleware/rate-limit.test.ts\ndocs/api.md`
- `AUDIENCE`: `reviewer`

**Output:**
```
## What Changed
This PR introduces token-bucket rate limiting middleware applied to all public REST API
endpoints. The limits are configurable per environment via environment variables, enabling
different thresholds for development, staging, and production.

## Files Affected
- **Middleware** (`src/middleware/rate-limit.ts`): New rate-limiting middleware implementation.
- **App bootstrapping** (`src/app.ts`): Middleware registered in the Express application.
- **Tests** (`test/middleware/rate-limit.test.ts`): Unit tests for the new middleware.
- **Docs** (`docs/api.md`): API documentation updated to reflect rate-limit headers.

## Key Risks / Review Focus
- Ensure the default limits are appropriate for production traffic.
- Verify that rate-limit headers (`X-RateLimit-*`) conform to RFC 6585.
- Check that the middleware correctly identifies clients (IP vs. API key).
- Confirm environment variable names are documented and align with existing conventions.

## Testing Coverage
Unit tests cover normal operation and limit-exceeded scenarios. Integration tests against
a live server are not included — consider adding them if this is a high-traffic endpoint.
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | Tested 2024-01-01. Consistent structure across runs. |
| claude-3-5-sonnet | ✅ | Tested 2024-01-01. Slightly more verbose but well-structured. |
| copilot-gpt-4o | ✅ | Works well inline in IDE chat. |

## Changelog

### 1.0.0 — 2024-01-01
- Initial version.
