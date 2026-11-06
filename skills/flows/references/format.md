# lanka-flows/v1

One JSON document describes one or more flows of a bot. The installer, the exporter and the draft writer all speak it.

```json
{
  "format": "lanka-flows/v1",
  "title": "Запис на стрижку",
  "params": ["site_url"],
  "flows": [
    {
      "id": "flow_Xk3mQ8pZ",
      "key": "booking",
      "name": "✂️ Запис",
      "group": "Салон",
      "trigger": { "type": "command", "value": "/book" },
      "draft_version": "1757580000",
      "root": { "type": "message", "name": "Вітання", "paragraphs": ["Привіт!"] }
    }
  ]
}
```

## Document

| key | meaning |
|---|---|
| `format` | always `"lanka-flows/v1"` |
| `title`, `description` | free text, for people |
| `params` | names of `{{placeholders}}` the caller must supply (besides the built-ins) |
| `mini_apps` | the Mini App kinds the flows open (`shop`, `calendar`, `week`), created in the write's data space unless the bot has them there (mini-apps.md) |
| `seed_app_variables` | `app.*` keys with initial values, written only where the bot has none; they land on write, before anything is published. A key is one top-level name, never a path: `{"veres": {"squad": …}}` is read as `app.veres.squad`; `{"veres.squad": …}` stores a variable literally named `veres.squad`, which no token or script can reach |
| `seed_audiences` | audiences the flows send to: `[{"key", "name", "filters": [{"field", "operator", "value"}]}]`, created on install unless the bot already has one of that name (then the owner's is used as is) |
| `panels` | the owner's pages over the flows' data — a Storefront, a Schedule or a table: `[{"key", "title", "icon", "definition"}]` (data.md § Panels). Created on install unless the bot has one of that key; a document written to drafts replaces it. An action names its flow by `"flow": "<key>"` |
| `flows` | the flows, in order |

## Flow

| key | meaning |
|---|---|
| `key` | unique inside the document; how other flows refer to it (`target_flow`, `notify_flow`) |
| `id` | the flow's key (`flow_…`, from an export or the flow list) — **update this flow's draft**. Absent or another bot's → a new flow; a string that is not a flow key is refused |
| `draft_version` | as exported; required while that flow has a dirty draft, refused (409) if stale |
| `name` | shown in the sidebar |
| `group` | sidebar group |
| `area` | `business` for a flow of the owner's own Telegram chats (Telegram Business); absent, the chats with the bot. Chosen when the flow is created, never changed: a write that names another area is refused |
| `trigger` | `{type, value}`: `command` (`/book`), `text_match` (a phrase; of several that match, the flow higher in the list wins), `fallback` (no value: the text no command, phrase or button took; one active per bot and per area), or omit for a flow reached only from buttons |
| `internal` | `true` for a notification flow that belongs to a `notify` node, hidden from lists |
| `root` | the first node |
| `published` | (export only) the live tree, when `root` came from a draft; ignored on write |
| `unpublished` | (export only) `true` for a flow that was never published — `root` is its draft; other flows still refer to it by key. Ignored on write |

## Node

```json
{
  "id": "node_7hRv2Lq9",
  "type": "message",
  "name": "Меню",
  "paragraphs": ["<b>Що робимо?</b>", "Оберіть:"],
  "content": { "keyboard_persistent": true },
  "buttons": [
    { "text": "Записатись", "node": { "type": "navigation", "name": "→ Запис", "content": { "action": "start_flow", "target_flow": "booking" } } },
    [ { "text": "Ціни", "node": { … } }, { "text": "Адреса", "node": { … } } ]
  ],
  "next": { … }
}
```

| key | meaning |
|---|---|
| `type` | one of the node types (node-types.md). A node that keeps its `id` keeps its type, or takes a kindred one (`message` ↔ `reply_keyboard`, the `ask_*` steps among themselves, `invoice_stars` ↔ `invoice_card`); any other change is a new step without the old id |
| `name` | the node's name in the editor; a button child's name usually equals the button text |
| `id` | the key of the live node this node stands for (`node_…`, from an export) — keeps its identity on publish; a node without an id is new; a string that is not a node key is refused |
| `focus` | `true` on one node: where the owner's editor lands after the write |
| `paragraphs` | the message body: one string per paragraph, Telegram HTML inside; paragraphs are joined with an empty line. Alternatively `content.html` + `content.text` verbatim — see *Lines without a gap*. A rich text has `content.blocks` instead — see *Buttons inside a rich text* |
| `content` | the type's own fields, by its schema (generated.json → `node_types.<type>.content`: fields, types, required, defaults, an example; node-types.md explains them). An unknown field or a wrong type is refused, and so is a missing required field on a new node. Never a field borrowed from a kindred step: `check_subscription` checks one chat, `channel_chat_id`; `channel_access` and `channel_revoke` take a list, `channel_chat_ids`. Chat ids are strings. Strings may carry `{{placeholders}}` and `###tokens###` |
| `buttons` | keyboard rows: an entry is one button `{text, node?}` on its own row, or an array — one row of several. A button's `node` is what happens on tap: a step, or a *button kind* (`url`, `web_app`, `share`, `copy_text`, `list_buttons`), or a `visibility` node with the kind under it. Options are a button whose child is a `list_buttons` node. The button kinds exist only under a button; a reply keyboard is the `reply_keyboard` node |
| slots | one child per slot the type has — see below |

### Slots by type

| type | slots |
|---|---|
| `ask_text`, `ask_contact`, `ask_location`, `ask_file`, `ask_voice`, `ask_date` | `answer` (index 0), `timeout` (1) |
| `invoice_stars`, `invoice_card` | `next` (0, at once), `paid` (1), `timeout` (2) |
| `channel_access` | `next` (0), `expiring` (1), `expired` (2), `failed` (3) |
| `condition`, `check_subscription` | `yes` (0), `no` (1) |
| `script`, `ai` | `next` (0), `error` (1) |
| `album` | `next` (0), `no_images` (1) |
| `web_app` | `next` (0, after sendData), `error` (1), `hooks` (series from 10: one entry per hook of the app, `null` for a hook without a branch) |
| `switch` | `branches` (series from 0: case i → branch i, the fallback last) |
| everything else | `next` (0) |

A slot the type does not have is refused: *"message «Hi» has no error slot"*.

A `message` or `reply_keyboard` with `buttons` waits for a tap: its `next` is accepted and never runs. What follows a tap goes under that button, as its `node`.

### Lines without a gap

`paragraphs: ["a", "b"]` is `<p>a</p><p></p><p>b</p>` — the empty `<p></p>` is the blank line. For lines that follow each other directly, write the body yourself: every line its own `<p>`, an empty `<p></p>` only where a gap belongs, and `text` the same with `\n`:

```json
"content": {
  "html": "<p><b>Траси</b></p><p></p><p>1. Щілина — <b>5b</b></p><p>2. Горобина — <b>6a</b></p>",
  "text": "Траси\n\n1. Щілина — 5b\n2. Горобина — 6a"
}
```

Never `\n` inside a paragraph: it sends fine, but the editor drops it the moment the owner saves the node, and the lines run together. `<br>` survives the editor, but adjacent `<p>`s are what the owner's own lines look like.

### Buttons inside a rich text

A `message` with `content.format: "rich"` keeps its body in `content.blocks` (node-types.md § Rich text). Its buttons can sit inside the text: a `{"type": "button", "button": {…}}` part of a text, or a `{"type": "buttons", "buttons": [{…}, …]}` row. Each such button has `text` and either `url` or `node`. The `node` is the step a tap opens, written like any other node. A button with neither is refused.

```json
"content": {
  "format": "rich",
  "blocks": [
    { "type": "heading", "size": 1, "text": "Довідка" },
    { "type": "paragraph", "text": "Оберіть розділ:" },
    { "type": "buttons", "buttons": [
      { "text": "Кроки", "node": { "type": "message", "name": "Кроки", "paragraphs": ["…"] } },
      { "text": "Сайт", "url": "{{site_url}}" }
    ] }
  ]
}
```

An export gives the step back as the button's `node`, with its `id`. Keep that `id` when you write the document back: the button then keeps its own id too, and messages already sent still open the step. A button whose step was deleted in the editor is left out of the export, with a warning.

Buttons in the text make the step wait for a tap, like a keyboard does: a `next` on it never runs. Keyboard `buttons` under a rich message work as usual.

### Cross-flow references

Inside `content`: `target_flow` (navigation `start_flow`) and `notify_flow` (notify) name a flow by `key` — a flow of the **document**, or one of the bot outside it: by its `flow_…` key from the flow list, or by the key its own export would give it (`"faq"` from the name, `"f_k3Jd8sQ2"` for a name that does not transliterate; the export warns about each). Writing back fewer flows than you exported works the same way: a key of a flow you left out still names it. The writer looks such a key up among the bot's flows and refuses one that no flow or several flows answer to.

To change one flow, write a document with that flow alone, its `id` on it: the bot's other flows are not touched, and its jumps to them keep their keys. Leave `panels` out of such a document unless you mean to change them — an export of one flow carries every panel of the bot, and a written panel replaces the bot's. `audience_keys` on an announce names the document's `seed_audiences` by key and resolves into `audience_ids` (added to any ids already there).

### Placeholders

`{{site_url}}` — this Lanka host; `{{bot.username}}`, `{{bot.public_key}}`; `{{owner_chat}}` — the owner's Telegram id (for `notify` → `target_chat_id`); `{{app_query}}` / `{{app_query_and}}` — `?scope=…` / `&scope=…` of the data space, empty in the main space; `{{flow.<key>}}` — the link key of a flow of this document, for deep links `t.me/{{bot.username}}?start={{flow.<key>}}-<data>`. Anything else must be listed in `params` and passed in the write body as `"params": {"name": "value"}`. Write placeholders instead of literal hosts and keys.

Mini App url: `{{site_url}}/apps/<kind>/{{bot.public_key}}{{app_query}}`.

## Write body

```json
{ "document": { … }, "params": { "site_url": "…" }, "scope": "shafa" }
```

`params.site_url` defaults to the Lanka host; `scope` — the data space (data.md), the main one when left out.
