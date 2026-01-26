# Plugins and Marketplaces

Plugins extend Claude Code with new tools and capabilities. This guide covers installation and recommended plugins for Go/React/Shopify development.

---

## Marketplaces

Marketplaces are repositories of installable plugins.

### Adding a Marketplace

```bash
# Add official Anthropic marketplace
claude plugin marketplace add https://github.com/anthropics/claude-plugins-official

# Add community marketplaces
claude plugin marketplace add https://github.com/mixedbread-ai/mgrep
```

### Recommended Marketplaces

| Marketplace | Source |
|-------------|--------|
| claude-plugins-official | `anthropics/claude-plugins-official` |
| claude-code-plugins | `anthropics/claude-code` |
| Mixedbread-Grep | `mixedbread-ai/mgrep` |

---

## Installing Plugins

```bash
# Open plugins browser
/plugins

# Or install directly
claude plugin install typescript-lsp@claude-plugins-official
```

### Recommended Plugins for Go/React/Shopify Stack

**Go Development:**
- `gopls` - Go language server (if available)
- `golangci-lint` - Go linting integration

**TypeScript/React:**
- `typescript-lsp` - TypeScript intelligence
- `context7` - Live documentation lookup (React, Shopify APIs)

**Code Quality:**
- `code-review` - Code review
- `pr-review-toolkit` - PR automation
- `security-guidance` - Security checks (important for Shopify HMAC verification)

**Search:**
- `mgrep` - Enhanced search (better than ripgrep)

**Workflow:**
- `commit-commands` - Git workflow
- `frontend-design` - UI patterns for Polaris components
- `feature-dev` - Feature development

---

## Quick Setup

```bash
# Add marketplaces
claude plugin marketplace add https://github.com/anthropics/claude-plugins-official
claude plugin marketplace add https://github.com/mixedbread-ai/mgrep

# Open /plugins and install what you need
```

---

## Plugin Files Location

```
~/.claude/plugins/
|-- cache/                    # Downloaded plugins
|-- installed_plugins.json    # Installed list
|-- known_marketplaces.json   # Added marketplaces
|-- marketplaces/             # Marketplace data
```

---

## MCP Servers for Shopify Development

Consider adding these MCP servers for Shopify app development:

```json
{
  "mcpServers": {
    "shopify-dev-mcp": {
      "command": "npx",
      "args": ["-y", "@shopify/dev-mcp@latest"]
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "postgresql://..."
      }
    }
  }
}
```

See the main README.md for full MCP configuration details.
