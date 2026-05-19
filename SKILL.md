---
name: claude-code-setup
description: "Use when the user asks to: set up claude code, scaffold a plugin, convert project to claude code, add claude code structure, audit .claude structure, make claude code compliant, add agents/hooks/skills, claude code compliance, build with claude code, initialize claude code project, create .mcp.json, scaffold .claude directory, set up settings.json. Handles both regular projects and plugin projects end-to-end: detect type, audit current structure, create all missing files and directories, configure settings/CLAUDE.md/plugin.json/.mcp.json."
license: MIT
compatibility: "Designed for Claude Code. Requires git and bash."
metadata:
  author: aloshy-ai
  version: "1.0"
---

# Claude Code Setup

Sets up, audits, and converts any project to full "Build with Claude Code" spec compliance.
Five phases: BOOTSTRAP → DETECT → SCAFFOLD → CONFIGURE → VALIDATE.

Read `references/spec-checklist.md` when auditing any existing project structure.
Read `references/templates.md` when generating file content from scratch.

---

## Phase 0 — BOOTSTRAP

Before doing anything else, ensure all required sub-skills are installed.

```bash
# Check for each required sub-skill
for skill in agent-development hook-development mcp-integration plugin-structure command-development skill-development; do
  if [ ! -d ~/.agents/skills/$skill ] && [ ! -d ~/.claude/skills/$skill ]; then
    echo "MISSING: $skill"
  fi
done
```

If any are missing, install them from `anthropics/claude-code`:

```bash
DISABLE_TELEMETRY=1 npx skills add anthropics/claude-code \
  --skill "Agent Development" \
  --skill "Hook Development" \
  --skill "MCP Integration" \
  --skill "Plugin Structure" \
  --skill "Command Development" \
  --skill "Skill Development" \
  -g -y
```

Once all six sub-skills are confirmed installed, proceed to Phase 1.

---

## Phase 1 — DETECT

### Determine project type

```bash
# Is this a plugin project?
ls .claude-plugin/plugin.json 2>/dev/null && echo "PLUGIN" || echo "REGULAR"

# Snapshot current state
ls -la .claude/ 2>/dev/null
ls -la .claude-plugin/ 2>/dev/null
cat .claude/settings.json 2>/dev/null
cat CLAUDE.md 2>/dev/null | head -5
cat .mcp.json 2>/dev/null
```

**Plugin project** = has `.claude-plugin/plugin.json` (or user says "this is a plugin").
**Regular project** = everything else (library, app, monorepo, personal project dir).

### Audit against spec

Check each item in `references/spec-checklist.md`. Build two lists:
- **PRESENT**: files/dirs that already exist and look correct
- **MISSING**: everything else

Do not overwrite existing files without asking. For files that exist but look wrong, note them as **NEEDS REVIEW**.

---

## Phase 2 — SCAFFOLD

### Regular project scaffold

```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
cd "$PROJECT_ROOT"

# Core .claude/ structure
mkdir -p .claude/rules .claude/skills .claude/agents .claude/hooks .claude/output-styles

# Create files only if missing
[ -f CLAUDE.md ]               || touch CLAUDE.md
[ -f .mcp.json ]               || touch .mcp.json
[ -f .worktreeinclude ]        || touch .worktreeinclude
[ -f .claude/settings.json ]   || touch .claude/settings.json
[ -f .claude/settings.local.json ] || touch .claude/settings.local.json
```

### Plugin project scaffold (additional)

```bash
mkdir -p .claude-plugin commands agents hooks/scripts
mkdir -p skills  # plugin skills at root, not .claude/skills

[ -f .claude-plugin/plugin.json ] || touch .claude-plugin/plugin.json
[ -f hooks/hooks.json ]           || touch hooks/hooks.json
```

For each new subdirectory, create a `.gitkeep` only if the directory is otherwise empty AND the spec requires the dir to exist (agents/, commands/, etc.).

---

## Phase 3 — CONFIGURE

Fill in every file that was just created (or flag as NEEDS REVIEW if it exists).
Use exact templates from `references/templates.md`.

### 3a. CLAUDE.md

Required sections (keep total under 200 lines):
1. **Project overview** — one paragraph: what it does, primary language/framework
2. **Build / test / lint commands** — exact commands, not prose
3. **Architecture** — key dirs and their purpose (5–10 bullets)
4. **Conventions** — naming, formatting, import style
5. **Important notes** — gotchas, non-obvious constraints

Ask the user for project-specific info you can't infer from the codebase if filling these sections.

### 3b. .claude/settings.json

Populate from the minimal template in `references/templates.md`.
Tailor `allowedTools` to the actual tools the project uses (grep for MCP usage, check package.json/requirements.txt for known tools).

Key fields to set:
- `permissions.allow` — array of tool patterns (e.g. `"Bash(git:*)"`, `"Read"`, `"mcp__server__*"`)
- `hooks` — at minimum a `Stop` hook for quality enforcement
- `permission_mode` — valid values: `"default"` (interactive, recommended), `"dontAsk"` (auto-approve all), `"acceptEdits"` (auto-approve file edits only), `"bypassPermissions"` (no checks, dangerous), `"plan"` (read-only), `"auto"` (smart). Omit to use `"default"`.

