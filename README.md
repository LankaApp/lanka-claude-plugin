# Lanka plugin for Claude Code

Build and edit Lanka chatbot flows from Claude Code. The plugin reads a bot's flows through Lanka's API and writes **drafts** — the owner reviews and publishes them in the Lanka editor. Nothing reaches visitors without that click.

## Setup

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

3. In Claude Code:

   ```
   /plugin marketplace add LankaApp/lanka-claude-plugin
   /plugin install lanka@lanka
   ```

   (or, from a checkout: `claude --plugin-dir /path/to/lanka-claude-plugin`)

## What is inside

- `skills/flows/SKILL.md` — the workflow: `/lanka:flows <session>` ties the chat to an open editor page (the Claude chip shows the name) → export → author → validate → draft → the owner publishes.
- `skills/flows/references/` — the `lanka-flows/v1` format, every node type and its content, the safe-expr script language, hosted Mini Apps, plugins and their `app.*` data.

## Development

```bash
claude --plugin-dir .
claude plugin validate .
```

`skills/flows/references/generated.json` is produced by Lanka's code — node types, slots, enums, script functions. Regenerate it from a Lanka checkout beside this one after a node type, slot, operator or function changes (Lanka's `spec/claude_plugin_spec.rb` fails when the file here is stale):

```bash
cd ../lanka && bin/rails lanka:skill_docs   # writes ../lanka-claude-plugin/skills/flows/references/generated.json
```
