# Plan: Multi-Project Content Discovery — "What content do I have related to the launch?"

## Interpretation of the request

The user wants a cross-project view of content items whose subject relates to "the launch" (likely a product/feature launch — ambiguous, but I'll treat "launch" as a keyword/topic to filter on rather than asking up front, since the goal is to *show what's already been written* and then let them refine). They explicitly want coverage **across all projects**, not just one. So the strategy is:

1. Enumerate projects.
2. Pull the full content list (which is account-wide).
3. Cross-reference content to projects.
4. Filter/score by relevance to "launch" (title/topic match + RAG search across context as a backstop).
5. Present grouped by project, with unassigned content in its own bucket.

---

## 1. First message back to user (verbatim)

> On it — I'll pull every piece of content across your projects and surface anything related to "the launch." Give me a few seconds to scan your library.
>
> Quick note: I'll match on titles, topics, and content body. If "launch" means a specific product or campaign (e.g. "Q2 launch," "Mercury launch"), tell me and I'll narrow the filter — otherwise I'll cast a wide net and you can tell me which hits matter.

---

## 2. Ordered tool calls

### Step 1 — Enumerate projects (so we can group results)

**Tool:** `list_projects`
**Params:** *(none / defaults)*
**Rationale:** Need the full project roster up front so we can attribute each content item to its project and present results grouped. Cheap call; do it first.

### Step 2 — Pull all content (account-wide, paginated if needed)

**Tool:** `list_content`
**Params:** *(default — pull first page; paginate if the response indicates more)*
**Rationale:** `list_content` returns content across the whole account, not scoped per project, so a single call (or a few paginated calls) is more efficient than calling `get_project` for every project and unioning their `documents` arrays. Each item should include `project_id` (or null), title, category, created date, and status — enough to filter and group.

**Filtering after the call:**
- Keyword match on `title` and any `topic`/`prompt`/`description` field for: `launch`, `launching`, `launched`, `go-live`, `GA`, `release` (close synonyms — keep narrow to avoid noise).
- Keep all matches; do NOT discard non-matches yet — we'll show the user the count of "launch-related vs total" so they have context.

### Step 3 — RAG backstop for body-text matches

**Tool:** `get_relevant_context`
**Params:** `query: "launch"` (and a second pass with `query: "product launch announcement go-to-market"` for semantic breadth)
**Rationale:** `list_content` only exposes metadata. A piece titled "Spring Announcement" with launch-heavy body text would be missed by title-only filtering. RAG search across the user's library catches semantic matches the keyword pass misses. Cross-reference any returned content IDs with the `list_content` result and merge.

### Step 4 — (Conditional) Pull full text for top hits

**Tool:** `get_content`
**Params:** `content_id: <each top hit ID>` — only for the **top 3-5 most relevant** items, not every match.
**Rationale:** Lets me write a one-line summary per top item ("Blog post — soft-launch announcement, 850 words, draft status") rather than just listing titles. Cap at 5 to avoid blowing time/tokens on what may be a wide net.
**Skip condition:** If `list_content` already returns a `summary`/`excerpt` field, skip this step — metadata is enough for a first-pass overview.

### Step 5 — (Conditional) Project detail lookup for context

**Tool:** `get_project`
**Params:** `project_id: <id>` — only for projects that have launch-related content AND whose name doesn't make their purpose obvious.
**Rationale:** Helps me say "Project X (Mercury launch campaign) has 4 launch-related pieces" instead of just the project name. Skip for projects with self-explanatory names.

---

## 3. How I'd organize and present the results

### Structure: grouped by project, sorted by recency within each group.

```
Found N pieces of launch-related content across M projects (out of TOTAL total content items in your library).

── Project: Mercury Launch Campaign ──
  • [Blog Post] "Mercury is here: introducing X" — Final, 2026-04-15
  • [Email] "Launch day announcement to existing customers" — Draft, 2026-04-12
  • [Social] "Launch teaser thread (Twitter/X)" — Final, 2026-04-08

── Project: Q2 Marketing ──
  • [Landing Page] "Mercury launch landing page copy" — Draft, 2026-04-10
  • [Press Release] "Mercury press release v3" — Final, 2026-04-09

── No project assigned ──
  • [Blog Post] "Lessons from our last launch" — Final, 2026-02-20

── Probable matches (semantic, not keyword) ──
  • [Email] "We're going live next week" (in: Q2 Marketing) — Final, 2026-04-11
```

### Presentation rules
- Lead with a **count summary** so the user knows the scope at a glance.
- Group by project; "no project" gets its own bucket.
- Each line: `[Category] "Title" — Status, Date`.
- Separate keyword-match results from semantic-only matches so the user can see why each hit is included.
- Cap at ~25 items shown; if there are more, summarize the overflow ("…and 12 more in Q2 Marketing — say 'show all in Q2' if you want them").
- No raw IDs in the user-facing output unless they ask — keep it scannable.

---

## 4. Follow-up actions to offer

After presenting the list, end with a short menu:

> Want me to:
> 1. **Narrow the filter** to a specific launch (e.g. "just Mercury", "just the Q2 launch")?
> 2. **Pull full text** of any specific piece — just point me at the title?
> 3. **Show what's still in draft** vs already finalized, so you can see what needs finishing?
> 4. **Generate a launch-content gap analysis** — e.g. "you have a blog post and email but no social copy"?
> 5. **Bundle these into a new project** if they're scattered and you want a single home?
> 6. **Create a share link** for any of these to send to a teammate?

Pick the question that gets us to action; don't over-offer.

---

## Notes on what I'm deliberately NOT doing

- **Not** asking clarifying questions before the first tool call. The user asked a discoverable question; the right move is to *show them what's there* and then let them refine. Asking "which launch?" first would feel slow.
- **Not** calling `get_project` for every project. That's N+1 when `list_content` gives me the cross-project view in one shot.
- **Not** calling `get_content` for every match — only top hits get full-text pulls. Otherwise we burn tokens summarizing things the user can see by title.
- **Not** invoking `get_core_context` or `list_context_collections` — those are about brand voice and reference material, not the user's authored content.
- **Not** touching `list_blueprints` / `list_workflows` — they're templates and pipelines, not "content I have written."
