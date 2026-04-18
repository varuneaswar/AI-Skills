---
id: test-generate-test-cases-from-user-story
title: Generate Test Cases from User Story
type: prompt
version: "1.0.0"
status: active
workstream:
  - testers
author: ai-skills-maintainer
created: 2024-01-01
updated: 2024-01-01
tags:
  - test-cases
  - user-story
  - bdd
  - quality-assurance
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
  - gemini-1.5-pro
  - copilot-gpt-4o
description: |
  Generates a comprehensive set of test cases (happy path, edge cases, and negative scenarios)
  from a user story and its acceptance criteria, formatted as Gherkin scenarios.
security_classification: internal
inputs:
  - name: USER_STORY
    type: string
    description: The full user story text (As a... I want... So that...)
    required: true
  - name: ACCEPTANCE_CRITERIA
    type: string
    description: The acceptance criteria for the user story
    required: true
  - name: CONTEXT
    type: string
    description: Additional business context, constraints, or known edge cases
    required: false
---

## Overview

Use this prompt to quickly generate a thorough set of test cases for a user story, saving testers significant time on initial test design. The output is formatted as Gherkin (`Given / When / Then`) scenarios that can be imported into Cucumber, SpecFlow, or similar BDD frameworks, or used as manual test cases.

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `{{USER_STORY}}` | string | Yes | The full user story |
| `{{ACCEPTANCE_CRITERIA}}` | string | Yes | Acceptance criteria (numbered or bulleted list) |
| `{{CONTEXT}}` | string | No | Extra business context or known edge cases |

## Prompt

```
You are an experienced QA engineer specializing in behaviour-driven development (BDD).

Given the following user story and acceptance criteria, generate a comprehensive set of
test cases formatted as Gherkin scenarios.

User Story:
{{USER_STORY}}

Acceptance Criteria:
{{ACCEPTANCE_CRITERIA}}

Additional Context:
{{CONTEXT}}

Generate test cases covering:
1. Happy path scenarios (all ACs passing)
2. Edge cases (boundary values, empty inputs, maximum lengths)
3. Negative scenarios (invalid inputs, missing required fields, unauthorized access)
4. Any business rule variations implied by the ACs

Format each test case as:

**TC-NNN: Short descriptive title**
```gherkin
Scenario: [title]
  Given [precondition]
  When [action]
  Then [expected outcome]
  And [additional assertion if needed]
```
Priority: High / Medium / Low
Type: Functional / Edge Case / Negative / Security

After the test cases, provide a brief **Coverage Summary** listing which acceptance
criteria each test case covers (use AC numbers).

Aim for completeness over brevity. Include at least one security-related scenario if
user authentication or data access is involved.
```

## Usage

### GitHub Copilot Chat

```
Generate BDD test cases for this user story:
Story: <paste user story>
ACs: <paste acceptance criteria>
```

### Standalone

1. Copy the prompt above.
2. Fill in the three placeholders.
3. Paste into your LLM interface.

## Examples

### Example 1

**Input:**
- `USER_STORY`: `As a registered user, I want to reset my password via email, so that I can regain access to my account if I forget my password.`
- `ACCEPTANCE_CRITERIA`: `1. User enters registered email address. 2. System sends reset link valid for 1 hour. 3. User clicks link and is prompted to enter new password. 4. Password must be 8+ characters with at least one uppercase letter and one number. 5. After reset, old password is invalidated.`

**Output (abbreviated):**
```
**TC-001: Successful password reset flow**
\`\`\`gherkin
Scenario: User successfully resets password with valid email
  Given a user with email "user@example.com" is registered
  When the user submits the forgot password form with "user@example.com"
  Then a password reset email is sent to "user@example.com"
  And the email contains a reset link valid for 1 hour
\`\`\`
Priority: High | Type: Functional

**TC-002: Reset link expires after 1 hour**
...

**TC-003: Unregistered email address**
\`\`\`gherkin
Scenario: User enters an email address not in the system
  Given no account exists for "unknown@example.com"
  When the user submits the forgot password form with "unknown@example.com"
  Then the system displays a generic confirmation message
  And no email is sent (prevents user enumeration)
\`\`\`
Priority: High | Type: Security
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | Tested 2024-01-01. Good coverage, occasionally needs a follow-up to add more edge cases. |
| claude-3-5-sonnet | ✅ | Tested 2024-01-01. Excellent at security scenarios. |

## Changelog

### 1.0.0 — 2024-01-01
- Initial version.
