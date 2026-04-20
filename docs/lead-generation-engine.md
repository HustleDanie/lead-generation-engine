# Lead Generation Engine — Operator Reference

## What this workflow does

A 3-stage lead pipeline in n8n cloud (`hustledaniel.app.n8n.cloud`) that takes form submissions through **capture → enrich → CRM → score → notify**, with a duplicate gate before enrichment (to save API spend) and score-based routing to three destinations plus a unified activity log.

The workflow is built in **mock mode** — all external services (Clearbit, Apollo, Slack, Gmail, HubSpot) are replaced by Code / Set nodes so the demo runs zero-credential. Swap-in points for real providers are called out below.

| | |
|---|---|
| Main workflow ID | `lPgbSzFg5XHLZUxs` |
| Error handler workflow ID | `dKh8YrXQS07DPbnQ` |
| Status | Both active |
| Validation | `validate_workflow` profile `runtime` — 0 errors, 21 advisory warnings |

## Architecture

```
━━━ ① LEAD CAPTURE ━━━
New Lead Form (Form Trigger, responseMode=onReceived)
   ↓
Validate Email (IF: email notEmpty AND matches regex)
   ↓ true
Normalize Lead (Set: lowercase+trim email, map field-0..4 → name/email/…)
   ↓ (fan-out to 2 dedupe nodes)
Dedupe — New Lead?  (DataTable rowNotExists on lead_contacts.email)
Dedupe — Duplicate? (DataTable rowExists    on lead_contacts.email)
        ↓                                   ↓
     (new path)                          Log Duplicate → end

━━━ ② CRM ENRICHMENT ━━━
Clearbit Mock (Code: returns full profile if domain ∈ enriched list)
   ↓
Clearbit Hit? (IF enriched_by != null)
   ├─ true  → Merge Enrichment (input 0)
   └─ false → Apollo Fallback Mock (Code) → Merge Enrichment (input 1)
                          ↓
        Merge Enrichment (append — unifies both branches)
                          ↓
        Score Lead (Rule Engine) — Code node, returns { tier, reasoning, score_num }
                          ↓
        Insert Contact (DataTable insert → lead_contacts)

━━━ ③ TEAM NOTIFICATION ━━━
Route by Tier (Switch on $json.lead_tier)
   ├─ "hot"  → Slack SDR Alert (Mock)  → Log Hot Activity
   ├─ "warm" → Email Digest (Mock)     → Log Warm Activity
   └─ "cold" → Nurture Queue (Mock)    → Log Cold Activity
```

Error workflow (separate): `Error Trigger → Format Error (Set) → Log Failure (DataTable → lead_failed_leads)`. Wired in main via `settings.errorWorkflow`.

## Persistence — n8n Data Tables

Three tables replace HubSpot / Google Sheets for the mock demo.

### `lead_contacts` (id `Xr0poCxlcDKgwZjg`) — CRM substitute

| Column | Type | Purpose |
|---|---|---|
| email | string | Unique business key used for dedupe |
| name | string | Full name from form |
| company | string | Company from form |
| job_title | string | Enriched title (falls back to form input) |
| enrichment_source | string | `"clearbit"` / `"apollo"` / `"none"` |
| lead_tier | string | `"hot"` / `"warm"` / `"cold"` |
| lead_score | number | 0–100 |
| scoring_reasoning | string | Human-readable why |
| nurture_sequence | boolean | `true` for cold leads |
| created_at | date | ISO timestamp |

### `lead_activity_log` (id `HcJfpPtUqnMszQ9c`) — one row per lead processed (including duplicates)

| Column | Type | Notes |
|---|---|---|
| timestamp | date | |
| email, name, company | string | Denormalized for quick reporting |
| tier | string | `hot` / `warm` / `cold` / `n/a` (duplicates) |
| score | number | Same as `lead_score`, or 0 for duplicates |
| status | string | `notified` / `duplicate` |
| source | string | Enrichment provider or `dedupe` |
| notification_channel | string | e.g. `slack:#sales-hot-leads`, `none` for duplicates |

### `lead_failed_leads` (id `mTzRsyI7awVGmNGd`) — error sink

| Column | Type |
|---|---|
| timestamp | date |
| email | string |
| stage | string (node name where the error occurred) |
| error_message | string |
| raw_payload | string (JSON-stringified input) |

## Scoring rubric (rule engine)

Implemented in `Score Lead (Rule Engine)` as a Code node (deterministic, replayable, no OpenAI credential required).

```
title matches /chief|cto|ceo|cfo|coo|cpo|cmo|vp|vice president|director|head of/  → "exec"
title matches /manager|lead|principal|senior/                                     → "mgr"

enrichment_source == "none"              → tier=cold,  score=10,  reason="No enrichment data"
exec AND (size ≥ 50 OR revenue ≥ $10M)   → tier=hot,   score=90,  reason="Executive title at {size}-employee company with ${revenue}"
exec OR mgr OR size ≥ 10                 → tier=warm,  score=55,  reason="Mid-level profile (title={t}, size={s})"
else                                     → tier=cold,  score=25,  reason="Below threshold"
```

To swap in an AI scorer: replace the Code node with an `@n8n/n8n-nodes-langchain.agent` node + OpenAI chat model + Structured Output Parser enforcing `{ tier, reasoning, score_num }`.

