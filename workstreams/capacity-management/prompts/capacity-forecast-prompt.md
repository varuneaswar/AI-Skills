---
id: cap-capacity-forecast-prompt
title: Capacity Forecast Report Generator
type: prompt
version: "1.0.0"
status: active
workstream:
  - capacity-management
author: ai-skills-maintainer
created: 2026-04-20
updated: 2026-04-20
tags:
  - capacity
  - forecasting
  - scaling
  - infrastructure
  - planning
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
  - copilot-gpt-4o
description: |
  Generates a 30/60/90-day capacity forecast from current utilization metrics, weekly
  growth trend data, and optional seasonality context — producing projected breach dates,
  headroom timelines, and prioritised scaling recommendations in a structured Markdown table report.
security_classification: internal
delivery_modes:
  - copilot-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
inputs:
  - name: SERVICE_NAME
    type: string
    description: Name of the service or resource group being forecast
    required: true
  - name: CURRENT_METRICS
    type: string
    description: "Current utilization snapshot: CPU %, memory %, storage %, requests/s"
    required: true
  - name: GROWTH_TREND
    type: string
    description: "Weekly growth rate, e.g., '+3% CPU/week, +1.5% memory/week'"
    required: true
  - name: RESOURCE_TYPE
    type: string
    description: "Primary resource to forecast: cpu | memory | storage | requests"
    required: true
  - name: SEASONALITY_NOTES
    type: string
    description: "Optional: known seasonal patterns, e.g., 'Black Friday spike 3×', 'end-of-month batch'"
    required: false
outputs:
  - name: CAPACITY_FORECAST
    type: string
    description: Structured 30/60/90-day forecast report with breach dates and scaling actions
---

## Overview

Poorly timed capacity upgrades lead to either costly over-provisioning or unplanned outages. This prompt transforms current utilization snapshots and growth trends into a forward-looking forecast so capacity managers can act before resources are exhausted.

Use it before quarterly planning sessions, ahead of known traffic events, or as part of a weekly capacity review cycle.

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `{{SERVICE_NAME}}` | string | Yes | Service or resource group name |
| `{{CURRENT_METRICS}}` | string | Yes | Current CPU %, memory %, storage %, throughput |
| `{{GROWTH_TREND}}` | string | Yes | Weekly growth rate per resource dimension |
| `{{RESOURCE_TYPE}}` | string | Yes | `cpu`, `memory`, `storage`, or `requests` |
| `{{SEASONALITY_NOTES}}` | string | No | Known seasonal patterns or upcoming traffic events |

## Prompt

```
[System]
You are a capacity planning specialist. Your role is to analyse resource utilization trends
and produce accurate, actionable capacity forecasts that help engineering teams avoid
resource exhaustion and plan scaling activities in advance.

[Task]
Produce a 30/60/90-day capacity forecast for {{SERVICE_NAME}} based on the metrics and
growth trend provided. Assume linear growth unless SEASONALITY_NOTES indicates otherwise.
Apply seasonal multipliers to the affected forecast window when seasonality is noted.

Service: {{SERVICE_NAME}}
Current metrics: {{CURRENT_METRICS}}
Weekly growth trend: {{GROWTH_TREND}}
Primary resource type: {{RESOURCE_TYPE}}
Seasonality notes: {{SEASONALITY_NOTES}}

[Rules]
1. Use 80% utilization as the warning threshold and 90% as the critical breach threshold.
2. Calculate projected utilization at 30, 60, and 90 days from today using the growth rate.
3. If a metric will breach the 90% threshold within 90 days, mark the row as BREACH RISK.
4. For each BREACH RISK resource, provide at least two scaling options (vertical and horizontal).
5. Express breach dates as "~N weeks" rather than calendar dates to stay tool-agnostic.
6. If SEASONALITY_NOTES is provided, add a separate row for the peak seasonal scenario.
7. Keep all numeric projections rounded to one decimal place.

[Output Format]
Respond with a Markdown report using exactly the structure below.

## Capacity Forecast — {{SERVICE_NAME}}

### 30/60/90-Day Projection

| Resource | Now | +30 days | +60 days | +90 days | Breach? | Weeks to Breach |
|---|---|---|---|---|---|---|
| CPU | | | | | ✅ Safe / ⚠️ Warning / 🔴 Breach | — or ~N wks |
| Memory | | | | | | |
| Storage | | | | | | |
| Requests/s | | | | | | |

*(Add a SEASONAL PEAK row if SEASONALITY_NOTES is provided.)*

### Breach Risk Summary
For each 🔴 Breach row, produce one section:

**[Resource] — Breach in ~N weeks**
- Current: X%
- Projected at breach: 90%
- Risk: [description of impact if not addressed]

### Scaling Recommendations (Priority Order)
| Priority | Resource | Action | Type | Estimated Lead Time | Expected Headroom Gained |
|---|---|---|---|---|---|
| P1 | | | Vertical / Horizontal / Config | | |

### Immediate Actions (if any breach within 30 days)
- Bullet list of urgent steps.

### Next Review Trigger
State the metric thresholds that should trigger a follow-up capacity forecast.
```

