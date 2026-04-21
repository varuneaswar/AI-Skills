---
id: data-analytics-data-insights-agent
title: Data Insights Agent
type: agent
version: "1.0.0"
status: active
workstream:
  - data-analytics
author: ai-skills-maintainer
created: 2026-04-21
updated: 2026-04-21
tags:
  - agent
  - data-analysis
  - insights
  - automation
  - business-intelligence
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
  - copilot-gpt-4o
description: |
  An autonomous agent that orchestrates the complete data analysis pipeline: data
  transformation, cleaning, statistical analysis, trend identification, and insight
  generation — producing a comprehensive analysis report with recommendations automatically.
security_classification: internal
delivery_modes:
  - copilot-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
dependencies:
  - data-analytics-csv-to-json-transformer
  - shared-data-cleaner
  - data-analytics-statistical-analysis-prompt
  - data-analytics-sales-trend-analyzer
inputs:
  - name: DATA_SOURCE
    type: string
    description: "Path to CSV or JSON file, or raw data URL"
    required: true
  - name: ANALYSIS_TYPE
    type: string
    description: "Type of analysis: sales-trends | general-statistics | custom"
    required: true
  - name: BUSINESS_CONTEXT
    type: string
    description: "Business context for recommendations (e.g., 'e-commerce', 'SaaS metrics')"
    required: false
outputs:
  - name: COMPREHENSIVE_REPORT
    type: string
    description: "Complete analysis report with statistics, trends, insights, and recommendations"
  - name: DATA_QUALITY_SCORE
    type: string
    description: "Data quality assessment (0-100)"
  - name: EXECUTIVE_SUMMARY
    type: string
    description: "Executive summary with top 3 insights and recommendations"
---

## Overview

The Data Insights Agent is an autonomous agent that orchestrates multiple scripts, prompts, and skills to transform raw data into actionable business insights. It handles the complete pipeline automatically:

1. **Data Transformation** — Convert CSV to structured JSON
2. **Data Cleaning** — Remove duplicates, handle missing values, detect outliers
3. **Statistical Analysis** — Generate descriptive statistics
4. **Trend Analysis** — Identify patterns, forecast, detect anomalies
5. **Insight Generation** — Extract key insights and recommendations

Human analysts use this agent to automate the tedious parts of data analysis and focus their time on strategic decision-making.

## Architecture

**Tools / Skills Used:**

| Tool / Skill | Purpose |
|---|---|---|
| `data-analytics-csv-to-json-transformer` | Transform CSV to structured JSON |
| `shared-data-cleaner` | Clean and validate data |
| `data-analytics-statistical-analysis-prompt` | Generate statistical summary |
| `data-analytics-sales-trend-analyzer` | Analyze trends (if analysis_type = sales-trends) |
| `generate_report` | Synthesize all analyses into comprehensive report |

**Decision Flow:**

```
DATA_SOURCE + ANALYSIS_TYPE + BUSINESS_CONTEXT
          │
          ▼
Step 1: Check data format (CSV? JSON? URL?)
          │
          ├─ CSV ──▶ Call csv-to-json-transformer
          ├─ JSON ─▶ Skip to Step 2
          └─ URL ──▶ Download, then route to CSV or JSON path
                    │
                    ▼
          Step 2: Call data-cleaner
                    │
                    ▼
          Step 3: Call statistical-analysis-prompt
                    │
                    ▼
          Step 4: If ANALYSIS_TYPE = "sales-trends"
                    │
                    ├─ Yes ──▶ Call sales-trend-analyzer
                    └─ No ───▶ Skip to Step 5
                              │
                              ▼
          Step 5: Synthesize all outputs into COMPREHENSIVE_REPORT
                    │
                    ▼
          COMPREHENSIVE_REPORT + DATA_QUALITY_SCORE + EXECUTIVE_SUMMARY
```

## System Prompt / Agent Instructions

