# AI-Skills

> **A centralized repository for AI ideas, automation scripts, skills, agents, workflows, and prompts — organized by work stream and built for collaboration.**

---

## Overview

AI-Skills is a shared knowledge base that helps teams standardize, reuse, and build upon AI-powered capabilities across the organization. Whether you are a system designer exploring architecture patterns, a developer looking for ready-made prompts, or an SRE automating incident response, this repository has a home for your contribution.

The repository is designed to be:

- **Reusable** — every asset follows a consistent schema so it can be discovered and consumed easily.
- **Collaborative** — any team member can propose ideas, refine existing content, and review contributions through pull requests.
- **Secure** — all contributions are reviewed against the security standards described in [SECURITY.md](SECURITY.md).
- **Portable** — content is stored as plain text (Markdown + JSON) so it works with any LLM runtime today (GitHub Copilot, GPT, Claude, Gemini) and can be surfaced through a future portal or API layer.

---

## Repository Structure

```
AI-Skills/
├── README.md                  # You are here
├── CONTRIBUTING.md            # How to contribute
├── SECURITY.md                # Security standards and responsible AI guidelines
├── CODE_OF_CONDUCT.md         # Community standards
│
├── .github/
│   ├── ISSUE_TEMPLATE/        # Templates for raising ideas, skills, and bugs
│   └── PULL_REQUEST_TEMPLATE.md
│
├── docs/
│   ├── architecture.md        # Repository design decisions
│   ├── getting-started.md     # Onboarding guide
│   └── standards.md           # Naming, metadata, and quality standards
│
├── templates/                 # Starter files for each content type
│   ├── skill-template.md
│   ├── agent-template.md
│   ├── workflow-template.md
│   ├── prompt-template.md
│   ├── automation-template.md
│   └── idea-template.md
│
├── schemas/                   # JSON schemas for validation
│   ├── skill.schema.json
│   ├── agent.schema.json
│   ├── workflow.schema.json
│   ├── prompt.schema.json
│   ├── automation.schema.json
│   └── idea.schema.json
│
├── workstreams/               # Work-stream-specific content
│   ├── architects/
│   ├── capacity-management/
│   ├── developers/
│   ├── performance-engineers/
│   ├── sre/
│   ├── system-designers/
│   └── testers/
│
├── shared/                    # Cross-workstream reusable content
│   ├── skills/
│   ├── agents/
│   ├── workflows/
│   ├── prompts/
│   ├── automations/
│   └── ideas/
│
└── catalog/
    └── index.json             # Machine-readable catalog of all entries
```

Each work stream folder follows the same internal layout:

```
workstreams/<work-stream>/
├── README.md
├── skills/
├── agents/
├── workflows/
├── prompts/
├── automations/
└── ideas/
```

---

## Content Types

| Type | Description |
|---|---|
| **Idea** | An early-stage concept or use-case proposal — not yet implemented. |
| **Prompt** | A reusable prompt or prompt template targeting a specific task. |
| **Skill** | A focused, single-purpose AI capability (e.g. "summarize pull request"). |
| **Agent** | An autonomous AI agent that combines multiple skills or tools. |
| **Workflow** | A multi-step process that orchestrates prompts, skills, or agents. |
| **Automation** | A script or pipeline that automates a repetitive task using AI. |

---

## Work Streams

| Work Stream | Description |
|---|---|
| `architects` | Solution and enterprise architecture patterns, design reviews, ADRs. |
| `capacity-management` | Demand forecasting, resource sizing, utilization analysis. |
| `developers` | Code generation, review, refactoring, documentation. |
| `performance-engineers` | Load testing, profiling, bottleneck analysis. |
| `sre` | Incident response, runbooks, alert triage, reliability automation. |
| `system-designers` | System design, diagramming, trade-off analysis. |
| `testers` | Test case generation, coverage analysis, exploratory testing. |

> Content that is useful across multiple work streams belongs in `shared/`.

---

## Quick Start

1. **Browse** existing content in `workstreams/` or `shared/`.
2. **Use a template** from the `templates/` folder to create a new asset.
3. **Submit** your asset via a pull request — see [CONTRIBUTING.md](CONTRIBUTING.md).
4. **Discuss** ideas early by opening a GitHub Issue with the *Idea Submission* template.

---

## LLM Compatibility

Assets in this repository are designed to be LLM-agnostic. Each prompt or skill includes a `llm_compatibility` metadata field so consumers know which models have been tested.

| LLM | Status |
|---|---|
| GitHub Copilot (GPT-4o) | ✅ Supported |
| OpenAI GPT-4 / GPT-4 Turbo | ✅ Supported |
| Anthropic Claude 3 / 3.5 | ✅ Supported |
| Google Gemini 1.5 | ✅ Supported |
| Azure OpenAI | ✅ Supported |
| In-house / self-hosted LLMs | 🔄 Planned |
| Copilot Studio | 🔄 Planned |

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for detailed instructions, including naming conventions, required metadata, and the review process.

## Security

See [SECURITY.md](SECURITY.md) for responsible AI guidelines, data classification rules, and how to report vulnerabilities.

## Code of Conduct

See [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md).

---

## License

This project is licensed under the [MIT License](LICENSE).
