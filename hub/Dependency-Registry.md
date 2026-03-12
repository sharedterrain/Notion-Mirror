# Dependency Registry

```yaml
---
document_id: "notion-hub-dependency-registry"
document_version: "1.0.0"
status: "active"
last_updated: "2026-02-18"
owner: "Jedidiah Duyf"
created: "2026-02-12"
parent: "Notion Hub"
change_log_source: "Change Management DB"
---
```

**Spine:** Tracks operations-critical dependencies only. See Architecture Principles §3 for policy.

---

## §1 Registry

| Source Artifact | Depends On | Break Type | Break Condition | Owner/Verifier | Last Verified |
| --- | --- | --- | --- | --- | --- |
| Delta close-out patches (v1.2) | Hub Steering §-numbered section headings (§1–§12 as of v3.0.0) | Execution | Renamed or removed heading causes patch to target wrong section or fail silently. **Note:** Hub Steering renumbered to §1–§12 in v3.0.0 (HUB-004). Contract sections extracted to CONTRACT (Hub). | Jedidiah | 2026-02-22 |
| CONTRACT (Hub) §-numbered section headings | Downstream docs referencing CONTRACT §-numbers | Execution | Renamed or removed heading breaks cross-references in Hub Steering, Tool Instruction Sets, AI Instructions, Architecture Principles, and any future docs citing CONTRACT §-numbers. | Jedidiah | 2026-02-22 |
| Session Playbook (planned) | Steering Spine v1 hot/cold boundary | Execution | Changed boundary definition makes playbook loading instructions wrong | Jedidiah | not yet created |
| Session Playbook (planned) | Session Close-Out Prompt version | Documentation | Playbook references v1.2 features that change in future versions | Jedidiah | not yet created |
| Working Agreements (Promoted rules) | Friction DB entries by FR-ID | Documentation | Deleted or renumbered FR-ID breaks provenance chain | Jedidiah | 2026-02-12 |
| Architecture Principles §4 (patch scope rule) | Architecture Principles §2 (spine definition) | Execution | Spine heading changes invalidate patch targeting rule. AP restructured to v1.0.0 with §1–§7; §2 and §4 confirmed intact. | Jedidiah | 2026-02-18 |

## §2 Maintenance Rules

Entries are added when creating or modifying automations, contracts, schemas, or parseable artifacts. Not a scheduled ritual - triggered by change events.

If a dependency is discovered during debugging, add it at that point. Just-in-time registration is the expected pattern.

Break Type definitions:

- **Execution:** change causes silent failure, wrong data, or blocked automation. Check before deploying changes to source artifact.

- **Documentation:** change makes a document factually outdated but nothing breaks operationally. Catch during next review cycle.
