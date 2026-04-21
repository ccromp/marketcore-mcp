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
4. STATE PLAN to user — name the blueprint, the persona option, the project
5. create_content                      → with blueprint_uuid, instructions, project_id,
                                          dimension_option_ids
                                       → create_content pulls all relevant context internally
                                          (Brand Foundation, Reference Library, Project Context)
                                       → returns generation_id (async)
6. get_generation_status (poll ~30s)   → until status == "completed"
                                       → surface progress to user every minute
7. Hand response.content.link_url to the user — they review the content in MarketCore.
   Offer next steps: share link, refinement.
```

**Don't:**
- Skip steps 1–3 and just guess parameters.
- Forget to poll. The async response gives you only `generation_id`.
- Call `get_current_user_info` proactively — every other tool call already runs as the authenticated user.
- Call `get_relevant_context` before `create_content` — `create_content` already pulls context internally.
- Call `get_content` after generation completes — hand the user the link, don't fetch the body. Only call `get_content` if the user later asks a question that requires reading the content body.

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

```
1. list_projects                     → find the project_id for "Acme Launch"
                                       (and list_content if needed to identify the messaging doc)
2. STATE PLAN ("I'll set the brief on Acme Launch to '<doc title>'.")
3. update_project(
     project_id=<project_uuid>,
     project_brief_id=<content_uuid>
   )
4. Hand the project link to the user.
```

`update_project` handles both cases automatically:
- Content is already in the project's documents → uses the existing wrapper.
- Content isn't in the project yet → attaches it AND sets it as the brief in one call.

**Don't:**
- Call `get_content` to "fetch the markdown" then `create_content(project_id=...)` to "duplicate" the content into the project. That creates a second standalone copy with a new UUID and is wrong. Attachment is a relationship, not a copy. `update_project` does the attachment for you.
- Call `get_project` first just to check if the doc is in the project — `update_project` handles both states.

---

## Recipe 7 — Q&A or ideation: find existing context on a topic

**User says:** "What context do I have about our enterprise pricing?" / "Have we written anything yet about competitor X?" / "I'm thinking about a campaign — what could I draw on?"

This is when the user wants YOU to look at their library and answer them. NOT when they want content generated (for that, use `create_content` directly — it pulls context internally).

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
   get_content (content_id)           → fetch the markdown body (this is one of the
                                        few legitimate uses for get_content — you need
                                        the body bytes to convert)
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
   - completed → hand response.content.link_url to the user (don't call get_content)
   - failed → tell the user, offer to retry

4. When complete, present the link and offer next steps (share, refine, export).
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
