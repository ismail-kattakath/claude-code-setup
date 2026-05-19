# Claude Code Setup Templates

Copy-paste ready templates for all required files. Replace `[PLACEHOLDERS]` with project-specific values.

---

## Template 1: Minimal settings.json

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Edit",
      "Write",
      "Bash(git:*)",
      "Bash(npm run *)",
      "Bash(npx *)"
    ],
    "deny": []
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
  },
  "permission_mode": "ask"
}
```

**Notes:**
- Add `"Bash(pytest *)"`, `"Bash(cargo *)"`, `"Bash(go test *)"` etc. as needed for your stack.
- Add `"mcp__server-name__*"` entries for any MCP servers in `.mcp.json`.
- `permission_mode: "allow"` disables interactive prompts (use in trusted environments only).

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

---

## Template 4: Agent frontmatter (.md file)

```markdown
---
name: agent-name
description: "Use this agent when [specific triggering conditions]. Examples:

<example>
Context: [Scenario where this agent is needed]
user: \"[What the user says to trigger this agent]\"
assistant: \"[How Claude should respond, invoking this agent]\"
<commentary>
[Why this agent is the right choice here — what capability it provides]
</commentary>
</example>

<example>
Context: [A different scenario]
user: \"[Alternative phrasing that should trigger this agent]\"
assistant: \"[Response]\"
<commentary>
[Why this agent applies in this case too]
</commentary>
</example>"
model: inherit
color: blue
tools: ["Read", "Write", "Grep", "Bash"]
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
- `description`: MUST include `<example>` blocks — agents without examples rarely trigger
- `model`: use `inherit` unless the agent specifically needs `opus` (most capable) or `haiku` (fastest)
- `color`: blue/cyan = analysis; green = success tasks; yellow = validation; red = critical/security; magenta = creative
- `tools`: omit for full access, or list minimum needed tools

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
    "hosted-service": {
      "type": "sse",
      "url": "https://mcp.example.com/sse"
    }
  }
}
```

**Transport selection guide:**
| Situation | Use |
|-----------|-----|
| Running a local Node/Python/Go binary | stdio (default, no `type` field needed) |
| REST API with token auth | `"type": "http"` |
| Hosted service with OAuth (Asana, GitHub, etc.) | `"type": "sse"` |
| Real-time/streaming requirements | `"type": "ws"` |

**Rules:**
- Server names: kebab-case
- stdio: use `${CLAUDE_PLUGIN_ROOT}` for paths inside plugin, `${CLAUDE_PROJECT_DIR}` for project-relative paths
- http/sse/ws: always HTTPS/WSS, never HTTP/WS
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

## Template 8: .gitignore additions

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
