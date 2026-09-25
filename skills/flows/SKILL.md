---
name: flows
description: Author, read and edit Lanka chatbot flows (ланки) through the Lanka API, tied to the editor page the user has open. Use whenever the user wants to build, change, inspect or copy a bot's flows, nodes, buttons, scripts or Mini App steps in Lanka — including requests phrased in Ukrainian ("зроби ланку", "додай кнопку", "подивись бота"). Invoked as /lanka:flows <session>, the argument is the name shown in the editor's Claude chip (e.g. тихий-кіт).
---

# Lanka flows

You work on a Lanka bot through its HTTP API. You **read** flows as `lanka-flows/v1` documents and **write drafts**; the owner publishes in the Lanka editor. You never publish, and nothing you write reaches visitors until they do.

Answer in the user's language. Flows, buttons and messages are written in the language the bot speaks to its visitors (usually Ukrainian) unless told otherwise.

## 0. Setup check (every session, once)

```bash
source ~/.lanka/config && test -n "$LANKA_API_URL" -a -n "$LANKA_API_TOKEN" && echo ok
```

If the file is missing or empty, stop and tell the user: create an API key in Lanka (user menu → **Integrations**) and save it as `~/.lanka/config` with `LANKA_API_URL=…` and `LANKA_API_TOKEN=…` (`chmod 600`). Do not guess a host or a key.

Every call below is:

```bash
source ~/.lanka/config && curl -sS -H "Authorization: Bearer $LANKA_API_TOKEN" "$LANKA_API_URL/api/v1/…"
```

Responses are JSON. `401` — the key is wrong or revoked; `403` — the account is not approved yet; `404` — not this owner's bot or flow; `409` — a stale draft (see §3); `422` — the document was refused, `error` says why.

## 1. Tie yourself to the editor page, then read

Every open bot page is a **session** with a sayable name (the Claude chip in the editor shows it, e.g. `тихий-кіт`); it lives as long as the tab stays on that bot, whichever page of it is open. Given as the skill's argument (`/lanka:flows тихий-кіт`) or mentioned by the user, **keep it for the whole chat**: pass it as `?session=` when reading the context and as `"session"` in every write, so what you write lands on that page and nowhere else.

```bash
GET /api/v1/context?session=тихий-кіт   # {open: {session, bot: {id, name}, page, flow: {id, name, group, draft_dirty, editor_url} | null, node_id, seen_at, stale, session_variables}, sessions: [{session, bot, page, flow, focused_at}]}
GET /api/v1/context                     # without a name: the page focused most recently, and the other live pages
```

When `open` is present and not `stale`, that bot and flow are what "here", "this flow", "add a button" refer to — do not ask which bot. `flow` is `null` while the tab is on another page of the bot (`page` says which: `broadcasts`, `users`, …) — then "here" is the bot, and "this flow" needs a name; a write or a look takes the tab into the editor by itself. `node_id` is the node the owner selected; `session_variables` are the names the flow's group already uses. With no session name: one live page in `sessions` — take it; several — ask which one, by name. Either way, from then on that page's name is your session for the chat.

When the user says **"go to …"** («перейди на …», «відкрий …») — a flow by name, or a node — take their page there:

```bash
POST /api/v1/context/look   # body: {"session": "тихий-кіт", "flow_id": 896, "node_id": 28549}  → {session, flow: {id, name, editor_url}, node_id}
```

The page switches to that flow of its bot and selects the node; nothing is written. Call it only on that request — never after a plain export or when you are just orienting yourself, or the page will jump around while you read. `404` — no live page of that name, or the flow belongs to another bot.

With nothing open, list the bots:

