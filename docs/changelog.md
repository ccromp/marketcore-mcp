# Changelog

All notable changes to the MarketCore MCP server will be documented in this file.

## 2026-04-16

### Enhanced

- `add_context` — Added `link_url` field returning a direct URL to view the new context item in the MarketCore app. The URL resolves to the project, collection, or reference library view depending on which scope the item was added to.
- `create_context_collection` — Added `link_url` field returning a direct URL to open the new collection in the MarketCore app.
- `get_context_collections` — Added `link_url` field on each returned collection. Documentation expanded to include the full output field table.

## 2026-04-14

### Enhanced

- `get_projects` — Added `link_url` field returning a direct URL to view each project in the MarketCore app. Expanded output documentation with full field table.

## 2026-04-09

### Initial Release

First public documentation for the MarketCore MCP Server with 25 tools across 7 categories:

**Account**
- `get_current_user_info`

**Context & Resources**
- `get_core_context`
- `get_context_collections`
- `create_context_collection`
- `add_context`
- `get_relevant_context`

**Reference**
- `get_content_categories`
- `get_targeting_dimensions`

**Blueprints**
- `get_blueprints`
- `get_blueprint`
- `create_blueprint`
- `create_blueprint_draft`
- `finalize_blueprint_draft`

**Community Blueprints**
- `get_community_blueprints`
- `get_community_blueprint_details`
- `import_community_blueprint`

**Content**
- `create_content`
- `get_generation_status`
- `get_content_list`
- `get_content`

**Sharing & Export**
- `create_external_share`
- `convert_markdown_to_word_doc`

**Projects**
- `get_projects`
- `get_project`
- `create_project`
