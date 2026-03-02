<!-- Auto-generated from Notion. Do not edit directly. -->


```yaml
---
doc_id: "phase_0"
last_updated: "2026-02-21"
contract_version: "0.2.0"
---
```


**Status:** ✅ Complete

**Time Estimate:** 1 hour

**Dependencies:** None


---


## Contract References


This phase establishes infrastructure prerequisites for all pipeline interfaces (retroactive documentation — Phase 0 is complete).

**Implements:**

- **CONTRACT §3b: Current Provider Mapping** — establishes connections for all providers (Slack, [Make.com](http://make.com/), Claude, Perplexity, Airtable)

- **CONTRACT §4: Canonical Entities & Definitions** — establishes core tables (People, Projects, Ideas, Admin, Events, Inbox Log)

- **CONTRACT §5: Input Contracts** — prerequisites: Slack webhook endpoint, channel configuration

- **CONTRACT §9: Data Contracts** — prerequisites: Airtable base with 12 tables matching CONTRACT table schemas

- **CONTRACT §11: Security & Redaction Rules** — API keys and tokens registered as placeholders

- **CONTRACT §12: Observability** — [Make.com](http://make.com/) webhook verification

- **Invariants Implemented:**


See: [CONTRACT (Brain Stem)](https://www.notion.so/cb5393105c784cc3969571a898b4f81e) v0.2.0 | [contracts/spec (Brain Stem)](https://www.notion.so/98d781fe40ed4e31a566f0d8886325fc)


---


## Overview


This phase establishes the foundational infrastructure for the entire Brain Stem system. You'll create the Airtable data layer, set up the Slack bot for message capture, configure [Make.com](http://make.com/) as the orchestration engine, and collect all necessary API credentials.


### Prerequisites


Before starting Phase 0, ensure you have:

- **Airtable account** (free tier acceptable, Team plan recommended for AI features)

- **Slack workspace admin access** (ability to create apps and channels)

- [**Make.com**](http://make.com/)** account** (will create during Step 3)

- **Credit card for API access** (Claude API, Perplexity API)

- **Password manager** (for secure API key storage)


**What you're building:**

- 12-table Airtable base (data storage + UI)

- Slack app with bot permissions (capture interface)

- [Make.com](http://make.com/) webhook (event receiver)

- API access for Claude and Perplexity (intelligence layer)


---


## As-Designed: Implementation Steps


### [Airtable] Step 1: Create Airtable Base ✅


**Time:** 45-90 minutes (includes AI builder bypass) | **Status:** Complete


1.1 Create Base

- Create new base named "Brain Stem"

- Start with blank base (not AI template)


**⚠️ If Airtable AI builder auto-launches:**

- Dismiss the AI builder interface

- Access 'Grid View' directly from base menu

- Manually create tables one-by-one (AI builder cannot replicate this specific schema)


1.2-1.8 Create 12 Tables


**Field Configuration Notes:**

- **Date fields with time:** Use "Date" type with "Include time" enabled (not "Created time" or "Last modified time" auto-generated fields)

- **Link fields:** Name singular for single-link ("Research Job"), plural for multi-link ("Source Articles")

- **Link configuration:** Works via Record IDs automatically - field names don't need to match between tables


**People Table:**

- Name (Title)

- Context (Long text)

- Follow-ups (Long text)

- Last Touched (Date with time)

- Tags (Multiple select)


**Projects Table:**

- Name (Title)

- Type (Single select: Digital, Physical, Hybrid)

- Status (Single select: Active, Waiting, Blocked, Someday, Done)

- Next Action (Long text)

- Notes (Long text)

- Last Touched (Date with time)

- Tags (Multiple select)


**Ideas Table:**

- Name (Title)

- One-Liner (Long text)

- Notes (Long text)

- Last Touched (Date with time)

- Tags (Multiple select)


**Admin Table:**

- Name (Title)

- Due Date (Date)

- Status (Single select: Todo, Done)

- Notes (Long text)

- Created (Date with time)


**Events Table:**

- Name (Title)

- Event Type (Single select: Meeting, Deadline, Appointment)

- Start Time (Date with time)

- End Time (Date with time)

- Attendees (Link to People, multiple)

- Location (Single line text)

- Notes (Long text)

- Calendar Sync Status (Single select: Synced, Pending, Manual)

- Created (Date with time)


**Inbox Log Table:**

- Original Text (Title)

- Filed To (Single select: People, Projects, Ideas, Admin, Events, Needs Review)

- Original Destination (Single select: People, Projects, Ideas, Admin, Events)

- Corrected Destination (Single select: People, Projects, Ideas, Admin, Events)

- Destination Name (Single line text)

- Destination URL (URL)

- Confidence (Number, 0-1 decimal format)

- Status (Single select: Processing, Filed, Needs Review, Fixed)

- Created (Date with time)

- Slack Channel (Single line text)

- Slack Message TS (Single line text)

- Slack Thread TS (Single line text)


**Research Jobs Table:**

- Query (Title)

- Domains (Multiple select)

- Recency (Single select: 24h, 3d, 1w, 1m)