```bash
GET /api/v1/bots                       # [{id, name, platform, username, url}]
GET /api/v1/bots/:bot_id/flows         # [{id, name, group, trigger_type, trigger_value, active, internal, draft_dirty, updated_at, editor_url}]
GET /api/v1/bots/:bot_id/flows/export             # the whole bot as one document
GET /api/v1/bots/:bot_id/flows/:flow_id/export    # one flow (+ the notification flows it fires)
GET /api/v1/bots/:bot_id/mini_apps                # the Mini Apps this bot's plugins offer, with their contracts
POST /api/v1/bots/:bot_id/uploads              # body {"url": "https://…/photo.jpg"} (or multipart file) → {url}: the picture copied into the bot's media — put THAT url into media_url or a catalog item, never the foreign one
GET /api/v1/bots/:bot_id/app/:key?scope=       # one app.* value → {key, scope, value, version}
PUT /api/v1/bots/:bot_id/app/:key              # body {"value": …, "version": "…"} — replaces it; 409 when version is stale
```

`app.*` is live data, not a draft: a `PUT` reaches visitors at once. Read it freely; write it only when the owner asks for that very change, and say what the value becomes.

Export answers `{document, warnings}`. **Read `warnings` and pass anything relevant to the user** — they name what the format could not carry (raw html, a reference to a flow outside the export).

In the document, every flow's **`root` is the tree to work on**: the editing draft while the owner has unpublished changes, the live tree otherwise. Next to a draft you also get `published` (what visitors see right now) and `draft_version`. Every flow and node carries its `id`.

When several bots exist and the request does not say which, ask. When a request mentions a flow by name or command, find it in the list first — do not invent ids.

## 2. Author or edit

Write a `lanka-flows/v1` document. The format is in [references/format.md](references/format.md); node types and their `content` keys in [references/node-types.md](references/node-types.md); scripts in [references/safe-expr.md](references/safe-expr.md); hosted Mini Apps in [references/mini-apps.md](references/mini-apps.md); plugins and their `app.*` data in [references/plugins.md](references/plugins.md). [references/generated.json](references/generated.json) is generated from Lanka's code — the exact node types, slots, enums and script functions; when a prose page and it disagree, it is right. The export of the user's own bot is your worked example.

Rules that are easy to get wrong:

- **Editing an existing flow: keep `id` on the flow and on every node you keep.** A node with its `id` keeps its identity when the owner publishes (buttons already sent to visitors keep working); a node without one is new. Drop the `id` only for nodes you replace on purpose. Keep `draft_version` as exported.
- **Mark where to look.** Put `"focus": true` on the node the owner should land on (the one you added or changed most) — their editor jumps to it the moment the draft is written.
- **New dialog: write it whole.** A new flow, or several that belong together (a menu and the flows its buttons start), goes in one document — half a dialog is not reviewable. **Existing flow: one change per write** — the owner watches the editor; do not rewrite a bot to add a button.
- A list of options is a **button** whose child is a `list_buttons` node (`options_var`, `save_to`) — there is no "choice" node. A conditional button is a `visibility` node standing first under the button; the button's kind (`url`, `web_app`, `share`, `copy_text`, `list_buttons`) goes under it.
- `url`, `share`, `copy_text`, `web_app`, `list_buttons` exist only as a child of a button. A reply keyboard is the `reply_keyboard` node; it cannot carry url buttons.
- HTTP inside a flow is `fetch(...)` in a `script` node — there is no api_call node. A fetch result needs a home: `temp.r = fetch(...)` or `session.r = fetch(...)`.
- Slots are the type's: `answer`/`timeout` on `wait_for_input`, `yes`/`no` on `condition` and `check_subscription`, `next`/`error` on `script`, `next`/`error`/`hooks` on `web_app`, `branches` on `switch`, `next` everywhere else. Anything else is refused.
- Placeholders: `{{site_url}}`, `{{bot.username}}`, `{{bot.public_key}}`, `{{owner_chat}}`, `{{app_query}}` — use them in URLs and deep links instead of literal hosts and keys. Declare extra params in `"params"` and pass them in the write body.
- Working inside a plugin instance (its data lives under `app.<scope>.*`)? Write `app.*` in scripts and pass `scope` in the body — the server re-roots it.
- `seed_app_variables` keys are top-level names, nested with objects: `{"veres": {"squad": …}}` → `app.veres.squad`. A dotted key (`"veres.squad"`) becomes a variable literally named so, and every `app.veres.*` token renders empty.
- Message text goes in `paragraphs` (plain strings, Telegram HTML allowed: `<b>`, `<i>`, `<code>`, `<a href>`); tokens `###user.name###` render the visitor's variables. Paragraphs are separated by an empty line. For lines with no gap between them (a list, a table of routes) never put `\n` inside a paragraph — the editor drops it the first time the owner saves the node and the lines run together. Write `content.html` with adjacent `<p>`s instead (format.md).
- `seed_app_variables` land when the draft is written, not when it is published, and only where the key is missing — a second write never replaces them.
- A tap continues the conversation that sent the button, and a visitor keeps at most 5 open ones; a tap on an older, closed one restarts that flow from its root. State a later tap must find (a moderation queue, a booking in progress) belongs in `app.*` or `user.*`, keyed by an id the button's script checks — not in `session.*` alone.

