# lanka-flows/v1

One JSON document describes one or more flows of a bot. The installer, the exporter and the draft writer all speak it.

```json
{
  "format": "lanka-flows/v1",
  "title": "Запис на стрижку",
  "params": ["site_url"],
  "flows": [
    {
      "id": 42,
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
| `requires.plugins` | plugin ids the flows need (`catalog`, `schedule`) — informational |
| `seed_app_variables` | `app.*` keys with initial values, written only where the bot has none |
| `flows` | the flows, in order |

## Flow

| key | meaning |
|---|---|
| `key` | unique inside the document; how other flows refer to it (`target_flow`, `notify_flow`) |
| `id` | the flow's id in this bot (from an export) — **update this flow's draft**. Absent or another bot's → a new flow |
| `draft_version` | as exported; required while that flow has a dirty draft, refused (409) if stale |
| `name` | shown in the sidebar |
| `group` | sidebar group |
| `trigger` | `{type, value}`: `command` (`/book`), `text_match` (a phrase), or omit for a flow reached only from buttons |
| `internal` | `true` for a notification flow that belongs to a `notify` node, hidden from lists |
| `root` | the first node |
| `published` | (export only) the live tree, when `root` came from a draft; ignored on write |

## Node

```json
{
  "id": 1001,
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
| `type` | one of the node types (node-types.md) |
| `name` | the node's name in the editor; a button child's name usually equals the button text |
| `id` | the live node this node stands for (from an export) — keeps its identity on publish |
| `paragraphs` | the message body: one string per paragraph, Telegram HTML inside. Alternatively `content.html` + `content.text` verbatim |
| `content` | type-specific keys (node-types.md). Strings may carry `{{placeholders}}` and `###tokens###` |
| `buttons` | keyboard rows: an entry is one button `{text, node?}` on its own row, or an array — one row of several. A button's `node` is what happens on tap: a step, or a *button kind* (`url`, `web_app`, `share`, `list_buttons`), or a `visibility` node with the kind under it |
| slots | one child per slot the type has — see below |

### Slots by type

| type | slots |
|---|---|
| `wait_for_input` | `answer` (index 0), `timeout` (1) |
| `condition`, `check_subscription` | `yes` (0), `no` (1) |
| `script` | `next` (0), `error` (1) |
| `web_app` | `next` (0, after sendData), `error` (1), `hooks` (series from 10: one entry per hook of the app, `null` for a hook without a branch) |
| `switch` | `branches` (series from 0: case i → branch i, the fallback last) |
| everything else | `next` (0) |

A slot the type does not have is refused: *"message «Hi» has no error slot"*.

### Cross-flow references

Inside `content`: `target_flow` (navigation `start_flow`) and `notify_flow` (notify) name a flow of the **document** by `key`. An export may leave `target_flow_id` / `notify_flow_id` — a numeric id of a flow outside the export; valid only on the same bot.

### Placeholders

`{{site_url}}` — this Lanka host; `{{bot.username}}`, `{{bot.public_key}}`; `{{owner_chat}}` — the owner's Telegram id (for `notify` → `target_chat_id`); `{{app_query}}` / `{{app_query_and}}` — `?scope=…` / `&scope=…` of the plugin instance, empty at the root instance. Anything else must be listed in `params` and passed in the write body as `"params": {"name": "value"}`.

Mini App url: `{{site_url}}/apps/<app>/{{bot.public_key}}{{app_query}}`.

## Write body

```json
{ "document": { … }, "params": { "site_url": "…" }, "scope": "clinic" }
```

`params.site_url` defaults to the Lanka host; `scope` defaults to the root instance.
