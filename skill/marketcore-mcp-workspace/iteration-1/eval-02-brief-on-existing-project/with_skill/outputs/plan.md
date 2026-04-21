# Plan — Set the brief on the "Q4 Product Launch" project to the messaging document the user just made

## Interpreting the request

The user wants to pin a specific Content item (a messaging document they just created) as the strategic anchor (project brief) of an existing project named "Q4 Product Launch". This is a textbook **Workflow 2 — Set or change a project brief** from `SKILL.md`.

Two unknowns to resolve before I can call `marketcore:update_project`:
1. The `project_id` for "Q4 Product Launch" (I have a name, not a UUID).
2. The Content UUID for "that messaging document I just made" (the user is referring to it deictically — I don't have its UUID in awareness from earlier turns).

I do NOT need to do any of the following (per the skill's pitfalls section), and I will explicitly avoid them:
- I will NOT call `marketcore:get_content` on the messaging doc and re-create it via `marketcore:create_content(project_id=...)` to "put it in the project." That would create an orphaned duplicate with a fresh UUID. **Attachment is a relationship, not a copy.**
- I will NOT call `marketcore:get_project` first to check whether the doc is already attached. `marketcore:update_project(project_brief_id=...)` handles BOTH cases automatically — if the content isn't already in the project's documents, the tool attaches it AND sets it as the brief in one call.
- I will NOT use `marketcore:add_context` — that creates a Context item, which is a different object from a Content item used as a brief.

The question of "is the messaging doc already in the project or not?" is therefore **not a branch in my logic**. The tool transparently handles both states.

---

## What I'd say back to the user (plan announcement, before any tool calls)

> Got it — I'll pin your messaging doc as the brief on the "Q4 Product Launch" project. Quick disambiguation: I want to make sure I grab the right "messaging doc you just made" — I'll check your most recent content and confirm the title with you before I pin it. One sec.

After resolving the project + content IDs, before the actual `update_project` call:

> I'll set the brief on "Q4 Product Launch" to "[messaging doc title]". One call to `update_project` does this — and if the doc isn't already attached to the project, the same call attaches it. No duplicate is created; the original messaging doc keeps its UUID and Reference Library / wherever-it-lives placement.

---

## Exact MCP tool call sequence

### Step 1 — Resolve the project UUID

**Call:** `marketcore:list_projects`

**Parameters:** none.

**Why:** I have the project name ("Q4 Product Launch") but not its UUID. `update_project` requires the UUID. Per Workflow 2 step 1.

**Expected handling:**
- Match the project whose name is exactly (or unambiguously close to) "Q4 Product Launch" and grab its UUID.
- If multiple projects match, surface them and ask the user to pick.
- If none match, surface that and ask whether they meant a different name or want to create the project first (Workflow 4). Do NOT auto-create.

### Step 2 — Resolve the messaging document's Content UUID

**Call:** `marketcore:list_content`

**Parameters:** none (rely on default ordering / recency in the response).

**Why:** The user said "that messaging document I just made." I don't have the UUID in awareness. Per Workflow 2 step 2, I need to find the most recent matching Content item and confirm with the user.

**Disambiguation strategy:**
- Look at the most recently created Content items in the response.
- Surface candidates whose title contains "messaging" / "positioning" / similar product-marketing terms.
- If exactly one obvious recent match exists, name it and ask the user to confirm: "Looks like you mean '[title]', created [time ago] — confirm and I'll proceed?"
- If multiple plausible candidates exist, list the top 2-3 by recency and ask the user to pick one.
- If nothing obvious matches, ask the user for the title or for the doc's MarketCore link.

I will NOT silently guess — the skill is explicit about confirming when ambiguous. I will NOT call `marketcore:get_content` to read each candidate's body to "figure out which is the messaging doc"; the title + recency is enough for disambiguation, and `get_content` is reserved for cases where I need the body itself.

### Step 3 — Set the brief (the one mutation)

**Call:** `marketcore:update_project`

**Parameters:**
- `project_id`: `<UUID from step 1>`
- `project_brief_id`: `<Content UUID from step 2>`

**Why:** This is the one and only mutation. Per Workflow 2 step 4 and the choosing-between-tools table:
> "To pin a Content item as a project's strategic anchor → `marketcore:update_project(project_brief_id=...)`"

And per the skill's explicit guidance on the tool's auto-attach behavior:
> "**The tool handles BOTH cases automatically:** if the content is already in the project's documents → uses the existing wrapper; if not → attaches it AND sets it as the brief in one call."

So this single call covers both possible states. PATCH semantics: only `project_id` and `project_brief_id` are passed; `name`, `visibility`, `status` are untouched.

---

## Handling the case where the messaging doc may or may not already be in the project

This is the heart of the question, and the skill's answer is unambiguous: **don't branch on it; let `update_project` handle both states in one call.**

- **If the messaging doc is already attached to "Q4 Product Launch":** `update_project` finds the existing `project_item` wrapper and pins the brief by setting `project.project_brief_id`.
- **If the messaging doc lives in the Reference Library, in another project, or unattached:** `update_project` attaches it to "Q4 Product Launch" AND sets it as the brief in the same call. The original Content item is not duplicated — its UUID is preserved; only a new `project_item` relationship row is added.

I will NOT call `get_project` to inspect the project's documents first. Per the pitfall:
> "Don't call `get_project` first to check if the doc is in the project. `update_project` handles both states."

I will also NOT re-create the messaging doc via `create_content(project_id=...)` to "put it in the project." Per the founding-misfire pitfall:
> "Don't fetch the content with `get_content` and re-create it via `create_content(project_id=...)` to 'put it in the project.' That creates a duplicate with a fresh UUID, orphaned from the original. Attachment is a relationship, not a copy. `update_project` handles it."

If the messaging doc happens to be marked private and `update_project` attaches it as the brief, the underlying PATCH endpoint will auto-set `project_item.private_override = true` so other project members can read it (per pitfall E7). I don't manage that flag manually.

---

## Error handling I'm prepared for

- **`"Document not found"`** on the `update_project` call → likely a wrong UUID for `project_brief_id`. Most common cause is mistakenly using a `project_id` where a content UUID was expected, or a `project_item.id` integer instead of a UUID (per pitfall E1). I'll surface the raw error verbatim, restate which UUID I used for `project_brief_id`, and ask the user to verify by sharing the doc's link.
- **`"Project not found in your current team."`** → "Q4 Product Launch" belongs to a different team the user is also a member of but isn't currently active in. I'll tell the user to switch active teams in the MarketCore app. I will not try to recover by switching teams programmatically.
- **Auth / `unauthorized`** → I'll tell the user to reconnect MarketCore in their MCP client integration settings. No silent retry.
- **Anything else** → Surface the raw error verbatim, name the tool I called and the parameters, suggest plausible next steps, and ask the user before retrying. Per pitfalls.md: don't assume a workaround that creates the wrong artifact in the wrong place — confirm first.

---

## What I'd surface to the user at the end (success path)

Per Workflow 2's "Output to user":
> "Brief set: [project link]."

Concretely, after `update_project` returns `success: true` with the updated project record, I'd say something like:

> Done — "[messaging doc title]" is now the brief on "Q4 Product Launch": [project link]. The doc itself is unchanged; it's just been pinned as the project's strategic anchor (and attached to the project, if it wasn't already). Every future generation inside this project will automatically draw on it as Project Context. Want me to add supporting research / competitor materials to the project, or generate something against the new brief?

Surface conventions I'll follow:
- Use the project's human-readable name and `link_url` — never raw UUIDs (pitfall E10).
- Don't expose internal field names like `deliverable_id`, `project_item.id`, or the API-level "deliverables" terminology (pitfall E9). Call them "content" / "brief" / "project."
- Don't call `get_content` to fetch and display the brief's body — the user can open the link if they want to read it. The `get_content` tool is reserved for the two narrow cases the skill calls out (answering a content question that requires the body, or feeding markdown into `convert_markdown_to_word_doc`).
