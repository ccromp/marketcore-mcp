# Long-tail workflow recipes

The 5 most common workflows are in `SKILL.md`. This file covers less-frequent recipes. Read the relevant section only when the user's request matches.

---

## Recipe A — Create a new blueprint from scratch

**Goal.** Author a reusable AI template for a content type the user produces repeatedly.

**Steps.**
1. **Check the Blueprint Exchange first.** `marketcore:list_community_blueprints` — is there a community template that fits? If yes:
   - `marketcore:get_community_blueprint_details` — confirm fit.
   - **State your plan**, then `marketcore:import_community_blueprint`. Done.
2. **If no community match**, branch on what the user has:
   - **Strong sample document** (paste-able markdown that captures the structure): `marketcore:create_blueprint` directly with `source_content`. Takes 1–3 min.
   - **Only a description**: `marketcore:create_blueprint_draft` (returns a reviewable draft) → review with the user → `marketcore:finalize_blueprint_draft` to publish. Both calls take 1–3 min.
3. Always pick a `category_id` from `marketcore:list_content_categories` (required for `create_blueprint`).
4. Surface the new `blueprint_uuid` to the user; they'll use it in future content generations.

---

## Recipe B — Browse and import community blueprints

**Goal.** Pull a community-published blueprint into the user's library.

**Steps.**
1. `marketcore:list_community_blueprints` — returns blueprints with names, summaries, contributor info, and `blueprint_exchange_id`.
2. For details on a specific one: `marketcore:get_community_blueprint_details(blueprint_exchange_id)`.
3. **State your plan** (this clones the blueprint into the user's library).
4. `marketcore:import_community_blueprint(blueprint_exchange_id)` — returns the new team-level `uuid` for use with `create_content`.

---

## Recipe C — Async generation followup

**Goal.** Check on (or report) a blueprint-driven generation that's still running.

**Steps.**
1. Recall the `generation_id` from earlier in the conversation. If lost, ask the user — `list_content` won't show pending generations.
2. `marketcore:get_generation_status(generation_id)`.
3. Status meanings:
   - `pending` / `gathering context` / `processing` → still working, surface progress.
   - `completed` → hand `response.content.link_url` to the user.
   - `failed` → tell the user; offer to retry.

---

## Recipe D — Share content externally

**Goal.** Generate a public share URL for a piece of content.

**Steps.**
1. `marketcore:create_external_share(content_id, expires_at?)` — `expires_at` is an optional Unix timestamp.
2. Hand the returned `share_link` to the user.

**Precondition.** Content must be `completed` — async generations need to finish first.

---

## Recipe E — Export content to Word

**Goal.** Produce a `.docx` download for a content item.

**Steps.**
1. `marketcore:get_content(content_id)` — fetch the markdown body. (This is one of the few legitimate uses of `get_content`.)
2. `marketcore:convert_markdown_to_word_doc(markdown_content, filename?, document_url?)` — `document_url` embeds as a footer link to the original.
3. Hand the returned `download_url` to the user.

---

## Recipe F — Update a project's name / visibility / status

**Goal.** Rename, archive, or change the visibility of a project.

**Steps.**
1. If you don't already know the `project_id`: `marketcore:list_projects`.
2. **State your plan** — name the field changing and the new value.
3. `marketcore:update_project(project_id, …)` with ONLY the fields changing (PATCH semantics — omitted fields are untouched).
   - `name` (text)
   - `visibility` enum: `team` | `private`
   - `status` enum: `active` | `archived`

**Note.** Setting `status="active"` requires available active-project usage on the team's plan — call returns `"You have reached your active project limit."` if exhausted.

---

## Recipe G — Workflow execution (advanced)

Workflows orchestrate multi-step content generations in sequence. Newer feature with fewer published patterns. Treat as advanced — most user requests are simpler with a single `create_content` call.

- `marketcore:list_workflows` — list defined workflows.
- `marketcore:get_workflow(workflow_id)` — inspect a workflow.
- `marketcore:create_workflow` / `marketcore:update_workflow` — author / modify.
- `marketcore:run_workflow` — execute.
- `marketcore:get_workflow_runs` — execution history.

Suggest a workflow only when the user describes a repeatable multi-step generation process they want to automate.

---

## Recipe H — Read the team's Brand Foundation directly

Most of the time you don't need to call `marketcore:get_core_context` — every `create_content` call pulls Brand Foundation in automatically (Layer 1 of the four-layer model).

Call it explicitly when:
- The user asks "what brand voice does the AI use?"
- You're generating content *outside* MarketCore (drafting a tweet in their voice elsewhere) and need the brand context.
- You're debugging "why does the content sound off?" — maybe Brand Foundation is stale.

Returns a single markdown string with all four sections concatenated.
