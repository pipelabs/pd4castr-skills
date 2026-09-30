---
name: write-pd4castr-model-view
description: >
  Writes the SQL and the .pd4castrrc.json entry for a pd4castr model run view,
  a sensitivity, an input aggregation, or a run datetime query, and validates
  it with pd4castr test and pd4castr publish. Use when adding a view, a report,
  a chart, a dataset for a view, a scenario, a what-if sensitivity, an input
  aggregation, or a runDatetimeQuery to a model, when setting a model's run
  datetime from its input data, when a query fails validation at publish
  (view dry run failed, Catalog Error, View validation failed), or when a
  published view renders an empty table. Not for
  building the model container, declaring inputs and outputs, or axes
  (author-pd4castr-model), and not for ad hoc queries in the Data Explorer.
compatibility: Requires a running Docker daemon, the @pd4castr/cli package, and network access to the pd4castr API.
---

# Write a pd4castr model query

A view, a sensitivity, an input aggregation, and a run datetime query are each
one SQL file that the platform runs in DuckDB. A view reports on a finished
run. A sensitivity rewrites the inputs before a run. An aggregation summarises
the inputs for a chart. A run datetime query sets the run datetime from the
inputs.

Copy this checklist and check off each step as it passes:

```text
Query progress:
- [ ] 1. Pick the kind and its tables
- [ ] 2. Write the SQL
- [ ] 3. Declare it in .pd4castrrc.json
- [ ] 4. Validate
```

## 1. Pick the kind and its tables

