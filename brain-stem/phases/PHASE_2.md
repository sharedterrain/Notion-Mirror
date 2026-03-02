<!-- Auto-generated from Notion. Do not edit directly. -->


```yaml
---
doc_id: "phase_2"
last_updated: "2026-02-26"
contract_version: "0.2.0"
---
```


**Status:** ✅ **Live** — fix: handler operational, conditional typed-field PATCHes on main and fix routes

**Time Estimate:** 1-2 hours

**Dependencies:** Phase 1 functional (routes tested; Inbox Log + Slack confirmations working)


---


## Contract References


This phase extends the Classification Interface implementation from Phase 1 with calibration and correction capabilities.

- **CONTRACT §7: Route Semantics** — Implements the `fix:` route (threaded reply correction/refile)

- **CONTRACT §3.5: Classification Interface** — Calibration via few-shot examples, confidence threshold tuning

- **CONTRACT §3.5: Extraction Interface** — Re-extraction on corrected routing

- **CONTRACT §3.5: Storage Interface** — Record updates from fix handler

- **CONTRACT §8: LLM Contracts** — Few-shot prompt version updates (`brain_dump_classifier_v1`)

- **CONTRACT §12: Observability & Error Handling** — Weekly misclassification digest, calibration metrics

- **CONTRACT §15: Change Control Protocol** — fix→refile workflow as a discovery→reconcile path


