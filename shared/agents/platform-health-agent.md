---
id: shared-platform-health-agent
title: Platform Health Assessment Agent
type: agent
version: "1.0.0"
status: active
workstream:
  - sre
  - performance-engineers
  - capacity-management
author: ai-skills-maintainer
created: 2026-04-20
updated: 2026-04-20
tags:
  - platform-health
  - cross-workstream
  - sre
  - performance
  - capacity
  - agent
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
  - copilot-gpt-4o
description: |
  An autonomous cross-workstream agent that assesses overall platform health by combining
  SRE incident awareness, performance regression detection, and capacity utilisation
  forecasting into a single unified Platform Health Report. Each service is scored 🟢/🟡/🔴
  and the report concludes with prioritised recommended actions across all three disciplines.
security_classification: internal
delivery_modes:
  - copilot-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
dependencies:
  - sre-incident-triage-prompt
  - sre-runbook-generator
  - perf-bottleneck-analysis
  - perf-performance-baseline-prompt
  - cap-utilization-analysis
  - cap-capacity-forecast-prompt
inputs:
  - name: PLATFORM_SUMMARY
    type: string
    description: List of services with their recent alert counts and health statuses (e.g. CSV or YAML block)
    required: true
  - name: PERFORMANCE_METRICS
    type: string
    description: Recent performance data — p50/p95/p99 latency, error rates, throughput per service
    required: true
  - name: UTILIZATION_DATA
    type: string
    description: Current CPU, memory, and storage utilisation per service or node
    required: true
  - name: INCIDENT_HISTORY
    type: string
    description: Optional — list of incidents in the last 7–30 days with severity and MTTR
    required: false
  - name: FORECAST_HORIZON
    type: string
    description: "Optional — capacity forecast window: 30, 60, or 90 (days). Defaults to 30."
    required: false
outputs:
  - name: PLATFORM_HEALTH_REPORT
    type: string
    description: Full structured report with per-service health scores, findings, and analysis across all three workstreams
  - name: RISK_SUMMARY
    type: string
    description: One-page executive risk summary with a traffic-light score (🟢/🟡/🔴) for each service
  - name: RECOMMENDED_ACTIONS
    type: string
    description: Prioritised list of recommended actions across SRE, Performance, and Capacity workstreams
---

## Overview

The Platform Health Assessment Agent is the single agent to run when you need a 360° view of your platform's health before a major release, after a sustained incident, or as part of a weekly operational review.

It autonomously invokes tools from three workstreams — SRE, Performance Engineering, and Capacity Management — and synthesises their findings into one unified report, so you do not need three separate conversations with three separate skill sets.

**Cross-workstream value:**
- **SRE** perspective: Are there active or recurring incidents? Which services have noisy alerts? Are runbooks up to date?
- **Performance Engineering** perspective: Are any services regressing on latency or error rate? Where are the bottlenecks?
- **Capacity Management** perspective: Which services are close to resource limits? When will they breach capacity if traffic grows at the current rate?

Combining all three into one report eliminates the situation where an SRE, a performance engineer, and a capacity planner each produce a partial picture that nobody assembles into action.

**Audience:** Platform engineering leads, SRE managers, heads of infrastructure, on-call incident commanders.

## Architecture

**Skills / Tools Used:**

| Tool / Skill | Workstream | Purpose |
|---|---|---|
| `sre-incident-triage-prompt` | SRE | Classify severity of any active or recurring incidents visible in the platform summary |
| `sre-runbook-generator` | SRE | Identify services without runbook coverage and flag the gap |
| `perf-bottleneck-analysis` | Performance Engineers | Detect latency regressions and throughput bottlenecks from performance metrics |
| `perf-performance-baseline-prompt` | Performance Engineers | Compare current metrics against established baselines and flag deviations |
| `cap-utilization-analysis` | Capacity Management | Evaluate current CPU/memory/storage utilisation and identify services near limits |
| `cap-capacity-forecast-prompt` | Capacity Management | Project capacity breaches over the specified forecast horizon |

