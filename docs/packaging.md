# Packaging Guide

This document explains how skills are packaged, what a package contains, and how to request or deliver a package through the portal (current and future).

---

## What Is a Package?

A **package** is a bundle of one or more AI-Skills assets that together provide a cohesive capability for a specific work stream. It is the unit of delivery — what the portal produces when a user requests "the SRE incident response toolkit" rather than a single prompt.

A package answers the question: *"When I ask the portal for the developers code-review capability, exactly which files from which folders are included, and how should they be delivered to me?"*

---

## Package Manifest

Every package is described by a JSON manifest in `catalog/packages/<package-id>.json`. The manifest:

1. Lists **every asset** included in the package, explicitly stating the `source_folder` it was drawn from.
2. Defines **delivery modes** (how the package can be consumed today and in future).
3. Specifies the **entry asset** (the primary skill or agent the user invokes).
4. Includes **configuration** (recommended model, temperature, token limits).

### Anatomy of a Package Manifest

```json
{
  "id": "developers-core-pack",          // globally unique identifier
  "name": "Developers Core Pack",        // display name in portal
  "version": "1.0.0",
  "status": "active",
  "workstream": "developers",            // primary work stream
  "entry_asset": "dev-code-review-skill",

  "assets": [
    // Every file bundled, with its source_folder explicitly declared
    { "id": "dev-code-review-skill",  "role": "entry_skill",      "source_folder": "skills",        "path": "...", "required": true },
    { "id": "dev-summarize-pull-request", "role": "prompt",       "source_folder": "prompts",       "path": "...", "required": true },
    { "id": "shared-explain-concept", "role": "supporting_skill", "source_folder": "shared/skills", "path": "...", "required": false }
  ],

  "delivery_modes": [
    { "mode": "copilot-chat",   "availability": "current",  "instructions": "..." },
    { "mode": "api-endpoint",   "availability": "planned",  "endpoint": "{{API_BASE_URL}}/..." },
    { "mode": "mcp-tool",       "availability": "planned",  "endpoint": "{{MCP_SERVER_URL}}/..." },
    { "mode": "copilot-studio", "availability": "planned",  "instructions": "..." },
    { "mode": "in-house-llm",   "availability": "planned",  "instructions": "..." }
  ],

  "configuration": { "model": "gpt-4o", "temperature": 0.2 }
}
```

### Source Folder Mapping

The `source_folder` field in each asset entry is the key traceability mechanism. It tells you exactly which sub-folder within the repository each component comes from:

| `source_folder` value | What it means |
|---|---|
| `skills` | From `workstreams/<workstream>/skills/` |
| `prompts` | From `workstreams/<workstream>/prompts/` |
| `workflows` | From `workstreams/<workstream>/workflows/` |
| `automations` | From `workstreams/<workstream>/automations/` |
| `agents` | From `workstreams/<workstream>/agents/` |
| `shared/skills` | From `shared/skills/` — cross-workstream |
| `shared/prompts` | From `shared/prompts/` — cross-workstream |
| `shared/workflows` | From `shared/workflows/` — cross-workstream |
| `shared/automations` | From `shared/automations/` — cross-workstream |
| `shared/agents` | From `shared/agents/` — cross-workstream |

This ensures that when the portal generates a package download (ZIP, JSON bundle, or API response), it knows exactly which files to include.

---

## Delivery Modes: Current vs Future

This is how the repository differentiates between what works **today** and what is **planned** for the future portal and API layer.

| Delivery Mode | Availability | How It Works |
|---|---|---|
| `copilot-chat` | **Current** | Copy the skill definition from the Markdown file, paste as system prompt into GitHub Copilot Chat, ChatGPT, Claude, or any IDE LLM interface. |
| `api-endpoint` | **Planned** | A future REST API serves the packaged skill. The portal calls `POST /packages/<id>/invoke` with the inputs payload and returns the LLM response. |
| `mcp-tool` | **Planned** | The skill is registered as an MCP (Model Context Protocol) tool. IDEs, bots, or agent frameworks call it by tool name. |
| `copilot-studio` | **Planned** | The package is imported as a Copilot Studio custom skill or connector, making it available in Microsoft 365 Copilot and Teams. |
| `in-house-llm` | **Planned** | The skill definition is deployed to an in-house LLM endpoint via an internal API layer. No data leaves the corporate network. |

