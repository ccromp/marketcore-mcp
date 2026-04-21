# Common Workflows — Step-by-Step Recipes

Use these as templates for the most common end-to-end requests. Adapt the specifics — the order of operations is the point.

---

## Recipe 1 — Generate a piece of content using a blueprint

**User says:** "Write a case study about our Acme deployment using my case study template, targeting healthcare CIOs."

```
1. list_blueprints                     → find the user's case study blueprint
                                          (also check blueprint_drafts in case it's still a draft)
2. list_targeting_dimensions           → find the "Healthcare CIO" persona option ID
3. list_projects                       → is this for an existing project? if yes, get project_id
4. (optional) get_relevant_context     → preview context for "Acme deployment"
                                          to surface anything missing
5. STATE PLAN to user — name the blueprint, the persona option, the project
6. create_content                      → with blueprint_uuid, instructions, project_id,
                                          dimension_option_ids
                                       → returns generation_id (async)
7. get_generation_status (poll ~30s)   → until status == "completed"
                                       → surface progress to user every minute
8. get_content (with content_id)       → fetch full result
9. Offer next steps: share link, Word export, refinement
```

**Don't:**
- Skip steps 1–3 and just guess parameters.
- Forget to poll. The async response gives you only `generation_id`.
- Call `get_current_user_info` proactively — every other tool call already runs as the authenticated user.

---

## Recipe 2 — Add a brand-new context item to the user's library

**User says:** "Add this competitive analysis as context."

```
1. Disambiguate scope first:
   - Top-level Reference Library?  (available across all projects)
   - Project-scoped?               (only for one initiative)
   - Project brief?                (sets a project's primary anchor — see Recipe 6)
   - Document-specific?            (only for one piece of content — don't store)
   ASK if unclear.

2. If top-level (Reference Library):
   a. list_context_collections      → does a fitting collection exist?
   b. (if no good fit, optionally) create_context_collection
   c. add_context                   → with name, content, and optional collection_id

3. If project-scoped (project's "Context Items" tab — supporting context):
   a. list_projects                 → confirm project_id
   b. add_context                   → with name, content, project_id

4. If brief: see Recipe 6.

5. Confirm with link_url and offer next step (e.g. "Want to use this in a content generation now?")
```

---

## Recipe 3 — Create a new blueprint from scratch

**User says:** "Make me a blueprint for product launch announcements."

```
1. list_community_blueprints         → is there a community template that fits?
   - If yes:
     a. get_community_blueprint_details
     b. STATE PLAN ("I'll import the 'Product Launch Announcement' blueprint by [author]")
     c. import_community_blueprint
     d. Done.

2. If no good community match, branch on what the user has:
   - User has a strong sample document → create_blueprint with source_content
   - User has only a description       → create_blueprint_draft, review with user, then finalize_blueprint_draft

3. list_content_categories           → pick a category_id (required for create_blueprint)

4. STATE PLAN before calling create_blueprint or finalize_blueprint_draft
   (both take 1–3 minutes)

5. Surface the new blueprint_uuid to the user — they'll use it next time
   they generate content.
```

---

## Recipe 4 — Create a new project

**User says:** "Set up a project for our Q3 product launch."

```
1. list_projects                     → confirm it doesn't already exist

2. ASK whether to seed a brief now:
   "Do you want to add a project brief now (auto-generated from a description)
    or set up the project empty? Either path is fine — you can always set
    or change the brief later via update_project."

3. STATE PLAN

4. create_project                    → with name, optional visibility, optional project_brief_details

5. Offer to add project context next:
   "Want to add some research / competitor materials to this project's context now?"
   → use add_context with project_id
```

---

## Recipe 5 — Generate freeform content (no blueprint)

**User says:** "Draft a quick LinkedIn post about today's product news."

```
1. list_blueprints                   → quick check — is there an obvious fit?
                                       If yes, propose using it instead.

2. list_projects                     → if the user mentions an initiative, scope to it

3. list_targeting_dimensions         → if the user mentions an audience, target it

4. STATE PLAN — clarify it'll be a synchronous generation taking 1–3 minutes

5. create_content                    → with instructions only (no blueprint_uuid),
                                       optional project_id, dimension_option_ids,
                                       collection_ids
                                       Optionally use_extended_thinking=true for complex content

6. Returns content object directly (no polling needed for sync mode)

7. Offer next steps
```

---

## Recipe 6 — Set or change the brief on an existing project

**User says:** "Set the brief on the Acme Launch project to that messaging doc I just wrote."

