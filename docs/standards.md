# Naming, Metadata, and Quality Standards

This document is the definitive reference for standards that all assets in the AI-Skills repository must follow.

---

## 1. File Naming

| Rule | Correct | Incorrect |
|---|---|---|
| Lowercase kebab-case | `summarize-pr.md` | `SummarizePR.md`, `summarize_pr.md` |
| Descriptive — conveys what the asset does | `generate-test-cases.md` | `prompt1.md` |
| No type suffix inside a type folder | `incident-triage.md` (inside `prompts/`) | `incident-triage-prompt.md` |
| No spaces | `code-review-checklist.md` | `code review checklist.md` |

---

## 2. Front-Matter Standards

All assets must start with a YAML front-matter block (between `---` delimiters).

### Required Fields

| Field | Type | Rules |
|---|---|---|
| `id` | string | Globally unique across the repo. Kebab-case. Example: `sre-incident-triage-v1` |
| `title` | string | Title-case, human-readable. Max 80 characters. |
| `type` | enum | One of: `idea`, `prompt`, `skill`, `agent`, `workflow`, `automation` |
| `version` | string | SemVer string, e.g., `"1.0.0"` |
| `status` | enum | One of: `draft`, `active`, `deprecated` |
| `workstream` | string[] | List of applicable work streams. Use `shared` for cross-cutting content. |
| `author` | string | GitHub username of the original author. |
| `created` | date | ISO 8601 date, `YYYY-MM-DD`. |
| `updated` | date | ISO 8601 date, `YYYY-MM-DD`. Must be updated on every change. |
| `tags` | string[] | At least one tag. Lowercase. No spaces (use hyphens). |
| `llm_compatibility` | string[] | At least one entry. See approved values below. |
| `description` | string | One or two sentences. Plain text (no Markdown). |

### Approved `llm_compatibility` Values

- `gpt-4o`
- `gpt-4-turbo`
- `gpt-4`
- `gpt-3.5-turbo`
- `claude-3-5-sonnet`
- `claude-3-opus`
- `claude-3-haiku`
- `gemini-1.5-pro`
- `gemini-1.5-flash`
- `azure-openai-gpt-4o`
- `copilot-gpt-4o`
- `all` *(use only for simple, model-agnostic prompts)*

### Optional but Recommended Fields

| Field | Type | Description |
|---|---|---|
| `security_classification` | enum | `public`, `internal` (default), or `confidential` |
| `dependencies` | string[] | IDs of other assets this one depends on |
| `inputs` | object[] | Parameter list: `name`, `type`, `description`, `required` |
| `outputs` | object[] | Output list: `name`, `type`, `description` |
| `examples` | object[] | Sample `input` → `output` pairs |
| `security_notes` | string | Required for `confidential` assets |

---

## 3. Document Body Standards

### Section Order

Every asset file should contain sections in this order (omit sections that do not apply):

1. `## Overview` — expands on the front-matter description.
2. `## Inputs` — if the asset accepts parameters.
3. `## Prompt` / `## Skill Definition` / `## Agent Configuration` — the core content.
4. `## Usage` — how to use the asset; copy-paste instructions.
5. `## Examples` — at least one worked example.
6. `## Testing Notes` — which models were tested and any known limitations.
7. `## Changelog` — version history.

### Placeholders

Use `{{UPPER_SNAKE_CASE}}` for variables that the user must supply:

```
Summarize the following pull request for a {{AUDIENCE}} audience:

{{PR_DESCRIPTION}}
```

Document every placeholder in the `## Inputs` section.

---

## 4. Catalog Entry Standards

Every new asset must add an entry to `catalog/index.json`. The entry must include:

```json
{
  "id": "string",
  "title": "string",
  "type": "prompt | skill | agent | workflow | automation | idea",
  "status": "draft | active | deprecated",
  "workstream": ["string"],
  "tags": ["string"],
  "description": "string",
  "path": "relative/path/to/file.md",
  "updated": "YYYY-MM-DD"
}
```

---

## 5. Version Increment Rules

| Change | Version bump |
|---|---|
| Typo fix, wording improvement | PATCH (1.0.0 → 1.0.1) |
| New optional input, new example | MINOR (1.0.0 → 1.1.0) |
| Changed required inputs, different output format | MAJOR (1.0.0 → 2.0.0) |

Always add a changelog entry:

```markdown
## Changelog

### 1.1.0 — 2024-09-15
- Added `AUDIENCE` input parameter.

### 1.0.0 — 2024-08-01
- Initial version.
```

---

## 6. Quality Checklist

Before opening a PR, confirm:

- [ ] File name follows kebab-case and lives in the correct folder.
- [ ] All required front-matter fields are present and valid.
- [ ] No secrets, tokens, PII, or real internal endpoints.
- [ ] At least one `## Examples` section with a real input/output.
- [ ] `llm_compatibility` lists only models that were actually tested.
- [ ] `catalog/index.json` has been updated.
- [ ] `updated` date reflects today's date.
