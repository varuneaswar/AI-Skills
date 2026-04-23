---
id: test-generation-workflow
title: Test Generation Workflow
type: workflow
version: "1.0.0"
status: active
workstream:
  - testers
author: ai-skills-maintainer
created: 2026-04-20
updated: 2026-04-20
tags:
  - testing
  - bdd
  - gherkin
  - workflow
  - test-generation
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
description: |
  A four-step workflow that transforms a user story into a full test suite: extract
  testable behaviours, generate test cases, write BDD Gherkin scenarios, and identify
  coverage gaps — producing a ready-to-use test suite with minimal manual effort.
security_classification: internal
delivery_modes:
  - llm-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
dependencies:
  - test-coverage-gap-analysis
  - test-generate-test-cases-from-user-story
inputs:
  - name: USER_STORY
    type: string
    description: The full user story text with acceptance criteria
    required: true
  - name: EXISTING_TESTS
    type: string
    description: Description or list of existing test cases (pass empty string if none)
    required: false
  - name: FRAMEWORK
    type: string
    description: "Target test framework: jest | pytest | junit | cucumber (default: cucumber)"
    required: false
outputs:
  - name: FULL_TEST_SUITE
    type: string
    description: Complete test suite including test plan, BDD scenarios, and coverage gap report
---

## Overview

This workflow guides a tester through the complete test design process for a single user story. Each step produces a concrete, reusable output that feeds into the next step.

**Audience:** QA engineers, SDETs, product owners validating acceptance criteria.

## Steps

### Step 1 — Extract Testable Behaviours

**Asset used:** `test-generate-test-cases-from-user-story`  
**Input:** `{{USER_STORY}}`  
**Output:** `BEHAVIOURS` — a numbered list of distinct, testable behaviours derived from the story and acceptance criteria.

```
[Prompt — paste into any LLM]

You are a senior QA engineer skilled at decomposing user stories.

User Story:
{{USER_STORY}}

Extract every distinct testable behaviour from the user story and its acceptance criteria.
For each behaviour:
1. Give it a short identifier (B-001, B-002, …).
2. Write a one-sentence description of what must be true.
3. Label it as: Core | Edge | Constraint | Security.

Format:
**B-NNN: [Short title]**
- Type: Core / Edge / Constraint / Security
- Description: What the system must do or not do.

Output only the behaviour list. No commentary.
```

---

### Step 2 — Generate Test Cases

**Asset used:** `test-generate-test-cases-from-user-story`  
**Input:** `BEHAVIOURS` (from Step 1), `{{FRAMEWORK}}`  
**Output:** `TEST_CASES` — a complete test case list covering happy path, edge cases, and negative scenarios.

```
[Prompt — paste into any LLM after Step 1]

You are a senior QA engineer generating test cases from a behaviour list.

Behaviours:
{{BEHAVIOURS}}

Target framework: {{FRAMEWORK}}

Generate test cases covering:
1. Happy path — all behaviours pass under normal conditions.
2. Edge cases — boundary values, empty inputs, maximum lengths, concurrent access.
3. Negative scenarios — invalid inputs, missing required fields, unauthorized access.
4. Security scenarios — if any behaviour involves authentication or data access.

For each test case:

**TC-NNN: [Short descriptive title]**
- Behaviour covered: B-NNN
- Priority: High / Medium / Low
- Type: Functional | Edge Case | Negative | Security
- Precondition: [system state before the test]
- Action: [what the user or system does]
- Expected result: [what must be true afterwards]

Aim for a minimum of 2 happy-path, 3 edge-case, and 2 negative test cases.
```

---

### Step 3 — Write BDD Scenarios (Gherkin)

**Asset used:** `test-generate-test-cases-from-user-story`  
**Input:** `TEST_CASES` (from Step 2), `{{FRAMEWORK}}`  
**Output:** `BDD_SCENARIOS` — Gherkin scenarios formatted for the target framework.

```
[Prompt — paste into any LLM after Step 2]

You are a BDD specialist converting test cases into Gherkin scenarios.

Test cases:
{{TEST_CASES}}

Target framework: {{FRAMEWORK}}

Convert every test case into a Gherkin scenario using Given/When/Then.
- Use Scenario Outline + Examples table for parameterised cases (boundary values).
- Add a Feature header grouping all scenarios.
- Preserve TC-NNN identifiers as comments above each scenario.
- Use synthetic data only — no real email addresses, names, or PII.

Output only the .feature file content.
```

---

### Step 4 — Identify Coverage Gaps

**Asset used:** `test-coverage-gap-analysis`  
**Input:** `TEST_CASES` (Step 2), `{{EXISTING_TESTS}}`  
**Output:** `COVERAGE_GAP_REPORT` — prioritised list of gaps in the existing test suite.

