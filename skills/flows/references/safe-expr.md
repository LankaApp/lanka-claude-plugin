# safe-expr — the script language

Scripts run in `script` nodes, jobs, panel actions and Mini App actions. The language is Lanka's own — not JavaScript, not Python. A script is **assignment lines only**: `target = expression`. No control flow, no function definitions, no bare expressions. The function list with arities is in `generated.json` (`guide` topic `steps`).

```
# a comment is a whole line starting with #
week = default(week, 0)
week_start = add_days(platform.today, week * 7)
calendar = find(app.calendars, item.id == calendar_id)
free = filter(calendar.slots, item.status == "free" && item.date >= week_start)
day_labels = map(free, format(item.date, "dd.MM"))
user.visits = default(user.visits, 0) + 1
temp.resp = fetch("https://api.example.com/slots", {method: "POST", body: {date: week_start}, timeout: 10})
slots = temp.resp.status == 200 ? temp.resp.body.slots : []
```

## What other languages write, and what goes here

| not this | this |
|---|---|
| `if x == "y" then z = 1`, `else`, a loop | `z = x == "y" ? 1 : z`. A branch of the dialog is a `condition` or `switch` step after the script; a loop is `map` / `filter` |
| `a and b`, `a or b`, `not a` | `a && b`, `a \|\| b`, `!a` |
| `filter(list, "item.status == 'free'")`, `filter(list, x => x.ok)` | `filter(list, item.status == "free")` — a bare expression, `item` is the element |
| `name.toUpperCase()`, `list.length` | `upper(name)`, `count(list)` or `list.size` — functions only, no methods |
| `now`, `now()`, `today()` | `platform.today` (the date) and `platform.time` (`HH:mm`) on the bot's clock; `platform.now` is the UTC timestamp |
| `format(day, "YYYY-MM-DD")` | `format(day, "yyyy-MM-dd")` — date-fns tokens |
| `list[i]`, `list[count - 1]` | `nth(list, i)` — positions count from 0, here and in `substr` and `slice` |
| `let x = 1`, `x += 1` | `x = 1`, `x = x + 1` |
| `x = 1 # note`, `// note` | a comment on a line of its own, starting with `#` |

Validation (`validate_flows`, a save) checks the syntax, the function names, their arities and a date pattern written as a literal — not the names. A bare `now` or `list.length` passes it and reads `null` when the script runs: an unset name is `null`, not an error. A name that is neither assigned above nor set by an earlier step is a slip to look for by eye.

## Syntax

- One statement per line: `target = expression`. A statement may continue on following lines while a bracket is open.
- Literals: numbers, `"strings"` or `'strings'`, `true`/`false`/`null`, `[a, b]`, `{key: value}` (explicit keys only). Max 50 elements/keys per literal.
- Template strings: `` `${item.name} ×${item.qty}` `` — any expression inside `${…}`, spelled out as `str()` would; `null`/unset fills in as `""`. The readable way to build text per item: `join(map(record.items, `${item.name} ×${item.qty}`), "\n")`.
- Operators: `+ - * / %`, `> < >= <= == !=`, `&& || !`, `cond ? a : b`. `==` is strict (`1 == "1"` is false; there is no `===`). `+` concatenates when either side is a string, otherwise both must be numbers.
- Indexing: `items[0]` or `items.0` — literal non-negative integers only; a computed position is `nth(list, i)`. `list.size` / `object.size`.
- **No lambdas.** `filter`, `map`, `flat_map`, `find`, `update_where` take a plain expression as their second argument, evaluated per element with `item` bound — `item.field` of an object, bare `item` of a string or a number: `filter(app.orders, item.status == "new")`. Such a call may be another's list or value — `map(filter(list, item.ok), item.name)` — never a part of its predicate: there is one `item` at a time. What an inner list needs from the outer element goes through a line of its own — `temp.cal = find(app.calendars, item.id == calendar_id)`, then `filter(temp.cal.slots, item.status == "free")`.- Reading an unset name gives `null` — use `default(x, fallback)` / `is_null(x)`.
- Limits: 50 statements, 4096 chars per script, 1024 chars and about 100 terms per expression (split a longer one over lines with `temp.*`), paths ≤5 segments after the namespace, lists ≤500 items.

## Namespaces

