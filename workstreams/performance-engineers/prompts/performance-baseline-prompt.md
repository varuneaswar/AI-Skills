---
id: perf-performance-baseline-prompt
title: Performance Baseline Report Generator
type: prompt
version: "1.0.0"
status: active
workstream:
  - performance-engineers
author: ai-skills-maintainer
created: 2026-04-20
updated: 2026-04-20
tags:
  - performance
  - baseline
  - slo
  - latency
  - throughput
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
description: |
  Generates a structured Performance Baseline Report that evaluates each metric against
  defined SLOs, reports Pass/Fail per metric, calculates variance, classifies failures
  by severity (Critical/High/Medium), and produces an overall service readiness rating
  (Ready / Conditionally Ready / Not Ready) — providing a consistent baseline record
  before performance regression analysis.
security_classification: internal
delivery_modes:
  - llm-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
inputs:
  - name: SERVICE_NAME
    type: string
    description: Name of the service being baselined
    required: true
  - name: METRICS
    type: string
    description: "Measured performance metrics: p50/p95/p99 latency (ms), throughput (req/s), error rate (%)"
    required: true
  - name: TARGET_SLOS
    type: string
    description: "SLO targets for each metric, e.g., 'p99 < 500ms, error rate < 0.1%, throughput > 1000 req/s'"
    required: true
  - name: TEST_ENVIRONMENT
    type: string
    description: "Environment where the test was run: staging | production | load-test"
    required: true
outputs:
  - name: BASELINE_REPORT
    type: string
    description: Structured performance baseline report with SLO assessment, variance, failure classification, and readiness rating
---

## Overview

A consistent, well-documented performance baseline is the foundation of regression detection. This prompt generates a standardised baseline report that teams can store alongside their service documentation and compare against future test runs.

Use it after every load test, after major releases, or when establishing SLOs for a new service.

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `{{SERVICE_NAME}}` | string | Yes | Service name |
| `{{METRICS}}` | string | Yes | p50/p95/p99 latency (ms), throughput (req/s), error rate (%) |
| `{{TARGET_SLOS}}` | string | Yes | Target SLO thresholds per metric |
| `{{TEST_ENVIRONMENT}}` | string | Yes | `staging`, `production`, or `load-test` |

## Prompt

```
[System]
You are a performance engineering specialist responsible for establishing and documenting
performance baselines. Your reports must be precise, objective, and directly actionable.

[Task]
Generate a Performance Baseline Report for {{SERVICE_NAME}} using the measured metrics
and SLO targets provided. Evaluate each metric against its SLO, classify any failures,
and produce an overall service readiness rating.

Service: {{SERVICE_NAME}}
Test environment: {{TEST_ENVIRONMENT}}
Measured metrics: {{METRICS}}
Target SLOs: {{TARGET_SLOS}}

[Rules]
1. Evaluate every metric in METRICS against the corresponding SLO in TARGET_SLOS.
2. Calculate variance as: (Actual − Target) / Target × 100%. Positive variance = over target (bad for latency, throughput depends on direction).
3. Classify SLO failures by severity:
   - Critical: metric exceeds SLO by >50% (e.g., p99 target 500ms, actual >750ms)
   - High:     metric exceeds SLO by 20–50%
   - Medium:   metric exceeds SLO by 1–20%
4. A metric that meets its SLO is "Pass ✅"; one that does not is "Fail ❌".
5. Readiness Rating rules:
   - Ready:                 all metrics Pass
   - Conditionally Ready:   only Medium failures present
   - Not Ready:             any Critical or High failure present
6. Do not invent metrics not present in the input — if a metric has no SLO, list it as "No SLO defined".
7. Variance for throughput: negative variance is bad (actual < target).

[Output Format]
Produce the following Markdown report exactly.

## Performance Baseline Report — {{SERVICE_NAME}}

**Environment:** {{TEST_ENVIRONMENT}}
**Date:** [today's date]
**Overall Readiness:** Ready ✅ | Conditionally Ready ⚠️ | Not Ready ❌

---

### SLO Assessment

| Metric | Target SLO | Actual | Variance | Status | Failure Severity |
|---|---|---|---|---|---|
| p50 Latency | | ms | | ✅ Pass / ❌ Fail | — / Medium / High / Critical |
| p95 Latency | | ms | | | |
| p99 Latency | | ms | | | |
| Throughput  | | req/s | | | |
| Error Rate  | | % | | | |

---

### Failure Analysis
*(Omit this section if all metrics Pass.)*

For each failing metric, one paragraph:
**[Metric] — [Severity]**
- Actual vs target: [value] vs [target] ([variance]%)
- Impact: [what this means for end users or downstream services]
- Recommended investigation: [specific next step]

---

### Baseline Summary
One paragraph summarising the overall health of the service, key strengths, and
the most critical issue to address before the service can be considered production-ready.

---

### Stored Baseline Values
*(Use this table as the reference for future regression comparisons.)*

| Metric | Baseline Value | Environment | Date |
|---|---|---|---|
```

