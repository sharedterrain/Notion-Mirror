# If-else & Merge — Capability Report for Phase 3+

```yaml
---
doc_id: "ifelse_merge_report"
last_updated: "2026-03-10"
contract_version: "0.4.0"
---
```

**Purpose:** Inform Phase 3+ build decisions regarding [Make.com](http://make.com/)'s new If-else and Merge flow control modules. This report is intended as a reference for Sonnet-guided implementation sessions.

**Source:** [Make.com](http://make.com/)[ If-else and Merge documentation](https://www.make.com/en/help/if-else-and-merge) (Open Beta, last updated 2026-03-10)

**Contract References:** CONTRACT §3a (Functional Pipeline), §3b (Provider Mapping), §7 (Route Semantics), §13 (Implementation State)

---

## §1 Capability Summary

[Make.com](http://make.com/) has introduced two new Flow Control modules currently in **Open Beta**:

### If-else Module

- Splits a scenario into **conditional routes** — only the **first condition that evaluates as true** executes.

- If no conditions pass, an **Else route** runs as fallback.

- Unlimited condition routes can be added.

- Uses operations but **does not consume credits**.

- Conditions are evaluated top-down; structure from most specific to least specific.

### Merge Module

- Reconnects If-else routes into a **single downstream flow**.

- Passes data from the active route to subsequent modules via configurable **output name/value mappings**.

- Can only be used after an If-else module — not standalone.

- Uses operations but **does not consume credits**.

### Key Differences from Router

| **Attribute** | **If-else** | **Router** |
| --- | --- | --- |
| Execution | First true condition only | All routes, in order |
| Reconvergence | Yes, via Merge module | No — routes stay separate |
| Nesting | Cannot nest If-else inside If-else | Can nest Routers |
| Credit cost | Zero (operations only) | Zero (operations only) |

### Restrictions

- **No nesting:** Cannot place a Router or another If-else module inside an If-else flow.

- **Merge requires If-else:** The Merge module cannot be added without a preceding If-else.

- **Sequential only:** For multi-level branching, use sequential If-else → Merge → If-else → Merge chains.

---

## §2 Possible Uses in Brain Stem

The following maps If-else/Merge capabilities to Brain Stem's architecture. Ordered by impact.

### 2.1 Consolidate Duplicated Downstream Modules (High Impact)

**Current state:** 12+ duplicated module sets across primary and fix routes — each route has its own Inbox Log creation, Open Brain HTTP POST (`ingest-thought`), and Slack confirmation reply. Total: 24 routes including fix-delete and parked backup.

**With If-else/Merge:** After destination-specific logic (Airtable record creation, typed-field PATCHes), all routes merge back into:

- **One** Inbox Log creation module

- **One** Open Brain `ingest-thought` POST

- **One** Slack confirmation reply

The Merge module's output mapping passes route-specific values (destination, record_url, confidence, record_id) into these shared modules. Estimated reduction: **~60–70% fewer tail-end modules**.

### 2.2 Prefix Routing (Medium Impact)

**Current state:** Prefix detection (PRO:, BD:, CAL:, R:, fix:) likely uses a Router that evaluates all routes even though prefixes are mutually exclusive.

**With If-else:** Only the matching prefix route executes. Semantically correct and operationally leaner. Conditions ordered by expected frequency (BD: first, PRO: second, etc.).

### 2.3 Confidence Gating on BD Route (Medium Impact)

**Current state:** After Claude classification, confidence ≥ 0.60 auto-files; < 0.60 routes to Needs Review. These paths then diverge into separate downstream chains.

**With If-else/Merge:** Split on confidence threshold, handle divergent logic (auto-file vs. review queue), then Merge back for shared Inbox Log / Open Brain / Slack modules.

### 2.4 Phase 3 Research Relevance Scoring (Medium Impact — New Build)

**Architecture & Flows spec:** Research articles get a relevance score 0–100. Borderline scores (45–55) optionally trigger a Claude re-score.

**With If-else/Merge:**

- Condition 1: Score ≥ 55 → create Articles record directly

- Condition 2: Score 45–55 → Claude re-score → then create Articles record

- Else: Score < 45 → discard

- Merge back for shared record creation and Gallery placement

This is a **clean greenfield use case** — no refactoring required.

### 2.5 Fix Route Destination Routing (Lower Impact)

**Current state:** fix: handler runs across 5 destination routes (People, Projects, Ideas, Admin, Events) with separate typed-field PATCHes.

**With If-else/Merge:** If-else on the re-extracted destination → route to correct PATCH → Merge back for shared Inbox Log update (Status=Fixed), Slack confirmation, and Open Brain POST.

### 2.6 Degraded Mode Switching (Future — Phase 3+)

**Architecture spec:** When Claude is unavailable >1 hour, switch to PRO: prefix requirement (manual classification).

**With If-else:** A health-check variable or flag could gate between "normal classification" and "degraded manual-prefix mode" — then Merge back to the same downstream flow. Cleaner than current manual switchover.

---

## §3 Implementation Recommendations

### What to Build With If-else/Merge (Phase 3 Onward)

> ✅ **Recommended: Use If-else/Merge for all new Phase 3+ routes**

**Rationale:** New routes (R:, CAL:, publishing, metrics) don't exist yet. Building them on If-else/Merge from the start avoids the duplication pattern that inflated Phases 1–2. Zero refactoring cost.

**Specifically:**

1. **Research Flow (R: route):** Build the relevance scoring pipeline (§2.4) using If-else/Merge. Three conditions, one Merge, shared Articles record creation.

1. **Calendar Flow (CAL: route):** When scaffolding becomes implementation, use If-else for any event-type sub-routing, Merge for shared Events table write + Inbox Log.

1. **Any new prefix routes:** Add as conditions in a new If-else block rather than adding Router branches.

### What to Defer (Existing Phases 1–2)

> ⏸️ **Deferred: Do not refactor existing live routes until Beta exits**

**Rationale:**

- Phase 2 is live and stable (fix: handler operational, 12 Open Brain modules reconciled).

- Refactoring working infrastructure carries risk with no functional payoff.

- CONTRACT §15 (Change Control) requires updating all downstream docs in the same session — a full refactor touches Architecture & Flows, Phase docs, Invariants, and potentially the CONTRACT itself.

**Trigger to revisit:** When If-else/Merge exits Beta **and** a natural refactoring opportunity arises (e.g., a Phase 1–2 bug fix that touches routing modules, or a performance issue from Router evaluating all branches).

### Migration Path (When Ready)

When the time comes to migrate existing routes:

1. **Prefix routing first** — Replace the main Router with If-else. Lowest risk, highest clarity gain.

1. **Tail-end consolidation second** — Merge downstream Inbox Log / Open Brain / Slack modules. Highest module-count reduction.

1. **Confidence gating third** — Refactor BD route's threshold split to use If-else/Merge.

1. **Fix routes last** — Most complex, most conditional logic, most risk.

Each step should be a discrete, testable change with its own validation pass.

---

## §4 Risk Assessment — Open Beta Status

| **Risk** | **Severity** | **Likelihood** | **Mitigation** |
| --- | --- | --- | --- |
| **Breaking changes to If-else/Merge behavior during Beta** | High | Medium | Build new routes on If-else/Merge but keep existing Router-based routes untouched. If Beta breaks, only new routes need fixes — live system unaffected. |
| **Feature removed or significantly altered post-Beta** | High | Low | Make has invested in docs, tutorials, and UI integration — removal is unlikely. If altered, the sequential If-else → Merge pattern is simple enough to adapt. |
| **Nesting restriction blocks multi-level routing** | Medium | High (confirmed) | Use sequential If-else → Merge → If-else → Merge chains. Brain Stem's two-level routing (prefix → destination) maps cleanly to this pattern. |
| **Merge output mapping doesn't support complex objects** | Medium | Medium | Test with Brain Stem's actual data shapes (Inbox Log fields, Open Brain payload) before committing. Fall back to Set Variable modules if needed. |
| **Debugging difficulty with consolidated modules** | Low | Medium | Merged flows are harder to trace than duplicated-but-explicit routes. Mitigate with clear Merge output naming and Runs table logging. |
| **Credit/operation cost surprises** | Low | Low | Docs confirm zero credit cost. Monitor operations count post-implementation. |

### Overall Risk Posture

> 🟢 **Low risk for new builds, medium risk for refactoring existing routes.**

Using If-else/Merge for **new Phase 3+ routes** is low-risk because:

- No existing functionality is disturbed

- The feature's core behavior (conditional branching + reconvergence) is straightforward and well-documented

- If the feature changes, only new routes need updating

- The nesting restriction is a known constraint with a clean workaround

Refactoring **existing Phase 1–2 routes** carries medium risk due to Beta instability, change control overhead, and the debugging trade-off of consolidation.

---

## §5 Decision Summary

| **Decision** | **Action** | **Timing** |
| --- | --- | --- |
| New Phase 3+ routes | Build on If-else/Merge | Immediate — Phase 3 kickoff |
| Existing Phase 1–2 routes | Do not refactor | Deferred — revisit post-Beta |
| Merge output testing | Validate with Brain Stem data shapes before committing to consolidated modules | Early Phase 3 — before building shared downstream modules |
| Migration planning | Prefix routing → tail-end consolidation → confidence gating → fix routes | Post-Beta, opportunistic |

---

## Contract References

- **Pipeline architecture:** [CONTRACT (Brain-Stem)](https://www.notion.so/cb5393105c784cc3969571a898b4f81e) §3a, §3b

- **Route semantics:** CONTRACT §7

- **Implementation state:** CONTRACT §13

- **Change control:** CONTRACT §15

- **Architecture & Flows:** [Brain Stem Architecture & Flows](https://www.notion.so/8d45305a868d4e73a6555b9e96d53a18)

- **Source documentation:** [Untitled](https://www.notion.so/31f70cff90fc80b3afd2ce484c344c3e)
