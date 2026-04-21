---
id: dev-pr-review-agent
title: Pull Request Review Agent
type: agent
version: "1.0.0"
status: active
workstream:
  - developers
author: ai-skills-maintainer
created: 2026-04-20
updated: 2026-04-20
tags:
  - code-review
  - pull-request
  - automation
  - quality
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
  - copilot-gpt-4o
description: |
  An autonomous agent that reviews a pull request end-to-end: summarises changes, runs a
  structured code review, checks test coverage adequacy, and posts a consolidated review
  comment — reducing reviewer load while raising baseline quality.
security_classification: internal
delivery_modes:
  - copilot-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
dependencies:
  - dev-code-review-skill
  - dev-summarize-pull-request
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
    description: List of changed files with diffs (or file paths when using IDE mode)
    required: true
  - name: LANGUAGE
    type: string
    description: Primary programming language of the changed code
    required: false
  - name: REVIEW_FOCUS
    type: string
    description: "Optional focus area: security, performance, readability, or all (default: all)"
    required: false
outputs:
  - name: REVIEW_SUMMARY
    type: string
    description: One-paragraph executive summary of the PR
  - name: CODE_REVIEW_FINDINGS
    type: string
    description: Structured list of findings categorised by severity
  - name: COVERAGE_ASSESSMENT
    type: string
    description: Assessment of whether the PR includes adequate tests
  - name: OVERALL_VERDICT
    type: string
    description: "Approve / Request changes / Comment — with rationale"
---

## Overview

The PR Review Agent replaces the first-pass human review step for routine pull requests. It sequentially invokes the summarisation skill and the code review skill, then synthesises their outputs into a single, actionable review comment.

Use it as a first reviewer so that human reviewers focus on design and business-logic concerns rather than catching basic issues.

## Architecture

**Skills / Tools Used:**

| Tool / Skill | Purpose |
|---|---|
| `dev-summarize-pull-request` | Produce a plain-language summary of changes |
| `dev-code-review-skill` | Run structured code review on changed files |

**Decision Flow:**

```
PR_TITLE + PR_DESCRIPTION + CHANGED_FILES
         │
         ▼
Step 1: Summarise PR (dev-summarize-pull-request)
         │
         ▼
Step 2: Code Review  (dev-code-review-skill)
         │
         ├─ Critical findings ──▶ VERDICT = "Request changes"
         │
         ├─ Major findings only ──▶ VERDICT = "Request changes" (with explanation)
         │
         └─ Minor / no findings ──▶ VERDICT = "Approve" (with notes)
                                          │
                                          ▼
                              Step 3: Coverage Assessment
                                          │
                                          ▼
                              REVIEW_SUMMARY + FINDINGS + COVERAGE_ASSESSMENT + VERDICT
```

## System Prompt / Agent Instructions

```
[System Prompt]

You are an autonomous pull request review agent for a software engineering team.

Your goals:
1. Summarise the pull request so reviewers quickly understand the intent.
2. Review all changed code files for correctness, security, performance, and maintainability.
3. Assess whether the PR includes adequate test coverage for the changes made.
4. Produce a clear, constructive review verdict with actionable feedback.

Tools available to you:
- summarize_pr: call with (title, description, changed_files) → returns REVIEW_SUMMARY
- code_review:  call with (code, language, context, focus) → returns CODE_REVIEW_FINDINGS

Rules:
- Never approve a PR with Critical severity findings.
- Always explain your verdict before listing findings.
- If test files are absent for new functionality, flag this in COVERAGE_ASSESSMENT.
- Keep tone professional and constructive — focus on the code, not the author.
- If uncertain about intent, ask a clarifying question rather than making assumptions.
- Do not flag style issues as Critical or Major — they are always Minor.

Output format:

## PR Summary
{REVIEW_SUMMARY}

## Code Review Findings
{CODE_REVIEW_FINDINGS}

## Test Coverage Assessment
{COVERAGE_ASSESSMENT}

## Verdict
{OVERALL_VERDICT}
```

## Configuration

