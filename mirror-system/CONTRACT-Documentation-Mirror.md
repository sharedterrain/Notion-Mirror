# CONTRACT (Documentation Mirror)

```yaml
---
doc_id: "contract_documentation_mirror"
last_updated: "2026-03-10"
contract_version: "0.2.0"
parent_contract: "contract_hub"
---
```

**Status:** Live

### Metadata (operational)

- **Owner:** Jedidiah Duyf

- **Notion source:** [🏠 Documentation Mirror System](https://www.notion.so/9f062b4d8d134901a8c896a7c233dc33)

- **Mirror repo:** notion-mirror (public), magis-workshop (private)

- **Implementation state:** v2 (trigger-only Make + GitHub Action)

- **Last change:** v0.2.0 — §5 added INV-M01 note (manual cleanup caveat) and INV-M08 (Script Source of Truth). §6 added Last Mirrored Visibility and Cleanup columns, Last Updated caveat. §9 removed point-in-time row count. §10 updated visibility change detection status. §11 deviation log entry.

---

## 1. Purpose

The Documentation Mirror System exists to give **external models read-only access** to canonical Notion docs via durable GitHub URLs. It converts Notion pages to clean Markdown and commits them to two Git repositories — one public, one private — controlled by a centralized Export Scope Mapping database.

**Core capability:** One-button publish from Notion → dual-repo Git mirrors, with automated staleness detection and visibility-based routing.

---

## 2. Scope & Boundaries

**In scope:**

- Notion page → Markdown conversion and Git mirroring

- Dual-repo visibility routing (Public / Private)

- Export Scope Mapping database as the routing control plane

- Staleness detection and Mirror Status tracking

- Source Page auto-resolution by title

- Make webhook trigger → GitHub Actions `repository_dispatch`

**Out of scope:**

- Real-time sync (trigger-based only)

- Git → Notion reverse sync

- MCP-based live access (deferred; see §10)

- CI checks on generated markdown (deferred)

- Per-page publish buttons (bulk publish covers the need)

**External dependencies:**

- Notion API (content source + ESM database)

- GitHub (dual-repo target: `sharedterrain/notion-mirror`, `sharedterrain/magis-workshop`)

- [Make.com](http://make.com/) (webhook trigger, 3-module scenario)

- GitHub Actions (conversion runtime)

- Python 3.12 + `requests` (converter + staleness scripts)

---

## 3. System Architecture

### 3a. Functional Pipeline

The mirror system processes pages through four abstract stages:

- **Trigger** — Accepts a publish signal and dispatches conversion jobs to both repos.

- **Selection** — Queries the ESM for active rows, filters by Visibility, skips unchanged pages.

- **Conversion** — Fetches Notion page blocks recursively, converts to clean Markdown.

- **Commit** — Writes `.md` files to the repo and updates mirror status back to the ESM.

**Pipeline flow:**

```plain text
Trigger → Selection → Conversion → Commit → Status Writeback
```

**Staleness pipeline (independent, daily cron):**

```plain text
Cron → Fetch all active ESM rows → Resolve empty Source Pages → Compare last_edited_time vs Last Mirrored → Flag Stale
```

### 3b. Current Provider Mapping

| **Functional Stage** | **Current Provider** | **Auth Method** |
| --- | --- | --- |
| Trigger | [Make.com](http://make.com/) (3-module scenario: webhook receiver → HTTP POST to notion-mirror → HTTP POST to magis-workshop) | Custom webhook URL + scoped PATs |
| Selection | Python script (`notion_to_md.py`) | Notion API token |
| Conversion | Python script (`notion_to_md.py`) | Notion API token |
| Commit | GitHub Actions (`convert.yml`) | Git push (Actions token) |
| Status Writeback | Python script (`notion_to_md.py`) | Notion API token |
| Staleness Detection | Python script (`check_staleness.py`) | Notion API token |
| Source Page Resolution | Python script (`check_staleness.py`) | Notion API token |

**Trust boundaries:**

- [Make.com](http://make.com/) → GitHub: scoped Personal Access Tokens (one per repo)

- GitHub Actions → Notion API: `NOTION_API_TOKEN` secret

- ESM database: read/write by scripts only; no external model access

---

## 4. Interface Definitions

### Trigger Interface

**Boundary:** Notion button → [Make.com](http://make.com/) → GitHub Actions

**Input:** Webhook POST (no payload required)

**Output:** Two `repository_dispatch` events (`event_type: "notion-mirror"`) — one to each repo.

### ESM Query Interface

**Boundary:** Python script → Notion API

**Input:** Database query with filter: `Active = true` AND `Visibility = {env VISIBILITY}`

**Output per row:**

- `page_id` (string) — extracted from Source Page URL

- `path` (string) — target `.md` file path in repo

- `name` (string) — Page Name title

- `row_id` (string) — ESM row ID for writeback

- `last_mirrored` (ISO-8601 datetime or empty)

### Conversion Interface

**Boundary:** Notion API → Markdown file

**Input:**

- `page_id` (string) — Notion page to convert

**Output:**

- `title` (string) — page title from Notion

- `blocks` (array) — recursively fetched block tree (full pagination, no 100-block limit)

- `md_content` (string) — `# {title}\n\n{blocks_to_md(blocks)}`

### Status Writeback Interface

**Boundary:** Python script → Notion API (ESM row update)

**Input:**

- `row_id` (string) — ESM row to update

- `status` (string) — "Current" or "Failed"

**Output:** ESM row updated with Mirror Status + Last Mirrored timestamp.

### Staleness Interface

**Boundary:** Cron script → Notion API

**Input:** All active ESM rows

**Output per row:**

- Last Updated column written with `last_edited_time` from Notion page metadata

- Mirror Status set to "Stale" if `last_edited_time > Last Mirrored`

- Source Page URL auto-resolved by title search if empty

---

## 5. Invariants

### INV-M01: Dual-Repo Visibility

Every active ESM row routes to exactly one repo based on its Visibility column. Public → `notion-mirror`. Private → `magis-workshop`. A page must never appear in both repos simultaneously.

*Note: Visibility change detection is implemented in the converter. Cleanup of stale copies in the old repo when Visibility changes remains a manual step — see §10.*

### INV-M02: Canonicality

Notion is the managed source of truth. GitHub repos are read-only mirrors. No content is authoritative until mirrored. No manual edits to `.md` files in the repos.

### INV-M03: Make Wires, It Does Not Transform

[Make.com](http://make.com/) fires webhook dispatches only. All transformation, pagination, error handling, and commit logic runs in GitHub Actions + Python. Make never touches page content.

### INV-M04: Atomic Commits

Each mirror run produces at most one Git commit per repo. The commit includes all converted pages for that run. No per-page commits (prevents race conditions — see v1 failure mode in overview page).

### INV-M05: Skip Unchanged

The converter compares `last_edited_time` against `Last Mirrored` for each page. Unchanged pages are skipped. This is a performance optimization and consistency check, not a gating mechanism.

### INV-M06: Credential Isolation

Notion API token, GitHub PATs, and Make webhook URL are never committed to any repo. They exist only in GitHub Secrets and Make scenario config. The `<<PLACEHOLDER>>` pattern is used in all documentation.

### INV-M07: Inbox Log (ESM)

Every mirror run writes Mirror Status and Last Mirrored back to each processed ESM row. Failed pages get `Mirror Status = Failed`. This is the system's audit trail.

### INV-M08: Script Source of Truth

Notion child pages are the authoritative source for all scripts (notion_to_[md.py](http://md.py/), check_[staleness.py](http://staleness.py/), convert.yml, staleness.yml). GitHub copies are mirrors. Scripts are edited in Notion and mirrored to repos — never edited directly in GitHub.

---

## 6. ESM Schema Contract

The Export Scope Mapping database is the control plane for the mirror system. Its schema is a contract — column names and types must match what the Python scripts expect.

| **Column** | **Type** | **Written By** | **Purpose** |
| --- | --- | --- | --- |
| Page Name | Title | Human | Display name; used for Source Page auto-resolution |
| Active | Checkbox | Human | Rows with Active=false are ignored by all scripts |
| Path | Rich text | Human | Target `.md` file path in repo (e.g. `brain-stem/CONTRACT.md`) |
| Source Page | URL | Human or staleness cron | Full Notion page URL; auto-resolved by title if empty |
| Visibility | Select (Public/Private) | Human | Controls which repo receives the file |
| Section | Select | Human | Organizational grouping (Brain Stem, Magi, Hub, etc.) |
| Mirror Status | Select (Current/Failed/Never Mirrored/Stale) | Scripts | Visual indicator; staleness cron writes Stale, converter writes Current/Failed |
| Last Mirrored | Date | Converter script | Timestamp of last successful mirror for this row |
| Last Updated | Date | Staleness cron | Notion's `last_edited_time` for the source page. Skipped when visibility mismatch is detected — row exits early after Mirror Status is set to Stale. |
| Last Mirrored Visibility | Select (Public/Private) | Converter script | Records which Visibility value was active at last mirror. Used by both converter (force-convert on mismatch) and staleness cron (flag Stale on mismatch). Written on every successful mirror run. |
| Cleanup | Checkbox | Human | Flags rows where stale copies need manual deletion from the old repo after a visibility change. |

**Schema invariant:** If a column is renamed or removed, both Python scripts (`notion_to_md.py` and `check_staleness.py`) must be updated in the same session. Column names are hardcoded in the scripts' Notion API queries.

---

## 7. Supported Block Types

The converter handles the following Notion block types:

- **Text:** headings (1-3), paragraphs, bulleted/numbered lists, to-do items

- **Structure:** toggles, quotes, callouts, dividers, columns, synced blocks

- **Code:** code blocks (with language tag)

- **Media:** images, video, file, PDF, audio, embeds, bookmarks

- **Data:** tables (header row + data rows)

- **Meta:** table of contents, equations, child pages/databases (as labels), breadcrumbs

Unsupported block types render as HTML comments: `<!-- unsupported block type: {type} -->`

---

## 8. Credential & Placeholder Tracker

| **Placeholder** | **Scope** | **Storage** |
| --- | --- | --- |
| `<<NOTION_TOKEN>>` | Notion API access (read pages + write ESM) | GitHub Secret (`NOTION_API_TOKEN`) in both repos; device-only local |
| `<<GITHUB_PAT_PUBLIC>>` | `repository_dispatch` to notion-mirror | Make scenario Module 2 |
| `<<GITHUB_PAT_PRIVATE>>` | `repository_dispatch` to magis-workshop | Make scenario Module 3 |
| `<<MAKE_WEBHOOK_URL>>` | Bulk publish trigger | Notion button on overview page |
| `<<EXPORT_SCOPE_DB_ID>>` | ESM database ID for API queries | Hardcoded in scripts (env var override available) |

---

## 9. Implementation State

**v1:** Make-native conversion — *Retired*

- 11-module Make scenario with in-scenario transformation

- Failed: race conditions from per-page commits, 100-block truncation, self-deactivation

- See FR-20260302-001 in Friction Log

**v2:** Trigger-only Make + GitHub Action — *Live*

- Make reduced to 3 modules (webhook → 2× HTTP POST)

- Python converter runs in GitHub Actions with full pagination

- Atomic single commit per repo per run

- Staleness cron: daily at 14:00 UTC (6am PT)

- Source Page auto-resolution: searches Notion by title, writes URL back to ESM

**Known gap:** Visibility change detection — moving a page from Private → Public (or vice versa) does not trigger cleanup of the old repo copy. Stale copy cleanup in the old repo remains a manual step. See §10 Deferred.

---

## 10. Deferred

- **Visibility change detection** — staleness cron compares Last Mirrored Visibility vs Visibility and flags Stale on mismatch (implemented). Converter force-converts on mismatch (implemented). Cleanup of stale `.md` from wrong repo remains manual.

- CI checks on generated markdown

- Slack capture

- Promotion rituals and drift control

- Sessions DB

- Per-page publish buttons (bulk covers the need)

- Budget guardrails automation (manual tracking is fine at this volume)

- MCP-based live access (may revisit for real-time use cases)

- Branch protection on notion-mirror / magis-workshop repos (low priority at current volume; would complicate atomic-commit flow)

---

## 11. Deviation Log

| **Version** | **Date** | **Description** |
| --- | --- | --- |
| 0.2.0 | 2026-03-10 | §5 added INV-M01 note (visibility cleanup is manual — see §10) and INV-M08 (Script Source of Truth — Notion child pages are authoritative for all scripts). §6 added Last Mirrored Visibility (select, written by converter) and Cleanup (checkbox, written by human) columns; Last Updated purpose updated with visibility-mismatch early-exit caveat. §9 removed point-in-time ESM row count. §10 updated visibility change detection from "not yet implemented" to partially implemented (detection live, cleanup still manual). Minor version bump — new invariant + new ESM columns, additive only, no breaking changes. |
| 0.1.0 | 2026-03-09 | Initial contract. Extracted governance content from Documentation Mirror System overview page. Codified 7 invariants, ESM schema contract, 5 interface definitions, credential tracker, and implementation state (v1 retired, v2 live). Script source moved to dedicated Notion child pages (notion-mirror and magis-workshop variants). Parent page restructured as system overview; contract is the governance artifact. |

---

## 12. Change Control Protocol

**Rule:** Contract defines *what must be true*. The overview page describes *how it works in practice*. Script child pages hold *implementation source*.

### 12.1 Version Bump Rules

- **Patch (0.0.x):** Clarifications, typo fixes, no interface or invariant changes.

- **Minor (0.x.0):** New invariant, new interface, new ESM column, additive changes. Backward compatible.

- **Major (x.0.0):** Breaking change to ESM schema, interface shape change, invariant removal.

### 12.2 Impact Radius

**Downstream docs:** Documentation Mirror System (overview page), script child pages (notion_to_[md.py](http://md.py/), check_[staleness.py](http://staleness.py/), convert.yml, staleness.yml, validate.yml), ESM database schema.

### 12.3 Downstream Sync Deadline

After a contract change, all downstream docs in the impact radius must be updated **in the same work session**. If a downstream doc cannot be updated, add: `⚠️ STALE — see [version]`

---

## 13. Appendix

### Related Documents

- [🏠 Documentation Mirror System](https://www.notion.so/9f062b4d8d134901a8c896a7c233dc33) (overview + parent page)

- [notion_to_md.py (notion-mirror)](https://www.notion.so/3de220c93cf9444cbfb747e482bbb320)

- [notion_to_md.py (magis-workshop)](https://www.notion.so/3a6321e4055642009481ed3a59f71c63)

- [check_staleness.py (notion-mirror)](https://www.notion.so/f0d4945d98174c899e9d7925ccae1c81)

- [convert.yml (notion-mirror)](https://www.notion.so/0fa8741b4aa74c64b8800819dbcacb9d)

- [convert.yml (magis-workshop)](https://www.notion.so/a18f315ba91b4ea688f96d8f3ab389b7)

- [validate.yml (Archived)](https://www.notion.so/0fe2b88d819645eebd8e774e79dff450)

- [staleness.yml (notion-mirror)](https://www.notion.so/960b9dce92f04f7f9deb48d76ee1aa85)

- [Export Scope Mapping](https://www.notion.so/38f8657d2479419599377864111fea70)
