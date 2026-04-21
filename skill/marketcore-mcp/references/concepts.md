# MarketCore Concepts — Extended Reference

Read this when you need a deeper understanding of a specific concept than `SKILL.md` provides. Each section is independent — load only what's relevant to the current task.

---

## Content (formerly Canvas / Deliverable)

A **content item** is the unit of output in MarketCore. Every blog post, email, case study, one-pager, sales script, etc. is content.

### Two creation paths
- **Freeform content** — Generated from an AI prompt with no template. The AI is guided only by Brand Foundation, Reference Library context (relevancy-scored), and any project / document-specific context you pass. In the older app UX this was called a **Canvas** or sometimes "AI Scratchpad".
- **Blueprint-driven content** — Generated from a reusable template that supplies structure and additional AI instructions. In the older UX this was called a **Deliverable**.

The MCP server now treats both as **content** and exposes them through one tool: `create_content`. The distinction at call time is whether you pass `blueprint_uuid` (blueprint-driven, async) or just `instructions` (freeform, sync).

You can also pass `content` (your own pre-written text) to `create_content` to save a document directly with no AI generation. Useful for importing existing docs or for the agent to save its own drafted text.

### Content lifecycle
- `stage`: `in_progress` (AI is still generating, or user is editing) or `ready` (final).
- `is_ready` is also surfaced in `get_project`'s document list.
- Content can be associated with a project (`project_id`), categorized (`category_id`), and targeted (`dimension_option_ids`).

### Multi-format output
A blueprint can define multiple coordinated content formats (e.g., blog + email + in-app message). One `create_content` call generates the whole campaign as a single content item with sectioned output.

### Internal vs. exposed identifiers (for understanding API output)
- The `canvas` table has `id` (int internal PK) and `canvas_id` (the public UUID).
- The `deliverable` table has `id` (int internal PK) and `deliverable_id` (the public UUID).
- Display field name differs: canvas → `title`, deliverable → `name`.
- Privacy field name differs: canvas → `privacy` (enum `private`/`team`), deliverable → `visibility` (same enum).
- `get_project` normalizes both as a single `documents` array with `deliverable_id` as the public UUID regardless of underlying type — don't be confused if a "deliverable_id" actually points at a canvas.

---

## Blueprint

A **blueprint** is a reusable AI content template. It defines:
- Sample structure (the source markdown).
- A polished reference version (AI-generated).
- "Blueprint DNA" — a structural / tonal analysis that guides generation.
- Input instructions — guidance to the user (or agent) on what context to provide when generating from it.
- An optional content category.

### Why blueprints exist
Without a blueprint, the AI has to invent structure every time. With one, it can produce consistent, on-brand documents in a known shape — case studies that look like case studies, battle cards that look like battle cards.

### Two ways to create a blueprint

1. **Direct (`create_blueprint`)** — You provide a strong markdown sample (`source_content`). MarketCore analyzes it and generates Blueprint DNA. Takes 1–3 minutes.
2. **Draft → Finalize (`create_blueprint_draft` → `finalize_blueprint_draft`)** — You provide a prompt describing the blueprint plus initial markdown content. MarketCore drafts a template you can review/edit before finalizing. Use this when the user doesn't have a clean sample yet.

### Multi-format blueprints
A blueprint's `source_content` can contain multiple sections separated by headers (e.g. `## BLOG POST`, `## EMAIL`, `## SOCIAL`). Generation produces all sections in one go, with coordination instructions baked in. Sweet spot: 3–5 formats per blueprint.

### Community blueprints (Blueprint Exchange)
Other MarketCore users publish blueprints to a shared exchange. Use `list_community_blueprints` to browse, `get_community_blueprint_details` to inspect, and `import_community_blueprint` to clone into the user's library. Always check the exchange before creating a new blueprint from scratch.

---

## Project

A **project** is a workstream — a focused container for related content + project-scoped context + an optional brief and system prompt.

