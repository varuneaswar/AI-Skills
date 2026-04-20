# Runtime Adapters

This document defines how a Markdown asset (with `delivery_modes`, `inputs`, and `outputs` in its front-matter and a `## Skill Definition` section in its body) is converted into a concrete runtime payload — for an **API endpoint**, an **MCP tool**, or a **Copilot Studio connector**.

This is the canonical contract for the runtime team. Asset Markdown files need **zero modifications** to support new delivery modes — adapters wrap the same source of truth.

---

## Table of Contents

- [The Canonical Skill Contract](#the-canonical-skill-contract)
- [Delivery Mode 1: Copilot Chat (current)](#delivery-mode-1-copilot-chat-current)
- [Delivery Mode 2: REST API Endpoint (future portal)](#delivery-mode-2-rest-api-endpoint-future-portal)
- [Delivery Mode 3: MCP Tool (future)](#delivery-mode-3-mcp-tool-future)
- [Delivery Mode 4: Copilot Studio Connector (future)](#delivery-mode-4-copilot-studio-connector-future)
- [Delivery Mode 5: In-House LLM API (future)](#delivery-mode-5-in-house-llm-api-future)
- [Switching Between Delivery Modes](#switching-between-delivery-modes)
- [Adapter Implementation Reference](#adapter-implementation-reference)

---

## The Canonical Skill Contract

Every skill, agent, and workflow asset is self-describing. The runtime adapter needs only three things from the Markdown file:

| Source | What to extract | How to use it |
|---|---|---|
| Front-matter: `inputs[]` | Parameter names, types, descriptions, required flag | Function signature / JSON schema for request validation |
| Front-matter: `outputs[]` | Output names, types, descriptions | Response shape / return type |
| Body: `## Skill Definition` section | The system prompt text (between the triple-backtick fence) | LLM `system` message |

**Extraction algorithm:**

```
1. Read the Markdown file.
2. Parse YAML front-matter (everything between the first --- and second ---).
3. Read `inputs`, `outputs`, and `delivery_modes` from the parsed YAML.
4. Find the `## Skill Definition` section in the body.
5. Extract the content of the first code block (``` ... ```) in that section.
   → This is the system_prompt.
6. The user_message is constructed from the inputs provided at invocation time.
```

**Example — source asset front-matter (skill):**

```yaml
id: dev-code-review-skill
inputs:
  - name: CODE
    type: string
    description: The code to review
    required: true
  - name: LANGUAGE
    type: string
    description: Programming language
    required: true
  - name: FOCUS
    type: string
    description: "security | performance | readability | all"
    required: false
outputs:
  - name: REVIEW_REPORT
    type: string
    description: Structured review findings
delivery_modes:
  - copilot-chat
  - api-endpoint
  - mcp-tool
  - copilot-studio
  - in-house-llm
```

**Extracted system_prompt:** The content of the ``` block under `## Skill Definition` in the Markdown body.

---

## Delivery Mode 1: Copilot Chat (current)

**How it works:** The user copies the system prompt from the `## Skill Definition` section, sets it as the LLM's system message, and provides their inputs as the user message.

**No adapter code required.** This is the manual "copy-paste" mode.

**User workflow:**

```
1. Open the skill file (e.g., workstreams/developers/skills/code-review.md).
2. Copy the text inside the ``` block under ## Skill Definition.
3. In GitHub Copilot Chat / ChatGPT / Claude:
   - Set the system prompt to the copied text.
   - Replace {{PLACEHOLDER}} values with your actual inputs.
4. Send your user message.
```

**GitHub Copilot Chat shortcut:**

```
@workspace #file:workstreams/developers/skills/code-review.md
Review this TypeScript code for security issues.
[paste code here]
```

---

## Delivery Mode 2: REST API Endpoint (future portal)

**How it works:** A portal backend reads the catalog, resolves the asset file, extracts the system prompt, substitutes user-provided input values, and calls the LLM API.

### Request Shape

```http
POST /api/v1/invoke
Content-Type: application/json
Authorization: Bearer <token>

{
  "asset_id": "dev-code-review-skill",
  "inputs": {
    "CODE": "function getUserById(id) { return db.query('SELECT * FROM users WHERE id=' + id); }",
    "LANGUAGE": "JavaScript",
    "FOCUS": "security"
  }
}
```

### Response Shape

```json
{
  "asset_id": "dev-code-review-skill",
  "status": "success",
  "outputs": {
    "REVIEW_REPORT": "## Summary\nCritical security issue found...\n\n## Critical\n- [LINE 1] SQL injection..."
  },
  "metadata": {
    "model": "gpt-4o",
    "tokens_used": 842,
    "latency_ms": 1240
  }
}
```

### Adapter Pseudocode

```python
def invoke_skill(asset_id: str, inputs: dict) -> dict:
    # 1. Resolve asset from catalog
    asset = catalog.get(asset_id)
    file_path = asset["path"]

    # 2. Extract system prompt from Markdown
    system_prompt = extract_skill_definition(file_path)

    # 3. Substitute placeholders
    for name, value in inputs.items():
        system_prompt = system_prompt.replace(f"{{{{{name}}}}}", value)

    # 4. Build user message from remaining inputs
    user_message = build_user_message(inputs, asset["inputs"])

    # 5. Call LLM
    response = llm_client.chat(
        model=asset.get("llm_compatibility", ["gpt-4o"])[0],
        system=system_prompt,
        user=user_message,
        temperature=0.1
    )

    # 6. Return structured output
    return { "asset_id": asset_id, "outputs": { "REVIEW_REPORT": response.content } }
```

### Input Validation

Before calling the LLM, validate inputs against the asset's `inputs[]` array:

```python
def validate_inputs(asset_inputs: list, provided: dict) -> list[str]:
    errors = []
    for field in asset_inputs:
        if field.get("required") and field["name"] not in provided:
            errors.append(f"Missing required input: {field['name']}")
    return errors
```

---

## Delivery Mode 3: MCP Tool (future)

**How it works:** Each `active` skill in the catalog is registered as a named MCP tool. The MCP server reads the catalog on startup and auto-generates tool definitions from asset front-matter.

### MCP Tool Definition (auto-generated from front-matter)

```json
{
  "name": "dev-code-review-skill",
  "description": "Reviews a code snippet or file for correctness, maintainability, security issues, and adherence to best practices, returning findings categorised by severity.",
  "input_schema": {
    "type": "object",
    "required": ["CODE", "LANGUAGE"],
    "properties": {
      "CODE": {
        "type": "string",
        "description": "The code snippet or file content to review"
      },
      "LANGUAGE": {
        "type": "string",
        "description": "Programming language (e.g., Python, TypeScript, Java)"
      },
      "FOCUS": {
        "type": "string",
        "description": "Optional focus area: security, performance, readability, or all"
      }
    }
  }
}
```

### MCP Tool Invocation Payload

```json
{
  "tool": "dev-code-review-skill",
  "parameters": {
    "CODE": "function getUserById(id) { ... }",
    "LANGUAGE": "JavaScript",
    "FOCUS": "security"
  }
}
```

### MCP Server Adapter (Node.js sketch)

```javascript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { catalog } from "./catalog.js";
import { extractSkillDefinition } from "./extractor.js";

const server = new Server({ name: "ai-skills", version: "1.0.0" }, {
  capabilities: { tools: {} }
});

// Register all active skills as MCP tools
server.setRequestHandler("tools/list", async () => ({
  tools: catalog.assets
    .filter(a => a.status === "active" && a.delivery_modes?.includes("mcp-tool"))
    .map(asset => ({
      name: asset.id,
      description: asset.description,
      inputSchema: buildInputSchema(asset.inputs)
    }))
}));

server.setRequestHandler("tools/call", async (request) => {
  const asset = catalog.assets.find(a => a.id === request.params.name);
  const systemPrompt = extractSkillDefinition(asset.path);
  const result = await callLLM(systemPrompt, request.params.arguments);
  return { content: [{ type: "text", text: result }] };
});
```

---

## Delivery Mode 4: Copilot Studio Connector (future)

**How it works:** Each package manifest becomes a Copilot Studio connector. The connector YAML is auto-generated from the package manifest and the asset front-matter it references.

### Connector YAML (auto-generated from package manifest)

```yaml
# Auto-generated from catalog/packages/developers-core-pack.json
# Do not edit manually — regenerate using: ai-skills generate-connector developers-core-pack

openapi: "3.0.0"
info:
  title: Developers Core Pack
  version: "1.0.0"
  description: Core AI skills for the developers work stream

paths:
  /invoke/dev-code-review-skill:
    post:
      operationId: devCodeReviewSkill
      summary: Structured Code Review
      description: Reviews code for correctness, security, performance, and maintainability.
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [CODE, LANGUAGE]
              properties:
                CODE:
                  type: string
                  description: The code to review
                LANGUAGE:
                  type: string
                  description: Programming language
                FOCUS:
                  type: string
                  description: "security | performance | readability | all"
      responses:
        "200":
          description: Review report
          content:
            application/json:
              schema:
                type: object
                properties:
                  REVIEW_REPORT:
                    type: string
```

### Mapping: Package Manifest → Connector

| Package Manifest Field | Connector Field |
|---|---|
| `id` | `info.title` base |
| `description` | `info.description` |
| `assets[].path` | Resolved to get inputs/outputs from front-matter |
| Asset `inputs[]` | `requestBody.content.schema.properties` |
| Asset `outputs[]` | `responses.200.content.schema.properties` |
| `delivery_modes: [copilot-studio]` | Asset included in connector |

---

## Delivery Mode 5: In-House LLM API (future)

**How it works:** Identical to Delivery Mode 2 (REST API), but the LLM endpoint is an internal proxy that routes traffic to a self-hosted or VNet-peered model. No data leaves the corporate network.

### Configuration Difference

```python
# Delivery Mode 2 (external)
llm_client = OpenAIClient(api_key=os.environ["OPENAI_API_KEY"], base_url="https://api.openai.com/v1")

# Delivery Mode 5 (in-house)
llm_client = OpenAIClient(api_key=os.environ["INTERNAL_LLM_KEY"], base_url=os.environ["INTERNAL_LLM_ENDPOINT"])
```

The adapter code, asset files, and catalog are **identical**. Only the LLM client configuration changes.

---

## Switching Between Delivery Modes

The switching cost between delivery modes is **near zero** because:

1. Asset Markdown files are format-agnostic — the system prompt lives in the `## Skill Definition` section regardless of delivery mode.
2. The `delivery_modes` field in front-matter declares intent — the adapter layer reads it to decide which endpoint to register.
3. Input/output schemas in front-matter map directly to function signatures, JSON schemas, and OpenAPI paths.

### Migration path

| Phase | Action |
|---|---|
| **Today** | Declare `delivery_modes: [copilot-chat]` on all new assets. Copy-paste usage works immediately. |
| **Portal phase** | Enable `api-endpoint` in front-matter for assets to be served via REST. Run the adapter server. |
| **MCP phase** | Enable `mcp-tool` in front-matter. Run the MCP server — it auto-generates tool definitions. |
| **Copilot Studio** | Enable `copilot-studio`. Run the connector generator against each package manifest. |
| **In-house LLM** | Enable `in-house-llm`. Point the API adapter at the internal endpoint. |

---

## Adapter Implementation Reference

### `extractSkillDefinition(filePath)` — reference implementation

```javascript
import { readFileSync } from "fs";

/**
 * Extracts the system prompt from the ## Skill Definition section of a Markdown asset.
 * @param {string} filePath - Path to the Markdown file
 * @returns {string} - System prompt text (content of the first ``` block in the section)
 */
export function extractSkillDefinition(filePath) {
  const content = readFileSync(filePath, "utf8");
  const sectionMatch = content.match(/## Skill Definition\s*\n([\s\S]*?)(?=\n## |\n---|\s*$)/);
  if (!sectionMatch) throw new Error(`No ## Skill Definition section found in ${filePath}`);
  const codeBlockMatch = sectionMatch[1].match(/```[^\n]*\n([\s\S]*?)```/);
  if (!codeBlockMatch) throw new Error(`No code block found in ## Skill Definition in ${filePath}`);
  return codeBlockMatch[1].trim();
}
```

### `buildInputSchema(inputs)` — MCP schema builder

```javascript
/**
 * Converts asset front-matter inputs[] to a JSON Schema object (for MCP tool registration).
 */
export function buildInputSchema(inputs = []) {
  const properties = {};
  const required = [];
  for (const input of inputs) {
    properties[input.name] = { type: input.type || "string", description: input.description || "" };
    if (input.required) required.push(input.name);
  }
  return { type: "object", properties, required };
}
```

### `parseFrontMatter(filePath)` — YAML extractor

```javascript
import { readFileSync } from "fs";
import yaml from "js-yaml";

export function parseFrontMatter(filePath) {
  const content = readFileSync(filePath, "utf8");
  const match = content.match(/^---\n([\s\S]*?)\n---/);
  if (!match) return null;
  return yaml.load(match[1]);
}
```
