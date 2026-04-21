---
id: shared-incident-to-fix-workflow
title: Incident to Fix Workflow
type: workflow
version: "1.0.0"
status: active
workstream:
  - sre
  - developers
  - testers
author: ai-skills-maintainer
created: 2026-04-20
updated: 2026-04-20
tags:
  - incident-response
  - cross-workstream
  - bug-fix
  - regression-testing
  - workflow
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
  - copilot-gpt-4o
description: |
  A four-step cross-workstream workflow that takes a live incident from initial alert all the
  way to a merged hotfix with regression tests. SRE triages and classifies the incident, the
  Developers workstream reviews the suspect code and drafts the fix, and the Testers workstream
  generates regression test cases — producing a complete, ready-to-act INCIDENT_REPORT package.
security_classification: internal
delivery_modes:
  - copilot-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
dependencies:
  - sre-incident-triage-prompt
  - dev-code-review-skill
  - dev-commit-message-generator
  - test-generate-test-cases-from-user-story
inputs:
  - name: INCIDENT_DESCRIPTION
    type: string
    description: Human-readable description of the incident as reported (e.g. alert body or on-call note)
    required: true
  - name: ALERT_DETAILS
    type: string
    description: Raw alert payload or monitoring excerpt (metrics, error rates, log lines)
    required: true
  - name: AFFECTED_SERVICE
    type: string
    description: Name of the service at the centre of the incident
    required: true
  - name: SUSPECTED_CODE
    type: string
    description: Optional — diff or file path of the recently changed code suspected to have caused the incident
    required: false
  - name: LANGUAGE
    type: string
    description: Optional — primary programming language of the suspected code (e.g. TypeScript, Python, Go)
    required: false
outputs:
  - name: INCIDENT_REPORT
    type: string
    description: Complete package containing triage summary, root-cause analysis, hotfix PR description, commit message, and regression test cases
---

## Overview

This workflow chains together assets from three workstreams to take an incident from raw alert to a deployed-and-tested fix in the shortest possible time.

**Cross-workstream value:**
- **SRE** contributors who normally stop at "root cause identified" now hand off a structured triage that directly feeds the developer and tester steps.
- **Developer** contributors receive a pre-classified incident so they skip re-investigation and focus on writing the fix.
- **Tester** contributors receive a precise bug description so they can immediately generate targeted regression tests rather than guessing what to cover.

The output is a single `INCIDENT_REPORT` package that can be attached to the incident ticket and linked from the hotfix PR.

**Audience:** On-call SREs, developers on hotfix rotation, QA engineers, incident commanders.

## Steps

### Step 1 — Triage the Incident `[SRE]`

**Asset used:** `sre-incident-triage-prompt`
**Input:** `{{INCIDENT_DESCRIPTION}}`, `{{ALERT_DETAILS}}`, `{{AFFECTED_SERVICE}}`
**Output:** `TRIAGE` — severity classification, probable root-cause service/component, immediate mitigation options.

```
[Prompt — paste into any LLM]

You are a senior Site Reliability Engineer performing incident triage.

Incident description: {{INCIDENT_DESCRIPTION}}
Alert details: {{ALERT_DETAILS}}
Affected service: {{AFFECTED_SERVICE}}

Produce a structured triage report with the following sections:

## Severity
Classify as SEV1, SEV2, or SEV3 with a one-sentence justification.

## Probable Root Cause
Identify the most likely root-cause service, component, or recent change.
Be specific: name files, endpoints, config keys, or deployment timestamps where visible.

## Blast Radius
Who is affected (users / downstream services) and what is the observable impact?

## Immediate Mitigation Options
List up to three actions an on-call engineer can take RIGHT NOW to reduce impact
(e.g. roll back deployment, toggle feature flag, increase replica count).

## Recommended Next Step
Which code area should the developer examine first?
```

---

### Step 2 — Review the Suspected Code `[Developers]`

**Asset used:** `dev-code-review-skill`
**Input:** `{{SUSPECTED_CODE}}`, `{{LANGUAGE}}`, `TRIAGE` (from Step 1)
**Output:** `CODE_FINDINGS` — bug identification with file, line, and suggested fix.

