# Tools Reference

The MarketCore MCP Server exposes a set of tools that your AI assistant can call directly to interact with your workspace. Each tool maps to a specific action — from generating content to retrieving context — and can be invoked naturally through conversation.

The tools below are available to any connected AI assistant once the MarketCore MCP Server is configured.

---

## Account

### `get_current_user_info`

Returns profile and subscription information for the currently authenticated user, including active team, role, subscription plan, and usage statistics.

**Parameters:** None

**Output:**

| Field | Type | Description |
|---|---|---|
| `name` | string | User's display name |
| `email` | string | User's email address |
| `active_team_name` | string | Name of the user's active team |
| `active_team_role` | string | User's role on the active team |
| `plan_name` | string | Subscription plan display name |
| `plan_slug` | string | Subscription plan identifier |
| `subscription_status` | string | Current subscription status |
| `usage` | object | Usage statistics for the current billing period |

**Example prompts:**
- "What plan am I on?"
- "Show me my account info"
- "How much usage do I have left?"

---

## Context & Resources

### `get_core_context`

Returns the team's core context — foundational brand and company information used to guide all AI-generated content, including company overview, brand voice, writing style, and examples.

**Parameters:** None

**Output:**

| Field | Type | Description |
|---|---|---|
| `core_context` | string | Full core context document in markdown, containing company overview, brand voice, writing style, and writing examples sections |

**Example prompts:**
- "What's my brand context?"
- "Show me my company's brand voice"
- "What context do I have in MarketCore?"

---

### `get_context_collections`

Returns all context collections accessible to the current user. Collections organize reference materials (context items) that inform AI-generated content.

**Parameters:** None

**Output:** Array of context collections, each with the following fields:

| Field | Type | Description |
|---|---|---|
| `id` | integer | Collection ID. Pass to `add_context` or `get_relevant_context` |
| `name` | string | Collection name |
| `description` | string | Collection description |
| `is_private` | boolean | Whether this collection is private to the creator |
| `item_count` | integer | Number of context items in this collection |
| `link_url` | string | Direct URL to view this collection in the MarketCore app |

**Example prompts:**
- "Show me my context collections"
- "What reference materials do I have?"

---

### `create_context_collection`

Create a new context collection to organize your reference materials. Collections group related context items together for easier management and targeted retrieval.

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | A descriptive name for the collection |
| `description` | string | Yes | A short description of what this collection contains |
| `is_private` | boolean | Yes | If true, only you can see this collection. If false, all team members can access it |

**Output:**

| Field | Type | Description |
|---|---|---|
| `id` | integer | Collection ID. Pass as `collection_id` to `add_context` |
| `name` | string | Collection name |
| `description` | string | Collection description |
| `is_private` | boolean | Whether this collection is private to the creator |
| `created_at` | integer | Unix timestamp of creation |
| `link_url` | string | Direct URL to view this collection in the MarketCore app |

**Example prompts:**
- "Create a new collection called 'Q2 Product Research'"
- "Make a private collection for competitive analysis"

---

### `add_context`

Add a new context item to your reference library. Context items are reference materials that power AI generation — they help the AI produce more accurate, on-brand, and relevant content.

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | Descriptive name for the context item |
| `content` | string | Yes | The reference content to store |
| `collection_id` | integer | No | Collection ID to organize the item (from `get_context_collections` or `create_context_collection`) |
| `project_id` | string | No | Project ID to associate with (from `get_projects`) |

**Output:**

| Field | Type | Description |
|---|---|---|
| `id` | string | Context item ID |
| `name` | string | Context item name |
| `content` | string | The stored reference content |
| `word_count` | integer | Word count of content |
| `collection_id` | integer | Collection this item belongs to (if assigned) |
| `project_id` | string | Project association (if assigned) |
| `created_at` | integer | Unix timestamp of creation |
| `link_url` | string | Direct URL to view this context item in the MarketCore app. Resolves to the project, collection, or reference library view depending on the item's scope |

**Example prompts:**
- "Add our brand guidelines to MarketCore"
- "Store this competitive analysis as context"
- "Add this product brief to the 'Product Launch' collection"

---

### `get_relevant_context`

