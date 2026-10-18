# Lanka flows in ChatGPT

A Custom GPT that does what the Claude Code skill does: reads a bot's flows and writes drafts you publish in the Lanka editor. It calls the same Lanka API with the same API key.

Needs a ChatGPT plan that can create GPTs.

## Set up

1. In Lanka: user menu → **Integrations** → create an API key. Copy it.
2. In ChatGPT: **GPTs → Create → Configure**.
   - **Name**: Lanka flows (any).
   - **Instructions**: paste [`instructions.md`](instructions.md).
   - **Knowledge**: upload the files from [`../skills/flows/references/`](../skills/flows/references/) — `format.md`, `node-types.md`, `safe-expr.md`, `plugins.md`, `mini-apps.md`, `generated.json`.
3. **Actions → Create new action**.
   - **Authentication**: API Key, auth type **Bearer**, paste the key.
   - **Schema**: paste [`openapi.yaml`](openapi.yaml).
4. Save it as **Only me**.

In a chat, name the editor session from the Claude chip in the editor: «тихий-кіт — додай кнопку…».

## Keep it to yourself

The API key lives inside the GPT. Anyone you share the GPT with works on **your** bots with it. Each owner makes their own GPT with their own key.

## After a Lanka update

When the skill's references change, upload them to Knowledge again; when the API changes, paste `openapi.yaml` again.
