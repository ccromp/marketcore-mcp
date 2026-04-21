# MarketCore MCP Server

> AI-powered product context management — connect your AI tools to MarketCore through the Model Context Protocol.

## About

This is the **public documentation repository** for MarketCore's hosted MCP server. The server itself is a commercial, closed-source remote service. This repo provides metadata, connection instructions, tool documentation, and support resources.

| | |
|---|---|
| **Type** | Hosted remote MCP server |
| **Implementation** | Closed-source (commercial) |
| **Transport** | Streamable HTTP |
| **Authentication** | OAuth 2.0 or API token |
| **Status** | Production |

## What It Does

MarketCore is a product context management platform for go-to-market teams. It stores your brand context, manages reusable content templates (blueprints), and generates marketing documents informed by your full product knowledge. The MCP server brings all of this into your AI assistant — create content, manage context, browse community templates, and organize projects without leaving your chat.

MarketCore's MCP server enables AI clients to:

- Manage product context and reference materials
- Create and manage blueprints for content generation
- Generate marketing content from context and blueprints
- Organize work with projects and collections
- Browse and import community blueprints and templates
- Share content externally and export to Word

## Connection

**Server URL (OAuth):** `https://mcp.marketcore.ai`

**Authentication options:**
- **OAuth 2.0** (recommended for interactive clients) — connect using `https://mcp.marketcore.ai`
- **API Token** (for non-interactive environments) — generate a key at [Integration Settings](https://app.marketcore.ai/integration-settings) and use one of the API key URLs below

**API Key URLs:**

| Transport | URL |
|---|---|
| SSE | `https://api.marketcore.ai/x2/mcp/EbZaDl-X/mcp/sse` |
| Streamable HTTP | `https://api.marketcore.ai/x2/mcp/EbZaDl-X/mcp/stream` |

## Quickstart

Add to your MCP client config:

```json
{
  "mcpServers": {
    "marketcore": {
      "url": "https://mcp.marketcore.ai"
    }
  }
}
```

See [docs/quickstart.md](docs/quickstart.md) for setup instructions for Claude, ChatGPT, Claude Desktop, Cursor, and VS Code.

## AI companion skill (recommended)

A companion **Anthropic Skill** ships alongside this server. It teaches your AI client MarketCore's mental model, the right tool sequencing, the questions to ask the user before acting, and the pitfalls to avoid — so the agent stops misfiring on tasks that look ambiguous from tool descriptions alone (e.g. setting a project brief, choosing between context destinations, generating from a blueprint vs. freeform).

The skill lives at [`skill/marketcore-mcp/`](skill/marketcore-mcp/). Install it in whichever client supports Anthropic Skills:

### Claude Code

Clone the repo and copy or symlink the skill into your skills directory:

```bash
git clone https://github.com/ccromp/marketcore-mcp.git
ln -s "$(pwd)/marketcore-mcp/skill/marketcore-mcp" ~/.claude/skills/marketcore-mcp
```

Restart Claude Code. The skill loads automatically on any MarketCore-related task.

### Claude Desktop

Drop the `skill/marketcore-mcp/` directory into your Claude Desktop Skills folder. See [Anthropic's Skills documentation](https://support.anthropic.com/en/articles/agent-skills) for the current path on your platform.

### ChatGPT, Cursor, and other non-Anthropic clients

These clients don't directly support Anthropic Skills (yet). As a workaround, paste the contents of [`skill/marketcore-mcp/SKILL.md`](skill/marketcore-mcp/SKILL.md) into your client's custom instructions or system-prompt slot. The two reference files (`workflows.md` and `pitfalls.md`) can be loaded on demand or pasted in if your client supports longer context.

### Releases

Skill releases are tagged `skill-vX.Y.Z` — see [Releases](https://github.com/ccromp/marketcore-mcp/releases). The current version is in the `metadata.version` field of `SKILL.md`.

## Available Tools

| Category | Tool | Description |
|---|---|---|
| Account | `get_current_user_info` | Get profile, subscription, and usage info |
| Context & Resources | `get_core_context` | Get team's core brand context |
| Context & Resources | `get_context_collections` | List context collections |
| Context & Resources | `create_context_collection` | Create a new context collection |
| Context & Resources | `add_context` | Add a context item to your library |
| Context & Resources | `get_relevant_context` | Search context library by prompt |
| Reference | `get_content_categories` | List content categories |
| Reference | `get_targeting_dimensions` | List targeting dimensions and options |
| Blueprints | `get_blueprints` | List all blueprints |
| Blueprints | `get_blueprint` | Get blueprint details by UUID |
| Blueprints | `create_blueprint` | Create a reusable content template |
| Blueprints | `create_blueprint_draft` | Create an AI-assisted blueprint draft |
| Blueprints | `finalize_blueprint_draft` | Publish a blueprint draft |
| Community Blueprints | `get_community_blueprints` | Browse community templates |
| Community Blueprints | `get_community_blueprint_details` | Get community blueprint details |
| Community Blueprints | `import_community_blueprint` | Import a community blueprint |
| Content | `create_content` | Generate content (with or without blueprint) |
| Content | `get_generation_status` | Check async generation status |
| Content | `get_content_list` | List all content |
| Content | `get_content` | Get full content by ID |
| Sharing & Export | `create_external_share` | Create a public share link |
| Sharing & Export | `convert_markdown_to_word_doc` | Export markdown as Word doc |
| Projects | `get_projects` | List all projects |
| Projects | `get_project` | Get project details |
| Projects | `create_project` | Create a new project |

See [docs/tools.md](docs/tools.md) for the complete tool reference.

## Documentation

- [Overview](docs/overview.md)
- [Quickstart](docs/quickstart.md)
- [Authentication](docs/authentication.md)
- [Tools Reference](docs/tools.md)
- [Errors & Troubleshooting](docs/errors.md)
- [Security & Privacy](docs/security.md)
- [Changelog](docs/changelog.md)
- [Support](docs/support.md)
- [MarketCore Website Documentation](https://marketcore.ai/mcp)

## Security & Privacy

See [docs/security.md](docs/security.md) for details on data handling, permissions, and responsible disclosure.

## Support

- **Bug reports & feature requests:** [GitHub Issues](https://github.com/ccromp/marketcore-mcp/issues)
- **Website:** [marketcore.ai](https://marketcore.ai)

## License

This repository contains documentation and metadata only. The MarketCore MCP server implementation is proprietary and not included.

Documentation is licensed under [CC BY 4.0](LICENSE).
