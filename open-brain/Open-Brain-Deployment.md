# Open-Brain Deployment

---

# Open Brain: Architecture & Build Specification

```yaml
---
doc_id: "open_brain_build_spec"
created: "2026-03-04"
author: "Claude Opus 4.6 (architectural spec)"
builder: "Claude Sonnet (execution)"
contract_version: "1.0.0"
status: "Steps 1-11 Complete — Primary Routes Live"
---
```

## 1. Purpose

Open Brain is a semantic memory layer that stores vector embeddings of every captured thought, enabling meaning-based retrieval across all AI tools in Jedidiah's ecosystem. It serves two integration points:

**Magi/OpenClaw** — gains persistent semantic memory beyond its 50k token session context. Can store and retrieve thoughts via tool calls.

**Brain Stem (Make.com)** — gains a secondary push target. Every classified capture gets embedded and stored alongside its Brain Stem metadata, making the full capture history semantically searchable.

## 2. Architecture Overview

```plain text
┌─────────────────────────────────────────────────────────────┐
│                     WRITE PATH (Ingest)                      │
│                                                              │
│  Brain Stem (Make.com)──┐                                    │
│    POST after each       ├──→ Supabase Edge Function         │
│    destination route     │    "ingest-thought"                │
│                          │      │                             │
│  Magi (OpenClaw)────────┘      ├─→ OpenRouter embedding      │
│    store_thought tool          ├─→ INSERT into thoughts table │
│                                └─→ Return {id, status}       │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│                     READ PATH (Retrieval)                     │
│                                                              │
│  Magi (OpenClaw)──────────→ Supabase Edge Function           │
│    search_brain tool          "open-brain-mcp"               │
│                                 │                             │
│  Claude Desktop/Code──────→     ├─→ search_thoughts (semantic)│
│    MCP config                   ├─→ recent_thoughts (time)    │
│                                 └─→ brain_stats (counts)      │
│                                                              │
└──────────────────────────────────────────────────────────────┘

Storage: Supabase (PostgreSQL + pgvector)
Embeddings: OpenRouter → text-embedding-3-small (1536 dimensions)
```

## 3. Components

### 3.1 Supabase Project

**Project name:** `open-brain` **Region:** Closest to Victoria, BC (US West or Canada if available)

### 3.2 Database Schema

**Extension required:** `pgvector`

**Table: **`thoughts`

| Column | Type | Constraints | Notes |
| --- | --- | --- | --- |
| `id` | uuid | PK, default `gen_random_uuid()` |  |
| `content` | text | NOT NULL | Raw captured text |
| `embedding` | vector(1536) |  | text-embedding-3-small output |
| `metadata` | jsonb | DEFAULT '{}' | Structured metadata (see §3.3) |
| `created_at` | timestamptz | DEFAULT `now()` |  |
| `updated_at` | timestamptz | DEFAULT `now()` |  |

**Function: **`match_thoughts`

```sql
create or replace function match_thoughts (
  query_embedding vector(1536),
  match_threshold float default 0.5,
  match_count int default 10
)
returns table (
  id uuid,
  content text,
  metadata jsonb,
  similarity float,
  created_at timestamptz
)
language plpgsql
as $$
begin
  return query
  select
    thoughts.id,
    thoughts.content,
    thoughts.metadata,
    1 - (thoughts.embedding <=> query_embedding) as similarity,
    thoughts.created_at
  from thoughts
  where 1 - (thoughts.embedding <=> query_embedding) > match_threshold
  order by thoughts.embedding <=> query_embedding
  limit match_count;
end;
$$;
```

**Security:**

```sql
alter table thoughts enable row level security;
create policy "Service role full access"
  on thoughts for all
  using (auth.role() = 'service_role');
```

### 3.3 Metadata Schema

The `metadata` JSONB column carries source-specific context. All fields optional.

**From Brain Stem captures:**