**Architecture Diagram:**

```
PLATFORM_SUMMARY + PERFORMANCE_METRICS + UTILIZATION_DATA
+ INCIDENT_HISTORY (opt) + FORECAST_HORIZON (opt)
              │
              ▼
┌─────────────────────────────────────────────────┐
│           Platform Health Assessment Agent       │
│                                                  │
│  ┌──────────────────────┐                        │
│  │   SRE Workstream      │                       │
│  │  sre-incident-triage  │──▶ INCIDENT_FINDINGS  │
│  │  sre-runbook-         │──▶ RUNBOOK_GAPS        │
│  │  generator            │                       │
│  └──────────────────────┘                        │
│                                                  │
│  ┌──────────────────────┐                        │
│  │  Performance          │                       │
│  │  Workstream           │                       │
│  │  perf-bottleneck-     │──▶ PERF_FINDINGS      │
│  │  analysis             │                       │
│  │  perf-performance-    │──▶ BASELINE_DELTA      │
│  │  baseline-prompt      │                       │
│  └──────────────────────┘                        │
│                                                  │
│  ┌──────────────────────┐                        │
│  │  Capacity             │                       │
│  │  Workstream           │                       │
│  │  cap-utilization-     │──▶ UTIL_FINDINGS      │
│  │  analysis             │                       │
│  │  cap-capacity-        │──▶ CAPACITY_FORECAST  │
│  │  forecast-prompt      │                       │
│  └──────────────────────┘                        │
│              │                                   │
│              ▼                                   │
│   Synthesise all findings                        │
│   Assign 🟢/🟡/🔴 per service                   │
└──────────────┬──────────────────────────────────┘
               │
               ▼
  PLATFORM_HEALTH_REPORT + RISK_SUMMARY
       + RECOMMENDED_ACTIONS
```

## System Prompt / Agent Instructions

```
[System Prompt]

You are an autonomous Platform Health Assessment Agent. You have access to tools from
three engineering workstreams: SRE, Performance Engineering, and Capacity Management.

Your goals:
1. Identify active or recurring incidents and classify their severity using SRE triage.
2. Detect services with missing runbook coverage using the SRE runbook tool.
3. Identify performance regressions and bottlenecks using performance analysis tools.
4. Evaluate current capacity utilisation and forecast future breaches using capacity tools.
5. Synthesise all findings into a unified Platform Health Report with per-service risk scores.
6. Produce a prioritised RECOMMENDED_ACTIONS list that an engineering team can act on immediately.

Tools available to you:
- sre_incident_triage:       call with (incident_description, alert_details, affected_service)
                              → returns severity classification and root-cause hypothesis
- sre_runbook_check:         call with (service_name, system_context)
                              → returns whether a runbook exists and its completeness score
- perf_bottleneck_analysis:  call with (service_name, metrics)
                              → returns identified bottlenecks ranked by impact
- perf_baseline_comparison:  call with (service_name, current_metrics, baseline_metrics)
                              → returns deviation from baseline with regression flag
- cap_utilization_analysis:  call with (service_name, utilization_data)
                              → returns current headroom and risk level
- cap_capacity_forecast:     call with (service_name, utilization_trend, horizon_days)
                              → returns projected breach date and recommended action

Rules:
- Invoke all six tools. Do not skip a workstream even if data appears healthy.
- Assign a health score to each service: 🟢 (healthy), 🟡 (at risk), 🔴 (critical).
  A service is 🔴 if: active SEV1/SEV2 incident OR p99 latency > 2× baseline OR
  projected capacity breach within the forecast horizon.
  A service is 🟡 if: recurring alerts OR p99 latency > 1.5× baseline OR
  utilisation > 80% of limit.
  Otherwise 🟢.
- List RECOMMENDED_ACTIONS in priority order: 🔴 issues first, then 🟡, then 🟢 improvements.
- Keep the RISK_SUMMARY to one page (≤ 400 words). The full PLATFORM_HEALTH_REPORT has no length limit.
- Never include raw credentials, PII, or internal IP addresses in the report.

Output format:

## Platform Health Report — {DATE}

### Executive Risk Summary
{RISK_SUMMARY — traffic-light table per service}

### SRE Assessment
{INCIDENT_FINDINGS + RUNBOOK_GAPS}

### Performance Assessment
{PERF_FINDINGS + BASELINE_DELTA}

### Capacity Assessment
{UTIL_FINDINGS + CAPACITY_FORECAST}

### Recommended Actions
{RECOMMENDED_ACTIONS — numbered, prioritised}
```

