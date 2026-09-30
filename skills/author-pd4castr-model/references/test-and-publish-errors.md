# Test and publish failures

Each message below is quoted as `pd4castr test` or `pd4castr publish` emits it.
Match on the stable text, because most carry an interpolated key or path.
Query validation failures (views, sensitivities, aggregations, the run
datetime query) belong to `write-pd4castr-model-view`. The docs pages for the
two commands are
[Testing your model](https://docs.v2.pd4castr.com.au/authoring/testing-your-model) and
[Publishing](https://docs.v2.pd4castr.com.au/authoring/publishing).

## Contents

- Config: the file is missing or invalid
- Container: the image does not build or the container does not run
- Input and output exchange: a GET or the PUT did not happen
- Output contract: the PUT body is not a valid forecast
- Input files: a declared input is missing or misdeclared
- Authentication
- Publish state: local `$$` fields disagree with the server, or a flag is wrong
- Axes
- Warnings, which do not stop a publish

## Config

**`No config found at <path>`**

Run the CLI from the project root, where `.pd4castrrc.json` lives.

**`Failed to parse project config`**

`.pd4castrrc.json` is not valid JSON. Fix the syntax.

**`Config validation failed`**

Followed by one `✘ <path> - <message>` line per problem, where `<path>` is
the dotted location in the config, such as `outputs.0.key`. Fix each field.

**`Unknown timeHorizon "<value>".`**

Use one of the values the message lists. When the platform supports a newer
value, upgrade `@pd4castr/cli`.

## Container

**`Failed to build docker image`**

The container's build output follows the message. Fix the `Dockerfile` or the
dependency it names. A dependency with no `linux/amd64` build fails here on an
Apple Silicon machine.

**`Failed to run model container`**

The container exited non-zero. Its stderr follows the message. Fix the
exception it shows.

## Input and output exchange

`test` prints one line per input and one for the output, then a result:

```text
	✔ Input Fetched - <key>.<targetFileFormat>
	✘ Output Uploaded
Model I/O test failed
```

`publish` reports the same failure as
``Model I/O test failed. Please run `pd4castr test` to debug the issue.``

**`✘ Input Fetched - <key>.<targetFileFormat>`**

The entrypoint never sent a GET to that input's `INPUT_<KEY>_URL`. Read every
declared input, or remove the entry from `inputs[]`.

**`✘ Output Uploaded`**

No PUT to `OUTPUT_URL` was accepted. Check that the entrypoint sends a PUT,
that the body is valid JSON, and that the body is under 100 MB.

**`✔ Output Uploaded`, then an output failure that does not match the model**

A PUT without `Content-Type: application/json` is accepted but writes
nothing, so the output check reads whatever `test_output/` held from an
earlier run, or fails to read it. Send the header, delete `test_output/`, and
run `pd4castr test` again.

## Output contract

`test` prints each failure as a `• <message>` line under
`Model output validation failed`. `publish` prints the same list under
`Model output validation failed:`. The array and row-type checks cover every
element. The timestamp and key checks read only the first row that carries
`forecast_datetime`.

**`Output must be a JSON array of forecast rows.`**

The body parsed but is not an array. A bare object, including one wrapping the
rows under a key such as `{"rows": [...]}`, fails here. Send the array itself.

**`Output must contain only object rows.`**

At least one element is an array or a primitive. A common cause is emitting rows
as positional arrays rather than named fields.

**`Output contains no forecast rows.`**

The array is empty. The model ran and produced nothing, so check the input
filtering before looking at the output code.

**`No forecast row carries a "forecast_datetime" field.`**

No row has `forecast_datetime` by that exact name. A differently named
timestamp column does not substitute, and neither does an index.

**`forecast_datetime is not a supported ISO 8601 timestamp`**

The message quotes the offending value. Write the date, `T` or a space, then
the time with seconds, an optional fraction of up to nine digits, and an
optional `Z` or `±HH:MM` offset. An epoch integer, a date-only value, a missing
seconds field, or a `+10` or `+1000` offset fails. A value with no offset is
read as UTC.

**`Output is missing configured output variables:`**

The message lists the missing keys. Every key declared in `outputs[]` must
appear on every row. Either emit the values or remove the entries from
`.pd4castrrc.json`.

## Input files

**`Data fetcher input (<key>) data not found.`**

The message suggests `pd4castr fetch`, and that is the fix. The file it looks
for is `test_input/<key>.<targetFileFormat>`.

**`Static input (<key>) data not found at <path>`**

The `file` path in the config does not resolve. It resolves from the working
directory, so run the CLI from the project root and write the path relative
to it.

**`Static input (<key>) requires conversion (<from> -> <to>), which is not
supported via the CLI.`**

`uploadFileFormat` and `targetFileFormat` differ. Make them the same and convert
the file yourself before publishing.

**`Data fetcher input (<key>) must not require conversion (<from> -> <to>).`**

The same mismatch on a fetcher-backed input. A fetcher input must declare the
same format for both.

**`Dataset (<key>) data not found at <path>`**

The dataset's `file` does not resolve. Fix the path as for a static input.

## Authentication

**``Not authenticated. Please run `pd4castr login` to login.``**

Run `pd4castr login`, or `pd4castr login --client-credentials` in an
unattended session. `fetch` and `publish` always need it. `test` needs it
once the model is published, on the first run after that publish.

**`Client ID and secret are required for client credentials login.`**

Set `PD4CASTR_CLIENT_ID` and `PD4CASTR_CLIENT_SECRET`, or pass `--client-id`
and `--client-secret`.

## Publish state

**`Aborted`**, **`Model revision update cancelled`**

A confirmation prompt was answered no. In an unattended session, add
`--non-interactive` so the publish asks nothing.

**`OUT OF SYNC: Local revision (N) does not match the current published revision
(M)`**

Someone else published since you last pulled. Pull their updated
`.pd4castrrc.json` and publish on top of it. Do not edit `$$revision` to match:
that field records what the server said, and forcing it past a revision you have
not seen overwrites their work.

**`OUT OF SYNC: Local model group ID (...) does not match the current published
model group ID (...)`**

The config points at a different model group than the one the id resolves to.
This usually means a config was copied between projects with its `$$` fields
intact. Recover the correct config from version control rather than editing the
fields.

**`--publish-action is required when using --non-interactive with an existing
model.`**

Add `--publish-action=new-revision` or `--publish-action=update`.

**`Invalid --publish-action value:`**

Only `new-revision` and `update` are accepted. The CLI reads the flag only
with `--non-interactive`, and the argument parser does not validate it, so a
typo reaches the model flow and fails here.

**`Model group '<name>' has time horizon '<old>'; cannot publish a revision
with '<new>'`**

`timeHorizon` is fixed for the model group. Restore the old value, or publish
a new model from a config with no `$$` fields.

## Axes

**`Model axis validation failed`**

The message below the title gives the cause:

- `adding axis "<key>"`, `removing axis "<key>"`, or
  `moving axis "<key>" default`: `update` rejects the change. Publish with
  `--publish-action new-revision` instead.
- `axis "<key>" is declared as type <type>, but the model group already
  contains a revision that declares it as <type>`: an axis keeps its `type`
  across the model group. Restore the type, or declare the axis under a new
  key.

**`Axis validation against input files failed`**

A partitioned input or dataset file does not fit its axis. The message below
the title gives the cause:

- `declares partition column "<column>" for axis "<key>", but that column is
  not present in the file`: add the column, or fix `column` in the config.
- `column "<column>" has type <type>, which does not map to axis "<key>"'s
  declared type`: cast the column to match the axis `type`.
- `column "<column>" is missing declared value(s)`: add rows for each listed
  value, or remove the value from the axis.

**`axis "<key>" default <value> is not one of the declared values`**

Set `default` to one of `valuesFrom.values`.

## Warnings, which do not stop a publish

A warning prints as a `WARN` block under a header, and the publish goes on to
land.

**`N grant(s) pruned during publish`**

A view, sensitivity, or input key disappeared from every revision, or the
newest revision dropped an axis or an axis value, so the grants on it were
removed. The listed grantees lost access to data they
could see before. Report it to the user.

The warnings below print under `Model published with N warning(s)`.

**`axis "<key>" is not used by any input`**

Every trigger queues one run per value of the axis, all reading the same
input files. A dataset partitioned by the axis does not clear the warning.
Partition an input by the axis, or remove the axis.

**`axis "<key>" removes previously declared value(s)`**

Runs of the removed variants still exist but can no longer be selected from
the run history.

**`input "<key>" changed its check or fetch query, so its fetch state resets`**

The next runs may pair files fetched in different cycles. Check the inputs of
the next runs. Publish a fetch query change while no input's data is arriving,
so the reset settles within one cycle.

**`Views validated with N warning(s)`**

Prints during validation, and the publish continues. A view's dry run
returned no rows, or a joined table matched no rows. The view ships and
renders empty. See `write-pd4castr-model-view`.
