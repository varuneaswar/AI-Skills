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
  - github-actions
  - weekly-report
llm_compatibility:
  - gpt-4o
  - copilot-gpt-4o
description: |
  A weekly scheduled GitHub Actions automation (Monday 08:00 UTC) that reads a fleet-wide
  utilization JSON file, calls an LLM for a 90-day capacity forecast, and opens or updates
  a GitHub Issue labelled "capacity-review" with the scaling recommendations — ensuring
  capacity reviews happen consistently without manual intervention.
security_classification: internal
delivery_modes:
  - copilot-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
inputs:
  - name: METRICS_FILE_PATH
    type: string
    description: "Path to the utilization JSON file (GitHub Actions variable, default: data/utilization.json)"
    required: false
  - name: LLM_API_KEY
    type: string
    description: API key for the LLM provider (store in GitHub Secrets)
    required: true
  - name: LLM_API_ENDPOINT
    type: string
    description: Base URL for the LLM API (store in GitHub Variables)
    required: true
outputs:
  - name: GITHUB_ISSUE_URL
    type: string
    description: URL of the created or updated GitHub Issue containing the scaling recommendations
---

## Overview

This automation removes the manual effort of running weekly capacity reviews. Every Monday morning it reads the latest utilization data, generates a 90-day forecast and scaling recommendations via an LLM, then posts the results as a GitHub Issue that the capacity team can act on directly.

The automation is **idempotent**: if an open issue labelled `capacity-review` already exists, it updates that issue rather than creating a duplicate.

**Trigger:** Schedule — Monday 08:00 UTC (`cron: '0 8 * * 1'`).

## Prerequisites

- GitHub repository with Actions enabled.
- `data/utilization.json` committed to the repository and kept up to date (see expected format below).
- `LLM_API_KEY` stored as a GitHub Secret.
- `LLM_API_ENDPOINT` stored as a GitHub Variable.
- `jq` and `curl` available on the runner (both present on `ubuntu-latest`).
- A `capacity-review` label created in the repository.

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
# Store these in your repository's Settings → Secrets and variables
env:
  LLM_API_KEY: "{{LLM_API_KEY}}"             # GitHub Secret — never commit
  LLM_API_ENDPOINT: "{{LLM_API_ENDPOINT}}"   # e.g. https://api.openai.com/v1
  METRICS_FILE_PATH: "data/utilization.json"  # GitHub Variable — path to utilization data
```

## Implementation

```yaml
# .github/workflows/capacity-scaling-review.yml
name: Weekly Capacity Scaling Recommendations

on:
  schedule:
    - cron: '0 8 * * 1'   # Every Monday at 08:00 UTC
  workflow_dispatch:        # Allow manual trigger for testing

permissions:
  issues: write
  contents: read

