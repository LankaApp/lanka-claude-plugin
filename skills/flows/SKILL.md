---
name: flows
description: Author, read and edit Lanka chatbot flows (ланки) through Lanka's MCP server or API, tied to the editor page the user has open. Use whenever the user wants to build, change, inspect or copy a bot's flows, nodes, buttons, scripts or Mini App steps in Lanka — including requests phrased in Ukrainian ("зроби ланку", "додай кнопку", "подивись бота"). Invoked as /lanka:flows <session>, the argument is the name shown in the editor's Claude chip (e.g. тихий-кіт).
---

# Lanka flows

You work on a Lanka bot through the **`lanka` MCP server** (tools `mcp__plugin_lanka_lanka__*`: get_context, look, open_review, list_bots, list_flows, export_flows, validate_flows, write_draft, discard_draft, delete_flow, get_app_value, put_app_value, delete_app_variable, list_mini_apps, upload_by_url, list_panels, save_panel, delete_panel, list_channels, list_broadcasts, save_broadcast, delete_broadcast, guide). You **read** flows as `lanka-flows/v1` documents and **write drafts**; the owner publishes in the Lanka editor. You never publish, and nothing you write reaches visitors until they do.

The server's own instructions carry the condensed workflow; this page adds the editor-page tie, what to advise before building, the references and the fallback without MCP. Where they disagree, the server is newer.

Answer in the user's language. Flows, buttons and messages are written in the language the bot speaks to its visitors (usually Ukrainian) unless told otherwise.

## 0. Which way in (every session, once)

**The `lanka` tools are in the session** → use them, nothing to set up. The first call opens Lanka's OAuth consent in the browser once; a tool answering *unauthorized* means the token is gone — tell the user to run `/mcp`, pick `lanka` and sign in again. No `~/.lanka/config` and no curl while the tools are there.

**No `lanka` tools** (server off, older Claude Code) → the REST API with a permanent key:

```bash
source ~/.lanka/config && test -n "$LANKA_API_URL" -a -n "$LANKA_API_TOKEN" && echo ok
```

If the file is missing or empty, stop and tell the user: create an API key in Lanka (user menu → **Integrations**) and save it as `~/.lanka/config` with `LANKA_API_URL=…` and `LANKA_API_TOKEN=…` (`chmod 600`). Never guess a host or a key. Every call is then:

```bash
source ~/.lanka/config && curl -sS -H "Authorization: Bearer $LANKA_API_TOKEN" "$LANKA_API_URL/api/v1/…"
```

Responses are JSON. `401` — the key is wrong or revoked; `403` — the account is not approved yet; `404` — not this owner's bot or flow; `409` — stale draft or value (§3); `422` — refused, `error` says why; `428` — waiting for `confirm` (§5).

Each tool has a REST route with the same arguments and answer: `tool` · `METHOD /route` below. Send a body from a file (`--data @/tmp/lanka-doc.json`), never inline.

## 1. Tie yourself to the editor page, then read

Every open bot page is a **session** with a sayable name (the Claude chip in the editor shows it, e.g. `тихий-кіт`); it lives as long as the tab stays on that bot, whichever page of it is open. Given as the skill's argument (`/lanka:flows тихий-кіт`) or mentioned by the user, **keep it for the whole chat**: pass it as `session` when reading the context and in every write, so what you write lands on that page only.

`get_context {session}` · `GET /api/v1/context?session=тихий-кіт` → `{open: {session, bot: {id, name}, page, flow: {id, name, group, draft_dirty, editor_url} | null, node_id, seen_at, stale, session_variables}, sessions: [{session, bot, page, flow, focused_at}], skill: {digest}}`. Without a name: the page focused most recently, and the other live pages.

**Is this skill current?** `skill.digest` is the digest of what Lanka's code allows now; `digest` at the top of [references/generated.json](references/generated.json) is what this copy knows. When they differ, Lanka has steps, slots or functions this copy lacks: tell the user once, at the start (`claude plugin marketplace update lanka && claude plugin update lanka@lanka`, then a new session), and go on with the task — `guide {topic}` (`format`, `data`, `steps`, `index`) reads the current text from the server.

