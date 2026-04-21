# Plan — Case study about Acme Corp deployment, targeting healthcare CIOs

## 1. Reading the request

The user said: *"I want to write a case study about our recent deployment with Acme Corp. Targeting healthcare CIOs. Can you do that for me?"*

This is **Workflow 1 — Generate content** with three concrete signals I can act on:

- **Content type:** "case study" — a canonical content type that very likely has a corresponding blueprint in the user's library. Blueprint-driven generation is async (3–5 min, returns `generation_id` to poll).
- **Subject:** "recent deployment with Acme Corp" — a *specific named customer engagement*. This is exactly the narrow exception in Workflow 1 where one targeted `marketcore:get_relevant_context` sourcing-check up front is reasonable, so I can ask "do you have Acme deployment notes loaded, or should we add some first?" Acme-specific facts are unlikely to be in the standard Reference Library.
- **Audience:** "healthcare CIOs" — maps to targeting dimension *options* (likely Persona = CIO and/or Industry = Healthcare). Skill is explicit: pass *option* IDs, not dimension IDs.

What the user did NOT specify:
- Whether this should live in a project (an Acme account project, a customer-marketing project, a launch).
- Which case-study blueprint (if multiple exist) or whether they want freeform.
- Whether they have Acme source material loaded.

I'll ask the small disambiguations in my opening message and run the safe pure-read discovery calls in parallel while waiting for their answer.

---

## 2. What I'd say back to the user (opening message)

> Happy to draft that. Before I kick off generation, a couple quick checks:
>
> 1. I'll look for an existing case-study blueprint in your library and a Healthcare-CIO targeting option (or the closest equivalent — likely Persona = CIO + Industry = Healthcare). I'll use whatever exists.
> 2. Should this case study live in a project (an Acme account, a customer-marketing project, a launch) or stand on its own?
> 3. Do you have source material on the Acme deployment already loaded in MarketCore — call notes, deployment recap, customer quotes? I'll do a quick check, but if it's not there yet, adding some context first will produce a much sharper draft (Acme-specific facts probably aren't in your standard Reference Library).
>
> Once you confirm, I'll generate via the case-study blueprint targeted at Healthcare CIOs. Blueprint generation is async (typically 3–5 minutes) — I'll poll and surface the link as soon as it's ready.

I'd send that and proceed in parallel with the read-only discovery calls (they don't need user confirmation).

---

## 3. Tool call sequence

### Phase A — Discovery (run in parallel, before any generation)

1. **`marketcore:list_blueprints`**
   - **Why:** Find the case-study blueprint. Skill explicitly says check both the `blueprints` and `blueprint_drafts` arrays — the user may have a draft they're iterating on.
   - **Params:** none.
   - **Decision rule:**
     - One match → propose it.
     - Multiple matches (e.g. "Customer Case Study", "Technical Case Study", "Healthcare Case Study") → ask user to pick.
     - No match → fall back to freeform `create_content` (sync, 1–3 min) and offer Recipe A (create a reusable case-study blueprint) afterward, or check Blueprint Exchange via `list_community_blueprints`.

2. **`marketcore:list_targeting_dimensions`**
   - **Why:** Resolve "healthcare CIOs" to dimension *option* IDs. I expect a Persona dimension with a CIO option and an Industry dimension with a Healthcare option (the actual structure may differ — could be one combined dimension).
   - **Params:** none.
   - **Decision rule:** Drill into the `options` array on each dimension and grab the option UUIDs that match. If only one of the two attributes has a matching option, use what's available and fold the rest into the `instructions` text as audience framing.

3. **`marketcore:list_projects`**
   - **Why:** See whether there's an obvious Acme-related or customer-marketing project home before proposing one. Don't assume — surface candidates.
   - **Params:** none.

(I will NOT call `marketcore:get_current_user_info` to "verify auth" — auth is implicitly confirmed by any successful tool call.)

### Phase B — Sourcing check (the narrow Workflow-1 exception)

4. **`marketcore:get_relevant_context`**
   - **Why:** "Acme Corp" is a specific named customer — exactly the narrow case the skill carves out. Single targeted call so I can ask the user about source material if it's missing. **This is NOT a pre-load for `create_content`** (which pulls relevant context internally) — it's a one-off check to drive the disambiguation question.
   - **Params:**
     - `prompt`: `"Acme Corp deployment, customer outcomes, results, healthcare implementation"`
     - No `project_id` / `collection_ids` — I want a broad sweep across the whole library at this stage.
   - **Decision rule:**
     - Hits referencing Acme → tell the user "I see source material on the Acme deployment — I'll let the model draw on it."
     - No Acme-specific hits → tell the user "I don't see source material on the Acme deployment yet — want to paste in deployment notes / win-wire / interview transcript before I generate? It'll be much sharper." Branch to Phase C if yes; otherwise generate with a flagged caveat in the prompt that deployment specifics may need editing.

