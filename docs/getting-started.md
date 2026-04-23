# Getting Started with AI-Skills

This guide helps you browse, use, and contribute to the AI-Skills repository in under 10 minutes.

---

## Prerequisites

- A GitHub account with access to this repository.
- Basic familiarity with Markdown.
- (Optional) A local Git client if you want to contribute.

---

## 1. Find Existing Content

### Browse by Work Stream

Navigate to `workstreams/<your-work-stream>/` and explore the sub-folders:

- `prompts/` — ready-to-use prompt templates.
- `skills/` — focused AI capabilities.
- `agents/` — autonomous agents.
- `workflows/` — multi-step orchestrations.
- `automations/` — scripts and pipelines.
- `ideas/` — proposals in progress.

### Browse the Catalog

Open `catalog/index.json` for a machine-readable list of every asset with key metadata.

### Search by Tag

Use the repository search to filter by tags:

```
repo:varuneaswar/AI-Skills "tags:" "tag1"
```

---

## 2. Use a Prompt

1. Open the Markdown file for the prompt.
2. Copy the content between the `## Prompt` heading (or the `prompt_text` field in the front matter).
3. Replace any `{{PLACEHOLDER}}` variables with your actual values.
4. Paste into your IDE's Copilot Chat, ChatGPT, Claude, or other LLM interface.

---

## 3. Use a Skill with an LLM

1. Find the skill file in `workstreams/<work-stream>/skills/` or `shared/skills/`.
2. Follow the instructions in the skill's **Usage** section.
3. Many skills provide an example Copilot Chat command you can copy directly.

---

## 4. Contribute a New Asset

See [CONTRIBUTING.md](../CONTRIBUTING.md) for the full guide. The short version:

```bash
# 1. Create a branch
git checkout -b feat/developers/prompts/my-new-prompt

# 2. Copy the relevant template
cp templates/prompt-template.md workstreams/developers/prompts/my-new-prompt.md

# 3. Edit the file, fill in all front matter fields and body sections

# 4. Update the catalog
# Edit catalog/index.json and add an entry for your new asset

# 5. Commit and push
git add .
git commit -m "feat(developers/prompts): add my-new-prompt"
git push origin feat/developers/prompts/my-new-prompt

# 6. Open a pull request
```

---

## 5. Propose an Idea

Not ready to write a full asset? Open a Jira ticket using the **Idea Submission** template. A maintainer or community member may pick it up, or you can return to it later.

---

## 6. Get Help

- Open a GitHub Discussion for questions.
- Open a Jira ticket with the `question` label for specific repository problems.
- Tag a maintainer in your PR if you are stuck during review.
