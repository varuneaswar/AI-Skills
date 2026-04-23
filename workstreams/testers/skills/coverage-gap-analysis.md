---
id: test-coverage-gap-analysis
title: Test Coverage Gap Analysis
type: skill
version: "1.0.0"
status: active
workstream:
  - testers
author: ai-skills-maintainer
created: 2024-01-01
updated: 2024-01-01
tags:
  - coverage
  - gap-analysis
  - quality-assurance
  - risk
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
description: |
  Analyses an existing test suite description or list of test cases against a feature's
  requirements or source code to identify untested scenarios, missing edge cases, and
  risk-weighted coverage gaps.
security_classification: internal
inputs:
  - name: REQUIREMENTS_OR_CODE
    type: string
    description: The feature requirements, user stories, or source code being tested
    required: true
  - name: EXISTING_TESTS
    type: string
    description: Description or list of existing test cases
    required: true
  - name: RISK_AREAS
    type: string
    description: "Optional: known high-risk areas to prioritise (e.g., 'payment processing', 'authentication')"
    required: false
outputs:
  - name: GAP_REPORT
    type: string
    description: Prioritised list of coverage gaps with suggested test cases
delivery_modes:
  - llm-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
---

## Overview

Use this skill before releasing a feature to identify what is not tested. The output is a risk-prioritised gap report with concrete suggested test cases, not just a generic coverage checklist.

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `{{REQUIREMENTS_OR_CODE}}` | string | Yes | Requirements or source code under test |
| `{{EXISTING_TESTS}}` | string | Yes | List or description of current tests |
| `{{RISK_AREAS}}` | string | No | High-risk areas to prioritise |

## Outputs

| Name | Type | Description |
|---|---|---|
| `GAP_REPORT` | string | Prioritised gap report with suggested test cases |

## Skill Definition

```
[System Prompt]

You are a senior QA engineer specialising in test coverage analysis. You identify
gaps systematically, prioritise by risk, and suggest concrete test cases to fill them.

You are given:
1. Requirements or source code: {{REQUIREMENTS_OR_CODE}}
2. Existing test cases: {{EXISTING_TESTS}}
3. Known high-risk areas: {{RISK_AREAS}}

Produce a coverage gap report using this format:

## Coverage Summary
Brief assessment of overall coverage completeness (e.g., "Core happy-paths covered, significant gaps in error handling and boundary values").

## Gap Analysis

For each gap found:

**GAP-NNN: [Short title]**
- Risk: High / Medium / Low
- Category: Happy-path / Edge-case / Negative / Security / Performance
- What is missing: Description of the untested scenario.
- Suggested test case:
  - Given: [precondition]
  - When: [action]
  - Then: [expected result]

## Summary Table

| Gap ID | Risk | Category | Effort to Add |
|---|---|---|---|
| GAP-001 | High | Security | Low |
...

Sort gaps by risk descending. Focus on {{RISK_AREAS}} first if specified.
```

## Usage

### LLM Chat (current)

```
@workspace #file:<test-file>
Analyse the test coverage for the code below. Identify gaps and suggest missing tests.

Requirements / Code:
{{REQUIREMENTS_OR_CODE}}

Existing tests:
{{EXISTING_TESTS}}
```

### API Endpoint (future portal)

```http
POST {{API_BASE_URL}}/packages/testers-quality-pack/invoke
Content-Type: application/json

{
  "skill_id": "test-coverage-gap-analysis",
  "inputs": {
    "requirements_or_code": "...",
    "existing_tests": "...",
    "risk_areas": "authentication, payment processing"
  }
}
```

## Examples

### Example 1

**Input:** Password reset feature with existing happy-path tests only.

**Output (excerpt):**
```
## Coverage Summary
Happy-path is covered. Major gaps in security scenarios and token expiry behaviour.

**GAP-001: Reset token expiry not tested**
- Risk: High
- Category: Edge-case
- What is missing: No test verifies that a reset link is rejected after 1 hour.
- Suggested test case:
  - Given a reset token was issued 61 minutes ago
  - When the user submits the reset form with that token
  - Then the system returns a 400 error with message "Link has expired"

**GAP-002: User enumeration via reset flow**
- Risk: High
- Category: Security
- What is missing: No test checks that the system returns the same response for registered and unregistered emails.
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | 2024-01-01. Excellent at security gap identification. |
| claude-3-5-sonnet | ✅ | 2024-01-01. Thorough across all gap categories. |

## Changelog

### 1.0.0 — 2024-01-01
- Initial version.
