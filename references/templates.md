# Claude Code Setup Templates

Copy-paste ready templates for all required files. Replace `[PLACEHOLDERS]` with project-specific values.

---

## Template 1: Minimal settings.json

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "permissions": {
    "allow": [
      "Read",
      "Edit",
      "Write",
      "Bash(git:*)",
      "Bash(npm run *)",
      "Bash(npx *)"
    ],
    "deny": [],
    "defaultMode": "default"
  },
  "hooks": {
    "Stop": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "prompt",
            "prompt": "Verify task completion before stopping. Check: were all requested changes made? Did tests pass? Were any open questions answered? If anything is incomplete, return 'block' with a specific reason. Otherwise return 'approve'."
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "prompt",
            "prompt": "Check if this file write is safe: no system paths, no credentials, no path traversal (../ sequences). Return 'approve' if safe, 'deny' with reason if not."
          }
        ]
      }
    ]
  }
}
```

**Notes:**
- Add `"Bash(pytest *)"`, `"Bash(cargo *)"`, `"Bash(go test *)"` etc. as needed for your stack.
- Add `"mcp__server-name__*"` entries for any MCP servers in `.mcp.json`.
- `permissions.defaultMode` valid values: `"default"` (interactive), `"dontAsk"` (auto-approve all), `"acceptEdits"` (auto-approve edits), `"bypassPermissions"` (no checks — dangerous), `"plan"` (read-only), `"auto"` (smart). Omit to use `"default"`.

---

## Template 2: CLAUDE.md

```markdown
# [Project Name]

[One-paragraph description: what it does, primary language/framework, who uses it.]

## Build / Test / Lint

```bash
# Install dependencies
[npm install | pip install -e ".[dev]" | cargo build | go mod tidy]

# Run tests
[npm test | pytest | cargo test | go test ./...]

# Lint / format
[npm run lint | ruff check . | cargo clippy | golangci-lint run]

# Build
[npm run build | python -m build | cargo build --release]
```

## Architecture

- `src/` — [description]
- `tests/` — [description]
- `[key-dir]/` — [description]
- `[key-dir]/` — [description]
- `CLAUDE.md` — project instructions for Claude Code
- `.claude/` — Claude Code settings, hooks, agents

## Conventions

- **Naming:** [kebab-case files, PascalCase classes, snake_case functions — pick one]
- **Imports:** [relative vs absolute, import order]
- **Formatting:** [tab size, max line length, auto-formatter used]
- **Commits:** [conventional commits? branch naming?]
- **Tests:** [where they live, naming pattern, required coverage]

## Important Notes

- [Gotcha 1: e.g. "Always run migrations before running tests"]
- [Gotcha 2: e.g. "The API client is a singleton — don't instantiate directly"]
- [Gotcha 3: e.g. "Feature flags live in config/flags.yaml, not environment variables"]
```

**Rules:** Under 200 lines. All five sections required. No tokens/passwords.

---

## Template 3: plugin.json

### Minimal (required only)
```json
{
  "name": "plugin-name"
}
```

### Recommended (full metadata)
```json
{
  "name": "plugin-name",
  "version": "1.0.0",
  "description": "One-sentence description of what this plugin does.",
  "author": {
    "name": "Author Name",
    "email": "author@example.com"
  },
  "repository": "https://github.com/owner/plugin-name",
  "license": "MIT",
  "keywords": ["keyword1", "keyword2"]
}
```

### With custom component paths
```json
{
  "name": "plugin-name",
  "version": "1.0.0",
  "description": "...",
  "commands": "${CLAUDE_PLUGIN_ROOT}/commands",
  "agents": ["${CLAUDE_PLUGIN_ROOT}/agents", "${CLAUDE_PLUGIN_ROOT}/specialized-agents"],
  "hooks": "${CLAUDE_PLUGIN_ROOT}/hooks/hooks.json",
  "mcpServers": "${CLAUDE_PLUGIN_ROOT}/.mcp.json"
}
```

**Rules:**
- `name` must be kebab-case, unique across installed plugins.
- All path values MUST use `${CLAUDE_PLUGIN_ROOT}` — never hardcode absolute paths.
- Custom paths supplement (not replace) auto-discovery in default directories.
- `agents` field accepts a string OR an array of strings.
- For persistent data that survives plugin updates, use `${CLAUDE_PLUGIN_DATA}` (resolves to `~/.claude/plugins/data/{plugin-id}/`). `${CLAUDE_PLUGIN_ROOT}` changes on update; `${CLAUDE_PLUGIN_DATA}` is stable.

---

## Template 4: Agent frontmatter (.md file)

```markdown
---
name: agent-name
description: "Use this agent when [specific triggering conditions — be precise about what tasks, files, or situations should trigger delegation to this agent]."
# --- optional fields below ---
model: inherit              # sonnet | opus | haiku | full-model-id | inherit
color: blue                 # red | blue | green | yellow | purple | orange | pink | cyan
tools: ["Read", "Grep", "Bash"]   # omit for full tool access
# disallowedTools: ["Write"]       # deny specific tools
# permissionMode: acceptEdits      # default | acceptEdits | plan | auto | dontAsk | bypassPermissions
# maxTurns: 20                     # max agentic turns
# skills:                          # preloaded skills
#   - api-conventions
# memory: project                  # user | project | local
# background: false                # true to always run as background task
# isolation: worktree              # ONLY valid value is "worktree"
# effort: medium                   # low | medium | high | xhigh | max
# mcpServers:                      # inline or named MCP servers
#   - github
# hooks:                           # hooks scoped to this agent
#   PreToolUse: [...]
# initialPrompt: "..."             # auto-submitted as first turn
---

