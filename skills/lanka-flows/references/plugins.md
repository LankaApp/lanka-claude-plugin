# Plugins and `app.*` data

A plugin is a visual editor over `app.*` variables — the data stays in ordinary bot variables that scripts, tokens, `list_buttons` and Mini Apps read. A bot may run several **instances** of one plugin; instance `clinic` keeps its data under `app.clinic.*`, the root instance under `app.*` itself.

**Write `app.*` in a document and pass `scope` in the write body** — the server re-roots every `app.` to `app.<scope>.` and fills `{{app_query}}`. Export with `?scope=<scope>` to get portable `app.*` back.

| plugin | variables | shape |
|---|---|---|
| `catalog` | `app.catalog` | `[{ "id": "sneakers", "name": "…", "price": 1200, "image": "https://…", "description_html": "…", "description_text": "…" }]` — stable string ids (they end up in deep links `?start=product_<id>`), numbers as numbers |
| `schedule` | `app.schedule`, `app.slots` | `app.schedule = { "slot_minutes": 30, "days": { "2026-09-07": [{ "start": "09:00", "end": "13:00" }] } }`; `app.slots` is generated from it: `[{ "date": "2026-09-07", "time": "09:00", "status": "free" \| "booked", … }]` — a booking flow marks a slot booked with `update_where` |

Rules for data flows read: lists of flat objects, stable string ids, numbers as numbers, rich text as an `html` + `text` pair, depth ≤ 5.

`seed_app_variables` in a document writes initial values only where the instance has none — the plugin owns the data afterwards.

## Deep links

`https://t.me/{{bot.username}}?start=<payload>` arrives as the message `/start <payload>` — it does not match the `/start` command, so route it with a `text_match` flow (`"value": "/start product_"`) and read the payload with `replace(platform.message_text, "/start product_", "")`. Bare `/start` stays a `command` flow and wins first.
