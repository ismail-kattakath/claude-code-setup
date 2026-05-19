# Claude Code Spec Compliance Checklist

Use this checklist when auditing any project. Mark each item: ✓ present, ✗ missing, ~ needs review.

---

## 1. CLAUDE.md

| Check | Notes |
|-------|-------|
| File exists at project root | Required |
| Under 200 lines | Hard limit |
| Has project overview section | What it does, primary stack |
| Has build/test/lint commands | Exact commands, not prose |
| Has architecture section | Key dirs and their purpose |
| Has conventions section | Naming, formatting, import style |
| Has important notes / gotchas | Non-obvious constraints |
| No sensitive data (tokens, passwords) | Should never be committed |

**Fail condition:** Missing any of the 5 required sections, or over 200 lines.

---

## 2. .claude/ Directory Structure

| Path | Required | Notes |
|------|----------|-------|
| `.claude/` | Yes | Base directory |
| `.claude/settings.json` | Yes | Permissions + hooks |
| `.claude/settings.local.json` | Yes (gitignored) | Personal overrides |
| `.claude/rules/` | Recommended | Scoped rules .md files (frontmatter: `globs`, `alwaysApply`) |
| `.claude/agents/` | If using agents | Subagent .md files |
| `.claude/skills/` | If using project skills | skill-name/SKILL.md pattern |
| `.claude/hooks/` | If using hooks | Hook scripts |
| `.claude/output-styles/` | If using output styles | Reusable output format definitions |

---

## 3. settings.json

| Check | Notes |
|-------|-------|
| Valid JSON | `jq empty .claude/settings.json` |
| Has `permissions` key | Object with `allow` and/or `deny` arrays |
| `permissions.allow` uses correct patterns | See patterns table below |
| Has `hooks` key | Object with event arrays |
| Has at least one Stop hook | Quality enforcement |
| `permission_mode` set (if explicit) | Valid: `"default"` `"dontAsk"` `"acceptEdits"` `"bypassPermissions"` `"plan"` `"auto"` |
| No hardcoded secrets | Env vars only |

**Allowed tool pattern syntax:**
```
"Read"                   # exact tool name
"Edit"                   # exact tool name
"Write"                  # exact tool name
"Bash(git:*)"            # tool with command prefix filter
"Bash(npm run *)"        # tool with command prefix filter
"mcp__server-name__*"    # all tools from an MCP server
"mcp__server__tool"      # specific MCP tool
```

**Hooks structure (settings format — no wrapper):**
```json
{
  "hooks": {
    "PreToolUse": [{"matcher": "Write|Edit", "hooks": [...]}],
    "PostToolUse": [{"matcher": "*", "hooks": [...]}],
    "Stop": [{"matcher": "*", "hooks": [...]}],
    "SubagentStop": [...],
    "SessionStart": [...],
    "SessionEnd": [...],
    "UserPromptSubmit": [...],
    "PreCompact": [...],
    "Notification": [...]
  }
}
```

---

## 4. .mcp.json

| Check | Notes |
|-------|-------|
| File exists at project root | Committed to git |
| Valid JSON | `jq empty .mcp.json` |
| `mcpServers` key present | Top-level object |
| All server names kebab-case | No underscores |
| stdio servers: `command` field set | Absolute or `${CLAUDE_PLUGIN_ROOT}` relative |
| http/sse servers: `url` field is HTTPS | Not HTTP |
| No hardcoded API keys or tokens | Use `env` field with `${VAR}` syntax |
| Environment variable names documented | In README or CLAUDE.md |

**Transport types:**
| Type | Required fields | Best for |
|------|----------------|----------|
| stdio (default) | `command`, `args` | Local processes, custom servers |
| http | `type: "http"`, `url` | Remote REST APIs, token auth |
| sse | `type: "sse"`, `url` | Hosted services, OAuth |
| ws | `type: "ws"`, `url` | Real-time, streaming |

---

## 5. .worktreeinclude

| Check | Notes |
|-------|-------|
| File exists at project root | Even if empty |
| Lists any gitignored files needed in worktrees | .env, local configs, etc. |
| One path per line | Relative to project root |

---

## 6. .gitignore Requirements

| Entry | Reason |
|-------|--------|
| `.claude/settings.local.json` | Personal overrides, never committed |
| `*.local.md` | Plugin local config files |
| `.claude/worktrees/` | Worktree scratch space |
| `.env` | Secrets |

---

## 7. Agents (if used)

