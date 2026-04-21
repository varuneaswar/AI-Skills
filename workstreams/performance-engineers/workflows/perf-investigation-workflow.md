---
id: perf-investigation-workflow
title: Performance Investigation Workflow
type: workflow
version: "1.0.0"
status: active
workstream:
  - performance-engineers
author: ai-skills-maintainer
created: 2026-04-20
updated: 2026-04-20
tags:
  - performance
  - investigation
  - workflow
  - load-testing
  - regression
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
  - copilot-gpt-4o
description: |
  A four-step workflow that takes baseline and current load test results through metric
  comparison and SLO compliance, bottleneck identification, severity classification and
  release recommendation, and root cause synthesis — producing a comprehensive performance
  investigation report with prioritised fix recommendations.
security_classification: internal
delivery_modes:
  - copilot-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
dependencies:
  - perf-bottleneck-analysis
  - perf-performance-baseline-prompt
inputs:
  - name: BASELINE_RESULTS
    type: string
    description: Performance metrics from the baseline/previous test run (p50/p95/p99 latency, throughput, error rate)
    required: true
  - name: CURRENT_RESULTS
    type: string
    description: Performance metrics from the current/candidate test run (same format as BASELINE_RESULTS)
    required: true
  - name: SERVICE_NAME
    type: string
    description: Name of the service under investigation
    required: true
  - name: SLO_THRESHOLDS
    type: string
    description: "Optional: SLO targets per metric for pass/fail assessment alongside regression detection"
    required: false
outputs:
  - name: PERF_INVESTIGATION_REPORT
    type: string
    description: Full performance investigation report with metric comparison, bottleneck findings, severity classification, release recommendation, root cause analysis, and fix recommendations
---

## Overview

This workflow provides a structured, repeatable process for investigating performance regressions after a load test run. Engineers follow the four steps in order, using each step's output as input to the next — producing a comprehensive report in a single session.

**Audience:** Performance engineers, SREs, release managers deciding on release go/no-go.

## Steps

### Step 1 — Compare Baseline vs Current Metrics and Compute SLO Compliance

**Asset used:** `perf-performance-baseline-prompt`  
**Input:** `{{BASELINE_RESULTS}}`, `{{CURRENT_RESULTS}}`, `{{SLO_THRESHOLDS}}`  
**Output:** `METRIC_COMPARISON` — delta table and SLO pass/fail assessment.

```
[Prompt — paste into any LLM]

You are a performance engineer comparing two load test runs.

Service: {{SERVICE_NAME}}
Baseline results: {{BASELINE_RESULTS}}
Current results:  {{CURRENT_RESULTS}}
SLO thresholds:   {{SLO_THRESHOLDS}}

Produce the following:

## Metric Comparison — {{SERVICE_NAME}}

| Metric      | Baseline | Current | Delta % | Regression? | SLO Target | SLO Status |
|---|---|---|---|---|---|---|
| p50 Latency | | | | Yes/No | | ✅/❌/— |
| p95 Latency | | | | | | |
| p99 Latency | | | | | | |
| Throughput  | | | | | | |
| Error Rate  | | | | | | |

Rules:
- Delta % = (current − baseline) / |baseline| × 100%. Round to one decimal place.
- Regression = Yes if delta > +5% for latency/error rate, or < −5% for throughput.
- SLO Status = ✅ if metric meets SLO, ❌ if not, — if no SLO provided.
- List all regressions found at the bottom: "Regressions detected: [list] / None detected."
```

---

### Step 2 — Identify Bottleneck Type for Each Regression

**Asset used:** `perf-bottleneck-analysis`  
**Input:** `METRIC_COMPARISON` (Step 1 output), `{{SERVICE_NAME}}`  
**Output:** `BOTTLENECK_FINDINGS` — classified bottleneck type and evidence for each regression.

