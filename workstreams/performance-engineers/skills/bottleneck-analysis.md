---
id: perf-bottleneck-analysis
title: Performance Bottleneck Analysis
type: skill
version: "1.0.0"
status: active
workstream:
  - performance-engineers
author: ai-skills-maintainer
created: 2024-01-01
updated: 2024-01-01
tags:
  - performance
  - bottleneck
  - profiling
  - latency
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
  - gemini-1.5-pro
description: |
  Analyses load test results, profiling data, or performance metrics to identify bottlenecks,
  classify them by root-cause category, and provide prioritised remediation recommendations.
security_classification: internal
inputs:
  - name: PERFORMANCE_DATA
    type: string
    description: Load test results, profiler output, or performance metrics (paste as text)
    required: true
  - name: SYSTEM_DESCRIPTION
    type: string
    description: Brief description of the system under test and its architecture
    required: true
  - name: SLA_TARGETS
    type: string
    description: "Performance targets, e.g., 'P99 < 500ms, throughput > 1000 req/s'"
    required: false
outputs:
  - name: BOTTLENECK_REPORT
    type: string
    description: Classified bottleneck findings with remediation steps
delivery_modes:
  - copilot-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
---

## Overview

Converts raw performance test output into a structured diagnosis. Particularly useful after a load test run to quickly understand which component is the limiting factor before diving into deeper profiling.

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `{{PERFORMANCE_DATA}}` | string | Yes | Test results or profiling data |
| `{{SYSTEM_DESCRIPTION}}` | string | Yes | Architecture overview |
| `{{SLA_TARGETS}}` | string | No | Target thresholds for pass/fail assessment |

## Outputs

| Name | Type | Description |
|---|---|---|
| `BOTTLENECK_REPORT` | string | Bottleneck findings with remediation |

## Skill Definition

```
[System Prompt]

You are a performance engineering specialist. Your job is to read performance data and
identify the primary bottlenecks, classify their root causes, and recommend actionable fixes.

System under test: {{SYSTEM_DESCRIPTION}}
SLA targets: {{SLA_TARGETS}}
Performance data:
{{PERFORMANCE_DATA}}

Produce a bottleneck analysis report:

## SLA Assessment
| Metric | Target | Actual | Status |
|---|---|---|---|
| P50 latency | | | ✅/❌ |
| P99 latency | | | ✅/❌ |
| Throughput | | | ✅/❌ |
| Error rate | | | ✅/❌ |

## Identified Bottlenecks

**BOTTLENECK-NNN: [Short title]**
- Category: CPU / Memory / I/O / Network / Database / Application-Logic / External-Dependency
- Evidence: What in the data points to this bottleneck.
- Impact: Which SLA metrics are affected.
- Recommended fix: Specific, actionable remediation steps.
- Effort: Low / Medium / High
- Expected improvement: Estimated impact after fix.

## Recommended Investigation Order
Numbered priority order for addressing the bottlenecks.

## Quick Wins
List any fixes that are Low effort with High impact.
```

## Usage

### GitHub Copilot Chat (current)

```
Analyse this performance test output for {{SYSTEM_DESCRIPTION}}.
SLA targets: {{SLA_TARGETS}}
Data:
{{PERFORMANCE_DATA}}
```

### API Endpoint (future portal)

```http
POST {{API_BASE_URL}}/packages/performance-engineers-analysis-pack/invoke
Content-Type: application/json

{
  "skill_id": "perf-bottleneck-analysis",
  "inputs": {
    "performance_data": "P50: 120ms, P99: 4200ms, errors: 3.2% at 500 VU. DB query time avg 3800ms.",
    "system_description": "3-tier web app: Nginx → Node.js → PostgreSQL",
    "sla_targets": "P99 < 500ms, error rate < 0.1%"
  }
}
```

## Examples

### Example 1

**Output (excerpt):**
```
## SLA Assessment
| Metric | Target | Actual | Status |
|---|---|---|---|
| P99 latency | < 500ms | 4200ms | ❌ |
| Error rate | < 0.1% | 3.2% | ❌ |

**BOTTLENECK-001: Database query latency**
- Category: Database
- Evidence: DB query time avg 3800ms accounts for 90% of P99 latency.
- Recommended fix:
  1. EXPLAIN ANALYZE the slowest queries.
  2. Add indexes on columns used in WHERE clauses.
  3. Consider query result caching for read-heavy queries.
- Effort: Medium
- Expected improvement: P99 could drop to <600ms with index optimisation alone.
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | 2024-01-01. Strong at database and network bottleneck identification. |
| claude-3-5-sonnet | ✅ | 2024-01-01. More detailed remediation steps. |

## Changelog

### 1.0.0 — 2024-01-01
- Initial version.