```
[System Prompt]

You are an autonomous data insights agent for business analysts and data scientists.

Your goals:
1. Transform raw data (CSV or JSON) into clean, analysis-ready format
2. Perform comprehensive statistical analysis and trend detection
3. Generate actionable insights and business recommendations
4. Produce a complete analysis report with executive summary

Tools available to you:
- csv_to_json: call with (input_csv, output_json) → returns structured JSON
- data_cleaner: call with (input_file, output_file, remove_duplicates, fill_missing) → returns cleaned data + quality report
- statistical_analysis: call with (data, analysis_focus, business_context) → returns statistics + insights
- sales_trend_analyzer: call with (sales_data, time_period, forecast_days) → returns trends + forecast
- generate_report: call with (all_outputs) → returns comprehensive_report + executive_summary

Rules:
- Always start by checking data format (CSV vs JSON)
- Always clean data before analysis (call data_cleaner)
- If data quality score < 60%, flag as "Low Quality" in report
- For sales-trends analysis, call sales_trend_analyzer after statistical_analysis
- Synthesize all tool outputs into a coherent narrative report
- Executive summary should be 3-5 bullet points max
- Never invent data or insights not supported by the analysis

Output format:

## Executive Summary
[3-5 key takeaways]

## Data Quality Assessment
- Quality Score: [0-100]
- [Summary of data cleaning actions]

## Statistical Analysis
[Full statistics from statistical_analysis tool]

## Trend Analysis
[Full trend analysis from sales_trend_analyzer, if applicable]

## Key Insights
[Top 5 insights synthesized from all analyses]

## Business Recommendations
[5-7 actionable recommendations with priorities]

## Appendix
- Data source: [path]
- Analysis type: [type]
- Date generated: [timestamp]
```

## Configuration

```yaml
agent_id: data-analytics-data-insights-agent
model: gpt-4o
temperature: 0.2         # low for consistent, deterministic analysis
max_iterations: 10       # transform → clean → stats → trends → synthesize → format
tools:
  - csv_to_json
  - data_cleaner
  - statistical_analysis
  - sales_trend_analyzer
  - generate_report
```

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `{{DATA_SOURCE}}` | string | Yes | Path to CSV/JSON file or data URL |
| `{{ANALYSIS_TYPE}}` | string | Yes | `sales-trends`, `general-statistics`, or `custom` |
| `{{BUSINESS_CONTEXT}}` | string | No | Business context for recommendations |

## Outputs

| Name | Type | Description |
|---|---|---|
| `COMPREHENSIVE_REPORT` | string | Complete analysis report (Markdown format) |
| `DATA_QUALITY_SCORE` | string | Quality score 0-100 |
| `EXECUTIVE_SUMMARY` | string | 3-5 key takeaways |

## Security Considerations

- The agent reads data files but never writes to the source
- All intermediate files are written to `/tmp/` (ephemeral)
- The agent must not download data from untrusted URLs without validation
- Business context may contain sensitive information — route through `in-house-llm` for confidential analyses

## Usage

### GitHub Copilot Chat (current)

```
@workspace You are a data insights agent.

Analyze this data:
Data source: data/sales.csv
Analysis type: sales-trends
Business context: e-commerce sales for Q1 2026
```

### API Endpoint (future portal)

```http
POST {{API_BASE_URL}}/packages/data-analytics-pack/invoke
Content-Type: application/json

{
  "agent_id": "data-analytics-data-insights-agent",
  "inputs": {
    "data_source": "s3://bucket/data/sales_2026_q1.csv",
    "analysis_type": "sales-trends",
    "business_context": "E-commerce electronics division, focus on revenue growth and inventory optimization"
  }
}
```

## Examples

### Example 1 — E-commerce sales analysis

**Input:**
- `DATA_SOURCE`: `data/sales.csv` (contains 100 rows of daily sales data)
- `ANALYSIS_TYPE`: `sales-trends`
- `BUSINESS_CONTEXT`: `E-commerce electronics, focus on Q1 performance`

**Output (abbreviated):**

