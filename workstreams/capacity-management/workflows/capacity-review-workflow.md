---
id: cap-capacity-review-workflow
title: Capacity Review Workflow
type: workflow
version: "1.0.0"
status: active
workstream:
  - capacity-management
author: ai-skills-maintainer
created: 2026-04-20
updated: 2026-04-20
tags:
  - capacity
  - review
  - workflow
  - forecasting
  - scaling
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
description: |
  A four-step workflow that takes raw utilization data for a set of services through
  hotspot identification, 30/60/90-day capacity projection, scaling option evaluation,
  and consolidated report generation — producing an actionable capacity review report
  with a prioritised action plan ready for stakeholder review.
security_classification: internal
delivery_modes:
  - llm-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
dependencies:
  - cap-utilization-analysis
  - cap-capacity-forecast-prompt
inputs:
  - name: UTILIZATION_DATA
    type: string
    description: Current utilization metrics for all services in scope (CPU/memory/storage/requests + growth rates)
    required: true
  - name: SERVICES_LIST
    type: string
    description: Comma-separated list of service names to include in the review
    required: true
  - name: FORECAST_HORIZON
    type: string
    description: "Forecast window in days: 30 | 60 | 90"
    required: true
  - name: COST_BUDGET
    type: string
    description: "Optional: available budget for scaling actions (e.g., '$3,000/month')"
    required: false
outputs:
  - name: CAPACITY_REVIEW_REPORT
    type: string
    description: Consolidated capacity review report with hotspot findings, forecasts, scaling options, and action plan
---

## Overview

This workflow orchestrates the end-to-end capacity review process. Capacity managers run it before quarterly planning sessions, before major traffic events, or on a regular weekly cycle to maintain fleet health visibility.

**Audience:** Capacity management engineers, infrastructure leads, SREs.

## Steps

### Step 1 — Analyse Current Utilization and Identify Hotspots

**Asset used:** `cap-utilization-analysis`  
**Input:** `{{UTILIZATION_DATA}}`, `{{SERVICES_LIST}}`  
**Output:** `HOTSPOT_ANALYSIS` — per-service utilization summary with resources above 60% flagged.

```
[Prompt — paste into any LLM]

You are a capacity management engineer analysing resource utilization data.

Services in scope: {{SERVICES_LIST}}
Utilization data:
{{UTILIZATION_DATA}}

Analyse the utilization data and produce:

## Utilization Summary
Table with columns: Service | Resource | Avg Utilization | Peak Utilization | Trend | Headroom | Status
Mark Status as: ✅ Healthy (<60%) | ⚠️ Hotspot (60–79%) | 🔴 Critical (≥80%)

## Hotspot Findings
For each ⚠️ or 🔴 service/resource:
- Service and resource name
- Current utilization and trend
- Risk if left unaddressed

## Services Requiring Forecast
List all services with any ⚠️ or 🔴 resources — these proceed to Step 2.
```

---

### Step 2 — Project Capacity Needs for 30/60/90 Days

**Asset used:** `cap-capacity-forecast-prompt`  
**Input:** `HOTSPOT_ANALYSIS` (Step 1 output), `{{FORECAST_HORIZON}}`  
**Output:** `CAPACITY_FORECASTS` — per-service breach projections for the forecast horizon.

```
[Prompt — paste into any LLM after Step 1]

You are a capacity planning specialist. Produce capacity forecasts for the hotspot
services identified in Step 1.

Hotspot services and their metrics:
{{HOTSPOT_ANALYSIS}}

Forecast horizon: {{FORECAST_HORIZON}} days

For each hotspot service, produce:

### [Service Name] — Capacity Forecast

| Resource | Now | +30 days | +60 days | +90 days | Breach? | Weeks to Breach |
|---|---|---|---|---|---|---|

Rules:
- Use 80% as warning threshold, 90% as breach threshold.
- Apply linear growth rate from the trend in HOTSPOT_ANALYSIS.
- Flag any resource breaching within the FORECAST_HORIZON as BREACH RISK.
- If growth rate is not available for a resource, note "trend data unavailable".
```

---

### Step 3 — Evaluate Scaling Options and Costs

**Asset used:** *(inline prompt)*  
**Input:** `CAPACITY_FORECASTS` (Step 2 output), `{{COST_BUDGET}}`  
**Output:** `SCALING_OPTIONS` — scaling options with effort, cost, and lead time for each at-risk resource.

```
[Prompt — paste into any LLM after Step 2]

You are a cloud infrastructure architect evaluating scaling options for at-risk services.

Capacity forecasts with breach risks:
{{CAPACITY_FORECASTS}}

Available cost budget: {{COST_BUDGET}}

For each service and resource marked as BREACH RISK, produce:

### [Service Name] — Scaling Options

| Option | Type | Action | Estimated Cost | Lead Time | Headroom Gained | Fits Budget? |
|---|---|---|---|---|---|---|
| 1 | Horizontal | | $/month | days | % | ✅/❌ |
| 2 | Vertical   | | $/month | days | % | ✅/❌ |

Recommendation: State which option is preferred and why, considering cost, lead time, and risk.

If COST_BUDGET is not provided, omit the "Fits Budget?" column and cost estimates.
```

---

### Step 4 — Produce Capacity Review Report with Action Plan

