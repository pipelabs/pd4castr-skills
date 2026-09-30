<div align="center">
  <img src="./assets/logo.png" alt="pd4castr logo" width="92">

  <h1>pd4castr Agent Skills</h1>

  <p>Agent skills for the pd4castr forecasting platform.</p>

  <p>
    <a href="https://docs.v2.pd4castr.com.au">Documentation</a> •
    <a href="https://v2.pd4castr.com.au">Platform</a>
  </p>
</div>

---

A skill is a package of instructions an AI coding agent loads when it starts a
task the skill covers. These skills teach an agent to take a model from an empty
directory to a published revision, write the SQL behind its views and
sensitivities, and backtest and compare its runs.

## Install

Install one skill by name:

```sh
npx skills add pipelabs/pd4castr-skills --skill author-pd4castr-model
```

Running it without `--skill` opens a picker over every skill.

## Skills

| Skill | Use it when |
| --- | --- |
| [`author-pd4castr-model`](./skills/author-pd4castr-model/) | Taking a model from an empty directory to a published revision. |
| [`write-pd4castr-model-view`](./skills/write-pd4castr-model-view/) | Writing the SQL behind a model run view, a sensitivity, an input aggregation, or a run datetime query. |
| [`compare-pd4castr-runs`](./skills/compare-pd4castr-runs/) | Backtesting, or comparing runs, revisions, sensitivities, and variants. |

## Contributing

[`AGENTS.md`](./AGENTS.md) holds the scope and the authoring conventions: the
directory layout, what belongs in a description, and how a skill earns its
place against a documentation page.
