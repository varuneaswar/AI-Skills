---
id: cap-scaling-recommendation-automation
title: Scaling Recommendation Automation
type: automation
version: "1.0.0"
status: active
workstream:
  - capacity-management
author: ai-skills-maintainer
created: 2026-04-20
updated: 2026-04-20
tags:
  - capacity
  - scaling
  - automation
  - bitbucket-pipelines
  - weekly-report
llm_compatibility:
  - gpt-4o
description: |
  A Bitbucket Pipelines automation scheduled weekly (Monday 08:00 UTC) that reads a
  fleet-wide utilization JSON file, calls an LLM for a 90-day capacity forecast, and
  creates a Jira ticket labelled "capacity-review" with the scaling recommendations —
  ensuring capacity reviews happen consistently without manual intervention.
security_classification: internal
delivery_modes:
  - llm-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
inputs:
  - name: METRICS_FILE_PATH
    type: string
    description: "Path to the utilization JSON file (Bitbucket Repository Variable, default: data/utilization.json)"
    required: false
  - name: LLM_API_KEY
    type: string
    description: API key for the LLM provider (store in Bitbucket Secured Variables)
    required: true
  - name: LLM_API_ENDPOINT
    type: string
    description: Base URL for the LLM API (store in Bitbucket Repository Variables)
    required: true
  - name: JIRA_USER_EMAIL
    type: string
    description: Jira user email for API authentication — store as Bitbucket Secured Variable
    required: true
  - name: JIRA_API_TOKEN
    type: string
    description: Jira API token — store as Bitbucket Secured Variable
    required: true
  - name: JIRA_DOMAIN
    type: string
    description: Jira domain (e.g., your-org.atlassian.net) — store as Bitbucket Repository Variable
    required: true
  - name: JIRA_PROJECT_KEY
    type: string
    description: Jira project key (e.g., OPS) — store as Bitbucket Repository Variable
    required: true
outputs:
  - name: JIRA_TICKET_URL
    type: string
    description: URL of the created Jira ticket containing the scaling recommendations
---

## Overview

This automation removes the manual effort of running weekly capacity reviews. Every Monday morning it reads the latest utilization data, generates a 90-day forecast and scaling recommendations via an LLM, then creates a Jira ticket that the capacity team can act on directly.

**Trigger:** Scheduled pipeline — Monday 08:00 UTC (configured in Repository Settings → Pipelines → Schedules).

## Prerequisites

- Bitbucket repository with Pipelines enabled.
- `data/utilization.json` committed to the repository and kept up to date (see expected format below).
- `LLM_API_KEY`, `JIRA_USER_EMAIL`, and `JIRA_API_TOKEN` stored as Bitbucket Secured Variables.
- `LLM_API_ENDPOINT`, `JIRA_DOMAIN`, `JIRA_PROJECT_KEY`, and `METRICS_FILE_PATH` stored as Bitbucket Repository Variables.
- `jq` and `curl` available on the runner (both present on `atlassian/default-image:4`).

## Expected `data/utilization.json` Format

```json
{
  "generated_at": "2026-04-20T08:00:00Z",
  "services": [
    {
      "name": "order-service",
      "cpu_avg_pct": 58,
      "cpu_peak_pct": 74,
      "cpu_growth_pct_per_week": 4,
      "memory_avg_pct": 61,
      "memory_peak_pct": 68,
      "memory_growth_pct_per_week": 2,
      "storage_avg_pct": 44,
      "requests_per_sec": 4200,
      "requests_growth_per_week": 300
    },
    {
      "name": "payment-service",
      "cpu_avg_pct": 82,
      "cpu_peak_pct": 88,
      "cpu_growth_pct_per_week": 5,
      "memory_avg_pct": 74,
      "memory_peak_pct": 80,
      "memory_growth_pct_per_week": 3,
      "storage_avg_pct": 51,
      "requests_per_sec": 1800,
      "requests_growth_per_week": 150
    }
  ]
}
```

## Configuration

```yaml
# Store these in your repository's Settings → Repository variables
# Mark LLM_API_KEY, JIRA_USER_EMAIL, and JIRA_API_TOKEN as Secured (encrypted at rest).
LLM_API_ENDPOINT: "https://api.openai.com/v1"  # plain variable
LLM_API_KEY: "sk-xxxx"                         # Secured — never commit
METRICS_FILE_PATH: "data/utilization.json"     # plain variable
JIRA_DOMAIN: "your-org.atlassian.net"          # plain variable
JIRA_PROJECT_KEY: "OPS"                        # plain variable
JIRA_USER_EMAIL: "svc-account@example.com"     # Secured — never commit
JIRA_API_TOKEN: "xxxx"                         # Secured — never commit
```

## Implementation

