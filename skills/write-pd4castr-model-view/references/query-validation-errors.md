# Query validation errors and warnings

Messages that `pd4castr test` and `pd4castr publish` print while they validate
the config, sensitivities, input aggregations, the run datetime query, and
views. Match on the stable text, because most messages carry an interpolated
key, path, or DuckDB error.

A query failure prints under a title such as `View validation failed`, one
block per failure:

```text
  ERROR  view · <key> · <field>  <check>
         <message>
```

A warning prints the same block with `WARN`. A config failure prints under
`Config validation failed` as `✘ <path> - <message>`.

## Contents

- Any kind: SQL file not found, fixture failed to load, table does not exist
- Config: param fields, `options`, `defaultFrom`, `sample`, dataset
  `fileFormat`, duplicate view key
- Sensitivities
- Input aggregations
- Run datetime query
- Views: dry run, parameters, options queries, `defaultFrom`, columns
- Warnings: no rows, join matched no rows

## Any kind

**`view SQL file not found (<path>)`**, **`sensitivity SQL file not found
(<path>)`**, **`input aggregation SQL file not found (<path>)`**,
**`run datetime SQL file not found (<path>)`**,
**`option SQL file not found for "<param>" (<path>)`**

The path in the config does not resolve. Paths resolve from the working
directory, so run the CLI from the project root and write the path relative to
it.

**`failed to load "<table>" from <path>: <message>`**

An input or dataset file could not be loaded into DuckDB. Check that the file
exists and parses in its declared format. For a fetcher-backed input, run
`pd4castr fetch`.

**`Catalog Error: Table with name <x> does not exist!`**

Arrives inside `view dry run failed:`, `sensitivity query failed:`,
`input aggregation query failed:`, or `run datetime query failed:`. The SQL
uses a table name that was not built from the key:

- A hyphen in the key. `raw-data` is the table `raw_minus_data`, and in a view
  `input_raw_minus_data`.
- View naming in another kind, or the reverse. A view reads `input_<key>`.
  A sensitivity, an aggregation, or a run datetime query reads `<key>`.
- A dataset or `output` outside a view. Only a view reads them.

## Config

**`✘ views.<i>.params.<name>.label - …`**

Every view parameter needs a non-empty `label` and a `type`.

**`sample must be a <type> to match the declared type`**

The JSON type of `sample` differs from `type`. A `number` parameter takes an
unquoted number.

**`options is required when control is "select"`**

Add `options` with the path to a SQL file that returns `value` and `label`.

**`options is only valid when control is "select"`**

Remove `options`, or set `control` to `select`.

**`defaultFrom requires optional: true`**

A `<param>.metadata.<key>` default needs `optional: true`. A `run.axis.<key>`
default does not.

**`defaultFrom must match "<srcParamName>.metadata.<key>" or "run.axis.<key>"`**

Rewrite `defaultFrom` in one of those two forms.

**`duplicate view key "<key>"`**

Give each view a unique `key`.

**`fileFormat must be declared explicitly for "<file>" (cannot be inferred from
extension "<extension>").`**

A dataset file has no `.csv`, `.json`, or `.parquet` extension. Add
`fileFormat`.

**`key is required (lowercase alphanumeric with hyphens/underscores)`**

A sensitivity or output on a model that has not been published has no `key`.
Add one.

## Sensitivities

**`sensitivity query failed: <DuckDB message>`**

The SQL raised an error against the local inputs. Each sensitivity runs alone
on the original inputs, so a statement that depends on another sensitivity
fails here. Fix the SQL from the DuckDB message.

## Input aggregations

**`input aggregation query failed: <DuckDB message>`**

The SQL raised an error against the local inputs. Fix it from the DuckDB
message.

**`input aggregation query is missing required output column(s): <columns>`**

Alias the result columns to `datetime`, `category_name`, `input_name`, and
`value`.

## Run datetime query

