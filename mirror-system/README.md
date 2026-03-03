# 🏠 Documentation Mirror System

## First Principles

This system exists to give **external models read access** to canonical Notion docs via durable GitHub URLs. The driving needs:

1. **Security** — Git mirrors are inherently read-only. No external model can mutate the workspace through a GitHub URL. Essential for OpenClaw and any sandboxed agent environment.

1. **Context portability** — A raw GitHub URL works in Claude, ChatGPT, Perplexity, or any tool that accepts URLs. No per-model integration, no auth setup. Switch models mid-session and hand the same URLs.

1. **Token efficiency** — Clean markdown is 3–5× lighter than Notion API JSON for the same content. Cheaper to ingest.

### Why not MCP?

MCP can provide live Notion access, but introduces write permission risk, sequencing problems with concurrent model access, and higher token cost per page. For the "give a model current context" use case, a periodically refreshed Git mirror is simpler, safer, and cheaper. MCP may be revisited later for real-time use cases.

---

## Invariant: Framework vs Projects

- **Framework content** is reusable across projects → repo: [mirror-framework](https://github.com/sharedterrain/mirror-framework)

- **Project content** is project-specific → repo per project (e.g., Brain Stem: [brainstem-docs](https://github.com/sharedterrain/brainstem-docs))

- Never place framework-only docs into a project repo.

**Canonicality Rule:**

- Notion is the managed source of truth for content and planning

- GitHub repos are public mirrors for external AI access

- No content is considered published until mirrored to GitHub

---

## Phase 1: MVP (Trigger-Only Make + GitHub Action)

> ✅ **Status: Built.** v2 architecture is live. Make triggers only; all conversion runs in GitHub Action + Python.

> 🚀 [**Trigger Git Mirror**](https://hook.us2.make.com/4th6fnprehytjx6t30rnz9vyni50463j) — fires Make webhook → repository_dispatch → GitHub Action → converts and commits all active pages

### What it does

One button press refreshes all mirrored pages. Make fires a single `repository_dispatch` event to GitHub. A GitHub Action runs a Python script that reads the Export Scope Mapping from Notion, fetches all block content (with full pagination), converts to clean markdown, commits the `.md` files, and writes mirror status back to Notion.

### Architecture (v2 — as-built)

```plain text
Notion Button (webhook)
    → Make.com (2 modules: webhook trigger → HTTP POST repository_dispatch)
        → GitHub Action (.github/workflows/convert.yml)
            → Python script (scripts/notion_to_md.py)
                → Query Export Scope Mapping DB for active rows
                → For each row:
                    → Fetch page title (Notion API)
                    → Fetch all blocks recursively (Notion API, paginated)
                    → Convert blocks to Markdown
                    → Write .md file to repo
                    → Write Mirror Status + Last Mirrored back to Notion
                → Single atomic git commit with [skip ci]
```

### Why v2 replaced v1

The original v1 design placed all transformation logic inside Make: an 11-module scenario that fetched blocks, encoded JSON, and committed per-page inside an iterator loop. This caused:

- **7 simultaneous Git commits** that produced race conditions

- **GitHub Actions cancelling each other** from concurrent pushes

- **Make self-deactivating** on repeated conflicts

- **100-block truncation** because Make lacks native pagination

Each failure was patched incrementally instead of addressing the root cause. v2 moved all transformation to a code environment where pagination, error handling, and atomic commits are trivial. See FR-20260302-001 in the Friction Log and the "Make wires, it does not transform" candidate in Working Agreements.

### Export Scope Mapping

Maintained as a Notion database (see below on this page). The Python script queries it directly via the Notion API, filtering for rows where **Active = true**. Each row maps a source page to an output `.md` path.

**Key columns:** Page Name (title), Active (checkbox), Path (text — target `.md` path), Source Page (url — full Notion page URL), Mirror Status (select: Current/Failed/Never Mirrored/Stale), Last Mirrored (date — written by Python script).

### Make Scenario Spec

**Scenario name:** `Mirror Bulk Publish`

**Trigger:** Custom webhook (fired from Notion button on this page)

**Module 1:** Webhook — receives the trigger

**Module 2:** HTTP POST — sends `repository_dispatch` event to GitHub

```plain text
POST https://api.github.com/repos/sharedterrain/notion-mirror/dispatches
Headers:
  Authorization: Bearer <<GITHUB_TOKEN>>
  Accept: application/vnd.github+json
Body:
  {"event_type": "notion-mirror"}
```

**Total per run:** 2 ops (regardless of page count)

**Monthly at weekly cadence:** ~8 ops

### GitHub Action

**File:** `.github/workflows/convert.yml`

**Repo:** `sharedterrain/notion-mirror`

```yaml
name: convert

on:
  workflow_dispatch:
  repository_dispatch:
    types: [notion-mirror]

permissions:
  contents: write

jobs:
  convert:
    runs-on: ubuntu-latest
    timeout-minutes: 15

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 1

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install dependencies
        run: pip install requests

      - name: Run Notion converter
        env:
          NOTION_API_TOKEN: $ secrets.NOTION_API_TOKEN 
        run: python scripts/notion_to_md.py

      - name: Commit converted markdown files
        run: |
          git config user.name "notion-mirror[bot]"
          git config user.email "notion-mirror[bot]@users.noreply.github.com"
          git add .
          git diff --cached --quiet || git commit -m "docs: sync Notion pages to markdown [skip ci]"
          git pull --rebase origin main
          git push
```

### Python Converter

**File:** `scripts/notion_to_md.py`

**Repo:** `sharedterrain/notion-mirror`

Queries the Export Scope Mapping DB for active rows, fetches block content with full pagination (no 100-block limit), converts to clean Markdown, writes `.md` files, and writes Mirror Status + Last Mirrored back to each mapping row.

### Supported block types

The converter handles: headings (1-3), paragraphs, bulleted/numbered lists, to-do items, toggles, quotes, callouts, code blocks, dividers, tables, images, video/file/pdf/audio embeds, bookmarks, equations, columns, synced blocks, child pages/databases (as labels), and table of contents. Unsupported block types render as HTML comments.

### Placeholders

- `<<NOTION_TOKEN>>` — Integration token. Stored device-only + as `NOTION_API_TOKEN` GitHub Secret in `sharedterrain/notion-mirror`

- `<<GITHUB_TOKEN>>` — Personal access token (repo scope, `sharedterrain/notion-mirror`). Expires May 25, 2026. Used by Make Module 2 for `repository_dispatch`

- `<<MAKE_WEBHOOK_URL>>` — Webhook URL for the bulk publish trigger

### Staleness Checker (Cron)

A daily GitHub Actions cron job that does two things:

1. **Resolves empty Source Page URLs** — searches Notion by Page Name title, writes the matching page URL back to the Source Page column. This means you only need to fill in Page Name and Path when adding rows; the script populates Source Page automatically on the next run.

1. **Checks staleness** — fetches `last_edited_time` from Notion for each source page and writes it to the **Last Updated** column. If Last Updated > Last Mirrored, the doc is stale.

No manual stamping, no manual URL pasting, works for every page in the mapping.

**File:** `.github/workflows/staleness.yml`

**Repo:** `sharedterrain/notion-mirror`

```yaml
name: Check Mirror Staleness

on:
  schedule:
    - cron: "0 14 * * *" # 6am PT daily
  workflow_dispatch:

jobs:
  staleness:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - run: pip install requests

      - run: python scripts/check_staleness.py
        env:
          NOTION_API_TOKEN: $ secrets.NOTION_API_TOKEN 
```

**File:** `scripts/check_staleness.py`

**Repo:** `sharedterrain/notion-mirror`

**How staleness works:** The cron writes Notion's `last_edited_time` to Last Updated. The mirror script writes `now()` to Last Mirrored. If Last Updated > Last Mirrored, the source has been edited since the last mirror — it's stale. Source Page URLs are auto-resolved on each run, so new rows only need Page Name and Path.

---

### What's explicitly deferred

- CI checks on generated markdown

- Friction DB integration and Slack capture

- Promotion rituals and drift control

- Sessions DB

- Per-page publish buttons (bulk covers the need)

- Budget guardrails automation (manual tracking is fine at this volume)

- Status writeback design (currently handled by Python script; separate mechanism TBD)

---

## Future Phases (Reference Material)

The following child pages contain detailed designs for the full mirror system vision. They are **not in active scope** but are preserved for future phases.

**Child page:** 📦 Framework (Governance & CI)

10 child pages covering mirror publishing rules, drift control, CI checks, automation config, contract templates, bootstrap guide, and protocol files. Written for the full automated system — pick up when MVP is proven.

**Child page:** 📁 Projects

Brain Stem mirror mapping and project-specific mirror config. The export scope mapping in Phase 1 covers this need for now.

**Child database:** Friction Log

**Child database:** Export Scope Mapping Trigger Git Mirror — fires Make webhook → repository_dispatch → GitHub Action → converts and commits all active pages

**Child page:** Make Scenario Build Reference
