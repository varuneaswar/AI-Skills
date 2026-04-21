---
id: data-analytics-statistical-analysis-prompt
title: Statistical Analysis and Insights Generator
type: prompt
version: "1.0.0"
status: active
workstream:
  - data-analytics
author: ai-skills-maintainer
created: 2026-04-21
updated: 2026-04-21
tags:
  - statistics
  - analysis
  - insights
  - data-science
  - reporting
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
  - gemini-1.5-pro
  - copilot-gpt-4o
description: |
  Generates comprehensive statistical analysis and actionable insights from structured data,
  including descriptive statistics, trend identification, anomaly detection, and business
  recommendations — transforming raw numbers into decision-ready insights.
security_classification: internal
delivery_modes:
  - copilot-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
inputs:
  - name: DATA
    type: string
    description: "Structured data in JSON format (array of objects)"
    required: true
  - name: ANALYSIS_FOCUS
    type: string
    description: "What to analyze: trends | anomalies | correlations | distributions | all (default: all)"
    required: false
  - name: BUSINESS_CONTEXT
    type: string
    description: "Business context to inform recommendations (e.g., 'e-commerce sales', 'customer support tickets')"
    required: false
outputs:
  - name: STATISTICAL_SUMMARY
    type: string
    description: "Descriptive statistics for all numeric columns"
  - name: KEY_INSIGHTS
    type: string
    description: "Top 3-5 insights from the data"
  - name: RECOMMENDATIONS
    type: string
    description: "Actionable business recommendations"
---

## Overview

This prompt transforms structured data into comprehensive statistical analysis and business insights. It's designed to work with clean, typed JSON data (typically output from csv-to-json-transformer or data-cleaner scripts) and produce both technical statistical summaries and business-friendly recommendations.

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `{{DATA}}` | string | Yes | JSON array of data objects |
| `{{ANALYSIS_FOCUS}}` | string | No | Analysis focus: `trends`, `anomalies`, `correlations`, `distributions`, or `all` (default) |
| `{{BUSINESS_CONTEXT}}` | string | No | Business context (e.g., "e-commerce sales", "support tickets") |

## Outputs

| Name | Type | Description |
|---|---|---|
| `STATISTICAL_SUMMARY` | string | Descriptive statistics (mean, median, std dev, min, max) |
| `KEY_INSIGHTS` | string | Top 3-5 insights identified in the data |
| `RECOMMENDATIONS` | string | Actionable business recommendations |

## Prompt

```
[System]
You are a data analyst and business intelligence expert who transforms data into
actionable insights. You analyze structured data and provide both statistical summaries
and business recommendations.

[Task]
Analyze the following data and provide comprehensive insights.

Data:
{{DATA}}

Analysis Focus: {{ANALYSIS_FOCUS}}
Business Context: {{BUSINESS_CONTEXT}}

[Instructions]
Perform the following analysis:

1. **Descriptive Statistics**
   - For all numeric columns, calculate: count, mean, median, std deviation, min, max
   - Identify the data range and distribution characteristics
   - Note any missing or zero values

2. **Trend Analysis** (if {{ANALYSIS_FOCUS}} includes "trends" or "all")
   - Identify upward or downward trends over time
   - Calculate growth rates or percentage changes
   - Highlight significant shifts or inflection points

3. **Anomaly Detection** (if {{ANALYSIS_FOCUS}} includes "anomalies" or "all")
   - Identify outliers (values > 2 standard deviations from mean)
   - Flag unusual patterns or unexpected values
   - Note data quality issues

4. **Correlations** (if {{ANALYSIS_FOCUS}} includes "correlations" or "all")
   - Identify relationships between numeric columns
   - Highlight strong positive or negative correlations
   - Suggest causal hypotheses (correlation ≠ causation)

5. **Key Insights**
   - Extract the top 3-5 most important insights from the analysis
   - Each insight should be specific, data-backed, and business-relevant
   - Include supporting numbers/percentages

6. **Business Recommendations**
   - Based on the insights and {{BUSINESS_CONTEXT}}, provide 3-5 actionable recommendations
   - Each recommendation should be specific, measurable, and achievable
   - Prioritize recommendations by potential impact

[Output Format]

## Statistical Summary

| Metric | [Column1] | [Column2] | ... |
|---|---|---|---|
| Count | ... | ... | ... |
| Mean | ... | ... | ... |
| Median | ... | ... | ... |
| Std Dev | ... | ... | ... |
| Min | ... | ... | ... |
| Max | ... | ... | ... |

## Key Insights

1. **[Insight title]**: [Description with supporting data]
2. **[Insight title]**: [Description with supporting data]
3. **[Insight title]**: [Description with supporting data]

[Add more as needed]

## Trends

[If analyzing trends, describe patterns over time with growth rates]

## Anomalies

[If detecting anomalies, list outliers and unusual patterns]

## Correlations

[If analyzing correlations, describe relationships between variables]

## Business Recommendations

1. **[Recommendation title]** (Priority: High/Medium/Low)
   - Action: [Specific action to take]
   - Expected Impact: [What will improve]
   - Supporting Data: [Why this recommendation]

2. [Continue for 3-5 recommendations]

[Rules]
- Base all insights on actual data patterns — do not invent trends
- Use clear, non-technical language for business recommendations
- Include specific numbers/percentages to support each insight
- If data is insufficient for certain analyses, state that clearly
- Prioritize actionable insights over academic observations
```