## Configuration

```yaml
agent_id: shared-platform-health-agent
model: gpt-4o
temperature: 0.1          # deterministic output for consistent reports
max_iterations: 10        # triage → runbook-check → bottleneck × N services → baseline × N → utilisation × N → forecast × N → synthesise → format
tools:
  - sre_incident_triage
  - sre_runbook_check
  - perf_bottleneck_analysis
  - perf_baseline_comparison
  - cap_utilization_analysis
  - cap_capacity_forecast
parallel_tool_calls: true  # performance + capacity tools run in parallel per service
forecast_horizon_default: 30
health_score_thresholds:
  red_latency_multiplier: 2.0
  yellow_latency_multiplier: 1.5
  yellow_utilization_pct: 80
```

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `{{PLATFORM_SUMMARY}}` | string | Yes | Services list with alert counts and current statuses |
| `{{PERFORMANCE_METRICS}}` | string | Yes | p50/p95/p99 latency, error rate, throughput per service |
| `{{UTILIZATION_DATA}}` | string | Yes | CPU, memory, storage utilisation per service or node |
| `{{INCIDENT_HISTORY}}` | string | No | Last 7–30 days of incidents with severity and MTTR |
| `{{FORECAST_HORIZON}}` | string | No | `30`, `60`, or `90` days (default: `30`) |

## Outputs

| Name | Type | Description |
|---|---|---|
| `PLATFORM_HEALTH_REPORT` | string | Full structured report across all three workstreams |
| `RISK_SUMMARY` | string | One-page executive summary with 🟢/🟡/🔴 scores per service |
| `RECOMMENDED_ACTIONS` | string | Prioritised action list across SRE, Performance, and Capacity |

## Security Considerations

- The agent reads metrics and alert data but never modifies infrastructure configuration.
- All inputs are treated as read-only operational data — no commands are executed.
- Do not include secrets, API keys, or internal database connection strings in any input field.
- For sensitive environments, route through `in-house-llm` to prevent metric data leaving the private network.
- The PLATFORM_HEALTH_REPORT should be treated as `internal` classification — do not share publicly.

## Usage

### GitHub Copilot Chat (current)

Open a new Copilot Chat session and paste:

```
You are a Platform Health Assessment Agent.

Platform summary: {{PLATFORM_SUMMARY}}
Performance metrics: {{PERFORMANCE_METRICS}}
Utilisation data: {{UTILIZATION_DATA}}
Incident history (last 7d): {{INCIDENT_HISTORY}}
Forecast horizon: {{FORECAST_HORIZON}} days

Assess platform health across SRE, Performance, and Capacity dimensions.
Assign a 🟢/🟡/🔴 score per service. Produce PLATFORM_HEALTH_REPORT, RISK_SUMMARY, and RECOMMENDED_ACTIONS.
```

### API Endpoint (future portal)

```http
POST {{API_BASE_URL}}/packages/shared-cross-workstream-pack/invoke
Content-Type: application/json

{
  "agent_id": "shared-platform-health-agent",
  "inputs": {
    "platform_summary": "payment-service: 14 alerts/7d | order-service: 2 alerts/7d | ...",
    "performance_metrics": "payment-service p99=420ms baseline=180ms error_rate=2.1% ...",
    "utilization_data": "payment-service cpu=78% mem=64% | order-service cpu=31% mem=40% ...",
    "incident_history": "INC-4821 SEV1 2026-04-18 payment-service MTTR=47min ...",
    "forecast_horizon": "30"
  }
}
```

