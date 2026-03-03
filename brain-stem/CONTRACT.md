<!-- Auto-generated from Notion. Do not edit directly. -->


```yaml
---
doc_id: "contract_brain_stem"
last_updated: "2026-02-22"
contract_version: "0.2.0"
parent_contract: "contract_hub"
---
```


**Status:** Live


### Metadata (operational)

- **Owner:** Jedidiah Duyf

- **Notion source:** [Brain Stem Contracts](https://www.notion.so/3f24933e570a4de58ee0c55c6be56775)

- **Mirror repo:** brainstem-docs

- **Implementation state:** phase_1

- **Last mirror sync:** 

- **Last QA run:** 

- **Last known good commit:** 

- **Last change:** v0.2.0 — §3 split, §3.5 interfaces added (formerly MOD-005)


---


## 1. Purpose


Brain Stem is a **capture, classification, and publishing system** that transforms Slack messages into structured records across multiple Airtable tables (People, Projects, Ideas, Admin, Events), enriches content via research APIs, synthesizes drafts, and publishes to multiple platforms.

**Core capability:** Touchless transformation from thought → structured data → published content.


---


## 2. Scope & Boundaries


**In scope:**

- Slack message capture (#brain-stem channel)

- Prefix-based routing (PRO, BD, CAL, R, fix)

- Claude-based classification and extraction

- Airtable record management

- Research enrichment via Perplexity

- Content synthesis and publishing


**Out of scope:**

- Account provisioning

- User authentication (uses existing Slack workspace)

- Direct database access (all via Airtable API)

- Real-time collaboration


**External dependencies:**

- Slack (ingress)

- [Make.com](http://make.com/) (orchestration)

- Claude API (intelligence)

- Perplexity API (research)

- Airtable (storage)


---


## 3. System Architecture


### 3a. Functional Pipeline


Brain Stem processes information through six abstract stages. Each stage defines *what* happens, not *how* or *who* provides it.

- **Capture** — Accepts unstructured input from a messaging source and produces a raw event record.

- **Classification** — Analyzes raw input text to determine destination entity and confidence score.

- **Extraction** — Derives structured fields from input text for the target entity type.

- **Enrichment** — Augments stored records with external research and contextual data.

- **Synthesis** — Combines captured records and research into publication-ready drafts.

- **Publishing** — Formats and delivers content to one or more output platforms.

- **Metrics** — Collects engagement and performance data from published outputs.


**Pipeline flow (provider-agnostic):**


```javascript
Capture → Classification → Extraction → Storage → Enrichment → Synthesis → Publishing → Metrics
```


*Note: Classification and Extraction may be performed in a single invocation. The pipeline stages are logical, not necessarily separate calls.*


### 3b. Current Provider Mapping


The following table maps each functional stage to its current provider. This mapping is **configuration, not architecture** — providers are swappable at each boundary.




**Trust boundaries (provider-specific):**

- Slack → [Make.com](http://make.com/): webhook + signing secret verification

- [Make.com](http://make.com/) → Claude / Perplexity: API key authentication

- [Make.com](http://make.com/) → Airtable: Personal Access Token

- Publishing channels: TBD per platform


---


## 3.5 Interface Definitions


Each boundary in the pipeline has a named interface that defines the data shape crossing it. These definitions are **provider-agnostic** — they describe what crosses the boundary, not which service implements it. For route-specific details and full LLM output schemas, see §7 (Route Semantics) and §8 (LLM Contracts).


### Capture Interface


**Boundary:** External messaging source → Pipeline entry

**Input (from capture source):**

- `text` (string, non-empty after trim) — raw message content

- `timestamp` (string) — source-native timestamp

- `channel` (string) — source channel or context identifier

- `user` (string) — source user identifier

- `thread_id` (string, optional) — thread or reply context


**Output (to Classification):**

- `original_text` (string) — preserved verbatim, immutable after capture

- `clean_text` (string) — prefix stripped, whitespace trimmed

- `detected_prefix` (string | null) — one of PRO, BD, CAL, R, fix, or null

- `captured_at` (ISO-8601 datetime)

- `source_metadata` (object) — channel, user, timestamp, thread_id


### Classification Interface


**Boundary:** Capture output → Intelligence layer

**Input:**

- `clean_text` (string) — prefix-stripped input


**Output:**

- `destination` (string) — one of: people, projects, ideas, admin, events, needs_review

- `confidence` (number, 0.0–1.0)

- `data` (object) — structured fields for the destination entity (see §8 for per-route schemas)

- `reason` (string) — explanation of classification decision


**Gating rule:** confidence ≥ 0.60 → auto-file; confidence < 0.60 → route to needs_review.

*Note: For prefix-routed messages (e.g., PRO:), destination is predetermined and classification is skipped. The Extraction Interface is used directly.*


### Extraction Interface


**Boundary:** Capture output → Intelligence layer (when destination is known)

**Input:**

- `clean_text` (string) — prefix-stripped input

- `destination` (string) — predetermined target entity type


**Output:**

- Structured fields matching the destination entity schema (see §8 for per-route output schemas)

- `reason` (string) — explanation of extraction decisions


*Note: Classification and Extraction may be performed in a single invocation. The BD route combines both; the PRO route uses Extraction only. These are logical interfaces, not necessarily separate calls.*


### Storage Interface


**Boundary:** Intelligence layer output → Persistent storage

**Input:**

- `destination` (string) — target table name

- `fields` (object) — structured fields matching the destination entity schema (see §9 for table definitions)

- `inbox_log_data` (object) — original_text, confidence, status, source_metadata, AI output raw
