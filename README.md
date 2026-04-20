# Lead Generation Engine

A 3-stage n8n pipeline that turns inbound form submissions into **routed, scored, deduplicated** CRM entries — without burning API spend on leads it's already seen.

![n8n](https://img.shields.io/badge/built%20with-n8n%20Cloud-orange)
![validation](https://img.shields.io/badge/validate__workflow-runtime%20clean-brightgreen)
![mock mode](https://img.shields.io/badge/demo-zero%20credentials-blue)
![tiers](https://img.shields.io/badge/lead%20tiers-4%20(hot%20%C2%B7%20warm%20%C2%B7%20cold%20%C2%B7%20duplicate)-informational)

---

## The flow at a glance

```
━━━ ① LEAD CAPTURE ━━━
Form Trigger → Validate Email → Normalize
                                    ↓ fan-out
             Dedupe — New Lead?         Dedupe — Duplicate?
              ↓ rowNotExists              ↓ rowExists
            (continue)                  Log Duplicate → end

━━━ ② CRM ENRICHMENT ━━━
Clearbit (mock) → Hit?
   ├─ true  → Merge Enrichment
   └─ false → Apollo (mock) → Merge Enrichment
                          ↓
        Score Lead (rule engine — { tier, score_num, reasoning })
                          ↓
        Insert Contact → lead_contacts table

━━━ ③ TEAM NOTIFICATION ━━━
Route by Tier (Switch on lead_tier)
   ├─ hot   → Slack SDR Alert   → Log Hot Activity
   ├─ warm  → Email Digest      → Log Warm Activity
   └─ cold  → Nurture Queue     → Log Cold Activity

━━━ ERROR WORKFLOW (separate) ━━━
Error Trigger → Format → Log Failure (lead_failed_leads)
```

Two workflows (main + error handler). Three Data Tables (contacts, activity log, failed leads). Zero external credentials to reproduce the demo.

---

## Why this is worth a look

- **Dedupe gates enrichment.** Repeat submissions short-circuit in ~110ms. No Clearbit call, no Apollo call, no OpenAI call — just an activity-log row and a 200 OK. At any meaningful volume that's the difference between a small enrichment bill and an embarrassing one.
- **Parallel `rowExists` / `rowNotExists` DataTable ops** replace the brittle `get → IF empty?` pattern most n8n tutorials teach. Cleaner semantics, no conditional gymnastics, no edge case on empty arrays.
- **Provider fallback chain with graceful failure.** Clearbit primary, Apollo fallback, cold tier when both miss — no lead ever disappears.
- **Dedicated Error Trigger workflow** catches every unhandled failure and writes `{ timestamp, email, stage, error_message, raw_payload }` to a `failed_leads` table. Silent drops are no longer a failure mode.
- **Mock-mode by design.** Every external service (Clearbit, Apollo, OpenAI, HubSpot, Slack, Gmail, Google Sheets) is a Code or Set node with a documented 1-line swap to the real provider. You can reproduce the demo with zero credentials in about a minute.

---

## Reproduce in 60 seconds (mock mode)

You need an n8n Cloud instance (or self-hosted n8n ≥ 1.60 with Data Tables enabled). No API keys, no OAuth grants.

1. **Create three Data Tables** in your n8n instance (any project):
   - `lead_contacts` — columns: `email`, `name`, `company`, `job_title`, `enrichment_source`, `lead_tier`, `lead_score`, `scoring_reasoning`, `nurture_sequence`, `created_at`
   - `lead_activity_log` — columns: `timestamp`, `email`, `name`, `company`, `tier`, `score`, `status`, `source`, `notification_channel`
   - `lead_failed_leads` — columns: `timestamp`, `email`, `stage`, `error_message`, `raw_payload`
2. **Import both workflows** from [`workflows/`](workflows/):
   - [`lead-generation-error-handler.json`](workflows/lead-generation-error-handler.json) — import this first so the main workflow can reference its ID
   - [`lead-generation-engine.json`](workflows/lead-generation-engine.json)
3. **Re-point each Data Table node to your table IDs** (they're in the JSON as specific hashes from my instance — re-map them in the node UI).
4. **Update the main workflow's `settings.errorWorkflow`** to the error handler's new ID.
5. **Activate both workflows** and submit the form via the public form URL n8n generates.

Submit `hot@stripe.com` with title "CTO" → see a hot-tier routing. Submit it again → watch the 110ms duplicate short-circuit.

Full step-by-step (including the column schemas copy-pasteable into the Data Table UI) is in [`docs/lead-generation-engine.md`](docs/lead-generation-engine.md).

---

## Evidence

Every path verified end-to-end against the live workflow. Raw execution exports:

| File | Path | Duration | Outcome |
|---|---|---|---|
| [`evidence/lead-gen-hot-run.json`](evidence/lead-gen-hot-run.json) | **Hot** (Bob@notion.so, CTO) | 1547ms | Clearbit hit → tier=hot/90 → Slack SDR alert |
| [`evidence/lead-gen-warm-run.json`](evidence/lead-gen-warm-run.json) | **Warm** (Carol@google.com, Sr Manager) | 247ms | Clearbit miss → Apollo hit → tier=warm/55 → email digest |
| [`evidence/lead-gen-cold-run.json`](evidence/lead-gen-cold-run.json) | **Cold** (Dave@unknown.xyz, Student) | 244ms | Both providers miss → tier=cold/10 → nurture queue |
| [`evidence/lead-gen-duplicate-run.json`](evidence/lead-gen-duplicate-run.json) | **Duplicate** (Alice@airbnb.com, 2nd submission) | 110ms | Dedupe fires — no enrichment, no scoring, no notification |

Validation: `validate_workflow` clean under `profile: "runtime"` on both workflows (0 errors, advisory warnings only).

The rendered hot-lead notification, straight from the execution:

```
:fire: New HOT lead: *Bob Hot* (CTO) at *Notion* — score 90. Executive title at 250-employee company with $50000000 revenue
```

---

## Going live — swap the mocks

Every mock node has a documented real-provider target in [`docs/lead-generation-engine.md`](docs/lead-generation-engine.md#going-live--swap-points). The short version:

| Mock (what's in the repo) | Real replacement |
|---|---|
| Clearbit Mock (Code) | HTTP Request → `GET person.clearbit.com/v2/people/find?email=…` + Header Auth |
| Apollo Fallback Mock (Code) | HTTP Request → `POST api.apollo.io/api/v1/people/match` + Header Auth |
| Score Lead rule engine (Code) | `@n8n/n8n-nodes-langchain.agent` + OpenAI chat model + Structured Output Parser |
| Insert Contact (DataTable) | HubSpot `contact.create` with custom properties |
| Slack SDR Alert (Set) | Slack incoming webhook HTTP Request |
| Email Digest (Set) | Gmail OAuth2 node |
| Nurture Queue (Set) | HubSpot contact update — set `nurture_sequence=true` |
| Activity log tables | Google Sheets `appendOrUpdate` |

---

## Repo layout

```
workflows/   both exported n8n workflow JSONs (main + error handler)
docs/        lead-generation-engine.md — full operator reference, gotcha log, swap-in paths
evidence/    raw execution exports for all 4 lead paths
promo/       portfolio entry content, GitHub setup guide, LinkedIn post draft
CLAUDE.md    build manual — how the engine was designed, validated, and iterated
README.md    you are here
```

---

## Deeper reading

- [`docs/lead-generation-engine.md`](docs/lead-generation-engine.md) — operator reference: data tables, scoring rubric, form field mapping, gotcha log, real-provider swap points
- [`CLAUDE.md`](CLAUDE.md) — build manual: how every node was discovered via n8n-mcp, validated, and iteratively updated

---

## License

MIT — use it, fork it, ship it. The `workflows/*.json` files are safe to import into any n8n instance; no secrets are embedded.
