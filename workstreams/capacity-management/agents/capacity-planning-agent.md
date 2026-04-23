---
id: cap-capacity-planning-agent
title: Capacity Planning Agent
type: agent
version: "1.0.0"
status: active
workstream:
  - capacity-management
author: ai-skills-maintainer
created: 2026-04-20
updated: 2026-04-20
tags:
  - capacity
  - planning
  - forecasting
  - scaling
  - automation
  - agent
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
description: |
  An autonomous agent that ingests a fleet-wide utilization report, forecasts capacity
  breaches for each service over a 30/60/90-day horizon, identifies at-risk services,
  and produces a prioritised scaling recommendation report with draft Jira ticket bodies
  for P1 services — reducing manual capacity review effort across large service fleets.
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
  - name: UTILIZATION_REPORT
    type: string
    description: Fleet-wide utilization report (JSON or structured text) with per-service CPU/memory/storage/requests metrics and weekly growth rates
    required: true
  - name: SERVICES_IN_SCOPE
    type: string
    description: Comma-separated list of service names to analyse; use "all" to process every service in the report
    required: true
  - name: FORECAST_HORIZON
    type: string
    description: "Forecast window: 30 | 60 | 90 (days). Default: 90"
    required: false
  - name: COST_CONSTRAINTS
    type: string
    description: "Optional: budget ceiling or cost targets to factor into scaling recommendations"
    required: false
outputs:
  - name: CAPACITY_REPORT
    type: string
    description: Full capacity planning report with per-service forecasts and fleet-wide summary
  - name: BREACH_RISK_LIST
    type: string
    description: Ordered list of services at breach risk within the forecast horizon, by severity
  - name: SCALING_RECOMMENDATIONS
    type: string
    description: Prioritised scaling action table covering all at-risk services
  - name: PLANNING_TICKETS
    type: string
    description: Draft Jira ticket body (Markdown) for each P1 breach-risk service
---

## Overview

The Capacity Planning Agent replaces the manual step of running individual forecasts across a service fleet. It processes all services in a single pass, correlates utilization patterns, and surfaces the highest-risk services first — so capacity managers focus attention where it matters most.

Use it weekly as part of capacity review cycles or before major traffic events such as product launches or seasonal peaks.

## Architecture

**Skills / Tools Used:**

| Tool / Skill | Purpose |
|---|---|
| `cap-utilization-analysis` | Analyse current utilization per service and identify hotspots |
| `cap-capacity-forecast-prompt` | Project 30/60/90-day breach dates per service |

**Decision Flow:**

```
UTILIZATION_REPORT + SERVICES_IN_SCOPE + FORECAST_HORIZON + COST_CONSTRAINTS
         │
         ▼
Step 1: Parse utilization report → per-service metric snapshots
         │
         ▼
Step 2: For each service → run utilization analysis (cap-utilization-analysis)
         │
         ▼
Step 3: For each service → run capacity forecast (cap-capacity-forecast-prompt)
         │
         ├─ Breach within 30 days ──▶ P1 — draft Jira ticket, flag IMMEDIATE ACTION
         │
         ├─ Breach within 31–60 days ──▶ P2 — add to scaling recommendations
         │
         ├─ Breach within 61–90 days ──▶ P3 — add to watch list
         │
         └─ No breach within horizon ──▶ ✅ Healthy — record in fleet summary
                                              │
                                              ▼
                             Step 4: Synthesise CAPACITY_REPORT
                                     + BREACH_RISK_LIST
                                     + SCALING_RECOMMENDATIONS
                                     + PLANNING_TICKETS
```

## System Prompt / Agent Instructions

```
[System Prompt]

You are an autonomous capacity planning agent for a service fleet.

Your goals:
1. Parse the fleet-wide utilization report and extract per-service metrics.
2. For each service in scope, analyse current utilization and identify hotspots (>60%).
3. Forecast capacity breaches at 30, 60, and 90 days using linear growth rates.
4. Classify breach risk: P1 (≤30 days), P2 (31–60 days), P3 (61–90 days).
5. Produce a consolidated capacity planning report, breach risk list, scaling recommendations,
   and draft Jira for P1 services.

Tools available to you:
- utilization_analysis: call with (service, metrics, resource_type, time_period) → analysis
- capacity_forecast: call with (service, metrics, growth_trend, resource_type) → forecast

Rules:
- Process every service in SERVICES_IN_SCOPE (or all services if "all" is specified).
- Use 80% as the warning threshold and 90% as the critical breach threshold.
- When COST_CONSTRAINTS are provided, prefer horizontal scaling options that fit within budget.
- P1 services must always include a draft Jira ticket body in PLANNING_TICKETS.
- Do not fabricate metrics — if a service's data is missing a dimension, note it as "data unavailable".
- Order BREACH_RISK_LIST by time-to-breach ascending (most urgent first).
- Scaling recommendations must include estimated lead time so teams can plan procurement or deployments.

Output format:

## Fleet Capacity Report

### Executive Summary
{fleet-level summary: total services assessed, P1/P2/P3 counts, recommended immediate actions}

### Per-Service Forecasts
{one sub-section per service with utilization snapshot and breach projection table}

### Breach Risk List
{ordered table: Service | Priority | Resource | Breach In | Recommended Action}

### Scaling Recommendations
{prioritised action table for all at-risk services}

### Planning Tickets (P1 services)
{one Jira ticket draft per P1 service}

### Fleet Health Summary
{table: Service | Status | Next Review Trigger}
```

## Configuration

