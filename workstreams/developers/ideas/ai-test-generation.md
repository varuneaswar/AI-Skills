---
id: dev-ai-test-generation
title: AI-Assisted Unit Test Generation
type: idea
version: "1.0.0"
status: draft
workstream:
  - developers
author: ai-skills-maintainer
created: 2026-04-20
updated: 2026-04-20
tags:
  - testing
  - unit-test
  - automation
  - skill
  - developer-productivity
description: |
  A skill that generates a comprehensive unit test suite for any function or class from its
  source code — covering happy paths, edge cases, and negative scenarios — eliminating the
  most repetitive part of test authoring while maintaining test quality.
security_classification: internal
---

## Overview

Writing unit tests is time-consuming, repetitive, and often skipped under deadline pressure. This idea proposes a skill that reads a function or class definition and produces a ready-to-run test suite with full coverage of the public contract.

**Target users:** Developers working in any language with a unit testing framework.

## Proposed Approach

**Asset type:** `skill` (single-purpose, callable unit)  
**Potential delivery modes:**
- `copilot-chat` — immediately available; developer pastes function, receives test file
- `api-endpoint` — future: integrated into CI pipeline to auto-generate tests on new functions
- `mcp-tool` — future: IDE extension that auto-suggests tests as functions are written

**Implementation sketch:**

1. Accept `SOURCE_CODE` (function or class) + `LANGUAGE` + `FRAMEWORK` as inputs.
2. Analyse the function signature, return types, and branching paths.
3. Generate test cases for:
   - Happy paths (typical valid inputs)
   - Boundary values (empty string, zero, max length, etc.)
   - Negative cases (invalid types, null, exception scenarios)
   - Side effects (database calls, external API calls — stub/mock guidance included)
4. Return a complete test file in the target framework format.

**LLMs to target:** gpt-4o, claude-3-5-sonnet (both have strong code generation capabilities).

## Expected Benefits

- Reduces time to write initial test suite from 30–60 minutes to under 2 minutes.
- Increases test coverage baselines across the codebase without adding sprint capacity.
- Provides a teachable artifact — junior engineers can learn from generated test patterns.
- Catches missing edge cases that developers commonly overlook.
- Reduces the "cost" of the PR review quality gate (fewer untested functions flagged).

## Acceptance Criteria

- [ ] Skill generates runnable tests (syntax-correct) for Python (pytest), TypeScript (Jest), and Java (JUnit 5).
- [ ] Generated test suite covers at least 3 happy path, 3 boundary, and 2 negative scenarios per function.
- [ ] Skill correctly identifies and generates mock/stub guidance for external dependencies.
- [ ] Output passes the target framework's linter without modification.
- [ ] Tested against at least 10 real functions from the codebase.
- [ ] Added to `catalog/index.json` and cross-referenced in `developers-core-pack`.

## Open Questions

- Should the skill also generate integration tests, or only unit tests? (Scope creep risk)
- How to handle functions with no clear return value (void/None)?
- Which mocking libraries to assume by default for each language?
- Should the skill accept existing tests as input to avoid duplicates?

## References

- Related skill: `dev-code-review-skill` — can validate the generated tests in a follow-up step
- Related workflow: `dev-pr-review-workflow` — coverage assessment step would benefit from this
- Related automation: `dev-pr-validator` — could invoke this skill on untested functions in the diff
- Template to follow: `templates/skill-template.md`
