# Hosted Mini Apps

A Mini App is a page Lanka hosts and a `web_app` button opens inside Telegram. URL in a document:

```
{{site_url}}/apps/<app>/{{bot.public_key}}{{app_query}}
{{site_url}}/apps/<app>/{{bot.public_key}}?mode=cart{{app_query_and}}    # when the url already has a query
```

`{{app_query}}` carries the plugin instance (`?scope=clinic`), empty at the root instance.

What the visitor picks comes back through `sendData` into `user.<save_to>` of the `web_app` node, and the flow continues to its `next`. **Only a button on a `reply_keyboard` can send data back**; an inline button just opens the app. Some apps also have **hooks**: after the page calls one of the app's actions, the branch under the `web_app` node at `hooks[<position>]` runs (scripts and conditions only) with the page's values in `session.input`.

| app | plugin | reads | returns via `sendData` | hooks |
|---|---|---|---|---|
| `shop` | `catalog` | `app.catalog` (products); the cart lives in `user.cart`, `user.cart_total`, `user.cart_names` | `{ "product_id": "…" }` from the storefront; `{ "checkout": true }` from `?mode=cart` | — |
| `calendar` | `schedule` | `app.slots` — days with a `status: "free"` slot are tappable | `{ "date": "2026-09-07" }` | — |
| `week` | `schedule` | the **session**: `week_slots`, `week_start`, `day_offset` — a script before the button must fill them | `{ "date": "…", "slot": <value from week_slots> }` | `hooks[0]` = `week_changed`, fired by the page's `week_get`/`week_shift` (`session.input.delta` = ±7 on shift) |

## week — the contract

```
week_slots = [ {date: "2026-09-07", label: "09:20", value: {id: "slot-42", start: "2026-09-07T09:20:00"}}, … ]
week_start = "2026-09-07"     # ISO date of the first shown day
day_offset = 0                # days from today
```

`value` is opaque — put whatever the next step needs into it; the pick returns it whole. The page's texts (`title`, `subtitle`, `empty_text`) are configured on the plugin instance, not in the node; tokens work there (`###session.service###`).

A typical week step:

```json
{ "type": "script", "name": "Тиждень", "content": { "script": "day_offset = default(day_offset, 0)\nweek_start = add_days(platform.now, day_offset)\n…week_slots = …" },
  "next": { "type": "reply_keyboard", "name": "Оберіть час", "paragraphs": ["Оберіть вільний час:"],
    "buttons": [ { "text": "📅 Відкрити тиждень", "node": {
      "type": "web_app", "name": "Тиждень", "content": { "url": "{{site_url}}/apps/week/{{bot.public_key}}{{app_query}}", "save_to": "week_pick" },
      "next": { "type": "script", "name": "Обраний слот", "content": { "script": "chosen = user.week_pick.slot" } },
      "hooks": [ { "type": "script", "name": "Тиждень змінено", "content": { "script": "day_offset = default(day_offset, 0) + default(input.delta, 0)\nweek_start = add_days(platform.now, day_offset)\n…" } } ]
    } } ] } }
```

The shop template (`catalog/shop.uk`) and the booking template (`schedule/booking.uk`) shipped with Lanka are complete, working examples of `shop` and `calendar` — export a bot that has them installed to read them.