## Usage

### GitHub Copilot Chat (current)

```
Generate a capacity forecast for {{SERVICE_NAME}}.
Current metrics: {{CURRENT_METRICS}}
Weekly growth: {{GROWTH_TREND}}
Resource type: {{RESOURCE_TYPE}}
Seasonality: {{SEASONALITY_NOTES}}
```

### Standalone (any LLM — current)

1. Copy the prompt above and replace all `{{placeholders}}` with real values.
2. Paste into your preferred LLM interface.
3. Save the output as `capacity-forecast-<service>-<date>.md`.

### API Endpoint (future portal)

```http
POST {{API_BASE_URL}}/packages/capacity-management-analysis-pack/invoke
Content-Type: application/json

{
  "prompt_id": "cap-capacity-forecast-prompt",
  "inputs": {
    "service_name": "order-service",
    "current_metrics": "CPU 58%, memory 61%, storage 44%, requests 4200 req/s",
    "growth_trend": "+4% CPU/week, +2% memory/week, +1% storage/week, +300 req/s/week",
    "resource_type": "cpu",
    "seasonality_notes": "Black Friday (week 47): expected 3× normal traffic for 5 days"
  }
}
```

## Examples

### Example 1 — order-service pre-Black Friday

**Input:**
- `SERVICE_NAME`: `order-service`
- `CURRENT_METRICS`: `CPU 58%, memory 61%, storage 44%, requests 4200 req/s`
- `GROWTH_TREND`: `+4% CPU/week, +2% memory/week, +1% storage/week, +300 req/s/week`
- `RESOURCE_TYPE`: `cpu`
- `SEASONALITY_NOTES`: `Black Friday (week 8 from now): expected 3× normal traffic for 5 days`

**Output:**

```
## Capacity Forecast — order-service

### 30/60/90-Day Projection

| Resource | Now | +30 days | +60 days | +90 days | Breach? | Weeks to Breach |
|---|---|---|---|---|---|---|
| CPU | 58% | 74% | 90% | 106%* | 🔴 Breach | ~9 wks |
| Memory | 61% | 69% | 77% | 85% | ⚠️ Warning | — |
| Storage | 44% | 48% | 52% | 57% | ✅ Safe | — |
| Requests/s | 4200 | 5500 | 6800 | 8100 | ⚠️ Warning | — |
| CPU (Black Friday peak) | 58% | **174%*** | — | — | 🔴 Breach | ~8 wks (event) |

*Values above 100% indicate projected overload — actual service degradation begins at 90%.

### Breach Risk Summary

**CPU — Breach in ~9 weeks**
- Current: 58%
- Projected at breach: 90%
- Risk: CPU saturation causes request queuing, latency spikes, and potential pod evictions
  in containerised environments. Black Friday seasonality brings the effective breach
  forward to ~8 weeks.

### Scaling Recommendations (Priority Order)

| Priority | Resource | Action | Type | Estimated Lead Time | Expected Headroom Gained |
|---|---|---|---|---|---|
| P1 | CPU | Scale out: add 4 additional pods (2 vCPU each) | Horizontal | 1 day | ~25% headroom at current load |
| P1 | CPU | Pre-scale for Black Friday: set min replicas to 12 before week 8 | Horizontal | Scheduled | Handles 3× traffic spike |
| P2 | Memory | Increase pod memory limit from 2 GB to 3 GB | Vertical | 1 day | 15% headroom |
| P3 | Requests/s | Enable autoscaler HPA target at 70% CPU | Config | 2 hours | Automatic headroom management |

### Immediate Actions (breach within 30 days via seasonality)
- Open capacity ticket NOW for Black Friday pre-scaling (8 weeks lead time needed).
- Schedule HPA configuration review this sprint.

### Next Review Trigger
Re-run this forecast if CPU exceeds 70%, memory exceeds 75%, or requests/s exceed 7000.
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | 2026-04-20. Accurate linear projections. Seasonal multiplier applied correctly. |
| claude-3-5-sonnet | ✅ | 2026-04-20. More detailed breach impact descriptions. Occasionally adds extra context rows. |

## Changelog

### 1.0.0 — 2026-04-20
- Initial version.