When `open` is present and not `stale`, that bot and flow are what "here", "this flow", "add a button" refer to — do not ask which bot. `flow` is `null` while the tab is on another page of the bot (`page` says which: `broadcasts`, `users`, …) — then "here" is the bot and "this flow" needs a name; a write or a look takes the tab into the editor by itself. `node_id` is the node the owner selected; `session_variables` are the names the flow's group already uses. With no session name: one live page in `sessions` — take it; several — ask which, by name. From then on that page's name is your session for the chat.

When the user says **"go to …"** («перейди на …», «відкрий …») — a flow by name, or a node — take their page there: `look {session, flow_id, node_id}` · `POST /api/v1/context/look` → `{session, flow: {id, name, editor_url}, node_id}`. The page switches to that flow and selects the node; nothing is written. Call it only on that request — never after a plain export or to orient yourself, or the page jumps around while you read. `404` — no live page of that name, or the flow belongs to another bot.

With nothing open, list the bots. Reading and live data:

| what | tool | route |
|---|---|---|
| the owner's bots | `list_bots` | `GET /api/v1/bots` → `[{id, name, platform, username, url}]` |
| a bot's flows | `list_flows {bot_id}` | `GET /api/v1/bots/:bot_id/flows` → `[{id, name, group, trigger_type, trigger_value, active, internal, draft_dirty, updated_at, editor_url}]` |
| one flow (+ the notification flows it fires), or the whole bot | `export_flows {bot_id, flow_id?, scope?}` | `GET …/flows/:flow_id/export`, `GET …/flows/export` → `{document, warnings}` |
| the bot's Mini Apps, urls, contracts | `list_mini_apps {bot_id}` | `GET …/mini_apps` (references/mini-apps.md) |
| a picture copied into the bot's media | `upload_by_url {bot_id, url}` | `POST …/uploads` → `{url}` — put THAT url into `media_url` or a catalog item, never the foreign one |
| one `app.*` value | `get_app_value {bot_id, key, scope?}` | `GET …/app/:key?scope=` → `{key, scope, value, version}` |
| replace it | `put_app_value {bot_id, key, scope?, value, version}` | `PUT …/app/:key` — `409` when `version` is stale |
| the owner's pages over `app.*` | `list_panels`, `save_panel {bot_id, panel: {key, title, icon?, scope?, definition}}`, `delete_panel` | `GET`, `POST`, `PUT …/panels/:key`, `DELETE …/panels/:key` (references/data.md § Panels) |
| the bot's linked channels and groups — where a channel post's `target_chat_ids` come from | `list_channels {bot_id}` | `GET …/channels` → `{channels: [{chat_id, title, username, chat_type, bot_is_admin, can_invite, can_ban}]}` |
| broadcasts and channel posts, latest first, each with its `editor_url` (a post lives on the posts page) | `list_broadcasts {bot_id, audience?, page?}` (`audience`: `visitors` \| `channels`), `save_broadcast {bot_id, id?, broadcast: {name, audience, audience_ids \| audiences, target_chat_ids, flow_id \| document, params, send_at_local, time_zone}}` | `GET …/broadcasts?audience=`, `POST`, `PATCH …/broadcasts/:id` — a DRAFT; nothing is sent or scheduled |

`app.*` is live data, not a draft: a `put_app_value` reaches visitors at once. Read it freely; write it only when the owner asks for that very change, and say what the value becomes.

**Read export `warnings` and pass anything relevant to the user** — they name what the format could not carry (raw html, a reference to a flow outside the export).

In the document, every flow's **`root` is the tree to work on**: the editing draft while the owner has unpublished changes, the live tree otherwise. Next to a draft you also get `published` (what visitors see now) and `draft_version`. Every flow and node carries its `id`.

Several bots and the request does not say which — ask. A flow mentioned by name or command — find it in the list first; do not invent ids.

## 2. Author or edit

### Know what Lanka already does — advise first

The owner builds a bot, not a flow. When a part Lanka already has serves the request, say so and point at it before building by hand; when the part is the owner's to set up (no API for it), say where and what to put there. Name the part, what it would do here, and its page — `https://lanka.bot/bots/<bot_id>/<path>`. Suggest; never build a part unasked.

