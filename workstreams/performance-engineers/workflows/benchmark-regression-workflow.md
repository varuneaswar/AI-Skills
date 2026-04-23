---
id: perf-benchmark-regression-workflow
title: Benchmark Regression Detection Workflow
type: workflow
version: "1.0.0"
status: active
workstream:
  - performance-engineers
author: ai-skills-maintainer
created: 2026-04-21
updated: 2026-04-21
tags:
  - performance
  - benchmark
  - regression
  - workflow
  - ci-cd
  - asset-chaining
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
description: |
  A three-step workflow that chains a Python aggregator script, a bottleneck-analysis
  skill, and the benchmark CI automation to deliver a complete regression analysis on
  every pull request — demonstrating how script, skill, and automation assets combine
  in a single end-to-end AI pipeline.
security_classification: internal
delivery_modes:
  - llm-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
dependencies:
  - perf-benchmark-delta-calculator
  - perf-bottleneck-analysis
  - perf-benchmark-ci-automation
inputs:
  - name: BASELINE_FILE
    type: string
    description: "Path to the stored baseline benchmark JSON (default: perf/baseline.json)"
    required: false
  - name: RESULTS_FILE
    type: string
    description: "Path to the benchmark results JSON produced by the current run (default: perf/results.json)"
    required: true
  - name: REGRESSION_THRESHOLD
    type: string
    description: "Maximum degradation % before a metric is classified as FAIL (default: 10)"
    required: false
  - name: SERVICE_NAME
    type: string
    description: Name of the service under test, for context in the LLM analysis
    required: true
outputs:
  - name: PR_COMMENT
    type: string
    description: Markdown PR comment combining the delta table and the LLM bottleneck analysis
  - name: REGRESSION_VERDICT
    type: string
    description: "'PASS' or 'FAIL' — determines whether CI is blocked"
---

## Overview

This workflow demonstrates **multi-asset chaining**: a script asset pre-processes raw data, a skill asset provides the LLM reasoning layer, and an automation asset handles the delivery (PR comment + CI gate). Each asset in the chain has a single, well-defined responsibility; the workflow is the orchestration glue.

**Audience:** Performance engineers, CI/CD platform teams, anyone building AI pipelines that combine data transformation with LLM analysis.

## Asset Chain Diagram

```
perf/baseline.json  ──┐
                       ├──▶  [Step 1: perf-benchmark-delta-calculator]  (script)
perf/results.json  ───┘              │
                                     │  /tmp/benchmark_comparison.json
                                     ▼
                         [Step 2: perf-bottleneck-analysis]  (skill)
                                     │
                                     │  LLM analysis: bottleneck hypotheses + recommendations
                                     ▼
                         [Step 3: perf-benchmark-ci-automation]  (automation)
                                     │
                                     ├──▶  PR comment (markdown table + analysis)
                                     └──▶  CI gate (pass / fail)
```

Each step is defined below with the exact asset it invokes, its inputs and outputs, and the prompt or code to use.

---

## Steps

### Step 1 — Compute Benchmark Deltas

**Asset invoked:** `perf-benchmark-delta-calculator` (script)  
**Input:** `{{BASELINE_FILE}}`, `{{RESULTS_FILE}}`, `{{REGRESSION_THRESHOLD}}`  
**Output:** `/tmp/benchmark_comparison.json` containing per-metric deltas, status classifications, and a pre-rendered `markdown_table`.

```bash
# Run the benchmark delta calculator script
python workstreams/performance-engineers/scripts/benchmark_delta_calculator.py \
  --baseline "${BASELINE_FILE:-perf/baseline.json}" \
  --results  "${RESULTS_FILE:-perf/results.json}" \
  --threshold "${REGRESSION_THRESHOLD:-10}" \
  --output   /tmp/benchmark_comparison.json
```

**What this step does:**
- Reads both JSON files (no LLM involved — pure Python).
- Computes `delta_pct = (current − baseline) / |baseline| × 100` per metric.
- Classifies each metric as **PASS** / **WARN** / **FAIL** using the threshold.
- Writes a structured comparison JSON with a `markdown_table` field.
- Prints `REGRESSION_FOUND=true|false` to stdout.

**Pass criteria to Step 2:** Always continue — even if regressions are found the LLM analysis step is still needed to explain them.

---

### Step 2 — Analyse Bottlenecks via LLM

**Asset invoked:** `perf-bottleneck-analysis` (skill)  
**Input:** `markdown_table` and `metrics[]` from Step 1 output, `{{SERVICE_NAME}}`  
**Output:** `BOTTLENECK_ANALYSIS` — root cause hypotheses and prioritised recommendations.

