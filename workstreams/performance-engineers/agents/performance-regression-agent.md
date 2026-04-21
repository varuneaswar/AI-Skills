---
id: perf-performance-regression-agent
title: Performance Regression Analysis Agent
type: agent
version: "1.0.0"
status: active
workstream:
  - performance-engineers
author: ai-skills-maintainer
created: 2026-04-20
updated: 2026-04-20
tags:
  - performance
  - regression
  - load-testing
  - profiling
  - analysis
  - agent
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
  - copilot-gpt-4o
description: |
  An autonomous agent that compares two load test runs, identifies performance regressions
  (>5% degradation), classifies them by severity (Critical/>50%, High/20–50%, Medium/5–20%),
  correlates regressions with code changes, produces root cause hypotheses, and delivers
  prioritised fix recommendations — accelerating post-load-test triage and release decisions.
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
    description: Name of the service being analysed
    required: true
  - name: CHANGED_CODE_SUMMARY
    type: string
    description: "Optional: summary of code changes between the two test runs (PR description, commit messages, or diff summary)"
    required: false
  - name: SLO_THRESHOLDS
    type: string
    description: "Optional: SLO targets to include in pass/fail assessment alongside regression detection"
    required: false
outputs:
  - name: REGRESSION_REPORT
    type: string
    description: Full regression analysis report with metric comparison table and narrative
  - name: SEVERITY_CLASSIFICATION
    type: string
    description: "Ordered list of regressions by severity: Critical | High | Medium"
  - name: ROOT_CAUSE_ANALYSIS
    type: string
    description: Hypotheses for the root cause of each regression, correlated with code changes where possible
  - name: FIX_RECOMMENDATIONS
    type: string
    description: Prioritised, actionable fix recommendations with estimated effort
---

## Overview

The Performance Regression Analysis Agent eliminates the manual work of comparing load test spreadsheets and writing up triage notes. It ingests two sets of test results, automatically surfaces regressions, and produces a structured report that engineers can use to decide whether to block a release or proceed.

Use it after every load test run on a PR or release candidate, or as part of a CI benchmark gate.

## Architecture

**Skills / Tools Used:**

| Tool / Skill | Purpose |
|---|---|
| `perf-bottleneck-analysis` | Identify bottleneck type and category for each regression |
| `perf-performance-baseline-prompt` | Generate structured baseline context for SLO comparison |

**Decision Flow:**

```
BASELINE_RESULTS + CURRENT_RESULTS + SERVICE_NAME + CHANGED_CODE_SUMMARY + SLO_THRESHOLDS
         │
         ▼
Step 1: Compute delta for each metric — (current − baseline) / baseline × 100%
         │
         ▼
Step 2: Filter regressions (delta > +5% for latency/error rate; delta < −5% for throughput)
         │
         ├─ No regressions ──▶ ✅ No regression detected — report green status
         │
         └─ Regressions found ──▶ Step 3: Classify severity
                   │
                   ├─ Critical (>50% degradation) ──▶ Block release
                   ├─ High     (20–50%)            ──▶ Block release
                   └─ Medium   (5–20%)             ──▶ Flag for investigation
                             │
                             ▼
                   Step 4: Bottleneck analysis (perf-bottleneck-analysis)
                             │
                             ▼
                   Step 5: Correlate with CHANGED_CODE_SUMMARY
                             │
                             ▼
                   Step 6: Generate ROOT_CAUSE_ANALYSIS + FIX_RECOMMENDATIONS
                             │
                             ▼
                   REGRESSION_REPORT + SEVERITY_CLASSIFICATION + ROOT_CAUSE_ANALYSIS
                   + FIX_RECOMMENDATIONS
```

## System Prompt / Agent Instructions

