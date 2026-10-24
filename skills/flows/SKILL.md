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
GET /api/v1/context?session=тихий-кіт   # {open: {session, bot: {id, name}, page, flow: {id, name, group, draft_dirty, editor_url} | null, node_id, seen_at, stale, session_variables}, sessions: [{session, bot, page, flow, focused_at}], skill: {digest}}
GET /api/v1/context                     # without a name: the page focused most recently, and the other live pages
```

**Is this skill current?** `skill.digest` is the digest of what Lanka's code allows now; `digest` at the top of [references/generated.json](references/generated.json) is what this copy knows. When they differ, Lanka has steps, slots or functions this copy does not describe: tell the user once, at the start, and go on with the task.

```bash
claude plugin marketplace update lanka
claude plugin update lanka@lanka
```

Then a new Claude Code session picks up the update.

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
GET /api/v1/bots/:bot_id/mini_apps                # the bot's Mini Apps by data space, with their urls and contracts (references/mini-apps.md)
POST /api/v1/bots/:bot_id/uploads              # body {"url": "https://…/photo.jpg"} (or multipart file) → {url}: the picture copied into the bot's media — put THAT url into media_url or a catalog item, never the foreign one
GET /api/v1/bots/:bot_id/app/:key?scope=       # one app.* value → {key, scope, value, version}
PUT /api/v1/bots/:bot_id/app/:key              # body {"value": …, "version": "…"} — replaces it; 409 when version is stale
GET /api/v1/bots/:bot_id/panels                # the owner's pages over app.* (Storefront, Schedule, tables) → {panels: [{key, title, icon, scope, definition}]}
POST /api/v1/bots/:bot_id/panels               # body {"panel": {key, title, icon?, scope?, definition}} — a new table panel (references/data.md § Panels)
PUT /api/v1/bots/:bot_id/panels/:key           # the same body — replaces title/icon/definition
DELETE /api/v1/bots/:bot_id/panels/:key
GET /api/v1/bots/:bot_id/channels              # the bot's linked channels and groups → {channels: [{chat_id, title, username, chat_type, bot_is_admin, can_invite, can_ban}]}; chat_id strings go into a channel broadcast's target_chat_ids
GET /api/v1/bots/:bot_id/broadcasts            # the bot's broadcasts, latest first, each with its editor_url
POST /api/v1/bots/:bot_id/broadcasts           # body {"broadcast": {name, audience, audience_ids | audiences, target_chat_ids, flow_id | document, params, send_at_local, time_zone}} — a DRAFT; nothing is sent or scheduled
PATCH /api/v1/bots/:bot_id/broadcasts/:id      # the same body, a draft only; DELETE removes a draft (two steps, §5)
```

`app.*` is live data, not a draft: a `PUT` reaches visitors at once. Read it freely; write it only when the owner asks for that very change, and say what the value becomes.

Export answers `{document, warnings}`. **Read `warnings` and pass anything relevant to the user** — they name what the format could not carry (raw html, a reference to a flow outside the export).

In the document, every flow's **`root` is the tree to work on**: the editing draft while the owner has unpublished changes, the live tree otherwise. Next to a draft you also get `published` (what visitors see right now) and `draft_version`. Every flow and node carries its `id`.

When several bots exist and the request does not say which, ask. When a request mentions a flow by name or command, find it in the list first — do not invent ids.

## 2. Author or edit

### Know what Lanka already does — advise first

The owner builds a bot, not a flow. When a request is served by a part Lanka already has, say so and point at it before building anything by hand; when the part is the owner's to set up (there is no API for it), say where and what to put there. Name the part, what it would do here, and its page — `$LANKA_API_URL/bots/<bot_id>/<path>`. Suggest; never build a part unasked.