```
[Prompt — paste into any LLM after Step 1]

You are a senior {{LANGUAGE}} engineer performing a focused bug-hunt code review.

Context from SRE triage:
{{TRIAGE}}

Review the following code for the specific bug that caused this incident.
Focus on: null/undefined handling, error propagation, race conditions,
misconfigured timeouts, and recently changed logic.

Format your findings as:

### Root Cause (confirmed)
- [FILE:LINE] Exact description of the bug.
  Suggested fix: ...

### Contributing Factors
- [FILE:LINE] Secondary issues that amplified the incident.

### Safe to Deploy Check
List any other code paths that will be affected by the proposed fix.

Code to review:
{{SUSPECTED_CODE}}
```

---

### Step 3 — Draft Hotfix PR Description and Commit Message `[Developers]`

**Asset used:** `dev-commit-message-generator`
**Input:** `TRIAGE` (Step 1), `CODE_FINDINGS` (Step 2), `{{AFFECTED_SERVICE}}`
**Output:** `HOTFIX_PR` — PR description in conventional format + conventional-commit message.

```
[Prompt — paste into any LLM after Step 2]

You are a senior developer drafting a hotfix pull request.

Incident triage:
{{TRIAGE}}

Code review findings:
{{CODE_FINDINGS}}

Affected service: {{AFFECTED_SERVICE}}

Produce:

1. A pull request description with the following sections:
   ## Problem
   (What went wrong and what was the user impact)

   ## Root Cause
   (Precise technical explanation from the code review)

   ## Fix
   (What the change does and why it is safe)

   ## Testing
   (How the fix was validated before merging)

   ## Rollback Plan
   (How to revert if the fix causes further issues)

2. A conventional-commit message (type(scope): summary, body, footer) for the squash merge.
   Use type: fix, scope: the affected service or module.
```

---

### Step 4 — Generate Regression Test Cases `[Testers]`

**Asset used:** `test-generate-test-cases-from-user-story`
**Input:** `TRIAGE` (Step 1), `CODE_FINDINGS` (Step 2), `{{LANGUAGE}}`
**Output:** `REGRESSION_TESTS` — test cases in table format plus stub code to prevent recurrence.

```
[Prompt — paste into any LLM after Step 3]

You are a QA engineer writing regression tests to prevent this incident from recurring.

Incident summary:
{{TRIAGE}}

Root cause from code review:
{{CODE_FINDINGS}}

Language / test framework: {{LANGUAGE}}

Generate regression test cases that directly target the identified bug and its contributing
factors. For each test case provide:

| ID | Description | Pre-condition | Steps | Expected Result | Priority |
|---|---|---|---|---|---|

After the table, generate stub test code in {{LANGUAGE}} for the three highest-priority
cases. Use the project's standard testing library (Jest for TypeScript/JS, pytest for Python,
testing package for Go). Use descriptive test names that make the regression intent obvious.
```

---

## Flow Diagram

```
INCIDENT_DESCRIPTION + ALERT_DETAILS + AFFECTED_SERVICE
              │
              ▼
┌─────────────────────────────┐
│  Step 1: Triage Incident     │  ◀── [SRE workstream]
│  sre-incident-triage-prompt  │
└─────────────┬───────────────┘
              │ TRIAGE
              ▼
┌─────────────────────────────┐
│  Step 2: Code Review         │  ◀── [Developers workstream]
│  dev-code-review-skill       │
│  + SUSPECTED_CODE (opt)      │
└─────────────┬───────────────┘
              │ CODE_FINDINGS
              ▼
┌─────────────────────────────┐
│  Step 3: Draft Hotfix PR     │  ◀── [Developers workstream]
│  dev-commit-message-         │
│  generator                   │
└─────────────┬───────────────┘
              │ HOTFIX_PR
              ▼
┌─────────────────────────────┐
│  Step 4: Regression Tests    │  ◀── [Testers workstream]
│  test-generate-test-cases-   │
│  from-user-story             │
└─────────────┬───────────────┘
              │
              ▼
        INCIDENT_REPORT
  (TRIAGE + CODE_FINDINGS +
   HOTFIX_PR + REGRESSION_TESTS)
```

## Usage

### Manual (Copilot Chat — current)

