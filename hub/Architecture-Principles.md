# Architecture Principles

```yaml
---
document_id: "notion-hub-architecture-principles"
document_version: "1.0.0"
status: "active"
last_updated: "2026-02-18"
owner: "Jedidiah Duyf"
parent: "Notion Hub"
change_log_source: "Change Management DB"
---
```

---

## §1. Wide-Context Model Time as a Limiter

### Policy

Wide-context model sessions (loading multiple full documents) are a constrained resource — expensive in tokens, subject to usage caps, and observed to reach context ceiling after approximately 6 substantive turns with multiple full-document pastes. Design all protocols to minimize wide-context dependency.

### Session Types

Sessions declare their type at start:

- **THIN (default):** Implementation work, single-project focus. Load only hot sections of the relevant project steering doc + pointers to everything else. Target: under 200 lines loaded at session start, leaving ~80% of context window for actual work.

- **WIDE (explicit):** Architecture, cross-project synthesis, phase boundary reviews, monthly compaction. Load full steering docs, contracts, friction audits as needed. Reserve for periodic synthesis, not daily operations.

### Hot/Cold Boundary

Every steering doc must contain a literal marker:

- `--- HOT/COLD BOUNDARY ---`

**Hot sections (always loaded in thin sessions):** Implementation State, Active Decisions, Active Frictions, Next Session Priorities.

**Cold sections (load on demand):** Open Questions, Ruled Out, Session History, project-specific extensions.

### Escalation Rules

- When a thin session discovers it needs context from a cold section: load that specific section, not the full document. One or two cold section loads is normal — don't force escalation to WIDE for a single lookup.

- When a thin session needs content spanning multiple cold sections or multiple documents: park the item for a wide session rather than loading incrementally until the window fills.

- Models should ask for specific sections rather than requesting full documents when a section would suffice.

### Enforcement

These are norms enforced through working practice, not automated gates. The human coordinator determines session type. Models that notice context getting heavy should flag it rather than continuing silently.

---

## §2. Standardized Project Steering Doc Format

### Policy

Every project steering doc follows a fixed spine of required sections in a fixed order. This enables delta close-outs, model handoffs, cross-project queries, and automation without per-doc customization. Section names in the spine are frozen for the current version. The spine carries a version tag: Steering Spine v1. If spine headings are ever renamed, added, or removed, increment the version, document the migration path, and update all close-out prompts and patch automation that depend on heading matches. Projects on the old spine version must be migrated explicitly — do not assume they auto-update.

### Standard Spine

# [Project Name] Steering Document

Last Updated: [timestamp America/Vancouver]

Phase: [current phase]

## Pointers

URLs only. No narrative. Links to: Notion path, repos, related DBs, Claude Project, Friction DB, Parking Lot. One line per pointer, label + URL.

## Implementation State (HOT)

What is built and verified right now. Not plans. Each item includes a verification date and one evidence pointer (commit, PR, run ID, screenshot note, or Notion page link). Use <<UNKNOWN>> for unverified claims.

## Active Decisions (HOT)

Numbered, dated, one-line rationale. Append-only — supersede with "Superseded on YYYY-MM-DD by Decision X", never silently delete.

## Active Frictions (HOT)

Top 3-5 unresolved pain points linked to Friction DB by FR-ID. Each includes: workaround (current) and prevention (target fix).

## Next Session Priorities (HOT)

3 concrete items for next session.

--- HOT/COLD BOUNDARY ---

## Open Questions

Deferred decisions with explicit trigger conditions for when to revisit. Not a wish list — each entry states what would cause it to become active.

## Ruled Out

Rejected approaches. What, why, and date. Append-only — never silently delete.

## Session History

Append-only. Each entry: date, focus, decisions made, frictions encountered, artifacts produced, next priorities.

### Extension Rules

Projects may add custom sections below Session History (e.g., "Route Status", "Make Scenario Registry", "Airtable Schema Notes").

Extensions must not duplicate spine sections. If content could live in a spine section, it goes there. Extensions are for project-specific structures that have no spine equivalent.

Extensions do not affect close-out automation — patches only target spine sections by heading name.

### Pointers Discipline

The Pointers section is where bloat creeps in first. Rules:

- One line per pointer: label + URL

- No descriptions, no narrative, no status notes

