# Scripts Folder Structure and Usage Guide

## Overview

Scripts are executable programs (Python, Bash, Node.js, etc.) that automate data transformation, validation, aggregation, and processing tasks within AI workflows. They prepare data for LLM consumption, clean outputs, and integrate with external systems.

## Where Scripts Live

### Shared Scripts (`shared/scripts/`)

**Purpose:** Common utilities used by multiple workstreams

**When to place here:**
- The script provides a foundational capability (e.g., data cleaning, JSON validation)
- Multiple workstreams or assets depend on it
- The script has no workstream-specific logic

**Examples:**
- `data-cleaner.md` — Data cleaning and validation utility
- `utilization-data-transformer.md` — Fleet utilization data transformer
- `json-schema-validator.md` — Generic JSON schema validation

### Workstream Scripts (`workstreams/<name>/scripts/`)

**Purpose:** Skill-specific scripts tailored to a workstream's needs

**When to place here:**
- The script implements logic specific to one workstream
- The script is tightly coupled to a skill or workflow in that workstream
- The script may depend on shared scripts but adds specialized functionality

**Examples:**
- `workstreams/data-analytics/scripts/csv-to-json-transformer.md` — CSV to JSON with business logic
- `workstreams/performance-engineers/scripts/benchmark-delta-calculator.md` — Performance benchmark analysis

---

## Folder Structure

```
AI-Skills/
├── shared/
│   └── scripts/
│       ├── data-cleaner.md                    # Common: data validation
│       ├── utilization-data-transformer.md    # Common: capacity data processing
│       └── json-schema-validator.md           # Common: schema validation
│
└── workstreams/
    ├── data-analytics/
    │   └── scripts/
    │       ├── csv-to-json-transformer.md     # Skill-specific: data conversion
    │       └── report-generator.md            # Skill-specific: report formatting
    │
    ├── performance-engineers/
    │   └── scripts/
    │       ├── benchmark-delta-calculator.md  # Skill-specific: benchmark analysis
    │       └── load-test-aggregator.md        # Skill-specific: load test data processing
    │
    └── developers/
        └── scripts/
            ├── code-metrics-extractor.md      # Skill-specific: code analysis
            └── dependency-graph-builder.md    # Skill-specific: dependency visualization
```

---

## Script Documentation Format

Every script must be documented using the `templates/script-template.md` format. Key sections:

### 1. Frontmatter (YAML metadata)

```yaml
---
id: unique-script-id
title: Human-Readable Script Name
type: script
version: "1.0.0"
status: active | draft | deprecated
workstream:
  - data-analytics
  - shared  # if applicable
author: github-username
created: YYYY-MM-DD
updated: YYYY-MM-DD
tags:
  - python
  - transformer
description: |
  One-paragraph description of what the script does and where it fits in the pipeline.
language: python | bash | node | go | ruby | powershell
runtime_version: "3.11+"
script_type: transformer | aggregator | validator | orchestrator | extractor | formatter
entry_point: script_filename.py
security_classification: internal | public
package_dependencies:
  - name: pandas
    version: ">=2.0.0"
    ecosystem: pip
asset_dependencies:
  - other-asset-id-that-calls-this-script
inputs:
  - name: INPUT_FILE
    type: string
    description: Path to input file
    required: true
outputs:
  - name: OUTPUT_FILE
    type: string
    description: Path to output file
security_notes: |
  What files/systems the script accesses, how credentials are managed,
  whether output is safe for LLM consumption.
---
```

### 2. Overview

Describe:
- Purpose of the script
- Where it sits in the broader pipeline
- What problem it solves
- What would happen without this script

### 3. Pipeline Position

Show where the script fits in the data flow:

```
[DATA SOURCE]  →  this script  →  [NEXT ASSET / LLM]
```

### 4. Implementation

Include the complete, working script implementation in a code block with:
- Clear documentation/comments
- Argument parsing (support both CLI flags and env vars)
- Error handling
- Exit codes (0 = success, non-zero = failure)

### 5. Examples

Show realistic examples with:
- Sample input data
- Command invocation
- Expected output

### 6. Calling Asset Reference

Document which assets (skills, workflows, agents) depend on this script.

### 7. Security Notes

- What files does the script read/write?
- Does it make network calls?
- Is the output safe to pass to an LLM?
- How are credentials managed?

---

## Script Types

### Transformer Scripts

**Purpose:** Convert data from one format to another

**Examples:**
- CSV to JSON conversion
- XML to JSON conversion
- API response transformation

**Template pattern:**
```python
def transform(input_data):
    # Transform logic
    return output_data

if __name__ == "__main__":
    input_data = load_input()
    output_data = transform(input_data)
    save_output(output_data)
```

### Aggregator Scripts

**Purpose:** Summarize or roll up large datasets

**Examples:**
- Aggregate logs by error type
- Summarize sales by region
- Calculate daily metrics from hourly data

