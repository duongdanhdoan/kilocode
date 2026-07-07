# Kilo Custom Proxy Agent (Agentic)

Personal fork of [Kilo Code](https://github.com/Kilo-Org/kilocode) configured as a **5-role agentic
pipeline** that mixes providers per role (Claude via an OpenAI-compatible proxy, Qwen via
DashScope), with human approval gates between phases. Runs inside any VS Code-family IDE
(VS Code, Cursor, Antigravity, ...) via the Kilo extension, or headless via the `kilo` CLI.

Config is **global** (`~/.config/kilo/`) — it applies to every project on the machine, not just
this checkout. This checkout is only the source you build the extension/CLI from.

> **Security note:** this file intentionally contains **no real API keys or proxy domains**.
> Everywhere you see `<...>` below, substitute your own values. Never commit real secrets into
> this repo or into any file tracked by git — the working config lives outside the repo, at
> `~/.config/kilo/`.

## 1. Build the fork

```bash
git clone https://github.com/Kilo-Org/kilocode.git
cd kilocode
bun install
```

Requirements: Bun 1.3.14+. Java 21 only if you touch the JetBrains plugin (`sdk install java 21-tem`).

Sanity check:

```bash
bun run lint
bun turbo typecheck --filter='!@kilocode/kilo-jetbrains'
```

## 2. Global config (`~/.config/kilo/`)

Everything below goes in your **global** Kilo config directory, not the project root. Kilo
auto-discovers this directory (XDG config path) for every session, in every project.

### 2.1 `~/.config/kilo/kilo.jsonc` — providers, permissions, MCP servers

```jsonc
{
  "$schema": "https://app.kilo.ai/config.json",
  "provider": {
    // Built-in DashScope/Qwen provider — just needs your key.
    "alibaba": {
      "options": {
        "apiKey": "<YOUR_DASHSCOPE_API_KEY>",
      },
    },
    // Any OpenAI-Chat-Completions-compatible proxy/gateway that serves Claude/GPT models
    // under one key. Replace baseURL/apiKey/model IDs with your own proxy's values.
    "my-proxy": {
      "name": "My Proxy",
      "npm": "@ai-sdk/openai-compatible",
      "options": {
        "baseURL": "<YOUR_PROXY_BASE_URL>", // e.g. https://your-proxy.example.com/v1
        "apiKey": "<YOUR_PROXY_API_KEY>",
      },
      "models": {
        "claude-sonnet-4-6": {
          "name": "Claude Sonnet 4.6 (proxy)",
          "limit": { "context": 200000, "output": 32000 },
        },
        // ...add whatever model IDs your proxy exposes (check `kilo models <provider>`)
      },
    },
  },
  "permission": {
    "bash": "allow", // or "ask" if you want per-command approval globally
  },
  // Extra instruction files ALWAYS attached to every session's system prompt.
  "instructions": ["~/.claude/agent_rules.yaml"], // optional, if you keep rules elsewhere
  // MCP servers available to every agent (subject to each agent's own permission rules).
  "mcp": {
    "my-local-mcp": {
      "type": "local",
      "command": ["/path/to/binary", "arg1"],
    },
    "my-remote-mcp": {
      "type": "remote",
      "url": "https://mcp.example.com/mcp", // OAuth sign-in via `kilo mcp auth <name>`
    },
  },
}
```

Verify:

```bash
kilo models my-proxy
kilo models alibaba
kilo mcp list
```

### 2.2 `~/.config/kilo/AGENTS.md` — rules applied to every agent, every project

This file is auto-loaded into the system prompt of **every** Kilo session (any agent, any
project) — it's the global rules file, equivalent to a global `CLAUDE.md`.

```markdown
# Global Kilo Rules

## <your convention name>

<plain-language rule text — Kilo agents follow it like any other instruction>
```

Example used in this setup — a branch-naming convention keyed off a ticket ID pattern:

```markdown
## Branch checkout convention

When asked to checkout/create a branch from a ticket (e.g. "checkout PROJ-1234"):

1. Look up the ticket via your issue-tracker MCP tool to get its type and summary.
2. Derive a short `aim_name`: a few kebab-case words from the summary.
3. Pick the prefix by type: `feature/PROJ-xxx_aim_name`, `bug/PROJ-xxx_aim_name`.
4. Parent/epic tickets: branch off the specific SUBTASK's ID, not the parent's.
5. `git checkout -b <branch-name>`.
```

Kilo also auto-loads `~/.claude/CLAUDE.md` by default (same file Claude Code reads) — so any
rules you already keep there for Claude Code apply to Kilo too, for free.

### 2.3 `~/.config/kilo/agent/*.md` — custom agents/roles

Each file = one custom agent. Frontmatter binds a model + permissions; the markdown body is
the system prompt. Example role (`~/.config/kilo/agent/reviewer.md`):

```markdown
---
description: "Critically review a plan before code is written"
mode: subagent          # subagent = only invoked via the `task` tool; primary = user-selectable
model: my-proxy/claude-opus-4-8
permission:
  read: allow
  glob: allow
  grep: allow
  edit: deny
  bash: deny
  task: deny
  "my-local-mcp_*": deny   # deny heavy/irrelevant MCP tools per-role to save tokens
---

<system prompt body — what this role does, what it must output>
```

An **orchestrator** (`mode: primary`) can sequence several `subagent` roles via the `task`
tool and gate transitions on human approval, with zero core-code changes:

```yaml
permission:
  task:
    "search-analyst": allow      # runs immediately
    "solution-architect": ask    # pauses for human approval before this phase
    "coder": allow
    "final-checker": ask         # pauses again before the last phase
```

`ask` on a `task` permission pattern blocks that specific subagent spawn until a human
approves (via the IDE's permission dock, or `kilo mcp`/HTTP reply in headless mode). A denied
approval aborts that turn (not a soft error the agent can recover from in the same turn) — the
orchestrator's own prompt should tell it to report and stop, not retry.

Verify:

```bash
kilo agent list
kilo run "Use the task tool to spawn subagent_type=<your-role> with prompt '...'"
```

## 3. Project-local overrides (optional)

Everything in step 2 can ALSO be set per-project instead of globally, by creating the same
file layout under `<project>/.kilocode/` (or `.kilo/`) instead of `~/.config/kilo/`:

```
<project>/.kilocode/
├── kilo.jsonc
├── AGENTS.md
└── agent/*.md
```

Project-local config merges with (and can override) global config. **Don't duplicate the
same provider/agent in both places** — pick one source of truth per setting, or you'll get
stale-value drift (a project-local copy silently missing a field you later added globally).
This setup keeps everything global on purpose, for zero-setup use across projects.

`CLAUDE.local.md` at a project root is NOT auto-loaded by Kilo's native file list — it's only
picked up if your global/project `CLAUDE.md`/`AGENTS.md` explicitly instructs the agent to
check for it (Claude Code's own convention does this).

## 4. Install into your IDE

```bash
cd packages/kilo-vscode
bun run package
npm_config_ignore_scripts=true node_modules/.bin/vsce package \
  --no-dependencies --skip-license --target darwin-arm64 -o out/kilo-vscode-darwin-arm64.vsix

# VS Code
code --install-extension out/kilo-vscode-darwin-arm64.vsix
# Cursor
cursor --install-extension out/kilo-vscode-darwin-arm64.vsix
# Antigravity IDE (macOS)
"/Applications/Antigravity IDE.app/Contents/Resources/app/bin/antigravity-ide" \
  --install-extension out/kilo-vscode-darwin-arm64.vsix
```

Reload the IDE window after installing. Re-run this step any time `packages/kilo-vscode/`
source changes — global config changes (`~/.config/kilo/`) take effect immediately, no
rebuild needed.

## 5. Headless / CLI usage

```bash
kilo run --agent <primary-agent-name> "<task>"          # foreground, prompts for approval
kilo run --agent <primary-agent-name> --auto "<task>"   # auto-approve every gate (testing only)
kilo mcp list                                            # check MCP server connectivity
kilo session list / kilo export <sessionID>              # inspect what an agent actually did
```