```json
{
  "source": "brain_stem",
  "destination": "people|projects|ideas|admin|events",
  "confidence": 0.85,
  "prefix": "BD:|PRO:|null",
  "destination_record_id": "rec...",
  "classified_name": "string",
  "tags": ["string"]
}
```

**From Magi:**

```json
{
  "source": "magi",
  "session_id": "string (optional)",
  "type": "decision|action_item|insight|context|note",
  "topic": "string (optional)"
}
```

**From manual/other:**

```json
{
  "source": "manual|slack_direct|other",
  "category": "string (optional)"
}
```

## 4. Edge Function: `ingest-thought`

**Purpose:** Universal write endpoint. Accepts text + metadata, generates embedding, stores in Supabase.

**URL:** `https://[PROJECT_REF].supabase.co/functions/v1/ingest-thought`

**Method:** POST

**Authentication:** `x-brain-key` header checked against `MCP_ACCESS_KEY` secret.

**Request body:**

```json
{
  "text": "string (required — the thought to embed and store)",
  "metadata": { }
}
```

**Processing steps:**

1. Validate `x-brain-key` header against `MCP_ACCESS_KEY` env var. Return 401 if missing/wrong.

1. Validate `text` field exists and is non-empty after trim. Return 400 if invalid.

1. Generate embedding via OpenRouter API:
    - Endpoint: `https://openrouter.ai/api/v1/embeddings`
    - Model: `openai/text-embedding-3-small`
    - Input: the `text` value

1. INSERT into `thoughts` table: `content` = text, `embedding` = result, `metadata` = metadata object.

1. Return `{ "id": "<uuid>", "status": "stored" }` with 200.

**Error handling:**

- OpenRouter failure → return 502 with `{ "error": "Embedding generation failed", "detail": "..." }`

- Supabase insert failure → return 500 with `{ "error": "Storage failed", "detail": "..." }`

**Environment variables (Supabase secrets):**

- `OPENROUTER_API_KEY` — OpenRouter API key

- `MCP_ACCESS_KEY` — shared access key for both ingest and MCP endpoints

- `SUPABASE_URL` — auto-provided by Supabase

- `SUPABASE_SERVICE_ROLE_KEY` — auto-provided by Supabase

**Implementation notes:**

- This function does NOT generate metadata via LLM. Brain Stem and Magi both pass structured metadata at call time. This eliminates the redundant LLM call from the original Open Brain guide.

- No Slack reply logic. Callers handle their own confirmations.

- The function should handle the Slack event challenge handshake ONLY if direct Slack integration is added later. For now, skip it.

### Reference implementation (Deno/Supabase Edge Function):