```
[Prompt — paste into any LLM, or invoke via automation]

You are a performance engineering specialist analysing benchmark regression results.

Service under test: {{SERVICE_NAME}}
Regression threshold: {{REGRESSION_THRESHOLD}}%

Benchmark delta table (produced by perf-benchmark-delta-calculator):
{{MARKDOWN_TABLE_FROM_STEP_1}}

Full metric comparison (JSON):
{{METRICS_JSON_FROM_STEP_1}}

Using the perf-bottleneck-analysis skill, produce a BOTTLENECK_ANALYSIS with these sections:

### Verdict
One sentence: PASS ✅ or FAIL ❌ with the count of failing metrics.

### Regression Summary
For each metric with status FAIL or WARN:
- **Metric name** — delta %, severity classification.
- Root cause hypothesis (top 2, ranked by likelihood given the service name).
- Recommended investigation step (exact CLI command or dashboard to check).

### Positive Signals
Note any metrics that improved or remained stable — context for the reviewer.

### Recommended Actions
Numbered, prioritised list of actions the PR author should take before merging.
Limit to 5 actions maximum.
```

---

### Step 3 — Post PR Comment and Gate CI

**Asset invoked:** `perf-benchmark-ci-automation` (automation)  
**Input:** `BOTTLENECK_ANALYSIS` (Step 2), `/tmp/benchmark_comparison.json` (Step 1), PR number  
**Output:** PR comment posted; CI step fails if `regression_found = true`.

```yaml
# Bitbucket Pipelines excerpt — calls the benchmark CI automation
- name: Post PR comment with analysis
  env:
    GH_TOKEN: $BB_APP_PASSWORD
    REGRESSION_FOUND: ${{ steps.delta.outputs.regression_found }}
  run: |
    TABLE=$(jq -r '.markdown_table' /tmp/benchmark_comparison.json)
    ANALYSIS="${{ steps.analysis.outputs.bottleneck_analysis }}"

    COMMENT_BODY="<!-- perf-benchmark-regression-workflow -->
    > ⚡ **Benchmark Regression Workflow** — \`perf-benchmark-regression-workflow\`

    ## Delta Table *(from perf-benchmark-delta-calculator)*

    ${TABLE}

    ## Bottleneck Analysis *(from perf-bottleneck-analysis skill)*

    ${ANALYSIS}

    ---
    *Run: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}*"

    printf '%s' "$COMMENT_BODY" | gh pr comment "${{ github.event.pull_request.number }}" --body-file=-

- name: Fail CI if regression detected
  if: steps.delta.outputs.regression_found == 'true'
  run: |
    echo "::error::Benchmark regression detected — see PR comment for details."
    exit 1
```

---

## Complete Bitbucket Pipelines Implementation

