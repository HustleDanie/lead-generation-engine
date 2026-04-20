# LinkedIn post — Lead Generation Engine

Three versions below. Start with the **Primary post**. Keep the **Pinned reply** ready to paste as the first comment (pushes your own link above the algorithmic comment noise). The **Short version** is for cross-posting to X / Threads / Bluesky where the context budget is tighter.

---

## Primary post (LinkedIn, ~240 words)

> Most "lead generation" workflows burn API spend on leads they've already seen. Built one that doesn't.
>
> Just shipped the Lead Generation Engine — a 3-stage n8n pipeline that takes form submissions through capture → enrichment → scored routing, with a duplicate gate in front of every paid API call.
>
> Three design choices that matter:
>
> → **Dedupe gates enrichment.** A repeat submission short-circuits in ~110ms — no Clearbit call, no Apollo call, no OpenAI call, just an activity-log row and a 200 OK. At scale that's the difference between a $40/mo enrichment bill and a $400/mo one.
>
> → **Parallel `rowExists` / `rowNotExists` DataTable ops.** Cleaner than the usual `get → IF empty?` pattern (which breaks on empty arrays). Two nodes, one fan-out, no conditional gymnastics.
>
> → **A dedicated Error Trigger workflow.** Wired via `settings.errorWorkflow` on the main pipeline, catches every unhandled error and writes `{ timestamp, email, stage, error_message, raw_payload }` to a `failed_leads` table. "Silent drop" stops being a failure mode.
>
> Receipts: 19 nodes, 0 validation errors under n8n's `runtime` profile, and 4 lead tiers (hot · warm · cold · duplicate) verified end-to-end with execution exports in the repo.
>
> The demo runs in **mock mode** — zero external credentials — with documented 1-line swap-ins for Clearbit, Apollo, HubSpot, OpenAI, Slack, and Gmail.
>
> Repo (workflows, evidence JSONs, full operator docs): https://github.com/<YOUR_USERNAME>/lead-generation-engine
>
> #n8n #WorkflowAutomation #LeadGen #AIEngineering #CRM

**Char count:** ~1,950 (LinkedIn limit is 3,000 — plenty of headroom if you want to add a personal aside at the top).

---

## Pinned reply (paste as the first comment right after posting)

> For anyone wondering how this maps to a real production stack — the mock Code nodes swap 1:1 with:
>
> • Clearbit → HTTP Request (`/v2/people/find`) + Header Auth
> • Apollo → HTTP Request (`/api/v1/people/match`) + Header Auth
> • Score Lead → `@n8n/n8n-nodes-langchain.agent` + OpenAI chat model + Structured Output Parser
> • Insert Contact → HubSpot `contact.create` with custom properties for `lead_tier` / `lead_score` / `scoring_reasoning`
> • Slack SDR alert → Slack incoming webhook
> • Email Digest → Gmail OAuth2 node
> • Activity log → Google Sheets `appendOrUpdate`
>
> Full mapping + the gotchas I hit along the way (Form Trigger v2.5 doesn't support `responseMode: "responseNode"`, DataTable insert returns the row shape not the upstream shape, etc.) are in `docs/lead-generation-engine.md` in the repo.

---

## Short version (X / Threads / Bluesky, ~270 chars)

> Shipped a 3-stage n8n lead-gen pipeline. Duplicates short-circuit in 110ms before hitting a single paid API. Parallel rowExists/rowNotExists beats get→IF-empty. Dedicated Error Trigger workflow catches everything.
>
> 4 tiers verified, mock-mode demo, zero creds.
>
> https://github.com/<YOUR_USERNAME>/lead-generation-engine

---

## Posting tips (quick)

- **Best time:** Tuesday–Thursday, 9–11 AM local time (or ~8 AM PT if your target audience is US tech).
- **Hook retention:** LinkedIn shows the first ~210 characters before "…see more." The first 2 lines have to earn the click. The version above does that with the cost-angle counter-intuitive claim.
- **Visuals beat text 2–3×:** attach a screenshot of the architecture diagram or the rendered Slack hot-lead message. If you don't have time to build one, the ASCII diagram in `docs/lead-generation-engine.md` screenshots cleanly from a dark-themed terminal.
- **Don't edit the post after publishing.** LinkedIn demotes edited posts in the feed for ~an hour.
- **Engage the first 5 comments fast** (within the first 30 minutes). That's the window the algorithm uses to decide whether to push it wider.

---

## Metrics worth tracking after 72 hours

If any of these hit, it's working:

| Metric | Healthy baseline | Signal |
|---|---|---|
| Impressions | 2,000+ | Post cleared the initial seed audience |
| Reactions | 50+ | Topic resonates with your network |
| Repo stars | 5+ attributable | LinkedIn → GitHub funnel is converting |
| DMs from recruiters / founders | 1+ | The "depth signal" landed |
