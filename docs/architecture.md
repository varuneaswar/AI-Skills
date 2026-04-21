# Repository Architecture

This document explains the design decisions behind the AI-Skills repository structure.

---

## Design Goals

1. **Discoverability** — Any team member should be able to find a relevant asset within a few clicks.
2. **Standardization** — Every asset has the same front-matter schema, so tooling (portal, CLI, CI) can parse and validate it automatically.
3. **Portability** — Assets are stored as plain Markdown + JSON, making them usable with any LLM runtime today and in future API layers.
4. **Extensibility** — Adding a new work stream or content type requires only creating a new folder and registering it in the catalog.
5. **Security** — Data classification and responsible-AI guidelines are built into the contribution process.

---

## Directory Layout Rationale

### `workstreams/`

Each work stream (e.g., `developers`, `sre`) is a first-class namespace. This prevents naming collisions and makes it easy for a team to own and maintain their own corner of the repository without affecting others.

Within each work stream, content is split by **type** (`skills/`, `prompts/`, etc.) rather than by project. This keeps similar things together and simplifies search.

### `shared/`

Content that is genuinely reusable across multiple work streams lives here. Contributors should prefer the more specific `workstreams/` location unless the asset is clearly cross-cutting.

### `templates/`

Starter files reduce the cognitive overhead of contributing. Each template includes all required front-matter fields with inline comments explaining each field.

### `schemas/`

JSON schemas allow:
- CI/CD pipelines to validate that every asset has complete and well-typed metadata.
- A future portal to dynamically render asset cards without hard-coding field names.
- IDE extensions to provide autocomplete for front-matter fields.

### `catalog/index.json`

A flat list of all assets with key metadata. This file is the integration point for:
- A future web portal to power search and filtering.
- CLI tooling to list or retrieve assets.
- An internal API layer that serves assets to Copilot Studio or other consumers.

### `workstreams/registry.json`

The **single source of truth** for all registered work streams. Adding a new work stream requires only:
1. Appending an entry to `workstreams/registry.json`.
2. Creating the matching folder structure.

No schema files need to be modified — the `workstream` field in all schemas accepts any string value and defers to the registry for the authoritative list. See `docs/adding-workstreams.md` for the full guide.

### `catalog/packages/`

Package manifests define the unit of delivery. Each manifest lists exactly which asset files are included and from which sub-folder (`skills/`, `prompts/`, `workflows/`, `shared/skills/`, etc.), so the portal knows precisely what to bundle when a user requests a capability. See `docs/packaging.md` for the complete packaging guide.

Issue and PR templates guide contributors to provide the information maintainers need, reducing back-and-forth in reviews.

---

## Asset Lifecycle

```
Idea (GitHub Issue)
   │
   ▼
draft (file created, PR open)
   │
   ▼
active (PR merged, status = "active")
   │
   ▼
deprecated (superseded by newer version, status = "deprecated")
```

Deprecated assets are kept in the repository for historical reference and are clearly marked in their front matter.

---

## Versioning Strategy

Assets use [Semantic Versioning](https://semver.org/):

- **PATCH** (1.0.x) — wording tweaks, typo fixes.
- **MINOR** (1.x.0) — new optional parameters, added examples.
- **MAJOR** (x.0.0) — breaking changes to inputs/outputs, significant behaviour change.

Version changes must be documented in a `## Changelog` section at the bottom of the asset file.

---

## Future Extensibility

### Delivery Mode Differentiation: Current vs Future

Every asset and package manifest declares which delivery modes it supports and whether each is `current` or `planned`. This is the mechanism that differentiates the current IDE-based workflow from the future portal-and-API-layer workflow:

| Delivery Mode | Availability | Description |
|---|---|---|
| `copilot-chat` | **Current** | Skill definition is copied from the Markdown file and pasted into any LLM interface (GitHub Copilot Chat, ChatGPT, Claude, etc.) |
| `api-endpoint` | **Planned** | A portal REST API accepts a skill invocation request and returns the LLM response |
| `mcp-tool` | **Planned** | Skill is registered as an MCP (Model Context Protocol) tool, callable by agents and IDEs |
| `copilot-studio` | **Planned** | Package imported as a Copilot Studio connector for Teams / M365 Copilot |
| `in-house-llm` | **Planned** | Skill routed to an internal API layer — no data leaves the corporate network |

Assets that support all delivery modes simply declare `"delivery_modes": ["copilot-chat", "api-endpoint", "mcp-tool", "copilot-studio", "in-house-llm"]`. The same asset file works for all delivery modes because the skill definition (system prompt) is format-agnostic.

The `catalog/index.json` file is designed to be consumed by a web portal. Each entry contains enough metadata for a search card (id, title, type, description, tags, work stream). A backend API can serve this file or read individual Markdown files on demand.

### Copilot Studio / In-House LLM API

Assets with `type: skill` or `type: agent` will eventually be packaged as callable functions or MCP tools. The `inputs` and `outputs` fields in the front matter map directly to function parameters and return values.

### CLI Tooling

A future `ai-skills` CLI could:
- `ai-skills search "incident triage"` — search the catalog.
- `ai-skills get sre/prompts/incident-triage` — print the prompt text.
- `ai-skills validate workstreams/` — validate all front matter against schemas.

---

## Runtime Adapter Contract

The `## Skill Definition` section in every skill and agent asset is the **canonical system prompt** — the single source of truth that all delivery modes read from.

### How adapters consume assets

```
Markdown asset file
       │
       ├── YAML front-matter
       │     ├── inputs[]  ──▶  request parameters (API / MCP / Copilot Studio)
       │     ├── outputs[] ──▶  response shape
       │     └── delivery_modes[] ──▶  which adapters activate this asset
       │
       └── Body: ## Skill Definition
             └── ``` code block ``` ──▶  LLM system prompt (all delivery modes)
```

No changes to the Markdown file are required when adding a new delivery mode. The adapter reads the same file and wraps it differently.

### Delivery mode adapter map

| Delivery Mode | How the asset is consumed |
|---|---|
| `copilot-chat` | User copies `## Skill Definition` block and pastes into any LLM interface |
| `api-endpoint` | Portal backend reads file, extracts system prompt, substitutes inputs, calls LLM |
| `mcp-tool` | MCP server auto-generates tool definition from `inputs[]`; calls LLM with extracted system prompt |
| `copilot-studio` | Connector YAML auto-generated from package manifest + asset `inputs[]`/`outputs[]` |
| `in-house-llm` | Same as `api-endpoint` but LLM client points to internal proxy |

For the full adapter specification — including pseudocode, request/response shapes, and MCP tool definition format — see **[docs/runtime-adapters.md](runtime-adapters.md)**.

### CI / CD integration

Two GitHub Actions enforce consistency:

| Workflow | Trigger | What it does |
|---|---|---|
| `validate-frontmatter.yml` | Pull request | Validates all asset front-matter against JSON schemas; blocks merge on failure |
| `catalog-sync.yml` | Push to main | Rebuilds `catalog/index.json` from asset front-matter automatically |
