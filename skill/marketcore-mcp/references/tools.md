# Tool Reference

Full per-tool reference for every MarketCore MCP tool an agent can call. Organized by category, with required parameters, key outputs, and "call this when..." guidance.

For workflow-level guidance ("which tools in what order"), see `workflows.md`. For known limitations, see `pitfalls.md`.

---

## Account

### `get_current_user_info`

**When to call:** ONLY when the user explicitly asks about their MarketCore profile, plan, usage, or active team. Don't call proactively to "verify auth" — every other tool call is already scoped to the authenticated user, so any successful call implicitly confirms auth.

**Parameters:** None.

**Key outputs:**
- `name`, `email` — user identity.
- `active_team_name`, `active_team_role` — which team the calls operate against.
- `subscription_status`, `plan_name`, `plan_slug` — billing context.
- `usage` object: `ai_credits`, `ai_credits_max`, `deliverables`, `deliverables_max`, `canvas_sessions`, `canvas_sessions_max`, `ai_updates`, `ai_updates_max` (max=0 means unlimited; surface "deliverables" to the user as "content").

---

## Context — Reference Library + project + retrieval

### `get_core_context`

**When to call:** When the user asks about their brand voice, when you need brand foundation outside of MarketCore content generation (e.g. drafting a tweet in their voice elsewhere), or when debugging "why does the content sound off?"

**Parameters:** None.

**Key outputs:** `core_context` — single markdown string with company overview, brand voice, writing style, and writing examples.

**Note:** This is automatically pulled into every content generation. Don't manually inject it into `instructions`.

---

### `list_context_collections`

**When to call:** Before `add_context` (to see if a collection fits), or when the user asks "what collections do I have?"

**Parameters:** None.

**Key outputs:** Array of `{ id, name, description, is_private, item_count, link_url }`. Use `id` as `collection_id` in `add_context`.

---

### `create_context_collection`

**When to call:** When the user wants a new collection in their Reference Library to organize related context items. Confirm with `list_context_collections` first that one doesn't already exist.

**Parameters:**
- `name` (required) — descriptive name.
- `description` (required) — what's in it.
- `is_private` (required) — `true` (creator-only) or `false` (team).

**Key outputs:** `{ id, name, description, is_private, created_at, link_url }`. Use the new `id` as `collection_id` on subsequent `add_context` calls.

---

### `add_context`

**When to call:** When the user wants to store a reference document (brand guidelines, persona, competitor analysis, product spec, etc.) in their library.

**Critical disambiguation before calling:**
- Top-level Reference Library item → omit `project_id`.
- Project-scoped context item → include `project_id`.
- Document-specific context (one-off, not stored) → don't call this; instead pass collections/dimensions on `create_content`.
- Project brief → see `pitfalls.md` P1 (no MCP tool currently sets a brief on an existing project).

**Parameters:**
- `name` (required), `content` (required).
- `collection_id` (optional) — for top-level items, file under a collection.
- `project_id` (optional) — scope to a project. Omit for top-level.

**Key outputs:** `{ id, name, content, word_count, collection_id?, project_id?, created_at, link_url }`.

---

### `get_relevant_context`

**When to call:** ONLY when the user is asking a question or ideating and YOU need to read context to answer them. Examples: "what do we have on competitor X?", "remind me what our healthcare positioning is", "have we written anything about pricing yet?".

**When NOT to call:** Before `create_content`. `create_content` already pulls all relevant context internally — pre-fetching is wasted work.

**Parameters:**
- `prompt` (required) — descriptive search string.
- `collection_ids` (optional) — array of collection IDs to scope.
- `project_id` (optional) — scope to a project.
- `dimension_option_ids` (optional) — scope by targeting dimensions.
- `context_rag_ids` (optional) — array of previously-returned chunk IDs to exclude (pagination).

**Key outputs:**
- `relevant_context` — concatenated markdown of matching chunks.
- `context_rag_ids` — chunk IDs returned (use for pagination).
- `context_item_ids` — parent context item IDs.

**Note:** Returns chunks (~few hundred words each), not full items.

---

## Reference taxonomies

### `list_content_categories`

**When to call:** Before creating a blueprint (required category) or before creating content (optional but helpful organizationally).

**Parameters:** None.

