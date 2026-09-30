---
name: author-pd4castr-model
description: >
  Builds a pd4castr forecasting model from an empty directory to a published
  revision: scaffolds the project with the pd4castr CLI, declares inputs,
  outputs, and axes in .pd4castrrc.json, implements the container's
  INPUT_*_URL and OUTPUT_URL contract, runs pd4castr test until it passes, and
  publishes. Use when creating a new model, wrapping an existing forecast
  script in a Docker container for pd4castr, adding axes or variants,
  publishing a model or a new revision from CI or any non-interactive session,
  or fixing a failing pd4castr test or pd4castr publish, including output such
  as Input Fetched, Model I/O test failed, or OUT OF SYNC.
  Not for view, sensitivity, aggregation, and run datetime SQL
  (write-pd4castr-model-view), and not for a run that fails on the platform
  after publish.
compatibility: Requires Node.js 20.18 or newer, a running Docker daemon, the @pd4castr/cli package, and network access to the pd4castr API.
---

# Author a pd4castr model

A model is a Docker container. The platform hands it input files over HTTP,
runs it, and takes back a JSON array of forecast rows. The
[authoring docs](https://docs.v2.pd4castr.com.au/authoring/quick-start)
describe the platform. Each step links the page that covers it.

Copy this checklist and check off each step as it passes:

```text
Model progress:
- [ ] 1. Check the toolchain
- [ ] 2. Scaffold the project
- [ ] 3. Declare inputs and outputs
- [ ] 4. Declare axes (variant models only)
- [ ] 5. Read the inputs and write the output
- [ ] 6. Fetch input data (fetcher-backed inputs only)
- [ ] 7. Run pd4castr test until it passes
- [ ] 8. Publish
```

## 1. Check the toolchain

Docs: [Install the CLI](https://docs.v2.pd4castr.com.au/authoring/install-the-cli).

```sh
node -v
docker info
pd4castr --version
```

If the CLI is missing:

```sh
npm install -g @pd4castr/cli
```

## 2. Scaffold the project

Docs: [Scaffold a project](https://docs.v2.pd4castr.com.au/authoring/scaffold-a-project).

```sh
pd4castr init
```

`init` is interactive, takes no flags, and creates the project in a new
subdirectory. Run every later command from that directory.

In an unattended session, create the directory by hand instead: copy
[pd4castrrc.minimal.json](templates/pd4castrrc.minimal.json) to
`.pd4castrrc.json`, add a `Dockerfile` whose entrypoint runs the model, and
create `test_input/`.

## 3. Declare inputs and outputs

Docs: [Configuration file](https://docs.v2.pd4castr.com.au/authoring/configuration-file),
[Model inputs](https://docs.v2.pd4castr.com.au/authoring/model-inputs), [Model outputs](https://docs.v2.pd4castr.com.au/authoring/model-outputs).

Edit `.pd4castrrc.json`. Declare one entry in `inputs[]` per file the container
reads and one entry in `outputs[]` per forecast series it writes.

- Set `forecastVariable` to `price`, whatever the model forecasts. It is the
  only value the CLI accepts.
- Every `key` matches `^[a-z0-9][a-z0-9_-]*$`.
- Write a static input's `file` path relative to the project root.
- Never rename a `key` on a published model. The key carries a series across
  revisions, so renaming it starts a new series and orphans the history.
- Keep `timeHorizon` fixed. A new revision cannot change it.

## 4. Declare axes

Docs: [Model axes](https://docs.v2.pd4castr.com.au/authoring/model-axes).

Skip this step unless one trigger should run the model once per value of a
dimension, such as a scenario.

Add one entry to `axes[]` per dimension:

```json
{
  "key": "scenario",
  "type": "string",
  "valuesFrom": { "kind": "static", "values": ["low", "central", "high"] },
  "default": "central"
}
```

- `type` is `number`, `string`, or `boolean`. A number value is a whole
  number, zero or more. A string value matches `^[A-Za-z0-9][A-Za-z0-9_.-]*$`.
  An axis keeps its `type` across every revision of the model group.
- `default` is one of the `values`. It selects the variant a consumer gets
  when they pass no axis selector.
- Each trigger queues one run per combination of axis values, plus one run
  per sensitivity on each. Keep that product at 200 runs or fewer, and each
  axis at 100 values or fewer.

Partition a static input or a dataset by an axis so each variant reads its own
rows:

```json
"axes": { "scenario": { "kind": "partition", "column": "scenario" } }
```

- The input declares `parquet` for both `uploadFileFormat` and
  `targetFileFormat`. A dataset declares `fileFormat` `parquet`.
- The partition column is in the file, its type matches the axis `type`, and
  it holds every declared value of the axis.
- Partition each input or dataset by one axis at most. A fetcher-backed input
  cannot be partitioned.
- The container gets no variable that carries the variant. A partitioned
  input's URL serves only that variant's rows, with the partition column still
  in the file. Read the variant from that column.

Adding or removing an axis, or moving its `default`, needs a `new-revision`
publish. `update` rejects it. Removing a value under
`update` prints a warning, and the old runs of that variant can no longer be
selected from the run history.

## 5. Read the inputs and write the output

Docs: [Model inputs](https://docs.v2.pd4castr.com.au/authoring/model-inputs), [Model outputs](https://docs.v2.pd4castr.com.au/authoring/model-outputs).

The platform sets one environment variable per input and one for the output:

| Variable | Action |
| --- | --- |
| `INPUT_<KEY>_URL` | GET it to read that input's file. |
| `OUTPUT_URL` | PUT the finished forecast to it as JSON. |

`<KEY>` is the input's `key` uppercased, with each `-` replaced by `_`. An
input keyed `raw-data` arrives as `INPUT_RAW_DATA_URL`. The file at that URL
is in the input's `targetFileFormat`. Read the URL from the variable; do not
build it.

GET every declared input. PUT the output with `Content-Type: application/json`:

```python
import os, json, urllib.request

history = json.load(urllib.request.urlopen(os.environ["INPUT_HISTORY_URL"]))

rows = build_forecast(history)

request = urllib.request.Request(
    os.environ["OUTPUT_URL"],
    data=json.dumps(rows).encode(),
    headers={"Content-Type": "application/json"},
    method="PUT",
)
urllib.request.urlopen(request)
```

The output body is a JSON array of objects. Every row carries:

- `forecast_datetime` in one of these shapes. `T` or a space separates the date
  and the time, seconds are required, and a fraction of up to nine digits is
  optional. A value with no offset is read as UTC.

  ```text
  2024-01-01T00:30:00Z
  2024-01-01T00:30:00.000Z
  2024-01-01T00:30:00
  2024-01-01T10:30:00+10:00
  ```

  In Python, write a UTC datetime with `strftime("%Y-%m-%dT%H:%M:%SZ")`.
  `isoformat()` on a zone-aware datetime already ends in `+00:00`, so adding
  `Z` to it gives an invalid value.

- A value for every `key` declared in `outputs[]`. Declaring five keys and
  emitting four fails `pd4castr test` with
  `Output is missing configured output variables:`. Emit the missing values or
  remove the entries from `.pd4castrrc.json`.

The CLI checks the shape and the keys on the first row that carries
`forecast_datetime` only. Make every row match it.

## 6. Fetch input data

Docs: [Authentication](https://docs.v2.pd4castr.com.au/authoring/authentication), [Model inputs](https://docs.v2.pd4castr.com.au/authoring/model-inputs).

Skip this step when every input is a static file you supply.

```sh
pd4castr login
pd4castr fetch
```

`fetch` writes each fetcher-backed input to
`test_input/<key>.<targetFileFormat>`.

In an unattended session:

```sh
export PD4CASTR_CLIENT_ID=...
export PD4CASTR_CLIENT_SECRET=...
pd4castr login --client-credentials
pd4castr fetch
```

`login` takes `--client-credentials`, `--client-id`, and `--client-secret`,
and no other flags.

Read the fetched file before writing model code against it. Its shape is
whatever the query returned.

## 7. Run pd4castr test until it passes

Docs: [Testing your model](https://docs.v2.pd4castr.com.au/authoring/testing-your-model).

```sh
pd4castr test
```

`test` builds the image for `linux/amd64`, serves each input over a local
webserver, runs the container, and validates what it PUTs back. It serves a
static input from its `file` path and a fetcher-backed input from
`test_input/`. The output lands in `test_output/`.

1. Run `pd4castr test`.
2. If it fails, find the message in
   [test-and-publish-errors.md](references/test-and-publish-errors.md), apply
   the fix, and run again.
3. Proceed only on `Model I/O test passed` followed by
   `Model output validation passed`.

`✘ Input Fetched - <key>.<targetFileFormat>` means the entrypoint did not GET
that input's URL. Fix the entrypoint, not `.pd4castrrc.json`.

`test` needs no login until the model is published. The first `test` or
`fetch` after the first publish fails with
``Not authenticated. Please run `pd4castr login` to login.`` until you log
in. It then migrates `.pd4castrrc.json` against the published model and
rewrites the file. Commit that change.

Build through the CLI, not with `docker build` directly. The CLI pins the
platform to `linux/amd64`, so on an Apple Silicon machine the build is
cross-compiled and a dependency with no amd64 wheel fails here rather than on
the platform.

## 8. Publish

Docs: [Publishing](https://docs.v2.pd4castr.com.au/authoring/publishing),
[Run modes and scheduling](https://docs.v2.pd4castr.com.au/authoring/run-modes-and-scheduling).

```sh
pd4castr publish
```

The first publish asks `Do you want to continue?`, defaulting to no. Pass
`--non-interactive` to skip the prompt. It creates the model group at revision
0, reruns the container test, validates the queries and any axes, registers
the model, pushes the image, uploads static inputs and datasets, and triggers
a first run. A validation failure stops the publish before the model is
registered. A missing dataset file fails later, after the model is registered
and the image pushed.

On a model that already exists, the publish lands in one of two ways:

| Action | Effect |
| --- | --- |
| `update` | Replaces the current revision. Its inputs and outputs are recreated, and its earlier run history is kept but no longer shown in the app. |
| `new-revision` | Creates the next revision and keeps the old one readable. |

Interactively, the publish asks
`Do you want to update the existing revision or create a new one?`, then
`Are you sure you want to continue?`. With `--non-interactive`, pass
`--publish-action update` or `--publish-action new-revision` instead. The CLI
reads `--publish-action` only with `--non-interactive`.

Add `--skip-trigger` to skip the explicit run trigger. An `AUTOMATIC` model
still runs once its input files are processed.

The publish rewrites `.pd4castrrc.json`: it writes `$$id`, `$$modelGroupID`,
`$$revision`, and `$$dockerImage`, and can overwrite `displayTimezone` with
the value the server resolved. It writes them as soon as the model is
registered, before the image push, so commit the file even when the push
fails. Never hand-edit a `$$` field. A publish from a checkout with no `$$`
fields registers a second, duplicate model. A publish from a checkout with
stale `$$` fields fails with `OUT OF SYNC`.

The full unattended sequence for an existing model, on a runner with Node.js
20.18 or newer and Docker:

```sh
npm install -g @pd4castr/cli
pd4castr login --client-credentials
pd4castr fetch
pd4castr test
pd4castr publish --non-interactive --publish-action new-revision
git add .pd4castrrc.json
git commit -m "chore: publish revision"
```

In CI, run the commit step even when the publish step fails, and give the job
permission to push it.

After a publish:

1. Read every warning the publish printed. `N grant(s) pruned during publish`
   means a grantee lost access to a view, sensitivity, or input key that no
   revision declares any more, or to an axis or axis value the newest
   revision dropped. Report it to the user.
2. If the publish fails, find the message in
   [test-and-publish-errors.md](references/test-and-publish-errors.md), apply
   the fix, and publish again.

## Bundled files

- [test-and-publish-errors.md](references/test-and-publish-errors.md) maps
  each `test` and `publish` failure and warning to its cause and fix.
- [pd4castrrc.minimal.json](templates/pd4castrrc.minimal.json) is the smallest
  `.pd4castrrc.json` that publishes.