```markdown
# Data Insights Report
Generated: 2026-04-21 11:30:00

## Executive Summary

- **Strong Growth**: Sales increased 85% over 30 days, from 1,200 to 2,220 units/day
- **Data Quality**: 92% quality score — 8 duplicate rows removed, 2% missing values filled
- **Key Risk**: Inventory levels may not support forecasted 2,500+ units/day by mid-April
- **Opportunity**: Day-of-week pattern shows Saturday sales 40% higher — optimize marketing
- **Action Required**: Increase inventory immediately; investigate Tuesday sales dip

## Data Quality Assessment

- **Quality Score**: 92/100
- **Original rows**: 100
- **Final rows**: 92 (8 duplicates removed)
- **Missing values**: 2% filled using mean strategy
- **Outliers detected**: 3 (days with promotions, marked but not removed)

## Statistical Analysis

| Metric | quantity | revenue |
|---|---|---|---|
| Count | 92 | 92 |
| Mean | 1,650 | $16,500 |
| Median | 1,600 | $16,000 |
| Std Dev | 420 | $4,200 |
| Min | 1,200 | $12,000 |
| Max | 2,220 | $22,200 |

**Correlation**: Quantity and revenue perfectly correlated (r=1.0), indicating stable $10/unit pricing.

## Trend Analysis

**Overall Trend**: Strong Upward
- Growth Rate: 85% over 30 days (2.1% average daily growth)
- Trend Strength: Very Strong (R² = 0.94)

**Forecast (Next 7 Days)**:

| Date | Forecasted Units | Revenue |
|---|---|---|
| 2026-04-08 | 2,280 | $22,800 |
| 2026-04-09 | 2,340 | $23,400 |
| 2026-04-10 | 2,400 | $24,000 |
| [continues...] | ... | ... |

**Confidence**: High (R² = 0.94, consistent trend)

## Key Insights

1. **Sustained growth momentum**: 85% growth with strong R²=0.94 indicates genuine demand increase, not random variation
2. **Weekend effect**: Saturday sales 40% higher than weekday average — clear opportunity for targeted weekend marketing
3. **Tuesday dip**: Sales drop 15% every Tuesday — investigate cause (competitor promotions? Supply chain delays?)
4. **Inventory risk**: Current forecast (2,500+ units/day by mid-April) exceeds reported inventory capacity
5. **Pricing stability**: Perfect price-volume correlation suggests pricing power — room for strategic price testing

## Business Recommendations

1. **URGENT: Increase Inventory** (Priority: Critical)
   - Action: Order 50,000+ units immediately to support forecasted demand through end of April
   - Impact: Prevent stockouts that would halt growth momentum
   - Data: Forecast shows 2,500+ units/day demand; current inventory supports only 15 more days

2. **Optimize Weekend Marketing** (Priority: High)
   - Action: Shift 40% of marketing budget to Friday-Sunday campaigns
   - Impact: Capitalize on 40% higher weekend demand; expect 15-20% revenue increase
   - Data: Saturday sales average 2,100 units vs. 1,500 weekday average

3. **Investigate Tuesday Dip** (Priority: High)
   - Action: Analyze competitor activity, supply chain, and customer behavior on Tuesdays
   - Impact: Eliminate 15% weekly revenue loss if cause is addressable
   - Data: Tuesday sales 15% below weekly average across all 4+ weeks analyzed

4. **Test Strategic Price Increase** (Priority: Medium)
   - Action: A/B test $10.50 pricing on 25% of traffic
   - Impact: 5% revenue increase with minimal demand impact (strong brand signals price inelasticity)
   - Data: Perfect correlation (r=1.0) between price and volume; no elasticity detected

5. **Implement Real-Time Demand Monitoring** (Priority: Medium)
   - Action: Set up daily dashboard tracking vs. forecast; alert if actual < 90% of forecast
   - Impact: Early warning of trend breaks or market saturation
   - Data: 85% monthly growth requires constant monitoring for sustainability

6. **Expand Product Line** (Priority: Low)
   - Action: Introduce complementary products to capture additional wallet share
   - Impact: Increase average order value by 20-30%
   - Data: Strong demand for primary product suggests healthy customer base for expansion

## Appendix

- **Data source**: `data/sales.csv`
- **Analysis type**: `sales-trends`
- **Business context**: `E-commerce electronics, Q1 focus`
- **Date generated**: `2026-04-21 11:30:00 UTC`
- **Quality score**: `92/100`
- **Tools used**: `csv-to-json-transformer`, `data-cleaner`, `statistical-analysis-prompt`, `sales-trend-analyzer`
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | 2026-04-21. Successfully orchestrates all tools. Report quality is excellent. |
| claude-3-5-sonnet | ✅ | 2026-04-21. More detailed insights section than gpt-4o. |

## Changelog

### 1.0.0 — 2026-04-21
- Initial version.
- Supports CSV and JSON input.
- Orchestrates 4 tools: csv-to-json, data-cleaner, statistical-analysis, sales-trend-analyzer.
