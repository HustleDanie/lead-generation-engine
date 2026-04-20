# Publishing the Lead Generation Engine to GitHub

This is a two-step publish: (1) initialize a standalone git repo for just this project, then stage and commit what should ship, (2) create the remote and push. Pick path **A** (with `gh` CLI) or path **B** (pure git) depending on your setup.

> **Heads-up about the git setup.** Your Lead Generation Engine folder currently lives inside a larger git repository rooted at `C:/Users/DELL/` — the same repo that holds the earlier LinkedIn AI Job Alert Pipeline work in `Jobs/`. Publishing the Lead Generation Engine as its **own standalone repo** (which you want for a discoverable portfolio piece) means initializing a fresh, nested git repo inside this folder. The nested `.git` shadows the parent one for everything under this directory — clean and safe, just slightly unusual. Instructions below handle it in one step.

---

## Pre-flight (30 seconds, do this first)

Run these four checks before anything touches the remote:

```bash
cd "c:/Users/DELL/Portfolio Project/Automation/Lead Generation Engine"

# 1. Initialize a new standalone git repo INSIDE this folder (shadows the parent repo for this subtree)
git init -b main

# 2. Confirm the MCP config is gitignored
grep -i mcp .gitignore           # should print: .mcp.json
git check-ignore -v .mcp.json    # should print the .gitignore rule matching .mcp.json

# 3. Review what's about to be staged
git status --short

# 4. Sanity-check that .mcp.json is NOT in the "to be committed" list
git status --short | grep -i mcp || echo "clean — no mcp files surfaced"
```

If check #4 surfaces `.mcp.json`, **stop** — fix the `.gitignore` first, because a `git add` would track it.

---

## Path A — with `gh` CLI (preferred)

Assumes `gh auth status` shows you're logged in. Run this after the pre-flight (which already did `git init -b main`).

```bash
cd "c:/Users/DELL/Portfolio Project/Automation/Lead Generation Engine"

# Stage only the files that belong in the portfolio repo
git add README.md promo/ docs/ evidence/ workflows/ CLAUDE.md .gitignore

# Verify the staging set (no .mcp.json, no personal data)
git status

# First commit on this fresh repo
git commit -m "Initial commit: Lead Generation Engine (workflows, docs, evidence, promo)"

# Create the remote as PUBLIC, set origin, push main in one shot
gh repo create lead-generation-engine \
  --public \
  --source=. \
  --remote=origin \
  --push \
  --description "3-stage n8n lead pipeline: capture → enrichment → scored routing. 4 tiers, duplicate gating, dedicated error workflow."

# Add discoverability topics
gh repo edit --add-topic n8n,automation,lead-generation,crm,ai-scoring,workflow-automation,portfolio

# Open the repo in your browser to eyeball the rendered README
gh repo view --web
```

---

## Path B — pure git (no `gh` CLI)

1. **Create the repo on github.com manually:** go to https://github.com/new, name it `lead-generation-engine`, make it **Public**, do NOT initialize with README / license / .gitignore (you already have those locally).

2. Then in your terminal (assuming the pre-flight already ran `git init -b main`):

```bash
cd "c:/Users/DELL/Portfolio Project/Automation/Lead Generation Engine"

git add README.md promo/ docs/ evidence/ workflows/ CLAUDE.md .gitignore
git status
git commit -m "Initial commit: Lead Generation Engine (workflows, docs, evidence, promo)"

git remote add origin https://github.com/<YOUR_USERNAME>/lead-generation-engine.git
git push -u origin main
```

Replace `<YOUR_USERNAME>` with your GitHub handle.

3. On github.com, click the ⚙️ next to "About" on the repo page and add:
   - **Description:** `3-stage n8n lead pipeline: capture → enrichment → scored routing. 4 tiers, duplicate gating, dedicated error workflow.`
   - **Topics:** `n8n`, `automation`, `lead-generation`, `crm`, `ai-scoring`, `workflow-automation`, `portfolio`

---

## What gets pushed (and what doesn't)

Pushed:

| Path | Why |
|---|---|
| `README.md` | Landing page for the repo |
| `workflows/*.json` | Both exported n8n workflow JSONs — anyone can import them |
| `evidence/*.json` | Execution exports for all 4 lead paths (hot/warm/cold/duplicate) |
| `docs/lead-generation-engine.md` | Full operator reference with gotchas + real-provider swap-in paths |
| `promo/` | The three showcase files (portfolio entry, this setup guide, LinkedIn post) |
| `CLAUDE.md` | The build manual — shows reviewers how the project was built |
| `.gitignore` | Keeps `.mcp.json` and other local-only files out |

**NOT pushed:**
- `.mcp.json` (contains the n8n cloud API key — in `.gitignore`)
- Any `.claude/` or `.vscode/` local state
- Screenshots you haven't committed yet (none required — the evidence JSONs tell the story)

---

## After the push — a 3-minute polish pass

1. **Pin the repo** to your GitHub profile so it shows up first for visitors.
2. **Add a one-line social preview image** (GitHub repo → Settings → Social preview). A screenshot of the architecture diagram from `docs/lead-generation-engine.md` or the Slack-style hot-lead message works well.
3. **Drop the repo link into your portfolio site's project index** (`app/projects/page.tsx` in `My-Portfolio`).
4. **Cross-post a release tag** once you've pushed:
   ```
   git tag -a v1.0.0 -m "Initial release — mock-mode demo, 4 lead paths verified"
   git push --tags
   ```
   This gives LinkedIn viewers a semver-pinned version to reference.

---

## Rollback if something goes wrong

If you push and realize `.mcp.json` leaked despite the pre-flight:

```bash
# Make the repo private IMMEDIATELY
gh repo edit --visibility private --accept-visibility-change-consequences

# Rotate the n8n API key (hustledaniel.app.n8n.cloud → Settings → API)
# Then rewrite history and force-push
git rm --cached .mcp.json
echo ".mcp.json" >> .gitignore
git add .gitignore
git commit -m "Remove leaked MCP config"
git push --force-with-lease origin main
```

Rotating the key is non-negotiable — even after a force-push, git hosting providers keep the old commit in the reflog for some time, and crawlers may have already indexed it.