**Template pattern:**
```python
def aggregate(records):
    summary = {}
    for record in records:
        # Aggregation logic
    return summary
```

### Validator Scripts

**Purpose:** Check data quality, completeness, schema compliance

**Examples:**
- Data cleaner (removes duplicates, handles missing values)
- Schema validator (checks JSON against schema)
- Data quality scorer

**Template pattern:**
```python
def validate(data):
    issues = []
    # Validation logic
    return {
        'valid': len(issues) == 0,
        'issues': issues,
        'score': calculate_quality_score(data)
    }
```

### Orchestrator Scripts

**Purpose:** Chain multiple other scripts or tools

**Examples:**
- Run data pipeline (transform → clean → analyze)
- Multi-stage data processing
- Workflow automation

**Template pattern:**
```bash
#!/bin/bash
# Step 1: Transform
python transform.py --input raw.csv --output structured.json

# Step 2: Clean
python clean.py --input structured.json --output clean.json

# Step 3: Analyze
python analyze.py --input clean.json --output report.md
```

### Extractor Scripts

**Purpose:** Pull specific data from larger datasets

**Examples:**
- Extract metrics from logs
- Pull configuration from infrastructure
- Query database and format results

**Template pattern:**
```python
def extract(source, filters):
    extracted = []
    for item in source:
        if matches_filters(item, filters):
            extracted.append(item)
    return extracted
```

### Formatter Scripts

**Purpose:** Format data for human consumption or LLM ingestion

**Examples:**
- Generate Markdown tables from JSON
- Format code for display
- Create reports from raw data

**Template pattern:**
```python
def format_as_markdown(data):
    markdown = "# Report\n\n"
    markdown += render_table(data)
    return markdown
```

---

## Best Practices

### 1. Script Reusability

**✅ DO:**
- Make scripts configurable via CLI flags and environment variables
- Keep scripts focused on one task
- Document all inputs, outputs, and dependencies
- Make scripts idempotent (safe to re-run)

**❌ DON'T:**
- Hardcode file paths or credentials
- Mix multiple concerns in one script
- Modify source files (always write to new output)
- Use undocumented dependencies

### 2. Input/Output Conventions

**Standard pattern:**
```python
import argparse
import os

parser = argparse.ArgumentParser()
parser.add_argument("--input", default=os.getenv("INPUT_FILE", "data.csv"))
parser.add_argument("--output", default=os.getenv("OUTPUT_FILE", "/tmp/output.json"))
args = parser.parse_args()
```

**Benefits:**
- Works with both CLI and environment variables
- Easy to test locally
- Easy to use in CI/CD pipelines

### 3. Error Handling

**Always include:**
- Input validation (file exists, correct format)
- Meaningful error messages
- Non-zero exit codes on failure
- Logging to stdout/stderr

**Example:**
```python
if not Path(args.input).exists():
    print(f"ERROR: Input file not found: {args.input}", file=sys.stderr)
    sys.exit(1)
```

### 4. Security

**Required checks:**
- Never commit secrets or credentials
- Document what data the script accesses
- State whether output is safe for LLM consumption
- Use environment variables for sensitive data

**Example security note:**
```yaml
security_notes: |
  Reads local files only — no network calls.
  Output contains only numeric metrics (no PII, no secrets).
  Safe to pass to LLM prompts.
  Credentials (if needed) via environment variable API_KEY.
```

### 5. Testing

**Minimum testing:**
- Test with valid sample data
- Test with invalid/edge-case data
- Document tested runtimes (Python 3.11, 3.12, etc.)
- Include example invocations

**Testing notes section:**
```markdown
## Testing Notes

| Runtime | Tested | Notes |
|---|---|---|
| Python 3.11 | ✅ | 2026-04-21. All features work correctly. |
| Python 3.12 | ✅ | 2026-04-21. No compatibility issues. |
```

---

## Chaining Scripts

Scripts are designed to be chained together in pipelines:

### Pattern 1: Sequential Bash Script

```bash
#!/bin/bash
set -e  # Exit on any error

# Step 1: Transform
python workstreams/data-analytics/scripts/csv_to_json.py \
  --input data/sales.csv \
  --output /tmp/sales.json

# Step 2: Clean
python shared/scripts/data_cleaner.py \
  --input /tmp/sales.json \
  --output /tmp/sales_clean.json \
  --remove-duplicates true

# Step 3: Use cleaned data in skill/prompt
echo "Data ready for analysis: /tmp/sales_clean.json"
```

### Pattern 2: GitHub Actions Workflow