See: [CONTRACT (Brain Stem)](https://www.notion.so/cb5393105c784cc3969571a898b4f81e) v0.2.0 | [contracts/spec (Brain Stem)](https://www.notion.so/98d781fe40ed4e31a566f0d8886325fc)


---


## Overview


Enhance classification accuracy with few-shot learning and build the correction workflow that lets you fix misclassifications via Slack replies.

**Anchors from Phase 1 (as-built):**

- Inbox Log is the audit backbone for every capture.

- BD routing is based on `destination` + `confidence` with a Needs Review fallback.

- Slack confirmation replies happen in-thread.

- [Make.com](http://make.com/) filters must use mapped variable pills (avoid typed text conditions).


**What you're building:**

- fix: handler with original record deletion, re-extraction via Claude, and Slack confirmation

- Conditional typed-field PATCHes for main BD and fix routes (dates, single selects)

- Unfurl prevention on all Slack reply modules

- Classifier prompt date resolution (`now` pill for relative date conversion)


---


## Correction Semantics (fix:)


Define and implement a correction workflow that:

- Locates the original capture using Slack thread/message identifiers stored in Inbox Log.

- Refiles to a different destination when needed.

- Updates extracted fields appropriate to the corrected destination (not just “Filed To”).

- Preserves invariants (one Inbox Log entry per Slack message; consistent status transitions).


## Re-extraction on Fix


When a fix changes destination (or materially changes meaning), define one of:

- **Re-extract under fixed destination:** rerun a destination-specific extraction prompt (“destination is fixed; extract fields only”).

- **Destination-specific extraction prompts:** one extractor per table, called by the fix handler.


## Calibration Loop

- Add few-shot examples to the classifier prompt (ground truth comes from fix actions).

- Track confidence vs correction rate.

- Tune confidence threshold policy if needed.


## Weekly Misclassification Digest


Digest should be based on Phase 1 signals:

- Inbox Log entries in Needs Review.

- Captures with confidence below the auto-file threshold.

- Counts and examples of `fix:` corrections (these are labeled outcomes).


## Known Constraints to Respect (carried from Phase 1)

- Linked record fields can trigger `InvalidConfigurationError` when included in PATCH bodies (keep deferred unless fixed explicitly).

- Storing raw AI output requires escaping strategy (embedded quotes can break JSON bodies).

- PRO route Claude extraction is still deferred (define fix behavior for PRO items carefully).


---


## As-Designed: Implementation Steps


*To be designed during implementation*


---


## As-Built: Actual Implementation Notes


**Phase 2: fix: Handler + Typed Field Guards — As-Built Report**

**Date:** February 26, 2026

**Status:** ✅ **Live**

**contract_version:** 0.2.0


### Summary


Phase 2 build completed across a single session. The fix: handler is fully operational across all 5 destination routes (People, Projects, Ideas, Admin, Events) with original record deletion, re-extraction via Claude, Inbox Log update, linked record management, and Slack confirmation replies. Additionally, conditional PATCH modules were added to both main BD routes and fix routes to guard against empty typed fields (dates, single selects) that Airtable rejects.


### fix: Route Architecture


```plain text
1 (Webhook) → Bot filter → Prefix Router
  └── fix: route filter: 1.event.text Matches pattern ^fix: (case insensitive)
        ↓
      208 (Set Variable) — fix_thread_ts = 1.event.thread_ts
        ↓
      209 (HTTP GET) — Airtable lookup: find Inbox Log record by Slack Message TS
        URL: filterByFormula={Slack Message TS}="[208.fix_thread_ts]"&maxRecords=1
        ↓
      Router (6 branches, sequential by route order)
        ├── 1st-5th: DELETE routes (filtered by 209.Filed To value)
        │     ├── 242: Delete from People (filter: Filed To = People)
        │     ├── 243: Delete from Projects (filter: Filed To = Projects)
        │     ├── 244: Delete from Ideas (filter: Filed To = Ideas)
        │     ├── 245: Delete from Admin (filter: Filed To = Admin)
        │     └── 247: Delete from Events (filter: Filed To = Events)
        └── 6th: Pass-through (no filter) → continues to re-extraction chain
              ↓
            210 (Anthropic Claude) — re-extract fields for corrected destination
              ↓
            211 (Text Parser Replace) — strip JSON prefix
              ↓
            212 (Text Parser Replace) — strip JSON suffix
              ↓
            213 (Parse JSON)
              ↓
            214 (Router) — 5 routes by destination (filters on 1.event.text pattern)
              ├── People: 215 → linkify → 220 → 221
              ├── Projects: 222 → 254 → 226 → 227 → 273 → 274
              ├── Ideas: 228 → linkify → 229 → 230
              ├── Admin: 231 → linkify → 232 → 233 → 271
              └── Events: 234 → linkify → 235 → 236 → 269 → 270
```


**Critical:** Route ordering matters. DELETE routes must be numbered 1st–5th, pass-through 6th. Make processes routes sequentially by route order number — if the pass-through runs first and errors, deletes never fire.


### Key Module Reference (fix: Route)




### Delete Route Module Reference




**URL pattern:** `https://api.airtable.com/v0/appuT9wJR9eKmVfyU/[TABLE_ID]/` + `209.Data.records[1].fields.Destination Record ID` pill

**Critical:** No space between trailing `/` and record ID pill in DELETE URLs.


### fix: Destination Route Bodies


**People (215):**


```json
{"fields": {"Name": "213.data.name", "Context": "213.data.context", "Follow-Ups": "213.data.follow_ups", "Last Touched": "now", "Source": "Slack"}}
```


**Projects (222):**


```json
{"fields": {"Name": "213.data.name", "Next Action": "213.data.next_action", "Notes": "213.data.notes", "Last Touched": "now", "Source": "Slack"}}
```


Followed by conditional PATCHes: 273 (Type), 274 (Status)

**Ideas (228):**


```json
{"fields": {"Name": "213.data.name", "One-Liner": "213.data.one_liner", "Notes": "213.data.notes", "Last Touched": "now", "Source": "Slack"}}
```


**Admin (231):**


```json
{"fields": {"Name": "213.data.name", "Status": "Todo", "Notes": "213.data.notes", "Source": "Slack"}}
```


Followed by conditional PATCH: 271 (Due Date)

**Events (234):**


```json
{"fields": {"Title": "213.data.name", "Attendees": "213.data.attendees", "Location": "213.data.location", "Notes": "213.data.notes"}}
```


Followed by conditional PATCHes: 269 (Start Time), 270 (End Time)

**Note:** Events table does not have a Source field.


### fix: Inbox Log PATCH Bodies


All routes use the same pattern with Destination URL and Destination Record ID pointing to the new record:


```json
{"fields": {"Filed To": "[Destination]", "Confidence": 1.0, "Status": "Fixed", "Destination Name": "[create_module].data.fields.Name", "Destination URL": "https://airtable.com/appuT9wJR9eKmVfyU/[TABLE_ID]/[VIEW_ID]/[create_module].data.id", "Destination Record ID": "[create_module].data.id"}}
```


**Critical:** Destination Name uses the Airtable response (`[create_module].data.fields.Name`) not Claude's raw output (`213.data.name`) — Claude's text can contain newlines that break JSON.


### fix: Linked Record Modules


Placed after create module, before Inbox Log PATCH (matching main route pattern):




All use URL: `https://api.airtable.com/v0/appuT9wJR9eKmVfyU/tblrXultsjYl2aYqy/` + `209.Data.records[1].id` pill


### Module 210 Prompt (Authoritative Version)


```plain text
You are an extraction assistant. A message was misclassified in Brain Stem and needs to be refiled.

ORIGINAL MESSAGE:
209.Data.records[1].fields.Original Text

CORRECTION COMMAND:
1.event.text

INSTRUCTIONS:
1. The word after "fix:" in the correction command is the new destination (people, projects, ideas, admin, or events).
2. Extract fields using ONLY the schema matching that destination.
3. Extract from the ORIGINAL MESSAGE, not the correction command.
4. Return raw JSON only, no markdown code fences.

SCHEMAS:
PEOPLE: {"name":"string","context":"string","follow_ups":"string","notes":"string","tags":["string"],"reason":"string"}
PROJECTS: {"name":"string","project_type":"digital|physical|hybrid","status":"active|waiting|blocked|someday|done","next_action":"string","notes":"string","tags":["string"],"reason":"string"}
IDEAS: {"name":"string","one_liner":"string","notes":"string","tags":["string"],"reason":"string"}
ADMIN: {"name":"string","due_date":"YYYY-MM-DD|null","notes":"string","tags":["string"],"reason":"string"}
EVENTS: {"name":"string","start_time":"ISO8601|null","end_time":"ISO8601|null","attendees":"string|null","location":"string|null","notes":"string","tags":["string"],"reason":"string"}
```


**Note:** `209.Data.records[1].fields.Original Text` and `1.event.text` are Make pills that resolve at runtime.


### Conditional Typed-Field PATCHes (Main BD Routes)


Added to guard against empty typed fields that Airtable rejects. Placed at end of route after Slack reply. Filter checks if field Exists before PATCH fires.