```yaml
agent_id: dev-pr-review-agent
model: gpt-4o
temperature: 0.1         # low temperature for consistent, deterministic review output
max_iterations: 5        # summarise → review → coverage → synthesise → format
tools:
  - summarize_pr
  - code_review
```

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `{{PR_TITLE}}` | string | Yes | Pull request title |
| `{{PR_DESCRIPTION}}` | string | Yes | Pull request description |
| `{{CHANGED_FILES}}` | string | Yes | Changed files with diffs |
| `{{LANGUAGE}}` | string | No | Primary programming language |
| `{{REVIEW_FOCUS}}` | string | No | `security`, `performance`, `readability`, or `all` |

## Outputs

| Name | Type | Description |
|---|---|---|
| `REVIEW_SUMMARY` | string | Plain-language summary of what the PR does |
| `CODE_REVIEW_FINDINGS` | string | Findings by severity |
| `COVERAGE_ASSESSMENT` | string | Test coverage adequacy note |
| `OVERALL_VERDICT` | string | Approve / Request changes / Comment |

## Security Considerations

- The agent reads code but never writes or commits to the repository.
- No code or diff content is sent to external services — route through `in-house-llm` for confidential codebases.
- The agent must never reveal secrets it encounters in the diff; it should flag them as Critical findings instead.

## Usage

### GitHub Copilot Chat (current)

Open the PR diff in your IDE, then:

```
@workspace #file:<changed-file>
You are a PR review agent. Review this {{LANGUAGE}} code change.
PR Title: {{PR_TITLE}}
PR Description: {{PR_DESCRIPTION}}
Focus: {{REVIEW_FOCUS}}
```

### API Endpoint (future portal)

```http
POST {{API_BASE_URL}}/packages/developers-core-pack/invoke
Content-Type: application/json

{
  "agent_id": "dev-pr-review-agent",
  "inputs": {
    "pr_title": "feat: add rate limiting middleware",
    "pr_description": "Adds token-bucket rate limiting to the API gateway...",
    "changed_files": "src/middleware/rate-limit.ts\n...",
    "language": "TypeScript",
    "review_focus": "security"
  }
}
```

### MCP Tool (future)

```json
{
  "tool": "dev-pr-review-agent",
  "parameters": {
    "pr_title": "feat: add rate limiting middleware",
    "pr_description": "...",
    "changed_files": "...",
    "language": "TypeScript",
    "review_focus": "security"
  }
}
```

## Examples

### Example 1 — Security-focused review

**Input:**
- `PR_TITLE`: `feat: add JWT authentication to /api/users endpoint`
- `PR_DESCRIPTION`: `Adds JWT verification middleware. Tokens are validated against our public key.`
- `CHANGED_FILES`: *(diff of auth middleware and route handler)*
- `LANGUAGE`: `TypeScript`
- `REVIEW_FOCUS`: `security`

**Output:**

```
## PR Summary
This PR adds JWT-based authentication to the /api/users endpoint. The change introduces
a new middleware that verifies RS256-signed tokens using the application's public key,
then attaches the decoded claims to the request context.

## Code Review Findings

### Critical (must fix before merge)
- [middleware/auth.ts LINE 18] Token expiry (`exp` claim) is not validated.
  Suggested fix: Add `if (Date.now() >= decoded.exp * 1000) return res.status(401).json(...)`.

### Minor (consider fixing)
- [middleware/auth.ts LINE 5] Hardcoded algorithm `'RS256'` should come from config to allow rotation.

## Test Coverage Assessment
JWT middleware tests are present (auth.test.ts). Edge cases for expired and malformed
tokens appear to be missing — recommend adding at minimum two additional test cases.

## Verdict
**Request changes** — one Critical security finding must be resolved before merge.
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | 2026-04-20. Verdict accuracy high. Occasionally over-flags style as Major. |
| claude-3-5-sonnet | ✅ | 2026-04-20. More thorough security analysis than gpt-4o on this task. |

## Changelog

### 1.0.0 — 2026-04-20
- Initial version.