## Usage

### LLM Chat (current)

```
Generate a Performance Baseline Report.
SERVICE_NAME: {{SERVICE_NAME}}
TEST_ENVIRONMENT: {{TEST_ENVIRONMENT}}
METRICS: {{METRICS}}
TARGET_SLOS: {{TARGET_SLOS}}
```

### Standalone (any LLM — current)

1. Copy the prompt above and replace all `{{placeholders}}` with real values.
2. Paste into your preferred LLM interface.
3. Save the output as `perf-baseline-<service>-<date>.md` in your service's performance documentation.

### API Endpoint (future portal)

```http
POST {{API_BASE_URL}}/packages/performance-engineers-analysis-pack/invoke
Content-Type: application/json

{
  "prompt_id": "perf-performance-baseline-prompt",
  "inputs": {
    "service_name": "checkout-api",
    "metrics": "p50: 95ms, p95: 310ms, p99: 820ms, throughput: 1150 req/s, error_rate: 0.8%",
    "target_slos": "p99 < 500ms, p95 < 200ms, throughput > 1000 req/s, error rate < 0.1%",
    "test_environment": "load-test"
  }
}
```

## Examples

### Example 1 — checkout-api with p95 and error rate failures

**Input:**
- `SERVICE_NAME`: `checkout-api`
- `METRICS`: `p50: 95ms, p95: 310ms, p99: 820ms, throughput: 1150 req/s, error_rate: 0.8%`
- `TARGET_SLOS`: `p99 < 500ms, p95 < 200ms, throughput > 1000 req/s, error rate < 0.1%`
- `TEST_ENVIRONMENT`: `load-test`

**Output:**

```
## Performance Baseline Report — checkout-api

**Environment:** load-test
**Date:** 2026-04-20
**Overall Readiness:** Not Ready ❌

---

### SLO Assessment

| Metric      | Target SLO    | Actual     | Variance  | Status   | Failure Severity |
|---|---|---|---|---|---|
| p50 Latency | < 200ms       | 95ms       | −52.5%    | ✅ Pass  | —                |
| p95 Latency | < 200ms       | 310ms      | +55%      | ❌ Fail  | Critical         |
| p99 Latency | < 500ms       | 820ms      | +64%      | ❌ Fail  | Critical         |
| Throughput  | > 1000 req/s  | 1150 req/s | +15%      | ✅ Pass  | —                |
| Error Rate  | < 0.1%        | 0.8%       | +700%     | ❌ Fail  | Critical         |

---

### Failure Analysis

**p95 Latency — Critical**
- Actual vs target: 310ms vs 200ms (+55%)
- Impact: 5% of users experience response times 55% above the target, leading to perceived
  sluggishness during checkout and potential cart abandonment.
- Recommended investigation: Profile the 90th–99th percentile requests; check for slow
  database queries or synchronous external calls in the checkout path.

**p99 Latency — Critical**
- Actual vs target: 820ms vs 500ms (+64%)
- Impact: 1% of users — likely those triggering edge-case code paths — experience near-1s
  latency, which will cause timeout errors in upstream services with a 1s timeout budget.
- Recommended investigation: Capture traces for the slowest 1% of requests; look for
  N+1 query patterns or blocking I/O operations.

**Error Rate — Critical**
- Actual vs target: 0.8% vs 0.1% (+700%)
- Impact: 8 in 1000 requests fail, generating user-visible errors and potential data
  inconsistencies if checkout transactions are partially processed.
- Recommended investigation: Filter error logs by status code; identify the dominant
  error type (timeout, 5xx, connection refused) and the service or dependency causing it.

---

### Baseline Summary
checkout-api meets its throughput SLO comfortably and p50 latency is well within target.
However, three Critical SLO failures — p95 latency, p99 latency, and error rate — mean
the service is Not Ready for production in its current state. The error rate failure is
the most urgent: an 8× SLO breach indicates a systemic issue rather than an edge case.
The latency failures likely share the same root cause. Resolve the error-producing code
path first, then re-baseline.

---

### Stored Baseline Values

| Metric      | Baseline Value | Environment | Date       |
|---|---|---|---|
| p50 Latency | 95ms           | load-test   | 2026-04-20 |
| p95 Latency | 310ms          | load-test   | 2026-04-20 |
| p99 Latency | 820ms          | load-test   | 2026-04-20 |
| Throughput  | 1150 req/s     | load-test   | 2026-04-20 |
| Error Rate  | 0.8%           | load-test   | 2026-04-20 |
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | 2026-04-20. Variance calculations accurate. Readiness rating correctly derived. |
| claude-3-5-sonnet | ✅ | 2026-04-20. More detailed failure impact descriptions. Occasionally adds extra context rows. |

## Changelog

### 1.0.0 — 2026-04-20
- Initial version.