```yaml
jobs:
  data-pipeline:
    runs-on: ubuntu-latest
    steps:
      - name: Install dependencies
        run: |
          pip install pandas>=2.0.0 numpy>=1.24.0
      
      - name: Transform CSV to JSON
        env:
          INPUT_CSV: data/sales.csv
          OUTPUT_JSON: /tmp/sales.json
        run: python workstreams/data-analytics/scripts/csv_to_json.py
      
      - name: Clean data
        env:
          INPUT_FILE: /tmp/sales.json
          OUTPUT_FILE: /tmp/sales_clean.json
          REMOVE_DUPLICATES: true
        run: python shared/scripts/data_cleaner.py
      
      - name: Read cleaned data for LLM
        id: data
        run: |
          DATA=$(cat /tmp/sales_clean.json)
          echo "cleaned_data=${DATA}" >> "$GITHUB_OUTPUT"
```

### Pattern 3: Python Orchestrator

```python
#!/usr/bin/env python3
"""Orchestrate complete data pipeline."""
import subprocess
import sys

def run_script(script, env=None):
    result = subprocess.run(
        ["python", script],
        env=env,
        capture_output=True,
        text=True
    )
    if result.returncode != 0:
        print(f"ERROR: {script} failed:", file=sys.stderr)
        print(result.stderr, file=sys.stderr)
        sys.exit(1)
    return result.stdout

# Step 1
run_script("workstreams/data-analytics/scripts/csv_to_json.py", {
    "INPUT_CSV": "data/sales.csv",
    "OUTPUT_JSON": "/tmp/sales.json"
})

# Step 2
run_script("shared/scripts/data_cleaner.py", {
    "INPUT_FILE": "/tmp/sales.json",
    "OUTPUT_FILE": "/tmp/sales_clean.json"
})

print("Pipeline complete!")
```

---

## Integration with Other Assets

### Scripts → Prompts

Scripts prepare data for LLM prompts:

```bash
# Generate clean data
python shared/scripts/data_cleaner.py \
  --input raw.json --output clean.json

# Use in prompt
DATA=$(cat clean.json)

# Copy prompt from prompts/statistical-analysis-prompt.md
# Replace {{DATA}} placeholder with $DATA
# Paste into GitHub Copilot Chat
```

### Scripts → Skills

Skills consume script outputs:

```markdown
# In skills/sales-trend-analyzer.md
...
## Prerequisites
- Run csv-to-json-transformer.md first
- Run data-cleaner.md second
- Use cleaned output as input to this skill
...
```

### Scripts → Agents

Agents orchestrate scripts automatically:

```markdown
# In agents/data-insights-agent.md
...
Tools available:
- csv_to_json: call with (input, output) → returns JSON path
- data_cleaner: call with (input, output) → returns cleaned data path
...
```

### Scripts → Workflows

Workflows document complete pipelines:

```markdown
# In workflows/complete-data-analysis-workflow.md
...
Step 1: Run csv-to-json-transformer.md
Step 2: Run data-cleaner.md
Step 3: Use statistical-analysis-prompt.md
Step 4: Use sales-trend-analyzer.md skill
...
```

---

## Common vs Skill-Specific Decision Tree

```
Is the script useful to multiple workstreams?
│
├─ YES ──▶ Place in shared/scripts/
│           Examples: data-cleaner, json-validator
│
└─ NO ───▶ Does it implement workstream-specific logic?
           │
           ├─ YES ──▶ Place in workstreams/<name>/scripts/
           │           Examples: csv-to-json-transformer (data-analytics)
           │
           └─ NO ────▶ Re-evaluate: Could it be generalized?
                       ├─ YES ──▶ Generalize and place in shared/scripts/
                       └─ NO ───▶ Keep in workstreams/<name>/scripts/
```

---

## Examples

See the complete example in:
- **Workstream:** `workstreams/data-analytics/`
- **Common script:** `shared/scripts/data-cleaner.md`
- **Skill-specific script:** `workstreams/data-analytics/scripts/csv-to-json-transformer.md`
- **Complete workflow:** `workstreams/data-analytics/workflows/complete-data-analysis-workflow.md`

---

## Quick Start Checklist

### Adding a New Common Script

- [ ] Create script in `shared/scripts/`
- [ ] Use `templates/script-template.md` as starting point
- [ ] Set `workstream: [shared]` in frontmatter
- [ ] Document which workstreams/assets use it
- [ ] Add to `asset_dependencies` of calling assets
- [ ] Test with sample data
- [ ] Document security considerations

### Adding a New Skill-Specific Script

- [ ] Create script in `workstreams/<name>/scripts/`
- [ ] Use `templates/script-template.md` as starting point
- [ ] Set `workstream: [<name>]` in frontmatter
- [ ] Document integration with skills/workflows in that workstream
- [ ] Note dependencies on shared scripts (if any)
- [ ] Test with sample data
- [ ] Document security considerations

---

## See Also

- [Complete Data Analysis Workflow](../workstreams/data-analytics/workflows/complete-data-analysis-workflow.md) — Reference implementation
- [Script Template](../templates/script-template.md) — Template for new scripts
- [Standards Guide](../docs/standards.md) — Naming and quality standards
