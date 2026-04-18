---
id: cap-utilization-analysis
title: Resource Utilization Analysis
type: skill
version: "1.0.0"
status: active
workstream:
  - capacity-management
author: ai-skills-maintainer
created: 2024-01-01
updated: 2024-01-01
tags:
  - utilization
  - capacity
  - cost-optimisation
  - resources
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
  - gemini-1.5-pro
description: |
  Analyses resource utilization data (CPU, memory, storage, network) to identify
  over-provisioned and under-provisioned resources, headroom risk, and cost-saving
  opportunities with concrete recommendations.
security_classification: internal
inputs:
  - name: UTILIZATION_DATA
    type: string
    description: Paste or describe utilization metrics (average, peak, trend over time period)
    required: true
  - name: RESOURCE_TYPE
    type: string
    description: "Type of resource: compute, database, storage, network, or mixed"
    required: true
  - name: TIME_PERIOD
    type: string
    description: The time period the data covers (e.g., '30 days', 'Q4 2024')
    required: true
  - name: COST_CONTEXT
    type: string
    description: Optional cost information to support financial recommendations
    required: false
outputs:
  - name: ANALYSIS_REPORT
    type: string
    description: Utilization analysis with findings, risks, and recommendations
delivery_modes:
  - copilot-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
---

## Overview

Converts raw utilization numbers into actionable capacity and cost recommendations. Use this before quarterly planning sessions, during cost optimisation reviews, or after a significant traffic change.

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `{{UTILIZATION_DATA}}` | string | Yes | Utilization metrics |
| `{{RESOURCE_TYPE}}` | string | Yes | `compute`, `database`, `storage`, `network`, or `mixed` |
| `{{TIME_PERIOD}}` | string | Yes | Time period covered |
| `{{COST_CONTEXT}}` | string | No | Cost figures for financial analysis |

## Outputs

| Name | Type | Description |
|---|---|---|
| `ANALYSIS_REPORT` | string | Analysis with recommendations |

## Skill Definition

```
[System Prompt]

You are a capacity management engineer analysing resource utilization data to provide
actionable recommendations for right-sizing, cost optimisation, and risk mitigation.

Resource type: {{RESOURCE_TYPE}}
Time period: {{TIME_PERIOD}}
Cost context: {{COST_CONTEXT}}
Utilization data:
{{UTILIZATION_DATA}}

Produce an analysis report using this format:

## Utilization Summary
Table showing each resource with average utilization, peak utilization, trend (increasing / stable / decreasing), and headroom (remaining capacity at peak).

## Risk Findings
**RISK-NNN: [Resource Name] — [Risk description]**
- Severity: Critical / High / Medium / Low
- Finding: What the data shows.
- Impact: What happens if this is not addressed.

## Cost Optimisation Opportunities
For each over-provisioned resource:
- Current spec and utilization
- Recommended spec change
- Estimated saving

## Recommendations (Priority Order)
Numbered list of recommended actions, ordered by risk and impact.

## Next Review Trigger
State the metric thresholds that should trigger a follow-up capacity review.
```

## Usage

### GitHub Copilot Chat (current)

```
Analyse this resource utilization data for {{RESOURCE_TYPE}} over {{TIME_PERIOD}}:
{{UTILIZATION_DATA}}
```

### API Endpoint (future portal)

```http
POST {{API_BASE_URL}}/packages/capacity-management-analysis-pack/invoke
Content-Type: application/json

{
  "skill_id": "cap-utilization-analysis",
  "inputs": {
    "utilization_data": "web-server-01: avg 35%, peak 92% (spikes every Monday 09:00), trend increasing +5%/week",
    "resource_type": "compute",
    "time_period": "30 days",
    "cost_context": "Current: 4 vCPU / 16 GB at $200/month"
  }
}
```

## Examples

### Example 1

**Input:** Web server cluster with growing Monday morning spikes.

**Output (excerpt):**
```
## Utilization Summary
| Resource | Avg CPU | Peak CPU | Trend | Headroom at Peak |
|---|---|---|---|---|
| web-server-01 | 35% | 92% | +5%/week | 8% |

## Risk Findings
**RISK-001: web-server-01 — Headroom exhaustion in ~3 weeks**
- Severity: High
- Finding: Peak CPU at 92% with a +5%/week growth trend.
- Impact: Service degradation or outage within 3 weeks without intervention.

## Recommendations
1. Add 2 additional nodes to the web server cluster before next Monday.
2. Investigate and optimise the Monday morning batch job driving the spikes.
3. Set an auto-scaling policy with scale-out trigger at 75% CPU.
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | 2024-01-01. Good at identifying trends from narrative data. |
| claude-3-5-sonnet | ✅ | 2024-01-01. Excellent at cost calculations when figures are provided. |

## Changelog

### 1.0.0 — 2024-01-01
- Initial version.
