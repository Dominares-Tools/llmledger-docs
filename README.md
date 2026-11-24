# llmledger — Meter AI usage and stop calls past the allowance

**Paid tool: $99 once (plus sales tax where it applies), then a private GitHub repository you're invited to, with every update. Buy: https://dominares.org/tools/llmledger**

Commercial license v1.3 — see https://dominares.org/license · Support: support@dominares.org

llmledger records each model call to an append-only monthly ledger and checks a customer's plan allowance before the next call is made. It reads token counts from OpenAI, Gemini and Anthropic responses, prices them from a table you supply, and needs nothing beyond the standard library.

## Get it

Buy llmledger at https://dominares.org/tools/llmledger: one payment of $99, plus sales tax where it applies, and every future update. After checkout you are invited to a private GitHub repository with the source and every version published to it. The repository includes the Python library and the llmledger command, with an example price table, a config file and a runnable quickstart.

Requirements: The Python standard library. The [tz] extra adds tzdata, and [toml] adds TOML config on Python 3.10.

This page documents version 1.0.3. Install commands and file paths in the documentation below run inside the private repository you are invited to; the source is not in this repository.

## Documentation

**Meter every customer's AI usage and stop the call before it goes over their allowance.**

You sell a subscription with AI features in it: "500 AI replies a month", "$5 of AI included",
"100 call minutes". Somewhere in your code a model gets called on a customer's behalf, and your
provider bills YOU for every token. Without a meter:

- a customer on the $10 plan can run up $300 of model spend, and you find out from the invoice;
- usage counted from chat history resets when the customer clears their chats;
- one runaway script (or one abusive account) can spend a week's budget before lunch;
- your provider's dashboard shows one number for your whole account, not who spent it.

llmledger is a small library you call around each model call:

- **Before the call**, `admit(customer, plan)` answers "may this customer make another call?"
  from an append-only ledger, so the call over the allowance is never made.
- **After the call**, `record(...)` (or `record_response(resp)`) appends one line with the
  customer, model, token counts and cost. Never the prompt, never the reply.
- **Allowances** per plan: monthly dollars, tokens or calls, a daily dollar guard, and
  non-token units such as call minutes.
- **Alerts** at 80% and 100% of any allowance, once per customer per month, through a callback
  you supply (send the email, post to your chat, open a ticket).
- **A daily spend ceiling** across all customers: a backstop for the day something goes wrong.
  A misspelled ceiling is an error at start-up, never "no ceiling".
- **A report** per customer, model and kind: `llmledger report --month 2026-09`.

It reads token counts straight from OpenAI (Chat Completions and the Responses API), Gemini and
Anthropic responses, as JSON dicts or SDK objects. Prices come only from a table you write:
llmledger ships no prices, because any price built into a library is wrong for someone.

**Who it is for:** teams that bundle LLM features into a subscription and need per-customer
quotas and cost visibility that customers cannot reset. It is a drop-in library, not a service:
no server, no account, no network calls. The runtime is the Python standard library only
(3.10 or newer).

### Install

*The install commands are in the README of the private repository you are invited to when you buy. The notes on them are kept here.*

Installing it adds the package and the `llmledger` command. Optional extras:
`python3 -m pip install './llmledger[tz]'` adds a time zone database for systems without one
(Windows, some slim containers), and `'./llmledger[toml]'` reads TOML files on Python 3.10.

### 60-second quickstart

From the same folder:

```bash
python3 llmledger/examples/quickstart.py
llmledger report --dir quickstart-usage
llmledger prices check llmledger/examples/prices.example.json
```

The first command records five calls for a customer on a plan of 5 calls a month, reading token
counts from an OpenAI-, an Anthropic- and a Gemini-shaped response. It prints the 80% and 100%
alerts as they fire, and the sixth call is refused before any model would run:

```
call 1: recorded replace-me-small-model, 1200 in / 300 out, $0.000900
call 2: recorded replace-me-large-model, 2050 in / 400 out, $0.033500
call 3: recorded replace-me-small-model-2026-01, 800 in / 240 out, $0.001280
  ALERT: acme is at 80% of monthly_calls (4 of 5)
call 4: recorded replace-me-small-model, 1200 in / 300 out, $0.000900
  ALERT: acme is at 100% of monthly_calls (5 of 5)
call 5: recorded replace-me-small-model, 1200 in / 300 out, $0.000900
call 6: REFUSED before calling the model: refused: the plan's monthly calls are used up (monthly_calls 5 of 5)
```

