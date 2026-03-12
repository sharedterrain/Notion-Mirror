# Notion Hub Steering Document

```yaml
---
document_id: "notion-hub-steering"
document_version: "3.0.0"
status: "active"
last_updated: "2026-02-22"
owner: "Jedidiah Duyf"
created: "2026-02-10"
parent_contract: "contract_hub"
contract_version: "1.1.0"
mirror_target: "mirror-framework/hub/HUB_STEERING.md"
---
```

> 📜 **This document is subordinate to **[**CONTRACT (Hub)**](https://www.notion.so/c18af9cbec3b4c388c3561036a4871f1)**.** Where they conflict, the CONTRACT wins. Binding rules, invariants, and edit privileges live in the CONTRACT. This document contains operational state, tool routing, navigation, and session history.

---

## §1 Meta: How This Document Is Maintained

**Standard workflow**

1. **Primary operator:** Notion AI — architecture, synthesis, governance editing, and execution. This is the reliable admin layer for all workspace governance.

1. **External model input:** When work is done in an external model (Claude, GPT, etc.), Notion AI parses and incorporates the results directly. No standardized format required.

1. **Integration:** Apply validated changes to this document with timestamp + Change Log entry

1. **Promotion:** Governance-impacting changes promoted to Git mirror via standard process

**Scope:** This workflow applies ONLY to Hub Steering Document. Individual projects define their own coordination workflows in project steering docs.

**Superseded by:** Any decision logged in Global Decisions that explicitly changes this workflow.

---

## §2 Pointers

### Active Projects

- Brain Stem Project → [Brain Stem Steering Document (Deprecated)](https://www.notion.so/46201f5e5e284d1a98444667f62d382a)

- Documentation Mirror System → [Untitled](https://www.notion.so/1b1843a91b8942d190880fa333a4c424)

- Regenerative Systems Hub → [placeholder - not yet active]

### Governance

- CONTRACT (Hub) → [CONTRACT (Hub)](https://www.notion.so/c18af9cbec3b4c388c3561036a4871f1)

- Architecture Principles → [Architecture Principles](https://www.notion.so/9837a2e8c8a94f3a9320cf866d69aaac) — Operational guidance for working efficiently within Hub governance. Candidate status: items here are guidance until explicitly promoted to Working Agreements, at which point they become enforceable.

- Dependency Registry → [Dependency Registry](https://www.notion.so/abb5799fc94548c4a0ea87e45fe82267)

### Framework Resources

- Working Agreements (Framework) → [Working Agreements (Framework)](https://www.notion.so/a1b90a3d197647f9add8e638bdc5e4c3)

- Friction Log → [Friction Log](https://www.notion.so/3ff4a1e48fd2424b8cb72ef0a79e41cc)

- Parking Lot → [🅿️ Parking Lot](https://www.notion.so/20ad3780056b446b8affe27f7f621f24)

- Workspace Index → [Untitled](https://www.notion.so/a28ce8ce03a54e988781a64a2b341f1b)

- Prompt Library → [Workspace Prompt Library](https://www.notion.so/489f08675359425fae460753c48b6bf8)

### Tool Instruction Sets

- Tool Instruction Library → [Tool Instruction Sets](https://www.notion.so/30770cff90fc80df97d5c84915f89dbf) — Project-focused model instruction sets (e.g., Claude — Brain Stem, Claude — OpenClaw). Organized by model × project.

---

## §3 Tool Reality (Current Constraints)

- **ChatGPT Projects:** do not assume live Notion/GitHub tool access unless proven with artifacts in-session.

- **Claude MCP (Notion):** technically capable, but currently **uneconomical** for routine operations (tool-call overhead consumes usage quickly).

- **Operating mode:** Notion AI is the **reliable admin layer** — primary operator for governance, architecture, synthesis, and execution. External models provide specialist input; Notion AI parses and incorporates results directly. MCP tool-calling is "break-glass" only for obvious high-ROI batch actions.

- **Git mirror:** maintained as a **thin, curated publishing lane** (CI + immutable history + public showcase), not an operational dependency.

- **Mirror read surface (models):** prefer **raw** URLs (`raw.githubusercontent.com/`...) and/or commit-permalinks; GitHub `blob` pages may not yield file bodies reliably in this surface. If a model cannot fetch a link from an index, paste the relevant raw URLs or excerpt explicitly.

---

## §4 Tool Routing Guidance

**Purpose:** Route tasks to the tool/model with strongest capability for that task type.

### Tool Routing Table

| **Task Type** | **Primary Tool** | **Rationale/Notes** |  |
| --- | --- | --- | --- |
| Workspace governance, architecture, synthesis, execution | Notion AI | **Reliable admin layer.** Architecture, multi-doc synthesis, governance editing, page ops, audit |  |
| Specialist input (critique, gap analysis, deep reasoning) | External models (Claude, GPT, etc.) | Advisory/specialist; Notion AI parses and incorporates results directly |  |
| Orchestration / actuation across tools | [Make.com](http://make.com/) | Deterministic automation; contracts + idempotency; workspace navigation supplement |  |
| Current-state research, tool capabilities | Perplexity | Real-time web access, citations |  |
| Code generation, scripts, CI wiring | Claude Code / Codex | Implementation specialization |  |

**Clarification (role boundary):** Notion AI is the **reliable admin layer** for workspace governance. External models provide specialist input; Notion AI parses and incorporates results directly. Mirroring/actuation remains Notion button → Make → GitHub.

---

## §5 MCP Integration Posture

**Current operating mode (per CONTRACT §12 SR-5):**

- Notion AI is the reliable admin layer — handles architecture, synthesis, governance editing, and execution natively

- External models provide specialist input; Notion AI parses and incorporates results directly

- MCP tool-calling remains **break-glass only** — not assumed viable for routine operations due to usage economics

**Future phases:** `<<UNKNOWN>>` — timing and viability depend on proven net time/effort savings vs manual/Notion AI workflow. No quarter-specific timelines assumed.

**Decision point:** Revisit only after clear evidence of economic viability and reliability.

---

## §6 Global Decisions

**Format:** Decision number, date, what was decided, rationale, affected projects, supersedes (if any).

**Promotion rule:** When a decision proves durable and broadly applicable, it is promoted to a Standing Rule in [CONTRACT (Hub)](https://www.notion.so/c18af9cbec3b4c388c3561036a4871f1) §12. Promoted decisions are marked below. New decisions still land here first.

**Decision 1 (2026-02-10):** Use Opus for Hub Steering architecture sessions, GPT for functional refinement. *Superseded by SR-5 (Decision 13).*

**Decision 2 (2026-02-10):** Canonicality boundaries formalized: Notion = operational memory, Airtable = structured records, Git = promoted governance. *Incorporated into CONTRACT (Hub) §4.*

**Decision 3 (2026-02-10):** Edit privileges classified into 4 lanes (A/B/C/D). *Incorporated into CONTRACT (Hub) §5.*

**Decision 4 (2026-02-10):** Document size limits enforced; monthly compaction ritual. *Incorporated into CONTRACT (Hub) §10.*

**Decision 5 (2026-02-10):** MCP rollout staged: Read-only → Bounded write → Full integration. *Superseded by Decision 10.*

**Decision 6 (2026-02-11):** Tool Guidance stubs deprioritized. *Superseded by SR-6 (Decision 14).*

**Decision 7 (2026-02-11):** Demote Documentation Mirror System from critical path. *Promoted to CONTRACT (Hub) §12 SR-1.*

**Decision 8 (2026-02-11):** Promotion boundary rule — framework vs project repo. *Promoted to CONTRACT (Hub) §12 SR-2.*

**Decision 9 (2026-02-11):** Model check-in cadence on scope shifts. *Promoted to CONTRACT (Hub) §12 SR-3.*

**Decision 10 (2026-02-11):** MCP/connector tool-calling is not assumed viable for routine operations due to usage economics; default workflow remains manual/Notion AI with Git mirrors for durable read surfaces. MCP tool-calling is "break-glass" only when ROI is obvious.

**Decision 11 (2026-02-12):** Progressive close-out follows the model pipeline. *Superseded by SR-5 (Decision 13).*

**Decision 12 (2026-02-18):** Hub governance documents adopt Brain Stem v0.2.0 structural standard. *Promoted to CONTRACT (Hub) §12 SR-4.*

**Decision 13 (2026-02-19):** Notion AI is the reliable admin layer for workspace governance. *Promoted to CONTRACT (Hub) §12 SR-5.*

**Decision 14 (2026-02-19):** Tool Instruction Sets reframed as project-focused, model × project scoped. *Promoted to CONTRACT (Hub) §12 SR-6.*

**Decision 15 (2026-02-22):** Hub Steering Document split into CONTRACT (Hub) + operational steering doc. Binding rules, invariants, and boundaries extracted into CONTRACT (Hub) v1.0.0. Hub Steering renumbered to v3.0.0 as subordinate operational document. Change record: HUB-004.

---

## §7 Open Questions (Hub-Level)

**Q1:** Sessions DB canonical history vs pointer-first index (revisit in Chunk 5)

**Q2:** When to create official Airtable MCP integration vs rely on Make (trigger: official MCP OR ops > 6k/month)

**Q3:** Tool Instruction Sets in Notion vs mirrored to Git (revisit after first 3 project instruction sets + observed update frequency)

---

## §8 Active Conflicts

[Empty for now — populate when conflicts emerge]

---

## §9 Change Log (Append-Only)

**2026-02-22** | Notion AI: v3.0.0 — CONTRACT (Hub) v1.0.0 created. Hub Steering split: binding rules (old §1, §2, §6, §7, §8, §9, §10, §13, §15) extracted to CONTRACT; operational sections retained and renumbered §1–§12. Standing Rules (SR-1 through SR-6) extracted from Global Decisions. All downstream §-references updated. Change record: HUB-004.

**2026-02-19** | Notion AI (Opus 4.6): v2.2.0 — Decision 13 (Notion AI as reliable admin layer) and Decision 14 (project-focused tool instruction sets) recorded. §3 simplified, standardized output format dropped. §4 Tool Guidance reframed and wired to existing page. §5, §6, §9, §10, §11, §12, §16 updated for consistency. Decisions 6 and 11 marked superseded. §11 table empty column cleaned.

**2026-02-18** | Notion AI (Sonnet 4.5): v2.0.0 — Full structural rework to Brain Stem v0.2.0 standard. YAML header added. All sections §-numbered (§1–§21). §12 MCP condensed per Decision 10. §13 Change Control Protocol added. §14 renumbered, §15 linked to Dependency Registry. HOT/COLD boundary added. Decision 12 recorded. Change records: HUB-001 (Architecture Principles), HUB-002 (Hub Steering).

**2026-02-12** | Opus 4.6 + GPT 5.2: Architecture Principles page deployed. Dependency Registry created. Session Close-Out Prompt v1.2.1 deployed. Decision 11 recorded.

**2026-02-11** | Opus 4.6 + GPT 5.2: Decisions 7-9 added. Session Steering Doc consolidated - workspace-level content relocated to Hub Steering and Working Agreements. Workspace Prompt Library created. Parking Lot cleaned.

**2026-02-11** | GPT-5.2: Refined close-out prompts. Confirmed Git mirror reading requires raw URLs in this surface; updated Tool Reality + MCP Integration assumptions accordingly.

**2026-02-10** | Opus 4.6 + GPT 5.2: Created Hub Steering Document v1.0.

**2026-02-10** | GPT-5.2: v1.1 - Fixed Canonicality + Tool Routing to explicit Notion tables, added Table Interchange Protocol, clarified mirror boundary.

---

## §10 Next Steps (Hub-Level Priorities)

- Create Session Playbook - user-facing quick reference for session start/during/end workflow

- Reformat Brain Stem Steering Doc to Steering Spine v1

- Brain Stem Phase 1 implementation - Slack capture, Make webhook, route testing (thin session)

- Set up mirror for external model diversification

---

> ⚠️ **HOT / COLD BOUNDARY** — Content above this line is **HOT** (current priorities, active state, recent sessions). Content below is **COLD** (verification artifacts, full session history). Cold content is compacted at phase boundaries per CONTRACT (Hub) §10.

---

## §11 Verification Results (2026-02-11)

**Working Style section present?** Yes — now in CONTRACT (Hub) §7.

**Used across 2+ sessions?** Yes — Session History section exists and now records multiple entries (Sessions 1–4).

**Close-Out Processor wired?** Retired — close-out ritual superseded by Notion AI as reliable admin layer (SR-5 / Decision 13). Close-Out Processor v1.0 deleted.

---

## §12 Session History (Append-Only)

Append a new entry each session that materially changes hub-level decisions, tool reality, or governance.

**Template:**

```javascript
### YYYY-MM-DD — Session N — <short title>
- **What changed:** …
- **Evidence:** (links to Notion pages / commits / screenshots) …
- **Decisions recorded:** …
- **Next priorities:** …
```

### 2026-02-22 - Session 5 - Hub CONTRACT split

- **What changed:** CONTRACT (Hub) v1.0.0 created. Hub Steering restructured to v3.0.0 as subordinate operational doc. Binding rules extracted. All downstream §-references updated.

- **Evidence:** CONTRACT (Hub) page, HUB-004 change record, this document v3.0.0, Roadmap: Hub CONTRACT Split.

- **Decisions recorded:** Decision 15 (Hub Steering split).

- **Next priorities:** Mirror setup for external model diversification; Brain Stem Phase 1 implementation.

### 2026-02-18 - Session 4 - Hub governance docs rework to v0.2.0 standard

- **What changed:** Architecture Principles reworked to v1.0.0 (HUB-001). Hub Steering reworked to v2.0.0 (HUB-002). YAML headers, §-numbering, Change Control Protocol, HOT/COLD boundary, Decision 12 added.

- **Evidence:** HUB-001 and HUB-002 change records in Change Management DB.

- **Decisions recorded:** Decision 12 (adopt Brain Stem v0.2.0 structural standard for Hub docs).

- **Next priorities:** Dependency Registry rework (HUB-003); Phase C OpenClaw export layer design; Phase D AI instructions page.

### 2026-02-12 - Session 3 - Architecture Principles deployment + close-out protocol

- **What changed:** Architecture Principles page deployed; Dependency Registry created; Session Close-Out Prompt v1.2.1 deployed; Decision 11 recorded.

- **Evidence:** Architecture Principles page; Dependency Registry page; Session Close-Out Prompt v1.2.1 page.

- **Next priorities:** Session Playbook; Brain Stem steering spine reformat; Brain Stem Phase 1 implementation.

### 2026-02-11 - Session 2 - Close-out workflow hardening + mirror surface reality

- **What changed:** Close-out workflow tightened. Confirmed Git mirror is readable via raw URLs. Updated MCP assumptions and added Decision 10.

- **Evidence:** Public repos confirmed readable via raw URLs; Tool Reality section updated.

- **Decisions recorded:** Decision 10 (MCP not assumed viable for routine ops).

- **Next priorities:** Continue Brain Stem Phase 1; finish Brain Stem Steering cleanup.

### 2026-02-11 — Session 1 — Hub Steering patch + tool reality alignment

- **What changed:** Decision 6 added; Next Steps updated; Pointers links partially wired; Workspace Index entry created.

- **Evidence:** Workspace Index entry "Notion Hub Steering Document"; Close-Out Processor v1.0 page exists.

- **Decisions recorded:** Decision 6 (two-model workflow sufficient; revisit criteria defined).

- **Next priorities:** Test bounded write via MCP; wire remaining Pointers; run Hub Steering through a session refresh next session.

---

**End of Hub Steering Document v3.0.0**