Searches the team's context library and returns the most relevant chunks for a given prompt. Use this to gather supporting context before generating or refining content.

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `prompt` | string | Yes | Search string describing what context you need |
| `project_id` | string | No | Project ID to scope context search |
| `collection_ids` | array | No | Collection IDs to scope context search |
| `context_rag_ids` | array | No | Previously returned chunk IDs to exclude (for pagination) |

**Output:**

| Field | Type | Description |
|---|---|---|
| `relevant_context` | string | Concatenated relevant context text from matched chunks |
| `context_rag_ids` | array | Chunk IDs returned. Pass back to exclude from future searches |
| `context_item_ids` | array | Parent context item IDs that the chunks belong to |

**Example prompts:**
- "Find context about our enterprise pricing"
- "What do we know about competitor X?"
- "Get context relevant to writing a product launch blog post"

---

## Reference

### `get_content_categories`

Returns all content categories available to your team. Categories organize blueprints and content by type (e.g. GTM Strategy, Product Launch, etc).

**Parameters:** None

**Output:** Array of categories with `id`, `name`, and metadata. Use the `id` as `category_id` when creating blueprints or content.

**Example prompts:**
- "What content categories do I have?"
- "Show me the available content types"

---

### `get_targeting_dimensions`

Returns targeting dimensions and their options for the current team. Dimensions are categories (e.g. Buying Stage, Persona) with selectable options used to target content generation.

**Parameters:** None

**Output:** Array of dimensions, each containing selectable options with IDs. Pass `dimension_option_ids` to `create_content` to target generation.

**Example prompts:**
- "What targeting dimensions are available?"
- "Show me the persona options"

---

## Blueprints

### `get_blueprints`

Get all blueprints in your team's library, organized as a flat list. Each blueprint includes its content category and a web URL.

**Parameters:** None

**Output:**

| Field | Type | Description |
|---|---|---|
| `blueprints` | array | List of published blueprints with UUIDs, names, summaries, and categories |
| `blueprint_drafts` | array | In-progress blueprint drafts not yet finalized |

**Example prompts:**
- "Show me my blueprints"
- "What templates do I have?"
- "List all my content blueprints"

---

### `get_blueprint`

Retrieve the full details of a specific blueprint by its UUID, including content, AI-generated analysis, and metadata.

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `blueprint_uuid` | string | Yes | UUID of the blueprint to retrieve |

**Output:**

| Field | Type | Description |
|---|---|---|
| `name` | string | Blueprint name |
| `summary` | string | AI-generated summary |
| `blueprint_uuid` | string | Unique identifier |
| `source_content` | string | Original template content |
| `reference_content` | string | AI-polished reference version |
| `blueprint_dna` | string | AI-generated analysis of structure, tone, and sections |
| `input_instructions` | string | AI-generated guidance for what context to provide |
| `category` | object/null | Content category |
| `team_visibility` | string | Visibility within your team |
| `exchange_visibility` | string | Community exchange visibility |
| `web_url` | string | Direct URL to view in MarketCore |
| `created_at` | integer | Unix timestamp of creation |

**Example prompts:**
- "Show me the details of my blog post blueprint"
- "What does the product launch blueprint look like?"

---

### `create_blueprint`

Create a reusable blueprint template for generating content at scale. Blueprints define the structure and AI instructions for a document type. Takes 1–3 minutes to return.

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | Blueprint name |
| `category_id` | integer | Yes | Category ID from `get_content_categories` |
| `source_content` | string | Yes | Well-structured markdown template content |

**Output:**

| Field | Type | Description |
|---|---|---|
| `name` | string | Blueprint name |
| `summary` | string | AI-generated summary of what this blueprint produces |
| `blueprint_uuid` | string | Unique identifier for use with `create_content` |
| `blueprint_dna` | string | AI-generated analysis of the template structure |
| `source_content` | string | The original template content |
| `reference_content` | string | AI-polished reference version |
| `input_instructions` | string | Guidance for what context to provide when generating |
| `category` | object | Content category |
| `link_url` | string | Direct link to view in MarketCore |
| `team_visibility` | string | Team visibility setting |
| `created_at` | integer | Unix timestamp of creation |

**Example prompts:**
- "Create a blueprint for case studies"
- "Build me a template for product launch announcements"

