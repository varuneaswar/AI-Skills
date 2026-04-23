---
id: test-generation-agent
title: Test Generation Agent
type: agent
version: "1.0.0"
status: active
workstream:
  - testers
author: ai-skills-maintainer
created: 2026-04-20
updated: 2026-04-20
tags:
  - testing
  - test-generation
  - bdd
  - gherkin
  - coverage
  - agent
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
description: |
  An autonomous agent that takes a user story, generates a comprehensive test plan
  (functional, edge case, negative), produces BDD scenarios in Gherkin (Given/When/Then),
  and flags coverage gaps against an existing test suite — accelerating test design from
  hours to minutes.
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
    description: The full user story text (As a… I want… So that…) including acceptance criteria
    required: true
  - name: EXISTING_TESTS
    type: string
    description: Description or list of existing test cases for gap analysis (pass empty string if none)
    required: false
  - name: FRAMEWORK
    type: string
    description: "Target test framework: jest | pytest | junit | cucumber (default: cucumber)"
    required: false
  - name: FOCUS_AREAS
    type: string
    description: "Optional comma-separated areas to focus on: security, performance, boundary, negative"
    required: false
outputs:
  - name: TEST_PLAN
    type: string
    description: Structured test plan listing all test cases categorised by type and priority
  - name: BDD_SCENARIOS
    type: string
    description: Complete set of Gherkin scenarios ready for import into the target framework
  - name: COVERAGE_GAP_REPORT
    type: string
    description: Prioritised list of coverage gaps compared to EXISTING_TESTS
---

## Overview

The Test Generation Agent automates the test design phase for a user story. It replaces the manual process of writing an initial test plan, drafting Gherkin scenarios, and cross-checking coverage — tasks that typically consume several hours per story.

Human testers review the agent output, add domain-specific nuance, and promote scenarios to the test management system.

## Architecture

**Skills / Tools Used:**

| Tool / Skill | Purpose |
|---|---|
| `extract_behaviours` | Parse the user story and acceptance criteria into a structured list of testable behaviours |
| `generate_test_cases` | Produce functional, edge case, and negative test cases for each behaviour |
| `write_bdd_scenarios` | Translate test cases into Gherkin scenarios formatted for the target framework |
| `gap_analysis` | Compare generated test cases against EXISTING_TESTS and report gaps |

**Decision Flow:**

```
USER_STORY + EXISTING_TESTS + FRAMEWORK + FOCUS_AREAS
         │
         ▼
Step 1: Extract testable behaviours (extract_behaviours)
         │
         ▼
Step 2: Generate test cases — happy path, edge, negative (generate_test_cases)
         │
         ▼
Step 3: Write BDD scenarios in target framework syntax (write_bdd_scenarios)
         │
         ├─ EXISTING_TESTS provided ──▶ Step 4: Gap analysis (gap_analysis)
         │                                        │
         │                                        ▼
         │                              COVERAGE_GAP_REPORT
         │
         └─ No EXISTING_TESTS ──▶ Skip Step 4
                   │
                   ▼
         TEST_PLAN + BDD_SCENARIOS [+ COVERAGE_GAP_REPORT]
```

## System Prompt / Agent Instructions

```
[System Prompt]

You are an autonomous test generation agent for a software quality engineering team.

Your goals:
1. Extract every testable behaviour from the user story and its acceptance criteria.
2. Generate a comprehensive test plan covering happy path, edge cases, negative scenarios,
   and (where applicable) security and performance scenarios.
3. Produce BDD scenarios in Gherkin (Given/When/Then) formatted for {{FRAMEWORK}}.
4. If existing tests are provided, identify coverage gaps and prioritise them by risk.

Tools available to you:
- extract_behaviours: call with (user_story) → returns BEHAVIOURS list
- generate_test_cases: call with (behaviours, focus_areas) → returns TEST_CASES list
- write_bdd_scenarios: call with (test_cases, framework) → returns BDD_SCENARIOS
- gap_analysis: call with (test_cases, existing_tests) → returns COVERAGE_GAP_REPORT

Rules:
- Generate at least one security scenario if authentication or data access is present.
- Produce at minimum: 2 happy-path, 3 edge-case, and 2 negative scenarios per story.
- Never fabricate acceptance criteria not stated in the user story.
- Label each scenario with Priority (High/Medium/Low) and Type (Functional/Edge/Negative/Security).
- Keep scenario titles concise and unique — no duplicate Scenario names.
- If FRAMEWORK is not specified, default to Gherkin / Cucumber syntax.

Output format:

## Test Plan
{TEST_PLAN}

## BDD Scenarios
{BDD_SCENARIOS}

## Coverage Gap Report
{COVERAGE_GAP_REPORT}
```

## Configuration

