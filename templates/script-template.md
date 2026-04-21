---
id: replace-with-unique-kebab-case-id
title: Replace With Human-Readable Title
type: script
version: "1.0.0"
status: draft
workstream:
  - replace-with-workstream
author: your-github-username
created: YYYY-MM-DD
updated: YYYY-MM-DD
tags:
  - tag1
  - tag2
description: |
  One or two sentences describing what this script does, what data it transforms,
  and which asset(s) in the pipeline invoke it.
language: python          # python | bash | node | go | ruby | powershell | other
runtime_version: "3.11+"
script_type: transformer  # transformer | aggregator | validator | orchestrator | extractor | formatter
entry_point: transform.py # file name or function that callers invoke
security_classification: internal
package_dependencies:
  - name: "requests"
    version: ">=2.31.0"
    ecosystem: pip
  # Add other pip / npm / gem dependencies here
asset_dependencies:
  - replace-with-asset-id-that-calls-this-script
  # List the IDs of automations, workflows, or agents that invoke this script
inputs:
  - name: INPUT_FILE
    type: string
    description: Path to the input JSON / CSV file to transform
    required: true
outputs:
  - name: OUTPUT_FILE
    type: string
    description: Path to the transformed output file
security_notes: |
  Describe what files or external systems the script reads/writes and how credentials
  (if any) are managed. Note whether output is safe to pass to an LLM (i.e., free of
  secrets or PII).
---

## Overview

<!-- Describe the script's purpose in the broader AI pipeline.
     Which automation, workflow, or agent invokes it? What problem does it solve?
     What would happen without this script (raw data handed to LLM, etc.)? -->

## Pipeline Position

```
[DATA SOURCE]  →  this script ({{id}})  →  [LLM PROMPT / ASSET]
```

<!-- Replace the diagram above with the actual asset chain. Example:
     data/utilization.json  →  shared-utilization-data-transformer  →  cap-capacity-forecast-prompt
-->

## Prerequisites

- Python 3.11+ (or the relevant runtime)
- Install dependencies: `pip install -r requirements.txt`
- Input file at the expected path or provide it via the `INPUT_FILE` environment variable

## Installation

```bash
pip install requests>=2.31.0
# Add other dependencies here
```

Or create a `requirements.txt`:

```
requests>=2.31.0
```

## Implementation

```python
#!/usr/bin/env python3
"""
script: replace-with-unique-kebab-case-id
description: Replace With Human-Readable Title
version: 1.0.0

Usage:
    python transform.py --input <input_file> --output <output_file>

    Or via environment variables:
    INPUT_FILE=data/input.json OUTPUT_FILE=/tmp/output.json python transform.py
"""

import argparse
import json
import os
import sys
from pathlib import Path


def transform(input_path: str, output_path: str) -> dict:
    """
    Core transformation logic.
    Replace this function body with your actual aggregation / transformation.
    """
    with open(input_path) as f:
        data = json.load(f)

    # TODO: implement transformation
    result = {
        "transformed": True,
        "record_count": len(data) if isinstance(data, list) else 1,
        "summary": "Replace with actual output structure",
    }

    with open(output_path, "w") as f:
        json.dump(result, f, indent=2)

    return result


def main():
    parser = argparse.ArgumentParser(description="Replace With Human-Readable Title")
    parser.add_argument("--input",  default=os.getenv("INPUT_FILE",  "data/input.json"),  help="Input file path")
    parser.add_argument("--output", default=os.getenv("OUTPUT_FILE", "/tmp/output.json"), help="Output file path")
    args = parser.parse_args()

    if not Path(args.input).exists():
        print(f"ERROR: Input file not found: {args.input}", file=sys.stderr)
        sys.exit(1)

    result = transform(args.input, args.output)
    print(json.dumps(result, indent=2))


if __name__ == "__main__":
    main()
```

> **Note:** This is a reference implementation. Replace the `transform()` body with your actual aggregation or transformation logic. Keep the `argparse` / env-var interface so automations and GitHub Actions can invoke the script consistently.

## Inputs

| Name | Type | Required | Description |
|---|---|---|---|
| `INPUT_FILE` | string | Yes | Path to the input file (env var or `--input` flag) |

## Outputs

| Name | Type | Description |
|---|---|---|
| `OUTPUT_FILE` | string | Path to the transformed output file (env var or `--output` flag) |

## Invocation from an Automation

```yaml
# Excerpt from a GitHub Actions workflow that calls this script
- name: Install script dependencies
  run: pip install requests>=2.31.0

- name: Run transformer
  env:
    INPUT_FILE: data/input.json
    OUTPUT_FILE: /tmp/output.json
  run: python workstreams/<workstream>/scripts/transform.py
```

## Calling Asset Reference

This script is designed to be called by the following assets:

| Asset ID | Type | Role |
|---|---|---|
| `replace-with-asset-id` | automation | Invokes this script as a pre-processing step before the LLM call |

## Examples

### Example 1

**Input (`data/input.json`):**
```json
{
  "example": "input data"
}
```

**Output (`/tmp/output.json`):**
```json
{
  "transformed": true,
  "record_count": 1,
  "summary": "Replace with actual output"
}
```

## Security Notes

<!-- Required:
     - What files does this script read and write?
     - Does it make any network calls?
     - Is the output safe to pass to an LLM (no PII, no secrets)?
     - Is the script idempotent (safe to re-run)? -->

## Testing Notes

| Runtime | Tested | Notes |
|---|---|---|
| Python 3.11 | ✅ | Tested on YYYY-MM-DD |
| Python 3.12 | ⬜ | Not yet tested |

## Changelog

### 1.0.0 — YYYY-MM-DD
- Initial version.