**Location for regular project:** `.claude/agents/<name>.md`
**Location for plugin project:** `agents/<name>.md` at plugin root

| Check | Notes |
|-------|-------|
| File is kebab-case `.md` | e.g. `code-reviewer.md` |
| YAML frontmatter is valid | `---` delimiters |
| `name` field present | Kebab-case, 3–50 chars, no underscores |
| `description` field present | Contains triggering conditions |
| `description` has `<example>` blocks | REQUIRED for reliable triggering |
| Each `<example>` has context/user/assistant | Full structure |
| Each `<example>` has `<commentary>` | Explains why agent triggers |
| `model` field set | `inherit`, `sonnet`, `opus`, or `haiku` |
| `color` field set | One of: blue, cyan, green, yellow, magenta, red |
| `tools` field if restricting access | Array of tool names |
| System prompt under 10,000 chars | Hard limit |

---

## 8. Skills (if used)

**Location for regular project:** `.claude/skills/<name>/SKILL.md`
**Location for plugin project:** `skills/<name>/SKILL.md` at plugin root

| Check | Notes |
|-------|-------|
| Skill is in its own subdirectory | `skill-name/SKILL.md` pattern |
| SKILL.md has valid YAML frontmatter | `---` delimiters |
| `name` matches directory name | Kebab-case, 1–64 chars |
| `description` is imperative | "Use when..." triggers listed |
| SKILL.md body under 500 lines | Progressive disclosure |
| Large content in `references/` | Loaded on demand |
| References explicitly called out | "Read references/foo.md when X" |

---

## 9. Plugin-Specific Requirements

**Only check if `.claude-plugin/plugin.json` exists.**

### plugin.json

| Check | Notes |
|-------|-------|
| Located at `.claude-plugin/plugin.json` | NOT at plugin root directly |
| Valid JSON | `jq empty .claude-plugin/plugin.json` |
| `name` field present | Kebab-case, required minimum |
| `version` follows semver | Recommended: `"1.0.0"` |
| All custom paths use `${CLAUDE_PLUGIN_ROOT}` | Never absolute paths |
| Custom paths start with `./` in manifest | Relative to plugin root |
| No hardcoded absolute paths | Security and portability |

### hooks/hooks.json (plugin format)

| Check | Notes |
|-------|-------|
| Located at `hooks/hooks.json` | Not `.claude/` for plugins |
| Valid JSON | `jq empty hooks/hooks.json` |
| Has `"hooks"` wrapper key | DIFFERENT from settings.json format |
| Events nested inside `"hooks"` | `{"hooks": {"PreToolUse": [...]}}` |
| All hook commands use `${CLAUDE_PLUGIN_ROOT}` | Portable paths |
| All `"type"` values are valid | `"prompt"` or `"command"` |
| Matcher patterns are correct | Exact name, `Name|Other`, `*`, or regex |
| Timeouts set appropriately | Command: ≤60s, Prompt: ≤30s |

### Plugin directory structure

| Path | Required | Notes |
|------|----------|-------|
| `.claude-plugin/plugin.json` | Yes | Manifest, must be in this dir |
| `commands/` | If using slash commands | .md files, auto-discovered |
| `agents/` | If using agents | .md files, auto-discovered |
| `skills/<name>/SKILL.md` | If using skills | Auto-discovered |
| `hooks/hooks.json` | If using hooks | Plugin hooks format |
| `.mcp.json` | If using MCP | Committed to git |

**Auto-discovery rules:**
- All `.md` files in `commands/` → slash commands
- All `.md` files in `agents/` → agents
- All `SKILL.md` files in skill subdirs → skills
- Custom paths in `plugin.json` SUPPLEMENT (not replace) defaults

---

## 10. Commands (if used)

**Location:** `commands/` for plugins, or `.claude/commands/` for regular projects.

| Check | Notes |
|-------|-------|
| File is kebab-case `.md` | e.g. `run-tests.md` |
| Has YAML frontmatter with `description` | Required |
| `$ARGUMENTS` used for dynamic input | When command takes user args |
| `allowed-tools` scoped to needed tools | Least privilege |
| No hardcoded credentials in body | Dynamic via bash or env |

---

## Audit Summary Template

```
Project: [name]
Type: [regular | plugin]
Audited: [date]

PASS  (N items)
------
✓ [item]

FAIL  (N items)
------
✗ [item] — [reason]

NEEDS REVIEW  (N items)
------
~ [item] — [reason]
```
