# Lead Generation Engine — Claude Operating Manual

## Purpose

This repo is a workspace for designing, validating, and deploying **n8n workflows** with Claude as the builder. The inaugural workflow is the **Lead Generation Engine** (webhook → enrich → CRM → Slack). Additional workflows may follow.

Two toolchains are wired up and **must be used** for every workflow build:

1. **n8n-mcp server** — programmatic access to the user's n8n cloud instance. Config: [`.mcp.json`](.mcp.json). Exposes node search, schema fetching, validation, and workflow create/update/activate/deploy-template tools.
2. **n8n skills** — domain skills installed at `~/.claude/skills/`:
   - `n8n-mcp-tools-expert` — authoritative reference for every n8n-mcp tool
   - `n8n-workflow-patterns` — the 5 core architectural patterns (webhook / HTTP API / database / AI agent / scheduled)
   - `n8n-node-configuration` — operation-specific node config
   - `n8n-expression-syntax` — `{{ }}` expressions and data access rules
   - `n8n-code-javascript` / `n8n-code-python` — writing Code nodes
   - `n8n-validation-expert` — interpreting and fixing validation errors

Do not hand-write workflow JSON from memory. Node type strings, parameter names, and schemas drift between n8n versions — always pull from MCP.

---

## Reference architecture: Lead Generation Engine

Pattern: **Webhook Processing** (see `n8n-workflow-patterns` → `webhook_processing.md`).

```
Webhook → Validate → Enrich → Dedupe-lookup → CRM upsert → Slack notify
                                              ↓
                                           Error branches → Error workflow / collection node
```

### Stage 1 — Webhook (trigger)
- Form provider: Typeform / Tally / website contact form (user picks per deployment).
- Payload shape (minimum): `name`, `email`, `company`.
- **Gotcha**: webhook payload is nested under `$json.body`. Use `{{$json.body.email}}`, not `{{$json.email}}`.
- Configure with `responseMode: "onReceived"` for fire-and-forget, or `"lastNode"` if the form needs a custom response.

### Stage 2 — Enrich
- Primary: **Clearbit** (`/v2/people/find?email=…`). Fallback: **Apollo** (`/api/v1/people/match`).
- Expected fields: title, LinkedIn URL, company revenue, headcount, industry, tech stack, funding history.
- On 404/empty result, route to Slack notify with a "thin" lead rather than dropping it.

### Stage 3 — CRM write (dedupe → upsert)
- Target: **HubSpot** / **Pipedrive** / **Salesforce** (user picks one per deployment — do not assume).
- Flow: lookup-by-email → if exists, update; if not, create. This survives form re-submits without creating duplicates.

### Stage 4 — Notify
- Slack post to the sales channel. Suggested template:
  > New lead: *{name}*, *{title}* at *{company}* ({headcount} employees, {revenue} revenue, uses {tech_stack}). Added to {CRM} → *{crm_link}*.

---

## The non-negotiable build loop

Follow this order for every workflow. No shortcuts.

### 1. Invoke the right skill first
Before writing any MCP call or workflow JSON, invoke the relevant skill so its guidance is loaded:

| Situation | Skill |
|---|---|
| Designing a workflow's shape | `n8n-workflow-patterns` |
| About to call any n8n-mcp tool | `n8n-mcp-tools-expert` |
| Configuring a node | `n8n-node-configuration` |
| Writing `{{ }}` expressions | `n8n-expression-syntax` |
| Writing a Code node | `n8n-code-javascript` / `n8n-code-python` |
| Interpreting a validation error | `n8n-validation-expert` |

### 2. Discover nodes
```
search_nodes({ query: "slack", limit: 20 })
```
Returns both `nodeType` (for search/validate) and `workflowNodeType` (for create/update). Do not guess node types from memory.

### 3. Pull real schemas
```
get_node({ nodeType: "nodes-base.slack" })                 // detail="standard" (default, ~1-2K tokens)
get_node({ nodeType: "nodes-base.slack", mode: "docs" })   // readable markdown
get_node({ nodeType: "nodes-base.httpRequest", mode: "search_properties", propertyQuery: "auth" })
```
Only use `detail: "full"` when debugging deeply nested schemas — it's 3–8K tokens.