### What lives in a project
- **Documents** (`project_item` rows) — Each row links a canvas or deliverable to the project. The row has fields:
  - `purpose`: `core_output` (the project's main work product) or `supporting` (background material). Brief defaults to `supporting`.
  - `private_override`: when true, lets a normally-private document be visible at the project level. Auto-set to `true` when a private document is set as the brief.
  - `in_project_context`: whether the doc is included in project context for AI generation.
  - `sort_order`: UI ordering.
- **Context items** — Created with `add_context(project_id=...)`. Returned by `get_project` in the `context_items` array.
- **Members** — `project_member` rows. Each member has a `role`: `owner`, `editor`, or `viewer`. (Plus a separate `Collaborator` role for stakeholders with project-only access.)
- **Project brief** (optional) — A pinned canvas/deliverable that supplies high-level context. Stored as `project.project_brief_id` (an integer pointing to the `project_item.id` wrapper). Set via `create_project(project_brief_details)` or `update_project(project_id, project_brief_id=<content_uuid>)`.
- **Project system prompt** (optional) — Stored on the project record (`system_prompt` field). The PATCH endpoint accepts updates, but the in-app UI for editing it may not be exposed yet, AND the generation pipeline's use of this field is currently being verified — `update_project` does NOT yet expose this parameter.

### Visibility model
A project itself has visibility (`team` or `private`). Documents inside a project independently have their own visibility (`private`, `team`). They're orthogonal:
- *Private + In Project* = your private notes scoped to a project.
- *Team + In Project* = visible to all team members but organized in the project.

### Why projects matter for content quality
Projects are the *primary* mechanism for context isolation. Content created inside a project automatically gets the project's context layered on top of Brand Foundation + Reference Library (see "Context layering" below). Content created outside a project gets only Brand Foundation + Reference Library. **For any multi-document initiative, recommend the user create a project first.**

### The brief data model — exact mechanics

The brief is NOT a context item. It's a regular Canvas or Deliverable that's already in the project's documents, "pinned" via `project.project_brief_id`. The chain is:

```
project.project_brief_id (int, nullable)
   ↓ FK to
project_item.id (int)
   ↓ project_item has either canvas_id (int) or deliverable_id (int)
canvas.id OR deliverable.id (int)
   ↓ each has a public UUID
canvas.canvas_id (uuid) OR deliverable.deliverable_id (uuid)
```

When you call `update_project(project_id, project_brief_id=<content_uuid>)`:
1. The tool calls `get_unified_deliverable(item_uuid=<content_uuid>)` which checks both tables.
2. Determines whether it's a canvas or deliverable.
3. Looks up the `project_item` wrapper row (project_id + canvas_id/deliverable_id match).
4. Sets `project.project_brief_id` to that wrapper's integer ID.
5. The PATCH endpoint also auto-sets `project_item.private_override = true` if the underlying content was private (so other project members can see it as the brief).

**Therefore the precondition: the content must already be a `project_item` of this project.** No wrapper = no brief setting.

---

## Context items, Reference Library, Context Hub, and Collections

A **context item** is a reference document that informs AI generation. Brand guidelines, persona research, competitor analysis, product specs, customer interviews — anything that helps the AI produce more accurate, on-brand content.

### Three places context lives

1. **Reference Library** (a tab inside the Context Hub nav page in the app) — Top-level, team-wide context items. Always available across all projects via relevancy scoring. Tag with targeting dimensions for better retrieval. Created by `add_context` with no `project_id`.

2. **Project context** (the project's "Context Items" tab) — Context items scoped to a single project. Always included when generating content in that project (no relevancy filtering — it's all included). Created by `add_context` with `project_id`.

3. **Document-specific context** — One-off context attached to a single content generation: file uploads, live research, referenced docs. Not persisted as reusable context. Passed at generation time via parameters on `create_content`.

### Reference Library vs. Context Hub — these are NOT synonyms

In the current app:
- **Context Hub** = the containing nav page. It has tabs: **Overview**, **Reference Library**, **Brand Foundation**, **Targeting Dimensions**, **Context Collections**, **Findings**.
- **Reference Library** = the specific tab listing top-level context items.

Use "Reference Library" when the user means top-level context items. Use "Context Hub" when you mean the containing page or are unsure which tab they meant. Older marketing copy and Strapi pages may use "Context Hub" loosely to mean Reference Library — translate.

### Context item attributes
- `content_type` enum: `webpage` | `file` | `manual` | `call_transcript` (the source type).
- `tag_id` (optional): for organizing within the Reference Library; team-scoped.
- `collection_id` (optional): file under a Context Collection.

### Best practices for what goes where
- **Reference Library**: brand voice docs, writing standards, ongoing persona definitions, general product info, company positioning.
- **Project context**: research specific to one initiative, customer interview transcripts for one launch, competitor materials for one positioning exercise.
- **Document-specific**: data point you want in one specific blog post, one-time research finding, a single press release you want to react to.

### Context Collections
Optional folders inside the Reference Library. Created with `create_context_collection`. Listed with `list_context_collections`. When adding a context item with `collection_id`, it goes into that collection. Collections can be private to the creator or shared with the team. A project can have **default collections** automatically attached to its generations.

### Promoting context up the hierarchy
Project context that proves universally useful can be promoted to the Reference Library (manual operation in-app). The agent doesn't need to do this — it's a user UX flow.

---

## Brand Foundation

A separate tab inside Context Hub. Stores the team's company overview, brand voice, writing style, and writing examples — the most foundational AI context. Read with `get_core_context` (the MCP tool name; the in-app tab is **Brand Foundation**).

This is **automatically injected** as Layer 1 of the four-layer context model (see below) into every content generation. You don't need to add it manually.

Read it explicitly when:
- The user wants to know what brand voice the AI is using.
- You're generating content *outside* MarketCore (e.g. drafting a Discord message in Chris's voice elsewhere) and want the brand context.
- You're debugging why content "doesn't sound right" — maybe the Brand Foundation is stale.