Docs: [Model run views](https://docs.v2.pd4castr.com.au/authoring/model-run-views),
[Sensitivities](https://docs.v2.pd4castr.com.au/authoring/sensitivities),
[Input aggregations](https://docs.v2.pd4castr.com.au/authoring/input-aggregations),
[Model datasets](https://docs.v2.pd4castr.com.au/authoring/model-datasets).

| Kind | Runs over | Tables |
| --- | --- | --- |
| View | A finished run | `output`, `input_<key>`, `dataset_<key>` |
| Sensitivity | The inputs, before the run | `<key>` per input |
| Aggregation | The inputs, after the trigger | `<key>` per input |
| Run datetime query | The inputs, at the trigger | `<key>` per input |

In a table name, every `-` in the key becomes `_minus_`. An input keyed
`raw-data` is `input_raw_minus_data` in a view and `raw_minus_data` in the
other three kinds. Only a view reads `output` or a dataset. A view's options
queries read the same tables as the view.

## 2. Write the SQL

Docs: [Model run views](https://docs.v2.pd4castr.com.au/authoring/model-run-views),
[Sensitivities](https://docs.v2.pd4castr.com.au/authoring/sensitivities),
[Input aggregations](https://docs.v2.pd4castr.com.au/authoring/input-aggregations),
[Run modes and scheduling](https://docs.v2.pd4castr.com.au/authoring/run-modes-and-scheduling).

Write DuckDB SQL. Use DuckDB functions and casts.

View:

- Return one row per point on the chart.
- Read each declared parameter with `getvariable('<name>')`.
- Wrap an optional parameter in `coalesce()`. An unset optional parameter
  binds to `NULL`.
- Alias every column to a `columns` key. The match is case-sensitive.

Options query, for a `select` parameter:

- Return a non-null `value` and a string `label` on every row, with no
  duplicate `value`. Add `metadata` as a JSON object built with
  `json_object()` when another parameter reads it through `defaultFrom`.
- Put the default selection in the first row.

Sensitivity:

- Write `UPDATE`, `INSERT`, or `DELETE` statements against the input tables.
- Start from the original inputs. Each sensitivity runs alone, so one cannot
  read another's changes.

Aggregation:

- Return `datetime`, `category_name`, `input_name`, and `value`.
- Group rows into series with `category_name`. The chart names each series
  `<aggregation name> (<category_name>)`. `input_name` is required but not
  shown.

Run datetime query:

- Return a `run_datetime` column as a string on at least one row. The first
  row sets the run datetime.
- Format it as `2026-01-01T00:00:00Z` or `2026-01-01 00:00:00`. A `+HH:MM`,
  `+HHMM`, or `+HH` offset also works. A value with no zone is read as UTC.
- Omit the query to use the trigger time as the run datetime.

## 3. Declare it in .pd4castrrc.json

Docs: [Configuration file](https://docs.v2.pd4castr.com.au/authoring/configuration-file).

- Put a view in `views[]` with `key`, `name`, `sql`, `columns`, and `params`
  when the SQL reads any.
- Put a sensitivity in `sensitivities[]` with `name`, `key`, and `query`.
- Put an aggregation in `inputAggregations[]` with `name`, `key`, and `query`.
- Set `runDatetimeQuery` to the path of its SQL file.
- Put a dataset a view reads in `datasets[]` with `key` and `file`. Add
  `fileFormat` when the file's extension is not `.csv`, `.json`, or
  `.parquet`.
- Match every `key` to `^[a-z0-9][a-z0-9_-]*$` and keep it unique in its
  array. The CLI checks view keys only. A duplicate sensitivity, aggregation,
  or dataset key passes validation and fails at the server.
- Write every path relative to the project root, and run the CLI from the
  project root. Paths resolve from the working directory.

A view entry, with `params` as an object keyed by parameter name:

```json
{
  "key": "forecast-vs-actual",
  "name": "Forecast vs actual",
  "sql": "views/forecast-vs-actual.sql",
  "params": {
    "region": {
      "type": "string",
      "label": "Region",
      "control": "select",
      "options": "views/region-options.sql"
    }
  },
  "columns": [
    { "key": "forecast_datetime", "role": "dimension" },
    { "key": "forecast", "role": "measure", "unit": "units" },
    { "key": "actual", "role": "measure", "unit": "units" }
  ]
}
```

For a view:

- Give every parameter a `type` of `string`, `number`, or `boolean` and a
  `label`.
- Declare every parameter the SQL reads, and read every parameter you declare.
- Give a required parameter that is not a `select` a `sample` value of the
  declared `type`.
- Make a parameter a `select` with `"control": "select"` and `options` set to
  the path of its options query.
- Set `defaultFrom` to `<param>.metadata.<key>` to default from another
  parameter's selection. That parameter must be a `select`, and this one must
  set `optional: true` and leave out `control`.
- Set `defaultFrom` to `run.axis.<key>` to default to the run's axis value.
- Give every `columns` entry a `key` and a `role` of `dimension` or
  `measure`.
- Give every `measure` column the same non-empty `unit` when the view is a
  chart. A view with no measure, or with mixed or missing units, renders as a
  table.

Keep every `key` unchanged on a published model. When a view, sensitivity, or
input key disappears from every revision of the model, the grants on it are
removed. Aggregations carry no grants.

## 4. Validate

Docs: [Testing your model](https://docs.v2.pd4castr.com.au/authoring/testing-your-model),
[Publishing](https://docs.v2.pd4castr.com.au/authoring/publishing).

`pd4castr test` checks the config and the run datetime query. Views,
sensitivities, and aggregations are checked only by `pd4castr publish`, which
reruns the container test and then validates, in order, sensitivities,
aggregations, the run datetime query, and views. It stops at the first kind
that fails and reports every failure in that kind. A failure stops the publish
before anything reaches the server.

Publish a model that has never been published with `pd4castr publish`. On a
model that has `$$id` in `.pd4castrrc.json`, publish with one of:

```sh
pd4castr publish --non-interactive --publish-action update
pd4castr publish --non-interactive --publish-action new-revision
```

`update` replaces the current revision and keeps its number. `new-revision`
creates the next revision. The CLI reads `--publish-action` only with
`--non-interactive`; without it, the publish asks which to do.

1. Run `pd4castr test`. On a failure, fix it and run again.
2. Run the publish.
3. On a failure, find the message in
   [query-validation-errors.md](references/query-validation-errors.md), fix
   the SQL or the config, and return to step 1.
4. On `Views validated with N warning(s)`, the publish has gone ahead. The
   view returned no rows against the local data and may render empty. Find
   the warning in
   [query-validation-errors.md](references/query-validation-errors.md), fix
   it, and publish again with
   `pd4castr publish --non-interactive --publish-action update`.
5. Proceed when `publish` prints no failure and no warning.

## Bundled files

- [query-validation-errors.md](references/query-validation-errors.md) maps
  each validation failure and warning to its cause and fix.
