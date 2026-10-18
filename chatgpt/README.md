# Lanka flows in ChatGPT

A Custom GPT that does what the Claude Code skill does: reads a bot's flows and writes drafts you publish in the Lanka editor. It calls the same Lanka API, signed in with your Lanka account through OAuth — no API key to paste.

Needs a ChatGPT plan that can create GPTs.

## Set up

1. In ChatGPT: **GPTs → Create → Configure**.
   - **Name**: Lanka flows (any).
   - **Instructions**: paste [`instructions.md`](instructions.md).
   - **Knowledge**: upload the files from [`../skills/flows/references/`](../skills/flows/references/) — `format.md`, `node-types.md`, `safe-expr.md`, `data.md`, `mini-apps.md`, `generated.json`.
2. **Actions → Create new action**.
   - **Schema**: paste [`openapi.yaml`](openapi.yaml).
   - **Authentication**: **OAuth**. The schema already carries the authorization URL (`https://lanka.bot/oauth/authorize`), the token URL (`https://lanka.bot/oauth/token`) and the scope `bots`; the form asks for them again — enter the same, token exchange method **Default (POST request)**.
   - **Client ID / Client Secret**: to be created in Lanka → **Integrations → Connected apps** (this GPT as a connected app, with the callback URL from the next step). Until that page exists, register the GPT by hand and use the returned `client_id` and `client_secret`:

     ```bash
     curl -sS -X POST https://lanka.bot/oauth/register -H "Content-Type: application/json" \
       -d '{"client_name": "Lanka flows GPT", "redirect_uris": ["<the callback URL ChatGPT shows>"], "grant_types": ["authorization_code", "refresh_token"], "token_endpoint_auth_method": "client_secret_post", "scope": "bots"}'
     ```

   - **Callback URL**: after saving the action, ChatGPT shows the GPT's callback URL (`https://chat.openai.com/aip/<id>/oauth/callback`). It must be in Lanka's registration of the GPT: paste it into the connected app, or into `redirect_uris` above. ChatGPT changes it when the GPT is re-created, not on edits.
3. Save and publish as **Anyone with the link** (or **Only me**).

The first chat that calls Lanka opens the sign-in: ChatGPT sends you to Lanka, you allow the `bots` scope, and the GPT works on **your** bots from then on. Each person who uses the GPT signs in with their own Lanka account and sees only their bots; the access token lives an hour and ChatGPT refreshes it by itself. Revoke the GPT's access on Lanka's Integrations page.

In a chat, name the editor session from the Claude chip in the editor: «тихий-кіт — додай кнопку…».

## After a Lanka update

When the skill's references change, upload them to Knowledge again; when the API changes, paste `openapi.yaml` again. The GPT tells you at the start of a chat when the `generated.json` in Knowledge is older than what Lanka runs.
