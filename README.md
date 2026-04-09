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

<!-- TODO: MCP Engineer — populate from Strapi CMS overview -->

MarketCore's MCP server enables AI clients to:

- Manage product context and reference materials
- Create and manage blueprints for content generation
- Generate marketing content from context and blueprints
- Organize work with projects and collections
- Access community blueprints and templates

## Connection

<!-- TODO: MCP Engineer — populate OAuth URL and API token URL from Strapi CMS -->

**Server URL:** `TODO`

**Authentication options:**
- **OAuth 2.0** (recommended for interactive clients) — `TODO: OAuth URL`
- **API Token** (for non-interactive environments) — `TODO: Token URL`

Auth is required before any tool access.

## Quickstart

<!-- TODO: MCP Engineer — add copy-paste config examples for Claude Desktop, Cursor, VS Code -->

See [docs/quickstart.md](docs/quickstart.md) for setup instructions.

## Available Tools

<!-- TODO: MCP Engineer — populate full tool table from Strapi CMS mcp-tools collection -->

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
