# safe-expr — the script language

Scripts run in `script` nodes and Mini App actions. A script is **assignment statements only** — no control flow, no function definitions, no bare expressions. The function list with arities is in `generated.json`.

```
# a comment is a whole line starting with #
week = default(week, 0)
week_start = add_days(platform.now, week * 7)
free = filter(app.slots, item.status == "free" && item.date >= week_start)
day_labels = map(free, format(item.date, "dd.MM"))
user.visits = default(user.visits, 0) + 1
temp.resp = fetch("https://api.example.com/slots", {method: "POST", body: {date: week_start}, timeout: 10})
slots = temp.resp.status == 200 ? temp.resp.body.slots : []
```

## Syntax

- One statement per line: `target = expression`. A statement may continue on following lines while a bracket is open. No inline comments (`x = 1 # no`).
- Literals: numbers, `"strings"`, `true`/`false`/`null`, `[a, b]`, `{key: value}` (explicit keys only). Max 50 elements/keys per literal.
- Operators: `+ - * / %`, `> < >= <= == !=`, `&& || !`, `cond ? a : b`. `==` is strict (`1 == "1"` is false). `+` concatenates when either side is a string, otherwise both must be numbers.
- Indexing: `items[0]` or `items.0` — literal non-negative integers only. `list.size` / `object.size`.
- **No lambdas.** `filter`, `map`, `flat_map`, `find`, `update_where` take a plain expression as their second argument, evaluated per element with `item` bound: `filter(app.slots, item.status == "free")`. A collection call cannot be nested inside another's predicate.
- Reading an unset name gives `null` — use `default(x, fallback)` / `is_null(x)`.
- Limits: 50 statements, 4096 chars per script, 1024 chars per expression, paths ≤5 segments after the namespace, lists ≤500 items.

## Namespaces

| write | lives | read |
|---|---|---|
| `name = …` (bare) or `session.name = …` | the conversation (dies with it) | `name`, `session.name` |
| `user.name = …` | forever, per visitor | `user.name` |
| `app.name = …` | forever, bot-wide (plugin data lives here) | `app.name` |
| `temp.name = …` | this script only | `temp.name` |
| `platform.*` | read-only: `user_id`, `first_name`, `last_name`, `full_name`, `username`, `language_code`, `is_premium`, `message_text`, `now` (ISO timestamp) | |
| `params.*` | read-only, Mini App actions only | |
| `initiator.*` | read-only, inside an internal notification flow: the visitor who fired it | |
| `session.input` | in a Mini App hook branch: what the page posted | `input` |

- A bare read is **session only**; `resp.status` reads `session.resp.status`.
- Writing a nested path is refused (`user.cart.qty = 1`): rebuild the object, or use `update_where` on a list.
- Names starting with `_` are reserved.
- Rule of thumb: what the person entered persists (`user.*`), what a script computed lives with the conversation (bare).

## Functions (44)

- Math: `min(a,b,…)` `max(a,b,…)` `abs` `floor` `ceil` `round(x, decimals?)` `sqrt` `pow(x,y)` `clamp(v, lo, hi)`
- String: `len` (string or list) `upper` `lower` `trim` `contains(s, sub)` `starts_with` `ends_with` `substr(s, start, len?)` `replace(s, from, to)` (all occurrences)
- Conversion: `num(x)` (throws if not numeric) `to_num(x, fallback)` `str(x)` `bool(x)`
- Checks: `is_num` `is_str` `is_bool` `is_null` (true for unset too)
- Dates (ISO strings in and out; input must start with `YYYY-MM-DD`): `add_days(date, n)` → `"yyyy-MM-dd"`, `get_iso_day(date)` → 1 (Mon) … 7 (Sun), `format(date, pattern)` — date-fns patterns, e.g. `"dd.MM"`, `"HH:mm"`
- Utility: `default(x, fallback)` — fallback only for null/unset (`0` and `""` pass through)
- Lists: `count` `sort` `unique` `join(list, sep)` `append(list, item)` `sort_by(list, "key", "asc"|"desc")` `take(list, n)` `sum` `nth(list, i)` (out of range → `null`)
- Lists with `item`: `filter(list, pred)` `map(list, expr)` `flat_map(list, expr)` `find(list, pred)` (element or `null`) `update_where(list, pred, "key", value)`

## fetch

```
temp.resp = fetch("https://…", {method: "POST", headers: {…}, body: {…}, cache: 60, timeout: 10})
```

- Only as the **entire right-hand side** of an assignment, and the target must be named: `temp.x` (dropped when the script ends) or `session.x` (kept). A bare target is refused.
- `body`: an object is JSON-encoded; a string is sent as is. `cache`: seconds (≤3600), only 2xx answers are cached, shared between visitors for an identical request — for a schedule or a catalogue, never a booking. `timeout`: seconds, default 5, max 30.
- Result: `{status, body, error}`. `body` is parsed JSON or the raw string; on a transport failure `status` is `null` and `error` says why.
- **A failed request is data, not an error**: it does not take the script node's `error` branch. Check `temp.resp.status == 200` or `is_null(temp.resp.error)` and branch with a `condition`.
- Neighbouring fetch lines that do not read each other's results run in parallel (up to 6 per script).

## Tokens in messages

`###user.name###`, `###session.total###`, `###app.catalog.0.name###`, `###platform.first_name###`, `###initiator.username###`. Always with a namespace — `###score###` does not resolve. No expressions, no formatting: compute in a script, then render. `null` renders as empty, objects/lists as JSON.

Telegram HTML in bodies: `<b> <i> <u> <s> <code> <pre> <a href="…"> <span class="tg-spoiler"> <blockquote>`.
