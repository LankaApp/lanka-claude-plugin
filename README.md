# Lanka plugin for Claude Code

Build and edit [Lanka](https://github.com/LankaApp/lanka) chatbot flows from Claude Code. The plugin reads a bot's flows through Lanka's API and writes **drafts** — the owner reviews and publishes them in the Lanka editor. Nothing reaches visitors without that click.

## Setup

1. In Lanka, open **Integrations** (user menu → Інтеграції) and create an API key.
2. Save it where the plugin reads it:

   ```bash
   mkdir -p ~/.lanka && cat > ~/.lanka/config <<EOF
   LANKA_API_URL=https://your-lanka-host
   LANKA_API_TOKEN=your-key
   EOF
   chmod 600 ~/.lanka/config
   ```

3. In Claude Code:

   ```
   /plugin marketplace add LankaApp/lanka-claude-plugin
   /plugin install lanka@lanka
   ```

Then ask: *«збери ланку запису на стрижку в боті Barber»* — Claude lists your bots, reads what is there, writes a draft and hands you the editor link.

## What is inside

- `skills/lanka-flows/SKILL.md` — the workflow: list → export → author → validate → draft → publish in the editor.
- `skills/lanka-flows/references/` — the `lanka-flows/v1` format, every node type and its content, the safe-expr script language, hosted Mini Apps, plugins and their `app.*` data, worked examples.

## Development

```bash
claude --plugin-dir /path/to/lanka-claude-plugin
claude plugin validate .
```

`skills/lanka-flows/references/generated.json` comes from Lanka's code — regenerate it after a node type, slot, operator or script function changes:

```bash
cd ../lanka && bin/rails lanka:skill_docs > ../lanka-claude-plugin/skills/lanka-flows/references/generated.json
```
