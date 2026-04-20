---
id: dev-pr-review-workflow
title: Pull Request Quality Gate Workflow
type: workflow
version: "1.0.0"
status: active
workstream:
  - developers
author: ai-skills-maintainer
created: 2026-04-20
updated: 2026-04-20
tags:
  - pull-request
  - code-review
  - quality-gate
  - workflow
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
  - copilot-gpt-4o
description: |
  A three-step workflow that takes a pull request from raw diff to a structured, actionable
  review: summarise the changes, run a code review, then assess test coverage — producing a
  consolidated quality-gate report ready to post as a PR comment.
security_classification: internal
delivery_modes:
  - copilot-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
dependencies:
  - dev-summarize-pull-request
  - dev-code-review-skill
inputs:
  - name: PR_TITLE
    type: string
    description: The pull request title
    required: true
  - name: PR_DESCRIPTION
    type: string
    description: The pull request body / description as written by the author
    required: true
  - name: CHANGED_FILES
    type: string
    description: List of changed file paths and their diffs
    required: true
  - name: LANGUAGE
    type: string
    description: Primary programming language of the changed code
    required: true
  - name: REVIEW_FOCUS
    type: string
    description: "Focus area: security | performance | readability | all"
    required: false
outputs:
  - name: QUALITY_GATE_REPORT
    type: string
    description: Full review report suitable for posting as a PR comment, including summary, findings, coverage assessment, and verdict
---

## Overview

This workflow orchestrates the complete first-pass review of a pull request. Human reviewers run this workflow before doing their own review, so they can focus on architecture and business logic instead of hunting for basic issues.

**Audience:** Developers opening PRs, automated CI pipelines, tech leads.

## Steps

### Step 1 — Summarise the Pull Request

**Asset used:** `dev-summarize-pull-request`  
**Input:** `{{PR_TITLE}}`, `{{PR_DESCRIPTION}}`, `{{CHANGED_FILES}}`  
**Output:** `SUMMARY` — a concise, plain-language paragraph describing the intent and scope of the changes.

```
[Prompt — paste into any LLM]

You are a technical writer reviewing a pull request.

PR Title: {{PR_TITLE}}
PR Description: {{PR_DESCRIPTION}}
Changed files: {{CHANGED_FILES}}

Produce a concise summary (3–5 sentences) that covers:
1. What change is being made and why.
2. Which areas of the codebase are affected.
3. Any notable trade-offs or risks the reviewer should be aware of.

Format: plain paragraph. No headings.
```

---

### Step 2 — Code Review

**Asset used:** `dev-code-review-skill`  
**Input:** `CHANGED_FILES` (diffs from Step 1 input), `{{LANGUAGE}}`, `{{REVIEW_FOCUS}}`  
**Output:** `FINDINGS` — findings categorised as Critical / Major / Minor / Positive.

```
[Prompt — paste into any LLM after Step 1]

You are a senior {{LANGUAGE}} engineer doing a code review.

Review the following diffs for:
1. Correctness — logic errors, null handling, off-by-one errors
2. Security — OWASP Top 10, injection risks, secrets exposure
3. Performance — N+1 queries, blocking calls, unnecessary allocations
4. Maintainability — naming, complexity, duplication, missing error handling

Focus area: {{REVIEW_FOCUS}}

Format your response as:

### Critical (must fix before merge)
- [FILE:LINE] Issue. Suggested fix: ...

### Major (should fix)
- [FILE:LINE] Issue. Suggested fix: ...

### Minor (consider fixing)
- [FILE:LINE] Issue.

### Positive observations
- What the code does well.

Diffs to review:
{{CHANGED_FILES}}
```

---

### Step 3 — Test Coverage Assessment

**Asset used:** *(inline prompt — no separate skill needed)*  
**Input:** `CHANGED_FILES` (to identify which test files are present or absent)  
**Output:** `COVERAGE_ASSESSMENT` — one paragraph on test adequacy.

```
[Prompt — paste into any LLM after Step 2]

Review the changed files list below. Identify:
1. Whether test files exist for the changed source files.
2. Whether new public functions or API endpoints introduced by the PR have corresponding tests.
3. Whether any edge cases appear untested based on the code changes.

Respond in a single paragraph. If coverage is adequate, say so clearly.
If tests are missing or insufficient, name the specific gaps.

Changed files: {{CHANGED_FILES}}
```