```
[Prompt — paste into any LLM after Steps 1–3]

You are a QA coverage analyst.

Generated test cases (complete target suite):
{{TEST_CASES}}

Existing test suite:
{{EXISTING_TESTS}}

Compare the two lists. For each generated test case that has no equivalent in the existing
suite, report it as a coverage gap.

Format each gap as:

**GAP-NNN: [Short title]**
- Risk: High / Medium / Low
- Behaviour covered: B-NNN
- What is missing: One sentence describing the untested scenario.
- Suggested Gherkin scenario: Given / When / Then

Sort gaps by Risk descending. After the gap list, add a brief Coverage Summary paragraph.
```

---

## Flow Diagram

```
USER_STORY + EXISTING_TESTS + FRAMEWORK
         │
         ▼
[Step 1: Extract Behaviours] ──▶ BEHAVIOURS
         │
         ▼
[Step 2: Generate Test Cases] ──▶ TEST_CASES
         │
         ▼
[Step 3: Write BDD Scenarios] ──▶ BDD_SCENARIOS
         │
         ▼
[Step 4: Coverage Gap Analysis] ──▶ COVERAGE_GAP_REPORT
         │
         ▼
      FULL_TEST_SUITE
```

## Usage

### Manual (Copilot Chat — current)

1. Open your user story document in your IDE.
2. Copy each step prompt in order, substitute the placeholders, and paste into LLM Chat or your preferred LLM.
3. Save each step's output to use as input for the next step.
4. Combine all outputs into a single test suite document for your test management tool.

### Automated (Bitbucket Pipelines — current)

See `workstreams/testers/automations/test-coverage-automation.md` for a ready-to-use Bitbucket Pipelines implementation of this workflow.

### API Endpoint (future portal)

```http
POST {{API_BASE_URL}}/packages/testers-quality-pack/invoke
Content-Type: application/json

{
  "workflow_id": "test-generation-workflow",
  "inputs": {
    "user_story": "As a registered user, I want to reset my password via email...",
    "existing_tests": "TC-001: Successful reset with valid email",
    "framework": "cucumber"
  }
}
```

## Examples

### Example 1 — Password reset story, Cucumber framework

**Input:**
- `USER_STORY`: `As a registered user, I want to reset my password via email, so that I can regain access to my account. AC: 1. User enters registered email. 2. System sends reset link valid for 1 hour. 3. Password must be 8+ chars with 1 uppercase and 1 number. 4. Old password is invalidated after reset.`
- `EXISTING_TESTS`: `TC-001: Successful password reset flow`
- `FRAMEWORK`: `cucumber`

**Step 1 Output (BEHAVIOURS excerpt):**
```
**B-001: Reset email delivery**
- Type: Core
- Description: The system must send a password reset email when a registered user submits their email address.

**B-002: Reset link expiry**
- Type: Constraint
- Description: The reset link must become invalid after 60 minutes.

**B-003: Password complexity**
- Type: Constraint
- Description: The new password must be at least 8 characters and include one uppercase letter and one number.

**B-004: Old password invalidation**
- Type: Core
- Description: After a successful reset, the user's previous password must no longer be accepted.
```

**Step 3 Output (BDD_SCENARIOS excerpt):**
```gherkin
Feature: Password Reset

  # TC-002
  Scenario: Reset link rejected after expiry
    Given a registered user "user@example.com" requested a password reset 61 minutes ago
    When the user submits the reset form using the expired token
    Then the system returns HTTP 400
    And the response message is "This reset link has expired"

  # TC-005 — boundary value
  Scenario Outline: Password complexity boundary
    Given a valid reset token for "user@example.com"
    When the user submits a new password "<password>"
    Then the system responds with "<outcome>"
    Examples:
      | password  | outcome  |
      | Pass1234  | success  |
      | Pass123   | rejected |
      | pass1234  | rejected |
```

**Step 4 Output (COVERAGE_GAP_REPORT excerpt):**
```
**GAP-001: Reset link reuse (replay attack)**
- Risk: High
- Behaviour covered: B-002
- What is missing: No test verifies that a reset link cannot be reused after a successful reset.
- Suggested scenario:
    Given a reset link for "user@example.com" was successfully used
    When the user attempts to use the same reset link again
    Then the system returns HTTP 400 with "Link already used"
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | 2026-04-20. All four steps work end-to-end in a single Copilot Chat session. |
| claude-3-5-sonnet | ✅ | 2026-04-20. Step 3 Gherkin output is cleaner; Step 4 finds more security gaps. |

## Changelog

### 1.0.0 — 2026-04-20
- Initial version.