**Common errors:**
- Invalid `category_id` — use `get_content_categories` first to get valid IDs
- Empty `source_content` — provide a well-structured markdown template

---

### `create_blueprint_draft`

Create an AI-assisted blueprint draft from a prompt. This creates a draft you can review before saving as a full blueprint — it does not create content directly.

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | Name for the draft |
| `instructions` | string | Yes | Description of the blueprint to create |
| `content` | string | Yes | Initial content as markdown |
| `category_id` | integer | No | Category ID from `get_content_categories` |

**Output:**

| Field | Type | Description |
|---|---|---|
| `id` | integer | Record ID for the blueprint draft |
| `uuid` | string | Unique identifier for this draft |
| `title` | string | Blueprint draft name |
| `content` | string | AI-generated blueprint template content in markdown |
| `link_url` | string | Direct URL to view/edit in MarketCore |

**Example prompts:**
- "Draft a blueprint for weekly newsletters"
- "Help me create a template for competitor battle cards"

---

### `finalize_blueprint_draft`

Finalize (publish) a previously created blueprint draft into a full, usable blueprint. This is the final step in the draft workflow: `create_blueprint_draft` → review → `finalize_blueprint_draft`. Takes 1–3 minutes.

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `draft_uuid` | string | Yes | The UUID of the blueprint draft to finalize (from `create_blueprint_draft`) |
| `name` | string | No | Optional name override |
| `category_id` | integer | No | Optional category ID override |

**Output:**

| Field | Type | Description |
|---|---|---|
| `name` | string | Blueprint name |
| `summary` | string | AI-generated summary |
| `blueprint_uuid` | string | Unique identifier for use with `create_content` |
| `blueprint_dna` | string | AI-generated analysis |
| `source_content` | string | Template content from the finalized draft |
| `reference_content` | string | AI-polished reference version |
| `input_instructions` | string | Guidance for users on what context to provide |
| `category` | object | Content category |
| `link_url` | string | Direct link in MarketCore |
| `team_visibility` | string | Team visibility setting |
| `created_at` | integer | Unix timestamp of creation |

**Example prompts:**
- "Finalize my newsletter blueprint draft"
- "Publish the draft blueprint I just created"

---

## Community Blueprints

### `get_community_blueprints`

Browse community blueprints available for import. Returns blueprints shared by MarketCore users, including name, summary, contributor info, and category.

**Parameters:** None

**Output:** Array of community blueprints with IDs, names, summaries, contributor details, and categories.

**Example prompts:**
- "Show me community blueprints"
- "What templates has the community shared?"
- "Browse blueprints from other users"

---

### `get_community_blueprint_details`

Get the full details for a specific community blueprint, including complete content, style guide, and contributor information.

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `blueprint_exchange_id` | string | No | Blueprint exchange ID from `get_community_blueprints` |

**Output:**

| Field | Type | Description |
|---|---|---|
| `id` | string | Blueprint exchange ID |
| `name` | string | Blueprint name |
| `slug` | string | URL slug |
| `content` | string | Full example document content in markdown |
| `summary` | string | Blueprint summary |
| `content_description` | string | Template structure and style guidelines |
| `input_instructions` | string | What context to provide when generating |
| `contributor_name` | string | Author name |
| `contributor_company` | string | Author company |
| `contributor_job_title` | string | Author job title |
| `suggested_content_type` | string | Recommended content category |
| `is_featured` | boolean | Whether the blueprint is featured |
| `visibility` | string | Visibility setting |

**Example prompts:**
- "Show me the details of that case study blueprint"
- "What does the community blog post template look like?"

---

### `import_community_blueprint`

Import a blueprint from the MarketCore community exchange into your team's library. Once imported, use it like any of your own blueprints to generate content.

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `blueprint_exchange_id` | string | No | Blueprint exchange ID from `get_community_blueprints` |

**Output:**

| Field | Type | Description |
|---|---|---|
| `id` | integer | Blueprint record ID |
| `name` | string | Blueprint name |
| `uuid` | string | Unique identifier for use with `create_content` |
| `content` | string | Blueprint content |
| `team_visibility` | string | Team visibility setting |
| `imported_exchange_id` | string | Original exchange ID |
| `created_at` | integer | Unix timestamp of creation |