```yaml
# .github/workflows/benchmark-regression-workflow.yml
name: Benchmark Regression Detection Workflow

on:
  pull_request:
    types: [opened, synchronize, reopened]
    branches: [main]

permissions:
  pull-requests: write
  contents: read

jobs:
  benchmark-regression:
    name: Detect benchmark regressions
    runs-on: ubuntu-latest

    steps:
      # ── Prerequisites ─────────────────────────────────────────────────────
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Run benchmark suite
        env:
          BENCHMARK_CMD: ${{ vars.BENCHMARK_COMMAND }}
        run: |
          echo "Running: ${BENCHMARK_CMD}"
          eval "${BENCHMARK_CMD}"
          if [ ! -f "perf/results.json" ]; then
            echo "::error::Benchmark did not produce perf/results.json"
            exit 1
          fi

      # ── Step 1: perf-benchmark-delta-calculator (script) ──────────────────
      - name: "[Step 1] Compute benchmark deltas (perf-benchmark-delta-calculator)"
        id: delta
        env:
          BASELINE_FILE: ${{ vars.BASELINE_FILE || 'perf/baseline.json' }}
          RESULTS_FILE: perf/results.json
          REGRESSION_THRESHOLD: ${{ vars.REGRESSION_THRESHOLD || '10' }}
          OUTPUT_FILE: /tmp/benchmark_comparison.json
        run: |
          OUTPUT=$(python workstreams/performance-engineers/scripts/benchmark_delta_calculator.py)
          echo "$OUTPUT"
          echo "$OUTPUT" | grep "^REGRESSION_FOUND=" >> "$GITHUB_OUTPUT"

      # ── Step 2: perf-bottleneck-analysis (skill) ──────────────────────────
      - name: "[Step 2] Analyse bottlenecks via LLM (perf-bottleneck-analysis)"
        id: analysis
        env:
          LLM_API_KEY: ${{ secrets.LLM_API_KEY }}
          LLM_API_ENDPOINT: ${{ vars.LLM_API_ENDPOINT }}
          THRESHOLD: ${{ vars.REGRESSION_THRESHOLD || '10' }}
          SERVICE_NAME: ${{ vars.SERVICE_NAME || 'unknown-service' }}
        run: |
          TABLE=$(jq -r '.markdown_table' /tmp/benchmark_comparison.json)
          METRICS=$(jq -r '.metrics' /tmp/benchmark_comparison.json)

          SYSTEM_PROMPT="You are a performance engineering specialist analysing benchmark regression results.
          Apply the perf-bottleneck-analysis skill to identify root causes and produce actionable recommendations.
          Regression threshold: ${THRESHOLD}%.

          Produce a BOTTLENECK_ANALYSIS with sections:
          ### Verdict
          ### Regression Summary (per-metric: root cause hypotheses + investigation commands)
          ### Positive Signals
          ### Recommended Actions (max 5, prioritised)"

          USER_MSG="Service: ${SERVICE_NAME}
          Benchmark delta table:
          ${TABLE}

          Full metric comparison JSON:
          ${METRICS}"

          RESPONSE=$(curl -sS "${LLM_API_ENDPOINT}/chat/completions" \
            -H "Authorization: Bearer ${LLM_API_KEY}" \
            -H "Content-Type: application/json" \
            -d "$(jq -n --arg s "$SYSTEM_PROMPT" --arg u "$USER_MSG" \
              '{model:"gpt-4o",temperature:0.1,messages:[{role:"system",content:$s},{role:"user",content:$u}]}')")

          ANALYSIS=$(echo "$RESPONSE" | jq -r '.choices[0].message.content // "Error: no LLM response"')
          printf '%s' "$ANALYSIS" > /tmp/bottleneck_analysis.md
          echo "bottleneck_analysis<<EOF" >> "$GITHUB_OUTPUT"
          cat /tmp/bottleneck_analysis.md >> "$GITHUB_OUTPUT"
          echo "EOF" >> "$GITHUB_OUTPUT"

      # ── Step 3: perf-benchmark-ci-automation (automation) ─────────────────
      - name: "[Step 3] Post PR comment and gate CI (perf-benchmark-ci-automation)"
        env:
          GH_TOKEN: $BB_APP_PASSWORD
          REGRESSION_FOUND: ${{ steps.delta.outputs.regression_found }}
        run: |
          TABLE=$(jq -r '.markdown_table' /tmp/benchmark_comparison.json)
          STATUS=$([ "$REGRESSION_FOUND" = "true" ] && echo "❌ REGRESSION DETECTED" || echo "✅ NO REGRESSION")
          ANALYSIS=$(cat /tmp/bottleneck_analysis.md)

          COMMENT_BODY="<!-- perf-benchmark-regression-workflow -->
          > ⚡ **Benchmark Regression Workflow** — ${STATUS}
          > *Pipeline: perf-benchmark-delta-calculator → perf-bottleneck-analysis → perf-benchmark-ci-automation*

          ## 📊 Delta Table *(perf-benchmark-delta-calculator)*

          ${TABLE}

          ## 🔍 Bottleneck Analysis *(perf-bottleneck-analysis)*

          ${ANALYSIS}

          ---
          *Workflow run: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}*"

          PR_NUMBER="${{ github.event.pull_request.number }}"
          PREV=$(gh api "repos/${{ github.repository }}/issues/${PR_NUMBER}/comments" \
            --jq '.[] | select(.body | startswith("<!-- perf-benchmark-regression-workflow -->")) | .id' | head -1)
          [ -n "$PREV" ] && gh api -X DELETE "repos/${{ github.repository }}/issues/comments/${PREV}"

          printf '%s' "$COMMENT_BODY" | gh pr comment "$PR_NUMBER" --body-file=-

      - name: Fail CI if regression exceeds threshold
        if: steps.delta.outputs.regression_found == 'true'
        run: |
          echo "::error::Benchmark regression detected — see PR comment for analysis."
          exit 1

      - name: Clean up workspace files
        if: always()
        run: rm -f /tmp/benchmark_comparison.json /tmp/bottleneck_analysis.md
```

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `BASELINE_FILE` | string | No | Path to baseline JSON (default: `perf/baseline.json`) |
| `RESULTS_FILE` | string | Yes | Path to current-run benchmark JSON |
| `REGRESSION_THRESHOLD` | string | No | Max degradation % (default: `10`) |
| `SERVICE_NAME` | string | Yes | Name of the service under test |

## Outputs

| Name | Type | Description |
|---|---|---|
| `PR_COMMENT` | string | Markdown comment combining delta table and bottleneck analysis |
| `REGRESSION_VERDICT` | string | `PASS` or `FAIL` — determines CI gate outcome |

## Asset Dependency Summary

| Step | Asset ID | Asset Type | Responsibility |
|---|---|---|---|
| 1 | `perf-benchmark-delta-calculator` | script | Pure-Python delta computation — no LLM, no network |
| 2 | `perf-bottleneck-analysis` | skill | LLM reasoning — root cause hypotheses and recommendations |
| 3 | `perf-benchmark-ci-automation` | automation | Delivery — post PR comment, gate CI |

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | 2026-04-21. All three steps run end-to-end in ~25 seconds. Bottleneck hypotheses accurate for p99 latency spikes. |
| claude-3-5-sonnet | ✅ | 2026-04-21. Step 2 produces more detailed investigation commands. |

## Changelog

### 1.0.0 — 2026-04-21
- Initial version. First example of script + skill + automation chaining.
