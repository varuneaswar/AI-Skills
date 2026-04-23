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
  - bitbucket-pipelines
  - regression
llm_compatibility:
  - gpt-4o
description: |
  A Bitbucket Pipelines automation that runs on every pull request to main, executes the
  benchmark script, compares JSON output to a stored baseline, calls an LLM for
  regression analysis, posts findings as an idempotent PR comment, and fails the CI
  check if any metric exceeds the configured regression threshold — preventing
  performance regressions from reaching production.
security_classification: internal
delivery_modes:
  - llm-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
inputs:
  - name: BB_USERNAME
    type: string
    description: Bitbucket username — store as Bitbucket Repository Variable
    required: true
  - name: BB_APP_PASSWORD
    type: string
    description: Bitbucket App Password — store as Bitbucket Secured Variable
    required: true
  - name: LLM_API_KEY
    type: string
    description: API key for the LLM provider (store in Bitbucket Secured Variables)
    required: true
  - name: LLM_API_ENDPOINT
    type: string
    description: Base URL for the LLM API (store in Bitbucket Repository Variables)
    required: true
  - name: REGRESSION_THRESHOLD
    type: string
    description: "Maximum allowed degradation percentage before CI fails (Bitbucket Repository Variable, default: 10)"
    required: false
  - name: BASELINE_FILE
    type: string
    description: "Path to the stored baseline JSON file (Bitbucket Repository Variable, default: perf/baseline.json)"
    required: false
  - name: BENCHMARK_COMMAND
    type: string
    description: "Shell command to run the benchmark and produce a results JSON file (Bitbucket Repository Variable)"
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

- Bitbucket repository with Pipelines enabled.
- A benchmark script that writes results to `perf/results.json` (see expected format below).
- `perf/baseline.json` committed to the repository (see expected format below).
- `LLM_API_KEY` stored as a Bitbucket Secured Variable.
- `BB_APP_PASSWORD` stored as a Bitbucket Secured Variable.
- `LLM_API_ENDPOINT`, `REGRESSION_THRESHOLD`, `BASELINE_FILE`, and `BENCHMARK_COMMAND` stored as Bitbucket Repository Variables.
- `jq` and `curl` available on the runner (both present on `atlassian/default-image:4`).

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
# Store these in your repository's Settings → Repository variables
# Mark LLM_API_KEY and BB_APP_PASSWORD as Secured (encrypted at rest).
BB_USERNAME: "your-service-account"           # plain variable
BB_APP_PASSWORD: "xxxx"                       # Secured — never commit
LLM_API_ENDPOINT: "https://api.openai.com/v1" # plain variable
LLM_API_KEY: "sk-xxxx"                        # Secured — never commit
REGRESSION_THRESHOLD: "10"                    # plain variable — default: 10%
BASELINE_FILE: "perf/baseline.json"           # plain variable
BENCHMARK_COMMAND: "npm run benchmark"        # plain variable — command that produces perf/results.json
```

## Implementation

```yaml
# bitbucket-pipelines.yml (pull-requests section)
# Add to your existing bitbucket-pipelines.yml
pipelines:
  pull-requests:
    '**':
      - step:
          name: Benchmark CI — Performance Regression Gate
          image: atlassian/default-image:4
          script:
            - |
              BENCHMARK_CMD="${BENCHMARK_COMMAND}"
              BASELINE="${BASELINE_FILE:-perf/baseline.json}"
              THRESHOLD="${REGRESSION_THRESHOLD:-10}"

              # ── Run benchmark ─────────────────────────────────────────────────
              echo "Running benchmark: ${BENCHMARK_CMD}"
              eval "${BENCHMARK_CMD}"

              if [ ! -f "perf/results.json" ]; then
                echo "ERROR: Benchmark did not produce perf/results.json"
                exit 1
              fi
              jq empty "perf/results.json"

              # ── Validate baseline file ────────────────────────────────────────
              if [ ! -f "$BASELINE" ]; then
                echo "ERROR: Baseline file not found: $BASELINE — commit perf/baseline.json first."
                exit 1
              fi
              jq empty "$BASELINE"

              # ── Compute regression deltas ─────────────────────────────────────
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
                  d_rps = (c_rps - b_rps) / b_rps * 100
                  d_err = (c_err - b_err) / b_err * 100

                  printf "p50_delta=%.1f\np95_delta=%.1f\np99_delta=%.1f\nrps_delta=%.1f\nerr_delta=%.1f\n", \
                         d_p50, d_p95, d_p99, d_rps, d_err
                  printf "p50_status=%s\np95_status=%s\np99_status=%s\nrps_status=%s\nerr_status=%s\n", \
                    (d_p50 > threshold) ? "FAIL" : "PASS", \
                    (d_p95 > threshold) ? "FAIL" : "PASS", \
                    (d_p99 > threshold) ? "FAIL" : "PASS", \
                    (d_rps < -threshold) ? "FAIL" : "PASS", \
                    (d_err > threshold) ? "FAIL" : "PASS"
                }')

              echo "$DELTAS" > deltas.txt

              if echo "$DELTAS" | grep -q "=FAIL"; then
                echo "true" > regression_found.txt
              else
                echo "false" > regression_found.txt
              fi

              jq -n \
                --argjson baseline "$(cat $BASELINE)" \
                --argjson current "$(cat perf/results.json)" \
                '{baseline: $baseline, current: $current}' > comparison.json

              # ── Analyse regressions via LLM ───────────────────────────────────
              COMPARISON=$(cat comparison.json)

              SYSTEM_PROMPT="You are a performance engineering specialist reviewing benchmark results
              from a CI pipeline. Analyse the baseline vs current metrics and produce a concise
              PR comment.

              Regression threshold: ${THRESHOLD}%

              Rules:
              - Calculate delta % for each metric: (current − baseline) / |baseline| × 100%.
              - Flag as regression if: latency/error rate delta > ${THRESHOLD}%, throughput delta < −${THRESHOLD}%.
              - Classify severity: Critical (>50%), High (20–50%), Medium (5–${THRESHOLD}%).
              - Provide root cause hypothesis for each regression.
              - Include a clear Pass ✅ or Fail ❌ verdict.

              Output format (Markdown):
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

              # ── Post or update PR comment ─────────────────────────────────────
              REGRESSION_FOUND=$(cat regression_found.txt)
              STATUS_BADGE=$([ "$REGRESSION_FOUND" = "true" ] && echo "❌ REGRESSION DETECTED" || echo "✅ NO REGRESSION")
              ANALYSIS_TEXT=$(cat analysis.md)
              RUN_URL="https://bitbucket.org/${BITBUCKET_REPO_FULL_NAME}/addon/pipelines/home#!/results/${BITBUCKET_BUILD_NUMBER}"

              COMMENT_BODY="⚡ **Benchmark CI** — \`perf-benchmark-ci-automation\` · ${STATUS_BADGE}

