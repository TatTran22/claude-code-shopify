# Claude Code Shopify

**Complete collection of Claude Code configs for building Shopify apps with Go backend and React frontend.**

Production-ready agents, skills, hooks, commands, rules, and MCP configurations optimized for **Go/Fiber/Redis/React/Vite/Shopify Polaris Web Components stack**.

Comprehensive patterns for building backend APIs with Go, Shopify embedded apps with Polaris Web Components, queue workers with RabbitMQ, and modern React frontends using React Hook Form + Zod.

---

## What's Inside

This repo is a **Claude Code plugin** - install it directly or copy components manually.

```
claude-code-shopify/
|-- .claude-plugin/   # Plugin and marketplace manifests
|   |-- plugin.json         # Plugin metadata and component paths
|   |-- marketplace.json    # Marketplace catalog for /plugin marketplace add
|
|-- agents/           # Specialized subagents for delegation
|   |-- planner.md           # Feature implementation planning
|   |-- architect.md         # System design decisions
|   |-- tdd-guide.md         # Test-driven development with Go
|   |-- code-reviewer.md     # Go code quality and security review
|   |-- security-reviewer.md # Go + Shopify vulnerability analysis
|   |-- build-error-resolver.md # Go compilation error fixes
|   |-- e2e-runner.md        # Playwright E2E testing
|   |-- refactor-cleaner.md  # Dead code cleanup
|   |-- doc-updater.md       # Documentation sync
|
|-- skills/           # Workflow definitions and domain knowledge
|   |-- coding-standards/           # Go coding conventions and best practices
|   |-- backend-patterns/           # Go/Fiber v3, PostgreSQL (pgx), Redis caching
|   |-- frontend-patterns/          # React/Vite, Shopify Polaris components
|   |-- shopify-integration/        # OAuth, webhooks, GraphQL/REST APIs, GDPR compliance
|   |-- queue-worker-patterns/      # RabbitMQ producers, consumers, DLQ, retry strategies
|   |-- tdd-workflow/               # TDD methodology with Go table-driven tests
|   |-- security-review/            # Go + Shopify security checklist
|   |-- project-guidelines-example/ # Example project setup
|
|-- commands/         # Slash commands for quick execution
|   |-- tdd.md              # /tdd - Test-driven development (Go table-driven tests)
|   |-- plan.md             # /plan - Implementation planning
|   |-- e2e.md              # /e2e - E2E test generation
|   |-- code-review.md      # /code-review - Go code quality review
|   |-- build-fix.md        # /build-fix - Fix Go compilation errors
|   |-- refactor-clean.md   # /refactor-clean - Dead code removal
|
|-- rules/            # Always-follow guidelines (copy to ~/.claude/rules/)
|   |-- security.md         # Mandatory security checks (Go + Shopify)
|   |-- coding-style.md     # Go conventions (gofmt, error handling, MixedCaps)
|   |-- testing.md          # TDD, table-driven tests, 80% coverage requirement
|   |-- patterns.md         # Go patterns (Repository, Service layer, Fiber middleware)
|   |-- git-workflow.md     # Commit format, PR process
|   |-- agents.md           # When to delegate to subagents
|   |-- performance.md      # Model selection, context management
|
|-- hooks/            # Trigger-based automations
|   |-- hooks.json          # All hooks config (PreToolUse, PostToolUse, Stop, etc.)
|
|-- examples/         # Example configurations
|   |-- CLAUDE.md           # Example project-level config
|   |-- user-CLAUDE.md      # Example user-level config
|
|-- mcp-configs/      # MCP server configurations
|   |-- mcp-servers.json    # Shopify, PostgreSQL, etc.
```

---

## Installation

### Option 1: Install as Plugin (Recommended)

The easiest way to use this repo - install as a Claude Code plugin:

```bash
# Add this repo as a marketplace
/plugin marketplace add TatTran22/claude-code-shopify

# Install the plugin
/plugin install claude-code-shopify@claude-code-shopify
```

Or add directly to your `~/.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "claude-code-shopify": {
      "source": {
        "source": "github",
        "repo": "TatTran22/claude-code-shopify"
      }
    }
  },
  "enabledPlugins": {
    "claude-code-shopify@claude-code-shopify": true
  }
}
```

This gives you instant access to all commands, agents, skills, and hooks.

---

### Option 2: Manual Installation

If you prefer manual control over what's installed:

```bash
# Clone the repo
git clone https://github.com/TatTran22/claude-code-shopify.git

# Copy agents to your Claude config
cp claude-code-shopify/agents/*.md ~/.claude/agents/

# Copy rules
cp claude-code-shopify/rules/*.md ~/.claude/rules/

# Copy commands
cp claude-code-shopify/commands/*.md ~/.claude/commands/

# Copy skills
cp -r claude-code-shopify/skills/* ~/.claude/skills/
```

#### Add hooks to settings.json

Copy the hooks from `hooks/hooks.json` to your `~/.claude/settings.json`.

#### Configure MCPs