```
[Prompt — paste into any LLM after Step 1]

You are a performance engineering specialist analysing regression root causes.

Service: {{SERVICE_NAME}}
Metric comparison and regressions from Step 1:
{{METRIC_COMPARISON}}

For each regression identified in Step 1, determine the most likely bottleneck category.

## Bottleneck Findings

**BOTTLENECK-NNN: [Metric] regression**
- Category: CPU | Memory | I/O | Network | Database | Application-Logic | External-Dependency
- Evidence: What in the metric data points to this category.
- Affected metrics: List all metrics impacted.
- Likely code area: Where in the codebase this type of bottleneck typically originates.

If no regressions were detected in Step 1, output: "No regressions to analyse."
```

---

### Step 3 — Classify Regressions by Severity and Produce Release Recommendation

**Asset used:** *(inline prompt)*  
**Input:** `METRIC_COMPARISON` (Step 1), `BOTTLENECK_FINDINGS` (Step 2)  
**Output:** `SEVERITY_CLASSIFICATION` and `RELEASE_RECOMMENDATION`.

```
[Prompt — paste into any LLM after Step 2]

You are a performance engineering lead making a release go/no-go decision.

Metric comparison:
{{METRIC_COMPARISON}}

Bottleneck findings:
{{BOTTLENECK_FINDINGS}}

Classify each regression by severity and produce a release recommendation.

## Severity Classification

| Regression | Delta % | Severity | Rationale |
|---|---|---|---|

Severity rules:
- Critical: >50% degradation — release must be blocked
- High:     20–50% degradation — release should be blocked; exception requires VP approval
- Medium:   5–20% degradation — flag for investigation; release conditional on risk assessment
- None:     <5% change — within noise threshold

## Release Recommendation
**Block ❌ / Conditional ⚠️ / Pass ✅**
Rationale: One sentence explaining the decision.

## Conditions for Release (if Conditional)
- Bullet list of conditions that must be met before release can proceed.
```

---

### Step 4 — Synthesise Root Cause Analysis and Fix Recommendations

**Asset used:** *(inline prompt)*  
**Input:** `METRIC_COMPARISON` (Step 1), `BOTTLENECK_FINDINGS` (Step 2), `SEVERITY_CLASSIFICATION` (Step 3)  
**Output:** `PERF_INVESTIGATION_REPORT`

```
[Prompt — paste into any LLM after Steps 1–3]

Combine the outputs from the previous three steps into a complete Performance
Investigation Report ready for sharing with engineering stakeholders.

Metric comparison: {{METRIC_COMPARISON}}
Bottleneck findings: {{BOTTLENECK_FINDINGS}}
Severity classification and release recommendation: {{SEVERITY_CLASSIFICATION}}

Produce the following report:

## Performance Investigation Report — {{SERVICE_NAME}}

### Executive Summary
2–3 sentences: what changed, what regressed, and the release recommendation.

### Metric Comparison
(Include the table from Step 1.)

### Severity Classification and Release Recommendation
(Include the classification table and recommendation from Step 3.)

### Root Cause Analysis
For each regression:
**[Regression] — [Severity]**
- Bottleneck category: (from Step 2)
- Evidence: (from Step 2)
- Hypothesised cause: Specific, actionable hypothesis about what code change or system
  behaviour is causing this regression.
- Confidence: High / Medium / Low — based on how clearly the evidence points to the cause.

### Fix Recommendations (Priority Order)
| Priority | Regression | Fix | Effort | Expected Improvement |
|---|---|---|---|---|
| P1 | | | Low/Medium/High | |

### Verification Steps
Numbered list of steps to verify the regressions are resolved after fixes are applied.
```

---

## Flow Diagram

```
BASELINE_RESULTS + CURRENT_RESULTS + SERVICE_NAME + SLO_THRESHOLDS
         │
         ▼
[Step 1: Metric Comparison + SLO Assessment] ──▶ METRIC_COMPARISON
         │
         ▼
[Step 2: Bottleneck Identification] ──▶ BOTTLENECK_FINDINGS
         │
         ▼
[Step 3: Severity Classification + Release Recommendation] ──▶ SEVERITY_CLASSIFICATION
         │
         ▼
[Step 4: Root Cause Synthesis + Fix Recommendations] ──▶ PERF_INVESTIGATION_REPORT
```

