# Changelog

## 1.0.3

No change to the code.

- The 1.0.1 changelog entry was reworded.

## 1.0.2

Documentation and packaging only. The code is unchanged.

- The README now opens with a one-line description and a three-line example you can paste,
  and the licence and access notes moved to the end.
- The package version (`pip show llmledger`, `llmledger.__version__`) now matches the release tag.
  Release 1.0.1 reported 1.0.0.

## 1.0.1

Licence and contact details only. The code is unchanged.

- Licence text updated (version 1.3).
- One public contact address in the licence and the README.

## 1.0.0 (2026-09-29)

First release.

- `Ledger.record(customer, model=, input_tokens=, output_tokens=, ...)` appends one JSON line per
  model call to a monthly `usage-YYYY-MM.jsonl`: customer, kind, model, token counts (cached input
  and cache writes separately), cost and short tags. Never a prompt or a reply; tag names that
  suggest content are refused. The same `call_id` for one customer is counted once, across threads and processes
  (a file lock around the check and the append), including a retry just after the month turns.
- `Ledger.record_response(customer, response)` reads token counts from OpenAI Chat Completions,
  the OpenAI Responses API, Gemini `usageMetadata` and Anthropic Messages responses, as dicts or
  SDK objects. A response with no usage is recorded as an estimate instead of as free.
- Prices come only from a table you write (a dict, JSON or TOML; US dollars per million input,
  cached-input, cache-write and output tokens; family prefixes; per-unit prices). No prices ship
  with the package. An unpriced model is recorded with cost null and a warning; `strict_prices`
  raises instead.
- Plans with monthly dollar, token and call allowances, a daily dollar guard, per-kind counting
  and non-token unit allowances. `admit(customer, plan, estimate=...)` returns an `Admission`
  (`ok`, `used`, `limit`, `reason`, `remaining`, `resets_at`), counted from the append-only
  ledger; an unreadable ledger refuses. `guard(...)` admits, raises `AllowanceExceeded` before
  the provider is called, and records on exit.
- Months and days follow an explicitly configured IANA time zone; a ledger directory keeps the
  zone it was created with. Month summaries are read incrementally and cached.
- Threshold alerts (80% and 100% by default) through your callback, once per customer, metric and
  month, with the fired state kept in the store so restarts and other processes stay quiet.
- A daily spend ceiling across all customers, enforced by `admit`. A ceiling that cannot be read
  (`"50 USD"`, `""`, a misspelled setting or `LLMLEDGER_*` variable name) is an error at start-up, never "no ceiling".
- `ledger.units`: non-token usage keyed by your event id (call seconds, pages, searches). Several
  writers reporting one event reconcile to one count at the largest figure; `units.admit` takes
  reservations held by events still in progress.
- `FileStore` (a directory, safe across processes) and `MemoryStore`, behind a five-method
  `Store` interface with no delete.
- Settings from code, a JSON/TOML file (`LedgerConfig.load`) or `LLMLEDGER_*` environment
  variables. Unknown keys are errors, with a suggestion for near misses.
- Command line: `llmledger report --month YYYY-MM [--customer ID] [--json]` and
  `llmledger prices check TABLE [--dir DIR]`, which flags placeholder names, zero rates,
  per-thousand prices and models in the ledger the table does not price.
