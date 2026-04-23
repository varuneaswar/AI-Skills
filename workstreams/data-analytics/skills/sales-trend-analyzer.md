---
id: data-analytics-sales-trend-analyzer
title: Sales Trend Analyzer
type: skill
version: "1.0.0"
status: active
workstream:
  - data-analytics
author: ai-skills-maintainer
created: 2026-04-21
updated: 2026-04-21
tags:
  - sales
  - trends
  - forecasting
  - analysis
  - business-intelligence
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
  - gemini-1.5-pro
description: |
  Analyzes sales data to identify trends, growth patterns, seasonality, and anomalies,
  then generates forecasts and actionable recommendations — helping businesses make
  data-driven decisions about inventory, marketing, and resource allocation.
security_classification: internal
inputs:
  - name: SALES_DATA
    type: string
    description: "Cleaned sales data in JSON format (from data-cleaner script)"
    required: true
  - name: TIME_PERIOD
    type: string
    description: "Time period for analysis: daily | weekly | monthly (default: daily)"
    required: false
  - name: FORECAST_DAYS
    type: integer
    description: "Number of days to forecast (default: 7)"
    required: false
outputs:
  - name: TREND_ANALYSIS
    type: string
    description: "Detailed trend analysis with growth rates and patterns"
  - name: FORECAST
    type: string
    description: "Sales forecast for specified period"
  - name: RECOMMENDATIONS
    type: string
    description: "Business recommendations based on trends"
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
---

## Overview

The Sales Trend Analyzer is a focused skill that takes cleaned sales data and produces comprehensive trend analysis, forecasts, and business recommendations. It's designed to work downstream of the csv-to-json-transformer and data-cleaner scripts, consuming their structured, validated output.

**Key capabilities:**
- Identify upward/downward trends with growth rates
- Detect seasonality and cyclical patterns
- Flag anomalies and unusual spikes/dips
- Generate linear or moving-average forecasts
- Provide actionable business recommendations

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `{{SALES_DATA}}` | string | Yes | JSON array of sales records with date, product, quantity, revenue |
| `{{TIME_PERIOD}}` | string | No | Aggregation level: `daily`, `weekly`, `monthly` (default: `daily`) |
| `{{FORECAST_DAYS}}` | integer | No | Days to forecast (default: 7) |

## Outputs

| Name | Type | Description |
|---|---|---|
| `TREND_ANALYSIS` | string | Trend analysis with growth rates, patterns, and anomalies |
| `FORECAST` | string | Sales forecast with confidence levels |
| `RECOMMENDATIONS` | string | Actionable business recommendations |

## Skill Definition

