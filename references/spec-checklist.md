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
| `.claude/rules/` | Recommended | Scoped rules .md files (frontmatter: `globs`, `alwaysApply`, or `paths`) |
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
| `$schema` field present (recommended) | `"https://json.schemastore.org/claude-code-settings.json"` |
| `permissions.defaultMode` set (if explicit) | Valid: `"default"` `"dontAsk"` `"acceptEdits"` `"bypassPermissions"` `"plan"` `"auto"` |
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

**All 29+ hook event types:**
| Group | Events |
|-------|--------|
| Session | `SessionStart`, `Setup`, `SessionEnd` |
| Per-turn | `UserPromptSubmit`, `UserPromptExpansion`, `Stop`, `StopFailure` |
| Tool | `PreToolUse`, `PermissionRequest`, `PermissionDenied`, `PostToolUse`, `PostToolUseFailure`, `PostToolBatch` |
| Subagent | `SubagentStart`, `SubagentStop`, `TaskCreated`, `TaskCompleted`, `TeammateIdle` |
| Memory/config | `InstructionsLoaded`, `ConfigChange`, `CwdChanged`, `FileChanged` |
| Worktree | `WorktreeCreate`, `WorktreeRemove` |
| Compaction | `PreCompact`, `PostCompact` |
| MCP | `Elicitation`, `ElicitationResult` |
| Notifications | `Notification` |

**All 5 hook handler types:**
| Type | Description |
|------|-------------|
| `command` | Run a shell command; stdout controls approve/block |
| `http` | POST to an HTTP endpoint |
| `mcp_tool` | Call an MCP tool |
| `prompt` | Send a prompt to Claude for a decision |
| `agent` | Invoke a named agent |

**Permissions — `Agent(...)` syntax (v2.1.63+):**
- Use `"Agent(agent-name)"` in `permissions.allow` to permit delegating to a specific agent
- Previously called `"Task(agent-name)"` — `Task(...)` is the old name, now renamed to `Agent(...)`

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
| http | `type: "http"`, `url` | Remote REST APIs, token auth (also accepted as `streamable-http`) |
| sse | `type: "sse"`, `url` | **DEPRECATED** — migrate to `http` |
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
| `description` field present | Clearly states when to delegate — specific trigger conditions and task scope |
| System prompt under 10,000 chars | Hard limit |

**Full set of supported agent frontmatter fields:**

| Field | Required | Notes |
|-------|----------|-------|
| `name` | Yes | Kebab-case, 3–50 chars, no underscores |
| `description` | Yes | When to delegate; specific trigger conditions |
| `model` | No | `sonnet`, `opus`, `haiku`, full model ID, or `inherit` |
| `color` | No | `red`, `blue`, `green`, `yellow`, `purple`, `orange`, `pink`, `cyan` |
| `tools` | No | Tool allowlist (inherits all if omitted) |
| `disallowedTools` | No | Tool denylist |
| `permissionMode` | No | Same values as `defaultMode`; overrides session mode for this agent |
| `maxTurns` | No | Max agentic turns (integer) |
| `skills` | No | Array of preloaded skill names |
| `memory` | No | `user`, `project`, or `local` |
| `background` | No | `true` to always run as background task |
| `isolation` | No | Only valid value: `"worktree"` |
| `effort` | No | `low`, `medium`, `high`, `xhigh`, `max` |
| `mcpServers` | No | Inline or named MCP servers for this agent |
| `hooks` | No | Hooks scoped to this agent only |
| `initialPrompt` | No | Auto-submitted as the agent's first turn |

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

**Claude Code-specific SKILL.md frontmatter fields (beyond agentskills.io base spec):**

| Field | Notes |
|-------|-------|
| `when_to_use` | Extra trigger context shown to users |
| `paths` | Auto-load this skill when Claude reads matching files (glob patterns) |
| `disable-model-invocation: true` | User-only trigger — Claude will not auto-invoke |
| `context: fork` | Run skill in a forked subagent context |
| `agent` | Which subagent to use when `context: fork` is set |

**Rules file (`paths:` frontmatter):**
Rules in `.claude/rules/` support a `paths:` frontmatter key for file-scoped loading:
- Without `paths:` → loads at session start (like CLAUDE.md)
- With `paths:` → loads only when Claude reads a file matching the glob patterns

```markdown
---
paths:
  - "**/*.test.ts"
---
# Testing Rules
```

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
| Persistent data uses `${CLAUDE_PLUGIN_DATA}` | Survives plugin updates; resolves to `~/.claude/plugins/data/{plugin-id}/` |

### hooks/hooks.json (plugin format)

| Check | Notes |
|-------|-------|
| Located at `hooks/hooks.json` | Not `.claude/` for plugins |
| Valid JSON | `jq empty hooks/hooks.json` |
| Has `"hooks"` wrapper key | DIFFERENT from settings.json format |
| Events nested inside `"hooks"` | `{"hooks": {"PreToolUse": [...]}}` |
| All hook commands use `${CLAUDE_PLUGIN_ROOT}` | Portable paths |
| All `"type"` values are valid | `"command"`, `"http"`, `"mcp_tool"`, `"prompt"`, or `"agent"` |
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

## 11. Output Styles (if used)

**Location:** `.claude/output-styles/<name>.md`

| Check | Notes |
|-------|-------|
| File is kebab-case `.md` | e.g. `terse.md` |
| Has YAML frontmatter with `name` and `description` | Required |
| `keep-coding-instructions` set | `true` to preserve coding instructions alongside this style |
| `force-for-plugin` set (plugin only) | `true` to always apply this style for the plugin |

**Frontmatter fields:**
| Field | Type | Notes |
|-------|------|-------|
| `name` | string | Display name shown in `/config` |
| `description` | string | Short description of the style |
| `keep-coding-instructions` | bool | Preserve coding instructions when this style is active |
| `force-for-plugin` | bool | Plugin only — always apply this style |

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
