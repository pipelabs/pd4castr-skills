---
name: compare-pd4castr-runs
description: >
  Backtests and compares pd4castr forecasts through the pd4castr MCP server
  (list_runs, get_run_output) or the Python SDK: the run that was live at a
  given time, one model revision against another, a sensitivity
  against its base run, and one axis variant against another. Use when
  building a backtest, finding which forecast was live at a past time,
  comparing today's forecast with an earlier one, evaluating forecast
  accuracy over a date range, comparing models or revisions, or scoring
  sensitivities. Not for pulling a single latest forecast, which the MCP's
  get_latest_run_output or the SDK docs cover, and not for the platform's
  own performance metrics page, which runs the model author's SQL.
compatibility: Requires the pd4castr MCP server, or Python 3.11 or newer with the pd4castr-api-sdk package, and network access to the pd4castr API.
---

# Compare pd4castr runs

Every comparison is a set of run lists, one run per point in time from each
list, and one output per run, joined on `forecast_datetime`.

Copy this checklist and check off each step as it passes:

```text
Comparison progress:
- [ ] 1. Choose the MCP or the SDK
- [ ] 2. Decide the axis of comparison
- [ ] 3. List the candidate runs
- [ ] 4. Pick one run per point in time
- [ ] 5. Fetch and align the outputs
- [ ] 6. Check the comparison
```

## 1. Choose the MCP or the SDK