The brief is a Canvas or Deliverable that's already in the project's documents. The Project record's `project_brief_id` points at the wrapper. `update_project` resolves a content UUID to the wrapper internally — you just need to make sure the content is in the project first.

```
1. list_projects                     → find the project_id for "Acme Launch"
2. get_project (project_id)          → look at the documents array.
                                       Is the messaging doc already there?

3. If YES — content is in the project:
   a. STATE PLAN ("I'll set the brief on Acme Launch to '<doc title>' via
                   update_project with project_brief_id=<uuid>.")
   b. update_project(
        project_id=<uuid>,
        project_brief_id=<content_uuid>
      )
   c. Confirm with link_url to the project.

4. If NO — content isn't in the project yet:
   a. Two sub-options:
      A) Create the brief content fresh inside the project:
         - create_content(project_id=<project_id>, instructions=...)
           OR with content=<user's pre-written text>
         - then update_project(project_id, project_brief_id=<new content_uuid>)
      B) The user has standalone content they want to use:
         - State that there's no MCP tool to attach existing standalone content
           to a project's documents — they need to do this in the app first
           (open project → "+ Add Document" → pick the content).
         - Once added, follow path 3.

5. Verify by calling get_project again or directing the user to the project's URL.
```

**Note:** If `update_project` returns "Content is not associated with this project," go back to step 4 — the content isn't in the project's documents.

---

## Recipe 7 — Find existing context relevant to a topic

**User says:** "What context do I have about our enterprise pricing?"

```
1. get_relevant_context              → with prompt = "enterprise pricing"
                                       Optionally scope with project_id or collection_ids

2. Returns up to 10 chunks from the user's library + their parent context_item_ids

3. Surface a summary to the user. Don't dump raw chunks unless asked.

4. If results are sparse, offer to add new context.

5. If user wants more, paginate by passing context_rag_ids (already-returned IDs)
   on the next call to exclude them.
```

---

## Recipe 8 — Share or export finished content

**User says:** "Share that case study externally" or "export it as Word"

```
- For an external share link:
   create_external_share              → with content_id, optional expires_at (Unix ts)
                                        → returns share_link (public URL)

- For a Word export:
   get_content (content_id)           → fetch the markdown body
   convert_markdown_to_word_doc       → with markdown_content (the content body),
                                        optional filename, optional document_url
                                        → returns download_url (.docx file)

Surface the link/URL directly to the user.
```

---

## Recipe 9 — Async generation followup

**User says** (after an earlier blueprint generation): "Is my case study done yet?"

```
1. Recall the generation_id from the earlier turn (it's in your context).
   If not in context, list_content can show the latest content but won't have
   pending generations — ask the user for the generation_id.

2. get_generation_status              → check status

3. Status meanings:
   - pending / gathering context / processing → still working, surface to user
   - completed → fetch the content with get_content (use content_id from the response)
   - failed → tell the user, offer to retry

4. When complete, present the content and offer next steps.
```

---

## Recipe 10 — Browse and import from the community

**User says:** "Show me what blueprints other people have shared" or "import the case study one I saw"

```
1. list_community_blueprints         → returns array of available blueprints
                                       with names, summaries, contributor info

2. For details on a specific one:
   get_community_blueprint_details   → with blueprint_exchange_id
                                       → returns full content + style guide +
                                         input_instructions

3. STATE PLAN before importing — this clones it into the user's library

4. import_community_blueprint        → returns the new blueprint UUID
                                       → user can now use it in create_content
```

---

## Recipe 11 — Update a project's name, visibility, or status

**User says:** "Rename my project" / "Make this project private" / "Archive the Q3 launch project."

```
1. list_projects                     → find project_id by name if not given

2. STATE PLAN — name the field changing and the new value

3. update_project                    → with project_id and ONLY the fields changing
                                       (PATCH semantics; omitted fields untouched)
                                       - name (text)
                                       - visibility (enum: "team" | "private")
                                       - status (enum: "active" | "archived")

4. Confirm via the response (or get_project for verification)
```

**Note:** Setting `status="active"` requires the team to have available active-project usage; the call returns "You have reached your active project limit." if exhausted.

---

## General principle: announce → check → confirm → act

Every recipe above follows the same shape:

1. **Discover** what's already there (`list_*` calls).
2. **Disambiguate** with the user if multiple paths could fit.
3. **State** what you're about to do, with the tool name.
4. **Act** with the tool call.
5. **Verify and offer next steps** — link_url, share link, refinement, etc.

When in doubt, slow down at step 2. A wrong tool call is worse than a clarifying question.
