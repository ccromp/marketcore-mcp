# Plan — Set brief on "Q4 Product Launch" project to a recently-made messaging document

## User request
> "Set the brief on the 'Q4 Product Launch' project to that messaging document I just made."

## Interpretation & assumptions

The request has two ambiguous referents that I need to resolve before I can act:

1. **The project** — the user named it ("Q4 Product Launch"), but I need its `project_id` to call `update_project`. There may also be more than one project with a similar name, or the project might be archived.
2. **"That messaging document I just made"** — almost certainly a recently-created **content** item (since `create_content` is the only tool that produces user-authored documents in this MCP). I'll have to identify which content item by recency and likely by title (something matching "messaging"). I should NOT assume — I should confirm with the user before binding it.

The action itself is a single field update on the project: `update_project(project_id, project_brief_id=<content_id>)`. The schema for `update_project` explicitly lists `project_brief_id` as one of the updatable fields.

---

## 1. First message back to the user (verbatim)

> Got it — let me find the project and the messaging doc you just made so I can hook them up.
>
> One sec while I look both up — I'll confirm the exact match before I set the brief.

---

## 2. Ordered tool calls

### Step 1 — Find the project

**Tool:** `list_projects`
**Params:** (no filters; default listing)
**Rationale:** I need the `project_id` for "Q4 Product Launch". `list_projects` is the only way to enumerate projects in the user's library. I scan the result for a name matching "Q4 Product Launch" (case-insensitive, exact-ish).

**Branches:**
- **Exactly one match** → capture `project_id`, proceed to Step 2.
- **No match** → ask the user: *"I don't see a project called 'Q4 Product Launch' in your library. Did you mean one of these: [top 3 closest matches]? Or should I create it?"* Stop until they answer.
- **Multiple matches** (e.g. "Q4 Product Launch — Web", "Q4 Product Launch — APAC") → ask the user to disambiguate with the candidate names. Stop until they answer.

### Step 2 — Find the recently-made messaging document

**Tool:** `list_content`
**Params:** (no filters; rely on default ordering being recency-descending — if not, sort client-side by `created_at`)
**Rationale:** The user said "I just made" — strong signal of recency. I scan the top ~10 most recent content items for a title containing "messaging" (case-insensitive). I am explicitly NOT searching context items (`get_relevant_context`) because a project brief in MarketCore is a content item linked via `project_brief_id`, and context items live in a separate library.

**Branches:**
- **Exactly one recent match with "messaging" in the title** → capture `content_id`, proceed to Step 3 (still confirm with the user before writing).
- **Multiple plausible matches** (e.g. "Q4 messaging doc", "Messaging pillars v2") → list top 3 with title + created_at and ask which one.
- **No match** → ask: *"I don't see a recent doc with 'messaging' in the title. Can you give me the exact title, or paste a link?"* Stop until they answer.

### Step 3 — Confirm with the user before writing

This is a destructive-ish operation (overwrites whatever brief was there). Even if I'm confident, I confirm in one short message:

> Found these — confirming before I set the brief:
>
> - **Project:** "Q4 Product Launch" (id: `<project_id>`)
> - **Brief doc:** "<content title>" (created <timestamp>)
>
> Set this as the project brief? (yes / no)

**Branches:**
- **Yes** → proceed to Step 4.
- **No / "actually I meant X"** → loop back to Step 2 with the corrected reference.

### Step 4 — Optional: peek at the existing brief (only if cheap and useful)

**Tool:** `get_project`
**Params:** `{ project_id: <captured> }`
**Rationale:** Lets me see if there's already a `project_brief_id` set, so I can mention to the user "this replaces the existing brief 'X'" rather than silently clobbering. If the project response doesn't include the existing brief field, skip this step — don't burn a call on uncertainty.

If a prior brief exists, I'd amend the Step 3 confirmation with: *"FYI this will replace the current brief, '<existing brief title>'."*

### Step 5 — Set the brief

**Tool:** `update_project`
**Params:**
```json
{
  "project_id": "<captured project id>",
  "project_brief_id": "<captured content id>"
}
```
**Rationale:** `update_project` is the documented tool for changing `project_brief_id` on an existing project. No other field is being modified, so I pass only these two.

### Step 6 — Verify

**Tool:** `get_project`
**Params:** `{ project_id: <captured> }`
**Rationale:** Read-after-write check. I want to see `project_brief_id` reflected in the response before I report success to the user. If `update_project` returns the updated project object directly, this step is redundant — skip it.

---

## 3. What I would surface to the user at each meaningful step

| Step | What I say |
|---|---|
| Initial ack | The first message above. |
| After Step 1 if ambiguous | Ask for project disambiguation with candidate list. |
| After Step 2 if ambiguous | Ask for doc disambiguation with candidate list (title + created_at). |
| Step 3 (always) | Single confirmation message with both resolved IDs/titles, plus a note if it overwrites an existing brief. |
| After Step 5 success | "Done — 'Q4 Product Launch' brief is now set to '\<doc title\>'." Include the project name and doc title (not raw IDs — IDs are noise to the user). |
| On any failure | Plain-English error: which step failed, what the API said, and one suggested next action. |

I deliberately do NOT chatter between Steps 1–2 ("Looking up the project... now looking up the doc...") — those two lookups are fast and chatter is noise. The first ack covers the wait, and Step 3's confirmation is the next thing the user hears.

---

## 4. Final action when the brief is set

After `update_project` succeeds (and `get_project` confirms `project_brief_id` matches), reply:

> Done. **"Q4 Product Launch"** now has **"\<doc title\>"** set as its project brief.
>
> Want me to do anything else with it — e.g. generate content from this brief, share it, or open the project?

The trailing offer is one-line and optional — it teases the natural next moves (`create_content` from this project, `create_external_share` of the brief doc, etc.) without forcing a follow-up.

---

## Notes on what I am NOT doing

- **Not** creating a new content item or context item. The user said the doc already exists ("that messaging document I just made").
- **Not** touching `add_context` or `create_context_collection`. A project *brief* is a content link, not a context item.
- **Not** calling `get_content` to read the full doc text. The user didn't ask me to summarize or validate it — they asked me to wire it up. Reading it would be gold-plating and would burn tokens for no benefit.
- **Not** calling `create_project` as a fallback if the project isn't found. I ask first.
- **Not** asking the user for the project ID or content ID up front. Those are internal — the user gave me names, my job is to resolve them.