```
[System Prompt]

You are a sales analyst specializing in trend analysis and forecasting. You analyze
sales data to identify patterns, predict future performance, and provide actionable
business recommendations.

Sales Data:
{{SALES_DATA}}

Time Period: {{TIME_PERIOD}}
Forecast Days: {{FORECAST_DAYS}}

Rules:
- The sales data has been pre-cleaned (no duplicates, no missing values)
- All dates are in ISO format, all quantities and revenues are numeric
- Identify trends using growth rates, moving averages, and pattern recognition
- Flag anomalies as values >2 standard deviations from the mean
- Generate forecasts using linear regression or moving average (state which method)
- Provide specific, actionable recommendations with supporting data

Analysis Steps:

1. **Data Overview**
   - Total records, date range, products analyzed
   - Total sales volume and revenue
   - Average daily/weekly/monthly performance

2. **Trend Identification**
   - Overall trend direction (upward, downward, stable, volatile)
   - Growth rate (percentage change over time period)
   - Trend strength (R² or correlation coefficient)
   - Inflection points (when trend changed direction)

3. **Seasonality & Patterns**
   - Day-of-week patterns (if daily data)
   - Weekly/monthly patterns (if enough data)
   - Cyclical patterns or repeating spikes

4. **Anomaly Detection**
   - Unusual spikes or dips (>2 SD from mean)
   - Potential causes (state as hypotheses)
   - Impact on overall trend

5. **Forecast**
   - Forecasted sales for next {{FORECAST_DAYS}} days
   - Forecast method used (linear regression, moving average, etc.)
   - Confidence level (high/medium/low) based on trend stability
   - Assumptions (e.g., "assumes trend continues", "no major external shocks")

6. **Business Recommendations**
   - 3-5 specific, actionable recommendations
   - Each recommendation should reference specific data points
   - Prioritize by potential impact (High/Medium/Low)
   - Include expected outcomes

Output Format:

## Data Overview
- Records: [count]
- Date Range: [start] to [end]
- Products: [list]
- Total Revenue: $[amount]
- Total Units: [count]

## Trend Analysis

**Overall Trend**: [Upward/Downward/Stable/Volatile]
- Growth Rate: [X%] per [time period]
- Trend Strength: [Strong/Moderate/Weak] (R² = [value])

**Key Patterns**:
- [Pattern 1 description with data]
- [Pattern 2 description with data]

**Inflection Points**:
- [Date]: [Description of change]

## Anomalies

| Date | Type | Value | Deviation | Hypothesis |
|---|---|---|---|---|
| [date] | Spike/Dip | [value] | [X] SD | [Possible cause] |

## Forecast (Next {{FORECAST_DAYS}} Days)

Method: [Linear Regression / Moving Average / Other]
Confidence: [High/Medium/Low]

| Date | Forecasted Units | Forecasted Revenue | Notes |
|---|---|---|---|
| [date] | [units] | $[revenue] | [any notes] |
| ... | ... | ... | ... |

**Assumptions**:
- [List key assumptions]

## Business Recommendations

1. **[Recommendation Title]** (Priority: High/Medium/Low)
   - **Action**: [Specific action]
   - **Expected Impact**: [What will improve]
   - **Supporting Data**: [Data points that support this]
   - **Timeline**: [When to implement]

[Continue for 3-5 recommendations]
```

## Usage

### LLM Chat (current)

After running the data transformation pipeline:

```
Analyze sales trends from this data:
{{SALES_DATA}}

Time period: {{TIME_PERIOD}}
Forecast: {{FORECAST_DAYS}} days
```

### Complete Pipeline

```bash
# Step 1: Transform CSV to JSON
python workstreams/data-analytics/scripts/csv_to_json.py \
  --input data/sales.csv --output /tmp/sales.json

# Step 2: Clean the data
python shared/scripts/data_cleaner.py \
  --input /tmp/sales.json --output /tmp/sales_clean.json

# Step 3: Use sales-trend-analyzer skill with cleaned data
# (Paste cleaned data into LLM Chat with the skill prompt)
```

### API Endpoint (future portal)

```http
POST {{API_BASE_URL}}/packages/data-analytics-pack/invoke
Content-Type: application/json

{
  "skill_id": "data-analytics-sales-trend-analyzer",
  "inputs": {
    "sales_data": "[{...cleaned data...}]",
    "time_period": "daily",
    "forecast_days": 7
  }
}
```

## Examples

### Example 1 — Widget A sales trend analysis

**Input:**
- `SALES_DATA`:
```json
[
  {"date": "2026-01-01", "product": "Widget A", "quantity": 100, "revenue": 1000},
  {"date": "2026-01-02", "product": "Widget A", "quantity": 120, "revenue": 1200},
  {"date": "2026-01-03", "product": "Widget A", "quantity": 95, "revenue": 950},
  {"date": "2026-01-04", "product": "Widget A", "quantity": 150, "revenue": 1500},
  {"date": "2026-01-05", "product": "Widget A", "quantity": 180, "revenue": 1800},
  {"date": "2026-01-06", "product": "Widget A", "quantity": 200, "revenue": 2000},
  {"date": "2026-01-07", "product": "Widget A", "quantity": 210, "revenue": 2100}
]
```
- `TIME_PERIOD`: `daily`
- `FORECAST_DAYS`: `3`

**Output:**