| write | lives | read |
|---|---|---|
| `name = …` (bare) or `session.name = …` | the conversation (dies with it) | `name`, `session.name` |
| `user.name = …` | forever, per visitor | `user.name` |
| `app.name = …` | forever, bot-wide (panels show and edit it) | `app.name` |
| `temp.name = …` | this script only | `temp.name` |
| `record.name = …` | in a panel action: the record the button acts on (data.md § Table panels) | `record.name` |
| `platform.*` | read-only: the visitor's Telegram profile, the message being handled and the clock — the table below | |
| `initiator.*` | read-only, inside an internal notification flow: the visitor who fired it | |
| `secret.*` | read-only, only inside `fetch(...)`, `page_meta(...)` and `save_image(...)` (url, headers, body): the bot's secrets, set by the owner in the bot's settings — `headers: {Authorization: "Bearer " + secret.API_KEY}`. The value goes in only at request time and is replaced by `‹secret.NAME›` in what comes back; a script that reads `secret.*` anywhere else is refused | |
| `session.input` | in a Mini App hook branch: what the page posted | `input` |

- A bare read is **session only**; `resp.status` reads `session.resp.status`.
- A dotted target writes inside an object, making the objects on the way: `user.cart.qty = 1` keeps the rest of `user.cart`. An element of a list is changed with `update_where`, never by its index.
- `app.*` is shared by every visitor, and two chats changing it at once are safe: a script runs as steps between its `fetch` lines, each step reads the current `app.*` and, if someone wrote it meanwhile, is recomputed over the new data before its own write — both `append`s to `app.orders` land. Nothing to do on the author's side.
- Names starting with `_` are reserved, and so is `item`.
- Rule of thumb: what the person entered persists (`user.*`), what a script computed lives with the conversation (bare).

### platform.*

| name | example | |
|---|---|---|
| `platform.user_id` | `123456789` | a number: the visitor's Telegram id, also their private chat with the bot |
| `platform.first_name` | `"Оля"` | |
| `platform.last_name` | `"Коваль"` | `null` when the profile has none |
| `platform.full_name` | `"Оля Коваль"` | the names there are; `null` when neither — `default(platform.full_name, "гість")` |
| `platform.username` | `"olya_k"` | no `@`; `null` when the profile has none |
| `platform.language_code` | `"uk"` | the language of the visitor's Telegram |
| `platform.is_premium` | `false` | |
| `platform.first_seen`, `platform.last_seen` | `"2026-11-05T10:30:00Z"` | UTC |
| `platform.message_text` | `"Привіт"` | the text of the message being handled |
| `platform.start_param` | `"42"` | the data after the dash of a deep link, only while that `/start` is handled — the conversation it opened keeps it as `session.start_param` |
| `platform.now` | `"2026-11-05T10:30:00Z"` | the moment, UTC: a stamp on a record, compared with other stamps as it is |
| `platform.today` | `"2026-11-05"` | the date on the bot's clock — the time zone of the bot's settings |
| `platform.time` | `"15:30"` | the time on the bot's clock, `HH:mm`; compares with `>` `<` as it is |

A job has no visitor: its `platform.*` is the clock alone. What a step stores for the visitor's answer — a contact, a location, a file — is an object; its shape is in that step's description (`guide` topic `steps`).

## Functions

