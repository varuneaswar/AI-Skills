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

### `.github/`

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

### Portal Integration

The `catalog/index.json` file is designed to be consumed by a web portal. Each entry contains enough metadata for a search card (id, title, type, description, tags, work stream). A backend API can serve this file or read individual Markdown files on demand.

### Copilot Studio / In-House LLM API

Assets with `type: skill` or `type: agent` will eventually be packaged as callable functions or MCP tools. The `inputs` and `outputs` fields in the front matter map directly to function parameters and return values.

### CLI Tooling

A future `ai-skills` CLI could:
- `ai-skills search "incident triage"` — search the catalog.
- `ai-skills get sre/prompts/incident-triage` — print the prompt text.
- `ai-skills validate workstreams/` — validate all front matter against schemas.
