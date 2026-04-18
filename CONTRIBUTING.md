# Contributing to AI-Skills

Thank you for contributing! This guide explains how to add or improve content in the AI-Skills repository.

---

## Table of Contents

- [Before You Start](#before-you-start)
- [Content Types and When to Use Each](#content-types-and-when-to-use-each)
- [Where to Place Your Contribution](#where-to-place-your-contribution)
- [File Naming Conventions](#file-naming-conventions)
- [Required Metadata](#required-metadata)
- [Step-by-Step Contribution Guide](#step-by-step-contribution-guide)
- [Review Process](#review-process)
- [Quality Standards](#quality-standards)

---

## Before You Start

1. Search the repository to ensure your contribution does not duplicate something that already exists.
2. If you have an early-stage idea, open a GitHub Issue using the **Idea Submission** template before writing content.
3. Read [SECURITY.md](SECURITY.md) to ensure your content meets the responsible-AI and data-classification standards.

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
dependencies:
  - id-of-another-asset
inputs:
  - name: param_name
    type: string
    description: What this input represents
outputs:
  - name: output_name
    type: string
    description: What this output represents
examples:
  - input: "example input"
    output: "example output"
security_classification: public | internal | confidential
```

---

## Step-by-Step Contribution Guide

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

4. **Validate** your front matter against the JSON schema in `schemas/`:
   ```bash
   # Using ajv-cli (npm install -g ajv-cli)
   ajv validate -s schemas/prompt.schema.json -d workstreams/developers/prompts/explain-code-change.md
   ```

5. **Update** `catalog/index.json` by adding an entry for your new asset (follow the existing format).

6. **Commit** your changes with a descriptive message:
   ```
   git commit -m "feat(developers/prompts): add explain-code-change prompt"
   ```

7. **Open a pull request** against `main` using the PR template.

---

## Review Process

All contributions go through a pull-request review:

1. A maintainer will review the content for quality, accuracy, and adherence to standards within **5 business days**.
2. Security-sensitive content (agents, automations with external integrations) requires a secondary review from the security team.
3. Once approved, the PR is merged and the asset status is changed from `draft` to `active`.

---

## Quality Standards

- **Accuracy** — prompts and skills must have been tested against at least one supported LLM.
- **Examples** — include at least one input/output example.
- **No secrets** — never include API keys, tokens, passwords, or personally identifiable information.
- **No hard-coded endpoints** — use placeholder variables (e.g., `{{BASE_URL}}`) instead of real URLs.
- **Idempotency** — automations should be safe to run multiple times without unintended side effects.
- **Versioning** — when updating an existing asset, increment the `version` field and add a changelog entry at the bottom of the file.