You are [role description] specializing in [domain].

**Core Responsibilities:**
1. [Primary responsibility]
2. [Secondary responsibility]
3. [Additional responsibilities]

**Process:**
1. [Step one]
2. [Step two]
3. [Step three]

**Output Format:**
[What to return, how to structure it]

**Edge Cases:**
- [Edge case]: [How to handle it]
```

**Field rules:**
- `name`: kebab-case, 3–50 chars, alphanumeric start/end, no underscores
- `description`: clearly state when to delegate — specific trigger conditions, task types, or file patterns. No `<example>` blocks required.
- `model`: use `inherit` unless the agent specifically needs `opus` (most capable) or `haiku` (fastest)
- `color`: blue/cyan = analysis; green = success tasks; yellow = validation; red = critical/security; purple = creative
- `tools`: omit for full access, or list minimum needed tools
- `isolation: worktree` is the only valid isolation value — runs the agent in an isolated git checkout

---

## Template 5: hooks.json (plugin format)

```json
{
  "description": "Hooks for [plugin-name]: validate writes, enforce quality on stop",
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "prompt",
            "prompt": "Validate this file write. Check for: path traversal, sensitive file paths (*.env, *credentials*, *secret*), system directories. Approve safe writes, deny unsafe ones with reason.",
            "timeout": 15
          }
        ]
      },
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "bash ${CLAUDE_PLUGIN_ROOT}/hooks/scripts/validate-bash.sh",
            "timeout": 10
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "prompt",
            "prompt": "Review the edit result. Check for: syntax errors, obvious logic bugs, accidental removals. Provide concise feedback or confirm looks good.",
            "timeout": 20
          }
        ]
      }
    ],
    "Stop": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "prompt",
            "prompt": "Before stopping, verify: (1) all requested changes were made, (2) tests were run if code changed, (3) no outstanding questions. Return JSON: {\"decision\": \"approve\"} or {\"decision\": \"block\", \"reason\": \"[what's missing]\"}",
            "timeout": 30
          }
        ]
      }
    ],
    "SubagentStop": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "prompt",
            "prompt": "Verify subagent completed its assigned task fully. Return {\"decision\": \"approve\"} if done, or {\"decision\": \"block\", \"reason\": \"...\"}",
            "timeout": 20
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "bash ${CLAUDE_PLUGIN_ROOT}/hooks/scripts/load-context.sh",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

**CRITICAL:** Plugin `hooks.json` MUST have the `"hooks"` wrapper key.
`settings.json` uses events directly at top level — no wrapper.

---

## Template 6: .mcp.json

```json
{
  "mcpServers": {
    "local-tool": {
      "command": "node",
      "args": ["${CLAUDE_PLUGIN_ROOT}/servers/local-tool.js"],
      "env": {
        "LOG_LEVEL": "info",
        "DATA_DIR": "${CLAUDE_PROJECT_DIR}/.data"
      }
    },
    "remote-api": {
      "type": "http",
      "url": "https://api.example.com/mcp",
      "headers": {
        "Authorization": "Bearer ${EXAMPLE_API_TOKEN}",
        "X-Client": "claude-code"
      }
    },
    "real-time-service": {
      "type": "ws",
      "url": "wss://stream.example.com/mcp"
    }
  }
}
```

> **Note:** The `sse` transport type is deprecated. Migrate existing SSE servers to `"type": "http"` (also accepted as `"type": "streamable-http"`).

**Transport selection guide:**
| Situation | Use |
|-----------|-----|
| Running a local Node/Python/Go binary | stdio (default, no `type` field needed) |
| REST API with token auth | `"type": "http"` (or `"streamable-http"`) |
| Hosted service with OAuth (Asana, GitHub, etc.) | `"type": "http"` — SSE is deprecated |
| Real-time/streaming requirements | `"type": "ws"` |