```typescript
import { createClient } from "@supabase/supabase-js";

const OPENROUTER_URL = "https://openrouter.ai/api/v1/embeddings";
const EMBEDDING_MODEL = "openai/text-embedding-3-small";

Deno.serve(async (req: Request) => {
  // CORS preflight
  if (req.method === "OPTIONS") {
    return new Response(null, {
      headers: {
        "Access-Control-Allow-Origin": "*",
        "Access-Control-Allow-Headers": "content-type, x-brain-key",
        "Access-Control-Allow-Methods": "POST",
      },
    });
  }

  // Auth check
  const accessKey = req.headers.get("x-brain-key");
  const expectedKey = Deno.env.get("MCP_ACCESS_KEY");
  if (!accessKey || accessKey !== expectedKey) {
    return new Response(JSON.stringify({ error: "Unauthorized" }), {
      status: 401,
      headers: { "Content-Type": "application/json" },
    });
  }

  // Parse body
  let body: { text?: string; metadata?: Record<string, unknown> };
  try {
    body = await req.json();
  } catch {
    return new Response(JSON.stringify({ error: "Invalid JSON" }), {
      status: 400,
      headers: { "Content-Type": "application/json" },
    });
  }

  const text = body.text?.trim();
  if (!text) {
    return new Response(
      JSON.stringify({ error: "text field required and must be non-empty" }),
      { status: 400, headers: { "Content-Type": "application/json" } }
    );
  }

  const metadata = body.metadata ?? {};

  // Generate embedding
  let embedding: number[];
  try {
    const embRes = await fetch(OPENROUTER_URL, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${Deno.env.get("OPENROUTER_API_KEY")}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        model: EMBEDDING_MODEL,
        input: text,
      }),
    });
    if (!embRes.ok) {
      const detail = await embRes.text();
      return new Response(
        JSON.stringify({ error: "Embedding generation failed", detail }),
        { status: 502, headers: { "Content-Type": "application/json" } }
      );
    }
    const embData = await embRes.json();
    embedding = embData.data[0].embedding;
  } catch (err) {
    return new Response(
      JSON.stringify({
        error: "Embedding generation failed",
        detail: String(err),
      }),
      { status: 502, headers: { "Content-Type": "application/json" } }
    );
  }

  // Store in Supabase
  const supabase = createClient(
    Deno.env.get("SUPABASE_URL")!,
    Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!
  );

  const { data, error } = await supabase
    .from("thoughts")
    .insert({ content: text, embedding, metadata })
    .select("id")
    .single();

  if (error) {
    return new Response(
      JSON.stringify({ error: "Storage failed", detail: error.message }),
      { status: 500, headers: { "Content-Type": "application/json" } }
    );
  }

  return new Response(
    JSON.stringify({ id: data.id, status: "stored" }),
    { status: 200, headers: { "Content-Type": "application/json" } }
  );
});
```

## 5. Edge Function: `open-brain-mcp`

**Purpose:** MCP-compatible read endpoint. Exposes semantic search, recent browse, and stats to any MCP client.

**URL:** `https://[PROJECT_REF].supabase.co/functions/v1/open-brain-mcp`

**Authentication:** `x-brain-key` header, same key as ingest.

**Tools exposed:**

### 5.1 `search_thoughts`

- **Input:** `{ query: string, threshold?: number (default 0.5), limit?: number (default 10) }`

- **Process:** Generate embedding of query via OpenRouter (same model), call `match_thoughts` RPC.

- **Output:** Array of `{ id, content, metadata, similarity, created_at }`

### 5.2 `recent_thoughts`

- **Input:** `{ hours?: number (default 24), limit?: number (default 20), source?: string (optional filter) }`

- **Process:** SELECT from thoughts WHERE `created_at > now() - interval`, optionally filtered by `metadata->>'source'`, ordered by `created_at DESC`.

- **Output:** Array of `{ id, content, metadata, created_at }`

### 5.3 `brain_stats`

- **Input:** `{}`

- **Process:** COUNT total, COUNT by source, COUNT by destination, MIN/MAX created_at.

- **Output:** `{ total, by_source: {...}, by_destination: {...}, oldest, newest }`

**Implementation:** Follow the Open Brain guide's MCP server code (Step 11), which uses `@hono/mcp` and `@modelcontextprotocol/sdk`. Adapt the three tools above. The guide's implementation is sound — the only changes are:

1. Add the `source` filter option to `recent_thoughts`.

1. Add `by_destination` breakdown to `brain_stats`.

1. Use the same `x-brain-key` auth as ingest.

**Dependencies (deno.json):**

```json
{
  "imports": {
    "@hono/mcp": "npm:@hono/mcp@0.1.1",
    "@modelcontextprotocol/sdk": "npm:@modelcontextprotocol/sdk@1.24.3",
    "hono": "npm:hono@4.9.2",
    "zod": "npm:zod@4.1.13",
    "@supabase/supabase-js": "npm:@supabase/supabase-js@2.47.10"
  }
}
```

## 6. Brain Stem Integration (Make.com)

**What:** Add one HTTP POST module at the tail end of each Brain Stem destination route, after the Inbox Log PATCH completes. This is a fire-and-forget push — Brain Stem does not wait for or act on the Open Brain response.

