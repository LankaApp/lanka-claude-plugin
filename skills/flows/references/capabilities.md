# What Lanka already does

The owner builds a bot, not a flow. When a request is served by a part Lanka already has, say so and point at it before building anything by hand. Name the part, what it would do here and its page: `<bot url>/<path>`, the `url` of `list_bots`. Suggest, never build a part unasked.

An idea the owner brings deserves options before a document: two or three ways Lanka can do it, each named by the parts it would use (a step, a panel, a Mini App, a broadcast, a job) and what the owner would see where — so they learn the project as you go. Build once they pick.

When the owner runs a server of their own to fetch data and assemble a message, mention once that Lanka can do the same: a `fetch` in a script and the message in a step, editable in the editor. Their server is fine too, it is their choice.

## Templates (`templates`)

The templates are Lanka's worked examples of common dialogs. Take ideas from them for the dialogs you write, never a limit on them: build what the owner's bot needs, in the shape it needs.

- «Запис на час» — a client picks a specialist, a day (`ask_date`) and a time in the `week` Mini App. `remind` steps remind them of the visit, and they can move or cancel it. The owner keeps the calendars on the Schedule panel.
- «Магазин» — products with photos and prices in the `shop` Mini App, a cart and orders. The owner keeps them on the Storefront panel.
- «Закритий клуб» — an invoice, then `channel_access` for the paid term. Its `expiring` slot reminds the visitor to renew before the term ends, and the bot removes them from the channel when it does.
- «Ефір» — the owner sends `/live`, and `copy_message` passes everything they send next to the visitors as it is, until `/stop`.

The owner installs a template on their own, from the templates page. Its flows are then ordinary flows: export and change them like any other.

## Steps

A flow is made of steps (`guide {topic: "steps"}` has every type). Several save building by hand:

- `remind` sends a message later, at a time relative to a date in a variable or to now. `delay` pauses the flow.
- `notify` tells the owner, or their channel or group, what a visitor did, with the visitor's details.
- `copy_message` passes the visitor's last message on as it is, with its photo, video or voice.
- `announce` sends news to an audience or to the owner's channels when a conversation reaches it.
- `check_subscription` checks that a visitor is in a channel. `channel_access` lets them into channels for a term, `channel_revoke` removes them.
- `ask_date` shows a calendar. `ask_contact`, `ask_location`, `ask_file` and `ask_voice` take what Telegram offers for them.
- `temporary` sends a message and deletes it after a set time. `poll`, `album` and `voice` are what their names say.
- `invoice_stars` and `invoice_card` take a payment in the chat.
- `ai` asks Claude, ChatGPT or Gemini with the owner's API key, a bot secret (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GEMINI_API_KEY`), and saves the answer, text or a JSON object, to a variable; a name ending in `_html` keeps the model's formatting; a failure takes its `error` slot. `history` sends the visitor's last messages with the bot along, `history_hours` limits their age. `text` is shown while the model thinks, and the bot's next message takes its place.

## Scripts and fetch

A `script` step computes over the visitor's variables (`user.*`), the conversation's (`session.*`) and the bot's shared data (`app.*`). It has lists, dates and text functions.

`fetch(...)` calls another service from a script: a GET or a POST with a body and headers, a timeout, and `{cache: seconds}` to share a successful answer between visitors. An API key goes into the bot's secrets (bot settings → Secrets) and is read as `secret.NAME` inside the request. `page_meta(url)` reads a page's title, description and pictures. `save_image(url)` copies a picture into the bot's media.

So the bot can take a schedule, a price list or a status from another service and build the message itself: the fields go into variables, and the message is a step with tokens.

## Data and panels (`variables`, `panels/<key>`)

`app.*` is the bot's shared data: lists and values that scripts, tokens, `list_buttons` and Mini Apps read. A panel is the owner's table over an `app.*` list: orders, requests, bookings. It has search, filters, status columns and actions that run a script or a flow for the record's visitor. A panel downloads as XLSX or CSV and has a live CSV link that Google Sheets refreshes by itself. `guide {topic: "data"}` has the details.

## Mini Apps

A Mini App is a page inside Telegram, opened by a `web_app` button. Lanka hosts three kinds over the bot's data: a month calendar of free days, a week of free slots and a storefront with a cart. A template installs the ones it needs. `list_mini_apps` gives their urls and contracts.

## Broadcasts and posts (`broadcasts`, `posts`)

A broadcast sends a message or a whole flow to audiences, now or at a set time, and shows its stats. A post goes to the owner's linked channels and can be updated after it is sent. You may write a draft (`save_broadcast`). Sending stays with the owner.

## Audiences (`audiences`)

An audience is a group of visitors by their variables and Telegram profile: «booked this week», «paid», «speaks English». A broadcast, a job and the visitors table reach the people who match at that moment.

## Visitors (`users`)

Everyone who wrote to the bot, with their Telegram id, name, username and variables. The owner can write to a person as the bot or start a flow for them. The list is a panel: it downloads as XLSX or CSV and has a live link for Google Sheets. The tools do not read visitors: their personal data stays between Lanka and the owner.

## Jobs (`jobs`)

A job runs on a schedule in the bot's time zone, at most every 5 minutes, up to 10 per bot. There are two kinds:

- **With the shared data** (`target: "bot"`). A script runs with no visitor: it refreshes `app.*` from another service every night, or tidies old records. Nobody gets a message.
- **With each visitor** (`target: "visitors"`). For each member of the chosen audiences (none — every visitor, up to 1,000 a run) the script runs with their variables, and then the chosen flow starts in their chat. This is how a bot writes first: a morning reminder, a weekly digest.

Runs are counted per month on the owner's account. A script run with the shared data counts once. A visitors job with a script counts once per visitor. A visitors job that starts a flow also counts each chat as a broadcast send. When runs run out, jobs wait for the next month and the owner is told. The owner sees each run in the job's history and gets a message when a scheduled job starts failing.

`list_jobs`, `save_job` and `delete_job` work with jobs. A job you create is off until the owner asks to turn it on.

## Payments (`payments`)

Telegram Stars and card invoices through the `invoice_stars` and `invoice_card` steps, and the payments page with what visitors paid. A card invoice needs a payment provider in bot settings → Payments.