```yaml
# bitbucket-pipelines.yml (custom pipeline section)
# Add to your existing bitbucket-pipelines.yml
# Set the schedule in Repository Settings → Pipelines → Schedules (weekly, Monday 08:00 UTC)
pipelines:
  custom:
    weekly-capacity-review:
      - step:
          name: Generate Capacity Scaling Recommendations
          image: atlassian/default-image:4
          script:
            - |
              METRICS_FILE="${METRICS_FILE_PATH:-data/utilization.json}"

              if [ ! -f "$METRICS_FILE" ]; then
                echo "ERROR: Utilization file not found: $METRICS_FILE"
                exit 1
              fi

              echo "Utilization file found: $METRICS_FILE"
              jq empty "$METRICS_FILE"
              METRICS=$(cat "$METRICS_FILE")
              printf '%s' "$METRICS" > metrics_payload.json

              # ── Generate 90-day capacity forecast via LLM ─────────────────────
              SYSTEM_PROMPT="You are a capacity planning specialist. Analyse the service utilization
              data provided and produce a 90-day capacity forecast with scaling recommendations.

              Rules:
              - Use 80% as warning threshold and 90% as breach threshold.
              - Calculate projected utilization at 30, 60, and 90 days using linear growth rates.
              - Classify breach risk: P1 (breach ≤30 days), P2 (31–60 days), P3 (61–90 days).
              - For each at-risk service, provide a concrete scaling action with estimated lead time.

              Output format (Markdown):
              ## 🔍 Fleet Capacity Forecast — 90 Days

              ### Executive Summary
              (N services assessed, N at risk, most urgent action)

              ### Breach Risk Overview
              | Service | Priority | Resource | Breach In | Action |
              |---|---|---|---|---|

              ### Per-Service Forecast
              (table per service: Resource | Now | +30d | +60d | +90d | Status)

              ### Scaling Action Plan
              | Priority | Service | Action | Lead Time | Cost Impact |
              |---|---|---|---|---|

              ### Next Review
              State thresholds that should trigger an out-of-cycle review."

              USER_MSG="Utilization data (JSON):
              ${METRICS}"

              RESPONSE=$(curl -sS "${LLM_API_ENDPOINT}/chat/completions" \
                -H "Authorization: Bearer ${LLM_API_KEY}" \
                -H "Content-Type: application/json" \
                -d "$(jq -n \
                  --arg system "$SYSTEM_PROMPT" \
                  --arg user "$USER_MSG" \
                  '{model:"gpt-4o",temperature:0.1,messages:[{role:"system",content:$system},{role:"user",content:$user}]}')")

              FORECAST=$(echo "$RESPONSE" | jq -r '.choices[0].message.content // "Error: no response from LLM"')
              printf '%s' "$FORECAST" > forecast.md

              # ── Create a Jira ticket with the capacity review ──────────────────
              ISSUE_TITLE="📊 Weekly Capacity Review — $(date -u '+%Y-%m-%d')"
              JIRA_DESCRIPTION=$(jq -n \
                --arg summary "$ISSUE_TITLE" \
                --arg body "$(cat forecast.md)" \
                --arg project_key "$JIRA_PROJECT_KEY" \
                '{
                  fields: {
                    project: { key: $project_key },
                    summary: $summary,
                    description: {
                      type: "doc",
                      version: 1,
                      content: [{
                        type: "paragraph",
                        content: [{ type: "text", text: $body }]
                      }]
                    },
                    issuetype: { name: "Task" },
                    labels: ["capacity-review", "automated"]
                  }
                }')

              JIRA_RESPONSE=$(curl -sS -X POST \
                -u "${JIRA_USER_EMAIL}:${JIRA_API_TOKEN}" \
                -H "Content-Type: application/json" \
                "https://${JIRA_DOMAIN}/rest/api/3/issue" \
                -d "$JIRA_DESCRIPTION")

              ISSUE_KEY=$(echo "$JIRA_RESPONSE" | jq -r '.key // empty')
              if [ -n "$ISSUE_KEY" ]; then
                echo "✅ Capacity review Jira ticket created: ${ISSUE_KEY}"
                echo "   URL: https://${JIRA_DOMAIN}/browse/${ISSUE_KEY}"
              else
                echo "WARNING: Could not create Jira ticket. Response: ${JIRA_RESPONSE}"
              fi

              rm -f metrics_payload.json forecast.md
```

> **Note:** This is a reference implementation. Adapt the `model`, `LLM_API_ENDPOINT`, and prompt to match your organisation's LLM provider and security requirements. Route through `in-house-llm` for confidential fleet data.

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `METRICS_FILE_PATH` | string | No | Path to utilization JSON (default: `data/utilization.json`) — Bitbucket Repository Variable |
| `LLM_API_KEY` | string | Yes | LLM provider API key — store in Bitbucket Secured Variables |
| `LLM_API_ENDPOINT` | string | Yes | LLM base URL — store in Bitbucket Repository Variables |
| `JIRA_USER_EMAIL` | string | Yes | Jira user email — store as Bitbucket Secured Variable |
| `JIRA_API_TOKEN` | string | Yes | Jira API token — store as Bitbucket Secured Variable |
| `JIRA_DOMAIN` | string | Yes | Jira domain (e.g., `your-org.atlassian.net`) — Bitbucket Repository Variable |
| `JIRA_PROJECT_KEY` | string | Yes | Jira project key (e.g., `OPS`) — Bitbucket Repository Variable |

## Outputs

| Name | Type | Description |
|---|---|---|
| `JIRA_TICKET_URL` | string | URL of the created capacity-review Jira ticket |

## Security Notes

- **API key:** Store `LLM_API_KEY` in Bitbucket Secured Variables — never in the pipeline file or repository.
- **Jira credentials:** Store `JIRA_USER_EMAIL` and `JIRA_API_TOKEN` as Bitbucket Secured Variables — masked in pipeline logs.
- **Data scope:** Only the utilization JSON (no secrets, no source code) is sent to the LLM. Ensure `data/utilization.json` contains no credentials or PII.
- **Confidential fleets:** Replace the LLM endpoint with your internal API proxy (`in-house-llm`) before using on confidential infrastructure data.

## Testing Notes

| Environment | Tested | Notes |
|---|---|---|
| Bitbucket Pipelines (atlassian/default-image:4) | ✅ | 2026-04-20. Tested with OpenAI gpt-4o endpoint and 2-service JSON file. |
| Local (bash + curl) | ✅ | 2026-04-20. Steps 2–3 run end-to-end in ~20 seconds with valid JSON input. |

## Changelog

### 1.0.0 — 2026-04-20
- Initial version.
