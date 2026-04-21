# Pitfalls and Known Limitations

Read this when something isn't working as expected, or proactively before any operation that touches the patterns described below.

---

## P1 — Brief content must already be in the project before you can set it as the brief

**Symptom:** `update_project(project_id, project_brief_id=<uuid>)` returns `"Content is not associated with this project. Add the content to the project first, then try again."`

**Cause:** The brief field on a project is an integer pointing to a `project_item` row that wraps a canvas or deliverable already in the project's documents. If the content (canvas/deliverable) isn't yet a project document, no wrapper exists, so the brief can't be set.

**What to do:**
1. **For new content**: create it inside the project first — `create_content(project_id=<project_id>, ...)`. The project_item wrapper is created automatically. Then call `update_project` with that content's UUID.
2. **For existing standalone content**: there's currently no MCP tool to attach an existing standalone canvas/deliverable to a project's documents. The user has to do this in the app (open the project → "+ Add Document" → pick the content). Once it's in the project, the `update_project` call works.

State the limitation upfront before the user is surprised.

---

## P2 — `get_unified_deliverable` returns a less-specific error for unknown UUIDs

**Symptom:** `update_project(... project_brief_id=<bad-uuid>)` returns `{"code": "ERROR_CODE_NOT_FOUND", "message": "Document not found"}`.

**Cause:** The brief-resolution path delegates to the `get_unified_deliverable` Xano function, which throws its own precondition error when neither the canvases nor deliverables table has a matching UUID. The error message is "Document not found" — less specific than mentioning canvases/deliverables explicitly, but unambiguous: the UUID doesn't exist.

**What to do:** Verify the UUID. Common mistakes: pasting a project_id where a content UUID was expected; pasting a project_item.id (integer) instead of a content UUID.

---

## P3 — Async generations require polling

**Symptom:** You call `create_content` with `blueprint_uuid`, get back a `generation_id`, and the user thinks nothing happened.

**Cause:** Blueprint-driven content is async. The initial response returns only `generation_id` — the actual content is generated in the background and you have to poll `get_generation_status` until status is `completed`, then call `get_content` with the returned `content_id`.

**What to do:**
- Always poll after an async `create_content`. Surface progress every ~minute.
- Status enum: `pending`, `gathering context`, `processing`, `completed`, `failed`.
- Typical duration: 3–5 minutes.
- When complete, the response includes a `content` summary with `content_id` — fetch the full result with `get_content`.

---

## P4 — `content` and `instructions` are mutually exclusive on `create_content`

**Symptom:** API rejects the call.

**Cause:** `create_content` has three modes:
- `content` only — save your text directly, no AI generation. Sync.
- `instructions` only — generate freeform from a prompt. Sync (1–3 min).
- `instructions` + `blueprint_uuid` — generate from a blueprint. Async (3–5 min).

`content` cannot be combined with `blueprint_uuid` (blueprints require `instructions`). `content` and `instructions` cannot both be passed.

**What to do:** Pick one path and explain it to the user as you announce the call.

---

## P5 — `list_blueprints` returns drafts and published items separately

**Symptom:** User says "use my X blueprint" but you can't find it in `blueprints`.

**Cause:** The response has both `blueprints` (published) and `blueprint_drafts` (in-progress) arrays. The user may not distinguish in conversation.

**What to do:** Check both arrays. If you find it as a draft, ask the user whether to finalize it (`finalize_blueprint_draft`) before using it for content generation.

---

## P6 — Targeting dimension *option* IDs vs dimension IDs

**Symptom:** Generation runs but ignores targeting, or returns an error about invalid IDs.

**Cause:** `list_targeting_dimensions` returns dimensions, each containing options. `create_content` takes `dimension_option_ids`, not dimension IDs.

**What to do:** Drill into the options array of each dimension and grab option-level IDs. Pass those.

---

## P7 — "Deliverable" terminology in API outputs

**Symptom:** `get_project` returns a `documents` array where each item has `deliverable_id`. `list_projects` returns `deliverable_count`. `get_current_user_info.usage` has `deliverables` and `deliverables_max`.

**Cause:** The in-app term migrated from "Deliverable" to "Content", but some API output fields still use the old name.

**What to do:** Translate when surfacing to the user. Don't expose `deliverable_id` to the user as "deliverable ID" — call it "content ID". When `usage.deliverables` shows usage, call it "content created this period".

---

## P8 — Synchronous `create_content` can still take 1–3 minutes

**Symptom:** Client times out waiting for `create_content` to return.

**Cause:** Even sync mode (`instructions` without `blueprint_uuid`) involves AI generation that can take several minutes. The MCP returns the full content object directly, but the wait is real.

**What to do:**
- Tell the user "this will take 1–3 minutes" before calling.
- Don't apply tight client-side timeouts.
- For blueprint mode (3–5 min), use the async path — that's why it exists.

---

## P9 — Stale tool names in older docs

**Symptom:** README or older Strapi pages reference `get_blueprints`, `get_projects`, `get_content_list`, etc.

**Cause:** Those tool names were renamed to the `list_*` convention. The old names are not currently exposed.

**What to do:** Use the current names: `list_blueprints`, `list_projects`, `list_content`, `list_content_categories`, `list_targeting_dimensions`, `list_context_collections`, `list_community_blueprints`, `list_workflows`. The `get_*` form is reserved for fetching a single item by ID.

---

## P10 — `project.system_prompt` is deprecated — use the project brief instead

**Symptom:** User asks you to "set the project's instructions" or "make the project always emphasize X".

