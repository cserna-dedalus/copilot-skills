---
name: "copilot-cli-usage-report"
description: "Break down GitHub Copilot CLI consumption by model over a given period (requests, tokens, and a rough credit-usage proxy), sourced from local usage telemetry on this machine. Use for any request about Copilot CLI usage, spend, or consumption per model — by day, week, month, or 'this month' — in English, Spanish, or any other language."
compatibility: "Requires the `powershell` tool with local `python` (stdlib `sqlite3`/`csv`/`json` only, no extra packages) and read access to `$HOME/.copilot/session-store.db`. Windows paths shown; adapt separators on other OSes."
metadata:
  author: "cserna"
---

## User Input

```text
$ARGUMENTS
```

`$ARGUMENTS` may specify a time period in any language (e.g. `2026-09`, `September`,
`septiembre`, `this month`, `este mes`, `last 7 days`, `última semana`) and/or an output
location. If empty, default to the **current calendar month up to today**, and save outputs
under the current session's `files/` folder (or ask where to save if there is none).

## Language

Respond in whichever language the user used for their request (English, Spanish, or
otherwise) — table headers, narrative, and caveats should all match it. Don't force English
or Spanish; mirror the user.

## What this skill does

Produces a table (and CSV/JSON exports) of Copilot CLI consumption **grouped by model**,
for the requested period, using the real local telemetry table `assistant_usage_events`
inside `session-store.db` — the same store the CLI itself uses for `/usage`. This is **not**
a GitHub server-side API; it is local, per-machine data. Always be upfront about that scope
limitation (see Guardrails).

## Why this data source (must explain before running, briefly)

Before executing, tell the user in 2-3 sentences:
- GitHub does **not** expose a public per-user, per-model Copilot CLI usage API. The org/
  enterprise `/orgs/{org}/copilot/metrics` and `/enterprises/{enterprise}/copilot/metrics/reports/*`
  endpoints are admin-only, aggregate across all users, and require `manage_billing:copilot`
  or `read:enterprise`.
- The only official per-user, per-model breakdown is the Billing UI
  (Settings → Billing → Copilot), which this skill does **not** call.
- This skill instead reads the local `assistant_usage_events` table that Copilot CLI itself
  writes on this machine — real per-request data, but scoped to this installation only.

If the user explicitly wants org/enterprise-aggregated data instead, do not use this skill;
point them to the metrics endpoints above and ask for admin credentials/scopes.

## Procedure

### 1. Locate the local store

```powershell
Get-ChildItem "$env:USERPROFILE\.copilot\session-store.db"
```

On macOS/Linux the equivalent path is `$HOME/.copilot/session-store.db`. If missing, tell
the user this machine has no local Copilot CLI usage history and stop.

### 2. Resolve the target period

Translate the request (in whatever language) into a concrete date range: either a
`YYYY-MM` month prefix (default: current month, `date +%Y-%m` / `Get-Date -Format yyyy-MM`),
or an explicit `start_date`/`end_date` for ranges like "last 7 days" or "última semana".
Confirm it with the user only if genuinely ambiguous.

### 3. Export raw events (audit trail)

Write a small Python script (stdlib only) to a temp/session file and run it with
`python <script>.py` via the `powershell` tool — do not try to inline multi-line Python
with `-c` on Windows (quoting breaks). The script must:

1. Open `session-store.db` read-only via `sqlite3`.
2. `SELECT *` from `assistant_usage_events` filtered by the resolved period — either
   `substr(created_at,1,7) = '<YYYY-MM>'` for a month, or `created_at BETWEEN '<start>' AND
   '<end>'` for an explicit range.
3. Write every matching row verbatim to `copilot_cli_usage_raw_<period>.csv` (this is the
   "raw data used for the calculation" — always produce it, per user expectations set by
   prior runs of this analysis).

### 4. Aggregate by model

In the same or a second script, group the rows above by `model` and compute, per model:

- `requests` (row count) and `sessions` (COUNT DISTINCT `session_id`)
- `input_tokens`, `output_tokens`, `cache_read_tokens`, `cache_write_tokens`, `reasoning_tokens`
- `total_tokens` = sum of the four token columns above
- `pct_total_tokens` = share of the grand total across all models
- `sum_request_multiplier` and its `pct` share — label this explicitly as an **unverified
  proxy** for premium-request/AI-credit consumption (the `request_multiplier` and
  `total_nano_aiu` fields are internal, undocumented by GitHub; never call them "AI Credits"
  without this caveat)
- `total_nano_aiu` (raw internal unit, include but do not convert or rename)

Sort by `total_tokens` descending. Write:
- `copilot_cli_usage_by_model_<period>.json` (grand totals + per-model array)
- `copilot_cli_usage_by_model_<period>.csv` (same, flattened)

### 5. Report to the user

Present a Markdown table with columns (translated to match the user's language): Model,
Requests, Sessions, Input tokens, Output tokens, Cache read, Cache write, Total tokens, %
tokens. Add a totals row. Then a short paragraph giving the `sum_request_multiplier` view as
the AI-credit proxy, clearly labeled unverified. List the file paths written.

## Guardrails

- Never present `request_multiplier` or `total_nano_aiu` as official "AI Credits" — they are
  internal, undocumented fields; always flag them as an unverified local proxy.
- Never substitute Billing-page figures or invent an endpoint if the user asks for official
  numbers instead — say plainly that only Settings → Billing → Copilot has the authoritative
  per-user figure, and this skill cannot fetch that.
- Always state the single-machine scope limitation: this only covers Copilot CLI usage run
  from this local installation, not other devices/machines the user may have used.
- Do not modify `session-store.db`; open it read-only / only run `SELECT` queries.
- Always write the raw per-event CSV in addition to the aggregate — do not only report the
  aggregate table.