```markdown
## Data Overview
- Records: 7
- Date Range: 2026-01-01 to 2026-01-07
- Products: Widget A
- Total Revenue: $10,550
- Total Units: 1,055

## Trend Analysis

**Overall Trend**: Upward
- Growth Rate: 23.3% per day (average)
- Trend Strength: Strong (R² = 0.89)

**Key Patterns**:
- Consistent upward trajectory with 110% growth from Day 1 to Day 7
- Acceleration in recent days: Days 5-7 show 30% daily growth vs. 15% in Days 1-4
- One dip on Day 3 (-21% from Day 2) but trend recovered strongly

**Inflection Points**:
- 2026-01-04: Sales rebounded sharply (+58% from Day 3), marking acceleration phase

## Anomalies

| Date | Type | Value | Deviation | Hypothesis |
|---|---|---|---|---|
| 2026-01-03 | Dip | 95 units | -1.2 SD | Weekend effect or temporary stock issue |
| 2026-01-06 | Spike | 200 units | +1.8 SD | Possible promotional campaign or viral effect |

## Forecast (Next 3 Days)

Method: Linear Regression
Confidence: High (strong trend, R² = 0.89)

| Date | Forecasted Units | Forecasted Revenue | Notes |
|---|---|---|---|
| 2026-01-08 | 225 | $2,250 | Based on 7% growth from Day 7 |
| 2026-01-09 | 240 | $2,400 | Assumes continued momentum |
| 2026-01-10 | 255 | $2,550 | High confidence given trend strength |

**Assumptions**:
- Current upward trend continues
- No major external disruptions (stockouts, competitor actions)
- Day 3 dip was anomaly, not start of new pattern
- Pricing remains stable at $10/unit

## Business Recommendations

1. **Increase Inventory Immediately** (Priority: High)
   - **Action**: Order 1,000+ units of Widget A to cover forecasted demand (775+ units over 3 days)
   - **Expected Impact**: Prevent stockouts, capitalize on momentum
   - **Supporting Data**: 110% growth in 7 days, strong R² = 0.89, forecasted 255 units/day by Day 10
   - **Timeline**: Place order today for delivery within 2-3 days

2. **Investigate Day 6 Spike** (Priority: High)
   - **Action**: Analyze what drove the 200-unit spike (promotion, social media, external event)
   - **Expected Impact**: Identify repeatable growth drivers
   - **Supporting Data**: Day 6 sales were 1.8 SD above mean, 33% higher than Day 5
   - **Timeline**: Complete analysis within 24 hours while data is fresh

3. **Prepare Marketing Campaign** (Priority: Medium)
   - **Action**: Launch targeted campaign on Days 8-10 to sustain momentum
   - **Expected Impact**: Extend trend, acquire new customers, increase brand awareness
   - **Supporting Data**: 23.3% daily growth rate provides runway for aggressive marketing
   - **Timeline**: Launch on Day 8

4. **Monitor for Saturation** (Priority: Medium)
   - **Action**: Set up daily monitoring to detect when growth rate slows
   - **Expected Impact**: Early warning of market saturation or competitive pressure
   - **Supporting Data**: Current 23.3% daily growth is unsustainable long-term
   - **Timeline**: Implement immediately

5. **Review Pricing Strategy** (Priority: Low)
   - **Action**: Test small price increase ($10.50 or $11) given strong demand
   - **Expected Impact**: Increase revenue without reducing demand significantly
   - **Supporting Data**: Perfect price elasticity so far (no demand change despite growth)
   - **Timeline**: Test on Day 11-12 after inventory secured
```

## Pipeline Integration

This skill is designed to be called **after** data transformation and cleaning:

```
CSV file
   ↓
csv-to-json-transformer (script)
   ↓
data-cleaner (script)
   ↓
sales-trend-analyzer (skill) ← YOU ARE HERE
   ↓
Output: Trend analysis + forecast + recommendations
```

## Testing Notes

| Model | Tested | Notes |
|---|---|---|
| gpt-4o | ✅ | 2026-04-21. Accurate trend detection and forecasting. |
| claude-3-5-sonnet | ✅ | 2026-04-21. More detailed anomaly analysis than gpt-4o. |

## Changelog

### 1.0.0 — 2026-04-21
- Initial version.
