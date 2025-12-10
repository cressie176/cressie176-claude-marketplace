# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a personal Claude Code plugin marketplace for distributing custom plugins, skills, agents, slash commands, hooks, and MCP servers following best practices. Plugins can be consumed from this marketplace via the Claude Code `/plugin` command.

## Architecture

### Marketplace Structure

The repository follows the Claude Code marketplace specification:

- **`.claude-plugin/marketplace.json`**: Central registry defining all available plugins. Each plugin entry includes metadata (name, version, description, author, keywords) and a source path.
- **`plugins/[plugin-name]/`**: Individual plugin directories, each self-contained with their own manifest.

### Plugin Anatomy

Each plugin in `plugins/` follows this structure:

```
plugins/[plugin-name]/
├── .claude-plugin/
│   └── plugin.json          # Required: Plugin manifest
├── commands/                # Optional: Slash commands (.md files)
├── agents/                  # Optional: Custom agents (.md files)
├── skills/                  # Optional: Skills (SKILL.md with frontmatter)
├── hooks/                   # Optional: Event hooks
│   └── hooks.json
└── .mcp.json               # Optional: MCP server configs
```

**Key Concepts:**

1. **Commands** (`commands/*.md`): Explicit user-invoked workflows with `/command-name`. Markdown files with YAML frontmatter specifying `description`, `argument-hint`, `allowed-tools`, and optional `model`.

2. **Agents** (`agents/*.md`): Specialized autonomous agents for complex multi-step tasks. Defined in markdown with YAML frontmatter including `name`, `description`, and `allowed-tools`.

3. **Skills** (`skills/*/SKILL.md`): Capabilities Claude invokes automatically based on context. Must have YAML frontmatter with `name`, `description` (including usage triggers), and optional `allowed-tools`.

4. **Hooks** (`hooks/hooks.json`): Automated actions triggered on events (PreToolUse, PostToolUse, PermissionRequest, etc.). JSON structure with matchers and hook definitions.

5. **MCP Servers** (`.mcp.json`): Model Context Protocol server configurations for external tool integrations.

### Critical Requirements

- **All paths in plugin.json must be relative and start with `./`**
- **Use `${CLAUDE_PLUGIN_ROOT}` in hooks, MCP configs, and scripts for correct path resolution**
- **Command files in `commands/` directory MUST have `.md` extension in filesystem**
- **Skills require `SKILL.md` filename** (case-sensitive)
- **Plugin names must be kebab-case** (lowercase with hyphens)

## Adding a New Plugin

1. Create plugin directory: `mkdir -p plugins/my-plugin/.claude-plugin`
2. Create `plugins/my-plugin/.claude-plugin/plugin.json` with required fields: `name`, `version`, `description`, `author`, `license`
3. Add plugin components (commands, agents, skills, hooks, MCP servers)
4. Register in `.claude-plugin/marketplace.json`:
   ```json
   {
     "plugins": [
       {
         "name": "my-plugin",
         "source": "./my-plugin",
         "version": "1.0.0",
         "description": "...",
         "author": {...},
         "license": "MIT",
         "keywords": [...],
         "strict": true
       }
     ]
   }
   ```

## Command File Format

Slash commands are markdown files with frontmatter:

```markdown
---
description: Brief command description
argument-hint: [arg1] [arg2]
allowed-tools: Bash(git:*), Read, Grep, Glob
model: claude-sonnet-4
---

Command instructions using $1, $2 for positional args or $ARGUMENTS for all args.
```

## Skill File Format

Skills use SKILL.md with comprehensive frontmatter:

```markdown
---
name: skill-name
description: What it does and when Claude should use it. Include usage triggers.
allowed-tools: Read, Grep, Glob
---

# Skill instructions in markdown

Detailed guidance for Claude when executing this skill.
```

The `description` field must explain both functionality AND usage context to help Claude decide when to invoke the skill.

## Distribution Model

Add this marketplace to your projects via `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": [
    {
      "source": "github",
      "repo": "cressie176/claude-marketplace"
    }
  ]
}
```

Or via command: `/plugin marketplace add cressie176/claude-marketplace`

Plugins are then browsable and installable via `/plugin` command in Claude Code.

## Plugin Versioning

Follow semantic versioning (MAJOR.MINOR.PATCH):
- MAJOR: Breaking changes
- MINOR: New features, backwards compatible
- PATCH: Bug fixes

Update both `plugin.json` version AND marketplace.json entry when releasing new versions.
