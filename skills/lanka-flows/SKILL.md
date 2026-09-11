---
name: lanka-flows
description: Author, read and edit Lanka chatbot flows (ланки) through the Lanka API. Use whenever the user wants to build, change, inspect or copy a bot's flows, nodes, buttons, scripts or Mini App steps in Lanka — including requests phrased in Ukrainian ("зроби ланку", "додай кнопку", "подивись бота").
---

# Lanka flows

You work on a Lanka bot through its HTTP API. You **read** flows as `lanka-flows/v1` documents and **write drafts**; the owner publishes in the Lanka editor. You never publish, and nothing you write reaches visitors until they do.

Answer in the user's language. Flows, buttons and messages are written in the language the bot speaks to its visitors (usually Ukrainian) unless told otherwise.

## 0. Setup check (every session, once)

```bash
source ~/.lanka/config && test -n "$LANKA_API_URL" -a -n "$LANKA_API_TOKEN" && echo ok
```

If the file is missing or empty, stop and tell the user: create an API key in Lanka (user menu → **Інтеграції**) and save it as `~/.lanka/config` with `LANKA_API_URL=…` and `LANKA_API_TOKEN=…` (`chmod 600`). Do not guess a host or a key.

Every call below is:

```bash
source ~/.lanka/config && curl -sS -H "Authorization: Bearer $LANKA_API_TOKEN" "$LANKA_API_URL/api/v1/…"
```

Responses are JSON. `401` — the key is wrong or revoked; `403` — the account is not approved yet; `404` — not this owner's bot or flow; `409` — a stale draft (see §3); `422` — the document was refused, `error` says why.

## 1. Find the bot and read it

```bash
GET /api/v1/bots                       # [{id, name, platform, username, url}]
GET /api/v1/bots/:bot_id/flows         # [{id, name, group, trigger_type, trigger_value, active, internal, draft_dirty, updated_at, editor_url}]
GET /api/v1/bots/:bot_id/flows/export             # the whole bot as one document
GET /api/v1/bots/:bot_id/flows/:flow_id/export    # one flow (+ the notification flows it fires)
```

Export answers `{document, warnings}`. **Read `warnings` and pass anything relevant to the user** — they name what the format could not carry (raw html, a reference to a flow outside the export).

In the document, every flow's **`root` is the tree to work on**: the editing draft while the owner has unpublished changes, the live tree otherwise. Next to a draft you also get `published` (what visitors see right now) and `draft_version`. Every flow and node carries its `id`.

When several bots exist and the request does not say which, ask. When a request mentions a flow by name or command, find it in the list first — do not invent ids.

## 2. Author or edit

Write a `lanka-flows/v1` document. The format is in [references/format.md](references/format.md); node types and their `content` keys in [references/node-types.md](references/node-types.md); scripts in [references/safe-expr.md](references/safe-expr.md); hosted Mini Apps in [references/mini-apps.md](references/mini-apps.md); plugins and their `app.*` data in [references/plugins.md](references/plugins.md). [references/generated.json](references/generated.json) is generated from Lanka's code — the exact node types, slots, enums and script functions; when a prose page and it disagree, it is right. Worked examples: [references/examples/](references/examples/).

Rules that are easy to get wrong:

- **Editing an existing flow: keep `id` on the flow and on every node you keep.** A node with its `id` keeps its identity when the owner publishes (buttons already sent to visitors keep working); a node without one is new. Drop the `id` only for nodes you replace on purpose. Keep `draft_version` as exported.
- **Small steps.** One change per write — the owner watches the editor. Do not rewrite a whole bot to add a button.
- A list of options is a **button** whose child is a `list_buttons` node (`options_var`, `save_to`) — there is no "choice" node. A conditional button is a `visibility` node standing first under the button; the button's kind (`url`, `web_app`, `share`, `list_buttons`) goes under it.
- `url`, `share`, `web_app`, `list_buttons` exist only as a child of a button. A reply keyboard is the `reply_keyboard` node; it cannot carry url buttons.
- HTTP inside a flow is `fetch(...)` in a `script` node — there is no api_call node. A fetch result needs a home: `temp.r = fetch(...)` or `session.r = fetch(...)`.
- Slots are the type's: `answer`/`timeout` on `wait_for_input`, `yes`/`no` on `condition` and `check_subscription`, `next`/`error` on `script`, `next`/`error`/`hooks` on `web_app`, `branches` on `switch`, `next` everywhere else. Anything else is refused.
- Placeholders: `{{site_url}}`, `{{bot.username}}`, `{{bot.public_key}}`, `{{owner_chat}}`, `{{app_query}}` — use them in URLs and deep links instead of literal hosts and keys. Declare extra params in `"params"` and pass them in the write body.
- Working inside a plugin instance (its data lives under `app.<scope>.*`)? Write `app.*` in scripts and pass `scope` in the body — the server re-roots it.
- Message text goes in `paragraphs` (plain strings, Telegram HTML allowed: `<b>`, `<i>`, `<code>`, `<a href>`); tokens `###user.name###` render the visitor's variables.

## 3. Validate, then write

```bash
POST /api/v1/bots/:bot_id/flows/validate   # body: {"document": …, "params": {…}, "scope": "…"}
POST /api/v1/bots/:bot_id/flows/draft      # same body; writes the drafts
```

Send the body from a file, never inline a large document in the shell:

```bash
source ~/.lanka/config && curl -sS -X POST -H "Authorization: Bearer $LANKA_API_TOKEN" -H "Content-Type: application/json" \
  --data @/tmp/lanka-doc.json "$LANKA_API_URL/api/v1/bots/$BOT/flows/draft"
```

- Always `validate` first. It runs every check the write runs and keeps nothing. On `422`, fix what `error` names and validate again — do not retry the same document.
- `draft` answers `{flows: [{id, key, action: created|updated, name, editor_url}]}`. A flow named by its `id` had its draft replaced; a flow without one was created (nothing live until published).
- `409` means the owner changed the draft in the editor since your export. **Export again, redo your change on the fresh `root`, write again.** Never force it.
- After a successful write, give the user the `editor_url` and say what to look at, then: *«перевір у редакторі й опублікуй»*. That is the end of your job — publishing is theirs.

## 4. Copying flows between bots

Export from the source bot, write to the target: flow `id`s from another bot are ignored and the flows are created. References to flows outside the export stay as ids and are refused on another bot (`422 … is not a flow of this bot`) — include those flows in the export or drop the reference.
