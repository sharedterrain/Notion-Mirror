# CONTRACT (Open-Claw Deployment)

```yaml
---
doc_id: "contract_openclaw_deployment"
contract_version: "0.4.0"
parent_contract: "contract_hub"
last_updated: "2026-03-07"
owner: "Jedidiah Duyf"
created: "2026-02-27"
---
```

**Status:** Draft

---

## §1 Purpose & Scope

This contract governs **Magi** — a local OpenClaw agent deployment operating as the execution layer in a modular, open-stack automation system. It defines the durable rules, invariants, and boundaries for the OpenClaw platform itself, independent of any specific project Magi executes.

**What this contract governs:**

- OpenClaw runtime configuration, update discipline, and model routing

- Messaging channel integration and exec approval policies

- Hardware and infrastructure boundaries

- Security posture for the local execution environment

- Memory architecture and session continuity

- Scaling philosophy and investment triggers

**What this contract does NOT govern:**

- Project-specific deliverables (e.g., [livingsystems.earth](http://livingsystems.earth/) has its own contract)

- Brain Stem pipeline architecture (governed by CONTRACT (Brain Stem))

- Workspace-wide governance (governed by CONTRACT (Hub))

**Design principle:** This contract describes the *platform* — the reusable execution capability. Projects that Magi serves are downstream consumers of this platform and are governed by their own contracts.

---

## §2 Architecture: Open Service Stack

### §2.1 Design Principles

Magi operates within a **modular, open-stack architecture**. The system is not defined by a fixed number of actors or a fixed topology — it is defined by **open interfaces between replaceable services.** The stack grows as new services connect; nothing in this contract limits the number or type of services Magi can consume.

**Invariant:** All services connect through **open protocols** — MCP, REST APIs, webhooks, CLI, CDP. No proprietary integrations. If a service requires a closed protocol, that is a defect in the integration, not a trade-off to accept.

**Invariant:** Interface boundaries are the architecture. Providers are configuration. Every service is replaceable at its boundary — the integration point is durable, the provider behind it is not. Services can be added, replaced, or removed without architectural change.

**Invariant:** Magi does not write directly to governance documents. Governance lives in Notion and is maintained by the human operator. However, **Magi's observations, recommendations, friction reports, and status updates are valued and expected inputs to governance decisions.** Magi is not a passive executor — Magi's perspective actively shapes the system. The messaging channel is Magi's voice in governance.

### §2.2 Service Registry

Each service Magi connects to is registered by its **role, current provider, interface, and replacement path.** This registry is configuration — adding or removing a service does not require a contract change.

| **Role** | **Current Provider** | **Interface** | **Replacement Path** |
| --- | --- | --- | --- |
| Governance | Notion workspace | Human-maintained pages; mirrored to GitHub (Notion-Mirror repo, public read-only) for Magi read access | Any structured doc system with export capability |
| Execution | OpenClaw on Metacarcinus | Local agent runtime with approval gating | Any agent runtime supporting tool-use and exec approvals |
| Orchestration | [Make.com](http://make.com/) | Webhooks, HTTP modules | n8n (migration planned), or any webhook-capable automation platform |
| Shared Memory | Open Brain (Supabase) | REST via Supabase edge functions (bidirectional: search + ingest). Skill-based integration (`~/.openclaw/workspace/skills/open-brain/SKILL.md`) instructs Magi to call endpoints via `curl`. Contains Brain Stem captures, Magi insights, manual entries. | Any vector store with HTTP API |
| Operational Memory | Magi Brain (Supabase) | REST via Supabase edge functions (bidirectional: search + ingest). Skill-based integration (`~/.openclaw/workspace/skills/magi-brain/SKILL.md`). Magi's dedicated cross-session operational memory — learned patterns, infrastructure knowledge, resolved problems. | Any vector store with HTTP API |
| Pipeline Storage | Airtable | REST API (via Make scenarios and direct API) | Any structured data store with API access |
| Model Routing | OpenRouter | Unified API gateway | Any multi-model API gateway |
| Capture | Slack (#magination) | Bidirectional messaging, approval workflows | Any messaging platform with bot API and thread support |
| Surfacing | Chrome (browser tool) | CDP-based browsing | Any headless browser with CDP support |
| Publishing | GitHub: Notion-Mirror (public, sanitized), magis-workshop (private, full context); [livingsystems.earth](http://livingsystems.earth/) | Git commits via GitHub Actions; Magi has read/write access to magis-workshop-repo via SSH deploy key (`id_ed25519_github`); rsync/SFTP for [livingsystems.earth](http://livingsystems.earth/) | Any static hosting with CI/CD or file transfer |

### §2.3 Magi's Execution Model

Magi operates on a **capability-based** model with broad autonomous execution (see §7 for full policy):

1. **Scheduled work:** Cron-triggered tasks execute autonomously

1. **Commanded work:** Tasks initiated via messaging channel command

1. **Free-running work:** Most operations (shell, git, file, browser, memory) execute without approval gating — mistakes are self-repairable and the cost is tokens, not damage

1. **Hard-gated work:** Production deploys, OpenClaw config changes, and credential operations require explicit human approval via messaging channel

1. **Reporting:** All task outcomes reported to messaging channel — Magi never executes silently

**Session-start ritual:** At the beginning of every session, Magi follows a prescribed sequence (encoded in [AGENTS.md](http://agents.md/) §Memory "Session Start Ritual"):

1. Read `SOUL.md` (persona and values)

1. Read `USER.md` (Jedidiah's context)

1. Read `memory/YYYY-MM-DD.md` (today + yesterday's operational logs)

1. **Main session only:** Read `MEMORY.md` (curated long-term state)

1. Query Open Brain for context relevant to this session's topic

1. Query Magi Brain for Magi's own relevant operational context

This ritual ensures continuity across sessions despite Magi's stateless runtime. Magi selectively stores decisions, action items, and insights to memory during sessions — not casual conversation or troubleshooting chatter.

**Capability surface:** Magi's capabilities are a function of connected services. Magi with Open Brain has semantic memory. Magi with a browser tool has web surfacing. Magi with Airtable access has pipeline data. The execution model stays constant — the capability surface grows as services connect.

---

## §3 Runtime & Infrastructure

### §3.1 Current Configuration

| **Component** | **Current Value** | **Notes** |
| --- | --- | --- |
| Host machine | Metacarcinus (M1 MacBook Air, 8GB) | Local network at 10.0.0.102 |
| OpenClaw version | v2026.3.2 | Updated Mar 7. Post-update: exec settings reset to `security=full, ask=off` per CD-015/CD-017. |
| Gateway | Running, loopback 127.0.0.1:18789 | PID active |
| Slack scopes | Full 21-scope manifest | All resolved; no missing_scope errors |
| Control UI | Accessible on Metacarcinus | Token auth, loopback only |
| Heartbeat | 30m interval, firing on schedule | Model: Haiku (via OpenRouter) |
| Energy Saver | Sleep disabled | Always-on for heartbeat persistence |
| Git repos | Two repos in workspace | Notion-Mirror (public read-only), magis-workshop-repo (private read/write) |
| Browser tool | Chrome at `/Applications/Google Chrome.app` | CDP enabled on port 18792 |
| Screensaver | Disabled (Flurry) | Root cause of kernel panic; disabled per CD-019 |
| Remote access | Screen Sharing over SSH tunnel from M5 | GUI access for monitoring; established per CD-020 |

### §3.2 Scaling Philosophy

**Invariant:** Hardware is not a constraint on architectural decisions. The current M1 Air deployment is a low-risk starting point, not a ceiling. If OpenClaw demonstrates sufficient value, additional resources (dedicated hardware, cloud deployment, upgraded machine) are available.

**Invariant:** Time-to-capability is prioritized over cost optimization. Getting set up and generating revenue changes the monthly cost calculus. Architectural decisions should favor speed of deployment and automatability over marginal cost savings.

**Scaling triggers** (any one is sufficient):

- Sustained thermal throttling interferes with task completion

- Multiple projects compete for Magi's execution time

- A commercial engagement requires higher availability than a local machine provides

- OpenClaw capabilities warrant dedicated infrastructure (e.g., always-on server)

**Scaling path:** Metacarcinus (current) → dedicated local server OR cloud VM → managed deployment. Each transition preserves the service architecture — only the Magi provider changes.

---

## §4 Model Routing

### §4.1 Unified Gateway

**Invariant:** All models are routed through **OpenRouter** as the unified gateway. No direct provider API keys are configured for model access.

**Rationale:** Provider flexibility — the ability to switch models without reconfiguring provider auth — is worth the ~5.5% OpenRouter markup. This is accepted as a permanent architectural choice, not a temporary convenience.

### §4.2 Task-Specific Model Selection

**Invariant:** Model selection is task-driven, not provider-driven. The routing table maps task types to models based on demonstrated capability, not brand loyalty.

**Current routing table:**

| **Task Type** | **Model (via OpenRouter)** | **Selection Rationale** |
| --- | --- | --- |
| Heartbeat | Haiku (default) | Cheapest; switch to Flash only if testing shows better reliability |
| Code generation | Gemini 3.1 Pro | Best cost/reasoning ratio for generation tasks |
| Research and content drafting | Gemini 3.1 Pro | 1M context window + web grounding |
| Routine cron maintenance | Gemini 3 Flash | Ultra-cheap for low-complexity tasks |
| Tool-use-heavy tasks (exec ops, approval workflows) | Sonnet 4.5 | OpenClaw's tool-use layer built for Anthropic; best tool compliance |
| Complex architectural decisions | Sonnet 4.5 | First-class support in OpenClaw's agent runtime |
| Perspective reset / unblocking | Opus 4.6 | High-level reasoning when other models are stuck; not for routine work. Cost justified only when cheaper models have failed to resolve the issue. |

---

## §5 Messaging Channel

### §5.1 Channel-Agnostic Principle

**Invariant:** Magi's approval workflows, notifications, and command interface are not locked to any specific messaging platform. Slack, Telegram, Discord, and other OpenClaw-supported channels are equally valid. The specific channel is a configuration choice, not an architectural decision.

### §5.2 Current Configuration

- **Active channel:** Slack ([LivingSystems.Earth](http://livingsystems.earth/) workspace, #magination, channel ID: C0AGNFVRKFA)

- **Status:** Current default — not the permanent choice

- **Selection timing:** Channel selection is a Phase 0 task for each new project deployment; may be revisited at any time

### §5.3 Channel Requirements

Any messaging channel used with Magi must support:

- Bidirectional messaging (commands in, reports out)

- Approval workflows (Magi requests approval, human grants/denies)

- Notification delivery with reasonable latency

- Thread or reply context for multi-step operations

---

## §6 Update Discipline

### §6.1 Pin and Wait Principle

**Invariant:** OpenClaw updates follow the **pin and wait** discipline:

1. When a new OpenClaw release ships, **do not update immediately**

1. Wait 3–5 days for community regression reports

1. Check GitHub issues for the target version before updating

1. Test in an isolated session before committing to the update

1. If regressions are found, pin to the current stable version until fixed

### §6.2 Version Tracking

The current OpenClaw version is tracked in the Magi Status & State page. After any update:

- Record the new version, date, and any observed issues

- **Reset **`tools.exec.ask`** and **`tools.exec.security` in both `openclaw.json` and `exec-approvals.json` — updates reset these values. Current target: `security=full, ask=off`

- Verify heartbeat model bleed bug (#22133) status

- Test tool-use operations with both Sonnet and Gemini through OpenRouter

- Confirm exec approval workflows function correctly

---

## §7 Exec Policy: Capability-Based Model

### §7.1 Design Principle

**Invariant:** Magi operates with broad autonomous execution capability. The purpose of a long leash is to enable Magi to do real work — building, researching, committing, repairing — without human approval gating every action. An agent that must ask permission to write a file cannot build a site.

**Invariant:** The acceptance of autonomous mistakes is proportional to the value of autonomous work. The system is designed to produce substantially more value than the cost of any errors it creates. If Magi is generating thousands of dollars of equivalent work, occasional mistakes that cost tokens to detect and repair are an acceptable trade-off — not a failure of the system.

**Consequence ceiling:** The maximum tolerable blast radius for a single autonomous mistake — including detection and repair — is approximately $100. This is a design constraint evaluated by the human operator, not a spending gate that Magi checks before acting. If the value-to-error ratio degrades (low-value work with high mistake costs), the approval surface should be tightened. The ceiling is a function of output value, not a standalone budget.

### §7.2 Hard Gates

These operations always require explicit human approval via the messaging channel, regardless of trust level:

| **Operation Type** | **Rationale** |
| --- | --- |
| Production deploys (rsync/SFTP to live hosting) | Public-facing, hard to reverse. Always dry-run first, post summary to #magination, wait for approval. |
| OpenClaw config changes (~/.openclaw/) | A misconfigured agent can't fix itself. |
| Credential rotation or provisioning | Security boundary — non-negotiable human oversight. |

### §7.3 Free-Running Operations

All other operations execute without approval gating:

- Shell execution (file reads, writes, builds, scripts)

- Git operations (commit, push, branch, merge)

- Web search and fetch

- Browser automation via the managed openclaw profile

- Memory read/write (Open Brain, Magi Brain)

- Workspace file management

These operations are self-repairable: a bad commit can be reverted, a broken build can be re-run, a wrong file can be deleted. The cost of a mistake is tokens, not permanent damage.

### §7.4 Novel Command Discipline

With `tools.exec.ask=off`, OpenClaw does not enforce platform-level approval prompts. Hard gates (§7.2) are enforced by Magi's own behavioral discipline per [AGENTS.md](http://agents.md/), not by the exec approval system. For genuinely novel or unfamiliar commands, Magi should check in via #magination before first execution — this is expected professional judgment, not a platform gate. The approval surface narrows naturally over time as Magi's operational repertoire grows. This is intentional, not drift.

### §7.5 Tightening the Leash

If the value-to-error ratio degrades, the operator may tighten the approval surface by moving operation categories from §7.3 to §7.2. This is logged in Configuration Decisions. The default posture is broad autonomy; restrictions are the exception, not the starting point.

---

## §8 Security Posture

⚠️ **STALE — pending v1.0 rewrite via V3 architecture sessions.** The invariants below remain in force. Specific access mechanisms (zsh subshell wrapping, `~/.zshenv` keychain loading) were written for the pre-v2026.3.2 exec environment and must be re-verified post-update. The exec settings reset (§6.2) may have changed the shell environment behavior.

**Invariant:** The `<<PLACEHOLDER>>` pattern from CONTRACT (Hub) §11 applies to all Magi configuration. Real secrets (API keys, SSH keys, tokens) are never stored in Notion pages.

**Invariant:** Brain keys are stored in macOS keychain, never in bootstrap files, environment variable exports, or Notion. The specific mechanism for loading them into the exec environment is an implementation detail that must be verified against the current OpenClaw version.

**Magi-specific security rules:**

- **OpenRouter API key:** stored in OpenClaw's local config on Metacarcinus, never in Notion

- **SSH keys for GitHub:** stored in `~/.ssh` on Metacarcinus, never in Notion

- **magis-workshop-repo deploy key:** `id_ed25519_github`, read/write access, scoped to magis-workshop-repo only, private key on Metacarcinus

- **Hosting credentials (WHC SFTP/SSH):** stored locally on Metacarcinus, never in Notion

- **Open Brain API key (**`OPEN_BRAIN_KEY`**):** stored in macOS keychain on Metacarcinus, passed as `x-brain-key` header to Supabase edge functions, never in Notion. Access mechanism: previously loaded via `~/.zshenv` into zsh subshells only — **verify post v2026.3.2 update**

- **Magi Brain API key (**`MAGI_BRAIN_KEY`**):** stored in macOS keychain on Metacarcinus, passed as `x-brain-key` header to Supabase edge functions, never in Notion. Access mechanism: previously loaded via `~/.zshenv` into zsh subshells only — **verify post v2026.3.2 update**

- **Messaging channel tokens:** managed by OpenClaw's channel integration, never in Notion

**Brain access mechanism (needs re-verification):**

Pre-v2026.3.2, both `OPEN_BRAIN_KEY` and `MAGI_BRAIN_KEY` were loaded via `~/.zshenv` keychain lookup and available only in zsh subshells, requiring `/bin/zsh -c '...'` wrapping for all brain curl commands. Post-update, exec settings were reset to `security=full, ask=off` (see §6.2) — the shell environment behavior may have changed. Before relying on the previous access pattern, verify:

1. Whether `~/.zshenv` is still sourced in the current exec environment

1. Whether brain keys are accessible in the base exec environment or still require zsh subshell wrapping

1. Update [AGENTS.md](http://agents.md/) §Memory curl templates to match the verified mechanism

**Security posture tracking:** See Security Posture page under Magi project space for current state.

---

## §9 Primary Design Values

These values govern architectural trade-offs for the OpenClaw deployment:

1. **Automatability and efficiency first.** Prefer solutions that reduce manual intervention and increase autonomous capability. Manual steps are acceptable during bootstrap but should be automated as soon as practical.

1. **Portability for future migrations.** Every architectural choice should allow clean migration to a different provider, platform, or host. Avoid vendor lock-in at every layer. If a component creates migration friction, that friction is a defect to be logged and addressed.

1. **Time-to-capability over cost optimization.** Speed of getting set up and operational is more valuable than marginal cost savings. Revenue generation changes the cost calculus — optimize for getting there faster.

1. **Investment scales with demonstrated value.** Start minimal and low-risk. As OpenClaw proves its value through delivered projects, invest proportionally — better hardware, dedicated infrastructure, expanded capabilities.

---

## §10 Downstream Projects

Magi serves as the execution platform for project-specific work. Each project that uses Magi:

- Has its own contract governing project-specific decisions

- References this contract for platform-level rules

- May not contradict this contract; if a project need conflicts with a platform rule, the conflict is escalated and resolved here

**Current downstream projects:**

- [livingsystems.earth](http://livingsystems.earth/) (governed by [CONTRACT (livingsystems.earth)](https://www.notion.so/35560bdc09cf4e78bdf4ea9466ea9f91))

- Brain Stem integration (via Make; governed by CONTRACT (Brain Stem))

---

## §11 Change Control

**Version bump rules:**

- **Patch (0.0.x):** Configuration updates (routing table, current values), clarifications

- **Minor (0.x.0):** New sections, new standing rules, new capability boundaries

- **Major (x.0.0):** Architectural changes to the service architecture, changes to invariants

**Change tracking:** This contract is its own source of truth for change history. The `contract_version` in the YAML header tracks the current version. Notion's page version history provides the full diff timeline. Significant deviations or decisions are logged in §12 below.

**Downstream reconciliation:** Per CONTRACT (Hub) §2, when this contract is updated, downstream documents in the impact radius must be reconciled in the same work session or carry a staleness banner: `⚠️ STALE — see [version]`.

---

## §12 Deviation Log

*Tracks significant deviations, decisions, and rationale that led to contract changes. Not every patch-level edit needs an entry — only changes where the "why" matters for future reference.*

| Version | Date | Description |
| --- | --- | --- |
| 0.1.0 | 2026-02-27 | Initial draft. Codifies three-actor architecture, OpenRouter unified gateway, channel-agnostic messaging, exec approval tiers, scaling philosophy, and design values from Feb 27 advisory session. |
| 0.1.1 | 2026-02-28 | Phase 0 complete. §3.1 updated with v2026.2.23 runtime state (gateway, Slack scopes, Control UI, heartbeat, Energy Saver). §5.2 channel renamed to #magination. All configuration changes logged in CD-001–CD-012. |
| 0.2.0 | 2026-03-03 | §2 rewritten: replaced fixed "three-actor system" with open service stack architecture. Added service registry pattern (9 roles). Made explicit that Magi's input to governance is valued. Reflects current topology including Open Brain, Airtable, browser tool. All "three-actor" references updated throughout. |
| 0.2.1 | 2026-03-03 | Post-mirror build state report reconciliation. §2.2 Publishing row updated to reflect two-repo model (Notion-Mirror public, magis-workshop private) and Magi SSH deploy key access. §8 updated with magis-workshop deploy key entry. |
| 0.2.2 | 2026-03-04 | Open Brain integration reconciliation. §2.2 Memory row corrected: interface is REST via Supabase edge functions (not MCP), bidirectional (search + ingest), skill-based integration pattern documented. §2.3 added session-start context search behavior and selective memory storage policy. §8 added OPEN_BRAIN_KEY to security rules. Configuration details logged in CD-014. |
| 0.3.0 | 2026-03-07 | §7 rewritten: replaced tiered exec approval model with capability-based model. Magi now operates with broad autonomous execution — shell, git, file ops, browser, memory all run freely. Hard gates retained only for production deploys, OpenClaw config changes, and credential operations. Added proportionality principle: freedom to fail is justified by the ratio of value produced to damage possible, with ~$100 consequence ceiling as a design constraint (not a spending gate). Novel-command ramp-up codified as intentional trust expansion. §2.3 updated to reflect new exec model. Reconciles CONTRACT with [AGENTS.md](http://agents.md/) operational reality. |
| 0.4.0 | 2026-03-07 | Merged Magi's self-revision. §1 scope expanded to include memory architecture. §2.2 service registry split Memory into Shared Memory (Open Brain) and Operational Memory (Magi Brain); Governance row names Notion-Mirror repo; Publishing row corrected to read/write with SSH key name. §2.3 session-start ritual expanded to 6-step sequence including both brains. §3.1 updated: OpenClaw v2026.3.2, browser tool CDP config, screensaver disabled (CD-019), remote access via SSH tunnel (CD-020). §6.2 added post-update exec settings reset discipline. §7.4 reconciled with ask=off reality — renamed to Novel Command Discipline, behavioral enforcement not platform-gated. §8 rewritten: corrected key storage (macOS keychain not env var), added Magi Brain key, documented brain access hardening discipline (zsh subshell requirement). |

---

## §13 Related Documents

- [CONTRACT (Hub)](https://www.notion.so/c18af9cbec3b4c388c3561036a4871f1) — parent contract

- [Configuration Decisions](https://www.notion.so/a5354158620348de908d5ffb31e35036) — decision log for routing, channel, and config changes

- [AGENTS.md - Magi's Workspace](https://www.notion.so/6ec71b9a997f41e7b6cf6613861c5f5d) — operational guidance and session startup ritual

- [livingsystems.earth — Site Vision & Product Definition](https://www.notion.so/f11a671681874fea820c6d352104e311) — first downstream project strategy

---

**End of CONTRACT (OpenClaw Deployment) v0.4.0**