- Math: `min(a,b,…)` `max(a,b,…)` `abs` `floor` `ceil` `round(x, decimals?)` `sqrt` `pow(x,y)` `clamp(v, lo, hi)`
- String: `len` (string or list) `upper` `lower` `trim` `contains(s, sub)` `starts_with` `ends_with` `substr(s, start, len?)` (from 0: `substr("2026-11-05", 5, 2)` is `"11"`) `replace(s, from, to)` (all occurrences) `split(s, sep)` → list of pieces (`nth(split(url, "/"), 4)`)
- Conversion: `num(x)` (throws if not numeric) `to_num(x, fallback)` `str(x)` `bool(x)`
- Checks: `is_num` `is_str` `is_bool` `is_null` (true for unset too)
- Dates (ISO strings in and out; input must start with `YYYY-MM-DD`): `add_days(date, n)` → `"yyyy-MM-dd"`, `get_iso_day(date)` → 1 (Mon) … 7 (Sun), `format(date, pattern)` — date-fns patterns: `"dd.MM"`, `"yyyy-MM-dd"`, and `"HH:mm"` when the string carries a time (`"2026-11-05T15:30"`; without one it is `00:00`). The time is printed as the string writes it, in no zone: `format(platform.now, "HH:mm")` is UTC, the bot's own clock is `platform.time`. Dates written `yyyy-MM-dd` and times written `HH:mm` compare with `>` `<` `==` as they are: a slot still ahead is `item.date > platform.today || (item.date == platform.today && item.time > platform.time)`
- Utility: `default(x, fallback)` — fallback only for null/unset (`0` and `""` pass through)
- Lists: `count` `sort` `unique` `join(list, sep)` `append(list, item)` `sort_by(list, "key", "asc"|"desc")` `take(list, n)` `slice(list, start, count)` (a page, 0-based) `last(list, n)` (the tail, in order — the newest of an append-only list) `sum` `nth(list, i)` (from 0; out of range → `null`)
- Lists with `item`: `filter(list, pred)` `map(list, expr)` `flat_map(list, expr)` (a list per element, joined into one: `flat_map(app.calendars, item.slots)` is every slot of every calendar) `find(list, pred)` (element or `null`) `update_where(list, pred, {key: value, …})` — the matching objects with these fields set, the rest of each kept; the values are computed once, outside `item`. `update_where(list, pred, "key", value)` sets one field

## A record inside a list of a list

A slot of one calendar, an item of one order: change the inner list first, then put it back into its owner.

```
temp.cal = find(app.calendars, item.id == calendar_id)
temp.slots = update_where(temp.cal.slots, item.date == chosen_date && item.time == chosen_time, {status: "booked", by: platform.full_name, user_id: platform.user_id})
app.calendars = update_where(app.calendars, item.id == calendar_id, {slots: temp.slots})
```

## fetch

```
temp.resp = fetch("https://…", {method: "POST", headers: {…}, body: {…}, cache: 60, timeout: 10})
```

- Only as the **entire right-hand side** of an assignment, and the target must be named: `temp.x` (dropped when the script ends) or `session.x` (kept). A bare target is refused.
- `body`: an object is JSON-encoded; a string is sent as is. `cache`: seconds (≤3600), only 2xx answers are cached, shared between visitors for an identical request — for a schedule or a catalogue, never a booking. `timeout`: seconds, default 5, max 30.
- Result: `{status, body, error}`. `body` is parsed JSON or the raw string; on a transport failure `status` is `null` and `error` says why.
- A `fetch` line pauses the flow on the script node while the request runs in the background (the visitor sees «typing…»); the script resumes at the next line. Nothing changes for the author.
- **A failed request is data, not an error**: it does not take the script node's `error` branch. Check `temp.resp.status == 200` or `is_null(temp.resp.error)` and branch with a `condition`.
- Neighbouring fetch lines that do not read each other's results run in parallel (up to 6 per script).

## page_meta and save_image

Two more outbound lines with the same rules as `fetch` (whole line, named `temp.`/`session.` target, a failure is data, same address filter and timeouts), one argument each:

```
temp.page = page_meta(link)
temp.photo = save_image(temp.page.image)
app.catalog = append(app.catalog, {id: "p1", name: temp.page.title, image: temp.photo.url})
```

- `page_meta(url)` → `{status, title, description, image, images, url, site, type, extra, error}` — what the page says about itself in its Open Graph / meta tags; `images` is every picture the page shows on the same host as `og:image` (a shop's product photos, up to 10, the main one first); `extra` holds non-standard ones (`brand`, `availability`). Redirects are followed. Nothing else of the page reaches the script.
- `save_image(url)` or `save_image(list)` → `{url, urls, error}` — the picture(s) copied into the bot's own media (up to 10; `url` is the first) (images and video up to 20 MB, type read from the bytes). It stores only in a live dialog: the editor's dry run checks the file and answers with the source url, a Mini App action refuses. Never auto-run by the editor.

## Tokens in messages

`###user.name###`, `###session.total###`, `###app.catalog.0.name###`, `###platform.first_name###`, `###initiator.username###`. Always with a namespace — `###score###` does not resolve. No expressions, no formatting: compute in a script, then render. `null` renders as empty, objects/lists as JSON.

Bodies are [Telegram HTML](https://core.telegram.org/bots/api#html-style); Lanka passes `<b> <i> <u> <s> <code> <pre> <a href="…"> <span class="tg-spoiler"> <blockquote>` and drops the rest.
