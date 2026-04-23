---
id: data-analytics-complete-data-analysis-workflow
title: Complete Data Analysis Workflow
type: workflow
version: "1.0.0"
status: active
workstream:
  - data-analytics
author: ai-skills-maintainer
created: 2026-04-21
updated: 2026-04-21
tags:
  - workflow
  - data-analysis
  - pipeline
  - end-to-end
  - multi-asset
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
description: |
  A comprehensive end-to-end workflow demonstrating how multiple assets (scripts, prompts,
  skills, and agents) work together in a complete data analysis pipeline — from raw CSV
  files to actionable business insights and recommendations.
security_classification: internal
delivery_modes:
  - llm-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
dependencies:
  - data-analytics-csv-to-json-transformer
  - shared-data-cleaner
  - data-analytics-statistical-analysis-prompt
  - data-analytics-sales-trend-analyzer
  - data-analytics-data-insights-agent
inputs:
  - name: RAW_DATA_FILE
    type: string
    description: "Path to raw CSV file containing sales or business data"
    required: true
  - name: ANALYSIS_TYPE
    type: string
    description: "Type of analysis: sales-trends | general-statistics | custom"
    required: true
  - name: BUSINESS_CONTEXT
    type: string
    description: "Business context for analysis (e.g., 'Q1 e-commerce performance')"
    required: false
outputs:
  - name: ANALYSIS_REPORT
    type: string
    description: "Complete analysis report with insights and recommendations"
---

## Overview

This workflow demonstrates a **complete multi-asset data analysis pipeline** that showcases how different types of assets work together:

1. **Scripts** — Data transformation and cleaning (Python)
2. **Prompt** — Statistical analysis template
3. **Skill** — Focused capability (sales trend analysis)
4. **Agent** — Autonomous orchestration of all components
5. **Workflow** — End-to-end process documentation (you are here)

This is the **reference implementation** for understanding how to structure, develop, and use multiple assets in the AI-Skills repository.

## Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        INPUT: RAW_DATA_FILE                         │
│                       (sales_data.csv from source)                  │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 1: Data Transformation                                         │
│ Asset: data-analytics-csv-to-json-transformer (script)              │
│ Action: Convert CSV to structured JSON with type inference          │
│ Output: /tmp/structured_data.json                                   │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 2: Data Cleaning                                               │
│ Asset: shared-data-cleaner (script)                                 │
│ Action: Remove duplicates, handle missing values, detect outliers   │
│ Output: /tmp/cleaned_data.json + quality_report.json                │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 3: Statistical Analysis                                        │
│ Asset: data-analytics-statistical-analysis-prompt (prompt)          │
│ Action: Generate descriptive statistics, detect patterns            │
│ Output: Statistical summary + initial insights                      │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 4: Specialized Analysis (conditional)                          │
│ Asset: data-analytics-sales-trend-analyzer (skill)                  │
│ Action: Analyze trends, generate forecasts, detect anomalies        │
│ Condition: Only if ANALYSIS_TYPE = "sales-trends"                   │
│ Output: Trend analysis + 7-day forecast + recommendations           │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 5: Insight Synthesis (optional — automated)                    │
│ Asset: data-analytics-data-insights-agent (agent)                   │
│ Action: Orchestrate Steps 1-4 automatically + synthesize report     │
│ Note: Can replace manual Steps 1-4 for full automation              │
│ Output: Comprehensive report with executive summary                 │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    OUTPUT: ANALYSIS_REPORT                          │
│          (Complete insights + recommendations + forecast)           │
└─────────────────────────────────────────────────────────────────────┘
```

## Steps

### Step 1 — Data Transformation

**Asset:** `data-analytics-csv-to-json-transformer` (script)  
**Type:** Script (Python)  
**Input:** `{{RAW_DATA_FILE}}` (CSV)  
**Output:** `/tmp/structured_data.json`

**Purpose:** Convert flat CSV file into structured, typed JSON that LLMs can consume efficiently.

**Manual invocation:**
```bash
python workstreams/data-analytics/scripts/csv_to_json.py \
  --input {{RAW_DATA_FILE}} \
  --output /tmp/structured_data.json \
  --infer-types true
```

**What it does:**
- Reads CSV file
- Infers data types (numbers, dates, booleans)
- Optionally nests objects (e.g., `address.city` → `{address: {city: value}}`)
- Outputs structured JSON

---

### Step 2 — Data Cleaning

**Asset:** `shared-data-cleaner` (script)  
**Type:** Script (Python) — Common utility in `shared/scripts/`  
**Input:** `/tmp/structured_data.json`  
**Output:** `/tmp/cleaned_data.json` + `/tmp/cleaned_data_quality_report.json`

**Purpose:** Clean and validate data before analysis.

**Manual invocation:**
```bash
python shared/scripts/data_cleaner.py \
  --input /tmp/structured_data.json \
  --output /tmp/cleaned_data.json \
  --remove-duplicates true \
  --fill-missing mean
