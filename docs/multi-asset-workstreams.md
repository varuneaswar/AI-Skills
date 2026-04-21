# Developing Multi-Asset Workstreams

## Overview

A **multi-asset workstream** is a complete, production-ready capability that combines multiple types of AI assets—scripts, prompts, skills, agents, and workflows—to solve complex business problems end-to-end.

This guide shows you how to structure, develop, and deploy multi-asset workstreams in the AI-Skills repository, using the **data-analytics** workstream as a reference implementation.

---

## Table of Contents

1. [What is a Multi-Asset Workstream?](#what-is-a-multi-asset-workstream)
2. [Asset Types and Their Roles](#asset-types-and-their-roles)
3. [Workstream Structure](#workstream-structure)
4. [Development Process](#development-process)
5. [Asset Composition Patterns](#asset-composition-patterns)
6. [Integration Examples](#integration-examples)
7. [Best Practices](#best-practices)
8. [Complete Reference: Data Analytics Workstream](#complete-reference-data-analytics-workstream)

---

## What is a Multi-Asset Workstream?

A multi-asset workstream is a collection of complementary AI assets that work together to deliver a complete business capability.

### Single Asset vs Multi-Asset

**Single Asset (Simple):**
```
User question ──▶ Prompt ──▶ LLM ──▶ Answer
```

**Multi-Asset (Complex):**
```
Raw data ──▶ Script ──▶ Clean data ──▶ Prompt ──▶ LLM ──▶ Insights
                                           │
                                           ▼
                              Skill ──▶ Deep analysis ──▶ Recommendations
                                           │
                                           ▼
                              Agent ──▶ Orchestrate all ──▶ Report
```

### When to Build a Multi-Asset Workstream

**Use multi-asset when:**
- The task requires data preparation before LLM processing
- You need to chain multiple specialized capabilities
- The solution benefits from automation (agents)
- You want to provide both manual and automated paths

**Use single asset when:**
- The task is straightforward (simple prompt + LLM)
- No data transformation is needed
- The capability is self-contained

---

## Asset Types and Their Roles

### 1. Scripts

**Role:** Data transformation, validation, aggregation

**When to use:**
- Convert data formats (CSV → JSON)
- Clean messy data (remove duplicates, handle missing values)
- Transform data for LLM consumption (aggregate, summarize)
- Integrate with external systems

**Location:**
- `shared/scripts/` — Common utilities
- `workstreams/<name>/scripts/` — Skill-specific transformations

**Example:** `data-cleaner.md`, `csv-to-json-transformer.md`

---

### 2. Prompts

**Role:** Reusable LLM instructions for specific tasks

**When to use:**
- You need a template for a common task
- Multiple skills or workflows use the same prompt pattern
- You want to standardize LLM instructions

**Location:** `workstreams/<name>/prompts/`

**Example:** `statistical-analysis-prompt.md`

---

### 3. Skills

**Role:** Focused, single-purpose AI capabilities

**When to use:**
- You need a specialized capability (e.g., "analyze sales trends")
- The capability builds on prompts but adds structure
- Multiple workflows or agents will invoke this capability

**Location:** `workstreams/<name>/skills/`

**Example:** `sales-trend-analyzer.md`

---

### 4. Agents

**Role:** Autonomous orchestration of multiple tools and skills

**When to use:**
- You want to automate a multi-step process
- The process has decision points (conditional logic)
- You need to chain multiple scripts, prompts, and skills
- You want to reduce manual effort

**Location:** `workstreams/<name>/agents/`

**Example:** `data-insights-agent.md`

---

### 5. Workflows

**Role:** Documentation of complete end-to-end processes

**When to use:**
- You want to document how multiple assets work together
- You need to provide both manual and automated execution paths
- You want to show the big picture

**Location:** `workstreams/<name>/workflows/`

**Example:** `complete-data-analysis-workflow.md`

---

## Workstream Structure

Every workstream follows a standard folder structure:

```
workstreams/<workstream-name>/
├── README.md                    # Overview, use cases, contents
├── scripts/                     # Data transformation scripts
│   ├── script1.md
│   └── script2.md
├── prompts/                     # Reusable prompt templates
│   ├── prompt1.md
│   └── prompt2.md
├── skills/                      # Focused AI capabilities
│   ├── skill1.md
│   └── skill2.md
├── agents/                      # Autonomous agents
│   ├── agent1.md
│   └── agent2.md
├── workflows/                   # End-to-end process documentation
│   ├── workflow1.md
│   └── workflow2.md
├── automations/                 # CI/CD and integration scripts
│   └── automation1.md
└── ideas/                       # Future capabilities (not yet implemented)
    └── idea1.md
```

---

## Development Process

### Phase 1: Plan the Workstream

**Steps:**
1. Identify the business problem to solve
2. Define the end-to-end process
3. Break down into steps (data prep → analysis → insights)
4. Identify which assets are needed for each step
5. Document dependencies between assets

**Example: Data Analytics Workstream**

**Problem:** Business analysts need to transform raw sales data into actionable insights

**Process:**
1. Load CSV data → Script (csv-to-json-transformer)
2. Clean data → Script (data-cleaner)
3. Generate statistics → Prompt (statistical-analysis-prompt)
4. Analyze trends → Skill (sales-trend-analyzer)
5. Automate all → Agent (data-insights-agent)
6. Document everything → Workflow (complete-data-analysis-workflow)

---

### Phase 2: Build Assets (Bottom-Up)

**Order:**
1. **Scripts first** — Build data transformation/cleaning utilities
2. **Prompts second** — Create templates for LLM tasks
3. **Skills third** — Build specialized capabilities using prompts
4. **Agents fourth** — Orchestrate scripts, prompts, and skills
5. **Workflows last** — Document the complete process

**Why bottom-up?**
- Each layer builds on the previous
- You can test components independently
- You can provide both manual and automated paths

---

### Phase 3: Document Integration

**For each asset, document:**
- What it depends on (inputs, prerequisites)
- What depends on it (calling assets)
- Where it fits in the pipeline
- How to invoke it (manual + automated)

**Use the `asset_dependencies` field:**

```yaml
# In csv-to-json-transformer.md
asset_dependencies:
  - shared-data-cleaner          # This script is often chained with data-cleaner
  - sales-trend-analyzer         # This skill uses the transformed data
```

---

### Phase 4: Test End-to-End

**Test scenarios:**
1. **Manual path:** Run each script/prompt/skill manually
2. **Semi-automated:** Chain scripts, manually run prompts/skills
3. **Fully automated:** Use agent to orchestrate everything

**Document test results:**
- Working configurations
- Edge cases handled
- Known limitations

---

### Phase 5: Register the Workstream

Update `workstreams/registry.json`:

```json
{
  "workstreams": [
    {
      "id": "data-analytics",
      "name": "Data Analytics",
      "description": "Data analysis, insights generation, and business intelligence",
      "status": "active",
      "maintainer": "ai-skills-maintainer",
      "created": "2026-04-21",
      "asset_count": {
        "scripts": 1,
        "prompts": 1,
        "skills": 1,
        "agents": 1,
        "workflows": 1
      }
    }
  ]
}
```

---

## Asset Composition Patterns

### Pattern 1: Linear Pipeline

**Structure:**
```
Script 1 → Script 2 → Prompt → Skill → Output
```

**Example:** Data cleaning pipeline
```
CSV file → csv-to-json → data-cleaner → statistical-analysis → insights
```

**Use when:**
- Each step depends on the previous
- No conditional logic needed
- Process is deterministic

---

### Pattern 2: Parallel Processing

**Structure:**
```
Input → [Script 1, Script 2, Script 3] → Merge → Prompt → Output
```

**Example:** Multi-source data analysis
```
[CSV data, API data, DB data] → Transform all → Merge → Analyze → Report
```

**Use when:**
- Multiple independent data sources
- Parallel processing speeds up the pipeline
- All sources must be available before analysis

---

### Pattern 3: Conditional Branching

**Structure:**
```
Input → Script → Check condition
                     ├─ Path A → Skill A → Output A
                     └─ Path B → Skill B → Output B
```

**Example:** Analysis routing
```
Data → Clean → Check type
                ├─ Sales data → sales-trend-analyzer → Sales report
                └─ User data → user-behavior-analyzer → Behavior report
```

**Use when:**
- Different data types require different analysis
- Process has decision points
- Agent can make routing decisions

---

### Pattern 4: Agent Orchestration

**Structure:**
```
Agent decides:
  Step 1: Run script(s)
  Step 2: Check quality → If low, re-clean
  Step 3: Run prompt(s)
  Step 4: Run skill(s)
  Step 5: Synthesize report
```

**Example:** Data insights agent
```
Agent:
  1. Detect format (CSV/JSON) → Transform if needed
  2. Clean data → Check quality score
  3. Run statistical analysis
  4. If analysis_type=sales → Run sales-trend-analyzer
  5. Generate comprehensive report
```

**Use when:**
- Multi-step process with decision points
- You want full automation
- Manual execution is too tedious

---

## Integration Examples

### Example 1: Script → Prompt

**Scenario:** Clean data before passing to LLM

**Script:** `data-cleaner.md`
```bash
python shared/scripts/data_cleaner.py \
  --input raw.json \
  --output clean.json
```

**Prompt:** `statistical-analysis-prompt.md`
```
Analyze this data:
{{DATA}}  ← Insert contents of clean.json here
```

**Integration point:** Human manually copies cleaned data into prompt

---

### Example 2: Script → Script → Skill

**Scenario:** Transform, clean, then analyze

**Pipeline:**
```bash
# Step 1: Transform
python workstreams/data-analytics/scripts/csv_to_json.py \
  --input data.csv --output /tmp/structured.json

# Step 2: Clean
python shared/scripts/data_cleaner.py \
  --input /tmp/structured.json --output /tmp/clean.json

# Step 3: Use skill with cleaned data
# Copy skill definition from skills/sales-trend-analyzer.md
# Paste clean.json contents into {{SALES_DATA}} placeholder
# Run in GitHub Copilot Chat
```

**Integration point:** Bash script chains steps; human runs skill

---

### Example 3: Agent Orchestrates All

**Scenario:** Fully automated end-to-end

**Agent invocation:**
```
@workspace You are a data insights agent.

Data source: data/sales.csv
Analysis type: sales-trends
Business context: E-commerce Q1 2026
```

**What the agent does:**
1. Detects CSV format
2. Calls `csv-to-json-transformer` script
3. Calls `data-cleaner` script
4. Calls `statistical-analysis-prompt`
5. Calls `sales-trend-analyzer` skill
6. Synthesizes comprehensive report

**Integration point:** Agent has access to all tools and orchestrates automatically

---

## Best Practices

### 1. Design for Composability

**✅ DO:**
- Make each asset independent and testable
- Use clear input/output contracts
- Document dependencies explicitly
- Support both manual and automated invocation

**❌ DON'T:**
- Tightly couple assets (hard to test/reuse)
- Mix concerns (one script does too much)
- Skip documentation of integration points

---

### 2. Provide Multiple Execution Paths

**Levels of automation:**

**Level 1: Manual**
```bash
# User runs each script manually
python script1.py
python script2.py
# User copies outputs between steps
```

**Level 2: Semi-Automated**
```bash
# Bash script chains data prep
./run_pipeline.sh
# User manually runs prompts/skills with cleaned data
```

**Level 3: Fully Automated**
```
# Agent orchestrates everything
@workspace Run complete analysis on data/sales.csv
```

**Why provide all three?**
- Manual: Learning, debugging, customization
- Semi-automated: Production data prep + human analysis
- Fully automated: Routine tasks, scale

---

### 3. Document the Big Picture

**Every workstream needs:**
- **README.md** — Overview, use cases, folder contents
- **Workflow document** — End-to-end process with all assets
- **Asset reference table** — What each asset does and where it lives

**Example:** `workstreams/data-analytics/workflows/complete-data-analysis-workflow.md`

---

### 4. Test Incrementally

**Test each asset independently:**
- Scripts: Unit tests with sample data
- Prompts: Test with LLM using realistic inputs
- Skills: Test with clean sample data
- Agents: Test with full pipeline

**Then test integration:**
- Chain scripts: Do outputs match expected inputs?
- Chain scripts → prompts: Is data formatted correctly?
- Agent orchestration: Does agent call all tools correctly?

---

### 5. Version and Maintain

**Versioning:**
- Each asset has a `version` field in frontmatter
- Start at `1.0.0`
- Increment when making breaking changes

**Maintenance:**
- Keep `updated` field current
- Document changes in `Changelog` section
- Update dependencies when other assets change

---

## Complete Reference: Data Analytics Workstream

The **data-analytics** workstream is a complete, working example of a multi-asset workstream.

### Assets Included

| Asset | Type | File | Purpose |
|---|---|---|---|
| Data Cleaner | Script (common) | `shared/scripts/data-cleaner.md` | Clean and validate data |
| CSV to JSON | Script (skill-specific) | `workstreams/data-analytics/scripts/csv-to-json-transformer.md` | Convert CSV to structured JSON |
| Statistical Analysis | Prompt | `workstreams/data-analytics/prompts/statistical-analysis-prompt.md` | Generate descriptive statistics |
| Sales Trend Analyzer | Skill | `workstreams/data-analytics/skills/sales-trend-analyzer.md` | Analyze sales trends and forecast |
| Data Insights Agent | Agent | `workstreams/data-analytics/agents/data-insights-agent.md` | Orchestrate complete pipeline |
| Complete Analysis | Workflow | `workstreams/data-analytics/workflows/complete-data-analysis-workflow.md` | Document end-to-end process |

### How They Work Together

```
Raw CSV file (sales_data.csv)
    │
    ▼
csv-to-json-transformer (script)
    │  Converts CSV to structured JSON
    │  Infers types, handles nested objects
    ▼
/tmp/structured.json
    │
    ▼
data-cleaner (script - shared utility)
    │  Removes duplicates
    │  Handles missing values
    │  Detects outliers
    │  Generates quality report
    ▼
/tmp/cleaned.json + quality_report.json
    │
    ├───▶ statistical-analysis-prompt
    │      Generates descriptive statistics
    │      Identifies patterns
    │      Extracts insights
    │
    └───▶ sales-trend-analyzer (skill)
           Analyzes trends
           Generates 7-day forecast
           Provides recommendations
    │
    ▼
Comprehensive Report (Markdown)
    
OR: Use data-insights-agent to automate all steps
```

### Usage Examples

**Manual execution:**
```bash
python workstreams/data-analytics/scripts/csv_to_json.py \
  --input data/sales.csv --output /tmp/sales.json

python shared/scripts/data_cleaner.py \
  --input /tmp/sales.json --output /tmp/sales_clean.json

# Then use prompts/skills manually with clean data
```

**Fully automated:**
```
@workspace You are a data insights agent.
Analyze: data/sales.csv
Type: sales-trends
Context: Q1 e-commerce
```

### Key Lessons

1. **Scripts handle data prep** — CSV transformation and cleaning happen before LLM
2. **Prompts provide structure** — Statistical analysis follows a template
3. **Skills add specialization** — Sales trend analysis is a focused capability
4. **Agents orchestrate** — Data insights agent automates the entire pipeline
5. **Workflows document** — Complete-data-analysis-workflow shows the big picture

### Files to Review

- `workstreams/data-analytics/README.md` — Workstream overview
- `workstreams/data-analytics/workflows/complete-data-analysis-workflow.md` — Complete documentation
- All asset files in `workstreams/data-analytics/`

---

## Quick Start: Creating Your Own Multi-Asset Workstream

### Checklist

- [ ] 1. Define the business problem and end-to-end process
- [ ] 2. Create workstream folder structure (scripts, prompts, skills, agents, workflows)
- [ ] 3. Build scripts for data transformation/cleaning
- [ ] 4. Create prompts for LLM tasks
- [ ] 5. Build skills on top of prompts
- [ ] 6. Create agent to orchestrate (optional)
- [ ] 7. Document complete workflow
- [ ] 8. Test manual, semi-automated, and fully automated paths
- [ ] 9. Update `workstreams/registry.json`
- [ ] 10. Document in workstream README.md

---

## See Also

- [Scripts Guide](./scripts-guide.md) — How to structure and document scripts
- [Complete Data Analysis Workflow](../workstreams/data-analytics/workflows/complete-data-analysis-workflow.md) — Reference implementation
- [Standards Guide](./standards.md) — Naming and quality standards
- [Adding Workstreams](./adding-workstreams.md) — How to register a new workstream
