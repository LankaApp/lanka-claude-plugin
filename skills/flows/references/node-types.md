# Node types

20 types. A node is `{type, name, paragraphs?, content?, buttons?, <slots>}` (format.md). `paragraphs` is the message body; `content` holds the keys below; tokens `###user.x###` render inside body, button labels, `media_url`, `notify.target_chat_id`. The machine-readable list of types, slots, operators and units is `generated.json` — trust it over this page if they differ.

## Steps that show something

| type | content | behaviour |
|---|---|---|
| `message` | `media_url` (photo url, tokens ok), `reactive` (bool) | Sends the body (as a photo caption when `media_url`). With buttons it waits for a tap; without, continues to `next`. |
| `reply_keyboard` | as `message` + `keyboard_persistent` (default `true` → `is_persistent`) | Same, with a [ReplyKeyboardMarkup](https://core.telegram.org/bots/api#replykeyboardmarkup) instead of an inline one — so no `url`/`share` buttons, and a `web_app` button can send data back ([KeyboardButton](https://core.telegram.org/bots/api#keyboardbutton)). |
| `wait_for_input` | `variable_name` (default `"user_input"`), `input_kind` ∈ `text` `contact` `location` `photo` `document` `web_app`, `button_text`, `web_app_url`, `timeout_value`, `timeout_unit` ∈ `seconds` `minutes` `hours`, `media_url`, `reactive` | Sends the prompt, halts until the visitor answers with the right kind; the answer lands in `user.<variable_name>`. `contact` → `{phone_number, first_name, last_name, user_id}`; `location` → `{latitude, longitude}`; `photo` → `{file_id}`; `document` → `{file_id, file_name, mime_type}`; `web_app` → what the app sent. `contact`/`location`/`web_app` show a one-time [request button](https://core.telegram.org/bots/api#keyboardbutton) labelled `button_text`. Slots `answer` (0) and `timeout` (1) — the timer arms only when a `timeout` child exists. |
| `poll` | `question`, `options` (strings), `save_to`, `allows_multiple_answers` | A [poll](https://core.telegram.org/bots/api#sendpoll), always public — Telegram reports answers only for those; halts until answered. Stores the chosen label (or labels) in `user.<save_to>`. |
| `placeholder` | body only (default «⏳ Хвилинку…») | Sends a message the **next** send edits in place — put it before a slow `script`. |
| `remove_keyboard` | body (may be empty) | Removes a reply keyboard; with text the message stays, without it is deleted right after. |
| `typing` | `seconds` (1–5) | Shows «typing…» then continues. |
| `delay` | `delay_value`, `delay_unit` ∈ `seconds` `minutes` `hours` | Pauses the visitor, resumes on `next`. |

## Steps that decide

| type | content | behaviour |
|---|---|---|
| `condition` | `variable_name`, `operator`, `compare_value` | Slots `yes` (0) / `no` (1). Operators below. |
| `switch` | `variable_name`, `cases` (strings) | Case *i* → `branches[i]`; no match → `branches[cases.length]` (the fallback, last). String equality. Never renumber existing branches. |
| `check_subscription` | `channel_chat_id`, `save_to` | `yes` (0) when the visitor is a member of the channel ([getChatMember](https://core.telegram.org/bots/api#getchatmember)), `no` (1) otherwise or when the check itself fails (the owner gets a report). The bot must be an admin of the channel. |
| `visibility` | `variable_name`, `operator`, `compare_value` | **First node under a button**: the button shows only while the rule holds. The button's real target — a step or a button kind — is its `next`. No `variable_name` = always visible. |

### Operators (`condition`, `visibility`)

`eq` `neq` — string equality (`1` equals `"1"`); `gt` `lt` `gte` `lte` — numeric; `contains` — substring / list member / object key; `is_set` — present (note: `false`, `""`, `[]` are **not** set); `is_null` — unset. `is_set`/`is_null` ignore `compare_value`.

`variable_name` here (and in `switch`, `remind`, `list_buttons.options_var`) may be `user.x`, `session.x`, `app.x`, `platform.x`, `initiator.x`, or a bare name, which reads session → user → platform. Nested paths and list indexes work: `user.cart.0.name`.

## Steps that act

| type | content | behaviour |
|---|---|---|
| `script` | `script` (safe-expr, see safe-expr.md) | Runs the script. Slots `next` (0) and `error` (1) — `error` fires only on a script failure (syntax, type, limit), **not** on a failed `fetch`: branch on `temp.r.status` yourself. Without an `error` child a failure shows «⚠️ Щось пішло не так…» and stops. |
| `navigation` | `action` ∈ `restart` `start_flow` `go_back` `delete_message` `delete_and_back`, `target_flow` (key; `start_flow`), `message_mode` ∈ `resend` `clear_history` | Terminal: moves the visitor. `clear_history` sweeps the sent messages first. |
| `notify` | `target_chat_id` (tokens ok — `{{owner_chat}}` for the owner), `media_url` (a photo, tokens ok), `interrupt_level` ∈ `when_free` (default) `urgent`, `notify_flow` (key of an `internal` flow) | Either sends the body to another chat rendered with the **visitor's** variables, or starts the internal notification flow there — it begins with the firing conversation's session variables as its own — where `###initiator.name###`, `###initiator.username###`, `###initiator.user_id###` and `###initiator.<any user var>###` are the visitor who fired it. `when_free` waits while the recipient is mid-conversation. Never breaks the visitor's flow. Continues to `next`. |
| `announce` | `audience_ids` (ids of the bot's audiences; empty — every visitor), `media_url` (a photo, tokens ok), `interrupt_level` ∈ `when_free` (default) `urgent`, `notify_flow` (key of an `internal` flow) | The mirror of `notify`: tells an audience of visitors. Either sends the body, rendered with the **firing visitor's** variables, or starts the internal announcement flow in every recipient's chat. That flow begins with the firing conversation's **session variables** as its own (kept once with the broadcast, so a copy delivered later still shows what was announced): set `announced = product` before the node, read `###session.announced.name###` inside. `###initiator.*###` is the visitor who fired it, live. Tokens resolve in a `url` button's address too, so `?start={{flow.card}}-###session.announced.id###` deep-links into a product. Recorded as a broadcast (status, stats) and delivered in the background; `when_free` waits for each busy recipient. Never breaks the visitor's flow. Continues to `next`. |
| `remind` | `base` (a variable holding a date/time; blank = now), `offset_value`, `offset_unit` ∈ `minutes` `hours` `days`, `offset_direction` ∈ `before` `after` | Schedules the body as a message at base ± offset and continues **immediately**. A moment in the past is skipped. |

## Button kinds — only as a button's `node`

| type | content | behaviour |
|---|---|---|
| `url` | `url` | An [inline button](https://core.telegram.org/bots/api#inlinekeyboardbutton) with `url`. Inline keyboards only. |
| `share` | `mode` ∈ `link` (default) `inline`, `share_url`, `share_text` | `link` → a `url` button to `t.me/share/url`; `inline` → [`switch_inline_query`](https://core.telegram.org/bots/api#inlinekeyboardbutton) (needs inline mode in @BotFather). Inline keyboards only. |
| `copy_text` | `text` (tokens allowed, ≤ 256 chars after rendering) | [`copy_text`](https://core.telegram.org/bots/api#copytextbutton) button: Telegram puts the text on the clipboard, the bot hears nothing. Inline keyboards only. |
| `web_app` | `url` (`{{site_url}}/apps/<app>/{{bot.public_key}}{{app_query}}`), `save_to` (default `web_app_data`) | Opens a hosted [Mini App](https://core.telegram.org/bots/webapps) (mini-apps.md). From a **reply** keyboard the app can [`sendData`](https://core.telegram.org/bots/webapps#initializing-mini-apps): the payload lands in `user.<save_to>` and the flow continues to `next` (0); an inline opening cannot send data. `hooks[]` (from index 10) hold the branches that run after the app's hook actions — `script`/`condition` only, the app's values in `session.input`. |
| `list_buttons` | `options_var` (a variable holding a list of strings), `save_to`, `buttons_per_row` (≥1) | The button expands into one button per option at send time. A pick writes `user.<save_to>` = the label and `user.<save_to>_index` = its position, then continues to `next`. This is how a choice from data is built — there is no choice node. |

A conditional link/app/list: `{ "text": "…", "node": { "type": "visibility", …, "next": { "type": "url", … } } }`.

## Where the runtime writes for you

`wait_for_input.variable_name`, `list_buttons.save_to` (+ `_index`), `poll.save_to`, `check_subscription.save_to`, `web_app.save_to` — all land in `user.*` (forever, per visitor). What a script computes should live in `session.*` unless it must outlive the conversation.