Run it again and every call is refused: the allowance is used up for the month, and nothing
short of deleting the ledger itself hands it back. The report prints usage and cost per customer
and per model. `prices check` exits 1 and names the three placeholder rows in the example price
table: that is the point of the check.

### Use it in your code

```python
from llmledger import Ledger, Plan, PriceTable, AllowanceExceeded

ledger = Ledger(
    "/var/lib/myapp/llm-usage",               # a directory; many processes may share it
    timezone="UTC",                           # where months and days begin (required)
    prices=PriceTable.load("prices.json"),    # YOUR rates, see "Prices" below
    plans=[
        Plan("basic", monthly_calls=500, monthly_usd=5.00, daily_usd=1.00),
        Plan("team", monthly_usd=40.00, monthly_units={"call_seconds": 6000}),
    ],
    daily_ceiling_usd=50,                     # all customers together, per day
    on_alert=send_usage_email,                # called with an Alert, once per threshold
    plan_of=lambda customer_id: my_db.plan_name_for(customer_id),
)
```

**Admit, call, record.** The simplest pattern is `guard`, which admits on entry (raising
`AllowanceExceeded` so the provider is never called) and records on exit:

```python
try:
    with ledger.guard(customer_id, call_id=request_id, feature="summarize") as call:
        resp = client.chat.completions.create(model="your-model", messages=messages)
        call.response(resp)
except AllowanceExceeded as refused:
    return polite_limit_reply(refused.admission)     # .reason, .used, .limit, .resets_at
```

Or do it by hand:

```python
admission = ledger.admit(customer_id)                # plan from plan_of, or pass one
if not admission:
    return polite_limit_reply(admission)
resp = client.messages.create(...)                   # any provider
ledger.record_response(customer_id, resp, call_id=request_id)
```

`record_response` reads OpenAI, OpenAI Responses, Gemini and Anthropic responses (dicts or SDK
objects). When you have the counts another way, record them directly:

```python
ledger.record(customer_id, model="your-model", input_tokens=1200, output_tokens=300,
              cached_input_tokens=1000, kind="chat", call_id=request_id, region="eu")
```

`input_tokens` is all the input, including cached tokens and tokens written to a prompt cache
(`cache_write_tokens`). Extra keyword arguments are short tags (at most 200 characters each).
Tag names that suggest content are refused: any name containing `prompt`, `message`, `content`,
`reply`, `response`, `completion`, `question` or `answer`, plus `text`, `input`, `output`, `body`
and `system`.

**Idempotent.** Pass the same `call_id` twice for one customer (a retry, or two workers seeing the
same call) and it is counted once; `record` returns `None` for the duplicate. The id is scoped to
the customer, so two customers may use the same id. Appends are serialized with a file lock, so
this holds across processes, not just threads.

**Estimates.** `admit(customer, plan, estimate=0.02, estimate_tokens=4000)` also refuses a call
that would pass a limit, not only one that is already over it. Without an estimate, a call is
admitted while usage is under every limit, and the call that crosses a limit completes. A call
refused only because of its estimate has a reason starting `would_exceed_` (for example
`would_exceed_monthly_usd`), and `admission.message` says the call would pass the limit rather
than that the allowance is used up.

**Kinds.** `kind` labels what a call was for ("chat", "search", "summarize"). A plan with
`kinds=["search"]` counts only those calls, so a plan can carry separate allowances: call `admit`
once per allowance that applies.

**Usage by customer.** `ledger.usage(customer_id)` returns this month's calls, tokens, cost,
today's spend, and breakdowns by kind, model and day. `ledger.report("2026-09")` covers every
customer. `ledger.spend_today()` is the all-customer figure the ceiling checks.

**Units (non-token usage).** Call minutes, pages rendered, searches: anything with an event id.

```python
ledger.units.record(customer_id, unit="call_seconds", event_id=call_sid, quantity=1790,
                    source="media_server", at=call_started_at)
ledger.units.record(customer_id, unit="call_seconds", event_id=call_sid, quantity=1800,
                    source="carrier_webhook")          # same call, second writer
ledger.units.used(customer_id, "call_seconds")          # 1800: counted once, larger figure

adm = ledger.units.admit(customer_id, "team", unit="call_seconds", reserved=seconds_held_by_live_calls)
max_call_seconds = adm.remaining
```

Each event counts once whatever the number of writers, at the largest quantity and cost any
of them reported, in the month and day it started. `reserved` stops concurrent events from
jointly overrunning the allowance. A plan that does not list a unit includes none of it.
Unit costs come from the price table's `units` section (dollars per unit) or `cost_usd=`, and
count toward the daily ceiling.