---

### Step 4 — Consolidate Report

**Asset used:** *(inline prompt)*  
**Input:** `SUMMARY` (Step 1), `FINDINGS` (Step 2), `COVERAGE_ASSESSMENT` (Step 3)  
**Output:** `QUALITY_GATE_REPORT`

```
[Prompt — paste into any LLM after Steps 1–3]

Combine the following three review outputs into a single, well-formatted PR review comment.

Summary:
{{SUMMARY}}

Code review findings:
{{FINDINGS}}

Test coverage assessment:
{{COVERAGE_ASSESSMENT}}

Add a final "## Verdict" section with one of:
- ✅ Approve — if no Critical or Major findings and coverage is adequate.
- 🔁 Request changes — if any Critical or Major finding exists.
- 💬 Comment — if observations only (no blocking issues, but worth noting).

Include a one-sentence rationale for the verdict.
```

---

## Flow Diagram

```
PR_TITLE + PR_DESCRIPTION + CHANGED_FILES + LANGUAGE + REVIEW_FOCUS
         │
         ▼
[Step 1: Summarise PR] ──▶ SUMMARY
         │
         ▼
[Step 2: Code Review] ──▶ FINDINGS
         │
         ▼
[Step 3: Coverage Assessment] ──▶ COVERAGE_ASSESSMENT
         │
         ▼
[Step 4: Consolidate Report] ──▶ QUALITY_GATE_REPORT
```

## Usage

### Manual (Copilot Chat — current)

1. Open the PR diff in your IDE or browser.
2. Copy each step prompt in order, substitute the placeholders, and paste into GitHub Copilot Chat or your preferred LLM.
3. Save each step's output to use as input for the next.
4. Post the final `QUALITY_GATE_REPORT` as a PR review comment.

### Automated (GitHub Actions — current)

See `workstreams/developers/automations/pr-validator.md` for a ready-to-use GitHub Actions implementation of this workflow.

### API Endpoint (future portal)

```http
POST {{API_BASE_URL}}/packages/developers-core-pack/invoke
Content-Type: application/json

{
  "workflow_id": "dev-pr-review-workflow",
  "inputs": {
    "pr_title": "feat: add JWT auth to /api/users",
    "pr_description": "Adds RS256 JWT verification middleware...",
    "changed_files": "src/middleware/auth.ts\n...",
    "language": "TypeScript",
    "review_focus": "security"
  }
}
```

## Examples

### Example 1 — Security-focused review of a small feature PR

**Input:**
- `PR_TITLE`: `feat: add rate limiting to POST /api/orders`
- `PR_DESCRIPTION`: `Adds token-bucket rate limiting (100 req/min) to prevent abuse.`
- `CHANGED_FILES`: *(3 files: middleware, route, and test)*
- `LANGUAGE`: `TypeScript`
- `REVIEW_FOCUS`: `security`

**Step 1 Output (SUMMARY):**
```
This PR introduces a token-bucket rate limiter that caps POST /api/orders at 100 requests
per minute per client IP. The middleware is applied globally at the Express router level
and falls back to a 429 response on limit breach. Risk: IP-based rate limiting can be
bypassed with rotating IPs — consider user-session-based limiting for authenticated routes.
```

**Step 2 Output (FINDINGS excerpt):**
```
### Critical (must fix before merge)
- [middleware/rate-limit.ts:22] Rate limit key uses raw IP without normalisation — IPv6
  mapped IPv4 addresses (::ffff:1.2.3.4) may bypass the limiter.
  Suggested fix: Normalise with net.normalizeIP() before using as key.

### Positive observations
- Window reset logic is clean and well-commented.
```

**Final Output (QUALITY_GATE_REPORT verdict):**
```
## Verdict
🔁 Request changes — one Critical security finding must be resolved before merge.
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | 2026-04-20. All four steps work in a single Copilot Chat session. |
| claude-3-5-sonnet | ✅ | 2026-04-20. Step 2 findings are more detailed on security than gpt-4o. |

## Changelog

### 1.0.0 — 2026-04-20
- Initial version.