### 4. Validate every node, then the whole workflow
```
validate_node({ nodeType: "nodes-base.slack", config: {...}, profile: "runtime" })
validate_workflow({ ... })
```
Validation profiles (pick explicitly — don't rely on the default):
- `minimal` — very lenient (not enough for deploy)
- `runtime` — **recommended** for pre-deployment
- `ai-friendly` — balanced for AI workflows
- `strict` — maximum (production / compliance-sensitive)

Iterate: validate → fix → validate until clean. If stuck on a false positive, consult `n8n-validation-expert`.

### 5. Create / update / activate via MCP
```
n8n_create_workflow({ name, nodes, connections })
n8n_update_partial_workflow({ id, intent: "...", operations: [...] })   // prefer this for edits
n8n_update_partial_workflow({ id, operations: [{ type: "activateWorkflow" }] })
```
- Always include `intent: "..."` on partial updates — it improves tool responses and leaves a useful audit trail.
- Use **smart parameters** on IF/Switch connections: `branch: "true" | "false"`, `case: 0`. Don't compute `sourceIndex` by hand.
- Workflows are **built iteratively**, not one-shot. Auto-sanitization runs on every update (fixes operator structures automatically; cannot fix broken connections).

### nodeType format — the #1 gotcha
Two formats, one tool-category each:
- **Search / validate** tools: `nodes-base.slack`, `nodes-langchain.agent`
- **Workflow** tools (create/update): `n8n-nodes-base.slack`, `@n8n/n8n-nodes-langchain.agent`

`search_nodes` returns both. Use the right one for the right tool.

---

## Credentials and secrets

- **Never** hardcode API keys, tokens, or OAuth secrets in workflow JSON. Reference credentials by ID/name as configured in the n8n instance.
- Credentials expected for the Lead Generation Engine:
  - Clearbit API key (or Apollo API key)
  - HubSpot / Pipedrive / Salesforce (OAuth or API key, per the CRM chosen)
  - Slack bot token OR incoming webhook URL
- If a required credential is missing in n8n, **ask the user** — do not invent placeholders or commit fake values.
- `.mcp.json` itself contains an n8n API key. Keep it out of version control or rotate the key if it gets pushed. The project's git repo should `.gitignore` it.

---

## Quality bar

A workflow is not "done" until:

- [ ] `validate_workflow` returns clean under `profile: "runtime"`
- [ ] Every external API call (Clearbit / Apollo / CRM / Slack) has an error branch — a dropped lead is a failure mode, not an edge case
- [ ] CRM write dedupes by email (lookup → update-or-create) to survive form re-submits
- [ ] Unhandled cases route to an **Error Trigger** workflow or a collection node/data table so nothing is silently lost
- [ ] `n8n_test_workflow` or a real webhook fire has been run end-to-end, and the execution export or screenshot is saved under `evidence/`
- [ ] Workflow is activated via `activateWorkflow` operation (not left in draft)

---

## Repo layout conventions

```
workflows/   exported workflow JSON, one file per workflow, kebab-case slug
             (e.g. lead-generation-engine.json)
docs/        per-workflow notes — payload shapes, field mappings, Slack templates
evidence/    screenshots and execution exports proving workflows fire end-to-end
.mcp.json    n8n-mcp server config (contains N8N_API_KEY — keep out of git)
```

Create folders lazily — only when the first file lands in them.

---

## What Claude should NOT do

- Write workflow JSON from memory without MCP verification.
- Skip validation to save time.
- Use `detail: "full"` on `get_node` by default — wastes tokens.
- Guess nodeType prefixes (it's `nodes-base.*` for search/validate, `n8n-nodes-base.*` for workflow tools).
- Build a workflow in one shot — iterate via `n8n_update_partial_workflow`.
- Pick the CRM, enrichment provider, or form provider unilaterally — ask.
- Hardcode secrets anywhere, even "temporarily."
- Create README, sample data, or auxiliary files unless asked.

---

## When in doubt

Ask. A 10-second clarification beats a workflow that misroutes real leads.
For tool-surface questions, defer to `n8n-mcp-tools-expert` — it's the single source of truth for tool names, parameters, and performance characteristics.
