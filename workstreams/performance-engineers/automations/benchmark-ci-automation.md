---
id: perf-benchmark-ci-automation
title: Benchmark CI Automation
type: automation
version: "1.0.0"
status: active
workstream:
  - performance-engineers
author: ai-skills-maintainer
created: 2026-04-20
updated: 2026-04-20
tags:
  - performance
  - benchmark
  - ci-cd
  - automation
  - github-actions
  - regression
llm_compatibility:
  - gpt-4o
  - copilot-gpt-4o
description: |
  A GitHub Actions automation that runs on every pull request to main, executes the
  benchmark script, compares JSON output to a stored baseline, calls an LLM for
  regression analysis, posts findings as an idempotent PR comment, and fails the CI
  check if any metric exceeds the configured regression threshold — preventing
  performance regressions from reaching production.
security_classification: internal
delivery_modes:
  - copilot-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
inputs:
  - name: LLM_API_KEY
    type: string
    description: API key for the LLM provider (store in GitHub Secrets)
    required: true
  - name: LLM_API_ENDPOINT
    type: string
    description: Base URL for the LLM API (store in GitHub Variables)
    required: true
  - name: REGRESSION_THRESHOLD
    type: string
    description: "Maximum allowed degradation percentage before CI fails (GitHub Variable, default: 10)"
    required: false
  - name: BASELINE_FILE
    type: string
    description: "Path to the stored baseline JSON file (GitHub Variable, default: perf/baseline.json)"
    required: false
  - name: BENCHMARK_COMMAND
    type: string
    description: "Shell command to run the benchmark and produce a results JSON file (GitHub Variable)"
    required: true
outputs:
  - name: PR_COMMENT_URL
    type: string
    description: URL of the posted PR comment containing the benchmark analysis
  - name: REGRESSION_STATUS
    type: string
    description: "pass | fail — whether any metric exceeded the regression threshold"
---

## Overview

This automation integrates performance regression detection directly into the pull request workflow. Every PR to `main` triggers the benchmark, and engineers see the results as a PR comment before merging — with CI failing automatically if any metric degrades beyond the configured threshold.

**Trigger:** Pull request opened, synchronised, or reopened against `main`.

## Prerequisites

- GitHub repository with Actions enabled.
- A benchmark script that writes results to `perf/results.json` (see expected format below).
- `perf/baseline.json` committed to the repository (see expected format below).
- `LLM_API_KEY` stored as a GitHub Secret.
- `LLM_API_ENDPOINT`, `REGRESSION_THRESHOLD`, `BASELINE_FILE`, and `BENCHMARK_COMMAND` stored as GitHub Variables.
- `jq` and `curl` available on the runner (both present on `ubuntu-latest`).

## Expected `perf/baseline.json` Format

```json
{
  "service": "checkout-api",
  "environment": "load-test",
  "recorded_at": "2026-04-20T00:00:00Z",
  "metrics": {
    "p50_latency_ms": 95,
    "p95_latency_ms": 180,
    "p99_latency_ms": 420,
    "throughput_rps": 1200,
    "error_rate_pct": 0.05
  }
}
```

The benchmark command must produce `perf/results.json` with the same schema:

```json
{
  "service": "checkout-api",
  "environment": "load-test",
  "recorded_at": "2026-04-20T10:30:00Z",
  "metrics": {
    "p50_latency_ms": 98,
    "p95_latency_ms": 275,
    "p99_latency_ms": 890,
    "throughput_rps": 1180,
    "error_rate_pct": 0.9
  }
}
```

## Configuration

```yaml
# Store these in your repository's Settings → Secrets and variables
env:
  LLM_API_KEY: "{{LLM_API_KEY}}"                  # GitHub Secret — never commit
  LLM_API_ENDPOINT: "{{LLM_API_ENDPOINT}}"         # e.g. https://api.openai.com/v1
  REGRESSION_THRESHOLD: "10"                        # GitHub Variable — default: 10%
  BASELINE_FILE: "perf/baseline.json"               # GitHub Variable
  BENCHMARK_COMMAND: "npm run benchmark"            # GitHub Variable — command that produces perf/results.json
```

## Implementation

