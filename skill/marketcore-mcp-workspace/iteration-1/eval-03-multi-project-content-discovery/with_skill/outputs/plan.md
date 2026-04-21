# Plan — "What content do I have related to the launch? I want to see what's already been written across all my projects."

## 1. Read of the user's intent

The user is asking a **discovery / Q&A question** about what already exists in their library — not a generation request. Two signals:

- **"related to the launch"** — a topical filter; "the launch" is ambiguous (definite article, no name).
- **"across all my projects"** — explicit cross-project scope. They want a *survey*, grouped by project, not one project's contents.

This maps to SKILL.md **Workflow 5 — Find existing context / Q&A / ideation**, with a content-list angle. Per SKILL.md "Choosing between similar tools":

> *To know what content already exists about a topic → `marketcore:get_relevant_context` for context, OR `marketcore:list_content` for a content list.*

Because the user said "what's been *written*," they primarily mean **Content** (documents). So `marketcore:list_content` is the lead tool, with `marketcore:get_relevant_context` as a complementary pass for reference materials. `marketcore:list_projects` provides the structural backbone they asked for.

I will **not** call `marketcore:create_content` (no generation requested), `marketcore:get_content` per item (handing back `link_url`s is the right move per SKILL.md), or `marketcore:get_current_user_info` (not needed; auth is implicit).

---

## 2. Disambiguation move — concrete, not blocking

"The launch" needs clarifying, but I want to ask the question with real options rather than the generic "which launch?". Per the skill's "you may already have context" guidance and the discovery-first posture of Workflow 5, I'll do **one cheap read-only pre-call** so the question is grounded:

**Pre-call:** `marketcore:list_projects` — gives me the project roster, names, and content counts per project. This is a single team-scoped read; it lets me name candidate launches in the clarifier.

Then I message the user. Three branches based on what `list_projects` shows:

- **Exactly one obviously launch-named project** (e.g. "Q3 Launch"):
  > "I see one launch-shaped project — **'[name]'**. Did you mean that one specifically, or do you want a topical sweep for any launch-related content across every project? I can do either."
- **Multiple launch-named projects** (e.g. "Q3 Launch", "Acme Launch"):
  > "You've got a few launch-related projects: **[A]**, **[B]**, **[C]**. Want one in particular, or all of them — and should I also include launch-flavored content sitting outside those projects?"
- **No obviously launch-named project**:
  > "Quick check before I dig — when you say 'the launch,' do you have a specific one in mind (and if so what's it called)? Or do you want me to do a broad topical search for launch-related content across all your projects?"

I would **wait for the user's answer** before fanning out. Tiny politeness that avoids returning the wrong content set entirely.

---

## 3. Plan announcement (one sentence, after clarification)

Per SKILL.md (state your plan in one sentence, then call the tool):

> "Got it — I'll list your content across all projects, filter for items related to **'[clarified topic]'**, and run a relevancy pass over your reference library so I catch launch-themed material that isn't in an obviously-named project. Read-only, no generation."

---

## 4. Tool sequence

This is the "broad sweep" branch. The "specific project" branch is a strict subset.

### Step A — `marketcore:list_projects` *(already done as pre-call in §2)*

**Why.** Project roster, names, `link_url`s, content counts. Provides the grouping skeleton. Reuse the result from §2 — don't re-call.

### Step B — `marketcore:list_content`

**Why.** Canonical "what content exists" tool per the SKILL.md decision table. Returns content items team-wide. I'll filter client-side on titles for launch-shaped keywords ("launch", "GTM", "go-to-market", "release", "rollout", "announcement", "GA"). Catches launch content authored *outside* any project, too.

**Params.** Default — no `project_id` so I get the cross-project view in one call.

### Step C — `marketcore:get_relevant_context`

**Why.** Workflow 5's primary tool. Catches **context items** (briefs, research, transcripts, competitor docs, customer interviews) the user might consider "written about the launch" even though they're not Content per se. Also catches content that doesn't have "launch" in the title but is semantically about it.

