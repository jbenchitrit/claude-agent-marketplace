# MCP Server Configurations

MCP (Model Context Protocol) server configuration templates for common tools and integrations.

Copy and adapt the relevant template to your project's `.mcp.json` or `~/.claude/mcp.json`.

## Templates

```
mcp/
├── filesystem.json      # Local filesystem access
├── github.json          # GitHub API integration
├── postgres.json        # PostgreSQL database access
└── slack.json           # Slack workspace integration
```

## Usage

```bash
cp mcp/github.json /path/to/your/project/.mcp.json
# Edit the template with your credentials and settings
```

## Format

Each template follows the MCP configuration schema:

```json
{
  "mcpServers": {
    "server-name": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-name"],
      "env": {
        "API_KEY": "your-key-here"
      }
    }
  }
}
```