jobs:
  capacity-review:
    name: Generate capacity scaling recommendations
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Read utilization data
        id: metrics
        env:
          METRICS_FILE: ${{ vars.METRICS_FILE_PATH || 'data/utilization.json' }}
        run: |
          if [ ! -f "$METRICS_FILE" ]; then
            echo "::error::Utilization file not found: $METRICS_FILE"
            exit 1
          fi
          echo "Utilization file found: $METRICS_FILE"
          # Validate JSON
          jq empty "$METRICS_FILE"
          METRICS_CONTENT=$(cat "$METRICS_FILE")
          # Write to a workspace file to avoid shell injection via env var
          printf '%s' "$METRICS_CONTENT" > metrics_payload.json
          echo "services_count=$(jq '.services | length' "$METRICS_FILE")" >> "$GITHUB_OUTPUT"

      - name: Generate 90-day capacity forecast via LLM
        id: forecast
        env:
          LLM_API_KEY: ${{ secrets.LLM_API_KEY }}
          LLM_API_ENDPOINT: ${{ vars.LLM_API_ENDPOINT }}
        run: |
          METRICS=$(cat metrics_payload.json)

          SYSTEM_PROMPT="You are a capacity planning specialist. Analyse the service utilization
          data provided and produce a 90-day capacity forecast with scaling recommendations.

          Rules:
          - Use 80% as warning threshold and 90% as breach threshold.
          - Calculate projected utilization at 30, 60, and 90 days using linear growth rates.
          - Classify breach risk: P1 (breach ≤30 days), P2 (31–60 days), P3 (61–90 days).
          - For each at-risk service, provide a concrete scaling action with estimated lead time.

          Output format (Markdown, suitable for a GitHub Issue body):
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
          echo "forecast_length=$(wc -c < forecast.md)" >> "$GITHUB_OUTPUT"

      - name: Create or update capacity-review GitHub Issue
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          FORECAST=$(cat forecast.md)
          ISSUE_TITLE="📊 Weekly Capacity Review — $(date -u '+%Y-%m-%d')"
          ISSUE_BODY="<!-- cap-scaling-review-automation -->
          > 🤖 **Automated Capacity Review** — generated by \`cap-scaling-recommendation-automation\` on $(date -u '+%Y-%m-%d %H:%M UTC')

          ${FORECAST}

          ---
          *Data source: \`${{ vars.METRICS_FILE_PATH || 'data/utilization.json' }}\` · Workflow run: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}*"

          # Check for existing open capacity-review issue (idempotent)
          EXISTING_ISSUE=$(gh issue list \
            --label "capacity-review" \
            --state open \
            --json number,title \
            --jq '.[0].number // empty')

          if [ -n "$EXISTING_ISSUE" ]; then
            echo "Updating existing issue #${EXISTING_ISSUE}"
            printf '%s' "$ISSUE_BODY" | gh issue edit "$EXISTING_ISSUE" \
              --title "$ISSUE_TITLE" \
              --body-file=-
            ISSUE_URL=$(gh issue view "$EXISTING_ISSUE" --json url -q '.url')
          else
            echo "Creating new capacity-review issue"
            printf '%s' "$ISSUE_BODY" | gh issue create \
              --title "$ISSUE_TITLE" \
              --label "capacity-review" \
              --body-file=-
            ISSUE_URL=$(gh issue list \
              --label "capacity-review" \
              --state open \
              --json url \
              --jq '.[0].url')
          fi

          echo "Issue URL: ${ISSUE_URL}"
          echo "issue_url=${ISSUE_URL}" >> "$GITHUB_OUTPUT"

      - name: Clean up workspace files
        if: always()
        run: rm -f metrics_payload.json forecast.md
```

> **Note:** This is a reference implementation. Adapt the `model`, `LLM_API_ENDPOINT`, and prompt to match your organisation's LLM provider and security requirements. Route through `in-house-llm` for confidential fleet data.

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `METRICS_FILE_PATH` | string | No | Path to utilization JSON (default: `data/utilization.json`) — GitHub Variable |
| `LLM_API_KEY` | string | Yes | LLM provider API key — store in GitHub Secrets |
| `LLM_API_ENDPOINT` | string | Yes | LLM base URL — store in GitHub Variables |

## Outputs

| Name | Type | Description |
|---|---|---|
| `GITHUB_ISSUE_URL` | string | URL of the created or updated capacity-review GitHub Issue |

## Security Notes

- **API key:** Store `LLM_API_KEY` in GitHub Secrets — never in the workflow file or repository.
- **Data scope:** Only the utilization JSON (no secrets, no source code) is sent to the LLM. Ensure `data/utilization.json` contains no credentials or PII.
- **Confidential fleets:** Replace the LLM endpoint with your internal API proxy (`in-house-llm`) before using on confidential infrastructure data.
- **Idempotency:** The automation updates the existing open issue rather than creating duplicates — safe to run via `workflow_dispatch` for manual testing.
- **Permissions:** The workflow requests only `issues: write` and `contents: read` — principle of least privilege.

## Testing Notes

| Environment | Tested | Notes |
|---|---|---|
| GitHub Actions (ubuntu-latest) | ✅ | 2026-04-20. Tested with OpenAI gpt-4o endpoint and 2-service JSON file. |
| Local (bash + curl) | ✅ | 2026-04-20. Steps 2–3 run end-to-end in ~20 seconds with valid JSON input. |

## Changelog

### 1.0.0 — 2026-04-20
- Initial version.