- If a pointer needs context, that context lives in the linked page, not in Pointers

- Pointers are navigational only — they do not track state

---

## §3. Lightweight Dependency Registry

### Policy

Track only dependencies where a change in one artifact can break execution or invalidate governance. Everything else is navigational pointers — useful but not tracked as dependencies. A full dependency graph would become its own project; this registry is deliberately minimal.

### What to Track

Register a dependency when a change in the source artifact could cause one of these outcomes:

- An automation silently fails or produces wrong output

- A contract or spec becomes factually incorrect

- A governance artifact references something that no longer exists

- A schema change breaks downstream consumers

### Dependency Types

Distinguish between two break conditions:

**Breaks execution:** A change causes silent failure, wrong data, or blocked automation. These are urgent — must be checked before deploying changes to the source artifact. Examples: Airtable field rename breaks Make scenario mapping, route logic change invalidates scenario filters.

**Stales documentation:** A change makes a document factually outdated but nothing breaks operationally. These are important but not urgent — catch during next review cycle. Examples: steering doc references old phase status after transition, contract describes a field that was renamed but the automation was already updated.

### What NOT to Track

- Navigational pointers (steering doc links to Friction DB)

- Historical references (session history mentions a page)

- Informational relationships (Hub Steering mentions Brain Stem exists)

- Anything where a change in the source produces no operational or governance consequence

### Registry Format

