# Plan: Acme Corp Case Study for Healthcare CIOs

## 1. First message back to user (verbatim)

> Great — a case study on the Acme Corp deployment, targeted at healthcare CIOs. Before I generate it, I want to make sure we anchor it in real details and the right voice. Let me do a quick check on what we already have on file (your brand voice, any existing Acme context, and the right blueprint to use), then I'll come back with either (a) a small set of clarifying questions, or (b) a draft plan ready to run. Give me about a minute.

(Then I begin the discovery tool calls in parallel.)

---

## 2. Ordered tool calls

### Phase A — Discovery (parallel, no user input required yet)

**A1. `get_current_user_info`**
- Params: none
- Rationale: Confirm which team I'm operating on, plan tier (so I know which generation modes are available), and surface team name for grounding the case study byline.

**A2. `get_core_context`**
- Params: none
- Rationale: Pull the team's brand voice, company overview, writing style, and reference examples. A case study lives or dies on voice — this is non-negotiable before generating.

**A3. `get_relevant_context`** — query: `"Acme Corp deployment"`
- Rationale: RAG search the user's library for anything already on file about Acme — call notes, deployment docs, success metrics, prior collateral. If we have substance here, we don't need to ask the user for it.

**A4. `get_relevant_context`** — query: `"healthcare CIO buyer persona pain points"`
- Rationale: Surface any existing positioning/persona context for healthcare CIOs (compliance, integration risk, EHR interop, board reporting, budget cycles). Lets the case study speak the buyer's language without me inventing it.

**A5. `list_blueprints`**
- Rationale: See if the team already has a "Case Study" blueprint (or a customer-story/deployment-recap blueprint). Blueprints encode structure + voice and produce the highest-quality long-form output. Strongly preferred over a freeform AI-from-prompt run.

**A6. `list_projects`**
- Rationale: Check whether an Acme Corp project already exists. If so, the case study should be created inside it so context, share links, and follow-on assets stay grouped.

**A7. `list_targeting_dimensions`**
- Rationale: Confirm the canonical names for industry (healthcare) and role (CIO) targeting options so the content item is tagged correctly and shows up in the right filters later.

### Phase B — Decision point + clarifying questions to the user

After Phase A returns, I evaluate three branches:

- **B1.** If A3 returned substantive Acme deployment material AND A5 surfaced a usable case-study blueprint AND A2 has a clear voice → I can proceed with **minimal** clarifying questions (Phase B-light).
- **B2.** If A3 is thin or missing → I need source material from the user before generating anything (Phase B-full).
- **B3.** If A5 has no case-study blueprint → I propose creating one, but for *this* request I'll fall back to AI-from-prompt and offer to bottle the structure into a reusable blueprint after.

**Questions I'd ask the user (consolidated into one message, not drip-fed):**

1. **Source material** — "Do you have a deployment recap, win-wire, call notes, or stats sheet for Acme? Paste or point me at it. If not, I can interview you in 4–5 questions."
2. **Approved facts** — "What's the headline outcome (e.g., '37% reduction in claims denial cycle time over 90 days')? Any quotable line from the Acme team I can use, or should I write attributed quotes for you to approve later?"
3. **Acme-side approval status** — "Is Acme cleared as a named customer reference, or do we need to anonymize ('a Fortune 500 healthcare system')?"
4. **Length + format** — "Standard ~1200-word case study, or a punchier ~600-word one-pager? PDF-ready or web-page-ready?"
5. **CTA** — "Ending CTA: book a demo, download the technical deep-dive, or talk to sales?"
6. **Blueprint preference** — If A5 found a blueprint: "Use the existing `<blueprint name>` blueprint, or do you want me to draft something tailored to healthcare?"

I will *not* proceed to generation until I have at least answers to #1, #2, and #3 — those are load-bearing for accuracy and legal.

### Phase C — Capture any new context from the user's reply

**C1. `add_context`** (conditional, one call per new asset)
- Params: `content` = the deployment recap / quotes / metrics the user pastes back; `project_id` = Acme project (creating it in C2 if needed); `title` = e.g., "Acme Corp deployment recap — source for case study".
- Rationale: Preserve the source material so future related assets (sales deck slide, blog teaser, LinkedIn post) can reuse it via `get_relevant_context` instead of re-asking the user.

**C2. `create_project`** (conditional — only if A6 showed no Acme project)
- Params: `name` = "Acme Corp — Healthcare Case Study"; brief includes the targeting (healthcare CIOs), approved facts, and CTA decided in Phase B.
- Rationale: Give the case study a home so the share link, the .docx export, and any follow-on derivatives (LinkedIn carousel, sales-enablement one-pager) stay co-located.