```

**What it does:**
- Removes duplicate rows
- Handles missing values (drop, mean, median, zero)
- Detects outliers using IQR method
- Generates quality report with score 0-100

---

### Step 3 — Statistical Analysis

**Asset:** `data-analytics-statistical-analysis-prompt` (prompt)  
**Type:** Prompt template  
**Input:** `/tmp/cleaned_data.json`  
**Output:** Statistical summary + key insights

**Purpose:** Generate descriptive statistics and identify patterns.

**Manual invocation:**
```
[Copy prompt from prompts/statistical-analysis-prompt.md]

Substitute placeholders:
- DATA: [paste contents of /tmp/cleaned_data.json]
- ANALYSIS_FOCUS: all
- BUSINESS_CONTEXT: {{BUSINESS_CONTEXT}}

Paste into LLM Chat or your LLM
```

**What it does:**
- Calculates descriptive statistics (mean, median, std dev, min, max)
- Identifies trends, anomalies, correlations
- Extracts key insights with supporting data
- Provides business recommendations

---

### Step 4 — Specialized Analysis (Conditional)

**Asset:** `data-analytics-sales-trend-analyzer` (skill)  
**Type:** Skill (focused capability)  
**Input:** `/tmp/cleaned_data.json`  
**Output:** Trend analysis + forecast + recommendations  
**Condition:** Only if `{{ANALYSIS_TYPE}}` = `"sales-trends"`

**Purpose:** Deep-dive sales trend analysis with forecasting.

**Manual invocation:**
```
[Copy skill definition from skills/sales-trend-analyzer.md]

Substitute placeholders:
- SALES_DATA: [paste contents of /tmp/cleaned_data.json]
- TIME_PERIOD: daily
- FORECAST_DAYS: 7

Paste into LLM Chat or your LLM
```

**What it does:**
- Identifies trends with growth rates and strength (R²)
- Detects anomalies (spikes, dips >2 SD)
- Generates 7-day forecast using linear regression
- Provides prioritized business recommendations

---

### Step 5 — Automated Orchestration (Optional)

**Asset:** `data-analytics-data-insights-agent` (agent)  
**Type:** Autonomous agent  
**Input:** `{{RAW_DATA_FILE}}`, `{{ANALYSIS_TYPE}}`, `{{BUSINESS_CONTEXT}}`  
**Output:** Comprehensive report (executes Steps 1-4 automatically)

**Purpose:** Fully automate the pipeline — agent orchestrates all scripts and tools.

**Manual invocation:**
```
@workspace You are a data insights agent.

Analyze this data:
Data source: {{RAW_DATA_FILE}}
Analysis type: {{ANALYSIS_TYPE}}
Business context: {{BUSINESS_CONTEXT}}
```

**What it does:**
- Automatically calls csv-to-json-transformer
- Automatically calls data-cleaner
- Automatically calls statistical-analysis-prompt
- Automatically calls sales-trend-analyzer (if applicable)
- Synthesizes all outputs into comprehensive report with executive summary

---

## Usage Modes

### Mode 1: Manual (Step-by-Step)

**Use when:** You want full control over each step, or you're learning the pipeline.

```bash
# Step 1
python workstreams/data-analytics/scripts/csv_to_json.py \
  --input data/sales.csv --output /tmp/sales.json

# Step 2
python shared/scripts/data_cleaner.py \
  --input /tmp/sales.json --output /tmp/sales_clean.json

# Step 3: Copy prompt, paste data, run in Copilot Chat

# Step 4: Copy skill, paste cleaned data, run in Copilot Chat
```

### Mode 2: Semi-Automated (Scripts + Manual Analysis)

**Use when:** You want automated data prep but manual analysis.

```bash
# Automate Steps 1-2
./run_data_prep.sh data/sales.csv /tmp/sales_clean.json

# Then manually run Step 3 & 4 prompts/skills in Copilot Chat
```

### Mode 3: Fully Automated (Agent)

**Use when:** You want end-to-end automation.

```
@workspace You are a data insights agent.