## Usage

### Manual (Copilot Chat — current)

1. Gather baseline and current load test results in a consistent format (p50/p95/p99 latency, throughput, error rate).
2. Copy each step prompt in order, substitute the placeholders, and paste into GitHub Copilot Chat or your preferred LLM.
3. Save each step's output to use as input for the next step.
4. Post the final `PERF_INVESTIGATION_REPORT` as a PR comment or attach it to the release ticket.

### Automated (GitHub Actions — current)

See `workstreams/performance-engineers/automations/benchmark-ci-automation.md` for a ready-to-use GitHub Actions implementation that runs this workflow on every PR.

### API Endpoint (future portal)

```http
POST {{API_BASE_URL}}/packages/performance-engineers-analysis-pack/invoke
Content-Type: application/json

{
  "workflow_id": "perf-investigation-workflow",
  "inputs": {
    "baseline_results": "p50:95ms p95:180ms p99:420ms throughput:1200 error_rate:0.05%",
    "current_results":  "p50:98ms p95:275ms p99:890ms throughput:1180 error_rate:0.9%",
    "service_name": "checkout-api",
    "slo_thresholds": "p99 < 500ms, error rate < 0.1%"
  }
}
```

## Examples

### Example 1 — checkout-api latency and error rate regression

**Input:**
- `BASELINE_RESULTS`: `p50: 95ms, p95: 180ms, p99: 420ms, throughput: 1200 req/s, error_rate: 0.05%`
- `CURRENT_RESULTS`: `p50: 98ms, p95: 275ms, p99: 890ms, throughput: 1180 req/s, error_rate: 0.9%`
- `SERVICE_NAME`: `checkout-api`
- `SLO_THRESHOLDS`: `p99 < 500ms, error rate < 0.1%`

**Step 1 Output (METRIC_COMPARISON excerpt):**
```
| Metric      | Baseline | Current | Delta % | Regression? | SLO Target  | SLO Status |
|---|---|---|---|---|---|---|
| p95 Latency | 180ms    | 275ms   | +52.8%  | Yes         | —           | —          |
| p99 Latency | 420ms    | 890ms   | +111.9% | Yes         | < 500ms     | ❌         |
| Error Rate  | 0.05%    | 0.9%    | +1700%  | Yes         | < 0.1%      | ❌         |

Regressions detected: p95 Latency, p99 Latency, Error Rate
```

**Step 3 Output (RELEASE_RECOMMENDATION):**
```
## Release Recommendation
**Block ❌**
Rationale: Three Critical regressions detected — p99 latency and error rate both exceed
SLOs by a wide margin and must be resolved before release.
```

**Step 4 Output (PERF_INVESTIGATION_REPORT excerpt):**
```
### Root Cause Analysis

**p99 Latency / p95 Latency — Critical**
- Bottleneck category: External-Dependency
- Evidence: Tail latency increase (+111.9% at p99) disproportionate to p50 increase (+3.2%)
  indicates a slow external call in the request path, not a general compute bottleneck.
- Hypothesised cause: A synchronous call to an external service (inventory, payment gateway,
  or auth) added to the critical path is serialising at high concurrency, causing tail latency
  amplification.
- Confidence: High

### Fix Recommendations
| Priority | Regression  | Fix                                   | Effort | Expected Improvement |
|---|---|---|---|---|
| P1       | p99/p95/err | Add timeout + circuit breaker on ext call | Low  | Error rate to baseline |
| P1       | p99/p95     | Make external call asynchronous       | Medium | p99 returns to ~450ms  |
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | 2026-04-20. All four steps work in a single Copilot Chat session. Delta calculations accurate. |
| claude-3-5-sonnet | ✅ | 2026-04-20. Step 4 root cause analysis is more detailed. Step 3 occasionally adds extra severity tiers. |

## Changelog

### 1.0.0 — 2026-04-20
- Initial version.