**Key outputs:** Array of `{ id, name, ... }`. Use `id` as `category_id`.

---

### `list_targeting_dimensions`

**When to call:** When the user mentions an audience attribute (persona, industry, buying stage, etc.) and you want to target the generation.

**Parameters:** None.

**Key outputs:** Array of dimensions, each with an `options` array. Pass *option* IDs (not dimension IDs) as `dimension_option_ids` to `create_content`.

---

## Blueprints

### `list_blueprints`

**When to call:** Before creating content (to see if an existing blueprint fits), before creating a new blueprint (avoid duplicates), or when the user asks "what templates do I have?"

**Parameters:** None.

**Key outputs:** `{ blueprints: [...], blueprint_drafts: [...] }`. Check both — drafts may match the user's request.

---

### `get_blueprint`

**When to call:** When you need full details for a specific blueprint (DNA, source content, instructions) — typically before using it for generation if the user is unsure what it produces.

**Parameters:** `blueprint_uuid` (required).

**Key outputs:** `name`, `summary`, `blueprint_uuid`, `source_content`, `reference_content`, `blueprint_dna`, `input_instructions`, `category`, `team_visibility`, `web_url`, `created_at`.

---

### `create_blueprint`

**When to call:** When the user has a strong markdown sample document and wants a reusable template from it. Takes 1–3 minutes.

**Parameters:**
- `name` (required).
- `category_id` (required) — from `list_content_categories`.
- `source_content` (required) — well-structured markdown template.

**Key outputs:** `name`, `summary`, `blueprint_uuid` (use with `create_content`), `blueprint_dna`, `source_content`, `reference_content`, `input_instructions`, `category`, `link_url`, `team_visibility`, `created_at`.

**Common errors:** Invalid `category_id` (use `list_content_categories` first); empty `source_content`.

---

### `create_blueprint_draft`

**When to call:** When the user describes a blueprint they want but doesn't have a clean sample yet. Creates a reviewable draft, not a published blueprint. Pair with `finalize_blueprint_draft` to publish.

**Parameters:**
- `name` (required) — draft name.
- `instructions` (required) — describe the blueprint to create.
- `content` (required) — initial markdown.
- `category_id` (optional).

**Key outputs:** `id`, `uuid`, `title`, `content` (AI-generated draft), `link_url`.

---

### `finalize_blueprint_draft`

**When to call:** After the user has reviewed a draft and wants to publish it as a real blueprint. Takes 1–3 minutes.

**Parameters:**
- `draft_uuid` (required) — from `create_blueprint_draft`.
- `name` (optional) — override name.
- `category_id` (optional) — override category.

**Key outputs:** Same shape as `create_blueprint` — `blueprint_uuid` is what you'll use with `create_content`.

---

## Community blueprints (Blueprint Exchange)

### `list_community_blueprints`

**When to call:** Before creating a blueprint from scratch — check the exchange first.

**Parameters:** None.

**Key outputs:** Array of community blueprints with names, summaries, contributor info, categories, and `blueprint_exchange_id` (use for the next two tools).

---

### `get_community_blueprint_details`

**When to call:** To inspect a community blueprint before importing.

**Parameters:** `blueprint_exchange_id`.

**Key outputs:** Full content, style guide, input instructions, contributor details.

---

### `import_community_blueprint`

**When to call:** To clone a community blueprint into the user's library. Returns a fresh team-level UUID.

**Parameters:** `blueprint_exchange_id`.

**Key outputs:** `id`, `name`, `uuid` (use with `create_content`), `content`, `team_visibility`, `imported_exchange_id`, `created_at`.

---

## Content

### `create_content`

**When to call:** Whenever the user wants to create a document — from scratch, from a blueprint, or by saving their own text.

**This tool internally pulls all relevant context** (Brand Foundation + Reference Library via relevancy + Project Context if `project_id` is set + any `collection_ids` you pass). Don't pre-fetch with `get_relevant_context` — it's wasted work.

**Three modes (mutually exclusive):**

| Mode | Parameters | Behavior |
|---|---|---|
| Save user's text | `content` only | Saves directly. Sync. No AI generation. |
| Freeform AI | `instructions` only | AI generates from prompt. Sync (1–3 min). |
| Blueprint-driven | `instructions` + `blueprint_uuid` | AI generates from template. **Async** (3–5 min). |