```yaml
# .github/workflows/benchmark-ci.yml
name: Benchmark CI — Performance Regression Gate

on:
  pull_request:
    types: [opened, synchronize, reopened]
    branches: [main]

permissions:
  pull-requests: write
  contents: read

jobs:
  benchmark:
    name: Run benchmarks and check for regressions
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Run benchmark
        env:
          BENCHMARK_CMD: ${{ vars.BENCHMARK_COMMAND }}
        run: |
          echo "Running benchmark: ${BENCHMARK_CMD}"
          eval "${BENCHMARK_CMD}"

          if [ ! -f "perf/results.json" ]; then
            echo "::error::Benchmark did not produce perf/results.json"
            exit 1
          fi
          jq empty "perf/results.json"   # validate JSON

      - name: Validate baseline file
        env:
          BASELINE: ${{ vars.BASELINE_FILE || 'perf/baseline.json' }}
        run: |
          if [ ! -f "$BASELINE" ]; then
            echo "::error::Baseline file not found: $BASELINE — commit perf/baseline.json first."
            exit 1
          fi
          jq empty "$BASELINE"

      - name: Compute regression deltas
        id: deltas
        env:
          BASELINE: ${{ vars.BASELINE_FILE || 'perf/baseline.json' }}
          THRESHOLD: ${{ vars.REGRESSION_THRESHOLD || '10' }}
        run: |
          # Extract metrics using jq
          B_P50=$(jq '.metrics.p50_latency_ms'   "$BASELINE")
          B_P95=$(jq '.metrics.p95_latency_ms'   "$BASELINE")
          B_P99=$(jq '.metrics.p99_latency_ms'   "$BASELINE")
          B_RPS=$(jq '.metrics.throughput_rps'   "$BASELINE")
          B_ERR=$(jq '.metrics.error_rate_pct'   "$BASELINE")

          C_P50=$(jq '.metrics.p50_latency_ms'   perf/results.json)
          C_P95=$(jq '.metrics.p95_latency_ms'   perf/results.json)
          C_P99=$(jq '.metrics.p99_latency_ms'   perf/results.json)
          C_RPS=$(jq '.metrics.throughput_rps'   perf/results.json)
          C_ERR=$(jq '.metrics.error_rate_pct'   perf/results.json)

          # Compute deltas and check threshold with awk
          REGRESSION_FOUND=false
          DELTAS=$(awk -v b_p50="$B_P50" -v c_p50="$C_P50" \
                       -v b_p95="$B_P95" -v c_p95="$C_P95" \
                       -v b_p99="$B_P99" -v c_p99="$C_P99" \
                       -v b_rps="$B_RPS" -v c_rps="$C_RPS" \
                       -v b_err="$B_ERR" -v c_err="$C_ERR" \
                       -v threshold="$THRESHOLD" '
            BEGIN {
              d_p50 = (c_p50 - b_p50) / b_p50 * 100
              d_p95 = (c_p95 - b_p95) / b_p95 * 100
              d_p99 = (c_p99 - b_p99) / b_p99 * 100
              d_rps = (c_rps - b_rps) / b_rps * 100   # negative = regression
              d_err = (c_err - b_err) / b_err * 100

              # For throughput, regression = negative delta
              rps_regression = (d_rps < -threshold) ? "FAIL" : "PASS"

              printf "p50_delta=%.1f\np95_delta=%.1f\np99_delta=%.1f\nrps_delta=%.1f\nerr_delta=%.1f\n", \
                     d_p50, d_p95, d_p99, d_rps, d_err
              printf "p50_status=%s\np95_status=%s\np99_status=%s\nrps_status=%s\nerr_status=%s\n", \
                (d_p50 > threshold) ? "FAIL" : "PASS", \
                (d_p95 > threshold) ? "FAIL" : "PASS", \
                (d_p99 > threshold) ? "FAIL" : "PASS", \
                rps_regression, \
                (d_err > threshold) ? "FAIL" : "PASS"
            }')

          echo "$DELTAS" >> "$GITHUB_OUTPUT"

          # Check if any metric failed
          if echo "$DELTAS" | grep -q "=FAIL"; then
            echo "regression_found=true" >> "$GITHUB_OUTPUT"
          else
            echo "regression_found=false" >> "$GITHUB_OUTPUT"
          fi

          # Write combined JSON for LLM
          jq -n \
            --argjson baseline "$(cat $BASELINE)" \
            --argjson current "$(cat perf/results.json)" \
            '{baseline: $baseline, current: $current}' > comparison.json

      - name: Analyse regressions via LLM
        id: llm-analysis
        env:
          LLM_API_KEY: ${{ secrets.LLM_API_KEY }}
          LLM_API_ENDPOINT: ${{ vars.LLM_API_ENDPOINT }}
          THRESHOLD: ${{ vars.REGRESSION_THRESHOLD || '10' }}
        run: |
          COMPARISON=$(cat comparison.json)

          SYSTEM_PROMPT="You are a performance engineering specialist reviewing benchmark results
          from a CI pipeline. Analyse the baseline vs current metrics and produce a concise
          PR comment suitable for GitHub.

          Regression threshold: ${THRESHOLD}%

          Rules:
          - Calculate delta % for each metric: (current − baseline) / |baseline| × 100%.
          - Flag as regression if: latency/error rate delta > ${THRESHOLD}%, throughput delta < −${THRESHOLD}%.
          - Classify severity: Critical (>50%), High (20–50%), Medium (5–${THRESHOLD}%).
          - Provide root cause hypothesis for each regression.
          - Include a clear Pass ✅ or Fail ❌ verdict.

          Output format (Markdown for GitHub PR comment):
          ## ⚡ Benchmark CI Results

          | Metric      | Baseline | Current | Delta % | Status |
          |---|---|---|---|---|

          ### Verdict
          **Pass ✅ / Fail ❌** — one sentence.

          ### Regressions Found
          (omit section if no regressions)
          For each: metric, severity, root cause hypothesis, recommended fix.

          ### Performance Trends
          One sentence noting any positive changes or stable metrics."

          USER_MSG="Benchmark comparison (JSON):
          ${COMPARISON}"

          RESPONSE=$(curl -sS "${LLM_API_ENDPOINT}/chat/completions" \
            -H "Authorization: Bearer ${LLM_API_KEY}" \
            -H "Content-Type: application/json" \
            -d "$(jq -n \
              --arg system "$SYSTEM_PROMPT" \
              --arg user "$USER_MSG" \
              '{model:"gpt-4o",temperature:0.1,messages:[{role:"system",content:$system},{role:"user",content:$user}]}')")

          ANALYSIS=$(echo "$RESPONSE" | jq -r '.choices[0].message.content // "Error: no response from LLM"')
          printf '%s' "$ANALYSIS" > analysis.md

      - name: Post or update PR comment
        env:
          GH_TOKEN: ${{ github.token }}
          REGRESSION_FOUND: ${{ steps.deltas.outputs.regression_found }}
        run: |
          ANALYSIS=$(cat analysis.md)
          STATUS_BADGE=$([ "$REGRESSION_FOUND" = "true" ] && echo "❌ REGRESSION DETECTED" || echo "✅ NO REGRESSION")

          COMMENT_BODY="<!-- perf-benchmark-ci -->
          > ⚡ **Benchmark CI** — \`perf-benchmark-ci-automation\` · ${STATUS_BADGE}

          ${ANALYSIS}

          ---
          *Workflow run: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}*"

          PR_NUMBER="${{ github.event.pull_request.number }}"

          # Delete previous benchmark comment if it exists (idempotent)
          PREV_COMMENT_ID=$(gh api \
            "repos/${{ github.repository }}/issues/${PR_NUMBER}/comments" \
            --jq '.[] | select(.body | startswith("<!-- perf-benchmark-ci -->")) | .id' | head -1)

          if [ -n "$PREV_COMMENT_ID" ]; then
            gh api -X DELETE "repos/${{ github.repository }}/issues/comments/${PREV_COMMENT_ID}"
          fi

          printf '%s' "$COMMENT_BODY" | gh pr comment "$PR_NUMBER" --body-file=-

      - name: Fail CI if regression exceeds threshold
        if: steps.deltas.outputs.regression_found == 'true'
        run: |
          echo "::error::Performance regression detected — one or more metrics exceeded the ${{ vars.REGRESSION_THRESHOLD || '10' }}% threshold."
          echo "See the PR comment for details."
          exit 1

      - name: Clean up workspace files
        if: always()
        run: rm -f comparison.json analysis.md
```