**Example prompts:**
- "Import that community case study blueprint"
- "Add the blog post template to my library"

---

## Content

### `create_content`

Create content by supplying your own text directly, generating from an AI prompt, or generating from a blueprint.

You must provide either `content` or `instructions` (not both).

- **With `content`** (synchronous): Saves your supplied text directly as a document — no AI generation. Returns immediately.
- **With `instructions`** (synchronous): Creates a freeform document from an AI prompt. Takes 1–3 minutes.
- **With `instructions` + `blueprint_uuid`** (asynchronous): Generates content from a blueprint template. Returns a `generation_id` to poll via `get_generation_status`. Takes 3–5 minutes.

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `content` | string | No* | Your own text to save directly as a document. Cannot be used with `blueprint_uuid` |
| `instructions` | string | No* | Detailed instructions describing what to create (AI will generate it) |
| `blueprint_uuid` | string | No | Blueprint UUID to generate from. Makes the call async. Only works with `instructions` |
| `project_id` | string | No | Project to associate this content with |
| `category_id` | integer | No | Content category ID |
| `collection_ids` | array | No | Context collection IDs to include |
| `dimension_option_ids` | array | No | Targeting dimension option IDs |
| `use_extended_thinking` | boolean | No | Enable extended thinking for complex content (sync mode only, with `instructions`) |

\* You must provide either `content` or `instructions`, but not both.

**Output (without blueprint — synchronous):**

| Field | Type | Description |
|---|---|---|
| `id` | integer | Content record ID |
| `title` | string | Document title |
| `content` | string | Document content in markdown |
| `content_id` | string | Unique identifier for use with `get_content` and share tools |
| `link_url` | string | Direct URL to view in MarketCore |
| `created_at` | integer | Unix timestamp of creation |

**Output (with blueprint — asynchronous):**

| Field | Type | Description |
|---|---|---|
| `generation_id` | integer | ID to track async generation — pass to `get_generation_status` |

**Example prompts:**
- "Save this document to MarketCore" (with `content`)
- "Write a blog post about our new product launch" (with `instructions`)
- "Generate a case study using my case study blueprint" (with `instructions` + `blueprint_uuid`)
- "Create content from my newsletter blueprint for the enterprise persona"

**Common errors:**
- Providing both `content` and `instructions` — use one or the other
- Using `content` with `blueprint_uuid` — blueprints require `instructions`
- Invalid `blueprint_uuid` — use `get_blueprints` to find valid UUIDs
- Timeout on synchronous calls — AI-generated content can take 1–3 minutes, which is normal

---

### `get_generation_status`

Check the status of an async content generation. Call this after `create_content` (with `blueprint_uuid`), which returns a `generation_id`.

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `generation_id` | string | Yes | The generation ID returned by `create_content` |

**Output:**

| Field | Type | Description |
|---|---|---|
| `generation_id` | string | The generation ID being checked |
| `status` | string | Generation status: `pending`, `gathering context`, `processing`, `completed`, or `failed` |
| `content` | object | Content summary (present when status is `completed`). Use `get_content` with `content_id` to get full content |

**Example prompts:**
- "Check on my content generation"
- "Is my blog post done yet?"

---

### `get_content_list`

Returns all content visible to the current user as a single unified array. Content created from scratch and from blueprints are merged with consistent field names.

**Parameters:** None

**Output:** Array of content items with IDs, names, categories, creation dates, and metadata.

**Example prompts:**
- "Show me all my content"
- "List my documents"
- "What content have I created?"

---

### `get_content`

Retrieves the full content of a specific document by its `content_id` (UUID).

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `content_id` | string | Yes | The UUID of the content to retrieve |

**Output:**

| Field | Type | Description |
|---|---|---|
| `name` | string | Content name |
| `content` | string | Full document content in markdown format |
| `content_id` | string | Content identifier |
| `stage` | string | `in_progress` or `ready` |
| `category` | object/null | Content category, or null if not categorized |
| `visibility` | string | Visibility setting (e.g. `private`, `team`) |
| `link_url` | string | Direct URL to view in MarketCore |