Data source: data/sales.csv
Analysis type: sales-trends
Business context: Q1 e-commerce electronics
```

The agent executes all steps and produces a comprehensive report.

---

## Complete Example

### Input

**File:** `data/widget_sales_q1.csv`
```csv
date,product,quantity,revenue
2026-01-01,Widget A,100,1000
2026-01-02,Widget A,120,1200
2026-01-03,Widget A,95,950
[...90 more rows...]
```

**Parameters:**
- `ANALYSIS_TYPE`: `sales-trends`
- `BUSINESS_CONTEXT`: `E-commerce electronics, Q1 2026 performance review`

### Execution

**Option 1: Manual**
```bash
# Step 1
python workstreams/data-analytics/scripts/csv_to_json.py \
  --input data/widget_sales_q1.csv \
  --output /tmp/widget_q1.json

# Step 2
python shared/scripts/data_cleaner.py \
  --input /tmp/widget_q1.json \
  --output /tmp/widget_q1_clean.json

# Step 3 & 4: Use Copilot Chat with prompts
```

**Option 2: Fully Automated**
```
@workspace You are a data insights agent.

Data source: data/widget_sales_q1.csv
Analysis type: sales-trends
Business context: E-commerce electronics, Q1 2026 performance review
```

### Output

```markdown
# Widget A Sales Analysis — Q1 2026

## Executive Summary
- **85% growth** in Q1 (1,000 → 1,850 units/month)
- **Data quality**: 94/100 (6% duplicates removed)
- **Key risk**: Inventory capacity insufficient for forecasted demand
- **Opportunity**: Saturday sales 40% higher — optimize weekend marketing
- **Action**: Increase inventory immediately; test price increase

## Data Quality Report
- Original rows: 93
- Final rows: 87 (6 duplicates, 0 missing)
- Quality score: 94/100

## Statistical Summary
[Full statistics table]

## Trend Analysis
- Growth rate: 2.1% per day (85% cumulative)
- Trend strength: R² = 0.94 (very strong)
- Forecast: 2,500+ units/day by mid-April

## Key Insights
1. Sustained growth momentum with R²=0.94
2. Weekend effect: +40% Saturday sales
3. Tuesday dip: -15% every Tuesday
4. Inventory risk: current capacity insufficient
5. Pricing power: no elasticity detected

## Business Recommendations
1. URGENT: Increase inventory (Critical priority)
2. Optimize weekend marketing (High priority)
3. Investigate Tuesday dip (High priority)
4. Test strategic price increase (Medium priority)
5. Implement real-time monitoring (Medium priority)
```

---

## Asset Reference Table

| Asset | Type | Location | Role in Workflow |
|---|---|---|---|
| `csv-to-json-transformer` | Script | `workstreams/data-analytics/scripts/` | Step 1: Transform CSV to JSON |
| `data-cleaner` | Script | `shared/scripts/` | Step 2: Clean and validate data |
| `statistical-analysis-prompt` | Prompt | `workstreams/data-analytics/prompts/` | Step 3: Generate statistics |
| `sales-trend-analyzer` | Skill | `workstreams/data-analytics/skills/` | Step 4: Analyze trends |
| `data-insights-agent` | Agent | `workstreams/data-analytics/agents/` | Step 5: Automate entire pipeline |

---

## Key Concepts Demonstrated

### 1. Script Reusability
- **Common script** (`data-cleaner`) in `shared/scripts/` — used by multiple workstreams
- **Skill-specific script** (`csv-to-json-transformer`) in workstream folder — tailored to data-analytics needs

### 2. Asset Composition
- **Prompts** provide templates for specific tasks
- **Skills** are focused capabilities built on prompts
- **Agents** orchestrate multiple skills and scripts
- **Workflows** document the complete process

### 3. Progressive Enhancement
- **Level 1**: Manual scripts only
- **Level 2**: Scripts + prompts (semi-automated)
- **Level 3**: Scripts + prompts + skills (structured)
- **Level 4**: Agent automates all (fully automated)

### 4. Pipeline Flexibility
- Each step can be run independently
- Steps can be skipped if not needed
- Alternative paths based on `ANALYSIS_TYPE`

---

## Testing Notes

| Mode | Tested | Notes |
|---|---|---|
| Manual (Steps 1-4) | ✅ | 2026-04-21. All steps work independently. |
| Semi-Automated | ✅ | 2026-04-21. Scripts pipeline correctly. |
| Fully Automated (Agent) | ✅ | 2026-04-21. Agent orchestrates all steps successfully. |

---

## Changelog

### 1.0.0 — 2026-04-21
- Initial version.
- Demonstrates complete multi-asset pipeline.
- Includes manual, semi-automated, and fully automated execution modes.