## 3. Validate, then write

```bash
POST /api/v1/bots/:bot_id/flows/validate   # body: {"document": …, "params": {…}, "scope": "…", "session": "тихий-кіт"}
POST /api/v1/bots/:bot_id/flows/draft      # same body; writes the drafts and tells that page
```

Send the body from a file, never inline a large document in the shell:

```bash
source ~/.lanka/config && curl -sS -X POST -H "Authorization: Bearer $LANKA_API_TOKEN" -H "Content-Type: application/json" \
  --data @/tmp/lanka-doc.json "$LANKA_API_URL/api/v1/bots/$BOT/flows/draft"
```

- Always `validate` first. It runs every check the write runs and keeps nothing. On `422`, fix what `error` names and validate again — do not retry the same document.
- `draft` answers `{flows: [{id, key, action: created|updated, name, editor_url}]}`. A flow named by its `id` had its draft replaced; a flow without one was created (nothing live until published).
- `409` means the owner changed the draft in the editor since your export. **Export again, redo your change on the fresh `root`, write again.** Never force it.
- After a successful write the page tied to the session updates by itself and jumps to the focused node; a write to another flow of the bot switches that page there. Give the `editor_url` anyway and say what to look at, then: *«перевір у редакторі й опублікуй»*. That is the end of your job — publishing is theirs.

## 4. Copying flows between bots

Export from the source bot, write to the target: flow `id`s from another bot are ignored and the flows are created. References to flows outside the export stay as ids and are refused on another bot (`422 … is not a flow of this bot`) — include those flows in the export or drop the reference.

## 5. Undoing what you wrote

Three calls remove things. Each is **two steps**: the first request, without `confirm`, answers `428` with a `summary` of what would go and a `confirm` token bound to that one action (valid 10 minutes); the same request with `"confirm": "<token>"` executes it.

```bash
DELETE /api/v1/bots/:bot_id/flows/:flow_id            # a flow that was NEVER published; refused (422) once published or while another flow starts it
DELETE /api/v1/bots/:bot_id/flows/:flow_id/draft      # the unpublished changes; the live tree stays
DELETE /api/v1/bots/:bot_id/app_variables/:key        # an app.* key no node of the bot reads (a text match on `app.<key>`)
```

Pass `session` too, so the page follows: a deleted flow leaves the sidebar (and the page moves on if it had it open), a discarded draft shows the live tree again.

**The token is not yours to spend.** Before the second request, show the owner the `summary` in the chat — the flow's name and node count, the draft version, the key and its value preview — and send `confirm` only after they say yes to that. A "clean up" or "undo" in the request authorises the first call, never the second. Never delete anything through the editor's own routes, and never ask for a published flow to be removed — that stays with the owner.
