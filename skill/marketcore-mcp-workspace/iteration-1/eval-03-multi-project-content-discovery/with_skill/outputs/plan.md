# Plan: Multi-project content discovery for "the launch"

## Interpretation of the request

The user wants to see **what content (deliverables / canvases) has already been written** that relates to "the launch," across **all** their projects. Two ambiguities to flag and resolve before doing anything destructive — neither blocks discovery, but both shape how I present results:

1. **"The launch" is underspecified.** It could be one specific launch they have in mind, or "any launch-themed work." Since they said "across all my projects," I'll assume the broader interpretation first: surface every project that looks launch-related plus any standalone content matching launch themes, and let them narrow down.
2. **"Content" in MarketCore = both freeform content (canvases) and blueprint-driven content (deliverables).** The MCP unifies both under `list_content`, so a single sweep covers both. I will not expose the legacy "deliverable" / "canvas" terminology to the user (per skill §6.3) — I'll just say "content" or refer to specific items by title.

Note: `get_relevant_context` would search **context items** (Reference Library + project context) — that's reference material, not the user's authored content. The user asked about content they've *written*, so this is a `list_projects` + `list_content` job, not a RAG-search job. (I'll mention `get_relevant_context` as a follow-up if they want to also see source/research material on the launch.)

---

## 1. First message back to user (verbatim)

> Quick check before I dig in: by "the launch" do you mean one specific launch (e.g. a product/feature name I should anchor on), or do you want me to sweep up anything launch-themed across all your projects? Either way I can start now — I'll list your projects, scan the content in each one, and pull in any standalone content with launch-related titles. I'll group results by project so you can see what's where. Kicking off the sweep now and I'll refine once you tell me which read you want.

