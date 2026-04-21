# Plan: "Write a case study about Acme Corp deployment, targeting healthcare CIOs"

## Reading the request

The user wants:
- A **case study** (a recognized content type — almost certainly a fit for a blueprint)
- Subject: **recent deployment with Acme Corp** (a real customer engagement — needs source material)
- Audience: **healthcare CIOs** (a targeting decision — likely maps to a Persona + Industry dimension option pair)

Three things I need before generating anything:
1. **A blueprint** — case studies are highly structured, so I should prefer a blueprint over freeform. Check if one already exists; if not, propose using a community one or creating one.
2. **A project home** — "recent deployment with Acme Corp" sounds like an initiative that may already have a project (e.g., "Acme Launch" or "Customer Stories Q2"). Check `list_projects`.
3. **Source material on Acme** — case studies live or die on the customer-specific facts (problem, solution, metrics, quotes). Brand Foundation + Reference Library won't include this; I need to ask Chris where the Acme deployment details live or have him add them as project context / document-specific context.
4. **Targeting option IDs** — "healthcare CIOs" needs to map to actual `dimension_option_ids` from `list_targeting_dimensions`.

I will NOT just call `create_content` and hope. The skill explicitly says "discover before you create" (§3.3) and "state your plan before you act" (§3.1).

---

## 1. First message back to the user (verbatim)

> Happy to help with the Acme Corp case study for healthcare CIOs. Before I generate anything, a few quick checks so it lands right:
>
> 1. **Project**: Should this live in an existing project (e.g. an "Acme Launch" or "Customer Stories" project) or do you want me to create a new one? I'll check what exists.
> 2. **Blueprint**: I'll look for an existing case study blueprint in your library. If you don't have one, I can either import one from the Blueprint Exchange or create one — your call.
> 3. **Source material on Acme**: Case studies need the specifics — what problem they had, what you deployed, results/metrics, any quotes. Do you have a deployment recap, win-wire, call transcript, or notes I should pull from? If it's already in your Reference Library or a project, I'll find it. If not, the best move is to paste/upload it before I generate.
> 4. **"Healthcare CIO" targeting**: I'll check your Targeting Dimensions for the matching Persona and Industry options so the content speaks to that audience specifically.
>
> Let me run the discovery calls now and come back with what I find — should take a few seconds.

(I do **not** wait for the user before doing read-only discovery — those calls are safe and give me concrete options to bring back. I do wait for the user before any create/generate call.)

---

## 2. Ordered tool calls

### Phase A — Confirm session and discover existing assets (read-only, run in parallel)

**A1. `mcp__marketcore-dev__get_current_user_info`**
- Params: none
- Rationale: Confirm auth, see the active team, surface plan/usage so I know if there are quota concerns before kicking off an async generation. Skill §1 mandates this once per session.

**A2. `mcp__marketcore-dev__list_projects`**
- Params: none (default listing)
- Rationale: Look for an Acme-related project ("Acme", "Acme Launch", "Customer Stories", etc.). Avoids creating a duplicate (skill §3.3).

**A3. `mcp__marketcore-dev__list_blueprints`**
- Params: none
- Rationale: Find an existing case study blueprint. Check both `blueprints` AND `blueprint_drafts` arrays per pitfall §6.5.

**A4. `mcp__marketcore-dev__list_targeting_dimensions`**
- Params: none
- Rationale: Find the option IDs for "Persona = Healthcare CIO" (or closest match — could be "CIO/IT Decision Maker" + "Industry = Healthcare"). Per pitfall §6.8, I need the *option* IDs, not dimension IDs.

**A5. `mcp__marketcore-dev__list_content_categories`**
- Params: none
- Rationale: Find the `category_id` for case studies / customer stories (likely "Customer Marketing", "Sales Enablement", or similar). Optional on `create_content` but improves organization.

**A6. `mcp__marketcore-dev__get_relevant_context`**
- Params: `prompt: "Acme Corp deployment case study — implementation details, problem, solution, results, metrics, customer quotes"`
- Rationale: Check whether Chris already has Acme material in his Reference Library or any project. If he does, I can scope the generation against it. If not, I know I need to ask him to add it.

### Phase B — Report findings and confirm decisions with user

I'd report back with something like:

> Here's what I found:
> - **Project**: [Found "Acme Launch" project / No Acme project exists]
> - **Blueprint**: [Found "Customer Success Story" blueprint — let's use it / No case study blueprint exists. Recommend either the "B2B SaaS Case Study" template from the Blueprint Exchange or I can create one from a sample you provide]
> - **Targeting**: I'll target Persona = "[matched option]" and Industry = "Healthcare" — does that match what you want?
> - **Acme source material**: [Found N relevant context items — listing them / Found nothing on Acme. Before I generate, can you paste a deployment recap, share a doc, or point me at a call transcript? Without specifics the case study will be generic.]
>
> My proposed plan:
> - Project: [name] (existing / new)
> - Blueprint: [name]
> - Targeting: [persona + industry option labels]
> - Source material: [list what I'll attach]
>
> Reply "go" and I'll kick off generation. (Async, ~3–5 minutes — I'll poll and come back when it's ready.)

### Phase C — Branching depending on user response

**Branch C1 — Source material exists in MarketCore already:**
Skip ahead to D.

**Branch C2 — Chris pastes/uploads Acme material:**

**C2a. `mcp__marketcore-dev__add_context`**
- Params:
  - `content_type: "manual"` (or `"file"` / `"call_transcript"` depending on what he sends)
  - `title: "Acme Corp Deployment — Source Material"`
  - `content: <pasted text>`
  - `project_id: <acme_project_id>` if scoping to a project, else omit for top-level Reference Library
- Rationale: Persist it once so it's reusable, and so the case study generation can reference it via project context (layer 3) or Reference Library RAG (layer 2).
- I'd disambiguate per skill §3.2: ask if this should be a project context item (recommended — it's specific to this initiative), Reference Library item (if Acme is a recurring story), or just one-off (skip storage). Default ask: project context item.

**Branch C3 — No case study blueprint exists, Chris approves importing one:**

**C3a. `mcp__marketcore-dev__list_community_blueprints`**
- Params: filter for case study / customer story templates
- Rationale: Per skill §4.3, check the Blueprint Exchange before creating from scratch.

**C3b. `mcp__marketcore-dev__get_community_blueprint_details`**
- Params: `community_blueprint_uuid: <chosen_uuid>`
- Rationale: Show Chris the structure + style guide before importing so he can confirm the fit.

**C3c. `mcp__marketcore-dev__import_community_blueprint`**
- Params: `community_blueprint_uuid: <chosen_uuid>`
- Rationale: Clones it into Chris's library so we can use it.

**Branch C4 — No Acme project exists, Chris wants a new one:**

**C4. `mcp__marketcore-dev__create_project`**
- Params:
  - `name: "Acme Corp Case Study"` (or whatever Chris confirms)
  - `visibility: "team"` (default; ask if he'd prefer private)
  - Probably **skip** `project_brief_details` for this — the case study itself is the deliverable, not a brief-worthy initiative. Keep it lean.
- Rationale: Per skill §4.4. State to user before calling.

### Phase D — Generate the case study

**D1. `mcp__marketcore-dev__create_content`**
- Params:
  - `blueprint_uuid: <case_study_blueprint_uuid>`
  - `project_id: <acme_project_id>` (if there's a project home)
  - `instructions: "Case study about MarketCore's deployment with Acme Corp. Audience: healthcare CIOs. Lead with the IT/operational problem Acme had, the deployment approach, measurable results (cite specific metrics from the source material), and a CIO-relevant takeaway. Use the Acme deployment source material attached as the factual backbone — do not invent metrics or quotes."`
  - `dimension_option_ids: [<healthcare_industry_option_id>, <cio_persona_option_id>]`
  - `category_id: <case_study_category_id>` (if found)
  - `collection_ids: [<acme_or_customer_stories_collection_id>]` if a relevant collection exists
- Rationale: Blueprint-driven async generation. Returns a `generation_id`. Per pitfall §6.6, I do NOT pass `content` (that's for pre-written text). Per pitfall §6.8, I'm passing option IDs not dimension IDs.
- **State plan to user immediately before calling**, per §3.1.

**Surface to user before calling:**

> Calling `create_content` now with the "Customer Success Story" blueprint, scoped to the Acme Corp Case Study project, targeting Healthcare + CIO. This is async — typically 3–5 minutes. I'll poll and ping you when it's done.

### Phase E — Poll for completion

**E1. `mcp__marketcore-dev__get_generation_status`**
- Params: `generation_id: <returned_from_D1>`
- Rationale: Per skill §3.5 and pitfall §6.4 — async generation is silent unless polled. Poll every ~30–60s. Surface progress to the user roughly every minute ("still generating, currently `processing`").

Loop until `status: completed` (or surface a clear error if it fails — `failed` / `error_*`).

### Phase F — Fetch and present the result

**F1. `mcp__marketcore-dev__get_content`**
- Params: `content_id: <returned_when_status_completed>`
- Rationale: Pull the full markdown/text body so I can show it to Chris.

**Surface to user:**

> Done. Here's the draft case study:
>
> [paste the rendered case study content — or summary + first 200 words if it's long, with offer to paste in full]
>
> A few next-step options:
> - **Refine** — tell me what to tweak (tone, length, missing angle) and I'll regenerate or do a freeform edit pass.
> - **Share** — I can create a public share link (`create_external_share`).
> - **Export** — I can convert it to a `.docx` (`convert_markdown_to_word_doc`).
> - **Save as project context** — if you'd like the final draft pinned as a reference for future Acme content.

---

## 3. What I would surface back at each meaningful step

| Step | What I tell the user |
|---|---|
| Before any tool call | "Running discovery — checking your projects, blueprints, targeting dimensions, and existing Acme context. Back in a few seconds." |
| After Phase A | Concrete summary: project found/not found, blueprint found/not found, targeting options matched, existing Acme context items (with names). Frame as a proposed plan, ask for "go" or corrections. |
| If no Acme source material exists | Explicitly flag this as the highest-risk gap: "Without specifics the case study will be generic / hallucinatory. Can you paste a deployment summary, share a doc URL, or point me at a call recording?" |
| Before importing a community blueprint | Show name + author + brief description, ask "import this one?" |
| Before creating a new project | "I'll create a project called 'Acme Corp Case Study', team-visible, no brief seeded. Want a different name or private visibility?" |
| Before `create_content` | Full plan restatement (blueprint, project, targeting, source material attached). "Kicking off — async, 3–5 minutes." |
| During polling | At ~1 min: "Still generating, status `processing`." At ~3 min if not done: "Still going, this is normal — the longer end of the 3–5 min window." |
| On completion | Paste the draft + offer the four next-step options. |
| On failure | Surface the error verbatim, propose a remediation (retry, switch blueprint, add more source material, fall back to freeform). |

---

## 4. Final action when content is created

After `get_content` returns the completed case study:

1. **Paste the full draft into the chat** (or a structured summary + first section if it's very long, with an offer to show the rest).
2. **Confirm where it lives**: project name, content ID, link URL (from the `link_url` field in the response).
3. **Offer the four follow-ups** above (refine / share / export / save as context).
4. **Do NOT auto-share or auto-export** — those create artifacts Chris hasn't asked for. Wait for his nod.
5. **If Chris asks for revisions**: prefer regenerating with updated `instructions` (or a tighter `dimension_option_ids` set) over freeform editing — the blueprint structure is the value. For small wording tweaks, do a freeform `create_content` with `instructions` describing the edits, scoped to the same project so context is preserved.

---

## Pitfall checklist I'm explicitly avoiding

- **§6.4** Won't forget to poll `get_generation_status` after the async `create_content`.
- **§6.5** Will check both `blueprints` and `blueprint_drafts` in `list_blueprints` output.
- **§6.6** Won't pass both `content` and `instructions`, and won't pass `content` with `blueprint_uuid`.
- **§6.8** Will pass *option* IDs (not dimension IDs) in `dimension_option_ids`.
- **§3.2** Will explicitly disambiguate where Acme source material should live (project context vs Reference Library vs document-specific) rather than guessing.
- **§3.3** Discovery before creation — `list_projects` / `list_blueprints` / `get_relevant_context` all run before any create call.
- **§3.1** State the full plan before `create_content` fires — single biggest place to land in the wrong project / with the wrong blueprint.
- Will not create a project brief for this work — the case study is the output, not a brief-worthy initiative anchor (avoids §6.1 trap of needing the brief content to be in the project first).