**Alerts.** `on_alert` receives an `Alert` with `kind` (`"allowance"` or `"daily_ceiling"`),
`customer`, `plan`, `metric` (`monthly_usd`, `monthly_tokens`, `monthly_calls`,
`units:<unit>`, `platform_usd_today`), `threshold`, `percent`, `used`, `limit`, `period` and
`resets_at`. Each fires once per customer per metric per month (the ceiling once per day),
however many processes are running; the state is kept in the store, so a restart does not
repeat it. A usage jump past both 80% and 100% fires only the 100% alert. If your callback
raises, the alert is retried on the next check; it never breaks `record` or `admit`. With
`plan_of` set, thresholds are checked after every `record`; `admit` checks them too.

**Failing closed.** If the ledger cannot be read, `admit` refuses with
`reason="ledger_unreadable"` instead of treating the allowance as unlimited. `record` raises
`StoreError` if it cannot write, so a lost row is never silent.

### Prices

llmledger ships no prices. You write a table in JSON or TOML (or pass a dict), in US dollars per
**one million** tokens:

```json
{
  "models": {
    "your-model-id": {"input": 1.25, "cached_input": 0.125, "output": 10.0, "cache_write": 1.5}
  },
  "prefixes": {"your-model-id-": {"input": 1.25, "output": 10.0}},
  "units": {"call_seconds": 0.0025}
}
```

- `input` and `output` are required. Without `cached_input`, cached tokens are charged at the
  input rate (never less). Without `cache_write`, cache writes are charged at the input rate and
  a warning is logged once per model.
- `prefixes` price a family, so a dated snapshot id matches without its own row; the longest
  prefix wins. Names match exactly first, then ignoring case.
- A model with no row is **unpriced**: its calls are recorded with `cost_usd` null, a warning is
  logged once, and the report counts them. Unpriced calls add nothing to a dollar allowance, so
  keep the table complete. With `strict_prices=True`, `record` writes the row and then raises
  `UnpricedModelError`, and `admit(..., model=...)` raises before the call.
- Keys starting with `_` are comments.

`examples/prices.example.json` and `.toml` show the format with placeholder names and numbers.
Replace all of them with your providers' current prices. Then check the table:

```bash
llmledger prices check prices.json                       # placeholders, zeros, per-1K prices
llmledger prices check prices.json --dir /var/lib/myapp/llm-usage   # + models you forgot
```

### Configuration reference

`Ledger(store, **settings)` takes a directory path (or a `Store`) and these settings. The same
settings load from a file with `LedgerConfig.load("config.toml")` (see
`examples/config.example.toml`) or from the environment with `LedgerConfig.from_env()`; pass
either to `Ledger.from_config(config, store, on_alert=..., plan_of=...)`. Unknown keys are
errors, with a suggestion for near misses.

| Setting | Default | Meaning |
|---|---|---|
| `timezone` | required for a new ledger | IANA zone for month and day boundaries, e.g. `"UTC"`, `"America/New_York"`. The directory remembers it; opening it with another zone is an error. |
| `prices` | none | A `PriceTable`, a dict in the table format, or a path to a JSON/TOML table. |
| `plans` | none | A list of `Plan`s, or a mapping of name to limits. |
| `daily_ceiling_usd` | none | Dollars per day across all customers. A number of 0 or more, or `"off"`. Anything else (`"50 USD"`, `"5O"`, `""`) is an error. 0 admits nothing. |
| `thresholds` | `[80, 100]` | Alert percentages. An empty list turns alerts off. |
| `strict_prices` | `false` | Raise on an unpriced model (see Prices). |
| `fsync` | `false` | Flush every append to disk before `record` returns. |
| `on_alert` | none | Code only: `callback(alert)`. |
| `plan_of` | none | Code only: `customer -> Plan, plan name or None`. |
| `clock` | now | Code only: returns the current time; for tests. |

`Plan(name, ...)` limits; any left as `None` is not metered:

| Limit | Meaning |
|---|---|
| `monthly_usd` | model spend per month, in dollars |
| `monthly_tokens` | input plus output tokens per month |
| `monthly_calls` | model calls per month |
| `daily_usd` | spend per day, so a month is not spent in one afternoon |
| `monthly_units` | non-token allowances, e.g. `{"call_seconds": 6000}` |
| `kinds` | only calls of these kinds count toward the plan |