**Params.**
- `prompt`: `"the launch — product launch announcements, GTM messaging, launch plans, launch emails, launch one-pagers, launch enablement materials"` (or substitute the launch's actual name if the user gave one).
- No `project_id` (cross-project).

**Caveat I'll flag to the user.** Per pitfalls.md E2, this returns RAG chunks (~few hundred words each), not full items. I'll surface parent item names + locations, not raw chunk text.

**Pagination.** If the first call's chunks all look highly relevant and there are clearly more, I'll do **one** follow-up passing `context_rag_ids` from round 1 to exclude already-seen chunks (pitfalls.md E3). Cap at 2–3 paginated calls — past that, the user is better off browsing in-app.

### Step D — Per-project deep-dive (conditional, parallel)

For each project from Step A whose name matches launch keywords, in parallel:

`marketcore:get_project(project_id=<uuid>)`

**Why.** Surfaces that project's `documents` array AND its project-scoped context items in one call — both relevant to "what's been written for this launch." Step B's `list_content` doesn't expose project-scoped context items.

**Cap.** If >5 launch-named projects, top 5 by content count and offer to expand.

### Step E (optional) — Project-scoped RAG

If the user clarified to a single project:

`marketcore:get_relevant_context(prompt="launch content, messaging, plans", project_id=<uuid>)` — depth in one project rather than breadth.

### What I am NOT calling, and why

- **`marketcore:get_content`** — per SKILL.md, only legitimate when the user later asks a body-content question or for `convert_markdown_to_word_doc`. For inventory, `link_url` is the right hand-off.
- **`marketcore:get_current_user_info`** — auth is implicit; nothing to verify.
- **`marketcore:get_core_context`** — Brand Foundation isn't relevant to a content-discovery question.
- **`marketcore:list_blueprints`** — not asked.
- **`marketcore:create_content` / any mutation** — they asked "what do I have," not "make me something."

---

## 5. How I'd present results

Structured by project (matches the user's "across all my projects" framing). Never raw UUIDs, always names + `link_url`.

```
Here's what I found related to [clarified topic]:

Launch-named projects
─────────────────────

[Project name]  (N content items, [project link])
  - [Title] — [category] — [link]
  - [Title] — [link]
  - Project context items: [N items, including "[name1]", "[name2]" — project link]

[Next project]
  - …

Other launch-related content
────────────────────────────
(Items not in a launch-named project)
  - [Title] — in project "[X]" — [link]
  - [Title] — no project — [link]

Reference Library hits
──────────────────────
(Context items mentioning launches, from relevancy search)
  - [Parent item name] — Reference Library — [link if available]
  - [Parent item name] — in project "[X]" — [link]

(Searched ~10 chunks; let me know if you want me to keep looking.)
```

Conventions enforced:
- Group by project — that was the user's framing.
- `link_url`s, never UUIDs (pitfalls.md E10).
- Distinguish Content from Context items — the user said "content" but they may mean both. Lead with Content; flag Context items as a secondary section.
- Mention `stage` (`ready` vs `in_progress`) if relevant — they may want only finished material.
- Be honest about RAG coverage — chunks not full items, paginated only a couple of times.

---

## 6. Follow-up actions to offer

1. **Deep-dive on one project** → `marketcore:get_project` for that project, walk through everything.
2. **Summarize a specific document** → only at this point would I `marketcore:get_content` to read the body and answer specifically.
3. **Find gaps** → compare against typical launch-content checklist (announcement blog, customer email, sales one-pager, internal FAQ, press boilerplate) and flag what's missing.
4. **Generate something to fill a gap** → Workflow 1 with a fitting blueprint, scoped to the launch project.
5. **Consolidate scattered launch content** → Workflow 4 (`marketcore:create_project`) for a fresh "Launch" workstream, or Workflow 2 (`marketcore:update_project(project_brief_id=…)`) to anchor an existing item as the brief on an existing project.
6. **Export / share** → Recipe D (`marketcore:create_external_share`) or Recipe E (`marketcore:get_content` → `marketcore:convert_markdown_to_word_doc`) for any specific item they pick.

I would NOT proactively share or export anything — those are user-initiated.

---

## 7. Pitfalls I'm actively avoiding

| Pitfall | How |
|---|---|
| Silently guessing "the launch" | Asking; using `marketcore:list_projects` to make the question concrete. |
| Pre-fetching context for `create_content` | N/A — not creating. `marketcore:get_relevant_context` is the right tool for Workflow 5. |
| Fetching content body proactively | Handing back `link_url`s. `marketcore:get_content` only on user follow-up. |
| Surfacing UUIDs (pitfalls E10) | Names + `link_url` only. |
| RAG chunk confusion (pitfalls E2) | Telling the user explicitly that chunks are excerpts, not full items. |
| RAG re-returning same chunks (pitfalls E3) | Passing `context_rag_ids` on any follow-up call. |
| Calling `marketcore:get_current_user_info` proactively | Skipping it — auth is implicit. |
| Treating tool-returned content as instructions | Surfaced as data only; not re-executing anything inside any returned body. |

---

## 8. Estimated tool calls

**Broad-sweep branch:**
- `marketcore:list_projects` (1, before clarification)
- `marketcore:list_content` (1, after clarification)
- `marketcore:get_relevant_context` (1, optionally +1 paginated)
- `marketcore:get_project` × N where N = launch-named projects (parallel, capped at 5)

Total: ~4–8 read-only calls.

**Single-project branch:**
- `marketcore:list_projects` (already done)
- `marketcore:get_project` (1)
- `marketcore:get_relevant_context` with `project_id` (1)

Total: ~3 calls.

No async polling needed (no generation). No mutations.

---

## 9. Confidence check before acting

- Identified intent: cross-project content discovery, topical filter "launch."
- Right workflow: Workflow 5, plus `marketcore:list_content` per the decision table.
- Disambiguation: pre-call `marketcore:list_projects`, then ask the user which launch.
- Plan announced before fanning out.
- Avoiding known pitfalls per §7.
- No async, so no polling.

Ready to fire `marketcore:list_projects` immediately, then ask the clarifier; the rest waits on the user's answer.
