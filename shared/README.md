# Shared Assets

This folder contains AI skills, prompts, agents, workflows, and automations that are **useful across multiple work streams**.

## When to Use `shared/` vs. `workstreams/`

Put your asset here if:
- It is genuinely applicable to two or more work streams without modification.
- It provides a foundational capability that other work-stream-specific assets depend on.

Put your asset in the appropriate `workstreams/<name>/` folder if:
- It is primarily useful to one work stream, even if others might occasionally use it.

## Contents

| Folder | What Belongs Here |
|---|---|
| `prompts/` | Cross-cutting prompts (e.g., "summarize any document") |
| `skills/` | Foundational skills reused by multiple workstreams |
| `agents/` | General-purpose agents |
| `workflows/` | Generic multi-step processes |
| `automations/` | Utility automation scripts |
| `ideas/` | Ideas that span multiple work streams |

## Contributing

Follow the [CONTRIBUTING guide](../CONTRIBUTING.md) and place new files in the appropriate sub-folder. Use the templates in `../templates/` as your starting point.
