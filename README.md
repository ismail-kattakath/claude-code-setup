# claude-code-setup

A Claude Code skill that **audits, scaffolds, and converts any project** to fully comply with the official [Build with Claude Code](https://code.claude.com/docs/en/agents) specification.

## What it does

In five phases:

1. **BOOTSTRAP** — Checks for required sub-skills (`agent-development`, `hook-development`, `mcp-integration`, `plugin-structure`, `command-development`, `skill-development`) and installs any that are missing from `anthropics/claude-code`
2. **DETECT** — Determines if the project is a regular project or a Claude Code plugin, and audits the existing structure against spec
3. **SCAFFOLD** — Creates all missing directories and files (without overwriting existing ones)
4. **CONFIGURE** — Fills in required files using spec-compliant templates (CLAUDE.md, settings.json, plugin.json, hooks.json, .mcp.json)
5. **VALIDATE** — Runs structural and JSON validation checks, outputs a compliance report

## Skills it orchestrates

This skill acts as an orchestrator, invoking sub-skills for each component:

| Component | Sub-skill |
|-----------|-----------|
| Plugin directory layout, `plugin.json` | `plugin-structure` |
| Agent `.md` files with `<example>` blocks | `agent-development` |
| `hooks.json` / `settings.json` hooks | `hook-development` |
| `.mcp.json` MCP server configuration | `mcp-integration` |
| Slash command `.md` files | `command-development` |
| `skills/*/SKILL.md` skill files | `skill-development` |

## Installation

```bash
npx skills add ismail-kattakath/claude-code-setup
```

Or install globally:

```bash
cd ~ && npx skills add ismail-kattakath/claude-code-setup
```

## Trigger phrases

The skill activates when you ask Claude Code to:

- "set up Claude Code" / "scaffold a Claude Code project"
- "convert project to Claude Code"
- "add Claude Code structure"
- "audit .claude directory"
- "make this Claude Code compliant"
- "add agents/hooks/skills/commands to project"
- "build with Claude Code"
- "claude code compliance check"
- "initialize claude code project"

## Skill structure

```
claude-code-setup/
├── SKILL.md                          # Main instructions (283 lines)
├── references/
│   ├── spec-checklist.md             # Complete compliance audit checklist
│   └── templates.md                  # Copy-paste templates for all spec files
└── evals/
    └── evals.json                    # Test cases for skill evaluation
```

## Covered spec requirements

- `CLAUDE.md` — all 5 required sections, ≤200 line limit
- `.claude/settings.json` — permissions, hooks, permission_mode
- `.claude/agents/*.md` — frontmatter with `<example>` blocks (required for triggering)
- `.claude/skills/<name>/SKILL.md` — progressive disclosure structure
- `.claude-plugin/plugin.json` — `${CLAUDE_PLUGIN_ROOT}` paths, auto-discovery
- `hooks/hooks.json` — plugin format (with `"hooks"` wrapper key)
- `.mcp.json` — all 4 transport types (stdio, http, sse, ws)
- `.gitignore` — `settings.local.json`, `*.local.md`, `.claude/worktrees/`
- `.worktreeinclude` — gitignored files to copy into worktrees
- Naming conventions — kebab-case throughout

## License

MIT
