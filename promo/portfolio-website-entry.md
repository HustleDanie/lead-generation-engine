# Portfolio Page — Lead Generation Engine

> **How to use this file.** Your portfolio site is Next.js/TSX, not markdown. The content below is organized by section so you can paste it into a new `app/projects/lead-generation-engine/page.tsx` that mirrors the layout of your existing `app/projects/linkedin-ai-job-alert/page.tsx`. Each block below shows the **JSX target** it maps to, the **copy** to paste, and any **data** (stats, tags) you plug into that block's props. Don't retype structural JSX — copy the existing project file, swap the sections' bodies, adjust the slug, done.

---

## 1. Metadata / frontmatter block

**JSX target:** page-level constants and `<Head>` / article header.

```
slug:         lead-generation-engine
title:        Lead Generation Engine
subtitle:     A 3-stage n8n pipeline that captures, enriches, scores, and routes inbound leads — without burning API spend on duplicates.
readTime:     10 min read
publishDate:  2026-04-20
tags:         ["n8n", "Workflow Automation", "AI Lead Scoring", "CRM", "Error Handling"]
heroColor:    "from-orange-500 to-rose-500"   // or whatever matches your design system
```

---

## 2. Hero paragraph

**JSX target:** the intro paragraph under the h1 (above the hero image / stats grid).

High-intent leads are the most valuable rows in any sales CRM — and the most commonly lost. They get swallowed by duplicates, orphaned when an enrichment API 404s, routed to the wrong channel, or dropped because nothing catches the unhandled error. The Lead Generation Engine is a production-grade n8n workflow that closes every one of those gaps: capture → enrichment → scored routing → full activity log, with a duplicate gate in front of enrichment so repeat submissions never burn a single API call.

---

## 3. Key stats grid (4 tiles)

**JSX target:** the 4-column stat grid component (same one used in the LinkedIn AI Job Alert page).

| Value | Label | Hint / icon |
|---|---|---|
| **4** | Lead tiers routed | hot · warm · cold · duplicate |
| **19** | Nodes, 0 validation errors | `runtime` profile, clean |
| **110ms** | Duplicate short-circuit | skips enrichment API spend |
| **2 + 3** | Workflows, data tables | main + error handler |

---

## 4. Problem section

**JSX target:** the first `<h2>` "problem" block.

### The problem

Most homemade "lead gen workflows" are really three fragile Zaps in a trench coat. They enrich before deduping (wasting spend), route only the winners (losing the B-team), and drop any lead that hits an API error. That's fine for demos. It's expensive at volume and embarrassing in front of a real sales team.

I wanted a single workflow that:

- **Refuses to re-enrich known contacts.** Every form re-submit should short-circuit inside ~100ms — no Clearbit call, no Apollo call, no OpenAI call.
- **Makes a routing decision for every lead, not just the hot ones.** Hot leads go to a Slack SDR channel. Warm leads get digested into an email thread. Cold leads land in a nurture sequence. No one falls through a crack.
- **Treats its own failures as first-class data.** A dedicated Error Trigger workflow catches anything that blows up and writes it to a `failed_leads` table — so "silent drop" stops being a failure mode.

---

## 5. Architecture section

**JSX target:** the architecture / flow `<h2>` block. You can render the ASCII diagram inside a `<pre>` with monospace styling, or rebuild it as SVG if you have time.

### Architecture

```
━━━ ① LEAD CAPTURE ━━━
New Lead Form (Form Trigger)
   ↓
Validate Email (IF regex)
   ↓ true
Normalize Lead (Set: lowercase+trim email, map positional → semantic keys)
   ↓ (fan-out)
Dedupe — New Lead?     Dedupe — Duplicate?
   ↓ rowNotExists             ↓ rowExists
(continue)                   Log Duplicate → end

━━━ ② CRM ENRICHMENT ━━━
Clearbit (mock) → Hit?
   ├─ true  → Merge Enrichment (input 0)
   └─ false → Apollo Fallback (mock) → Merge Enrichment (input 1)
                          ↓
        Score Lead (Rule Engine) → { tier, reasoning, score_num }
                          ↓
        Insert Contact (lead_contacts)

━━━ ③ TEAM NOTIFICATION ━━━
Route by Tier (Switch on lead_tier)
   ├─ hot   → Slack SDR Alert  → Log Hot Activity
   ├─ warm  → Email Digest     → Log Warm Activity
   └─ cold  → Nurture Queue    → Log Cold Activity
```

Three n8n Data Tables back the pipeline: `lead_contacts` (CRM substitute), `lead_activity_log` (every lead processed — including duplicates), and `lead_failed_leads` (error sink for the sibling Error Trigger workflow).

---

## 6. Feature deep-dives (2-column feature grid)

**JSX target:** the feature grid / capability cards section. Five cards fit nicely in a 2-col layout (odd card spans both).