Copy desired MCP servers from `mcp-configs/mcp-servers.json` to your `~/.claude.json`.

**Important:** Replace `YOUR_*_HERE` placeholders with your actual API keys.

---

## Stack Features

This plugin is optimized for the following tech stack:

**Backend:**
- **Go 1.21+** with Fiber v3 - High-performance web framework (fasthttp, prefork)
- **PostgreSQL 17** with pgx - Type-safe parameterized queries
- **Redis 7** - Caching patterns (cache-aside, write-through)
- **RabbitMQ 3.12** - Queue workers with retry and DLQ patterns

**Frontend:**
- **React 19** with Vite 7 - Fast development with HMR
- **Shopify Polaris Web Components** - CDN-loaded components (`s-page`, `s-button`, etc.)
- **React Hook Form + Zod** - Type-safe form validation
- **TanStack Query** - Server state with query key factories
- **TypeScript** - Type-safe frontend code

**Shopify Integration:**
- OAuth authentication flow
- Webhook HMAC verification (CRITICAL for security)
- GraphQL Admin API patterns
- GDPR compliance (mandatory webhooks)
- Session token validation

**Monitoring:**
- Prometheus + Grafana integration patterns
- Structured logging (slog/zap)

All skills, agents, and hooks are tailored for this stack with production-ready patterns and security best practices.

---

## Key Concepts

### Agents

Subagents handle delegated tasks with limited scope. Example:

```markdown
---
name: code-reviewer
description: Reviews code for quality, security, and maintainability
tools: Read, Grep, Glob, Bash
model: opus
---

You are a senior code reviewer...
```

### Skills

Skills are workflow definitions invoked by commands or agents:

```markdown
# TDD Workflow (Go)

1. Define interfaces and types first
2. Write table-driven tests (RED)
3. Implement minimal code (GREEN)
4. Refactor with Go conventions (IMPROVE)
5. Run go test -cover (80%+ required)
```

### Hooks

Hooks fire on tool events. Example - auto-format Go files:

```json
{
  "matcher": "tool == \"Edit\" && tool_input.file_path matches \"\\\\.(go)$\"",
  "hooks": [{
    "type": "command",
    "command": "#!/bin/bash\ngofmt -w \"$file_path\" && goimports -w \"$file_path\" && echo '[Hook] Formatted Go file' >&2"
  }]
}
```

Hooks included:
- **gofmt/goimports** - Auto-format Go files after edits
- **fmt.Println warning** - Suggest structured logging (slog/zap)
- **console.log warning** - Warn about console.log in frontend files
- **Dev server tmux** - Ensure dev servers run in tmux for log access

### Rules

Rules are always-follow guidelines. Keep them modular:

```
~/.claude/rules/
  security.md      # No hardcoded secrets, SQL injection prevention, HMAC verification
  coding-style.md  # Go conventions (gofmt, MixedCaps, error wrapping with %w)
  testing.md       # TDD, table-driven tests, 80% coverage
  patterns.md      # Go patterns (Repository, Service layer, Fiber middleware)
```

---

## Important Notes

### Context Window Management

**Critical:** Don't enable all MCPs at once. Your 200k context window can shrink to 70k with too many tools enabled.

Rule of thumb:
- Have 20-30 MCPs configured
- Keep under 10 enabled per project
- Under 80 tools active

Use `disabledMcpServers` in project config to disable unused ones.

### Customization

These configs are optimized for **Go/Fiber/Redis/React/Vite/Shopify Polaris** stack. You should:
1. Start with what resonates
2. Modify for your specific tech stack
3. Remove skills/patterns you don't use (e.g., Shopify if not building Shopify apps)
4. Add your own patterns

**For other stacks:** The architecture (agents, skills, hooks, rules, commands) works universally. Just replace the language-specific content in skills and rules with your stack's patterns.

---

## Contributing

**Contributions are welcome and encouraged.**

This repo is meant to be a community resource. If you have:
- Useful agents or skills
- Clever hooks
- Better MCP configurations
- Improved rules

Please contribute! Open a PR or issue.

### Ideas for Contributions

- Additional Shopify patterns (App extensions, Checkout UI)
- More Go patterns (gRPC, GraphQL servers)
- DevOps agents (Kubernetes, Terraform, AWS)
- Testing strategies (integration tests, load testing)
- Additional queue patterns (Kafka, NATS)

---

## Credits

This plugin is based on the excellent work from [everything-claude-code](https://github.com/affaan-m/everything-claude-code) by [@affaanmustafa](https://x.com/affaanmustafa), forked and optimized for Go/React/Shopify development.

For the original guides and advanced techniques (token optimization, memory persistence, verification loops), check out:
- [The Shorthand Guide](https://x.com/affaanmustafa/status/2012378465664745795)
- [The Longform Guide](https://x.com/affaanmustafa/status/2014040193557471352)

---

## Author

**Tat Tran**
- GitHub: [TatTran22](https://github.com/TatTran22)
- Email: tattran.dev@gmail.com

---

## License

MIT - Use freely, modify as needed, contribute back if you can.