Environment variables (`LedgerConfig.from_env()` and the command line):

| Variable | Meaning |
|---|---|
| `LLMLEDGER_DIR` | the ledger directory (command line) |
| `LLMLEDGER_TIMEZONE` | `timezone` |
| `LLMLEDGER_DAILY_CEILING_USD` | `daily_ceiling_usd` |
| `LLMLEDGER_PRICES` | path to the price table |
| `LLMLEDGER_STRICT_PRICES` | `1`/`true` or `0`/`false` |
| `LLMLEDGER_CONFIG` | a JSON or TOML settings file; the variables above override it |

Any other variable starting `LLMLEDGER_` is an error naming the closest match, so a misspelled
ceiling cannot quietly mean no ceiling.

Command line:

```
llmledger report [--month YYYY-MM] [--customer ID] [--json] [--dir DIR]
llmledger prices check TABLE [--dir DIR [--month YYYY-MM]] [--json]
```

Exit codes: 0 fine, 1 `prices check` found a problem, 2 the command could not run.

### What it stores

In the ledger directory: `llmledger.json` (the time zone), `usage-YYYY-MM.jsonl` (one line per
model call), `units-YYYY-MM.jsonl` (one line per unit observation), `alerts-YYYY-MM.json` (which
alerts fired) and `.locks/`. A usage line:

```json
{"v":1,"ts":"2026-09-15T12:00:00+00:00","customer":"acme","kind":"chat","model":"your-model-id","input_tokens":1200,"cached_input_tokens":1000,"cache_write_tokens":0,"output_tokens":300,"cost_usd":0.00135,"call_id":"req-81f2"}
```

Customer ids, model ids, token counts, costs and your tags. No prompts, no replies, no end-user
data unless you put it in a customer id or a tag. The package has no method that edits or
deletes a row. To archive old months, move their files away; to keep the ledger somewhere other
than a directory, implement the five-method `Store` interface in `llmledger/store.py`.

### Honest limits

- **Figures are estimates, not bills.** They are token counts the provider reported, multiplied
  by the prices you wrote. Provider-side discounts, batch pricing, free allowances, taxes and price
  changes are not known to llmledger. Reconcile with your invoices, and keep the table current.
- **Allowances are soft by one call.** A call is admitted while usage is under the limit, so the
  call that crosses it completes, and calls admitted at the same moment in different processes
  can each complete. Pass `estimate=` to tighten it. It is a quota, not a lock.
- **It does not stream-meter.** For a streamed response, record the final usage chunk (OpenAI
  with `stream_options={"include_usage": True}`, Gemini's last chunk, Anthropic's
  `message_start` plus `message_delta`) or pass the counts to `call.usage(...)`. For Anthropic,
  input comes from `message_start` and output from `message_delta`:
  `call.usage(input_tokens=start["message"]["usage"]["input_tokens"], output_tokens=delta["usage"]["output_tokens"], model=start["message"]["model"])`.
  (Add `cached_input_tokens=` and `cache_write_tokens=` from the same `message_start` usage.)
- **It does not wrap provider SDKs** or intercept calls made elsewhere in your code. Every model
  call your app pays for needs a `record`.
- **Cache writes have one rate.** Anthropic prices 5-minute and 1-hour cache writes differently;
  the table has a single `cache_write` rate, so 1-hour writes are priced at it too.
- **`prices check` is a heuristic.** It flags placeholder names, zero rates, rates that look
  per-token or per-thousand, and rates over $1,000 per million. It cannot know your provider's
  real prices.
- **A row cut short by a crash is skipped.** It is counted in the report's `skipped_lines`, and
  the next row is written on a line of its own, so later calls are not lost.
- **Idempotency looks back one month.** A `call_id` is checked against this month and last month.
- **One directory per ledger, one machine or a shared filesystem.** `FileStore` locks with
  `flock` (Windows: `msvcrt`). Many hosts need a shared filesystem that supports locks, or your
  own `Store`. Each process reads each month file incrementally; very busy ledgers (millions of
  rows a month) will want archiving or a database-backed `Store`.
- **Units reconcile by the largest figure.** If two writers disagree, the larger quantity wins.
  That errs toward counting more, by design.
- **Time zone changes are refused** for an existing directory rather than silently re-splitting
  months.

### Tests

```bash
cd llmledger
python3 -m pytest
```

The suite uses no network: provider responses are fixtures, and the "provider" in the guard
tests is a fake that counts its calls.

---

Docs mirror of the private repository Dominares-Tools/llmledger; updated with each release.