### Duplicate gate — parallel `rowExists` / `rowNotExists`

Most n8n tutorials teach `dataTable.get → IF empty?` — which breaks the moment the get returns an empty array. This pipeline runs **two DataTable nodes off the same upstream fan-out**: one checking `rowExists`, one checking `rowNotExists`. Clean semantic split, no conditional gymnastics, and a 110ms end-to-end path when the email has already been seen.

### Provider fallback chain

Clearbit is the primary enrichment source. If it misses, Apollo gets a shot. If both miss, the lead still lives — it just lands in the cold tier with `enrichment_source="none"` so it's obvious why it scored low. The fallback is wired with a Merge node (append mode, 2 inputs) so the downstream scoring node sees one unified record regardless of which provider hit.

### Deterministic rule-engine scoring

Scoring is a Code node running regex title classification + size/revenue thresholds. Outputs `{ tier, score_num, reasoning }` with 100% schema conformance — no JSON-repair, no temperature drift, replayable byte-for-byte. The docs call out the exact swap-in point for an `@n8n/n8n-nodes-langchain.agent` + Structured Output Parser if you want LLM-based scoring instead.

### Universal error sink

A second workflow (`Lead Generation — Error Handler`) is wired via `settings.errorWorkflow` on the main pipeline. Any unhandled error — enrichment 500, scoring parse failure, DataTable write rejection — gets caught, formatted, and logged with `timestamp`, `email`, `stage` (the node that blew up), `error_message`, and the raw input payload. Silence as a failure mode is eliminated.

### Mock-mode by design

The demo runs with **zero external credentials**. Clearbit and Apollo are Code nodes with domain allow-lists. Slack, Gmail, and HubSpot are Set nodes that produce the notification payload you'd send to a real webhook. Every mock has a documented 1-line swap to the real provider — see the operator reference.

---

## 7. Tech stack display

**JSX target:** the stack / tools chip row.

**Built with:** n8n Cloud · Form Trigger v2.5 · IF v2.3 · Switch v3.4 · Merge v3.2 · Code (JavaScript) · DataTable v1.1 · Error Trigger v1

**Documented real-provider swap-ins:** Clearbit · Apollo · HubSpot · OpenAI (`gpt-4o-mini` via LangChain Agent) · Slack (incoming webhook) · Gmail OAuth2 · Google Sheets

---

## 8. Results / evidence section

**JSX target:** results `<h2>` block. Consider a code-block or styled quote for the rendered Slack message.

### Results

Four end-to-end execution paths verified against the live workflow:

| Execution | Path | Duration | Outcome |
|---|---|---|---|
| `#5` | **Hot** — Bob@notion.so, CTO | 1547ms | Clearbit hit → tier=hot/score=90 → Slack SDR alert |
| `#6` | **Warm** — Carol@google.com, Sr Manager | 247ms | Clearbit miss → Apollo hit → tier=warm/55 → email digest |
| `#7` | **Cold** — Dave@unknown.xyz, Student | 244ms | Both providers miss → tier=cold/10 → nurture queue |
| `#8` | **Duplicate** — Alice@airbnb.com, 2nd submission | 110ms | Dedupe fires — no enrichment, no scoring, no notification |

The rendered hot-lead Slack message, straight from execution #5:

> :fire: New HOT lead: **Bob Hot** (CTO) at **Notion** — score 90. Executive title at 250-employee company with $50000000 revenue

Validation: `validate_workflow` clean under `profile: "runtime"` on both workflows. Raw execution exports in [`evidence/`](https://github.com/HustleDanie/lead-generation-engine/tree/main/evidence).

---

## 9. CTA block

**JSX target:** bottom-of-page CTA card with the two buttons.

**Primary CTA:** View the repo → `https://github.com/HustleDanie/lead-generation-engine`

**Secondary CTA:** Read the operator reference → `/projects/lead-generation-engine/docs` (or link directly to the `docs/lead-generation-engine.md` on GitHub — your call)

Closing line (optional, above the buttons):

> Every node was discovered and validated through the n8n-mcp tooling, every decision is in the commit history, and every path is re-runnable in your own n8n instance in under a minute.

---

## Quick conversion checklist

- [ ] `cp app/projects/linkedin-ai-job-alert/page.tsx app/projects/lead-generation-engine/page.tsx`
- [ ] Update the slug, title, subtitle, tags, readTime
- [ ] Paste section 2 into the hero paragraph
- [ ] Populate the stats grid props with section 3
- [ ] Replace the problem, architecture, feature, results, CTA blocks with sections 4–9
- [ ] Swap the hero image (or reuse an existing automation-themed asset)
- [ ] Add the new project to your `app/projects/page.tsx` index / project list
- [ ] Fill in `HustleDanie` in the two repo links