**Asset used:** *(inline prompt)*  
**Input:** `HOTSPOT_ANALYSIS` (Step 1), `CAPACITY_FORECASTS` (Step 2), `SCALING_OPTIONS` (Step 3)  
**Output:** `CAPACITY_REVIEW_REPORT`

```
[Prompt — paste into any LLM after Steps 1–3]

Combine the outputs from the three previous steps into a single, well-structured
Capacity Review Report ready for stakeholder presentation.

Hotspot analysis:
{{HOTSPOT_ANALYSIS}}

Capacity forecasts:
{{CAPACITY_FORECASTS}}

Scaling options:
{{SCALING_OPTIONS}}

Produce the following report structure:

## Capacity Review Report — {{SERVICES_LIST}}
**Review Date:** [today's date]
**Forecast Horizon:** {{FORECAST_HORIZON}} days

### Executive Summary
2–3 sentences: how many services reviewed, how many at risk, and the most urgent action.

### Fleet Status
Table: Service | Overall Status | Most Urgent Resource | Action Required By

### Detailed Findings
(Include the per-service utilization tables, forecast tables, and scaling options from Steps 1–3.)

### Prioritised Action Plan
| Priority | Service | Resource | Action | Owner | Due Date |
|---|---|---|---|---|---|
| P1 | | | | | Within 1 week |
| P2 | | | | | Within 30 days |
| P3 | | | | | Within 90 days |

### Next Capacity Review
State the conditions (metric thresholds or calendar date) that should trigger the next review.
```

---

## Flow Diagram

```
UTILIZATION_DATA + SERVICES_LIST + FORECAST_HORIZON + COST_BUDGET
         │
         ▼
[Step 1: Utilization Analysis] ──▶ HOTSPOT_ANALYSIS
         │
         ▼
[Step 2: Capacity Forecast] ──▶ CAPACITY_FORECASTS
         │
         ▼
[Step 3: Scaling Option Evaluation] ──▶ SCALING_OPTIONS
         │
         ▼
[Step 4: Consolidate Report] ──▶ CAPACITY_REVIEW_REPORT
```

## Usage

### Manual (Copilot Chat — current)

1. Gather current utilization data for all services in scope.
2. Copy each step prompt in order, substitute the placeholders, and paste into LLM Chat or your preferred LLM.
3. Save each step's output to use as input for the next step.
4. Share the final `CAPACITY_REVIEW_REPORT` with stakeholders or attach it to the capacity review ticket.

### Automated (Bitbucket Pipelines — current)

See `workstreams/capacity-management/automations/scaling-recommendation-automation.md` for a ready-to-use Bitbucket Pipelines implementation that runs this workflow on a weekly schedule.

### API Endpoint (future portal)

```http
POST {{API_BASE_URL}}/packages/capacity-management-analysis-pack/invoke
Content-Type: application/json

{
  "workflow_id": "cap-capacity-review-workflow",
  "inputs": {
    "utilization_data": "order-service: CPU 58% avg, 74% peak, +4%/week...",
    "services_list": "order-service,payment-service",
    "forecast_horizon": "90",
    "cost_budget": "$5,000/month"
  }
}
```

## Examples

### Example 1 — 2-service fleet review (order-service + payment-service)

**Input:**
- `UTILIZATION_DATA`:
  ```
  order-service:   CPU avg 58%, peak 74%, trend +4%/week; memory avg 61%, peak 68%, trend +2%/week
  payment-service: CPU avg 82%, peak 88%, trend +5%/week; memory avg 74%, peak 80%, trend +3%/week
  ```
- `SERVICES_LIST`: `order-service, payment-service`
- `FORECAST_HORIZON`: `90`
- `COST_BUDGET`: `$5,000/month`

**Step 1 Output (HOTSPOT_ANALYSIS excerpt):**
```
## Utilization Summary
| Service         | Resource | Avg  | Peak | Trend     | Headroom | Status |
|---|---|---|---|---|---|---|
| order-service   | CPU      | 58%  | 74%  | +4%/week  | 26%      | ⚠️ Hotspot |
| payment-service | CPU      | 82%  | 88%  | +5%/week  | 12%      | 🔴 Critical |
| payment-service | Memory   | 74%  | 80%  | +3%/week  | 20%      | ⚠️ Hotspot |
```

**Step 4 Output (CAPACITY_REVIEW_REPORT excerpt):**
```
## Capacity Review Report — order-service, payment-service
**Review Date:** 2026-04-20
**Forecast Horizon:** 90 days

### Executive Summary
Two services reviewed; both require scaling actions. payment-service CPU is critical
with a breach projected in ~2 weeks — immediate action required. order-service CPU
will breach in ~9 weeks and should be addressed this sprint.

### Prioritised Action Plan
| Priority | Service         | Resource | Action                | Owner | Due Date       |
|---|---|---|---|---|---|
| P1       | payment-service | CPU      | Scale out +4 pods     | SRE   | Within 1 week  |
| P2       | order-service   | CPU      | Scale out +4 pods     | SRE   | Within 30 days |
| P3       | payment-service | Memory   | Increase memory limit | SRE   | Within 60 days |
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | 2026-04-20. All four steps work in a single Copilot Chat session. |
| claude-3-5-sonnet | ✅ | 2026-04-20. Step 3 produces more detailed cost breakdowns when budget is provided. |

## Changelog

### 1.0.0 — 2026-04-20
- Initial version.