**Cause:** The Project record has a legacy `system_prompt` field from an earlier data model. The **project brief** has taken over the role of persistent project-wide AI guidance. `system_prompt` is deprecated; `update_project` intentionally does NOT expose it.

**What to do:**
- Set a **project brief** instead: either at creation via `create_project(project_brief_details=...)` or on an existing project via `update_project(project_id, project_brief_id=<content_uuid>)`.
- If the user's intent is "custom instructions that apply to every generation in this project," write that into the brief document itself. Every generation in the project pulls the brief in as part of Project Context (Layer 3 of the four-layer model).

---

## P11 — `get_relevant_context` returns chunks, not full items

**Symptom:** You expect a complete document but get a few hundred words.

**Cause:** `get_relevant_context` returns RAG chunks (typically a few hundred words each), not full context items. It also returns `context_item_ids` so you can identify the parent items.

**What to do:** If you need the full item content, you can't currently retrieve it via MCP — you'll need the user to look it up in the app. For most "what context exists about X?" questions, the chunks themselves are sufficient.

---

## P12 — Pagination across `get_relevant_context` calls

**Symptom:** Repeating a `get_relevant_context` call returns the same chunks.

**Cause:** Without pagination, every call returns the top-N most relevant chunks, which is the same set.

**What to do:** Pass `context_rag_ids` (the IDs returned in the previous call) to exclude already-seen chunks. This effectively paginates.

---

## P13 — `add_context` to a project doesn't change project context budget

**Symptom:** User adds many large context items to a project and generation slows down or hits limits.

**Cause:** Project context items are *always* included verbatim during generation in that project (no relevancy filtering). If the project context grows large, every generation pulls all of it.

**What to do:** Periodically suggest pruning project context to only items still relevant to the initiative. Promote universally-useful items to the Reference Library (in-app operation) and remove from project context.

---

## P14 — Content `category_id` doesn't change generation behavior

**Symptom:** User expects setting a category to affect output style. It doesn't.

**Cause:** `category_id` is purely organizational — it helps the user find content later. Generation behavior is shaped by the blueprint (if any), instructions, context, and targeting dimensions.

**What to do:** Don't promise category-driven generation differences. If the user wants category-style output, recommend a blueprint instead.

---

## P15 — `create_external_share` requires a finalized content_id

**Symptom:** Share link creation fails on freshly generated content.

**Cause:** Async-generated content needs to reach `completed` status before sharing. Calling on a still-generating `generation_id` won't work — you need the `content_id` from the completed generation.

**What to do:** Always wait for `get_generation_status` to return `completed`, fetch via `get_content`, then share.

---

## P16 — `convert_markdown_to_word_doc` doesn't auto-fetch content

**Symptom:** User says "export my latest blog as Word", you call the conversion tool with no content.

**Cause:** This tool takes `markdown_content` as a parameter — it does not pull content by ID. You must first `get_content` to retrieve the markdown body, then pass it to the converter.

**What to do:** Two-step flow: `get_content` → `convert_markdown_to_word_doc(markdown_content=...)`. Optionally pass `document_url` (the original content's `link_url`) so the .docx footer links back.

---

## P17 — Don't surface raw deliverable IDs / generation IDs to the user as identifiers

**Symptom:** User sees a UUID like `7b2c4f...` in your message and is confused.

**What to do:** Surface the human-readable name + the `link_url` instead. The UUIDs are for tool calls, not user-facing references.

---

## P18 — `import_community_blueprint` requires the exchange ID, not the blueprint ID

**Symptom:** Import fails with an invalid ID error.

**Cause:** `list_community_blueprints` returns items with their own `blueprint_exchange_id` — distinct from a regular blueprint UUID. Use the exchange ID for `get_community_blueprint_details` and `import_community_blueprint`. The imported blueprint will then have a fresh team-level UUID you use with `create_content`.

---

## P19 — `update_project` empty-string inputs are silently no-op'd

**Symptom:** You pass `name=""` to `update_project` expecting a validation error or a clobber. Neither happens.

**Cause:** The MCP transport / Xano input filter coerces empty strings on optional nullable text fields to null. The tool's defensive precondition (`name must be non-empty`) is unreachable. Net effect: passing empty strings is a safe no-op rather than a data clobber.

**What to do:** This is the desired safety behavior — accept it. Don't try to force an empty-name update. If the user wants to clear a field, that operation isn't supported (intentionally).

---

## P20 — `update_project` is team-scoped — projects from other teams aren't visible

**Symptom:** `update_project` returns "Project not found in your current team." even though the user "knows" the project exists.

**Cause:** The tool calls `current_team_id` to resolve the user's active team, then scopes the project lookup to that team. If the project belongs to a different team the user is also a member of, they need to switch teams in the app first.

**What to do:** If a project lookup fails unexpectedly, ask the user to confirm their active team via `get_current_user_info`.

---

## P21 — Project document `purpose` field is `core_output` or `supporting`

When reading `get_project`'s `documents` array, each document has a `purpose`:
- `core_output` — the main work product of the project
- `supporting` — background material (the brief defaults to this)

Most agents don't need to set or change `purpose` — it's surfaced for the UI. If you do encounter it, treat it as informational. There's no MCP tool to change it currently.

---

## When you encounter something not in this list

If you hit an error or unexpected behavior not covered here:

1. Surface the raw error to the user — don't silently retry.
2. State what you tried and what the API returned.
3. Suggest plausible next steps (check parameters, retry, contact support at the GitHub repo issues page).
4. Don't assume a workaround that creates the wrong artifact in the wrong place — confirm with the user first.