```
[System Prompt]

You are an autonomous performance regression analysis agent for a software engineering team.

Your goals:
1. Compare baseline and current load test results to identify performance regressions.
2. Classify each regression by severity based on the percentage degradation.
3. Identify the bottleneck type for each regression using performance engineering principles.
4. Correlate regressions with code changes to hypothesise root causes.
5. Produce prioritised, actionable fix recommendations.

Tools available to you:
- bottleneck_analysis: call with (metrics, system_description, sla_targets) → bottleneck report
- baseline_report: call with (service, metrics, slos, environment) → baseline assessment

Severity classification rules:
- Critical: >50% degradation in any metric (latency increase or throughput/error rate breach)
- High:     20–50% degradation
- Medium:   5–20% degradation
- No regression: <5% change (within noise threshold)

Rules:
- Always calculate delta as: (current − baseline) / |baseline| × 100%.
- For latency and error rate, positive delta = regression.
- For throughput, negative delta = regression.
- If no regressions are detected, state clearly: "No regression detected (all deltas < 5%)."
- When CHANGED_CODE_SUMMARY is provided, attempt to correlate each regression with a specific
  change. If no correlation can be drawn, state "Correlation unclear from available context."
- Release recommendation: Block if any Critical or High regression; Conditional if only Medium;
  Pass if no regression.

Output format:

## Performance Regression Report — {{SERVICE_NAME}}

### Metric Comparison
{table: Metric | Baseline | Current | Delta % | Regression? | Severity}

### Regression Summary
{SEVERITY_CLASSIFICATION — ordered list}

### Release Recommendation
{Block ❌ | Conditional ⚠️ | Pass ✅} — with one-sentence rationale

### Root Cause Analysis
{ROOT_CAUSE_ANALYSIS — per regression}

### Fix Recommendations
{FIX_RECOMMENDATIONS — prioritised table}
```

## Configuration

```yaml
agent_id: perf-performance-regression-agent
model: gpt-4o
temperature: 0.1          # low for consistent numeric analysis
max_iterations: 6         # compute deltas → classify → bottleneck → correlate → recommend → format
tools:
  - bottleneck_analysis
  - baseline_report
```

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `{{BASELINE_RESULTS}}` | string | Yes | Metrics from the baseline/previous run |
| `{{CURRENT_RESULTS}}` | string | Yes | Metrics from the current/candidate run |
| `{{SERVICE_NAME}}` | string | Yes | Service name |
| `{{CHANGED_CODE_SUMMARY}}` | string | No | Summary of code changes between runs |
| `{{SLO_THRESHOLDS}}` | string | No | SLO targets for additional pass/fail assessment |

## Outputs

| Name | Type | Description |
|---|---|---|
| `REGRESSION_REPORT` | string | Full regression analysis with metric comparison table |
| `SEVERITY_CLASSIFICATION` | string | Regressions ordered by severity |
| `ROOT_CAUSE_ANALYSIS` | string | Per-regression root cause hypotheses |
| `FIX_RECOMMENDATIONS` | string | Prioritised fix actions with effort estimates |

## Security Considerations

- The agent reads test results and code change summaries but never modifies code or infrastructure.
- Do not include actual secrets, credentials, or PII in `CHANGED_CODE_SUMMARY` or test result payloads.
- Route through `in-house-llm` when test results contain internal service names, endpoints, or data shapes that must not leave the organisation's network.

## Usage

### GitHub Copilot Chat (current)

```
@workspace
You are a performance regression analysis agent. Compare these two load test runs for {{SERVICE_NAME}}.

BASELINE_RESULTS: {{BASELINE_RESULTS}}
CURRENT_RESULTS: {{CURRENT_RESULTS}}
CHANGED_CODE_SUMMARY: {{CHANGED_CODE_SUMMARY}}
SLO_THRESHOLDS: {{SLO_THRESHOLDS}}
```

### API Endpoint (future portal)

```http
POST {{API_BASE_URL}}/packages/performance-engineers-analysis-pack/invoke
Content-Type: application/json

{
  "agent_id": "perf-performance-regression-agent",
  "inputs": {
    "baseline_results": "p50:95ms p95:180ms p99:420ms throughput:1200 error_rate:0.05%",
    "current_results":  "p50:98ms p95:275ms p99:890ms throughput:1180 error_rate:0.9%",
    "service_name": "checkout-api",
    "changed_code_summary": "Added synchronous inventory check call before payment processing",
    "slo_thresholds": "p99 < 500ms, error rate < 0.1%"
  }
}
```