```yaml
agent_id: test-generation-agent
model: gpt-4o
temperature: 0.2         # low for deterministic, reproducible test cases
max_iterations: 6        # extract → generate → bdd → gap → synthesise → format
tools:
  - extract_behaviours
  - generate_test_cases
  - write_bdd_scenarios
  - gap_analysis
```

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `{{USER_STORY}}` | string | Yes | Full user story with acceptance criteria |
| `{{EXISTING_TESTS}}` | string | No | Existing test cases for gap comparison |
| `{{FRAMEWORK}}` | string | No | `jest` / `pytest` / `junit` / `cucumber` (default: `cucumber`) |
| `{{FOCUS_AREAS}}` | string | No | Comma-separated: `security`, `performance`, `boundary`, `negative` |

## Outputs

| Name | Type | Description |
|---|---|---|
| `TEST_PLAN` | string | Structured test plan by category and priority |
| `BDD_SCENARIOS` | string | Gherkin scenarios for the target framework |
| `COVERAGE_GAP_REPORT` | string | Prioritised gaps vs existing test suite |

## Security Considerations

- The agent reads user stories and test files but never writes to the repository.
- No source code or production data is required — route user stories through `in-house-llm` for confidential requirements.
- The agent must not generate test data containing real personal information (PII); always use synthetic data.
- Scenarios involving authentication must include negative security cases by default.

## Usage

### LLM Chat (current)

Open your test file or requirements document, then:

```
@workspace #file:<user-story-file>
You are a test generation agent. Generate a BDD test plan for this user story.
User Story: {{USER_STORY}}
Framework: {{FRAMEWORK}}
Focus areas: {{FOCUS_AREAS}}
```

### API Endpoint (future portal)

```http
POST {{API_BASE_URL}}/packages/testers-quality-pack/invoke
Content-Type: application/json

{
  "agent_id": "test-generation-agent",
  "inputs": {
    "user_story": "As a registered user, I want to reset my password via email...",
    "existing_tests": "TC-001: Happy path reset flow\nTC-002: Invalid email format",
    "framework": "cucumber",
    "focus_areas": "security, boundary"
  }
}
```

### MCP Tool (future)

```json
{
  "tool": "test-generation-agent",
  "parameters": {
    "user_story": "As a registered user, I want to reset my password via email...",
    "existing_tests": "",
    "framework": "pytest",
    "focus_areas": "security"
  }
}
```

## Examples

### Example 1 — Password reset user story

**Input:**
- `USER_STORY`: `As a registered user, I want to reset my password via email, so that I can regain access to my account. AC: 1. User enters registered email. 2. System sends reset link valid for 1 hour. 3. Password must be 8+ chars, 1 uppercase, 1 number. 4. Old password is invalidated after reset.`
- `EXISTING_TESTS`: `TC-001: Successful reset with valid email`
- `FRAMEWORK`: `cucumber`
- `FOCUS_AREAS`: `security, boundary`

**Output:**

```
## Test Plan

| ID | Title | Type | Priority |
|---|---|---|---|
| TC-001 | Successful password reset (happy path) | Functional | High |
| TC-002 | Reset link expires after 60 minutes | Edge Case | High |
| TC-003 | Reset link used twice (replay attack) | Security | High |
| TC-004 | Unregistered email returns generic message | Security | High |
| TC-005 | Password exactly 8 characters accepted | Boundary | Medium |
| TC-006 | Password 7 characters rejected | Boundary | Medium |
| TC-007 | Old password rejected after successful reset | Functional | High |
| TC-008 | Reset with empty email field | Negative | Medium |

## BDD Scenarios

Feature: Password Reset

  Scenario: User successfully resets password with valid email
    Given a registered user with email "user@example.com"
    When the user submits the forgot password form with "user@example.com"
    Then a password reset email is sent to "user@example.com"
    And the email contains a reset link valid for 1 hour

  Scenario: Reset link rejected after expiry
    Given a reset token issued 61 minutes ago for "user@example.com"
    When the user submits the reset form using the expired token
    Then the system returns a 400 error
    And the message reads "This reset link has expired"

  Scenario: Unregistered email returns generic confirmation
    Given no account exists for "unknown@example.com"
    When the user submits the forgot password form with "unknown@example.com"
    Then the system displays a generic success message
    And no email is sent

## Coverage Gap Report

**GAP-001: Reset link replay attack not tested**
- Risk: High
- Category: Security
- What is missing: No test verifies that a reset link cannot be used a second time.
- Suggested test case:
  - Given a reset link has already been used once
  - When the user attempts to use the same link again
  - Then the system returns 400 with "Link already used"
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | 2026-04-20. Strong BDD output; occasionally merges edge/negative categories. |
| claude-3-5-sonnet | ✅ | 2026-04-20. Best security scenario coverage across all test inputs. |

## Changelog

### 1.0.0 — 2026-04-20
- Initial version.