- **Templates** (`templates`) — for a booking, a shop, a paid private channel or a live feed, offer the template first: it installs ready flows with a short wizard. «Запис на час» (calendars, reminders, moving and cancelling a visit), «Магазин» (Storefront, cart, orders), «Закритий клуб» (payment, channel access for a term, renewal reminders), «Ефір» (the owner's posts go to subscribers). Installed again under another name, a template gets its own data space. Build by hand what no template covers, or adapt the installed flows — export them like any other.
- **Broadcasts** (`broadcasts`) — a message or a whole flow to audiences, now or at a set time, with stats. You may write a broadcast draft; sending stays with the owner.
- **Audiences** (`audiences`) — visitors grouped by their variables; a broadcast, a job or a visitors table reaches them.
- **Visitors** (`users`) — everyone who wrote to the bot, a table with export to XLSX/CSV.
- **Data and panels** (`variables`, `panels/<key>`) — `app.*` and the visitors' defaults; a panel is the owner's table over an `app.*` list, with status columns and actions (data.md).
- **Jobs** (`jobs`) — a script on a schedule: over the shared data (refresh a list every night) or for each visitor of some audiences, then a flow (a morning reminder). The owner sees each run and hears when one fails. No API for jobs: write the script, tell the owner to create the job and paste it.
- **Payments** (`payments`) — Telegram Stars and card invoices: `invoice_stars` / `invoice_card`.
- **Steps that save building** — channel access for a term, `remind`, `copy_message` (a visitor's message passed on as is), `notify` (to the owner or a chat), `announce`, `ask_date` with its calendar: references/node-types.md.

### The document

Write a `lanka-flows/v1` document. The format is in [references/format.md](references/format.md); node types and their `content` keys in [references/node-types.md](references/node-types.md); scripts in [references/safe-expr.md](references/safe-expr.md); hosted Mini Apps in [references/mini-apps.md](references/mini-apps.md); `app.*` data, data spaces and panels in [references/data.md](references/data.md). [references/generated.json](references/generated.json), format.md and data.md are generated from Lanka's code (`guide` serves them current); when a prose page and generated.json disagree, it is right. The export of the user's own bot is your worked example.

Rules that are easy to get wrong:

- **Editing an existing flow: keep `id` on the flow and on every node you keep.** A node with its `id` keeps its identity when the owner publishes (sent buttons keep working); a node without one is new. Drop the `id` only for nodes you replace on purpose. Keep `draft_version` as exported.
- **A node keeps its type, or gets a kindred one.** Change `type` only between steps that are the same thing in another form: a `message` and a `reply_keyboard`, the `ask_*` steps among themselves, `invoice_stars` and `invoice_card`. Any other change — a message turned into a `navigation`, a `script` into a `condition` — is a new step: drop the old node and write the new one without its `id`. A sent button points at the node it hangs on, so nothing a visitor holds breaks, and the review reads «removed / added» instead of a message that became a jump.
- **Mark where to look.** Put `"focus": true` on the node the owner should land on (the one you added or changed most) — their editor jumps to it when the draft is written.
- **New dialog: write it whole.** A new flow, or several that belong together (a menu and the flows its buttons start), goes in one document — half a dialog is not reviewable. **Existing flow: one change per write** — the owner watches the editor; do not rewrite a bot to add a button.
- **A very long flow is easier to keep as several.** When a flow has grown past what fits in one look (dozens of steps, several independent stretches — a catalog, a checkout, a FAQ), **suggest** splitting it into flows by meaning, joined by `navigation` `start_flow` and kept in one `group`; session variables carry across the jump. Say where you would cut and why, and wait for a yes — never split on your own initiative or as a side effect of another change. Not for a flow that reads in one screen: one idea per flow.
- A list of options is a **button** whose child is a `list_buttons` node (`options_var`, `save_to`) — no "choice" node. A conditional button is a `visibility` node standing first under the button; the button's kind (`url`, `web_app`, `share`, `copy_text`, `list_buttons`) goes under it.
- `url`, `share`, `copy_text`, `web_app`, `list_buttons` exist only as a button's child. A reply keyboard is the `reply_keyboard` node; it cannot carry url buttons.
- HTTP inside a flow is `fetch(...)` in a `script` node — no api_call node. A fetch result needs a home: `temp.r = fetch(...)` or `session.r = fetch(...)`.
- Slots are the type's (format.md § Slots by type): `answer`/`timeout` on the `ask_*` steps, `yes`/`no` on `condition`, `next`/`error` on `script`, `branches` on `switch`, `paid`, `expired`, `hooks` where the type has them, `next` everywhere else. Anything else is refused.
- Placeholders: `{{site_url}}`, `{{bot.username}}`, `{{bot.public_key}}`, `{{owner_chat}}`, `{{app_query}}` — in URLs and deep links instead of literal hosts and keys. Declare extra params in `"params"` and pass them in the write body.
- Inside a data space (data under `app.<scope>.*`) write `app.*` in scripts and pass `scope` in the body — the server re-roots it.
- `seed_app_variables` keys are top-level names, nested with objects: `{"veres": {"squad": …}}` → `app.veres.squad`. A dotted key (`"veres.squad"`) becomes a variable literally named so, and every `app.veres.*` token renders empty.
- Message text goes in `paragraphs` (plain strings, Telegram HTML allowed: `<b>`, `<i>`, `<code>`, `<a href>`); tokens `###user.name###` render the visitor's variables. Paragraphs are separated by an empty line. For lines with no gap (a list, a table of routes) never put `\n` inside a paragraph — the editor drops it when the owner saves the node and the lines run together. Write `content.html` with adjacent `<p>`s instead (format.md).
- `seed_app_variables` land when the draft is written, not when published, and only where the key is missing — a second write never replaces them.
- A tap continues the conversation that sent the button; a visitor keeps at most 5 open ones, and a tap on an older, closed one restarts that flow from its root. State a later tap must find (a moderation queue, a booking in progress) belongs in `app.*` or `user.*`, keyed by an id the button's script checks — not in `session.*` alone.

## 3. Validate, then write

`validate_flows {bot_id, document, params?, scope?, session}` · `POST /api/v1/bots/:bot_id/flows/validate`
`write_draft` — the same arguments · `POST /api/v1/bots/:bot_id/flows/draft` — writes the drafts and tells that page

- Always validate first: every check the write runs, nothing kept. On `422`, fix what `error` names and validate again — never retry the same document.
- The write answers `{flows: [{id, key, action: created|updated, name, editor_url}]}`. A flow named by its `id` had its draft replaced; a flow without one was created (nothing live until published).
- `409` means the owner changed the draft in the editor since your export. **Export again, redo your change on the fresh `root`, write again.** Never force.
- After a write the page tied to the session updates by itself and jumps to the focused node; a write to another flow of the bot switches that page there. Give the `editor_url` and say what to look at.
- **When the task is done** — its last draft written, not after each write — open the review on the owner's page and say: *«перевір і опублікуй»*. Publishing is theirs. `open_review {session}` · `POST /api/v1/context/review` → `{session, unpublished: [{id, name}]}`: the page opens one dialog with every unpublished flow, each publishable on its own or all at once.
- **Anything you changed besides flows is live already** — an `app.*` value, a panel, a Mini App, an upload, a broadcast draft — and the review does not show it. List each such change with a link to where the owner sees it: a panel `https://lanka.bot/bots/<bot_id>/panels/<key>`, the data `…/variables`, a broadcast the `editor_url` its answer gave.

## 4. Copying flows between bots

Export from the source bot, write to the target: flow `id`s from another bot are ignored and the flows are created. References to flows outside the export stay as ids and are refused on another bot (`422 … is not a flow of this bot`) — include those flows or drop the reference.

## 5. Undoing what you wrote

Removing is **two steps**: the first call, without `confirm`, answers a `summary` of what would go and a `confirm` token bound to that action (10 minutes; over REST the status is `428`); the same call with `confirm: "<token>"` executes it.

| tool | route | what goes |
|---|---|---|
| `delete_flow {bot_id, flow_id, session}` | `DELETE …/flows/:flow_id` | a flow that was NEVER published; refused (`422`) once published or while another flow starts it |
| `discard_draft {bot_id, flow_id, session}` | `DELETE …/flows/:flow_id/draft` | the unpublished changes; the live tree stays |
| `delete_app_variable {bot_id, key}` | `DELETE …/app_variables/:key` | an `app.*` key no node of the bot reads (a text match on `app.<key>`) |
| `delete_panel`, `delete_broadcast` | `DELETE …/panels/:key`, `DELETE …/broadcasts/:id` | a panel; a broadcast draft |

Pass `session` too, so the page follows: a deleted flow leaves the sidebar, a discarded draft shows the live tree again.

**The token is not yours to spend.** Before the second call, show the owner the `summary` in the chat — the flow's name and node count, the draft version, the key and its value preview — and send `confirm` only after they say yes to that. A "clean up" or "undo" in the request authorises the first call, never the second. Never delete through the editor's own routes, and never ask for a published flow to be removed — that stays with the owner.
