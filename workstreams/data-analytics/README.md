# Data Analytics Work Stream

This folder contains AI skills, prompts, agents, workflows, scripts, and automations designed for **data analysts, data scientists, and business intelligence professionals**.

## Use Cases

- Data cleaning and transformation
- Statistical analysis and insights generation
- Data visualization recommendations
- Automated reporting
- Pattern detection and anomaly identification
- Data quality assessment
- CSV/JSON/Excel data processing
- Trend analysis and forecasting
- Data pipeline automation

## Contents

| Folder | What Belongs Here |
|---|---|
| `prompts/` | Prompts for data analysis tasks |
| `skills/` | Focused capabilities (e.g., "analyze sales trends") |
| `agents/` | Autonomous agents (e.g., data quality analyzer) |
| `workflows/` | Multi-step processes (e.g., complete data analysis pipeline) |
| `scripts/` | Data transformation and processing scripts |
| `automations/` | Scripts for data pipeline integration |
| `ideas/` | Proposed but not-yet-implemented capabilities |

## Example Use Case: Complete Data Analysis Pipeline

This workstream includes a complete example demonstrating how multiple assets work together:

1. **Scripts** (`scripts/`) - Transform raw data into analysis-ready format
   - `csv-to-json-transformer.md` - Convert CSV files to structured JSON
   - Uses shared `data-cleaner.md` for data validation

2. **Prompt** (`prompts/statistical-analysis-prompt.md`) - Generate statistical insights

3. **Skill** (`skills/sales-trend-analyzer.md`) - Analyze sales patterns and trends

4. **Agent** (`agents/data-insights-agent.md`) - Autonomous data analysis with recommendations

5. **Workflow** (`workflows/complete-data-analysis-workflow.md`) - Ties everything together

See `workflows/complete-data-analysis-workflow.md` for the full end-to-end example.

## Scripts Folder Organization

The `scripts/` folder contains data transformation and processing scripts that:
- Prepare raw data for LLM consumption
- Transform between different data formats
- Clean and validate data
- Aggregate and summarize large datasets

Scripts can be:
- **Common scripts** - Reusable utilities (can be in `shared/scripts/`)
- **Skill-specific scripts** - Tailored to specific skills in this workstream

## Contributing

Follow the [CONTRIBUTING guide](../../CONTRIBUTING.md) and place new files in the appropriate sub-folder. Use the templates in `../../templates/` as your starting point.
