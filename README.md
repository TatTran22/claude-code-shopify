# Everything Claude Code

**The complete collection of Claude Code configs from an Anthropic hackathon winner.**

Production-ready agents, skills, hooks, commands, rules, and MCP configurations evolved over 10+ months of intensive daily use building real products.

**Optimized for Go/Chi/Redis/React/Vite/Shopify Polaris Web Components stack** - comprehensive patterns for building backend APIs with Go, Shopify embedded apps with Polaris Web Components, queue workers with RabbitMQ, and modern React frontends using React Hook Form + Zod.

---

## The Guides

This repo is the raw code only. The guides explain everything.

### Start Here: The Shorthand Guide

<img width="592" height="445" alt="image" src="https://github.com/user-attachments/assets/1a471488-59cc-425b-8345-5245c7efbcef" />

**[The Shorthand Guide to Everything Claude Code](https://x.com/affaanmustafa/status/2012378465664745795)**

The foundation - what each config type does, how to structure your setup, context window management, and the philosophy behind these configs. **Read this first.**

---

### Then: The Longform Guide

<img width="609" height="428" alt="image" src="https://github.com/user-attachments/assets/c9ca43bc-b149-427f-b551-af6840c368f0" />

**[The Longform Guide to Everything Claude Code](https://x.com/affaanmustafa/status/2014040193557471352)**

The advanced techniques - token optimization, memory persistence across sessions, verification loops & evals, parallelization strategies, subagent orchestration, and continuous learning. Everything in this guide has working code in this repo.

| Topic | What You'll Learn |
|-------|-------------------|
| Token Optimization | Model selection, system prompt slimming, background processes |
| Memory Persistence | Hooks that save/load context across sessions automatically |
| Continuous Learning | Auto-extract patterns from sessions into reusable skills |
| Verification Loops | Checkpoint vs continuous evals, grader types, pass@k metrics |
| Parallelization | Git worktrees, cascade method, when to scale instances |
| Subagent Orchestration | The context problem, iterative retrieval pattern |


---

## What's Inside

This repo is a **Claude Code plugin** - install it directly or copy components manually.

```
everything-claude-code/
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
|   |-- backend-patterns/           # Go/Chi router, PostgreSQL (pgx), Redis caching
|   |-- frontend-patterns/          # React/Vite, Shopify Polaris components
|   |-- shopify-integration/        # OAuth, webhooks, GraphQL/REST APIs, GDPR compliance
|   |-- queue-worker-patterns/      # RabbitMQ producers, consumers, DLQ, retry strategies
|   |-- continuous-learning/        # Auto-extract patterns from sessions (Longform Guide)
|   |-- strategic-compact/          # Manual compaction suggestions (Longform Guide)
|   |-- tdd-workflow/               # TDD methodology with Go table-driven tests
|   |-- security-review/            # Go + Shopify security checklist
|   |-- eval-harness/               # Verification loop evaluation (Longform Guide)
|   |-- verification-loop/          # Continuous verification (Longform Guide)
|
|-- commands/         # Slash commands for quick execution
|   |-- tdd.md              # /tdd - Test-driven development (Go table-driven tests)
|   |-- plan.md             # /plan - Implementation planning
|   |-- e2e.md              # /e2e - E2E test generation
|   |-- code-review.md      # /code-review - Go code quality review
|   |-- build-fix.md        # /build-fix - Fix Go compilation errors
|   |-- refactor-clean.md   # /refactor-clean - Dead code removal
|   |-- learn.md            # /learn - Extract patterns mid-session (Longform Guide)
|   |-- checkpoint.md       # /checkpoint - Save verification state (Longform Guide)
|   |-- verify.md           # /verify - Run verification loop (Longform Guide)
|
|-- rules/            # Always-follow guidelines (copy to ~/.claude/rules/)
|   |-- security.md         # Mandatory security checks (Go + Shopify)
|   |-- coding-style.md     # Go conventions (gofmt, error handling, MixedCaps)
|   |-- testing.md          # TDD, table-driven tests, 80% coverage requirement
|   |-- patterns.md         # Go patterns (Repository, Service layer, Chi middleware)
|   |-- git-workflow.md     # Commit format, PR process
|   |-- agents.md           # When to delegate to subagents
|   |-- performance.md      # Model selection, context management
|
|-- hooks/            # Trigger-based automations
|   |-- hooks.json                # All hooks config (PreToolUse, PostToolUse, Stop, etc.)
|   |-- memory-persistence/       # Session lifecycle hooks (Longform Guide)
|   |-- strategic-compact/        # Compaction suggestions (Longform Guide)
|
|-- contexts/         # Dynamic system prompt injection contexts (Longform Guide)
|   |-- dev.md              # Development mode context
|   |-- review.md           # Code review mode context
|   |-- research.md         # Research/exploration mode context
|
|-- examples/         # Example configurations and sessions
|   |-- CLAUDE.md           # Example project-level config
|   |-- user-CLAUDE.md      # Example user-level config
|
|-- mcp-configs/      # MCP server configurations
|   |-- mcp-servers.json    # GitHub, Supabase, Vercel, Railway, etc.
|
|-- marketplace.json  # Self-hosted marketplace config (for /plugin marketplace add)
```

---

## Installation

### Option 1: Install as Plugin (Recommended)

The easiest way to use this repo - install as a Claude Code plugin:

```bash
# Add this repo as a marketplace
/plugin marketplace add affaan-m/everything-claude-code

# Install the plugin
/plugin install everything-claude-code@everything-claude-code
```

Or add directly to your `~/.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "everything-claude-code": {
      "source": {
        "source": "github",
        "repo": "affaan-m/everything-claude-code"
      }
    }
  },
  "enabledPlugins": {
    "everything-claude-code@everything-claude-code": true
  }
}
```

This gives you instant access to all commands, agents, skills, and hooks.

---

### Option 2: Manual Installation

If you prefer manual control over what's installed:

```bash
# Clone the repo
git clone https://github.com/affaan-m/everything-claude-code.git

# Copy agents to your Claude config
cp everything-claude-code/agents/*.md ~/.claude/agents/

# Copy rules
cp everything-claude-code/rules/*.md ~/.claude/rules/

# Copy commands
cp everything-claude-code/commands/*.md ~/.claude/commands/

# Copy skills
cp -r everything-claude-code/skills/* ~/.claude/skills/
```

#### Add hooks to settings.json

Copy the hooks from `hooks/hooks.json` to your `~/.claude/settings.json`.

#### Configure MCPs

Copy desired MCP servers from `mcp-configs/mcp-servers.json` to your `~/.claude.json`.

**Important:** Replace `YOUR_*_HERE` placeholders with your actual API keys.

---

### Read the Guides

Seriously, read the guides. These configs make 10x more sense with context.

1. **[Shorthand Guide](https://x.com/affaanmustafa/status/2012378465664745795)** - Setup and foundations
2. **[Longform Guide](https://x.com/affaanmustafa/status/2014040193557471352)** - Advanced techniques (token optimization, memory persistence, evals, parallelization)

---

## Stack Features

This plugin is optimized for the following tech stack:

**Backend:**
- **Go 1.21+** with Chi router - Fast, lightweight HTTP router
- **PostgreSQL 17** with pgx - Type-safe parameterized queries
- **Redis 7** - Caching patterns (cache-aside, write-through)
- **RabbitMQ 3.12** - Queue workers with retry and DLQ patterns

**Frontend:**
- **React 19** with Vite 6 - Fast development with HMR
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
- **Memory persistence** - Save/load session state across context compaction

### Rules

Rules are always-follow guidelines. Keep them modular:

```
~/.claude/rules/
  security.md      # No hardcoded secrets, SQL injection prevention, HMAC verification
  coding-style.md  # Go conventions (gofmt, MixedCaps, error wrapping with %w)
  testing.md       # TDD, table-driven tests, 80% coverage
  patterns.md      # Go patterns (Repository, Service layer, Chi middleware)
```

---

## Contributing

**Contributions are welcome and encouraged.**

This repo is meant to be a community resource. If you have:
- Useful agents or skills
- Clever hooks
- Better MCP configurations
- Improved rules

Please contribute! See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

### Ideas for Contributions

- Language-specific skills (Python, Go, Rust patterns)
- Framework-specific configs (Django, Rails, Laravel)
- DevOps agents (Kubernetes, Terraform, AWS)
- Testing strategies (different frameworks)
- Domain-specific knowledge (ML, data engineering, mobile)

---

## Background

I've been using Claude Code since the experimental rollout. Won the Anthropic x Forum Ventures hackathon in Sep 2025 building [zenith.chat](https://zenith.chat) with [@DRodriguezFX](https://x.com/DRodriguezFX) - entirely using Claude Code.

These configs are battle-tested across multiple production applications.

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

These configs are optimized for **Go/Chi/Redis/React/Vite/Shopify Polaris** stack. You should:
1. Start with what resonates
2. Modify for your specific tech stack
3. Remove skills/patterns you don't use (e.g., Shopify if not building Shopify apps)
4. Add your own patterns (see CONTRIBUTING.md)

**For other stacks:** The architecture (agents, skills, hooks, rules, commands) works universally. Just replace the language-specific content in skills and rules with your stack's patterns.

---

## Links

- **Shorthand Guide (Start Here):** [The Shorthand Guide to Everything Claude Code](https://x.com/affaanmustafa/status/2012378465664745795)
- **Longform Guide (Advanced):** [The Longform Guide to Everything Claude Code](https://x.com/affaanmustafa/status/2014040193557471352)
- **Follow:** [@affaanmustafa](https://x.com/affaanmustafa)
- **zenith.chat:** [zenith.chat](https://zenith.chat)

---

## License

MIT - Use freely, modify as needed, contribute back if you can.

---

**Star this repo if it helps. Read both guides. Build something great.**