### 3c. plugin.json (plugin projects only)

Invoke the **plugin-structure** skill for full plugin.json details and custom path syntax.

Minimum viable content:
```json
{"name": "plugin-name"}
```

All internal paths in plugin.json MUST use `${CLAUDE_PLUGIN_ROOT}`. Never use absolute paths or `~/` shortcuts.

### 3d. hooks/hooks.json (plugin projects only)

Invoke the **hook-development** skill for hooks.json format details.

Plugin format wraps events inside a `"hooks"` key:
```json
{"hooks": {"PreToolUse": [...], "Stop": [...]}}
```

Settings format (`.claude/settings.json`) uses events directly at top level — no `"hooks"` wrapper.

### 3e. .mcp.json

Invoke the **mcp-integration** skill for transport type details.

Prefer `http` transport for remote servers, `stdio` for local processes.
Template in `references/templates.md` covers both patterns.

### 3f. agents/*.md

Invoke the **agent-development** skill for frontmatter format and `<example>` block requirements.

Critical: every agent `description` field MUST contain `<example>` blocks with context/user/assistant/commentary structure. Agents without examples will not trigger reliably.

### 3g. .gitignore additions

Ensure these are gitignored (append to .gitignore if missing):
```
.claude/settings.local.json
*.local.md
.claude/worktrees/
```

---

## Phase 4 — VALIDATE

### Run structural check

```bash
# Verify all required paths exist
for path in \
  "CLAUDE.md" \
  ".mcp.json" \
  ".claude/settings.json" \
  ".claude/settings.local.json" \
  ".claude/rules" \
  ".claude/agents" \
  ".claude/hooks"
do
  [ -e "$path" ] && echo "✓ $path" || echo "✗ MISSING: $path"
done
```

For plugin projects also check:
```bash
for path in \
  ".claude-plugin/plugin.json" \
  "commands" \
  "agents" \
  "hooks/hooks.json"
do
  [ -e "$path" ] && echo "✓ $path" || echo "✗ MISSING: $path"
done
```

### Validate file contents

| File | Check |
|------|-------|
| CLAUDE.md | Has all 5 sections, under 200 lines |
| settings.json | Valid JSON, has `permissions` and `hooks` keys |
| plugin.json | Valid JSON, has `name` field, all paths use `${CLAUDE_PLUGIN_ROOT}` |
| hooks.json | Valid JSON, plugin format has `hooks` wrapper |
| .mcp.json | Valid JSON, no hardcoded credentials |
| agents/*.md | Every file has `description` with `<example>` block |

```bash
# Validate JSON files
for f in .claude/settings.json .mcp.json; do
  [ -f "$f" ] && jq empty "$f" 2>&1 && echo "✓ valid JSON: $f" || echo "✗ invalid JSON: $f"
done
```

### Final report to user

Output a summary table:

```
Claude Code Setup Complete
==========================
Project type : [regular | plugin]

Created      : list of newly created files/dirs
Configured   : list of files with content written
Needs review : list of pre-existing files that may need manual attention
Skipped      : list of files that already matched spec

Sub-skills invoked: [list any that were triggered]

Next steps:
- Review CLAUDE.md sections marked [FILL IN]
- Add real tool patterns to settings.json allowedTools
- Set CLAUDE.md build/test commands for your stack
- Restart Claude Code to load new hooks/MCP config
```

---

## Sub-skill delegation map

| Task | Skill to invoke |
|------|----------------|
| plugin.json manifest, `${CLAUDE_PLUGIN_ROOT}`, auto-discovery | **plugin-structure** |
| agent frontmatter, `<example>` blocks, model/color/tools | **agent-development** |
| hooks.json format, 9 event types, prompt vs command hooks | **hook-development** |
| .mcp.json, transport types, tool naming `mcp__server__*` | **mcp-integration** |
| slash commands, YAML frontmatter, `$ARGUMENTS` | **command-development** |
| SKILL.md anatomy, progressive disclosure, references/ | **skill-development** |

Invoke the relevant sub-skill whenever the user asks a deeper question about that component, or when generating non-trivial content for it.

---

## Key spec rules (quick reference)

- All names: kebab-case throughout
- CLAUDE.md: under 200 lines, all 5 sections required
- settings.json `allowedTools` patterns: `"Read"`, `"Edit"`, `"Write"`, `"Bash(git:*)"`, `"mcp__server__*"`
- Plugin internal paths: always `${CLAUDE_PLUGIN_ROOT}`, never absolute
- settings.local.json: MUST be gitignored (personal overrides)
- Agent descriptions: MUST have `<example>` blocks or they won't trigger
- hooks.json (plugin): wrap events inside `{"hooks": {…}}` — different from settings.json format
- .worktreeinclude: lists gitignored files that should still be copied to worktrees