1. When an incident fires, copy the alert payload into `ALERT_DETAILS` and write a one-paragraph `INCIDENT_DESCRIPTION`.
2. Run Step 1 in Copilot Chat or your LLM. Save the output as `TRIAGE`.
3. Open the suspected code file in your IDE. Run Step 2 with the diff pasted in. Save as `CODE_FINDINGS`.
4. Run Step 3 to generate the PR description and commit message. Save as `HOTFIX_PR`.
5. Run Step 4 to generate regression tests. Combine all four outputs into the incident ticket.

### Automated (GitHub Actions — future)

```yaml
# .github/workflows/incident-fix-pipeline.yml (illustrative)
on:
  workflow_dispatch:
    inputs:
      incident_description: { required: true }
      alert_details:        { required: true }
      affected_service:     { required: true }
      suspected_code_path:  { required: false }
jobs:
  incident-to-fix:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run incident-to-fix workflow
        uses: org/ai-skills-action@v1
        with:
          workflow_id: shared-incident-to-fix-workflow
          inputs: ${{ toJson(github.event.inputs) }}
```

### API Endpoint (future portal)

```http
POST {{API_BASE_URL}}/packages/shared-cross-workstream-pack/invoke
Content-Type: application/json

{
  "workflow_id": "shared-incident-to-fix-workflow",
  "inputs": {
    "incident_description": "payment-service returning 23% HTTP 500s after v2.14.0 deploy",
    "alert_details": "AlertManager: payment-service error_rate > 20% for 5m. p99 latency 4200ms.",
    "affected_service": "payment-service",
    "suspected_code": "diff of src/payments/processor.ts",
    "language": "TypeScript"
  }
}
```

## Examples

### Example — payment-service 500s after deployment

**Input:**
- `INCIDENT_DESCRIPTION`: `payment-service returning 23% HTTP 500s since v2.14.0 deployed at 14:32 UTC. Checkout is broken for ~1 in 4 users.`
- `ALERT_DETAILS`: `AlertManager firing: payment-service http_error_rate{code="500"} = 0.23 for 8m. p99=4200ms (baseline 180ms). Downstream: order-service seeing 503s.`
- `AFFECTED_SERVICE`: `payment-service`
- `SUSPECTED_CODE`: *(diff of src/payments/processor.ts from v2.14.0)*
- `LANGUAGE`: `TypeScript`

---

**Step 1 Output — TRIAGE:**

```
## Severity
SEV1 — ~23% of checkout requests are failing, causing direct revenue loss.

## Probable Root Cause
Deployment v2.14.0 of payment-service at 14:32 UTC correlates exactly with the
error-rate spike. The change in src/payments/processor.ts (refactor of retry logic)
is the prime suspect.

## Blast Radius
All users attempting checkout via payment-service. order-service is receiving 503s
from payment-service, causing cascading failures in order confirmation emails.

## Immediate Mitigation Options
1. Roll back payment-service to v2.13.4: `kubectl rollout undo deployment/payment-service`
2. Enable the PAYMENT_LEGACY_PROCESSOR feature flag to route traffic to the old code path.
3. Scale payment-service replicas from 3 → 8 to reduce per-pod queue depth while investigating.

## Recommended Next Step
Examine the retry-logic refactor in src/payments/processor.ts, specifically the
new exponential-backoff implementation introduced in v2.14.0.
```

---

**Step 2 Output — CODE_FINDINGS:**

```
### Root Cause (confirmed)
- [src/payments/processor.ts:87] `retryDelay` is calculated as `baseDelay * attempt`
  but `attempt` starts at 0, making the first retry immediate (delay = 0ms).
  Under load this causes a retry storm: all in-flight requests retry simultaneously,
  exhausting the Stripe API rate limit and cascading into 500s.
  Suggested fix: Change to `baseDelay * (attempt + 1)` or use `Math.pow(2, attempt) * baseDelay`.

### Contributing Factors
- [src/payments/processor.ts:102] `maxRetries` was increased from 2 to 5 in this PR,
  amplifying the retry storm effect.

### Safe to Deploy Check
The fix only affects the retry loop in `processPayment()`. No other callers.
`refundPayment()` uses a separate retry utility and is unaffected.
```

---

**Step 3 Output — HOTFIX_PR:**

