# Plan — "Set the brief on the 'Q4 Product Launch' project to that messaging document I just made"

## Read of the request

The user wants to set an **existing piece of content** ("that messaging document I just made") as the **project brief** on an **existing project** named "Q4 Product Launch". This is exactly Recipe 6 in `references/workflows.md`. The phrase "I just made" strongly implies the messaging doc already exists as a Content (canvas or deliverable) in their MarketCore account — but it does NOT tell me whether it's already attached to the Q4 Product Launch project. That's the precondition I have to verify, because `update_project(project_brief_id=...)` requires the content to already be a document in the project (skill §6.1).

I also have one ambiguity I should call out but not block on: "that messaging document I just made" doesn't give me a title. I can resolve this with `list_content` (sorted recent-first, or filter by recency) and propose a candidate to confirm, rather than asking up front and adding a turn. If `list_content` makes it obvious (e.g. one item titled "Q4 Launch Messaging" created an hour ago), I'll propose that and ask for one-word confirmation. If multiple plausible candidates exist, I'll list the top 2-3 and ask the user to pick.

---

## 1. First message back to user (verbatim)

> Got it — I'll set your messaging doc as the project brief on **Q4 Product Launch**. Quick plan:
>
> 1. Look up the **Q4 Product Launch** project to get its ID and see what's already in its documents.
> 2. Find the messaging document you just made (I'll check your recent content and confirm the right one with you before doing anything).
> 3. If the messaging doc is already a document inside the Q4 Product Launch project, I'll set it as the brief in one call (`update_project` with `project_brief_id`).
> 4. If it's NOT in the project yet, I'll flag that — there are two paths and I'll let you pick before proceeding.
>
> Starting with the lookup now.

---

## 2. Ordered tool calls

### Step 1 — Confirm auth (cheap, once per session)

- **Tool:** `mcp__marketcore-dev__get_current_user_info`
- **Params:** none
- **Rationale:** Skill §1 says to call this once at the start of a session to confirm auth and active team. If this is the first turn of the session, do it; otherwise skip.
- **Surface to user:** Nothing unless it fails (then point them at `references/connection.md`).

### Step 2 — Find the project

- **Tool:** `mcp__marketcore-dev__list_projects`
- **Params:** none (or pagination params if the team has many projects)
- **Rationale:** I need the `project_id` (UUID) for "Q4 Product Launch". Skill §3.3 — discover before you act.
- **Surface to user:** If exactly one project matches, just continue. If zero match, ask: *"I don't see a project called 'Q4 Product Launch' — do you want me to create it, or is it under a slightly different name?"* If multiple match (unlikely but possible — e.g. "Q4 Product Launch — Hardware" and "Q4 Product Launch — Software"), list them and ask which.

### Step 3 — Inspect the project's current documents

- **Tool:** `mcp__marketcore-dev__get_project`
- **Params:** `project_id=<uuid from step 2>`
- **Rationale:** Recipe 6 step 2 — I need to look at the `documents` array to see whether the messaging doc is already in the project. This also tells me whether a brief is already set (so I can mention I'll be replacing it if so).
- **Surface to user:** Nothing yet — I'll combine the result with step 4 before saying anything.

### Step 4 — Find the messaging document

- **Tool:** `mcp__marketcore-dev__list_content`
- **Params:** none initially (rely on default recency sort); paginate if needed
- **Rationale:** "That messaging document I just made" implies recent. I want to identify a candidate before asking the user, so my clarifying question is concrete ("Is it 'Q4 Launch Messaging Framework'?") instead of open-ended ("which doc?"). Skill §3.2 — ask when ambiguous, but make the ask cheap.
- **Inline clarifying question** (only if needed):
  - **If one obvious recent candidate:** *"Looks like 'Q4 Launch Messaging Framework' (created 2 hours ago) is the one — confirm and I'll set it as the brief?"*
  - **If multiple plausible candidates:** *"I see a couple recent docs that could be it — (a) 'Q4 Launch Messaging Framework' (2h ago), (b) 'Q4 Positioning v3' (yesterday). Which one?"*
  - **If no plausible candidate:** *"I don't see a recent messaging-style doc in your library — can you give me the title, or did you draft it somewhere outside MarketCore?"*

### Step 5 — Cross-reference: is the messaging doc already in the project?

- **Tool:** none — this is a comparison between the `documents` array from step 3 and the content UUID from step 4.
- **Rationale:** Recipe 6 step 2 — this comparison is the fork in the road.
- **Two branches follow:**

#### Branch A — Doc IS in the project's `documents` array

##### Step 6A — State plan and set the brief

- **What I tell user first** (verbatim):
  > "'Q4 Launch Messaging Framework' is already in the project's documents. I'll set it as the brief now via `update_project` with `project_brief_id=<uuid>`. (This replaces the current brief — let me know if you want me to hold off.)"
  >
  > *(If no current brief is set, drop the parenthetical.)*
- **Tool:** `mcp__marketcore-dev__update_project`
- **Params:**
  - `project_id=<project uuid>`
  - `project_brief_id=<content uuid of messaging doc>`
- **Rationale:** Recipe 6 step 3, skill §2.1 (Project brief) and §3.2 (the "Set the brief on..." pattern). PATCH semantics — only the brief field changes. The MCP tool resolves the content UUID to the right `project_item` wrapper internally.
- **Pitfall guard:** If the API returns `"Content is not associated with this project"`, fall through to Branch B.

##### Step 7A — Verify

- **Tool:** `mcp__marketcore-dev__get_project`
- **Params:** `project_id=<uuid>`
- **Rationale:** Cheap confirmation that `project_brief_id` is now the new content's wrapper. Skill §3.3 (discover before you create — applied here as verify after you mutate).

#### Branch B — Doc is NOT in the project's `documents` array

##### Step 6B — Surface the gap and offer the two paths

- **What I tell user** (verbatim):
  > "I found 'Q4 Launch Messaging Framework' in your library, but it isn't currently a document inside the Q4 Product Launch project — and the brief has to live in the project before I can set it. Two ways forward:
  >
  > **(1) Easiest in MCP — create a fresh copy of the messaging doc inside the project**, then set that copy as the brief. Downside: you'll have two near-identical docs, and the standalone original won't auto-sync with the brief. I can do this end-to-end right now.
  >
  > **(2) Move the existing doc into the project first** (this isn't currently exposed via MCP — you'd do it in the app: open the project → '+ Add Document' → pick 'Q4 Launch Messaging Framework'). Once it's in the project, tell me and I'll set it as the brief in one call.
  >
  > Which do you prefer? If (1), do you want me to copy the doc as-is, or regenerate it with project context layered in?"

##### Step 7B — Execute the user's choice

- **If user picks (1) "copy as-is":**
  - First fetch the source content body:
    - **Tool:** `mcp__marketcore-dev__get_content`
    - **Params:** `content_id=<existing messaging doc uuid>`
    - **Rationale:** I need the markdown body to pass through.
  - Then create a project-scoped copy:
    - **Tool:** `mcp__marketcore-dev__create_content`
    - **Params:**
      - `project_id=<project uuid>`
      - `name="Q4 Launch Messaging Framework"` (or whatever the source title was)
      - `content=<markdown body from get_content>` — using the `content` path (not `instructions`), per skill §6.6, because we're saving pre-written text, not regenerating.
    - **Rationale:** Skill §6.1 — `create_content(project_id=...)` is the in-MCP way to put new content directly into a project's documents. Sync path — no polling needed.
- **If user picks (1) "regenerate with project context":**
  - **Tool:** `mcp__marketcore-dev__create_content`
  - **Params:**
    - `project_id=<project uuid>`
    - `name="Q4 Launch Messaging Framework"`
    - `instructions="<paraphrase of original intent, e.g. 'A messaging framework for the Q4 Product Launch — pillars, proof points, audience targets. Use the project's context.'>"`
  - **Rationale:** Skill §2.2 four-layer context model — project-scoped generation auto-pulls Brand Foundation + Reference Library + Project Context. Will likely improve on the standalone doc. Sync, 1-3 minutes — warn the user (skill §6.7).
- **If user picks (2):** Stop. Tell them I'll wait — and to ping me once they've added the doc in the app.

##### Step 8B — Set the brief (only after step 7B succeeds)

- **Tool:** `mcp__marketcore-dev__update_project`
- **Params:** `project_id=<uuid>`, `project_brief_id=<new content uuid from step 7B>`
- **Rationale:** Same as Branch A step 6A.

##### Step 9B — Verify

- Same as Branch A step 7A: `get_project` to confirm.

---

## 3. What I surface back to the user at each meaningful step

| Step | What I say |
|---|---|
| Before step 1 | The opening message above (plan + "starting now"). |
| After steps 2-4 | Either (a) "Found 'Q4 Product Launch' and your messaging doc 'X'" + the next-step plan, or (b) the disambiguation question (project not found / multiple docs / etc.). I do NOT narrate every individual `list_*` call — I batch the lookups and report the outcome once. |
| After step 5 (the comparison) | One sentence: either "It's already in the project — going to set it as the brief now" (Branch A step 6A) or the Branch B gap surfacing message. |
| After the `update_project` call | "Done. 'Q4 Launch Messaging Framework' is now the brief on Q4 Product Launch. [link to the project]." Pulled from the `link_url` on the project response. |
| After verify | Skip if the update response already includes the new brief reference; otherwise a short "confirmed". |

I keep narration light between turns — the skill says to state the plan **before** non-trivial mutating calls (§3.1), not to live-narrate every read.

---

## 4. Final action when the brief is set

- Confirm the change in one line, including the project's `link_url` so the user can click straight into the app and see the brief in place.
- Offer one logical next step (don't stack three): *"Want me to draft something off this brief now — e.g. the launch announcement, sales one-pager, or in-app message? Or are we good?"*
- Do **not** delete or modify the standalone source doc (Branch B path 1) without being asked. Two copies is the user's call.

---

## 5. What I do if the messaging document is not yet in the project's documents

This is the known precondition. My handling is **Branch B above** (steps 6B → 7B → 8B → 9B), summarized:

1. **Detect it via `get_project`** — the messaging doc's UUID is not in the `documents` array.
2. **Stop and surface to the user** — don't silently work around it. Skill §6.1 is explicit that this is the gating constraint, and the user needs to know there's a fork.
3. **Offer the two viable paths:**
   - **Path 1 (in-MCP):** `create_content(project_id=...)` to put a copy of the messaging doc into the project (either by passing the original markdown via `content`, or by regenerating via `instructions` with project context layered in). Then `update_project` to set it as the brief.
   - **Path 2 (in-app):** User adds the existing standalone doc to the project's documents via the MarketCore app ("+ Add Document"), then I run `update_project`. There's no MCP tool to attach an existing standalone content to a project — skill §6.1 is explicit.
4. **Wait for the user's choice** before mutating anything. Do not pre-emptively create a duplicate document — that's a "wrong place, wrong thing" failure mode (skill §3.2 is built around avoiding exactly this).
5. **Execute the chosen path**, then `update_project`, then verify with `get_project`.

The trap I'm avoiding: blindly creating a project-scoped copy without asking would leave the user with two messaging docs in different places, the standalone doc orphaned, and no way to tell which is canonical. That's worse than the extra turn.