- **Templates** (`templates`) — for a booking, a shop, a paid private channel or a live feed, offer the template first: it installs ready flows with a short wizard. «Запис на час» (calendars, reminders, moving and cancelling a visit), «Магазин» (Storefront, cart, orders), «Закритий клуб» (payment, channel access for a term, renewal reminders), «Ефір» (what the owner posts goes to the subscribers). Installed a second time under another name, a template gets its own data space. Build by hand what no template covers, or adapt the flows a template installed — export them like any other.
- **Broadcasts** (`broadcasts`) — a message or a whole flow to audiences, now or at a set time, with its stats. You may write a broadcast draft through the API; sending it stays with the owner.
- **Audiences** (`audiences`) — visitors grouped by their variables; a broadcast, a job or a visitors table reaches them.
- **Visitors** (`users`) — everyone who wrote to the bot, as a table with export to XLSX/CSV.
- **Data and panels** (`variables`, `panels/<key>`) — `app.*` and the visitors' defaults; a panel is the owner's table over an `app.*` list, with status columns and actions (references/data.md).
- **Jobs** (`jobs`) — a script on a schedule: over the shared data (refresh a list from another service every night) or for each visitor of some audiences, then a flow (a morning reminder). The owner sees each run and hears when one fails. There is no API for jobs: write the script, and tell the owner to create the job and paste it.
- **Payments** (`payments`) — Telegram Stars and card invoices, the `invoice_stars` / `invoice_card` steps.
- **Steps that save building** — channel access for a term, `remind`, `copy_message` (a visitor's message passed on as it is), `notify` (to the owner or a chat), `announce`, `ask_date` with its calendar: references/node-types.md.

### The document

Write a `lanka-flows/v1` document. The format is in [references/format.md](references/format.md); node types and their `content` keys in [references/node-types.md](references/node-types.md); scripts in [references/safe-expr.md](references/safe-expr.md); hosted Mini Apps in [references/mini-apps.md](references/mini-apps.md); `app.*` data, data spaces and panels in [references/data.md](references/data.md). [references/generated.json](references/generated.json) is generated from Lanka's code — the exact node types, slots, enums and script functions; when a prose page and it disagree, it is right. The export of the user's own bot is your worked example.

Rules that are easy to get wrong:

- **Editing an existing flow: keep `id` on the flow and on every node you keep.** A node with its `id` keeps its identity when the owner publishes (buttons already sent to visitors keep working); a node without one is new. Drop the `id` only for nodes you replace on purpose. Keep `draft_version` as exported.
- **A node keeps its type, or gets a kindred one.** Change `type` only between steps that are the same thing in another form: a `message` and a `reply_keyboard`, the `ask_*` steps among themselves, `invoice_stars` and `invoice_card`. Any other change — a message turned into a `navigation`, a `script` into a `condition` — is a new step: drop the old node and write the new one without its `id`. A sent button points at the node it hangs on, not at the step under it, so nothing a visitor holds breaks; and the owner's review reads «removed / added» instead of a message that became a jump.
- **Mark where to look.** Put `"focus": true` on the node the owner should land on (the one you added or changed most) — their editor jumps to it the moment the draft is written.
- **New dialog: write it whole.** A new flow, or several that belong together (a menu and the flows its buttons start), goes in one document — half a dialog is not reviewable. **Existing flow: one change per write** — the owner watches the editor; do not rewrite a bot to add a button.
- **A very long flow is easier to keep as several.** When a flow you read or are asked to grow has grown past what fits in one look (dozens of steps, several independent stretches — a catalog, a checkout, a FAQ), **suggest** splitting it into flows by meaning, joined by `navigation` `start_flow` and kept in one `group`; session variables carry across the jump. Say where you would cut and why, and wait for a yes — never split on your own initiative, and never as a side effect of another change. Do not suggest it for a flow that reads in one screen: one idea per flow, not one flow per step.
- A list of options is a **button** whose child is a `list_buttons` node (`options_var`, `save_to`) — there is no "choice" node. A conditional button is a `visibility` node standing first under the button; the button's kind (`url`, `web_app`, `share`, `copy_text`, `list_buttons`) goes under it.
- `url`, `share`, `copy_text`, `web_app`, `list_buttons` exist only as a child of a button. A reply keyboard is the `reply_keyboard` node; it cannot carry url buttons.
- HTTP inside a flow is `fetch(...)` in a `script` node — there is no api_call node. A fetch result needs a home: `temp.r = fetch(...)` or `session.r = fetch(...)`.
- Slots are the type's: `answer`/`timeout` on the `ask_*` steps, `next`/`paid`/`timeout` on `invoice_stars` and `invoice_card`, `next`/`expiring`/`expired`/`failed` on `channel_access`, `yes`/`no` on `condition` and `check_subscription`, `next`/`error` on `script`, `next`/`error`/`hooks` on `web_app`, `next`/`no_images` on `album`, `branches` on `switch`, `next` everywhere else. Anything else is refused.
- Placeholders: `{{site_url}}`, `{{bot.username}}`, `{{bot.public_key}}`, `{{owner_chat}}`, `{{app_query}}` — use them in URLs and deep links instead of literal hosts and keys. Declare extra params in `"params"` and pass them in the write body.
- Working inside a data space (its data lives under `app.<scope>.*`)? Write `app.*` in scripts and pass `scope` in the body — the server re-roots it.
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
- After a successful write the page tied to the session updates by itself and jumps to the focused node; a write to another flow of the bot switches that page there. Give the `editor_url` anyway and say what to look at.
- **When the task is done** — its last draft written, not after each write — open the review on the owner's page, then say: *«перевір і опублікуй»*. That is the end of your job — publishing is theirs.

  ```bash
  POST /api/v1/context/review   # body: {"session": "тихий-кіт"} → {session, unpublished: [{id, name}]}
  ```

  The page opens one dialog with every unpublished flow of the bot, step by step, each publishable on its own or all at once.
- **Anything you changed besides flows is live already** — an `app.*` value, a panel, a Mini App, an upload, a broadcast draft — and the review does not show it. List each such change in your answer with a link to where the owner sees it: a panel `$LANKA_API_URL/bots/<bot_id>/panels/<key>`, the data `…/variables`, a broadcast the `editor_url` its answer gave.

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