${ANALYSIS_TEXT}

---
*Pipeline run: ${RUN_URL}*"

              # Delete previous benchmark comment if it exists (idempotent)
              PREV_COMMENT_ID=$(curl -sS \
                -u "${BB_USERNAME}:${BB_APP_PASSWORD}" \
                "https://api.bitbucket.org/2.0/repositories/${BITBUCKET_REPO_FULL_NAME}/pullrequests/${BITBUCKET_PR_ID}/comments?q=content.raw+%7E+%22Benchmark+CI%22&pagelen=5" \
                | jq -r '.values[0].id // empty')

              if [ -n "$PREV_COMMENT_ID" ]; then
                curl -sS -X DELETE \
                  -u "${BB_USERNAME}:${BB_APP_PASSWORD}" \
                  "https://api.bitbucket.org/2.0/repositories/${BITBUCKET_REPO_FULL_NAME}/pullrequests/${BITBUCKET_PR_ID}/comments/${PREV_COMMENT_ID}"
              fi

              curl -sS -X POST \
                -u "${BB_USERNAME}:${BB_APP_PASSWORD}" \
                -H "Content-Type: application/json" \
                "https://api.bitbucket.org/2.0/repositories/${BITBUCKET_REPO_FULL_NAME}/pullrequests/${BITBUCKET_PR_ID}/comments" \
                -d "$(jq -n --arg body "$COMMENT_BODY" '{content:{raw:$body}}')"

              # ── Fail CI if regression exceeds threshold ───────────────────────
              if [ "$REGRESSION_FOUND" = "true" ]; then
                echo "ERROR: Performance regression detected — one or more metrics exceeded the ${THRESHOLD}% threshold."
                echo "See the PR comment for details."
                rm -f comparison.json analysis.md deltas.txt regression_found.txt
                exit 1
              fi

              rm -f comparison.json analysis.md deltas.txt regression_found.txt
              echo "✅ Benchmark CI complete — no regressions detected."
```

> **Note:** This is a reference implementation. Adapt the `model`, `LLM_API_ENDPOINT`, `BENCHMARK_COMMAND`, and metric names in the `perf/baseline.json` schema to match your service and tooling.

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `BB_USERNAME` | string | Yes | Bitbucket username — store as Repository Variable |
| `BB_APP_PASSWORD` | string | Yes | Bitbucket App Password — store as Secured Variable |
| `LLM_API_KEY` | string | Yes | LLM provider API key — store in Bitbucket Secured Variables |
| `LLM_API_ENDPOINT` | string | Yes | LLM base URL — store in Bitbucket Repository Variables |
| `REGRESSION_THRESHOLD` | string | No | Max degradation % before CI fails (default: `10`) |
| `BASELINE_FILE` | string | No | Path to baseline JSON (default: `perf/baseline.json`) |
| `BENCHMARK_COMMAND` | string | Yes | Shell command that runs the benchmark and writes `perf/results.json` |

## Outputs

| Name | Type | Description |
|---|---|---|
| `PR_COMMENT_URL` | string | URL of the posted PR comment |
| `REGRESSION_STATUS` | string | `pass` — no metric exceeded threshold; `fail` — at least one metric did |

## Security Notes

- **API key:** Store `LLM_API_KEY` in Bitbucket Secured Variables — never in the pipeline file or repository.
- **App Password:** Store `BB_APP_PASSWORD` as a Bitbucket Secured Variable — masked in pipeline logs.
- **Data scope:** Only metric values from `perf/baseline.json` and `perf/results.json` are sent to the LLM. Ensure these files contain no secrets, credentials, or PII.
- **Confidential services:** Replace the LLM endpoint with your internal API proxy (`in-house-llm`) before using on confidential repositories.
- **Idempotency:** Each run deletes the previous benchmark comment before posting a new one — safe to re-trigger on force-push.

## Testing Notes

| Environment | Tested | Notes |
|---|---|---|
| Bitbucket Pipelines (atlassian/default-image:4) | ✅ | 2026-04-20. Tested with OpenAI gpt-4o endpoint and sample benchmark JSON. |
| Local (bash + curl) | ✅ | 2026-04-20. Delta computation and LLM call run end-to-end in ~15 seconds. |

## Changelog

### 1.0.0 — 2026-04-20
- Initial version.