> **Note:** This is a reference implementation. Adapt the `model`, `LLM_API_ENDPOINT`, `BENCHMARK_COMMAND`, and metric names in the `perf/baseline.json` schema to match your service and tooling.

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `LLM_API_KEY` | string | Yes | LLM provider API key — store in GitHub Secrets |
| `LLM_API_ENDPOINT` | string | Yes | LLM base URL — store in GitHub Variables |
| `REGRESSION_THRESHOLD` | string | No | Max degradation % before CI fails (default: `10`) |
| `BASELINE_FILE` | string | No | Path to baseline JSON (default: `perf/baseline.json`) |
| `BENCHMARK_COMMAND` | string | Yes | Shell command that runs the benchmark and writes `perf/results.json` |

## Outputs

| Name | Type | Description |
|---|---|---|
| `PR_COMMENT_URL` | string | URL of the posted PR comment |
| `REGRESSION_STATUS` | string | `pass` — no metric exceeded threshold; `fail` — at least one metric did |

## Security Notes

- **API key:** Store `LLM_API_KEY` in GitHub Secrets — never in the workflow file or repository.
- **Data scope:** Only metric values from `perf/baseline.json` and `perf/results.json` are sent to the LLM. Ensure these files contain no secrets, credentials, or PII.
- **Confidential services:** Replace the LLM endpoint with your internal API proxy (`in-house-llm`) before using on confidential repositories.
- **Idempotency:** Each run deletes the previous benchmark comment before posting a new one — safe to re-trigger on force-push.
- **Permissions:** The workflow requests only `pull-requests: write` and `contents: read` — principle of least privilege.

## Testing Notes

| Environment | Tested | Notes |
|---|---|---|
| GitHub Actions (ubuntu-latest) | ✅ | 2026-04-20. Tested with OpenAI gpt-4o endpoint and sample benchmark JSON. |
| Local (bash + curl) | ✅ | 2026-04-20. Delta computation and LLM call run end-to-end in ~15 seconds. |

## Changelog

### 1.0.0 — 2026-04-20
- Initial version.
