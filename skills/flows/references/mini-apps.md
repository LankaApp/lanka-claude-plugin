# Hosted Mini Apps

A Mini App is a page Lanka hosts and a `web_app` button opens inside Telegram. **Which kinds exist and what each one reads, returns and fires is in `generated.json` → `mini_apps`** (id, hooks in slot order, actions, and the app's own notes); this page is only how they plug into a flow.

| kind | panel | shows |
|---|---|---|
| `calendar` | Schedule | a month; only days with a free slot can be tapped |
| `week` | Schedule | a week of free hours |
| `shop` | Storefront | the product grid; `?mode=cart` is the visitor's cart |

## The bot's Mini Apps

A Mini App works for a bot only when the bot has it in that data space. `GET /api/v1/bots/:bot_id/mini_apps` lists them: `[{id, scope, title, url, hooks, actions, notes}]` — `id` is the kind, `url` the one to put in a `web_app` node. A document gets the ones it needs by naming the kinds in `"mini_apps"` (format.md): writing it creates them in the write's data space, and keeps the ones the bot already has. A template or a panel the owner creates brings its own.

The look (button colour, corners) is the owner's, set in the panel next to the Mini App's preview. There is no API for it; do not try to style a Mini App from a flow.

## Opening

```
{{site_url}}/apps/<kind>/{{bot.public_key}}{{app_query}}
{{site_url}}/apps/<kind>/{{bot.public_key}}?mode=cart{{app_query_and}}    # when the url already has a query
```

`{{app_query}}` carries the data space (`?scope=shafa`), empty in the main space.

## Data back into the flow

What the visitor picks comes back into `user.<save_to>` of the `web_app` node, and the flow continues to its `next`. From a `reply_keyboard` button that is Telegram's [`sendData`](https://core.telegram.org/bots/webapps#initializing-mini-apps); from an inline button, where Telegram gives no such channel, the hosted page posts the same payload to Lanka (`/apps/<kind>/<key>/submit`) and closes — the flow sees no difference.

## Hooks

An app may declare **hooks** — named moments (e.g. the week shown changed) its page reports through an action. The `web_app` node that opened the page may carry a branch per hook in `hooks[<position>]` (position = the hook's index in the app's `hooks` list; `null` for a hook without a branch). The branch runs right after the action's own script, with the values the page posted in `session.input`, and may contain `script` and `condition` nodes only — nothing that talks to the chat. Its writes are what the page reads back.

## Feeding an app

`calendar` and `shop` read their panel's data (data.md). `week` reads the calendar's free slots, or — when the hours come from elsewhere — the variables its `notes` name, filled by a `script` node before the button.