### MCP Tool (future)

```json
{
  "tool": "perf-performance-regression-agent",
  "parameters": {
    "baseline_results": "p50:95ms p95:180ms p99:420ms throughput:1200 error_rate:0.05%",
    "current_results":  "p50:98ms p95:275ms p99:890ms throughput:1180 error_rate:0.9%",
    "service_name": "checkout-api",
    "changed_code_summary": "Added synchronous inventory check call before payment processing",
    "slo_thresholds": "p99 < 500ms, error rate < 0.1%"
  }
}
```

## Examples

### Example 1 — checkout-api sync inventory call causing latency spike

**Input:**
- `BASELINE_RESULTS`: `p50: 95ms, p95: 180ms, p99: 420ms, throughput: 1200 req/s, error_rate: 0.05%`
- `CURRENT_RESULTS`: `p50: 98ms, p95: 275ms, p99: 890ms, throughput: 1180 req/s, error_rate: 0.9%`
- `SERVICE_NAME`: `checkout-api`
- `CHANGED_CODE_SUMMARY`: `Added synchronous inventory-service HTTP call inside the payment processing transaction to validate stock before charging`
- `SLO_THRESHOLDS`: `p99 < 500ms, error rate < 0.1%`

**Output:**

```
## Performance Regression Report — checkout-api

### Metric Comparison
| Metric      | Baseline    | Current    | Delta %  | Regression? | Severity |
|---|---|---|---|---|---|
| p50 Latency | 95ms        | 98ms       | +3.2%    | No          | —        |
| p95 Latency | 180ms       | 275ms      | +52.8%   | ✅ Yes      | Critical |
| p99 Latency | 420ms       | 890ms      | +111.9%  | ✅ Yes      | Critical |
| Throughput  | 1200 req/s  | 1180 req/s | −1.7%    | No          | —        |
| Error Rate  | 0.05%       | 0.9%       | +1700%   | ✅ Yes      | Critical |

### Regression Summary
1. **Critical** — p99 Latency: +111.9% (420ms → 890ms)
2. **Critical** — p95 Latency: +52.8% (180ms → 275ms)
3. **Critical** — Error Rate: +1700% (0.05% → 0.9%)

### Release Recommendation
**Block ❌** — three Critical regressions detected; release must not proceed until resolved.

### Root Cause Analysis

**p95/p99 Latency — External Dependency (Synchronous I/O)**
The addition of a synchronous HTTP call to inventory-service inside the payment transaction
directly explains the latency increase. At p95 (275ms), the inventory-service response time
is adding ~95ms per request. At p99 (890ms), slow inventory-service tail latencies (possibly
under high concurrency) are serialised into the checkout critical path, adding ~470ms.
Correlation: Strong — the code change explicitly introduces this synchronous call.

**Error Rate — Timeout / Connection**
The 18× increase in error rate (0.05% → 0.9%) likely reflects inventory-service timeouts
or connection errors propagating as checkout failures. If inventory-service has no timeout
configured, slow responses will exhaust thread/connection pool capacity under load.
Correlation: Strong — the new synchronous call fails when inventory-service is unavailable
or slow, surfacing as a checkout error.

### Fix Recommendations
| Priority | Fix | Effort | Expected Improvement |
|---|---|---|---|
| P1 | Make inventory check asynchronous (fire-and-forget or pre-fetch before transaction) | Medium | p99 returns to ~450ms; error rate returns to baseline |
| P1 | Add a 200ms timeout + circuit breaker on the inventory-service HTTP call | Low | Error rate drops to near-baseline; latency capped at timeout |
| P2 | Cache inventory availability data (TTL 30s) to reduce per-request calls | Medium | p95 further reduced; throughput improved |
| P3 | Add retry with exponential backoff for transient inventory-service failures | Low | Error rate floor improvement under network instability |
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | 2026-04-20. Regression detection and severity classification accurate. Correlates well with code change summaries. |
| claude-3-5-sonnet | ✅ | 2026-04-20. More detailed root cause narratives. Occasionally over-classifies Medium as High. |

## Changelog

### 1.0.0 — 2026-04-20
- Initial version.