### MCP Tool (future)

```json
{
  "tool": "shared-platform-health-agent",
  "parameters": {
    "platform_summary": "...",
    "performance_metrics": "...",
    "utilization_data": "...",
    "incident_history": "...",
    "forecast_horizon": "30"
  }
}
```

## Examples

### Example — e-commerce platform weekly health review

**Input:**
- `PLATFORM_SUMMARY`:
  ```
  payment-service:   14 alerts in 7d (2× SEV1, 4× SEV2, 8× SEV3)
  order-service:      2 alerts in 7d (0× SEV1, 1× SEV2, 1× SEV3)
  catalogue-service:  0 alerts in 7d
  notification-svc:   1 alert  in 7d (0× SEV1, 0× SEV2, 1× SEV3)
  ```
- `PERFORMANCE_METRICS`:
  ```
  payment-service:   p50=210ms  p99=420ms  baseline_p99=180ms  error_rate=2.1%  rps=340
  order-service:     p50=85ms   p99=190ms  baseline_p99=170ms  error_rate=0.3%  rps=510
  catalogue-service: p50=22ms   p99=48ms   baseline_p99=50ms   error_rate=0.01% rps=2100
  notification-svc:  p50=310ms  p99=890ms  baseline_p99=400ms  error_rate=0.8%  rps=95
  ```
- `UTILIZATION_DATA`:
  ```
  payment-service:   cpu=78%  mem=64%  disk=32%  replicas=3/3
  order-service:     cpu=31%  mem=40%  disk=18%  replicas=3/3
  catalogue-service: cpu=14%  mem=28%  disk=55%  replicas=5/5
  notification-svc:  cpu=43%  mem=71%  disk=22%  replicas=2/2
  ```
- `INCIDENT_HISTORY`:
  ```
  INC-4821 2026-04-18 SEV1 payment-service retry-storm MTTR=47min
  INC-4815 2026-04-15 SEV2 payment-service DB slow queries MTTR=22min
  INC-4799 2026-04-11 SEV2 order-service upstream timeout MTTR=18min
  ```
- `FORECAST_HORIZON`: `30`

---

**Output — RISK_SUMMARY:**

```
## Platform Health Report — 2026-04-20

### Executive Risk Summary

| Service | Health | SRE | Performance | Capacity | Key Risk |
|---|---|---|---|---|---|
| payment-service | 🔴 | 🔴 High alert rate (2× SEV1 in 7d) | 🔴 p99 2.3× baseline | 🟡 CPU 78% | Recurring SEV1s + performance regression |
| order-service | 🟡 | 🟡 1× SEV2 in 7d | 🟢 p99 within 15% of baseline | 🟢 CPU 31% | Upstream dependency risk from payment-service |
| catalogue-service | 🟢 | 🟢 0 alerts | 🟢 p99 within baseline | 🟢 All metrics healthy | No action required |
| notification-svc | 🟡 | 🟢 1× SEV3 only | 🟡 p99 2.2× baseline | 🟡 Memory 71% | Latency regression + memory growth trend |

**Overall platform health: 🟡 At Risk — payment-service is the critical hot spot.**
```

---

**Output — PLATFORM_HEALTH_REPORT (SRE Assessment section):**