### Phase D — Generation (one of two paths)

**Path D-Blueprint (preferred when a case-study blueprint exists):**

**D1a. `get_blueprint`**
- Params: `id` = the blueprint id from A5
- Rationale: Inspect the blueprint's input fields so I can fill them precisely (e.g., `customer_name`, `industry`, `challenge`, `solution`, `outcome_metrics`, `quote`, `cta`).

**D2a. `create_content`** (AI from blueprint, async 3–5 min)
- Params:
  - `mode`: AI-from-blueprint
  - `blueprint_id`: from D1a
  - `project_id`: from C2 (or A6 hit)
  - `inputs`: filled from Phase B/C answers
  - `targeting`: industry=Healthcare, role=CIO (using the canonical option ids from A7)
  - `category`: case study (looked up via `list_content_categories` if needed)
  - `title`: "How Acme Corp [outcome] with [our product] — A Healthcare CIO Case Study"
- Rationale: Blueprint mode produces the most on-brand, repeatable structure.

**D3a. `get_generation_status`** (poll every ~30s up to 5 min)
- Rationale: Async job — surface a progress update to the user instead of going silent.

**Path D-Prompt (fallback when no usable blueprint):**

**D1b. `create_content`** (AI from prompt, sync 1–3 min)
- Params:
  - `mode`: AI-from-prompt
  - `project_id`: from C2
  - `prompt`: a tightly scoped brief that includes — brand voice excerpts from A2, the Acme facts/quotes from C1, the healthcare-CIO persona points from A4, the headline outcome, the chosen length, and the CTA
  - `targeting`: industry=Healthcare, role=CIO
  - `category`: case study
- Rationale: Sync mode is faster and gives a usable draft when no blueprint fits. After delivery I'll suggest **D-followup**.

**D-followup (offered, not auto-run):**

- **`create_blueprint`** (from the approved final markdown) or **`create_blueprint_draft`** (AI-assisted) — so the next case study is one tool call away. I propose this; I don't run it without a yes.

### Phase E — Delivery

**E1. `get_content`**
- Params: `id` = the new content id
- Rationale: Pull the full text back so I can quote the opening + key metric line into the user-facing summary (don't make them click to verify).

**E2. `create_external_share`**
- Params: `content_id` = E1's id
- Rationale: One-click shareable link the user can forward to product/sales/legal for review.

**E3. `convert_markdown_to_word_doc`**
- Params: `content_id` (or markdown body) from E1
- Rationale: Case studies almost always end up in a sales deck or PDF — having a .docx ready saves a round trip.

---

## 3. What I would surface back to the user at each meaningful step

- **After Phase A** — One short message: "Here's what I found: brand voice ✓, [N] existing Acme context items, [blueprint name] looks like the right structure, no existing Acme project (will create one). I have a few quick questions before I generate." Then the consolidated question list from Phase B.
- **After Phase C** — "Got it. Saved your deployment recap to the Acme project so we can reuse it. Kicking off generation now using the [blueprint name] blueprint — this is async, I'll check back in 3–5 minutes."
- **During Phase D polling** — One status nudge at ~the 2-minute mark: "Still generating, ~halfway through." No silent waits.
- **After Phase D completes** — Surface: title, word count, the opening paragraph (so they can sanity-check voice in 5 seconds), the headline metric line, and a flag for anything I had to assume.
- **After Phase E** — Deliver three links/artifacts in one message: (1) view in MarketCore, (2) public share link, (3) downloadable .docx. Plus: "Want me to bottle this structure into a reusable 'Healthcare Case Study' blueprint, draft a 250-word LinkedIn teaser from it, or spin up a one-pager variant?"

---

## 4. Final action when content is created

A single delivery message containing:

1. **Title** of the new content item.
2. **Direct link** in MarketCore (from the content id).
3. **Public share link** (from E2) — explicitly labeled as shareable for legal/Acme review.
4. **.docx download** (from E3) — for drop-into-deck use.
5. **Two-line preview** — opening sentence + headline metric, so the user can vet the voice without opening the doc.
6. **Assumption call-outs** — any number, quote, or claim I generated rather than pulled from their source material, listed explicitly with "verify before publishing".
7. **Three suggested next moves** (no auto-execution): bottle into a reusable blueprint, derive a LinkedIn teaser, or generate a sales-enablement one-pager variant for AEs.