**Where in the scenario:** After the final module in each of these chains:

- People route (primary + fallback)

- Projects route (primary + fallback)

- Ideas route (primary + fallback)

- Admin route (primary + fallback)

- Events route (primary + fallback)

- Needs Review route (primary + fallback)

- PRO bypass route

That's **13 HTTP modules** total (6 primary destinations + 6 fallback destinations + 1 PRO bypass). All identical in structure, only the pill references differ.

**Module configuration (each instance):**

```plain text
Type: HTTP > Make a request
Method: POST
URL: https://[PROJECT_REF].supabase.co/functions/v1/ingest-thought
Headers:
  x-brain-key: <<OPEN_BRAIN_ACCESS_KEY>>
  Content-Type: application/json
Body type: Raw
Content type: JSON (application/json)
Parse response: No (fire-and-forget)
```

**Body template (primary BD routes):**

```json
{
  "text": "<<CLEAN_TEXT_PILL>>",
  "metadata": {
    "source": "brain_stem",
    "destination": "<<55.destination or literal>>",
    "confidence": <<55.confidence>>,
    "classified_name": "<<55.data.name>>",
    "destination_record_id": "<<CREATE_MODULE.data.id>>"
  }
}
```

**Pill mapping per route (primary — off router 30):**

| Route | clean_text pill | destination | confidence pill | name pill | record_id pill |
| --- | --- | --- | --- | --- | --- |
| People | `21.clean_text` | `"people"` | `55.confidence` | `55.data.name` | People create module `.data.id` |
| Projects | `21.clean_text` | `"projects"` | `55.confidence` | `55.data.name` | Projects create module `.data.id` |
| Ideas | `21.clean_text` | `"ideas"` | `55.confidence` | `55.data.name` | Ideas create module `.data.id` |
| Admin | `21.clean_text` | `"admin"` | `55.confidence` | `55.data.name` | Admin create module `.data.id` |
| Events | `21.clean_text` | `"events"` | `55.confidence` | `55.data.name` | Events create module `.data.id` |
| Needs Review | `21.clean_text` | `"needs_review"` | `55.confidence` | `55.destination` | `null` |

**PRO bypass route body:**

```json
{
  "text": "<<7.clean_text>>",
  "metadata": {
    "source": "brain_stem",
    "destination": "projects",
    "confidence": 1.0,
    "prefix": "PRO:",
    "destination_record_id": "<<PRO_CREATE_MODULE.data.id>>"
  }
}
```

**Fallback routes (off OpenRouter branch):** Same structure, same pills but from the fallback Parse JSON module instead of `55`.

**Error handling:** None required. If the Open Brain push fails, Brain Stem continues normally — the capture is already in Airtable. The Open Brain push is additive, not critical path.

**Fix route:** Also add a push after fix handler creates the new record. Body uses `213` pills and `"source": "brain_stem_fix"`.

### 6.1 As-Built Report (Step 11 — March 5, 2026)

**Status:** ✅ Complete (primary routes)

**Modules built: 12** (not 13 as designed — fallback routes parked)

- 7 primary route modules: People (275), Projects (276), Events (277), Admin (278), Ideas (279), Needs Review (288), PRO bypass (289)

- 5 fix route modules: People (281), Projects (282), Ideas (283), Admin (284), Events (285)

**Module reference:**

| **Module #** | **Route** | **text field** | **record_id** |
| --- | --- | --- | --- |
| 275 | People | `55.data.name + 55.data.context` | `65.data.id` |
| 276 | Projects | `55.data.name + 55.data.next_action + 55.data.notes` | `73.data.id` |
| 277 | Events | `55.data.name + 55.data.attendees + 55.data.location + 55.data.notes` | `77.data.id` |
| 278 | Admin | `55.data.name + 55.data.notes` | `76.data.id` |
| 279 | Ideas | `55.data.name + 55.data.one_liner + 55.data.notes` | `75.data.id` |
| 288 | Needs Review | `21.clean_text` | — |
| 289 | PRO bypass | `7.clean_text` | `15.data.id` |
| 281 | fix: People | `213.name + 213.context` | `215.data.id` |
| 282 | fix: Projects | `213.name + 213.next_action + 213.notes` | `222.data.id` |
| 283 | fix: Ideas | `213.name + 213.one_liner + 213.notes` | `228.data.id` |
| 284 | fix: Admin | `213.name + 213.notes` | `231.data.id` |
| 285 | fix: Events | `213.name + 213.attendees + 213.location + 213.notes` | `234.data.id` |