`content` cannot be combined with `blueprint_uuid` or `instructions`.

**Optional shaping parameters (any mode):**
- `project_id` — associate with a project (and pull its Project Context into generation).
- `category_id` — content category (organizational only).
- `collection_ids` — extra context collections to include on top of the always-on layers.
- `dimension_option_ids` — targeting dimension *option* IDs.
- `use_extended_thinking` — boolean, only for sync mode with `instructions` (no blueprint).

**Outputs (sync modes):** `{ id, content_id (UUID), title, content, link_url, created_at }`.

**Outputs (async mode):** `{ generation_id }` only — poll `get_generation_status` until completed.

**After completion, hand `link_url` to the user.** Don't call `get_content` to fetch the body — the user reviews the content in MarketCore via the link.

**Common errors:** Both `content` and `instructions` set; `content` + `blueprint_uuid`; invalid `blueprint_uuid`; sync timeout (it's normal — wait).

---

### `get_generation_status`

**When to call:** After every async `create_content` (with `blueprint_uuid`). Poll every ~30 seconds.

**Parameters:** `generation_id` (required).

**Key outputs:**
- `status` — one of `pending`, `gathering context`, `processing`, `completed`, `failed`.
- `content` — present when completed. Contains `link_url` (give this to the user) and `content_id` (only needed if the user later asks a question requiring the body — then pass to `get_content`).

**When status is `completed`: hand `content.link_url` to the user.** Don't call `get_content` just to "show what was generated" — the user opens the content in MarketCore via the link.

---

### `list_content`

**When to call:** When the user asks "what content do I have?" or "show me my documents."

**Parameters:** None.

**Key outputs:** Array of content items with IDs, names, categories, creation dates, metadata.

---

### `get_content`

**When to call:** ONLY when YOU need to read the content body to answer a question the user asked about that content. Examples: "summarize what my latest case study says about pricing", "does the messaging doc mention HIPAA?", "what's the main argument in my Q3 launch blog?".

Also call when you need the markdown body to feed another tool — e.g. `convert_markdown_to_word_doc` for Word export.

**When NOT to call:** After `create_content` or `get_generation_status` returns `completed` — just hand the user the link. The user reviews the content in MarketCore. Cora doesn't read it for them.

**Parameters:** `content_id` (required, UUID).

**Key outputs:** `name`, `content` (full markdown), `content_id`, `stage` (`in_progress` | `ready`), `category`, `visibility`, `link_url`.

---

## Sharing & Export

### `create_external_share`

**When to call:** When the user wants a public share link (no login required) for a content item.

**Parameters:**
- `content_id` (required).
- `expires_at` (optional, Unix timestamp).

**Key outputs:** `share_link` — public URL.

**Note:** Content must be `completed` before sharing — wait for async generations to finish first.

---

### `convert_markdown_to_word_doc`

**When to call:** When the user wants a `.docx` download of a content item.

**Parameters:**
- `markdown_content` (required) — fetch via `get_content` first; this tool does NOT auto-fetch by ID.
- `filename` (optional) — without extension.
- `document_url` (optional) — embed as footer link to the original.

**Key outputs:** `filename`, `download_url`.

---

## Projects

### `list_projects`

**When to call:** Before creating a project, when the user mentions an initiative by name, or when you're deciding whether content should be scoped to a project.

**Parameters:** None.

**Key outputs:** Array of projects with `id` (UUID), `name`, `link_url`, `visibility`, `status`, `deliverable_count` (= content count), `created_by`, `member_count`.

---

### `get_project`

**When to call:** When you need to see what's already in a project (documents, context items, members).

**Parameters:** `project_id` (required, UUID).

**Key outputs:** `id`, `name`, `status`, `visibility`, `members`, `documents` (each with `deliverable_id`, `name`, `is_ready`, `category`, `purpose`, `in_project_context`, `web_url`), `context_items`, `created_at`.

**Note:** Document UUIDs are exposed as `deliverable_id` — surface to the user as "content ID".

---

### `create_project`

**When to call:** When the user wants a new workstream / initiative container.

**Parameters:**
- `name` (required).
- `visibility` (optional) — `team` (default) or `private`.
- `project_brief_details` (optional) — supplying this auto-generates a project brief Canvas/Deliverable inside the project AND points `project_brief_id` at it. Convenient one-shot for new projects.

**Key outputs:** `project_id`, `name`, `link_url`.

**Note:** You can also set or change the brief on an existing project via `update_project` (see below), so seeding the brief at creation time is convenient but not required.

---

### `update_project`

**When to call:** When the user wants to update mutable fields on an existing project — name, visibility, status, or the project brief. PATCH semantics: only fields you pass are modified.

**Parameters:**
- `project_id` (required, UUID) — get from `list_projects` or `get_project`.
- `name` (optional, text, trimmed) — new project name. Empty/whitespace is silently treated as unset (no clobber). Must be non-empty when provided meaningfully.
- `visibility` (optional, enum) — `team` or `private`.
- `status` (optional, enum) — `active` or `archived`. Setting `active` requires available active-project usage on the team's plan.
- `project_brief_id` (optional, UUID) — content UUID (canvas or deliverable) to set as the project's brief. **Works whether or not the content is already in the project** — if not, the tool attaches it AND sets the brief in one call. You don't need to add the document separately first; you don't need to check `get_project` first.

**Key outputs:** `success` (bool), `message` (text), `project` (the updated project record). Errors return `success: false` with a diagnostic `error.message`.

**Common errors:**
- `"Project not found in your current team."` — wrong project ID, or the project belongs to a different team than the user's active team.
- `"Document not found"` — `project_brief_id` UUID isn't in either canvases or deliverables.

**Internal behavior worth knowing:**
- Auth: requires editor or owner role on the team (calls `Check Team Role Authorization` with `allow_viewer: false` first).
- Brief auto-attach: if the content UUID you pass isn't already a document in the project, the tool calls the projects/documents POST endpoint to attach it, then resolves the new wrapper id and sets the brief. You only need to make one call.
- The underlying PATCH endpoint auto-sets `project_item.private_override = true` when a private content is set as the brief, so other project members can see it.
- The endpoint also accepts `system_prompt` (deprecated — superseded by the project brief; do NOT try to use it) and `default_collection_ids` (deferred — not yet exposed via `update_project`).

---

## Workflows

Workflows are a newer feature — fewer published patterns. The shape below is current; check via `list_workflows` for what the user already has.

### `list_workflows`

**When to call:** When the user asks about workflows or before creating a new one.

**Parameters:** None.

**Key outputs:** Array of workflow definitions.

---

### `get_workflow`

**When to call:** To inspect a specific workflow before running or editing it.

**Parameters:** `workflow_id`.

---

### `create_workflow`

**When to call:** When the user describes a repeatable multi-step generation process they want to automate.

**Parameters:** Workflow definition (steps, triggers, parameters).

---

### `update_workflow`

**When to call:** To modify an existing workflow.

---

### `run_workflow`

**When to call:** To execute a workflow.

---

### `get_workflow_runs`

**When to call:** To see the history of workflow executions.

---

## Quick categorization summary

| Need | Tool |
|---|---|
| User asks about their profile / plan / usage / active team | `get_current_user_info` (otherwise don't call it) |
| Get the brand foundation | `get_core_context` |
| User asks a question about their library / ideating | `get_relevant_context` (NOT for pre-fetching before `create_content`) |
| List collections | `list_context_collections` |
| Add a reference doc | `add_context` |
| Find templates | `list_blueprints`, `list_community_blueprints` |
| Make a template | `create_blueprint` (with sample) or `create_blueprint_draft` → `finalize_blueprint_draft` (from prompt) |
| Generate a document | `create_content` (it pulls all relevant context internally — don't pre-fetch with `get_relevant_context`) |
| Wait for async generation | `get_generation_status` (then hand `link_url` to user — DON'T call `get_content`) |
| List content | `list_content` |
| Read content body to answer a question about it | `get_content` (only for Q&A about content; not after generation) |
| Share / export | `create_external_share`, `convert_markdown_to_word_doc` (Word export legitimately needs `get_content` first) |
| Manage workstreams | `list_projects`, `get_project`, `create_project`, `update_project` |
| Set / change a project brief on an existing project | `update_project` (with `project_brief_id` — works whether or not the content is already in the project) |
| Rename, archive, or change project visibility | `update_project` |
| Multi-step automation | `list_workflows`, `run_workflow`, etc. |
