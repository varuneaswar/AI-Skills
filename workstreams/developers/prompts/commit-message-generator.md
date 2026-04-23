---
id: dev-commit-message-generator
title: Conventional Commit Message Generator
type: prompt
version: "1.0.0"
status: active
workstream:
  - developers
author: ai-skills-maintainer
created: 2026-04-20
updated: 2026-04-20
tags:
  - git
  - commit-message
  - conventional-commits
  - documentation
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
  - gpt-4-turbo
description: |
  Generates a well-formed Conventional Commits message (type, scope, subject, body, footer)
  from a diff or a plain-language description of the change — keeping git history readable
  and automating changelog generation.
security_classification: internal
delivery_modes:
  - llm-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
inputs:
  - name: CHANGE_DESCRIPTION
    type: string
    description: Plain-language description of what was changed, or the raw git diff
    required: true
  - name: SCOPE
    type: string
    description: "Optional: module or component name (e.g., auth, api, ui). Leave blank to auto-detect."
    required: false
  - name: BREAKING
    type: string
    description: "yes | no — whether this is a breaking change (default: no)"
    required: false
---

## Overview

Poorly written commit messages make git history hard to read and break automated changelog tools. This prompt generates a complete Conventional Commits message from a diff or plain-English description.

Supports all standard Conventional Commits types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`, `perf`, `ci`, `build`.

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `{{CHANGE_DESCRIPTION}}` | string | Yes | Git diff or plain-language description of the change |
| `{{SCOPE}}` | string | No | Component or module name |
| `{{BREAKING}}` | string | No | `yes` or `no` (default: `no`) |

## Prompt

```
[System]
You are a git commit message specialist who follows the Conventional Commits specification
(https://www.conventionalcommits.org/).

[Task]
Generate a complete Conventional Commits message for the following change.

Change description or diff:
{{CHANGE_DESCRIPTION}}

Scope (if known): {{SCOPE}}
Breaking change: {{BREAKING}}

[Rules]
1. Choose the correct type: feat | fix | docs | style | refactor | test | chore | perf | ci | build
2. The subject line must be ≤ 72 characters, imperative mood, no period at the end.
3. Write a body (wrapped at 72 chars) if the change is non-trivial — explain WHY, not just WHAT.
4. If breaking=yes, add "BREAKING CHANGE: <description>" as a footer.
5. Reference issue/ticket numbers in the footer as "Closes #<number>" if mentioned.

[Output Format]
<type>(<scope>): <subject>

<body — omit if the subject is self-explanatory>

<footer — omit if no breaking change or issue reference>
```

## Usage

### LLM Chat (current)

Stage your changes, then:

```
@workspace
Generate a Conventional Commits message for the staged changes.
CHANGE_DESCRIPTION: <paste your git diff or describe the change>
SCOPE: <component name>
BREAKING: no
```

Or use the Copilot Chat sidebar after running `git diff --staged`:

```
#terminalSelection
Write a Conventional Commits message for this diff.
```

### Standalone (any LLM — current)

1. Run `git diff --staged` to get your diff.
2. Copy the prompt above, replace all placeholders with your values.
3. Paste into your preferred LLM interface.

### API Endpoint (future portal)

```http
POST {{API_BASE_URL}}/packages/developers-core-pack/invoke
Content-Type: application/json

{
  "prompt_id": "dev-commit-message-generator",
  "inputs": {
    "change_description": "Added null check before accessing user.profile.email in profile controller",
    "scope": "auth",
    "breaking": "no"
  }
}
```

## Examples

### Example 1 — Bug fix, no breaking change

**Input:**
- `CHANGE_DESCRIPTION`: `Added null check before accessing user.profile.email in profile controller to prevent NullPointerException when profile is not yet set up`
- `SCOPE`: `auth`
- `BREAKING`: `no`

**Output:**
```
fix(auth): guard against null profile in email access

Profile may be null for users who registered via SSO before the profile
setup step was introduced. Accessing user.profile.email without a null
check caused a 500 on GET /api/me for those accounts.

Closes #417
```

### Example 2 — Breaking API change

**Input:**
- `CHANGE_DESCRIPTION`: `Renamed the /api/v1/users endpoint to /api/v2/members and changed the response shape to include a nested "account" object`
- `SCOPE`: `api`
- `BREAKING`: `yes`

**Output:**
```
feat(api)!: rename /v1/users to /v2/members with new response shape

The users endpoint has been renamed to members to better reflect the
domain model. The response now wraps user data inside an "account"
object to allow future extension without further breaking changes.

BREAKING CHANGE: /api/v1/users is removed. Clients must migrate to
/api/v2/members. Response shape has changed — see docs/api-migration.md.
```

### Example 3 — Documentation update

**Input:**
- `CHANGE_DESCRIPTION`: `Updated README to add architecture diagram and contribution guide link`
- `SCOPE`: *(left blank)*
- `BREAKING`: `no`

**Output:**
```
docs: add architecture diagram and contribution guide link to README
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | 2026-04-20. Excellent subject line quality. Rarely needs editing. |
| claude-3-5-sonnet | ✅ | 2026-04-20. More verbose body paragraphs — good for complex changes. |
| copilot-gpt-4o | ✅ | 2026-04-20. Works well in Copilot Chat with `#terminalSelection`. |

## Changelog

### 1.0.0 — 2026-04-20
- Initial version.