```
## Problem
payment-service was returning HTTP 500 for ~23% of checkout requests following the
v2.14.0 deployment, causing direct revenue loss and cascading failures in order-service.

## Root Cause
The exponential-backoff retry implementation in `processor.ts:87` used `attempt`
starting at 0, making the first retry delay 0ms. Combined with `maxRetries` being
raised from 2 to 5, this created a retry storm that exhausted the Stripe API rate limit.

## Fix
Changed retry delay to `baseDelay * (attempt + 1)` to ensure all retries have a
positive delay. Reverted `maxRetries` to 2 pending a proper load test of the new value.

## Testing
- Existing unit tests pass.
- Regression tests for zero-delay and retry-storm scenarios added (see test/payments/processor.retry.test.ts).
- Load tested locally at 500 rps: error rate dropped to 0.1%.

## Rollback Plan
`kubectl rollout undo deployment/payment-service` — rolls back to v2.13.4 in ~30s.

---
Commit: fix(payment-service): correct zero-delay retry storm in processor

The exponential-backoff retry loop used `attempt` (0-indexed) as the multiplier,
making the first retry delay 0ms and causing a storm under load.

Fix: use `attempt + 1` as the multiplier so all retries have a positive back-off.
Also reverts maxRetries from 5 to 2 pending load testing.

Fixes: INC-4821
Reviewed-by: @on-call-dev
```

---

**Step 4 Output — REGRESSION_TESTS:**

| ID | Description | Pre-condition | Steps | Expected Result | Priority |
|---|---|---|---|---|---|
| RT-01 | First retry delay is never zero | processor configured with baseDelay=100ms | Trigger one failed payment; capture retry timing | First retry fires after ≥ 100ms | P0 |
| RT-02 | Retry delays increase exponentially | processor configured with baseDelay=100ms, maxRetries=3 | Simulate 3 consecutive Stripe failures; capture all retry timings | Delays ≈ 100ms, 200ms, 300ms (±10%) | P0 |
| RT-03 | maxRetries default is 2 | Fresh processor instance | Read `processor.config.maxRetries` | Value equals 2 | P1 |
| RT-04 | Retry storm does not occur at 500 rps | 500 concurrent payment requests; Stripe returns 429 | Fire all requests simultaneously | No burst of simultaneous retries within a 10ms window | P0 |

```typescript
// test/payments/processor.retry.test.ts

import { processPayment } from '../../src/payments/processor';
import { mockStripeFailure } from '../helpers/stripe-mock';

describe('processPayment retry back-off', () => {
  it('RT-01: first retry delay is never zero', async () => {
    mockStripeFailure({ times: 1 });
    const timestamps: number[] = [];
    const origSetTimeout = global.setTimeout;
    jest.spyOn(global, 'setTimeout').mockImplementation((fn, delay) => {
      timestamps.push(delay as number);
      return origSetTimeout(fn, 0);
    });
    await processPayment({ amount: 100, currency: 'GBP' });
    expect(timestamps[0]).toBeGreaterThan(0);
  });

  it('RT-02: retry delays increase with each attempt', async () => {
    mockStripeFailure({ times: 3 });
    const delays: number[] = [];
    jest.spyOn(global, 'setTimeout').mockImplementation((fn, delay) => {
      delays.push(delay as number);
      return setTimeout(fn, 0);
    });
    await processPayment({ amount: 100, currency: 'GBP' }).catch(() => {});
    expect(delays[1]).toBeGreaterThan(delays[0]);
    expect(delays[2]).toBeGreaterThan(delays[1]);
  });

  it('RT-03: default maxRetries is 2', () => {
    const { config } = require('../../src/payments/processor');
    expect(config.maxRetries).toBe(2);
  });
});
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | 2026-04-20. All four steps work in a single Copilot Chat session with outputs passed forward. Step 3 PR description is high quality with minimal editing needed. |
| claude-3-5-sonnet | ✅ | 2026-04-20. Step 2 root-cause identification is more precise. Step 4 generates more thorough edge-case regression tests. |

## Changelog

### 1.0.0 — 2026-04-20
- Initial version. Cross-workstream combination of sre-incident-triage-prompt, dev-code-review-skill, dev-commit-message-generator, test-generate-test-cases-from-user-story.
