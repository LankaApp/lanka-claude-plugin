# Hosted Mini Apps

A Mini App is a page Lanka hosts and a `web_app` button opens inside Telegram. **Which apps exist and what each one reads, returns and fires is in `generated.json` → `mini_apps`** (id, url, hooks in slot order, actions, and the app's own notes) — every plugin ships its apps with that contract; this page is only how they plug into a flow.

## Opening

```
{{site_url}}/apps/<app>/{{bot.public_key}}{{app_query}}
{{site_url}}/apps/<app>/{{bot.public_key}}?mode=cart{{app_query_and}}    # when the url already has a query
```

`{{app_query}}` carries the plugin instance (`?scope=kiosk`), empty at the root instance. The app's texts (its `config` keys) are configured on the plugin instance, not in the node; tokens work there.

## Data back into the flow

What the visitor picks comes back into `user.<save_to>` of the `web_app` node, and the flow continues to its `next`. From a `reply_keyboard` button that is Telegram's [`sendData`](https://core.telegram.org/bots/webapps#initializing-mini-apps); from an inline button, where Telegram gives no such channel, the hosted page posts the same payload to Lanka (`/apps/<app>/<key>/submit`) and closes — the flow sees no difference.

## Hooks

An app may declare **hooks** — named moments (e.g. the week shown changed) its page reports through an action. The `web_app` node that opened the page may carry a branch per hook in `hooks[<position>]` (position = the hook's index in the app's `hooks` list; `null` for a hook without a branch). The branch runs right after the action's own script, with the values the page posted in `session.input`, and may contain `script` and `condition` nodes only — nothing that talks to the chat. Its writes are what the page reads back.

## Feeding an app

Some apps read `app.*` data a plugin maintains (a catalog, a schedule); others read the **session** — then a `script` node before the button must fill the variables the app's `notes` name.