## Usage

### GitHub Copilot Chat (current)

```
Analyze this data:
{{DATA}}

Focus on: {{ANALYSIS_FOCUS}}
Context: {{BUSINESS_CONTEXT}}
```

### API Endpoint (future portal)

```http
POST {{API_BASE_URL}}/packages/data-analytics-pack/invoke
Content-Type: application/json

{
  "prompt_id": "data-analytics-statistical-analysis-prompt",
  "inputs": {
    "data": "[{...}, {...}]",
    "analysis_focus": "trends",
    "business_context": "e-commerce sales"
  }
}
```

## Examples

### Example 1 — E-commerce sales analysis

**Input:**
- `DATA`:
```json
[
  {"date": "2026-01-01", "product": "Widget A", "quantity": 100, "revenue": 1000},
  {"date": "2026-01-02", "product": "Widget A", "quantity": 120, "revenue": 1200},
  {"date": "2026-01-03", "product": "Widget A", "quantity": 95, "revenue": 950},
  {"date": "2026-01-04", "product": "Widget A", "quantity": 150, "revenue": 1500},
  {"date": "2026-01-05", "product": "Widget A", "quantity": 180, "revenue": 1800}
]
```
- `ANALYSIS_FOCUS`: `trends`
- `BUSINESS_CONTEXT`: `e-commerce sales for Widget A product line`

**Output:**

```markdown
## Statistical Summary

| Metric | quantity | revenue |
|---|---|---|
| Count | 5 | 5 |
| Mean | 129 | 1290 |
| Median | 120 | 1200 |
| Std Dev | 33.2 | 332 |
| Min | 95 | 950 |
| Max | 180 | 1800 |

## Key Insights

1. **Strong upward sales trend**: Widget A sales increased 80% from Day 1 (100 units) to Day 5 (180 units), indicating growing demand.

2. **Consistent pricing**: Revenue correlates perfectly with quantity (1:10 ratio), suggesting stable $10 unit price with no discounting.

3. **Day 3 dip**: Sales dropped 21% on Day 3, possibly due to weekend or external factor — worth investigating.

4. **Acceleration in growth**: Days 4-5 showed 60% growth (95→180 units), faster than Days 1-2 (20% growth), suggesting momentum.

## Trends

- **Overall trend**: Strong upward trajectory with 40% average daily growth rate
- **Recent acceleration**: Last 2 days show 30% daily growth vs. 10% in first 3 days
- **Projected**: If trend continues, expect ~200+ units/day by end of week

## Business Recommendations

1. **Increase inventory for Widget A** (Priority: High)
   - Action: Order 2x current stock levels to meet accelerating demand
   - Expected Impact: Prevent stockouts, maintain momentum
   - Supporting Data: 80% growth in 5 days with acceleration

2. **Investigate Day 3 sales dip** (Priority: Medium)
   - Action: Analyze external factors (day of week, marketing, competition)
   - Expected Impact: Identify preventable causes of sales variation
   - Supporting Data: 21% drop on Day 3 vs. consistent growth otherwise

3. **Capitalize on momentum with promotion** (Priority: Medium)
   - Action: Launch targeted promotion for Days 6-7 to sustain growth
   - Expected Impact: Extend trend, acquire new customers
   - Supporting Data: 40% daily growth rate provides runway for promotion
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | 2026-04-21. Produces accurate statistics and actionable recommendations. |
| claude-3-5-sonnet | ✅ | 2026-04-21. More detailed correlation analysis than gpt-4o. |

## Changelog

### 1.0.0 — 2026-04-21
- Initial version.