All `record_id` pills confirmed. Main create modules: People (65), Projects (73), Ideas (75), Admin (76), Events (77).

**Key deviation from as-designed:** The `text` field uses Claude-extracted fields rather than raw `clean_text`. This was a deliberate improvement:

- **Primary BD routes (People, Projects, Ideas, Admin, Events):** Text is composed from `55.data.name` + destination-specific content fields (e.g., `55.data.context` for People, `55.data.next_action` + `55.data.notes` for Projects). This eliminates BD:/PRO: prefix contamination and stores richer semantic content.

- **Fix routes:** Same pattern using `213` pills. Hardcoded `confidence: 1.0`, `source: "brain_stem_fix"`.

- **Needs Review:** Uses raw `21.clean_text` (no extracted fields available — Claude returned low confidence).

- **PRO bypass:** Uses raw `7.clean_text` (no classification/extraction on this route).

**Parked/deferred:**

- Fallback routes (6 modules): Parked until primary routes are tuned through real usage

- CAL: and R: routes: Not built — routes don't exist in Brain Stem yet

- Fix route end-to-end test: Deferred to Step 12 — requires a naturally occurring misclassification

**Verified in testing:**

- 11 records in Supabase `thoughts` table

- All 6 destinations represented (people, projects, ideas, admin, events, needs_review)

- 9 `brain_stem` source entries, embeddings populating correctly

- Magi can access Open Brain via `brain_stats` and `search_thoughts`

- Full round-trip confirmed: Brain Stem capture → Supabase → Magi retrieval

**Credential tracker:** No new credentials generated. All existing Open Brain credentials remain valid.

## 7. Magi/OpenClaw Integration

### 7.1 MCP Tool Registration

Add to OpenClaw's MCP configuration (or tool config, depending on how Magi's tools are registered):

```json
{
  "mcpServers": {
    "open-brain": {
      "type": "http",
      "url": "https://[PROJECT_REF].supabase.co/functions/v1/open-brain-mcp",
      "headers": {
        "x-brain-key": "<<OPEN_BRAIN_ACCESS_KEY>>"
      }
    }
  }
}
```

This gives Magi access to `search_thoughts`, `recent_thoughts`, and `brain_stats` automatically.

### 7.2 Direct Ingest Tool (store_thought)

If OpenClaw supports custom HTTP tools outside MCP, add a direct tool definition for writing:

```yaml
name: store_thought
description: Store a thought, decision, or insight in Jedidiah's Open Brain for long-term semantic retrieval.
endpoint: https://[PROJECT_REF].supabase.co/functions/v1/ingest-thought
method: POST
headers:
  x-brain-key: "<<OPEN_BRAIN_ACCESS_KEY>>"
  Content-Type: "application/json"
body_schema:
  text: string (required) — the content to store
  metadata:
    source: "magi" (always)
    type: "decision|action_item|insight|context|note"
    topic: string (optional — project or subject name)
    session_id: string (optional)
```

If OpenClaw only supports MCP tools, add a fourth tool (`store_thought`) to the MCP server that accepts text + metadata and calls the ingest function internally.

### 7.3 Bootstrap File Additions

Add to Magi's `AGENTS.md` or equivalent bootstrap instructions:

```markdown
## Semantic Memory (Open Brain)

You have access to Jedidiah's Open Brain — a persistent semantic memory store.

### When to SEARCH (search_thoughts):
- At session start: search for context related to the current topic
- When Jedidiah references past work, decisions, or people
- When you need context beyond your current session transcript
- Before making recommendations that should account for prior decisions

### When to STORE (store_thought):
- Decisions made during this session (type: "decision")
- Action items assigned (type: "action_item")
- Key insights or realizations (type: "insight")
- Context that future sessions will need (type: "context")
- Do NOT store: casual conversation, greetings, troubleshooting back-and-forth

### Metadata conventions:
- Always set source: "magi"
- Always set type to one of: decision, action_item, insight, context, note
- Set topic to the project name when applicable (e.g., "brain_stem", "open_brain", "regenerative_consulting")
```

## 8. Build Sequence

Execute in this order. Each step is independently testable.

| Step | What | Time | Test |
| --- | --- | --- | --- |
| ✅ 1 | Create Supabase project, enable pgvector, run SQL (table + function + RLS) | 15 min | Table visible in Table Editor, function in Database > Functions |
| ✅ 2 | Get OpenRouter API key, add $5 credits | 5 min | Key in credential tracker |
| ✅ 3 | Generate `MCP_ACCESS_KEY` (`openssl rand -hex 32`) | 1 min | Key in credential tracker |
| ✅ 4 | Set Supabase secrets: `OPENROUTER_API_KEY`, `MCP_ACCESS_KEY` | 2 min | `supabase secrets list` shows both |
| ✅ 5 | Deploy `ingest-thought` Edge Function | 10 min | `curl -X POST` with test payload returns `{"id":"...","status":"stored"}` |
| ✅ 6 | Verify storage: check Supabase Table Editor for test row with embedding | 2 min | Row exists with 1536-dim vector |
| ✅ 7 | Deploy `open-brain-mcp` Edge Function | 10 min | Claude Desktop or `curl` can call `brain_stats` and see count=1 |
| ✅ 8 | Test semantic search: call `search_thoughts` with a query related to test row | 2 min | Returns the test row with similarity > 0.5 |
| ✅ 9 | Configure Magi MCP endpoint + `store_thought` tool | 15 min | Magi can search and store |
| ✅ 10 | Test Magi round-trip: store a thought, then search for it semantically | 5 min | Stored thought returns on semantic query |
| ✅ 11 | Add HTTP POST modules to Brain Stem Make scenario (12 modules — primary + fix, fallback parked) | 45 min | 11 records in Supabase, all 6 destinations, full round-trip confirmed |
| ⏸️ 12 | Fix route end-to-end test (deferred — requires naturally occurring misclassification) | 10 min | Fix a capture, verify corrected version appears in Supabase |

**Total estimated time:** ~2 hours

## 9. Credential Tracker

```plain text
=== OPEN BRAIN CREDENTIALS ===
Supabase Project Ref: ________________
Supabase Project URL: https://________________.supabase.co
Supabase DB Password: ________________
Supabase Service Role Key: ________________ (auto-available in Edge Functions)
OpenRouter API Key: ________________
MCP Access Key: ________________ (generated via openssl rand -hex 32)
Ingest URL: https://________________.supabase.co/functions/v1/ingest-thought
MCP URL: https://________________.supabase.co/functions/v1/open-brain-mcp
```

## 10. Security Notes

- `MCP_ACCESS_KEY` is the single shared secret for both endpoints. Rotate if compromised.

- Supabase Service Role Key never leaves the Edge Function environment.

- OpenRouter API Key never leaves the Edge Function environment.

- Make.com stores the `MCP_ACCESS_KEY` as a header value in HTTP modules — treat the Make scenario as a sensitive asset.

- The `x-brain-key` header is sent over HTTPS. No additional encryption needed.

- RLS policy restricts all access to service role — no anonymous reads possible.

## 11. Cost Estimate

