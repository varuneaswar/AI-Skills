---
id: dev-code-review-skill
title: Structured Code Review
type: skill
version: "1.0.0"
status: active
workstream:
  - developers
author: ai-skills-maintainer
created: 2024-01-01
updated: 2024-01-01
tags:
  - code-review
  - quality
  - security
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
  - copilot-gpt-4o
description: |
  Reviews a code snippet or file for correctness, maintainability, security issues,
  and adherence to best practices, returning findings categorised by severity.
security_classification: internal
inputs:
  - name: CODE
    type: string
    description: The code snippet or file content to review
    required: true
  - name: LANGUAGE
    type: string
    description: Programming language (e.g., Python, TypeScript, Java)
    required: true
  - name: CONTEXT
    type: string
    description: Brief description of what the code is meant to do
    required: false
  - name: FOCUS
    type: string
    description: "Optional focus area: security, performance, readability, or all (default)"
    required: false
outputs:
  - name: REVIEW_REPORT
    type: string
    description: Structured review with findings grouped by severity
delivery_modes:
  - copilot-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
---

## Overview

Performs a thorough code review without requiring a human reviewer as a first pass. Best used to catch obvious issues before requesting peer review, or to provide review feedback on small PRs automatically.

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `{{CODE}}` | string | Yes | Code to review |
| `{{LANGUAGE}}` | string | Yes | Programming language |
| `{{CONTEXT}}` | string | No | What the code is intended to do |
| `{{FOCUS}}` | string | No | `security`, `performance`, `readability`, or `all` |

## Outputs

| Name | Type | Description |
|---|---|---|
| `REVIEW_REPORT` | string | Findings categorised by severity |

## Skill Definition

```
[System Prompt]

You are a senior software engineer conducting a professional code review.

Review the provided {{LANGUAGE}} code for:
1. Correctness — logic errors, off-by-one errors, null/undefined handling
2. Security — injection risks, insecure defaults, exposed secrets, OWASP Top 10
3. Performance — unnecessary loops, missing indexes, blocking calls
4. Maintainability — naming, complexity, duplication, missing tests
5. Best practices — idiomatic patterns, error handling, logging

Focus area: {{FOCUS}} (if 'all', cover all categories above)
Code purpose: {{CONTEXT}}

Format your response as:

## Summary
One-line verdict (e.g., "Minor issues found — safe to merge with fixes")

## Critical (must fix before merge)
- [LINE X] Issue description. Suggested fix: ...

## Major (should fix)
- [LINE X] Issue description. Suggested fix: ...

## Minor (consider fixing)
- [LINE X] Issue description. Suggested fix: ...

## Positive observations
- What the code does well.

Code to review:
{{CODE}}
```

## Usage

### GitHub Copilot Chat (current)

```
@workspace #file:<filename>
Review this {{LANGUAGE}} code. Focus on {{FOCUS}}.
Context: {{CONTEXT}}
```

### Standalone (any LLM — current)

1. Copy the skill definition above.
2. Replace all placeholders.
3. Paste into your LLM interface.

### API Endpoint (future portal)

```http
POST {{API_BASE_URL}}/packages/developers-core-pack/invoke
Content-Type: application/json

{
  "skill_id": "dev-code-review-skill",
  "inputs": {
    "code": "...",
    "language": "TypeScript",
    "context": "Express middleware for rate limiting",
    "focus": "security"
  }
}
```

### MCP Tool (future)

```json
{
  "tool": "dev-code-review-skill",
  "parameters": {
    "code": "...",
    "language": "TypeScript",
    "context": "Express middleware",
    "focus": "security"
  }
}
```

## Examples

### Example 1

**Input:**
- `CODE`: *(a Node.js function that queries a database using string concatenation)*
- `LANGUAGE`: `JavaScript`
- `FOCUS`: `security`

**Output:**
```
## Summary
Critical security issue found — do not merge without fix.

## Critical (must fix before merge)
- [LINE 12] SQL query built by string concatenation — SQL injection risk.
  Suggested fix: Use parameterised queries: db.query('SELECT * FROM users WHERE id = $1', [userId])

## Major (should fix)
- [LINE 5] Missing error handling for database connection failure.
  Suggested fix: Wrap in try/catch and return a meaningful error response.

## Positive observations
- Function is small and single-purpose.
- Variable names are clear and descriptive.
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | 2024-01-01. Line numbers accurate. Security findings reliable. |
| claude-3-5-sonnet | ✅ | 2024-01-01. Particularly thorough on security. |

## Changelog

### 1.0.0 — 2024-01-01
- Initial version.
