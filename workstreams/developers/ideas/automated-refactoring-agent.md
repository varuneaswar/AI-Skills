---
id: dev-automated-refactoring-agent
title: Automated Code Refactoring Agent
type: idea
version: "1.0.0"
status: draft
workstream:
  - developers
author: ai-skills-maintainer
created: 2024-01-01
updated: 2024-01-01
tags:
  - refactoring
  - technical-debt
  - agent
  - automation
description: |
  An autonomous agent that continuously scans a codebase for technical debt, proposes and
  applies targeted refactors, and opens pull requests for human review — dramatically reducing
  the manual effort of maintaining code quality.
security_classification: internal
---

## Overview

Developer teams accumulate technical debt faster than they can address it manually. This idea proposes an agent that autonomously identifies and refactors code smells, outdated patterns, and maintainability issues — then raises PRs for human review rather than merging blindly.

## Proposed Approach

**Asset type:** `agent`  
**Potential delivery mode:** `api-endpoint` (scheduled or triggered) → future portal integration

The agent would:
1. Use the `dev-code-review-skill` to scan each changed or flagged file.
2. Classify issues by refactor category (rename, extract-method, simplify-condition, remove-dead-code).
3. Apply refactors using a code-transformation tool (e.g., a language server or AST manipulator).
4. Raise a PR with a clear explanation of each change.
5. Skip any file flagged as `confidential` in the asset metadata.

## Expected Benefits

- Reduces technical debt backlog without consuming developer sprint capacity.
- Consistent refactoring style across the codebase.
- Every change is reviewable before merge.
- Provides a learning resource for junior engineers reviewing the PRs.

## Acceptance Criteria

- [ ] Agent successfully identifies at least 5 distinct refactor categories.
- [ ] All generated PRs pass existing CI/CD checks before being raised.
- [ ] Agent never modifies files not explicitly in scope.
- [ ] PR descriptions clearly explain what was changed and why.
- [ ] Opt-out mechanism available per file/directory via a config file.

## Open Questions

- Which language(s) to support first? (JavaScript/TypeScript, Python, Java)
- How to handle merge conflicts if the base branch changes during refactoring?
- Should the agent learn from accepted/rejected PR history to improve suggestions?

## References

- Existing skill: `dev-code-review-skill` — can be reused as the analysis step
- Existing prompt: `dev-summarize-pull-request` — for generating PR descriptions