The response is a single markdown string with all four sections concatenated.

---

## The four-layer context model (use the actual UI labels)

When you generate content **inside a project**, MarketCore automatically combines (these are the in-app labels from the InlineCanvasCreator's context picker — quote them when explaining context to the user):

1. **Brand Foundation** — *"Includes your company's core context such as company overview, brand voice, and writing style."* Always on. Pulled via `get_core_context`.
2. **Reference Library** — *"Includes your website content, plus any other context and messaging documents you've added."* Always on. Filtered by relevancy scoring against the prompt.
3. **Project Context** (only when in a project) — *"Includes project brief and any project-specific context items."* Always on inside a project. Both the project brief and project-scoped context items count.
4. **Context Collections** — User-toggleable per generation. Pass `collection_ids` on `create_content`. A project can have default collections (set in-app under "Default Collections").

When you generate content **outside a project**, only Layers 1, 2, and 4 apply.

### Implications for the agent
- Don't repeat brand voice in your prompt. Layer 1 has it.
- Don't re-explain a project's strategic angle. Layers 2–3 have it (the brief + project context items).
- *Do* add anything one-off via document-specific context (Layer 4 collections, or per-call dimensions).
- If the user asks you to add a piece of info that they'll need across many documents, it belongs in Layer 2 (Reference Library) — call `add_context` separately, not as part of the generation.

---

## Targeting Dimensions

Categorical attributes the user has defined for their team — typically things like Buying Stage, Persona, Industry, Product Line. Each dimension has selectable options (e.g. Persona: VP Marketing, Director of Sales, CMO).

Surface them via `list_targeting_dimensions`. Pass option IDs (not dimension IDs) as `dimension_option_ids` to `create_content` to shape the output for a specific audience.

When the user mentions an audience attribute (any kind), check whether it maps to a targeting dimension option and propose using it.

---

## Content Categories

A taxonomy slot for blueprints and content. Examples: "GTM Strategy", "Product Launch", "Sales Enablement", "Customer Marketing".

- **Required** when creating a blueprint (`category_id`).
- **Optional** when creating content (`category_id`).

Get them with `list_content_categories`. Treat them as an organizational hint — they don't change generation behavior, but they help the user find their work later.

---

## Project member roles and Collaborators

Projects have members (`project_member` rows) with one of these roles:
- `owner` — full control, including deletion.
- `editor` — create/edit content and context in the project.
- `viewer` — read-only.

Plus a separate **Collaborator** role (set in-app) for stakeholders who get project-scoped access without a full Creator seat. Collaborators can view, edit, comment, and use the AI Assistant within their assigned project but can't see other team work.

The `update_project` MCP tool requires the calling user to have at least `editor` access (it calls `Check Team Role Authorization` with `allow_viewer: false`). Viewers can call read-only tools but not mutations.

---

## Workflows

Workflows orchestrate multi-step content generation — running several create-content operations in a defined sequence with shared context. Newer feature with fewer published patterns yet.

Tools:
- `list_workflows` — list defined workflows.
- `get_workflow` — fetch a workflow definition.
- `create_workflow`, `update_workflow` — author workflows.
- `run_workflow` — execute one.
- `get_workflow_runs` — see past runs.

Treat workflows as advanced — for most user requests, single content / blueprint operations are simpler. Suggest a workflow only when the user describes a repeatable multi-step generation process they want to automate.

---

## Sharing and Export

- **External share** (`create_external_share`) — Generates a public URL anyone can use to view the content (no MarketCore account required). Optional `expires_at` (Unix timestamp). Use when the user wants to share a draft externally.
- **Word export** (`convert_markdown_to_word_doc`) — Renders markdown to a `.docx` file and returns a download URL. Optional `filename` and `document_url` (footer link).

Both are read/transform operations on existing content — they don't generate new content.