```yaml
agent_id: cap-capacity-planning-agent
model: gpt-4o
temperature: 0.1          # low for consistent numeric calculations
max_iterations: 10        # parse → analyse × N services → synthesise → format
tools:
  - utilization_analysis
  - capacity_forecast
```

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `{{UTILIZATION_REPORT}}` | string | Yes | Fleet-wide report with per-service metrics and growth rates |
| `{{SERVICES_IN_SCOPE}}` | string | Yes | Service list or `"all"` |
| `{{FORECAST_HORIZON}}` | string | No | `30`, `60`, or `90` days (default: `90`) |
| `{{COST_CONSTRAINTS}}` | string | No | Budget ceiling or cost targets |

## Outputs

| Name | Type | Description |
|---|---|---|
| `CAPACITY_REPORT` | string | Full fleet capacity planning report |
| `BREACH_RISK_LIST` | string | Services at breach risk, ordered by urgency |
| `SCALING_RECOMMENDATIONS` | string | Prioritised action table for at-risk services |
| `PLANNING_TICKETS` | string | Draft Jira ticket bodies for P1 services |

## Security Considerations

- The agent reads utilization metrics but never modifies infrastructure directly.
- Utilization data may include internal service names and capacity figures — route through `in-house-llm` for confidential fleet data.
- Draft Jira contain service names and capacity details; review before posting to ensure no sensitive data is exposed in public repositories.
- `COST_CONSTRAINTS` may contain budget figures — treat as confidential and do not log.

## Usage

### LLM Chat (current)

```
@workspace
You are a capacity planning agent. Analyse this fleet utilization report and produce
a 90-day capacity forecast with breach risk list and scaling recommendations.

UTILIZATION_REPORT:
{{UTILIZATION_REPORT}}

SERVICES_IN_SCOPE: {{SERVICES_IN_SCOPE}}
FORECAST_HORIZON: 90
COST_CONSTRAINTS: {{COST_CONSTRAINTS}}
```

### API Endpoint (future portal)

```http
POST {{API_BASE_URL}}/packages/capacity-management-analysis-pack/invoke
Content-Type: application/json

{
  "agent_id": "cap-capacity-planning-agent",
  "inputs": {
    "utilization_report": "{\"order-service\":{\"cpu\":58,\"memory\":61,\"growth_cpu\":4,\"growth_mem\":2},\"payment-service\":{\"cpu\":82,\"memory\":74,\"growth_cpu\":5,\"growth_mem\":3}}",
    "services_in_scope": "all",
    "forecast_horizon": "90",
    "cost_constraints": "Max $5,000/month additional spend"
  }
}
```

### MCP Tool (future)

```json
{
  "tool": "cap-capacity-planning-agent",
  "parameters": {
    "utilization_report": "...",
    "services_in_scope": "order-service,payment-service",
    "forecast_horizon": "90",
    "cost_constraints": "Max $5,000/month additional spend"
  }
}
```

## Examples

### Example 1 — order-service + payment-service fleet

**Input:**
- `UTILIZATION_REPORT`:
  ```json
  {
    "order-service":   { "cpu": 58, "memory": 61, "storage": 44, "growth_cpu": 4, "growth_mem": 2 },
    "payment-service": { "cpu": 82, "memory": 74, "storage": 51, "growth_cpu": 5, "growth_mem": 3 }
  }
  ```
- `SERVICES_IN_SCOPE`: `all`
- `FORECAST_HORIZON`: `90`
- `COST_CONSTRAINTS`: `Max $5,000/month additional spend`

**Output (excerpt):**

```
## Fleet Capacity Report

### Executive Summary
- Services assessed: 2
- P1 (breach ≤30 days): 1 — payment-service (CPU)
- P2 (breach 31–60 days): 1 — order-service (CPU)
- Immediate action required: Scale payment-service CPU within 7 days.

### Per-Service Forecasts

#### order-service
| Resource | Now  | +30d | +60d | +90d | Breach? | Weeks to Breach |
|---|---|---|---|---|---|---|
| CPU      | 58%  | 74%  | 90%  | 106% | 🔴 Breach | ~9 wks |
| Memory   | 61%  | 69%  | 77%  | 85%  | ⚠️ Warning | — |

#### payment-service
| Resource | Now  | +30d | +60d | +90d | Breach? | Weeks to Breach |
|---|---|---|---|---|---|---|
| CPU      | 82%  | 102% | — | — | 🔴 Breach | ~2 wks |
| Memory   | 74%  | 86%  | 98% | — | 🔴 Breach | ~7 wks |

### Breach Risk List
| Service | Priority | Resource | Breach In | Recommended Action |
|---|---|---|---|---|
| payment-service | P1 | CPU | ~2 weeks | Scale out: +4 pods immediately |
| payment-service | P1 | Memory | ~7 weeks | Increase pod memory limit |
| order-service   | P2 | CPU | ~9 weeks | Scale out: +4 pods this sprint |

### Planning Tickets (P1 services)

**payment-service — CPU Breach Risk (P1)**

```markdown
## Capacity Alert — payment-service CPU

**Priority:** P1 — Breach in ~2 weeks
**Resource:** CPU (current: 82%, breach threshold: 90%)
**Growth rate:** +5%/week

### Impact
CPU saturation will cause request queuing and latency spikes within 2 weeks.

### Recommended Actions
1. Scale out: add 4 pods (2 vCPU each) immediately — estimated headroom: +20%
2. Configure HPA with CPU target at 75% to automate future scaling

### Acceptance Criteria
- [ ] CPU utilization below 75% after scaling
- [ ] HPA configured and tested
- [ ] Capacity re-forecast completed
```
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | 2026-04-20. Handles 2-service fleet correctly. Breach dates accurate. |
| claude-3-5-sonnet | ✅ | 2026-04-20. More detailed Jira ticket bodies. Slightly longer output. |

## Changelog

### 1.0.0 — 2026-04-20
- Initial version.