### Current Setup (Today)

```
Developer / SRE / Architect
        │
        ▼
  GitHub / Browser
  (reads repo Markdown)
        │
        ▼
  Copy skill definition
        │
        ▼
  Paste into Copilot Chat / ChatGPT / Claude
  (IDE plugin or web browser)
        │
        ▼
  LLM returns result
```

### Future Setup (Portal + API Layer)

```
User (any work stream)
        │
        ▼
  AI-Skills Portal
  (web UI — searches catalog, selects package)
        │
        ├──► copilot-chat   → "Use in IDE" button (copies skill to clipboard)
        │
        ├──► api-endpoint   → Portal calls POST /packages/<id>/invoke
        │                     → In-house API layer or Azure OpenAI / external LLM
        │
        ├──► mcp-tool       → MCP server serves the tool definition
        │                     → IDE / agent framework calls it by name
        │
        ├──► copilot-studio → Package imported as connector
        │                     → Available in Teams / M365 Copilot
        │
        └──► in-house-llm   → Internal API layer
                               → On-premises / private cloud LLM
```

---

## How the Portal Packages a Skill on Request

When a user requests a package through the portal (future state):

1. **User selects** a work stream and capability from the portal UI.
2. **Portal reads** `catalog/index.json` to find matching assets.
3. **Portal reads** the `catalog/packages/<package-id>.json` manifest.
4. **Portal collects** all files listed in the `assets` array (from their respective `source_folder` paths).
5. **Portal delivers** the package in the format matching the chosen `delivery_mode`:
   - `copilot-chat`: Returns a formatted system prompt ready to paste.
   - `api-endpoint`: Calls the LLM API and returns the result.
   - `mcp-tool`: Returns the MCP tool definition JSON.
   - `copilot-studio`: Returns a connector definition YAML.
   - `in-house-llm`: Routes to the internal API endpoint.

### Today's Manual Equivalent

Until the portal is built, this process is manual:
1. Browse `catalog/packages/<workstream>-*-pack.json` to find the right package.
2. Open each file listed in the `assets` array.
3. Copy the skill definition from the `## Skill Definition` section.
4. Paste into your LLM interface of choice.

---

## Creating a New Package

1. **Identify** the assets that belong together for a cohesive capability.
2. **Copy** `templates/package-template.json` to `catalog/packages/<workstream>-<name>-pack.json`.
3. **Fill in** all required fields: `id`, `name`, `workstream`, `entry_asset`, `assets`, `delivery_modes`.
4. **For each asset**, specify the correct `source_folder` value (see the mapping table above).
5. **Update** `catalog/index.json` — add a `package_ids` field to each asset entry that belongs to this package.
6. **Open a PR** using the PR template.

---

## Catalog + Package Cross-Reference

The catalog `index.json` and package manifests cross-reference each other:

- Each asset in `catalog/index.json` has an optional `package_ids` array listing which packages include it.
- Each package manifest lists all assets with their `path` and `source_folder`.

This bidirectional reference means:
- Starting from a package → you know exactly which files to bundle.
- Starting from an asset → you know which packages it belongs to.

---

## Packaging Rules

- A package **must** have exactly one `entry_asset` (the primary skill/agent the user invokes).
- All `required: true` assets must be present for the package to function.
- `required: false` assets enhance the package but are optional (e.g., shared utility skills).
- A package may include assets from **multiple work streams** (e.g., `system-designers-design-pack` includes `arch-trade-off-analysis` from the `architects` workstream).
- Never include real credentials, endpoints, or PII in package manifests — use `{{PLACEHOLDER}}` variables.