A simple table, maintained as a dedicated page under Notion Hub. The canonical registry is the [Dependency Registry](https://www.notion.so/abb5799fc94548c4a0ea87e45fe82267) page; the table below is illustrative.

| Source Artifact | Depends On | Break Type | Break Condition | Last Verified |
| --- | --- | --- | --- | --- |
| Make PRO Route scenario | Airtable Projects table schema | Execution | Field rename or type change breaks extraction module | 2026-02-07 |
| contracts/spec.yaml | Airtable all tables | Documentation | Schema drift makes contract inaccurate | 2026-02-03 |
| Working Agreements (Promoted) | Friction DB entries by FR-ID | Documentation | Deleted or renumbered FR-ID breaks provenance | 2026-02-09 |
| Close-out patch automation | Steering doc section headings | Execution | Renamed heading causes patch to fail silently | 2026-02-11 |

### Maintenance Rules

Update the registry only when creating or modifying an automation, contract, or schema. This is not a ritual — it is triggered by change events.

If a dependency is discovered during debugging (something broke because of an untracked relationship), add it to the registry at that point. Just-in-time registration is the expected pattern.

Do not schedule periodic "dependency audits." If the registry is missing an entry, the next breakage will surface it.

### Enforcement

Norm, not automated gate. When making a change to a source artifact, check the registry for downstream dependencies. If break type is "execution," verify before deploying. If "documentation," flag for next review.

---

## §4. Delta-First Close-Outs with Two-Tier Cadence

> ⏸️ **Suspended** — Retained for reference. This protocol is on hold pending trial of direct Notion AI management as the primary doc-maintenance workflow (per SR-5 / Decision 13, 2026-02-19). Reactivate or evolve if the current approach proves insufficient.

### Policy

Default close-out produces a patch set, not a full document rewrite. This reduces model token cost, eliminates accidental content deletion, and makes each session's changes auditable. Full consolidation is reserved for phase boundaries and compaction.

### Tier 1: Delta Close-Out (default, every session)

Output format is a patch set organized by spine section heading:

PATCH: [Project Name] Steering Doc — [Date]

Implementation State:

ADD: "Description of new verified state (evidence pointer)"

Active Decisions:

ADD: "Decision N (YYYY-MM-DD): One-line decision with rationale."

Active Frictions:

UPDATE FR-YYYYMMDD-NNN: "Changed field — new value"

MOVE TO RESOLVED FR-YYYYMMDD-NNN: "Resolution summary"

Next Session Priorities:

REPLACE:

1. First priority

1. Second priority

1. Third priority

Session History:

APPEND:

YYYY-MM-DD — Session N — Short title

- What changed: ...

- Decisions: ...

- Frictions: ...

- Next: ...

### Patch Verbs

**ADD:** Insert new item into section. Position: end of section unless specified.

**UPDATE [identifier]:** Modify existing item in place. Identifier is decision number, FR-ID, or unique text snippet.

**MOVE TO RESOLVED [FR-ID]:** Remove from Active Frictions, note resolution. The friction still exists in the Friction DB — this verb moves it off the hot list, not out of history.

**REPLACE:** Swap entire section content. Use only for Next Session Priorities and Implementation State when most items changed.

**APPEND:** Add to end of section. Use for Session History (append-only sections).

No DELETE verb. Content is never silently removed from steering docs. To remove an active friction, use MOVE TO RESOLVED. To remove a decision, use UPDATE with "Superseded on YYYY-MM-DD by Decision X." Session History and Ruled Out are append-only — never modified after creation.

Patch scope rule: Delta patches may only target spine section headings (as defined in §2). Project-specific extension sections are not patchable via delta close-out — they are modified only during Tier 2 full consolidation or by direct manual edit. This keeps patch automation simple and prevents ad hoc patching of unstructured sections.

### Patch Application

Patches can be applied by:

- Notion AI: paste the patch set into a prompt targeting the steering doc page

- Human: manually edit the steering doc following the patch instructions

- MCP agent (future): programmatically parse and apply patch verbs

Same format works for all three executors. This is the interface standardization principle in practice.

### Tier 2: Full Consolidation (rare)

Triggers — any one of these:

- Phase boundary (e.g., Phase 0 → Phase 1)

- Monthly compaction per Hub Steering Doc rules

- Steering doc exceeds 400 lines

- Accumulated patches become hard to reason about (soft threshold: 5+ consecutive deltas without consolidation)

Full consolidation is a wide-context session that:

1. Reads entire steering doc + all pending patches

1. Produces a clean rewrite of the full document

1. Archives older session history entries to cold storage (keep last 3 in doc, rest in Sessions DB when available)

1. Resets to a clean baseline for the next delta cycle

### Close-Out Prompt Versioning

Session Close-Out Prompt v1.2.1 (deployed 2026-02-12) supports this architecture:

- Default output is patch format (Tier 1)

- MODE flag: DELTA (default) or FULL (consolidation)

- Patch verbs replace prose instructions for section updates

- FULL mode retains v1.1 behavior for consolidation sessions

### Enforcement

Norm. The session type (THIN/WIDE from §1) determines close-out tier: thin sessions produce delta patches, wide sessions may produce full consolidation. The human decides which tier to use if the session type was ambiguous.

---

## §5. MCP as Optimization Layer, Not Foundation

### Policy

Every protocol, contract, and governance artifact must work without MCP. MCP accelerates execution of existing protocols — it does not change what gets executed. Design for manual-first; benefit from automation later.

### Current Reality (as of 2026-02-11)

MCP tool-calling through Claude is functional but uneconomical for routine operations. Observed: 9 tool calls in a single Parking Lot write task consumed approximately one-third of a 5-hour usage allowance. Contributing factors included unnecessary broad searches when direct fetches would suffice, but even optimized usage remains expensive relative to manual Notion AI prompts.

GitHub MCP does not work in Claude Projects. ChatGPT MCP is unreliable in Projects. These constraints may change — do not design around them, but do not depend on them resolving either.

### Design Implications

Close-out patches: text artifacts a human applies via Notion AI, or an MCP agent applies programmatically. Same patch format, different executor.

Dependency registry: a Notion table a human reads, or an agent queries. Same data, different consumer.

Steering doc spine: fixed section headings so either a human or an agent can locate and patch the right section. Same structure, different accessor.

Edit lanes: documented rules a model self-enforces, or an MCP middleware could technically enforce. Same rules, different enforcement layer.

Automation contracts: typed inputs, typed outputs, idempotent operations. These properties make scenarios both Make-executable and MCP-exposable without redesign.

### MCP Readiness Criteria

Do not invest in MCP integration until all of the following are met for Notion (the critical platform):

- Tool-call costs drop below approximately 10% of equivalent manual workflow time cost

- Reliability exceeds 95% for the target platform

- A single session can execute 20+ targeted operations without hitting usage caps

- The external MCP agent concept (parked as someday in Parking Lot) becomes buildable with reasonable effort

These criteria are measurable. Reassess quarterly or when Anthropic announces pricing/capability changes.

### What "MCP-Shaped" Means Now

Without building MCP integrations, make every artifact ready for them:

- Clean inputs: every automation and prompt has explicitly typed inputs, not "whatever the model figures out from context"

- Clean outputs: structured, parseable results, not prose buried in narrative

- Deterministic operations: same input produces same outcome regardless of executor

- Idempotent where possible: running the same operation twice does not corrupt state

This is good engineering practice independent of MCP. It just happens to make MCP adoption frictionless when the economics arrive.

### Enforcement

Norm. No automated gates. The readiness criteria above are the decision trigger — when they're met, revisit. Until then, treat MCP as break-glass only for obvious high-ROI batch actions (e.g., seeding a database with 20+ entries in one operation).

---

## §6. Document Change Control

### Version Bump Rules

The `document_version` field in the YAML header follows semantic versioning:

- **Patch (0.0.x):** Typo fixes, clarifications, no policy changes. Downstream docs unaffected.

- **Minor (0.x.0):** New principle added, existing principle expanded — backward compatible. Downstream docs may need additive updates.

- **Major (x.0.0):** Principle removed or fundamentally changed, breaking change to downstream governance. All referencing docs must be reviewed.

### Change Record Mandate

No edit to this document lands without a corresponding record in the [🪦 Change Management (Deprecated)](https://www.notion.so/8ebdd4b6f0e34c74ace1840cd1ab11ca) DB.

- Change record must exist with Status = Proposed before the edit is applied.

- After the edit lands, update the change record to Complete.

- Use Change IDs prefixed `HUB-` for Hub-level governance changes.

### Impact Radius

Every change must declare affected downstream documents in the change record's Impact Assessment field. Architecture Principles changes may affect:

- Hub Steering Document (references AP by §-number)

- Dependency Registry (governed by §3 policies)

- Project steering docs (governed by §2 spine format)

- Close-out prompts (governed by §4 patch protocol)

### Principle Lifecycle

Individual principles may be added with status *candidate*. A candidate principle is promoted to *active* when:

- It has been applied in at least one project without unresolved friction

- No open objections exist

- The owner explicitly promotes it with a change record

---

## §7. Unifying Principle

These seven sections (five principles, one change control protocol, and this summary) converge on one design rule: standardize the interfaces, make everything else optional.

- The steering doc spine is an interface — models and humans both know where to find what.

- Patch sets are an interface — same format whether applied by Notion AI, Claude MCP, or manual editing.

- The dependency registry is an interface — same data whether queried by an agent or read by a human.

- The hot/cold boundary is an interface — mechanically determines what gets loaded, regardless of which model loads it.

- Edit lanes are an interface — same rules regardless of enforcement mechanism.

The system behind these interfaces — manual copy-paste, Notion AI prompts, MCP agent, Make scenarios — can change without touching the governance layer. That is what makes the architecture durable.

When evaluating any proposed change to the ecosystem, apply this test: does it change an interface, or does it change an implementation behind an interface? Interface changes require governance review (Lane C). Implementation changes behind stable interfaces are free to evolve (Lane A or B).

**Child page:** Edit Lanes & Mutation Control Model (Opus Review)
## High-level review
**What they got right**
- **Correct decomposition:** Hub-level (global coordination) vs Project-level (operational reality) vs Cross-cutting DBs (system memory) is the right mental model for durable memory without bloating context windows.
- **Good “shallow vs deep context” concept:** Always load a *small* boot set (Hub + relevant project steering). Pull deep context on demand (Frictions, Sessions, Index). That’s the only scalable way.
- **MCP framed as an enabler:** Reading current state on demand is genuinely transformative *if* you govern writes and promotion.
- **Gap analysis is directionally correct:** Hub Steering + Tool Guidance + Sessions DB are natural missing pieces.
**Where it’s weak / risky**
- It **under-specifies control of mutation**. “Models write updates back” is where systems degrade fast unless you define *write boundaries + review gates + provenance*.
- It assumes “set up MCP + GitHub MCP” is straightforward and beneficial by default. In practice you’ll want **read-first** and **propose-first** modes before “write to canonical.”
- It lightly contradicts v1.2 sequencing: deferring Slack capture until Hub exists is fine, but you *can* do both in parallel if you keep Slack capture **project-scoped** and minimal. Don’t let a hub doc block capture flow.
- It doesn’t define **what becomes durable** vs **what remains mutable** in a testable way. You need hard categories.
## Parts to dig into more deeply (priority order)
### 1) Mutation control model (the biggest missing mechanism)
You need an explicit policy that answers:
**What can an agent edit directly?**
- Session records, friction entries, draft proposals: usually **yes**
- Steering docs: **partial** (specific sections only)
- Contracts / working agreements / publish rules: **proposal-only** unless explicitly promoted
**How do edits become authoritative?**
- “Propose → human review → promote → mirror” (default)
- “Direct edit allowed” (rare; must be bounded)
**How do you keep provenance?**
- Every write includes: actor/model, timestamp, reason, link to session ID/friction ID
- Append-only change log at hub and project levels
This is the single point where MCP can either **solve** drift or **amplify** it.
**Deep-dive deliverable:** a one-page “Write Boundary Contract” + a table of allowed operations per artifact type.
---
### 2) Canonicality boundaries across Notion vs Airtable vs Git
Opus references the canonicality invariant but not the full version you’re converging toward:
- **Notion canonical:** operational memory + decisions + prompts + governance proposals
- **Airtable canonical:** structured pipeline records (entities, runs, outputs)
- **Git canonical:** promoted/stable artifacts (framework + project docs), enforced by CI
You need this written as **an invariant + enforcement rule**, otherwise you’ll end up duplicating “truth” across DBs.
**Deep-dive deliverable:** “Ownership & Canonicality” section added to Hub Steering + mirrored into framework CONTRACT.
---
### 3) Economy/scaling strategy: how you keep the boot set small
Opus proposes Hub + Project steering as the shallow set. Good. But you also need:
- **Hard size limits** for boot docs (e.g., “Hub Steering must stay under X lines; Project Steering under Y; everything else goes in DBs”)
- A **summary compaction ritual** (monthly/phase): older session detail collapses into a few durable decisions + links
- A “hot cache vs cold store” policy:
    - Hot: current priorities, active decisions, active frictions
    - Cold: full session history, resolved frictions, archived steering snapshots
Without these, your Hub Steering turns into another unbounded document.
**Deep-dive deliverable:** “Document Size & Compaction Rules” + an archive cadence.
---
### 4) Sessions DB design: what it’s for and how it stays lightweight
They say “planned” and “queryable history,” which is correct, but you should decide:
- Is Sessions DB canonical **history** or just **index pointers** to steering change logs?
- Does every session create a row, or only sessions that created decisions/frictions/artifacts?
Given your economy constraint, treat Sessions DB as **index + pointers**, not full narrative.
**Deep-dive deliverable:** Sessions DB schema tuned for “pointer-first” records + the close-out output format that populates it.
---
### 5) Tool Guidance docs: avoid turning into another documentation swamp
Tool guidance pages are helpful if they remain:
- Short
- Tied to observed frictions
- Structured like an API reference (inputs/outputs, do/don’t, escalation)
Otherwise they become “wiki pages nobody reads.”
**Deep-dive deliverable:** a template that forces:
- “What this tool is for”
- “Allowed edits / prohibited edits”
- “Default deliverable formats”
- “Known frictions (linked)”
- “Change log (append-only)”
---
### 6) MCP rollout plan: staged adoption (read-first → propose → bounded write)
This is where Opus is a bit too optimistic.
Recommended staging:
1. **Read-only MCP**: models can fetch current steering + DB entries
1. **Propose mode**: model writes proposals into a “Proposed Changes” area
1. **Bounded write**: model allowed to update specific fields/sections (e.g., add session entry, add friction row, update “Last Updated”), not rewrite full docs
1. **Automated promotion** only after stable rules + consistent provenance
**Deep-dive deliverable:** an MCP “capabilities ladder” + the allowed operations per rung.
---
## Quick critique of their “next steps” list
**Good:**
- Creating Hub Steering + Tool Guidance stubs is the right order.
- Deferring full mirror automation until export scope mapping is stable is aligned with your v1.2 readiness gates.
**Needs adjustment:**
- “Set up GitHub MCP with Claude” shouldn’t be framed as a default. You likely want GitHub changes still done through PR flows + CI; MCP can *draft* PR content, but pushing commits is a governance decision.
- Don’t defer Slack friction capture just because the hub doc doesn’t exist. You can start collecting frictions now; they’ll help populate hub/tool docs with real evidence.
## The parts we should dig into next (my recommendation)
If you want the minimum sequence that reduces drift immediately:
1. **Write Boundary Contract** (global + per-project)
1. **Hub Steering Doc** (boot file + append-only changelog)
1. **Tool Guidance template** (short, friction-linked)
1. **MCP staged rollout plan** (read-first → bounded write)
Everything else gets easier once those four are defined.
If you paste Opus’ actual Hub Steering draft (if it exists) or your current Notion tree state, I can mark exactly what’s missing and what to change, but the above is the “where to focus” based on the output you provided.

”I agree with your clear eyed commentary, and your assessment that an editing proveledges free-for all will be problematic. On the other side approval mechanisms could become a bottleneck. How to you envision managing that balance?”

You manage the balance by **not treating “approval” as one thing**. You define **edit classes** and give each class the lightest governance that still prevents drift.
Here’s the model that scales without becoming a bottleneck:
## 1) Classify edits into 4 lanes
### Lane A — Free write (low risk, high volume)
**Agents can write directly. No approval.**
- Session records (new rows)
- Friction captures (new rows)
- Parking Lot entries
- Status fields (“Mirror status”, “blocked”, “needs review”)
- “Last updated” timestamps
**Why:** These are *raw inputs* and operational telemetry. If they’re messy, you can clean later.
---
### Lane B — Bounded write (medium risk, frequent)
**Agents can edit, but only inside fenced sections / specific fields.**
- Steering docs: update only these sections:
    - Implementation State (must link to evidence)
    - Active Frictions (must link FR-IDs)
    - Next Session Priorities
    - Session History entry (append-only)
- Tool guidance: add “Known frictions” + “Recommended patterns” sections, not rewrite the whole page
**Mechanism:** enforce “allowed fields/sections” + append-only logs.
**Why:** This prevents an agent from rewriting your canon while still reducing manual work.
---
### Lane C — Propose-first (high risk, lower volume)
**Agents draft changes, humans approve.**
- Working Agreements (Promoted section)
- Contracts/specs/routes
- Publishing/security rules
- Repo boundary rules
**Mechanism:** agent writes to a “Proposed Changes” area (Notion) or opens a PR stub; you approve/merge.
**Why:** These are governance artifacts. One bad edit here creates systemic drift.
---
### Lane D — Automated enforcement (no approval, but hard gates)
**No one “approves” manually because the system enforces it.**
- GitHub PR checks (secret scan, publish-scope)
- Schema validation
- “Must include FR-ID to promote”
- “Must include evidence links to claim ‘verified’”
**Why:** Make correctness cheaper than review.
---
## 2) Make “approval” asynchronous and batched
Instead of approving each change:
- **Approve on a cadence** (daily 10 minutes, or end-of-session)
- **Approve by exception** (only items tagged “needs review”)
- **Bundle changes** into one PR or one Notion “Promotion Review” view
This turns approval into a **small ritual**, not a constant interruption.
---
## 3) Put a throttle on what *needs* approval
Two key rules stop the queue from exploding:
### Rule: “Promotions are scheduled, not ad hoc”
- Only promote during a weekly/phase review unless it’s **publishing safety**.
### Rule: “Safety escalations are immediate”
- Anything involving secrets/identity/publish scope gets instant review and immediate promoted rule/update.
---
## 4) Use evidence to reduce review load
Require agents to attach:
- FR-ID (or create one)
- Trigger string
- Proposed prevention
- Where it should live (framework vs project)
- Link to the affected artifact
Then your “approval” is mostly checking **fit + scope**, not re-deriving the situation.
---
## 5) Define one “maintainer” bottleneck—and keep it small
You are the maintainer. Keep your required approvals limited to:
- Lane C proposals
- Lane D gate failures (rare once tuned)
Everything else should flow without you.
---
## The practical rule of thumb
- **Raw memory**: write freely (Lane A)
- **Operational summaries**: write with fences (Lane B)
- **Governance**: propose-first + batch approval (Lane C)
- **Safety/correctness**: enforce with gates (Lane D)
If you want, I can turn this into a one-page **“Edit Privileges & Promotion Policy”** you can paste into the Hub Steering Doc and mirror into `mirror-framework` so every model/tool is constrained the same way.