(Rationale: per skill §3.1 / §3.2 I state the plan and surface the ambiguity, but I don't *block* on it — discovery is read-only and the broad sweep is useful regardless. The user can narrow after seeing the inventory.)

---

## 2. Ordered tool calls

### Step 1 — Confirm session

**Tool:** `mcp__marketcore-dev__get_current_user_info`
**Params:** none.
**Rationale:** Per skill §1, call once at the start of a session to confirm auth and surface the active team. Cheap and prevents downstream "why did nothing return" confusion if there's an auth issue.

### Step 2 — Enumerate all projects

**Tool:** `mcp__marketcore-dev__list_projects`
**Params:** none (defaults — return all visible projects).
**Rationale:** I need the full project inventory for two reasons:
- (a) Identify launch-themed projects by name/description (e.g. anything with "launch", "GA", "release", "go-live", "v1.0", a product codename, etc.).
- (b) Get every `project_id` so I can call `get_project` on the launch-related ones to pull their associated content list.

### Step 3 — Enumerate all content (sweep)

**Tool:** `mcp__marketcore-dev__list_content`
**Params:** none (default page) — if pagination is supported and there are many items, iterate.
**Rationale:** A single flat list of all content the user has authored, regardless of project. I'll filter client-side by:
- Title containing launch-related keywords (`launch`, `release`, `GA`, `announcement`, `unveil`, `go-live`, `v1`, plus any product/feature names that surface in project titles from Step 2).
- Project association (I'll cross-reference each content item's `project_id` against the launch-themed projects from Step 2).
This is more efficient than calling `get_project` on every project just to list its content — `list_content` returns everything in one shot.

### Step 4 — Drill into projects that look launch-related (parallel)

**Tool:** `mcp__marketcore-dev__get_project` (one call per launch-themed project, issued in parallel in a single message)
**Params:** `project_id=<id>` for each launch-themed project identified in Step 2.
**Rationale:**
- `get_project` returns the full `documents` array (with `purpose: core_output` vs `supporting`) and the `project_brief_id`. This tells me which document is the *strategic anchor* (the brief) vs main work products vs background.
- It also confirms whether items I matched in Step 3 by title actually live inside a launch project, vs being standalone content that just *mentions* a launch.
- Parallel issuance keeps latency low — these are independent reads.

### Step 5 — (Conditional) Pull short previews for the most relevant items

**Tool:** `mcp__marketcore-dev__get_content`
**Params:** `content_id=<uuid>` for the top ~3–5 items the user is most likely to care about (project briefs of launch projects + any content explicitly titled like a launch announcement / launch plan / launch one-pager).
**Rationale:** Titles alone often aren't enough to know what's there. A 2–3 sentence excerpt per top hit lets the user pick what to open without me dumping the full corpus. I will NOT call `get_content` on every match — that's wasteful. Cap at ~5 to keep the response readable.

### Step 6 — (Skipped by default; offer as follow-up) RAG over context items

**Tool:** `mcp__marketcore-dev__get_relevant_context`
**Params:** `prompt="the launch"` (refined with whatever specific launch the user names), optionally scoped to a project.
**Rationale:** This searches **reference material** (Reference Library + project context), not authored content. The user asked about content they've *written*, so I'll skip this in the initial sweep but explicitly offer it as a follow-up: "Want me to also pull related research / context items on the launch from your Reference Library?"

---

## 3. How I'd organize and present the results

A single response, structured for fast scanning. The user's mental model is "what do I have, where does it live" — match that.

### Structure

**Headline summary (1 line):** "Found N pieces of content across M launch-related projects, plus K standalone items that mention the launch."

**Section 1 — Launch-related projects (grouped)**
For each project that looks launch-themed, a subsection:

```
### [Project Name]  (status: active | archived; visibility)
Brief: [title of project brief, if set] — [1-line excerpt if I pulled it in Step 5]

Content in this project:
- [Title] — [purpose: core output | supporting] — [last updated date]
- [Title] — [purpose] — [date]
  ...
```

I'll mark the brief explicitly so they see the strategic anchor first, then list `core_output` items, then `supporting` items.

**Section 2 — Standalone content that mentions the launch**
Content with launch-related titles that isn't in a project (or is in a non-launch project). Flat list with title, last-updated date, and project association if any. Useful for catching "I wrote this one-pager last quarter and forgot to file it."

**Section 3 — Possibly related (judgment-call matches)**
Items where the title is ambiguous (e.g. "Q2 announcement deck" — could be launch, could be quarterly update). Flagged separately so the main lists stay high-confidence.

**Section 4 — What I didn't search**
One sentence: "I didn't search your Reference Library or project context items — that's source/research material rather than content you've written. Want me to sweep that too?"

### Formatting choices

- **Titles linked to MarketCore URLs** if `link_url` is in the response payload — saves the user a click.
- **Dates in relative form** ("3 weeks ago") for recency scanning, plus absolute date in parens.
- **No raw UUIDs** in the body — they're noise unless the user needs to act on a specific item, in which case I'll surface IDs in the follow-up step.
- **Translate field names** per skill §3.4 — the user sees "content," "brief," "supporting material," not `deliverable_id` / `core_output` / `project_item`.

---

## 4. Follow-up actions to offer

A short menu at the bottom of the response, picking the most likely next moves so the user doesn't have to formulate them:

1. **"Want a quick excerpt from any of these?"** — Offer to pull the full text of any specific item via `get_content` and summarize.
2. **"Should I narrow to one specific launch?"** — If they name it, I'll re-run with tighter keyword filtering.
3. **"Want me to also search your Reference Library and project context for launch-related research?"** — Triggers `get_relevant_context` with `prompt="<the launch name>"`, optionally scoped to launch projects.
4. **"Want me to draft something new based on what's already there?"** — E.g. a launch one-pager that pulls from the existing brief + supporting content. Would be `list_blueprints` → `create_content(blueprint_uuid=..., project_id=...)` (per skill §4.1).
5. **"Want a shareable bundle?"** — If they want to send this inventory to a stakeholder, offer `create_external_share` on the most relevant items, or `convert_markdown_to_word_doc` for an export.
6. **"Want me to consolidate?"** — If standalone launch content should live inside a launch project, flag that — though note (per skill §6.1) that there's currently no MCP tool to attach existing standalone content to a project; that's an in-app move.

I'll present these as a short bulleted list, not a wall of text — the user picks what they want.
