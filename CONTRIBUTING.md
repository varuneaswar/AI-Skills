# Contributing to AI-Skills

Thank you for contributing! This guide explains how to add or improve content in the AI-Skills repository.

---

## Table of Contents

- [Two Ways to Contribute](#two-ways-to-contribute)
- [Lane 2 — Non-Technical: Submit an Idea via GitHub Issues](#lane-2--non-technical-submit-an-idea-via-github-issues)
- [Lane 1 — Technical: Contribute Assets via Pull Request](#lane-1--technical-contribute-assets-via-pull-request)
  - [Before You Start](#before-you-start)
  - [Content Types and When to Use Each](#content-types-and-when-to-use-each)
  - [Where to Place Your Contribution](#where-to-place-your-contribution)
  - [File Naming Conventions](#file-naming-conventions)
  - [Required Metadata](#required-metadata)
  - [Step-by-Step Contribution Guide](#step-by-step-contribution-guide)
  - [Review Process](#review-process)
  - [Quality Standards](#quality-standards)

---

## Two Ways to Contribute

**Not a developer? You can still contribute.** This repository supports two contribution lanes:

| Lane | Who it's for | How |
|---|---|---|
| **Lane 2** | Anyone — product managers, domain experts, business analysts | Submit an idea as a GitHub Issue. Maintainers convert approved ideas to assets. |
| **Lane 1** | Technical contributors — engineers, architects | Create or update Markdown assets directly via Pull Request. |

---

## Lane 2 — Non-Technical: Submit an Idea via GitHub Issues

You do not need to know Git, YAML, or Markdown to contribute an idea. Here is all you need to do:

### Step 1 — Open a GitHub Issue

1. Go to [github.com/varuneaswar/AI-Skills/issues](https://github.com/varuneaswar/AI-Skills/issues).
2. Click **New Issue**.
3. Select the **Idea Submission** template.

### Step 2 — Fill in the template

Answer these questions in plain language — no technical jargon required:

- **What problem does this idea solve?** Who is affected, and how often?
- **What would a solution look like?** Describe it in plain English.
- **Which team or workstream does this serve?** (e.g., developers, SRE, testers)
- **How will we know it is working?** List 2–3 measurable success criteria.

### Step 3 — Submit and wait for review

A maintainer will:

1. Add the label **`idea:under-review`** within 5 business days.
2. Ask any clarifying questions as comments on the issue.
3. Add **`idea:approved`** if the idea is accepted for development, or **`idea:needs-work`** if changes are needed.
4. Once approved, a maintainer or volunteer engineer will convert the idea to a Markdown asset via a PR, and add the **`automation:in-progress`** label.
5. When the asset is merged, the issue is closed with a link to the new asset file.

### Idea Vetting Flow

```
idea:submitted
    │
    ▼ (maintainer reviews)
idea:under-review
    │
    ├── needs more detail ──▶ idea:needs-work (author updates issue)
    │                              │
    │                              └──▶ idea:under-review (back to review)
    │
    └── accepted ──▶ idea:approved
                          │
                          ▼ (engineer picks it up)
                    automation:in-progress
                          │
                          ▼ (PR opened and merged)
                    asset status: active (issue closed)
```

### Issue Labels Reference

| Label | Meaning |
|---|---|
| `idea:submitted` | New idea, not yet reviewed |
| `idea:under-review` | Maintainer is reviewing |
| `idea:approved` | Approved for development |
| `idea:needs-work` | Returned to author for clarification |
| `idea:declined` | Not accepted (reason given in comments) |
| `automation:in-progress` | Engineer is building the asset |

---

## Lane 1 — Technical: Contribute Assets via Pull Request

---

## Lane 1 — Technical: Contribute Assets via Pull Request

### Before You Start

1. Search the repository to ensure your contribution does not duplicate something that already exists.
2. If you have an early-stage idea, open a GitHub Issue using the **Idea Submission** template before writing content.
3. Read [SECURITY.md](SECURITY.md) to ensure your content meets the responsible-AI and data-classification standards.
4. Read [docs/best-practices.md](docs/best-practices.md) for prompt engineering tips and quality checklists.
5. Read [docs/building-ideas.md](docs/building-ideas.md) for a step-by-step walkthrough of every asset type.

---

## Content Types and When to Use Each

| Type | Use when… | Template |
|---|---|---|
| **Idea** | You have a concept that is not yet built. | [`templates/idea-template.md`](templates/idea-template.md) |
| **Prompt** | You have a reusable prompt or prompt template. | [`templates/prompt-template.md`](templates/prompt-template.md) |
| **Skill** | You have a focused, single-purpose AI capability. | [`templates/skill-template.md`](templates/skill-template.md) |
| **Agent** | You have an autonomous agent combining multiple tools/skills. | [`templates/agent-template.md`](templates/agent-template.md) |
| **Workflow** | You have a multi-step process orchestrating several assets. | [`templates/workflow-template.md`](templates/workflow-template.md) |
| **Automation** | You have a script or pipeline automating a repetitive task. | [`templates/automation-template.md`](templates/automation-template.md) |

---

## Where to Place Your Contribution

- **Work-stream-specific content** → `workstreams/<work-stream>/<type>/`
  - E.g., a developer-focused prompt goes in `workstreams/developers/prompts/`.
- **Cross-workstream content** → `shared/<type>/`
  - Use `shared/` when your asset is genuinely useful across multiple work streams.

Available work streams:
- `architects`
- `capacity-management`
- `developers`
- `performance-engineers`
- `sre`
- `system-designers`
- `testers`

---

## File Naming Conventions

- Use **lowercase kebab-case**: `summarize-pull-request.md`
- Do not use spaces or underscores in file names.
- Be descriptive — the file name should convey what the asset does.
- Append the asset type only if it is ambiguous: prefer `incident-triage.md` over `incident-triage-prompt.md` inside a `prompts/` folder.

---

## Required Metadata

Every Markdown asset must begin with a YAML front-matter block containing at minimum:

```yaml
---
id: unique-kebab-case-id          # globally unique across the repo
title: Human-Readable Title
type: prompt | skill | agent | workflow | automation | idea
version: "1.0.0"
status: draft | active | deprecated
workstream:
  - developers                    # one or more work streams; use "shared" for cross-cutting
author: your-github-username
created: YYYY-MM-DD
updated: YYYY-MM-DD
tags:
  - tag1
  - tag2
llm_compatibility:
  - gpt-4o
  - claude-3-5-sonnet
description: |
  One or two sentences describing what this asset does and why it is useful.
---
```

Optional fields (include when relevant):

```yaml
delivery_modes:
  - copilot-chat          # current: copy-paste into any LLM interface
  - api-endpoint          # future: portal REST API
  - mcp-tool              # future: MCP server
  - copilot-studio        # future: Copilot Studio connector
  - in-house-llm          # future: internal LLM proxy
dependencies:
  - id-of-another-asset
inputs:
  - name: param_name
    type: string
    description: What this input represents
    required: true
outputs:
  - name: output_name
    type: string
    description: What this output represents
security_classification: public | internal | confidential
```

> **Note:** `inputs` and `outputs` are **required** for skills. They are strongly recommended for agents and workflows. See `schemas/skill.schema.json` for the full validation rules.

---

## Step-by-Step Contribution Guide

For a detailed walkthrough of every asset type including examples, see **[docs/building-ideas.md](docs/building-ideas.md)**.

**Quick steps:**

1. **Fork** the repository (external contributors) or **create a branch** (internal contributors):
   ```
   git checkout -b feat/<work-stream>/<type>/<short-description>
   ```
   Example: `feat/developers/prompts/explain-code-change`

2. **Copy** the appropriate template from `templates/` to the correct location:
   ```
   cp templates/prompt-template.md workstreams/developers/prompts/explain-code-change.md
   ```

3. **Fill in** the template — especially the YAML front matter and the body sections.

4. **Add an entry** to `catalog/index.json` for your new asset (follow the existing format).

5. **Commit** your changes with a descriptive message:
   ```
   git commit -m "feat(developers/prompts): add explain-code-change prompt"
   ```

6. **Open a pull request** against `main` using the PR template. The front-matter validation workflow will run automatically.

---

## Review Process

All contributions go through a pull-request review:

1. A maintainer will review the content for quality, accuracy, and adherence to standards within **5 business days**.
2. Security-sensitive content (agents, automations with external integrations) requires a secondary review from the security team.
3. Once approved, the PR is merged and the asset status is changed from `draft` to `active`.
4. `catalog/index.json` is automatically rebuilt on merge by the `catalog-sync` GitHub Action.

---

## Quality Standards

- **Accuracy** — prompts and skills must have been tested against at least one supported LLM.
- **Examples** — include at least one input/output example.
- **No secrets** — never include API keys, tokens, passwords, or personally identifiable information.
- **No hard-coded endpoints** — use placeholder variables (e.g., `{{BASE_URL}}`) instead of real URLs.
- **Idempotency** — automations should be safe to run multiple times without unintended side effects.
- **Versioning** — when updating an existing asset, increment the `version` field and add a changelog entry at the bottom of the file.
- **delivery_modes** — all skills, agents, and workflows must declare `delivery_modes` in the front-matter.

See [docs/best-practices.md](docs/best-practices.md) for a full quality checklist per asset type.
