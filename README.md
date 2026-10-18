# Lanka plugin for Claude Code

Build and edit Lanka chatbot flows from Claude Code. The plugin reads a bot's flows through Lanka's MCP server and writes **drafts** — the owner reviews and publishes them in the Lanka editor. Nothing reaches visitors without that click.

## Setup

1. Install the plugin. In Claude Code in a terminal (`claude`, also inside VS Code or JetBrains):

   ```
   /plugin marketplace add LankaApp/lanka-claude-plugin
   /plugin install lanka@lanka
   ```

   In the Claude desktop app, Code tab: **+ → Plugins → Add plugin**, enter `LankaApp/lanka-claude-plugin`, then install `lanka`.

   (or, from a checkout: `claude --plugin-dir /path/to/lanka-claude-plugin`)

2. In a new session run `/lanka:flows <session>` with the name the editor's Claude chip shows (any name does for the first time). The first call to Lanka opens the OAuth consent page in your browser: sign in to Lanka and allow the `bots` scope once. Claude Code keeps the token and refreshes it by itself; **Clear authentication** in `/mcp` revokes it, and so does Lanka's Integrations page.

That is all: the plugin declares Lanka's MCP server (`https://lanka.bot/mcp`, in [`.mcp.json`](.mcp.json)), and the skill works through its tools.

### Without MCP

Where the `lanka` MCP server cannot run (an older Claude Code, or the server turned off in `/mcp`), the skill falls back to the REST API with a permanent key:

1. In Lanka, open **Integrations** (user menu) and create an API key.
2. Save it where the plugin reads it (Integrations shows these commands with your key filled in). macOS and Linux:

   ```bash
   mkdir -p ~/.lanka && cat > ~/.lanka/config <<EOF
   LANKA_API_URL=https://lanka.bot
   LANKA_API_TOKEN=your-key
   EOF
   chmod 600 ~/.lanka/config
   ```

   Windows, in PowerShell. Claude Code runs the plugin's commands in Git Bash, so the file needs LF line ends — a CR would stick to the values:

   ```powershell
   $config = "LANKA_API_URL=https://lanka.bot`nLANKA_API_TOKEN=your-key`n"
   New-Item -ItemType Directory -Force "$HOME\.lanka" | Out-Null
   [IO.File]::WriteAllText("$HOME\.lanka\config", $config)
   ```

The fallback works only where Claude Code runs on your computer: it reads `~/.lanka/config` and calls the Lanka API with curl. The MCP path works anywhere Claude Code can open a browser for the consent.

## What is inside

- `.mcp.json` — Lanka's remote MCP server, `lanka`, whose tools mirror the REST API (`get_context`, `export_flows`, `validate_flows`, `write_draft`, …). Claude Code handles the OAuth sign-in.
- `skills/flows/SKILL.md` — the workflow: `/lanka:flows <session>` ties the chat to an open editor page (the Claude chip shows the name) → export → author → validate → draft → the owner publishes.
- `skills/flows/references/` — the `lanka-flows/v1` format, every node type and its content, the safe-expr script language, hosted Mini Apps, `app.*` data and panels.
- `chatgpt/` — the same skill as a Custom GPT for ChatGPT: instructions, the Actions schema (OAuth) and how to set it up.

## Development

```bash
claude --plugin-dir .
claude plugin validate .
```

**Generated files — do not edit by hand.** `skills/flows/references/generated.json` (node types, slots, enums, script functions), `references/format.md` and `references/data.md` are written by Lanka's code; the MCP server's `guide` tool serves the same text. Regenerate them from a Lanka checkout beside this one after a node type, slot, operator, function or format change (Lanka's `spec/claude_plugin_spec.rb` fails when the files here are stale):

```bash
cd ../lanka && bin/rails lanka:skill_docs   # writes skills/flows/references/{generated.json,format.md,data.md} in ../lanka-claude-plugin
```

Every PR bumps `version` in `.claude-plugin/plugin.json`; installed copies pick the update up with `claude plugin marketplace update lanka && claude plugin update lanka@lanka` and a new session.
