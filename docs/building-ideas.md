# Building Ideas — Step-by-Step Guide

This guide walks you through the complete journey from a raw idea to a finished, packaged AI asset — covering every combination of asset type (Prompt, Skill, Agent, Workflow, Automation) and every stage of the process.

Whether you are a first-time contributor or a maintainer converting approved ideas, this is your single reference.

---

## Table of Contents

- [Overview — The Idea-to-Asset Journey](#overview)
- [Stage 0 — Capture Your Idea](#stage-0--capture-your-idea)
- [Stage 1 — Choose Your Asset Type](#stage-1--choose-your-asset-type)
- [Stage 2A — Building a Prompt](#stage-2a--building-a-prompt)
- [Stage 2B — Building a Skill](#stage-2b--building-a-skill)
- [Stage 2C — Building an Agent](#stage-2c--building-an-agent)
- [Stage 2D — Building a Workflow](#stage-2d--building-a-workflow)
- [Stage 2E — Building an Automation](#stage-2e--building-an-automation)
- [Stage 3 — Test Your Asset](#stage-3--test-your-asset)
- [Stage 4 — Register and Submit](#stage-4--register-and-submit)
- [Stage 5 — Review and Merge](#stage-5--review-and-merge)
- [Combinations — Idea to Multiple Asset Types](#combinations)
- [Quick Reference — File Locations](#quick-reference)

---

## Overview

The full journey from idea to shipped asset:

```
Stage 0: Capture idea
    │  (Jira ticket or ideas/ Markdown file)
    ▼
Stage 1: Choose asset type
    │  (Prompt? Skill? Agent? Workflow? Automation?)
    ▼
Stage 2: Build the asset
    │  (copy template → fill in → iterate with LLM)
    ▼
Stage 3: Test it
    │  (at least 2 models, at least 1 real example)
    ▼
Stage 4: Register and submit
    │  (update catalog/index.json → open PR)
    ▼
Stage 5: Review and merge
       (maintainer reviews → status: active)
```

---

## Stage 0 — Capture Your Idea

Before writing any content, record the idea so it can be vetted.

### Option A — Submit via Jira ticket (non-technical / Lane 2)

1. Go to the repository on GitHub.
2. Click **Issues** → **New Issue**.
3. Choose the **Idea Submission** template.
4. Fill in the template:
   - **Problem statement** — what problem does this solve?
   - **Proposed solution** — what type of asset would this become?
   - **Work stream** — which team or workstream does this serve?
   - **Acceptance criteria** — how will we know it's done?
5. Submit the issue.
6. A maintainer will add the label `idea:under-review`. When approved (`idea:approved`), the maintainer converts it to a Markdown asset via PR.

**You are done at this point if you are not technical.** Maintainers handle the rest.

### Option B — Create an Idea Markdown file (technical / Lane 1)

1. Copy the idea template:
   ```bash
   cp templates/idea-template.md workstreams/<your-workstream>/ideas/<your-idea-name>.md
   ```
2. Fill in the YAML front-matter (see `templates/idea-template.md`).
3. Complete the body sections: Overview, Proposed Approach, Expected Benefits, Acceptance Criteria, Open Questions, References.
4. Open a PR — the idea will be reviewed and labelled.

**When to use an idea file vs. jump straight to the asset:** Use an idea file if you are not yet sure of the design (inputs/outputs not decided, asset type unclear). Skip the idea file and go straight to Stage 2 if the design is clear.

---

## Stage 1 — Choose Your Asset Type

Use this table to pick the right type:

| Your situation | Asset type | Template |
|---|---|---|
| The idea is not yet fully designed | **Idea** | `templates/idea-template.md` |
| You have a single, reusable instruction for one LLM call | **Prompt** | `templates/prompt-template.md` |
| You have a callable capability with defined inputs and outputs | **Skill** | `templates/skill-template.md` |
| The capability requires multiple tools / skills chained dynamically | **Agent** | `templates/agent-template.md` |
| The capability is a fixed sequence of 2+ steps | **Workflow** | `templates/workflow-template.md` |
| The capability should run automatically (trigger or schedule) | **Automation** | `templates/automation-template.md` |

**Decision examples:**

- "I want to generate a commit message from a git diff" → **Prompt** (single LLM call, parameterised)
- "I want to review code for bugs and security" → **Skill** (defined inputs/outputs, reusable)
- "I want to summarise + review + assess coverage for a PR automatically" → **Workflow** (fixed 3-step sequence)
- "I want an agent that monitors PRs and reviews them automatically" → **Automation** (triggered by event)
- "I want to build something that plans its own steps across multiple skills" → **Agent** (dynamic planning)

---

## Stage 2A — Building a Prompt

A prompt is the simplest asset type — a parameterised instruction for a single LLM call.

### Step 1 — Copy the template

```bash
cp templates/prompt-template.md workstreams/<workstream>/prompts/<your-prompt-name>.md
```

Example: `workstreams/developers/prompts/commit-message-generator.md`

### Step 2 — Fill in the YAML front-matter

Open your file and fill in every field in the `---` block:

```yaml
---
id: dev-commit-message-generator          # kebab-case, globally unique
title: Conventional Commit Message Generator
type: prompt
version: "1.0.0"
status: draft
workstream:
  - developers
author: your-username
created: 2026-04-20
updated: 2026-04-20
tags:
  - git
  - commit-message
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
description: |
  Generates a Conventional Commits message from a git diff or plain-language description.
security_classification: internal
delivery_modes:
  - llm-chat
inputs:
  - name: CHANGE_DESCRIPTION
    type: string
    description: Git diff or description of the change
    required: true
---
```

### Step 3 — Write the prompt

In the `## Prompt` section, write your instruction using `{{PLACEHOLDER}}` for every variable part:

```
[System]
You are a git commit message specialist who follows the Conventional Commits specification.

[Task]
Generate a commit message for the following change:
{{CHANGE_DESCRIPTION}}

[Output Format]
<type>(<scope>): <subject — ≤72 chars>

<body — explain WHY, not WHAT>
```

### Step 4 — Iterate with an LLM

1. Open LLM Chat, ChatGPT, or Claude.
2. Paste your prompt, replacing placeholders with real values.
3. Check the output. Adjust the prompt until the output is consistently good.
4. Save a real example as your `## Examples` section.

### Step 5 — Complete the rest of the file

- `## Overview` — expand on the description
- `## Inputs` table — match the front-matter `inputs[]`
- `## Usage` — show how to use it in Copilot Chat and standalone
- `## Examples` — at least one real input/output pair
- `## Testing Notes` — which models did you test?

**Reference:** See `workstreams/developers/prompts/commit-message-generator.md` for a complete example.

---

## Stage 2B — Building a Skill

A skill is a callable capability with defined inputs, outputs, and a complete system prompt.

### Step 1 — Copy the template

```bash
cp templates/skill-template.md workstreams/<workstream>/skills/<your-skill-name>.md
```

### Step 2 — Define inputs and outputs first

Before writing the prompt, decide on the contract:

```yaml
inputs:
  - name: CODE
    type: string
    description: The code to review
    required: true
  - name: LANGUAGE
    type: string
    description: Programming language
    required: true
outputs:
  - name: REVIEW_REPORT
    type: string
    description: Structured findings by severity
delivery_modes:
  - llm-chat
```

**Rule:** If you cannot define clear inputs and outputs, it is probably a **Prompt** or an **Idea**, not a Skill.

### Step 3 — Write the Skill Definition (system prompt)

The `## Skill Definition` section contains the complete system prompt. This is the most important part — it must be copy-pasteable and produce consistent output.

```
## Skill Definition

```
[System Prompt]

You are a senior {{LANGUAGE}} engineer conducting a professional code review.

Review the provided code for:
1. Correctness — logic errors, null handling
2. Security — OWASP Top 10
3. Performance — unnecessary loops, blocking calls
4. Maintainability — naming, complexity

Format your response as:
## Summary
One-line verdict.

## Critical
- [LINE X] Issue. Suggested fix: ...

## Major
- [LINE X] Issue.

Code to review:
{{CODE}}
```
```

### Step 4 — Add all delivery mode usage examples

In the `## Usage` section, show how to use the skill in each declared delivery mode:

- **Copilot Chat:** `@workspace #file:<filename> ...`
- **Standalone:** copy system prompt → set as LLM system message → send inputs as user message
- **API Endpoint:** `POST /invoke` JSON body (see `docs/runtime-adapters.md`)
- **MCP Tool:** JSON tool call

### Step 5 — Test and add a real example

Test your skill on at least two models. Add a real (redacted) input/output pair to `## Examples`.

**Reference:** See `workstreams/developers/skills/code-review.md` for a complete example.

---

## Stage 2C — Building an Agent

An agent orchestrates multiple skills dynamically. Use an agent when the task involves planning, branching, or iterative tool use.

### Step 1 — Design the architecture first

Before writing the agent instructions, answer:
- What skills/tools does the agent use?
- What decisions does the agent make on its own vs. deferring to humans?
- What is the blast radius if it makes a mistake?
- What is the maximum number of iterations?

Draw a simple flow diagram — you will include it in the `## Architecture` section.

### Step 2 — Copy the template

```bash
cp templates/agent-template.md workstreams/<workstream>/agents/<your-agent-name>.md
```

### Step 3 — Fill in the System Prompt / Agent Instructions

This is the most critical section. A good agent prompt includes:

```
[System Prompt]

You are an autonomous [role] responsible for [goal].

Your goals:
1. [Specific goal 1]
2. [Specific goal 2]

Tools available to you:
- tool_name: description of what it does and when to use it
- skill_id: description

Rules:
- Never take [destructive action] without explicit user confirmation.
- If uncertain, ask rather than guess.
- Always explain your reasoning before acting.

Output format:
[Define expected output structure]
```

### Step 4 — Fill in the Configuration section

```yaml
agent_id: your-agent-id
model: gpt-4o
temperature: 0.1
max_iterations: 10
tools:
  - tool_one
  - tool_two
```

### Step 5 — Write Security Considerations

For agents, this section is **required** and must be specific. Cover:
- What external systems the agent accesses
- What the blast radius of a mistake is
- What guardrails prevent runaway actions
- Which actions require human approval

**Reference:** See `workstreams/developers/agents/pr-review-agent.md` for a complete example.

---

## Stage 2D — Building a Workflow

A workflow is a fixed sequence of 2+ steps, each using a skill or prompt.

### Step 1 — Map out the steps on paper first

Draw the sequence before opening any files:

```
Input: PR Title, Description, Diff
  ↓
Step 1: Summarise PR → Summary text
  ↓
Step 2: Code Review → Findings list
  ↓
Step 3: Coverage Assessment → Coverage paragraph
  ↓
Step 4: Consolidate → Final report
  ↓
Output: Quality Gate Report
```

### Step 2 — Copy the template

```bash
cp templates/workflow-template.md workstreams/<workstream>/workflows/<your-workflow-name>.md
```

### Step 3 — Write each step as a self-contained prompt

Each step in the `## Steps` section must include:
- **Asset used** — link to the skill/prompt ID being invoked
- **Input** — what goes in (from the workflow input or previous step's output)
- **Output** — what this step produces (and what the next step consumes)
- **The prompt itself** — copy-pasteable, with placeholders

### Step 4 — Draw the Flow Diagram

Include an ASCII or Mermaid diagram in the `## Flow Diagram` section. This is what users skim first.

### Step 5 — Explain both manual and automated usage

Show how to run the workflow:
1. Manually (step by step in an LLM chat window)
2. Automatically (Bitbucket Pipelines, API, or other trigger)

**Reference:** See `workstreams/developers/workflows/pr-review-workflow.md` for a complete example.

---

## Stage 2E — Building an Automation

An automation is a scripted implementation of a workflow or skill that runs automatically.

### Step 1 — Start from a working Workflow

Every good automation starts as a workflow. Build and test the workflow manually first, then automate it.

### Step 2 — Copy the template

```bash
cp templates/automation-template.md workstreams/<workstream>/automations/<your-automation-name>.md
```

### Step 3 — Choose the trigger

| Trigger | When to use |
|---|---|
| PR opened/updated | Quality gates, auto-labelling |
| Push to main | Catalog sync, documentation generation |
| Schedule (cron) | Nightly reports, periodic analysis |
| Manual (workflow_dispatch) | On-demand batch operations |

### Step 4 — Write the Implementation

Include a complete, runnable script or workflow YAML. Reference implementations:
- Bitbucket Pipelines: see `workstreams/developers/automations/pr-validator.md`
- Shell script: use the bash template in `templates/automation-template.md`

### Step 5 — Document security requirements

For automations, the `## Security Notes` section is **required**. Cover:
- What credentials are required and how they are stored (always Bitbucket Secured Variables)
- What systems the automation accesses
- Whether the automation is idempotent

**Reference:** See `workstreams/developers/automations/pr-validator.md` for a complete example.

---

## Stage 3 — Test Your Asset

Before submitting, test your asset thoroughly:

### For Prompts and Skills

1. **Test on at least 2 LLMs** — gpt-4o and claude-3-5-sonnet are the minimum.
2. **Test at least 3 input variations** — typical input, edge case, and invalid/empty input.
3. **Check output consistency** — run the same input 3 times. Is the output consistently useful?
4. **Fill in the Testing Notes table** — mark ✅ for tested models, ⬜ for untested.

### For Agents

1. **Run a complete end-to-end session** — not just step 1.
2. **Test the failure path** — what happens if one tool fails?
3. **Verify guardrails work** — try to trick the agent into a destructive action.

### For Workflows

1. **Run every step manually** — paste each prompt into an LLM and verify the output.
2. **Verify hand-offs** — use the output of Step N as the actual input to Step N+1.

### For Automations

1. **Run locally first** — before pushing to Bitbucket Pipelines.
2. **Test idempotency** — run it twice and verify the second run does not double-post or duplicate work.

---

## Stage 4 — Register and Submit

### Step 1 — Add an entry to `catalog/index.json`

Follow the existing format. Minimum required fields:

```json
{
  "id": "dev-commit-message-generator",
  "title": "Conventional Commit Message Generator",
  "type": "prompt",
  "status": "draft",
  "workstream": ["developers"],
  "tags": ["git", "commit-message"],
  "description": "Generates a Conventional Commits message from a diff or description.",
  "path": "workstreams/developers/prompts/commit-message-generator.md",
  "updated": "2026-04-20",
  "delivery_modes": ["llm-chat"],
  "package_ids": ["developers-core-pack"]
}
```

> **Note:** `catalog/index.json` is automatically rebuilt on merge to main by the `catalog-sync` GitHub Action. You still need to add the entry manually in your PR for the review to see the full picture.

### Step 2 — Commit with a conventional commit message

```bash
git add workstreams/developers/prompts/commit-message-generator.md catalog/index.json
git commit -m "feat(developers/prompts): add commit-message-generator prompt"
```

### Step 3 — Open a Pull Request

Use the PR template. The front-matter validation GitHub Action will run automatically and post a comment on the PR.

---

## Stage 5 — Review and Merge

1. A maintainer reviews within 5 business days.
2. If the front-matter validation fails, fix the issues flagged in the PR comment.
3. On approval, the maintainer updates `status: draft` → `status: active` in the asset file.
4. After merge, the catalog-sync Action automatically rebuilds `catalog/index.json`.

---

## Combinations

### Idea → Prompt

1. Submit idea (Stage 0).
2. Idea approved (`idea:approved` label).
3. Build a prompt (Stage 2A).
4. No dependencies on other assets.

### Idea → Skill (most common)

1. Submit idea.
2. Design inputs and outputs.
3. Build the skill (Stage 2B).
4. Add to `catalog/index.json` and the relevant package manifest.

### Idea → Workflow (idea spans multiple existing skills)

1. Submit idea listing which skills it combines.
2. Ensure referenced skills already exist (or create them first).
3. Build the workflow (Stage 2D) referencing the skill IDs in `dependencies[]`.

### Idea → Agent (idea requires dynamic planning)

1. Submit idea explaining why fixed steps are insufficient.
2. Build and test the individual skills the agent will use.
3. Build the agent (Stage 2C) referencing those skills in `dependencies[]`.

### Idea → Automation (idea is "run X automatically")

1. Submit idea specifying the trigger (PR, push, schedule).
2. Build the workflow that the automation implements (Stage 2D) — test it manually.
3. Build the automation (Stage 2E) that wraps the workflow.

### Skill → Agent → Workflow → Automation (full stack)

Some ideas grow into a full stack:

```
1. Idea: "Auto-review PRs"
2. Skill: dev-code-review-skill (already exists)
3. Skill: dev-summarize-pull-request (already exists)
4. Agent: dev-pr-review-agent (orchestrates both skills dynamically)
5. Workflow: dev-pr-review-workflow (fixed sequence for teams that prefer determinism)
6. Automation: dev-pr-validator (runs the workflow on every PR via Bitbucket Pipelines)
```

Cross-reference all of these in `dependencies[]` and `catalog/index.json`.

---

## Quick Reference — File Locations

| Asset Type | Location | Naming |
|---|---|---|
| Idea | `workstreams/<ws>/ideas/<name>.md` | `my-idea-name.md` |
| Prompt | `workstreams/<ws>/prompts/<name>.md` | `action-verb-noun.md` |
| Skill | `workstreams/<ws>/skills/<name>.md` | `action-verb-noun.md` |
| Agent | `workstreams/<ws>/agents/<name>.md` | `noun-agent.md` |
| Workflow | `workstreams/<ws>/workflows/<name>.md` | `noun-workflow.md` |
| Automation | `workstreams/<ws>/automations/<name>.md` | `noun-validator.md` |
| Shared asset | `shared/<type>/<name>.md` | same as above |

**Templates:** `templates/<type>-template.md`  
**Schemas:** `schemas/<type>.schema.json`  
**Catalog:** `catalog/index.json`  
**Best practices:** `docs/best-practices.md`  
**Runtime adapters:** `docs/runtime-adapters.md`