**Rules:**
- Server names: kebab-case
- stdio: use `${CLAUDE_PLUGIN_ROOT}` for paths inside plugin, `${CLAUDE_PROJECT_DIR}` for project-relative paths
- http/ws: always HTTPS/WSS, never HTTP/WS
- All secrets via `${ENV_VAR}` syntax in `env` or `headers` — never hardcoded
- File is committed to git; document all required env vars in CLAUDE.md

---

## Template 7: SKILL.md (for project skills)

```markdown
---
name: skill-name
description: "Use when [triggering conditions]. Triggers on: keyword1, keyword2, keyword3."
license: MIT
compatibility: "Designed for Claude Code."
metadata:
  author: [author]
  version: "1.0"
---

# [Skill Name]

[One paragraph: what this skill provides, what domain it covers.]

## When to Use

- [Trigger condition 1]
- [Trigger condition 2]
- [Trigger condition 3]

## Process

### Step 1 — [Name]

[Instructions]

### Step 2 — [Name]

[Instructions]

## Reference Files

Read `references/[file].md` when [specific condition].
Read `references/[other].md` when [other condition].
```

**Rules:**
- SKILL.md body under 500 lines (aim for under 200 for optimal performance)
- Heavy content goes in `references/` — only load what's needed
- Name must match directory name exactly
- Description must be imperative and list trigger keywords

---

## Template 8: Rules file (.claude/rules/your-topic.md)

**Variant A — session-scoped (loads at session start, like CLAUDE.md):**
```markdown
---
description: [When this rule applies — shown in /rules list]
globs:
  - "src/**/*.ts"
  - "src/**/*.tsx"
alwaysApply: false
---

[Rule content — plain markdown instructions applied to matching files]

Example: Always use named exports. Never use default exports in this project.
```

**Variant B — file-scoped (loads only when Claude reads a matching file):**
```markdown
---
paths:
  - "**/*.test.ts"
  - "**/*.spec.ts"
---
# Testing Rules

- Use `describe`/`it` blocks, not `test`
- Each test must have a descriptive name that explains the expected behaviour
- Always use `beforeEach` for setup, never duplicate setup in individual tests
```

**Field rules:**
- `globs` + `alwaysApply`: traditional session-scoped loading
- `paths`: file-scoped loading — rule loads only when Claude opens a file matching the glob
- Without `paths:` → loads at session start; with `paths:` → loads on file access
- Body: plain markdown — no special format, just instructions

---

## Template 9: .gitignore additions

Append these lines to your existing `.gitignore` (or create it if missing):

```gitignore
# Claude Code — personal overrides and worktree scratch
.claude/settings.local.json
*.local.md
.claude/worktrees/

# Secrets and local env
.env
.env.local
.env.*.local
```

---

## Template 10: Hook handler types (all 5)

Use these inside any `"hooks"` array in `settings.json` or `hooks/hooks.json`:

```json
[
  {
    "type": "command",
    "command": "bash /path/to/script.sh",
    "timeout": 10
  },
  {
    "type": "prompt",
    "prompt": "Review the action and return approve or block with reason.",
    "timeout": 20
  },
  {
    "type": "http",
    "url": "https://hooks.example.com/claude-event",
    "method": "POST",
    "headers": {
      "Authorization": "Bearer ${HOOK_TOKEN}"
    },
    "timeout": 10
  },
  {
    "type": "mcp_tool",
    "server": "my-mcp-server",
    "tool": "validate_action",
    "timeout": 15
  },
  {
    "type": "agent",
    "agent": "security-reviewer",
    "timeout": 30
  }
]
```

**Handler type summary:**
| Type | What it does |
|------|-------------|
| `command` | Runs a shell command; stdout `approve`/`block` controls flow |
| `prompt` | Sends a prompt to Claude for a decision |
| `http` | POSTs event data to an HTTP endpoint |
| `mcp_tool` | Calls a named MCP tool with the event context |
| `agent` | Delegates the decision to a named agent |

---

## Template 11: Output style (.claude/output-styles/your-style.md)

```markdown
---
name: Terse
description: Short, direct answers with no preamble or filler
keep-coding-instructions: true
force-for-plugin: false
---

Be brief. Lead with the answer. No preamble, no "Sure!", no summaries at the end.
Use bullet points for lists. Use code blocks for code. Omit pleasantries.
```

**Frontmatter fields:**
| Field | Type | Notes |
|-------|------|-------|
| `name` | string | Display name shown in `/config` |
| `description` | string | Short description of the style |
| `keep-coding-instructions` | bool | `true` to keep coding instructions active alongside this style |
| `force-for-plugin` | bool | Plugin only — `true` to always apply this style for the plugin |