### Phase C — (Conditional) Add Acme source material as context

*Only run if the user provides notes after the Phase B prompt.*

5. **`marketcore:add_context`** (Workflow 3)
   - **Why:** Persist the Acme deployment notes so this generation (and future Acme work) can draw on them.
   - **Scope decision:**
     - Inside a project → `project_id` (project-scoped, always pulled in for project generations — Layer 3, included verbatim, no relevancy filtering per pitfall E4).
     - Standalone → top-level Reference Library (no `project_id`) so it's reusable and relevancy-filtered.
   - **Params:**
     - `name`: e.g. `"Acme Corp deployment notes"`
     - `content`: the user-pasted text.
     - `project_id`: only if scoped.

### Phase D — State the plan, then generate

6. **State plan in one sentence to the user** (Workflow 1 step 4):
   > "Generating a case study using the **[Blueprint Name]** blueprint, [scoped to project X | standalone], targeted at Persona = CIO + Industry = Healthcare. Blueprint generation is async (3–5 min) — I'll poll and ping you when it's ready."
   >
   > (If no blueprint exists: "Generating a case study freeform — sync, 1–3 min.")

7. **`marketcore:create_content`**
   - **Why:** The actual generation. Per the skill: do NOT pre-fetch with `get_relevant_context` to pass in. `create_content` already pulls Brand Foundation (Layer 1) + Reference Library via relevancy (Layer 2) + Project Context if scoped (Layer 3) + any explicit `collection_ids` (Layer 4) internally.
   - **Params:**
     - `instructions`: A focused prompt — something like *"Case study on our recent deployment with Acme Corp, framed for healthcare CIOs evaluating similar implementations. Lead with the business outcome and CIO-level decision criteria — security/compliance posture, integration risk, time-to-value, measurable results. Structure: customer snapshot → challenge → solution → results → quote → next steps. Keep technical depth at the level a healthcare CIO needs, not implementation detail."* (Skill reminder: don't restate brand voice — Layer 1 has it. Don't restate project strategy — Layer 3 has it if scoped.)
     - `blueprint_uuid`: from `list_blueprints` (if found).
     - `project_id`: if scoped.
     - `dimension_option_ids`: `[<CIO option UUID>, <Healthcare option UUID>]` — *option* IDs from the `options` array, not dimension IDs.
     - `category_id`: optional; skip unless an obvious "Case Study" / "Customer Marketing" category jumps out (organizational only — doesn't affect generation per pitfall E5).
   - **Validation before calling:** `content` is NOT being passed (mutually exclusive with `instructions` + `blueprint_uuid`). `instructions` + `blueprint_uuid` is the legal blueprint-driven combo.
   - **Returns:** `{ generation_id }` — no content yet (async).

### Phase E — Async polling

8. **`marketcore:get_generation_status`** — poll loop
   - **Why:** Blueprint generation is silent unless polled. Without polling, the user thinks nothing happened.
   - **Params:** `generation_id` from step 7.
   - **Cadence:** poll every ~30 seconds; surface progress to the user every ~minute (not every poll — would be noisy).
   - **Status enum:** `pending` → `gathering context` → `processing` → `completed` (or `failed`).
   - **User-facing updates:** "Still generating — currently `processing`. Should be done in another minute or two."
   - **Stop conditions:**
     - `completed` → response contains `content.link_url`. Proceed to Phase F.
     - `failed` → surface the error to the user verbatim, offer to retry (same params) or adjust (different blueprint, different instructions).

### Phase F — Hand off

9. **Surface to user:**
   > "Your case study is ready: **[Title]** — [link_url].
   >
   > Open it in MarketCore to review. From here I can:
   > - Generate a public share link for sending externally
   > - Export it as a Word doc
   > - Refine specific sections — just tell me what you want to change
   > - Pin it as the brief on a project
   >
   > Want any of those?"

   I will NOT call `marketcore:get_content` to fetch and quote the body. The user reads it in MarketCore via `link_url`. (The only legitimate post-gen `get_content` uses are: user later asks a body-dependent question, or feeding `marketcore:convert_markdown_to_word_doc` for Word export.)

---

## 4. Why each call (one-liner recap)

| # | Tool | One-line reason |
|---|---|---|
| 1 | `marketcore:list_blueprints` | Find the case-study blueprint (check both `blueprints` and `blueprint_drafts`). |
| 2 | `marketcore:list_targeting_dimensions` | Get the *option* ID(s) for healthcare CIO targeting. |
| 3 | `marketcore:list_projects` | See whether Acme has an obvious project home before proposing one. |
| 4 | `marketcore:get_relevant_context` | Narrow Workflow-1 sourcing-check — "Acme Corp" is a specific customer name unlikely to be in standard Reference Library. One call only. |
| 5 | `marketcore:add_context` *(conditional)* | Persist Acme deployment notes if user provides them. |
| 6 | `marketcore:create_content` | The actual generation — blueprint-driven, async. |
| 7 | `marketcore:get_generation_status` | Poll until `completed` because blueprint mode is async. |

---

## 5. Async waiting — how I'll handle it

- Tell the user up front that blueprint generation is async (3–5 min) so they can context-switch.
- Poll every ~30 seconds via `get_generation_status`.
- Surface progress to the user every ~minute (not every poll).
- Don't block silently. If `processing` lingers, say so.
- On `failed`: surface the error verbatim, offer retry or adjust.
- Never apply a tight client-side timeout — the wait is real and expected.
- Do NOT call `get_content` during the wait or after completion (hand the link, not the body).

---

## 6. What I will NOT do (anti-patterns explicitly avoided)

- **Will NOT call `marketcore:get_current_user_info`** to "verify auth" or "check quota." Auth is implicitly confirmed by any successful tool call.
- **Will NOT call `marketcore:get_core_context`** before generating. Brand Foundation auto-injects as Layer 1.
- **Will NOT pre-fetch broadly with `marketcore:get_relevant_context` before `create_content`.** `create_content` pulls all relevant context internally. The single Phase B call is the narrow Workflow-1 exception (specific customer name) — purely a user-facing sourcing check, not a generation prelude.
- **Will NOT call `marketcore:get_content` after generation completes** to display the body. Hand the `link_url`. (Only legitimate post-gen uses: user asks a body-dependent question, or Word export via `convert_markdown_to_word_doc`.)
- **Will NOT pass both `content` and `instructions`**, or `content` with `blueprint_uuid` (mutually exclusive).
- **Will NOT pass dimension IDs where option IDs are required** — drill into the `options` array.
- **Will NOT skip the announce-plan-before-act step.** I confirm blueprint, targeting, and project before calling `create_content`.
- **Will NOT silently assume a project.** If the user doesn't say, I generate standalone — and I ask first.
- **Will NOT duplicate content** to "put it in a project" later. If the user later wants this case study attached to a project (or set as a project brief), that's `marketcore:update_project(project_id, project_brief_id=<content_uuid>)` — which auto-attaches if needed. Never `get_content` + `create_content(project_id=...)`.
- **Will NOT expose UUIDs to the user** as identifiers (per E10). I surface the title + `link_url`.

---

## 7. What gets surfaced at the end

1. **The `link_url`** to the new case study in MarketCore (primary deliverable).
2. **The title** of the generated content (so the user can find it later in their library).
3. **Four follow-up offers:**
   - Public share link via `marketcore:create_external_share` (Recipe D).
   - Word export via `marketcore:get_content` + `marketcore:convert_markdown_to_word_doc` (Recipe E — the one legitimate `get_content` use case here, with the original `link_url` passed as `document_url` so the .docx footer links back).
   - Refinement (re-generate with adjusted instructions, or a different blueprint).
   - Pin as a project brief via `marketcore:update_project(project_brief_id=<this content's UUID>)` (Workflow 2) — though noting briefs are usually strategy anchors, not deliverables, so this is rarely the right move for a case study.

---

## 8. Error handling I'm prepared for

- **Auth error** → tell user to reconnect MarketCore in their MCP client's integration settings; don't retry.
- **Generation `failed`** → surface error verbatim, offer retry.
- **`list_targeting_dimensions` lacks Healthcare or CIO option** → fall back to folding audience into `instructions` text and tell the user we used a softer targeting approach.
- **No case-study blueprint exists** → check Blueprint Exchange (`list_community_blueprints`); if nothing fits, freeform sync generation (1–3 min), then offer Recipe A (create a reusable case-study blueprint) afterward.
- **Quota / rate-limit error** → surface message text; do not retry in a tight loop.
- **`"Project not found in your current team"`** (if the user names a project in another team) → ask them to switch active teams in MarketCore.