## Mock enrichment — domain allowlists

**Clearbit Mock** hits (returns full profile, size=250, revenue=$50M):
`stripe.com`, `airbnb.com`, `shopify.com`, `notion.so`, `linear.app`, `vercel.com`, `anthropic.com`

**Apollo Fallback Mock** hits (returns lighter profile, size=45, revenue=$5M):
`google.com`, `meta.com`, `microsoft.com`, `example.com`, `acme.io`, `techco.com`

Any other domain misses both — lead falls through to `cold` tier with `enrichment_source="none"`.

## Notification templates

| Tier | Channel | Message template |
|---|---|---|
| hot | `slack:#sales-hot-leads` | `:fire: New HOT lead: *{{name}}* ({{job_title}}) at *{{company}}* — score {{lead_score}}. {{scoring_reasoning}}` |
| warm | `gmail:sales-manager@example.com` | `[Warm Lead Digest] {{name}} — {{company}} — {{job_title}} — score {{lead_score}}` |
| cold | `hubspot:nurture_sequence` | `Added to nurture sequence: {{name}} at {{company}}` |

Each notification node is a Set that captures the message and channel; swap to HTTP Request (Slack webhook), Gmail OAuth2, or HubSpot contact update when going live.

## Form field mapping

n8n Form Trigger v2.5 auto-assigns positional keys (`field-0`, `field-1`, …) and stores labels for UI rendering. The public form shows:

| Label (shown to user) | Positional key (in payload) | Normalized key |
|---|---|---|
| Full Name | `field-0` | `name` |
| Work Email | `field-1` | `email` |
| Company | `field-2` | `company` |
| Job Title | `field-3` | `jobTitle` |
| What brings you here? | `field-4` | `message` |

The Normalize Lead node maps positional → semantic keys for everything downstream.

## Gotcha log (for future edits)

1. **Form Trigger v2.5 `responseMode`** — only `onReceived` or `lastNode`; `responseNode` (with a Respond to Webhook node) requires v2.2 or earlier. This workflow uses `onReceived` (fire-and-forget).
2. **Dedupe pattern** — instead of `dataTable.get` + IF on empty result, use **two parallel DataTable nodes**: `rowExists` → duplicate path, `rowNotExists` → new-lead path. Cleaner and avoids IF-on-empty edge cases.
3. **DataTable `insert` returns the row shape, not the upstream shape** — column names replace the expression names. After `Insert Contact`, `$json.tier` becomes `$json.lead_tier`, `$json.score_num` becomes `$json.lead_score`, etc. All downstream nodes (Switch, Set notifiers, activity log) must reference the table column names.
4. **Switch v3.4 outputs named in `outputKey`** — useful for labelling branches in the UI, doesn't affect routing (which uses numeric index `case: 0|1|2`).
5. **Error workflow expressions must guard null chains** — `$json.execution?.error?.message` triggers an n8n validator warning; use `$json.execution && $json.execution.error && $json.execution.error.message ? … : 'unknown'` instead.

## Verification results

| Check | Status |
|---|---|
| `validate_workflow` (runtime) on both workflows | ✅ 0 errors |
| Hot lead path — Slack alert fired with correct score/reasoning | ✅ execution 5 |
| Warm lead path — Apollo fallback → email digest | ✅ execution 6 |
| Cold lead path — double-miss → nurture queue | ✅ execution 7 |
| Duplicate path — short-circuits before enrichment | ✅ execution 8 (110ms, Log Duplicate only) |
| Activity log captures every lead including duplicates | ✅ `lead_activity_log` has rows for all 4 tiers |
| Both workflows `active: true` | ✅ |
| No secrets in committed files | ✅ `.mcp.json` stays `.gitignore`d |

Raw execution exports in [evidence/](../evidence/).

## Going live — swap points

| Mock node | Real replacement |
|---|---|
| `Clearbit Mock` (Code) | HTTP Request → `GET person.clearbit.com/v2/people/find?email=…` + Header Auth credential |
| `Apollo Fallback Mock` (Code) | HTTP Request → `POST api.apollo.io/api/v1/people/match` + Header Auth credential |
| `Score Lead (Rule Engine)` (Code) | `@n8n/n8n-nodes-langchain.agent` + OpenAI chat model + Structured Output Parser |
| `Insert Contact` (DataTable) | `nodes-base.hubspot` contact.create (+ property map for `lead_tier`/`lead_score`/`scoring_reasoning`/`nurture_sequence`) |
| `Slack SDR Alert (Mock)` (Set) | HTTP Request to Slack incoming webhook URL (or Slack node) |
| `Email Digest (Mock)` (Set) | `nodes-base.gmail` send operation |
| `Nurture Queue (Mock)` (Set) | `nodes-base.hubspot` contact update → set `nurture_sequence=true` |
| `Log *` / `Log Duplicate` (DataTable) | `nodes-base.googleSheets` appendOrUpdate |
| `Log Failure` in error workflow (DataTable) | Google Sheets `failed_leads` tab + Slack ops webhook |

The three dedupe nodes (`Dedupe — New Lead?`, `Dedupe — Duplicate?`) become HubSpot `contact.search` by email — same `rowExists` vs `rowNotExists` split pattern, just pointed at HubSpot.
