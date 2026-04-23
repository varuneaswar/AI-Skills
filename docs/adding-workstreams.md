# Adding a New Work Stream

This guide explains how to register a new work stream in the AI-Skills repository. Adding a work stream is designed to be a self-service, low-friction process.

---

## Overview

A **work stream** is a team or discipline namespace — a first-class folder that owns its own skills, prompts, agents, workflows, automations, and ideas. The design goal is that any team can onboard by following this guide, without changing any shared infrastructure files beyond the two registry steps below.

---

## Step-by-Step Guide

### Step 1 — Register in the Workstream Registry

Open `workstreams/registry.json` and add a new entry to the `workstreams` array:

```json
{
  "id": "data-engineers",
  "label": "Data Engineers",
  "description": "Engineers responsible for data pipelines, data quality, and analytics infrastructure.",
  "path": "workstreams/data-engineers",
  "owner": "your-username-or-team",
  "tags": ["data", "pipelines", "analytics"],
  "status": "active"
}
```

**Key rules:**
- `id` must be kebab-case, lowercase, unique across all work streams.
- `path` must match the folder you will create in Step 2.
- Set `status: "planned"` while the folder is empty, switch to `"active"` once the first asset is merged.

### Step 2 — Create the Folder Structure

Run the following commands from the repository root (or create the folders manually):

```bash
WORKSTREAM=data-engineers

mkdir -p workstreams/${WORKSTREAM}/{skills,prompts,agents,workflows,automations,ideas}
```

### Step 3 — Add a Work Stream README

Copy the README from another work stream and adapt it:

```bash
cp workstreams/developers/README.md workstreams/${WORKSTREAM}/README.md
```

Update the README with:
- Work stream name and description
- Team / point of contact
- Links to first assets (once added)
- Any domain-specific notes or conventions

### Step 4 — Add Your First Asset

Use a template from the `templates/` directory:

```bash
cp templates/skill-template.md workstreams/${WORKSTREAM}/skills/my-first-skill.md
```

Fill in the front matter (see `docs/standards.md` for field definitions). The key fields for your new work stream:

```yaml
workstream:
  - data-engineers   # Must match the id you registered in step 1
```

### Step 5 — Register the Asset in the Catalog

Open `catalog/index.json` and add an entry for your new asset. Example:

```json
{
  "id": "data-pipeline-health-check",
  "title": "Data Pipeline Health Check",
  "type": "skill",
  "status": "active",
  "workstream": ["data-engineers"],
  "tags": ["data", "pipelines", "monitoring"],
  "description": "Analyses pipeline run logs to identify failures, backpressure, and data quality issues.",
  "path": "workstreams/data-engineers/skills/pipeline-health-check.md",
  "updated": "YYYY-MM-DD",
  "delivery_modes": ["llm-chat", "api-endpoint"]
}
```

### Step 6 — Create a Package Manifest (optional but recommended)

Copy the package template and create your first package:

```bash
cp templates/package-template.json catalog/packages/data-engineers-core-pack.json
```

Fill in the `assets` array with all the files your package includes (see `docs/packaging.md` for the full guide).

### Step 7 — Open a Pull Request

Use the PR template (`.github/PULL_REQUEST_TEMPLATE.md`). In the PR description, link to:
- The registry entry you added in Step 1
- Your first asset file(s)
- The package manifest (if created)

---

## Schema Compatibility

The `workstream` field in all asset schemas and in `catalog/index.json` intentionally **does not use an enum constraint** — any string value is accepted. This means:

- You do **not** need to update any schema files to add a new work stream.
- The `workstreams/registry.json` is the single source of truth for the valid list.
- A future CI validation script will cross-check asset `workstream` values against the registry.

---

## Extensibility Design Principles

| Principle | How It Is Implemented |
|---|---|
| **No shared file changes required** | Schema enums are open; registry is append-only |
| **Self-service** | Templates + this guide are sufficient to onboard |
| **Discoverable** | Registry is read by the portal to populate workstream filters |
| **Reversible** | Set `status: "deprecated"` in the registry to retire a workstream without deleting files |
| **Cross-cutting support** | Assets can declare multiple workstreams; `shared/` folder exists for truly cross-cutting content |

---

## Retiring a Work Stream

To retire a work stream:

1. Set `"status": "deprecated"` in `workstreams/registry.json` for the work stream entry.
2. Update each active asset in that work stream to `status: deprecated` in their front matter.
3. Update `catalog/index.json` entries to `"status": "deprecated"`.
4. Do **not** delete any files — deprecated assets remain for historical reference.
5. Open a PR documenting the reason for retirement.

---

## FAQ

**Q: Can a work stream have the same id as an existing one?**  
No. IDs must be globally unique. Check `workstreams/registry.json` before choosing an id.

**Q: Can an asset belong to multiple work streams?**  
Yes. The `workstream` field is an array. An asset like `arch-trade-off-analysis` is tagged `["architects", "system-designers"]` because both teams use it.

**Q: When should something go in `shared/` instead of a work stream folder?**  
Use `shared/` only when an asset is genuinely reused by three or more work streams and has no single owner. Otherwise, place it in the work stream that created it and tag the other workstreams in the `workstream` array.

**Q: Do I need to update any Bitbucket Pipelines CI checks?**  
No. The current CI check validates front matter format, not the list of valid workstream names. If a strict validation check is added in future, it will read from `workstreams/registry.json` automatically.