```
### SRE Assessment

#### Incident Findings
payment-service has experienced 2 SEV1 and 4 SEV2 incidents in the past 7 days —
an unsustainable alert rate indicating a systemic reliability problem rather than
one-off events. INC-4821 (retry storm, MTTR 47min) and INC-4815 (DB slow queries,
MTTR 22min) both point to the payment processing path as the root locus.

SEV classification for current state: **SEV2** (elevated error rate 2.1%, no total outage).

#### Runbook Coverage Gaps
- payment-service: Runbook for "retry storm" scenario not found. Recommend generating
  via sre-runbook-generator with ALERT_OR_SCENARIO="exponential-backoff retry storm".
- notification-svc: No runbook found for high-latency / queue-depth scenario.

### Performance Assessment

#### Bottleneck Analysis
- payment-service p99 latency is 420ms vs 180ms baseline — a **2.3× regression**.
  Most likely bottleneck: Stripe API call in the critical path (synchronous, no timeout
  configured). Secondary bottleneck: DB connection pool saturation at peak load.
- notification-svc p99 = 890ms vs 400ms baseline — **2.2× regression**.
  Probable cause: message queue consumer is single-threaded; backpressure visible at
  95 rps. Recommend increasing consumer concurrency from 1 to 4.

#### Baseline Comparison
- order-service: p99 190ms vs baseline 170ms — 12% above baseline, within acceptable range.
- catalogue-service: p99 48ms vs baseline 50ms — **below baseline** (positive). No action.

### Capacity Assessment

#### Utilisation Analysis
- payment-service CPU 78% — above the 🟡 threshold (80% projected within 2 weeks at
  current growth rate without optimisation).
- notification-svc memory 71% — trending upward. Potential memory leak; investigate
  before next deployment.
- All other services: healthy headroom.

#### Capacity Forecast (30-day horizon)
- payment-service: At current traffic growth (+8%/week), CPU will breach 90% in **~18 days**.
  Recommended action: Scale to 5 replicas immediately; longer-term fix is async Stripe integration.
- notification-svc: Memory will reach 90% in **~25 days** if trend continues.
  Recommended action: Profile for memory leak; add horizontal scaling if no leak found.
```

---

**Output — RECOMMENDED_ACTIONS:**

```
### Recommended Actions

#### 🔴 Critical — Act within 24 hours
1. [SRE + Performance] payment-service: Make Stripe API call asynchronous with a job queue
   to eliminate the synchronous p99 bottleneck and prevent retry storms.
   Owner: payments team | Estimated effort: 2 days
2. [Capacity] payment-service: Scale replicas from 3 to 5 immediately to buy headroom
   before CPU breaches 90% in ~18 days.
   Owner: SRE on-call | Estimated effort: 30 minutes

#### 🟡 At Risk — Act within 1 week
3. [SRE] payment-service: Generate and publish a runbook for the "retry storm" scenario
   using sre-runbook-generator. Link from PagerDuty alert.
   Owner: SRE team | Estimated effort: 1 hour
4. [Performance] notification-svc: Profile memory usage; suspect message consumer is not
   releasing processed message buffers. Fix or add horizontal scaling.
   Owner: notifications team | Estimated effort: 1 day
5. [SRE] notification-svc: Generate runbook for high-latency / queue-depth alert.
   Owner: SRE team | Estimated effort: 1 hour
6. [Performance] notification-svc: Increase consumer concurrency from 1 to 4 to address
   p99 latency regression from 400ms to 890ms.
   Owner: notifications team | Estimated effort: 2 hours

#### 🟢 Improvements — Next sprint
7. [SRE] Add payment-service DB connection pool saturation to the existing SEV2 runbook
   as Scenario B. Diagnostic command: `psql -c "SELECT count(*) FROM pg_stat_activity"`.
8. [Capacity] Review catalogue-service disk usage at 55% — no breach risk in 30 days but
   schedule a capacity review if new product catalogue import is planned.
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | 2026-04-20. All six tools invoked correctly in a single session. Risk scores accurate given the threshold rules. Recommended actions well-prioritised. Runs in ~12 LLM calls with parallel tool invocation. |
| claude-3-5-sonnet | ✅ | 2026-04-20. More verbose SRE assessment section. Capacity forecast narrative is clearer. Slightly longer but higher-quality RECOMMENDED_ACTIONS reasoning. |

## Changelog

### 1.0.0 — 2026-04-20
- Initial version. Cross-workstream combination of sre-incident-triage-prompt, sre-runbook-generator, perf-bottleneck-analysis, perf-performance-baseline-prompt, cap-utilization-analysis, cap-capacity-forecast-prompt.