**`run datetime query failed: <DuckDB message>`**

The SQL raised an error against the local inputs.

**`Custom run datetime query returned no rows.`**

Return at least one row.

**`Custom run datetime query result is missing the "run_datetime" column.`**

Alias the result column to `run_datetime`.

**`Custom run datetime query "run_datetime" column is not string.`**

Cast the value to `VARCHAR` in ISO 8601 form.

**`Custom run datetime query "run_datetime" column is not a valid timestamp.`**

Return `2026-01-01T00:00:00Z` or `2026-01-01 00:00:00`. A bare date or year
fails.

## Views

**`view dry run failed: <DuckDB message>`**

The view SQL raised an error against the local output, inputs, and datasets,
with each parameter bound to its `sample`, its first option, or `NULL`. Fix it
from the DuckDB message.

**`param "<name>" is declared but never referenced as getvariable() in the
view or option SQL`**

Read the parameter in the SQL, or remove the entry.

**`<source> references getvariable('<name>') but no such param is declared`**

`<source>` is `view SQL` or `options SQL for "<param>"`. Add the entry to
`params`, or fix the spelling in the SQL.

**`required non-select param "<name>" needs a sample value for the publish dry
run`**

Add `sample` with a value of the declared `type`, or set `optional: true`.

**`option query for "<param>" failed: <DuckDB message>`**

The options SQL raised an error. Fix it from the DuckDB message.

**`<param> option query returned no rows`**

Return at least one option.

**`<param> option query returned duplicate value(s): <values>`**

Make `value` unique across the rows.

**`<param> option query must return a string "label" column`**

Alias a string column to `label`.

**`<param> option query returned a row with a null/missing "value" column`**

Return a non-null `value` on every row.

**`<param> option query "value" column must be string | number | boolean`**

Cast `value` to a string, number, or boolean.

**`<param> option query "metadata" column must be a JSON object`**

Build `metadata` with `json_object()`.

**`<param> is a select param and cannot declare defaultFrom`**

A `<source>.metadata.<key>` default only works on a parameter with no
`control`. Remove `defaultFrom`, or remove `control` and `options`.

**`<param> declares defaultFrom but is not marked optional: true`**

Set `optional: true` on the parameter.

**`<param>.defaultFrom "run.axis.<key>" names axis "<key>", which is not
declared`**

The message lists the declared axes. Use one of them in `defaultFrom`.

**`<param>.defaultFrom references undeclared param "<source>"`**

Declare the source parameter, or fix the name in `defaultFrom`.

**`<param>.defaultFrom references "<source>" which is not a select param;
metadata is only available on materialised options`**

Make the source a `select` parameter, or remove `defaultFrom`.

**`<param>.defaultFrom references "<source>" but no materialised options exist
for it`**

The source's options query returned nothing to read metadata from. Fix that
query first.

**`<param>.defaultFrom expects "<source>.metadata.<key>" present and non-null
on every option row`**

Add `<key>` to `metadata` on every row of the source's options query.

**`inferred column "<column>" has no columns[] declaration`**

The query returned a column with no `columns` entry. Add the entry, or drop
the column from the select list.

**`columns[] entry "<key>" matches no inferred column`**

A `columns` entry matches no returned column. The match is case-sensitive.
Fix the alias in the select list or the `key` in `columns`.

## Warnings

A warning does not stop the publish. It means the view returned no rows
against the local data, so the published view may render empty. A sparse
sample can return no rows for a view that works on live data.

**`join to "<table>" matched no rows during validation, so the report would
render empty. Check the join condition and filters on "<table>"`**

A joined table contributed no rows. Compare the join columns in both tables,
and check for case and type differences, such as an integer id joined to a
string column.

**`the view produced no rows during validation, so it will render an empty
table. Check that the joins and filters match your data`**

The query ran without error and returned nothing against the local data.
Check the filters against the files in `test_input/`, the datasets, and the
output the publish produced from the container.