**Example prompts:**
- "Show me the full content of that blog post"
- "Read my latest case study"

---

## Sharing & Export

### `create_external_share`

Creates a public share link for content, with optional expiration. Accessible publicly without a MarketCore account.

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `content_id` | string | Yes | The ID of the content to share |
| `expires_at` | integer | No | Unix timestamp for link expiration |

**Output:**

| Field | Type | Description |
|---|---|---|
| `share_link` | string | Public URL anyone can use to view the content |

**Example prompts:**
- "Create a share link for my latest blog post"
- "Share this document with an expiration in 7 days"

---

### `convert_markdown_to_word_doc`

Export a markdown document as a downloadable Word (.docx) file. Returns a download URL.

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `markdown_content` | string | Yes | Markdown content to convert |
| `filename` | string | No | Filename without extension |
| `document_url` | string | No | URL to embed as footer link to original document |

**Output:**

| Field | Type | Description |
|---|---|---|
| `filename` | string | Filename of the generated Word document |
| `download_url` | string | URL to download the generated .docx file |

**Example prompts:**
- "Export my blog post as a Word doc"
- "Convert this to a .docx file"
- "Download my case study as Word"

---

## Projects

### `get_projects`

Returns all projects visible to the current user. Projects organize content into workstreams.

**Parameters:** None

**Output:**

| Field | Type | Description |
|---|---|---|
| `id` | string (uuid) | Project ID. Pass to `get_project` or use when creating content |
| `name` | string | Project name |
| `link_url` | string (uri) | Direct URL to view this project in the MarketCore app |
| `visibility` | string | Visibility setting (e.g. team, private) |
| `status` | string | Project status (e.g. active, archived) |
| `content_count` | integer | Number of content items in this project |
| `created_by` | string | Name of the project creator |
| `member_count` | integer | Number of project members |

**Example prompts:**
- "Show me my projects"
- "What projects do I have?"

---

### `get_project`

Returns details for a specific project including its members, documents, and context items.

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `project_id` | string | Yes | Project UUID from `get_projects` |

**Output:**

| Field | Type | Description |
|---|---|---|
| `id` | string | Project ID |
| `name` | string | Project name |
| `status` | string | Project status |
| `visibility` | string | Visibility setting |
| `members` | array | Project members |
| `documents` | array | Documents in this project |
| `context_items` | array | Context items associated with this project |
| `created_at` | integer | Unix timestamp of creation |

**Example prompts:**
- "Show me the details of my product launch project"
- "What documents are in this project?"

---

### `create_project`

Create a new project for organizing content and context into a workstream.

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | Project name |
| `visibility` | string | No | Visibility setting |
| `project_brief_details` | string | No | Optional details to create a project brief |

**Output:**

| Field | Type | Description |
|---|---|---|
| `name` | string | Project name |
| `project_id` | string | Project identifier |
| `link_url` | string | Direct URL to view in MarketCore |

**Example prompts:**
- "Create a project for our Q3 product launch"
- "Start a new project called 'Brand Refresh 2025'"

---

### `update_project`

Update mutable fields on an existing project (name, visibility, status, project brief). Uses PATCH semantics — only fields you pass are changed; omit a field to leave it unchanged.

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `project_id` | string (uuid) | Yes | UUID of the project to update. Get from `list_projects` or `get_project`. |
| `name` | string | No | New project name. Must be non-empty when provided. |
| `visibility` | string | No | New visibility setting. `team` makes it visible to all team members; `private` restricts to the creator and explicit project members. |
| `status` | string | No | New project status. `active` for ongoing work; `archived` to hide from the active list while preserving content. Setting to `active` requires available active-project usage. |
| `project_brief_id` | string (uuid) | No | UUID of an existing content item to set as this project's brief. If the content isn't already attached to the project, this tool will attach it AND set it as the brief in one call. |

**Output:**

| Field | Type | Description |
|---|---|---|
| `success` | boolean | True if the update applied successfully |
| `message` | string | Human-readable status message |
| `project` | object \| null | The updated project record |

**Example prompts:**
- "Set the brief on the Acme Launch project to this content"
- "Rename this project to 'Q4 GTM'"
- "Make this project private"
- "Archive the Brand Refresh project"