| Component | Unit Cost | Usage (30 captures/day + 10 Magi queries/day) | Monthly |
| --- | --- | --- | --- |
| Embeddings (write) | ~$0.02/1M tokens | ~30 captures × 200 tokens avg = 6K tokens/day | ~$0.004 |
| Embeddings (read queries) | ~$0.02/1M tokens | ~10 queries × 50 tokens avg = 500 tokens/day | ~$0.001 |
| Supabase | Free tier | Well within limits | $0 |
| OpenRouter minimum | $5 prepaid | Lasts months at this rate | ~$0.15 amortized |

**Effective monthly cost: < $0.50**

## 12. Known Limitations & Future Enhancements

**Not in this build:**

- No LLM-generated metadata at ingest time (Brain Stem and Magi supply their own)

- No deduplication (same text stored twice = two rows). Acceptable at current volume.

- No deletion/update tools exposed via MCP (manual via Supabase dashboard if needed)

- No Slack confirmation from Open Brain (Brain Stem handles its own confirmations)

**Future possibilities:**

- Add `source` filter to `search_thoughts` for scoped queries ("only Brain Stem captures" or "only Magi insights")

- Passive Magi channel capture (webhook on Magi Slack channel → ingest)

- Weekly digest via Supabase scheduled function (trending topics, orphaned action items)

- Open Brain as context loader for Claude sessions (pre-load relevant thoughts into project context)

## 13. Known Issues (Carried Forward)

Issues discovered during build and integration testing, documented here for tracking. Each is also recorded in the respective project page.

### Brain Stem issues

- **Double curly brace encoding on **`clean_text`** pill** — The `21.clean_text` pill in [Make.com](http://make.com/) encodes double curly braces, producing `content` instead of raw text. Cosmetic only — does not affect Airtable storage or Open Brain ingestion.

- **Fix route command format ambiguity** — The fix route parser reads the word immediately after `fix:` as the destination keyword. Users naturally want to include context (e.g., `fix: admin dutch passport renewal...`) but the correct format is `fix: [destination]` — one word only. The re-extraction happens from the original captured message via module 209, not from the fix command text. The Slack bot reply doesn't make this format explicit enough.

- **Module 209 empty-result guard missing** — If the Airtable lookup-by-TS in module 209 returns no results (e.g., message was deleted or TS mismatch), the fix route proceeds with empty data. Needs a guard/error branch. Pre-existing issue.

### OpenClaw (Magi) issues

- **Magi defaulting to **`recent_thoughts`** instead of **`search_thoughts` — When asked to search Open Brain, Magi sometimes calls `recent_thoughts` (time-based) instead of `search_thoughts` (semantic). Behavioral — requires prompt tuning in Magi's bootstrap instructions.

- **Magi hallucinating on truncated MCP responses** — When MCP responses are truncated due to length, Magi fabricates the remaining content rather than acknowledging the truncation. Behavioral issue.

- **Allow-always not persisting for Supabase domain** — OpenClaw's allow-always permission for the Supabase domain does not persist across sessions. Requires re-approval each session. Platform limitation.

## 14. Architecture Notes

**Morning Brief — Option C (Recommended for Phase 7)**

During Step 11 integration testing, an architecture discussion was initiated regarding how Brain Stem and Magi should collaborate on daily/weekly digest delivery (Phase 7). Notion AI (Claude Opus 4.6) recommended **Option C — hybrid architecture:**

- **Brain Stem** collects structured digest data (counts, highlights, trends) and pushes to Open Brain

- **Magi** reads the digest data from Open Brain, synthesizes a natural-language brief, and delivers to #magination

- **Open Brain** serves as the bridge between the two systems, consistent with its role as the shared semantic memory layer

This avoids Brain Stem needing LLM synthesis capabilities and avoids Magi needing direct Airtable access. Flagged for Phase 7 planning.

---

*Steps 1–11 complete. Step 12 (fix route e2e test) deferred pending naturally occurring misclassification. Fallback routes parked for future tuning. No new credentials this session.*

---