Docs: [Python SDK quick start](https://docs.v2.pd4castr.com.au/python-sdk/quick-start).

Use the pd4castr MCP server when its tools are available, the answer is for
this conversation, and the comparison covers ten runs or fewer. Every page of
output it returns lands in the conversation. Use the SDK for a script, a
scheduled job, a backtest over more runs, or a comparison that includes
archived revisions, which the MCP cannot list.

When the comparison fits the MCP and no pd4castr MCP tools are available,
ask the user to add the server, then continue once they have. In Claude Code:

```sh
claude mcp add --transport http pd4castr https://mcp.v2.pd4castr.com.au/mcp
```

In claude.ai, add a custom connector with the URL
`https://mcp.v2.pd4castr.com.au/mcp`. The server asks the user to sign in to
pd4castr on first use. If the user declines, use the SDK.

Each SDK call in steps 2 to 5 has an MCP tool. The tool prefix depends on the
name the user gave the server, so match on the tool name:

| Step | SDK | MCP tool |
| --- | --- | --- |
| 2 | `get_model_group_models(group_id)` | `get_model_group` with `id`: live revisions as `models` |
| 2 | `model.sensitivities`, `model.axes`, `model.output_variables` | `get_model` with `id`: `sensitivities`, `axes`, `outputVariables` |
| 3 | `get_model_runs(...)` | `list_runs` with `id`, `sensitivity`, `after`, `before`, `limit` (100 at most), `cursor` |
| 5 | `get_model_run_output(model_id, run_id)` | `get_run_output` with `id`, `runId`, and `columns` set to the compared output keys |

- MCP results use camelCase: `runDatetime`, `createdAt`, `completedAt`,
  `axisValues`. The rules in steps 2 to 4 apply to them unchanged.
- `list_runs` returns `runs`, `hasMore`, and `nextCursor`. Call again with
  `nextCursor` as `cursor` until `hasMore` is false.
- `get_run_output` returns one page of `rows`. Call again with `nextCursor`
  as `cursor`, and the same `from`, `to`, and `columns`, until `nextCursor`
  is null.
- On `status` `pending`, wait `retryAfterSeconds` and call again with the
  same arguments. An error saying the output cannot be produced is final:
  record the run as skipped.
- On an error saying the platform is rate limiting the connection, wait the
  time it gives. MCP calls count against the signed-in user's own
  60-a-minute limit, separate from any SDK credential.

## 2. Decide the axis of comparison

Docs: [Discover models](https://docs.v2.pd4castr.com.au/python-sdk/discover-models).

With the SDK, create the client. Pass both credentials; the SDK reads neither from the
environment on its own:

```python
import os
from pd4castr_api_sdk import Client

client = Client(
    client_id=os.environ["PD4CASTR_CLIENT_ID"],
    client_secret=os.environ["PD4CASTR_CLIENT_SECRET"],
)
```

Build one run list per side of the comparison:

| Comparison | One list per | `get_model_runs` arguments | Then keep |
| --- | --- | --- | --- |
| Revisions of one model | Model id per revision, from the group's models as below | `sensitivity="base"` | Runs of the default axis variant |
| Two different models | Model id | `sensitivity="base"` | Runs of the default axis variant |
| Sensitivity against base | Sensitivity | `sensitivity="base"`, then `sensitivity=<id>` | Runs of one axis variant |
| Axis variants | Variant | `sensitivity="base"` | Runs whose `axis_values` equal that variant |

- Take each sensitivity id from `model.sensitivities` or
  `client.get_model_sensitivities(model.id)`. The filter takes an id or
  `"base"`, never a key.
- A base run has `run.sensitivity` set to `None`. A sensitivity run carries
  `run.sensitivity.id`, `.key`, and `.name`.
- `get_model_runs` has no axis filter. On a model with axes, `"base"` returns
  the base run of every variant, so filter on `run.axis_values` yourself. It
  holds a value for every axis. Read each axis's `default` from `model.axes`.
  On a model with no axes, `run.axis_values` is `None` and there is nothing
  to filter.
- `get_model_group_models()` returns live revisions only. `archived=True`
  returns archived revisions only, and only to the organisation that owns
  the model. List every revision with both calls:

  ```python
  revisions = client.get_model_group_models(group_id) + client.get_model_group_models(group_id, archived=True)
  by_number = {model.revision: model for model in revisions}
  ```
- A sensitivity run shares `run_datetime` with its base run. Variant runs of
  one trigger share `run_datetime` with each other.

## 3. List the candidate runs

Docs: [Browse historical runs](https://docs.v2.pd4castr.com.au/python-sdk/browse-historical-runs).

```python
def list_runs(model_id, sensitivity, after, before, variant=None):
    runs = []
    cursor = None
    while True:
        page = client.get_model_runs(
            model_id,
            after=after,
            before=before,
            sensitivity=sensitivity,
            limit=100,
            cursor=cursor,
        )
        runs.extend(page.data)
        if not page.has_more:
            break
        cursor = page.next_cursor
    if variant is not None:
        runs = [run for run in runs if run.axis_values == variant]
    runs.sort(key=lambda run: (run.run_datetime, run.created_at))
    return runs
```

- `status` defaults to `completed`, which is what a backtest needs.
- `after` and `before` filter on `run_datetime` and exclude the bound itself.
  Widen `after` by one run interval so the first point in time has a live run.
- Pages come back newest `created_at` first, so a retry or a backfill lands
  out of `run_datetime` order. Fetch the whole window, then sort as above.
- Pass `variant` as the full `axis_values` mapping, every axis included, or
  `None` on a model with no axes.

## 4. Pick one run per point in time

Docs: [Browse historical runs](https://docs.v2.pd4castr.com.au/python-sdk/browse-historical-runs).

The run that was live at time T is the run that had completed by T with the
greatest `run_datetime`. Among runs that share that `run_datetime`, it is the
one with the latest `created_at`:

```python
def live_at(runs, t):
    done = [run for run in runs if run.completed_at and run.completed_at <= t]
    return done[-1] if done else None
```

- `run_datetime` is the time the run was triggered, not the time its output
  existed. Filter on `completed_at` as above, or the backtest reads forecasts
  that were not yet available at T.
- The sort in step 3 orders by `run_datetime`, then `created_at`, so the last
  run in `done` is the one to keep.
- Pass `t` as `YYYY-MM-DDTHH:MM:SS.sssZ` in UTC, for example
  `2025-01-01T00:00:00.000Z`. `run_datetime`, `created_at`, and
  `completed_at` are strings in that format, so string order is time order. A
  `t` without milliseconds or with an offset compares wrongly.
- Record every T where `live_at` returns `None`.

## 5. Fetch and align the outputs

Docs: [Compare models](https://docs.v2.pd4castr.com.au/python-sdk/compare-models),
[Fetch the latest forecast](https://docs.v2.pd4castr.com.au/python-sdk/fetch-latest-forecast).

```python
from pd4castr_api_sdk import OutputUnavailableError

key = model.output_variables[0].key

try:
    result = client.get_model_run_output(model.id, run.id)
except OutputUnavailableError:
    skipped.append(run.id)
else:
    values = {row.forecast_datetime: getattr(row, key) for row in result.run.data}
```

- Each output row holds `forecast_datetime` plus one attribute per column
  the model emitted, with the name lowercased. Read a key from
  `model.output_variables` with `getattr(row, key)`, or read every column
  with `row.model_dump()`.
- `get_model_run_output()` polls through the 202 that a cold output returns,
  up to `poll_timeout` seconds (default 300), then raises `ApiError` with
  `status_code` 202. Retry that run later.
- `OutputUnavailableError` (409, code `MODEL_RUN_OUTPUT_UNAVAILABLE`) is
  final. Skip the run and record it.
- Cache each output on disk by run id. Every call counts against one
  counter per credential. An output call is refused once that counter passes
  120 in a minute, and every other call once it passes 60, so a burst of
  output calls also blocks the listing. A backtest over 90 days of daily runs
  is 90 output calls plus the listing.
- `comparison_model=[...]` on the same call fills `result.comparison_runs`, a
  dict keyed by model id. Each entry is that model's newest completed base
  run of its default axis variant with `run_datetime` at or before this
  run's. A model with no such run, or one the credential cannot read, is
  absent from the dict. An entry whose output is missing has an empty
  `data`. Use it only when that pairing is the one you want.

Join on `forecast_datetime`. Treat a datetime present in one run and absent in
the other as missing, not zero. Bring actuals from your own source and join on
the same key in UTC.

## 6. Check the comparison

1. Check that every run in a list has the `sensitivity` and `axis_values` the
   list is meant to hold.
2. Check that no two points in time in one list resolved to runs that
   disagree on `run.sensitivity` or `run.axis_values`.
3. Count the points in time with no live run and the runs skipped in step 5.
   Report both counts with the result.
4. On a mismatch in 1 or 2, fix the filter in step 3 and rerun from there.
   Proceed only when both checks pass.
