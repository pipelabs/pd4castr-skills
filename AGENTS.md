# AGENTS.md

Conventions for authoring skills in this repository. They follow the
[Agent Skills specification](https://agentskills.io/specification) and
Anthropic's
[skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices).

## Audience

Every skill here is read by an agent working for a pd4castr user.

## Scope

These skills cover the platform: the CLI, the configuration file, the API, the
SDK, and the shape of run data. They do not cover any particular model's subject
matter, and they carry no market or domain vocabulary. Skills for a given model
ship with that model.

Nothing here depends on infrastructure a user cannot reach. A skill that
needs a production database, a secret store, or an internal dashboard does not
belong in this repository.

## Layout

```
skills/<name>/
  SKILL.md        required
  references/     optional, documentation the agent reads on demand
  templates/      optional, files the agent copies
  scripts/        optional, files the agent runs
```

`<name>` is a verb-first phrase describing the task: `author-pd4castr-model`,
not `model-authoring` or `helpers`. The name is what a person types after
`--skill`, so it reads as an instruction. Every skill in the package follows the
same pattern.

## Frontmatter

```yaml
---
name: author-pd4castr-model
description: >
  What it does. When to use it. What it is not for.
compatibility: Requires Docker and the @pd4castr/cli package.
---
```

- `name` matches the directory name: lowercase letters, digits, and single
  hyphens, at most 64 characters.
- `description` is at most 1024 characters, written in the third person
  ("Builds a model", not "I build" or "You can build"). It carries three parts.
  What the skill does. When to use it, with the words a user's request would
  contain (`pd4castr test`, `.pd4castrrc.json`, `Dockerfile`). What it is not
  for, naming the sibling skill that covers it. The agent decides whether to load
  the skill from this field alone, so a description that omits the third part
  makes two skills fire on the same prompt.
- `compatibility` is present only when the skill needs a system tool or network
  access, and lists them in one sentence.
- No other keys.

## Body

The agent reads the body mid-task, so it is the procedure and the rules, and
nothing else.

- Keep it under 500 lines. Move reference material to `references/`.
- State what to do. Do not narrate why, and do not explain what the agent
  already knows.
- Give a multi-step task a checklist the agent can copy, then one section per
  step, in the imperative, one action per line.
- Wrap every validation in a loop: run it, look up the failure, fix, run again,
  and proceed only on a pass.
- Give one default per step. Add an escape hatch only when the default cannot
  work, and say when.
- Open each step with a `Docs:` line linking the page on
  https://docs.v2.pd4castr.com.au that covers it. The skill holds the procedure
  and the rules; the page holds the explanation.
- Use one term per concept throughout the skill and its references.
- Do not write anything that goes stale on a date. A rule tied to a version
  goes in a collapsed "Old patterns" section, not inline.

## Bundled files

- Link every bundled file from `SKILL.md` as a relative markdown link:
  `[test-and-publish-errors.md](references/test-and-publish-errors.md)`.
- Keep links one level deep. A reference file does not link to another
  reference file.
- Give a reference file longer than 100 lines a `## Contents` list at the top.
- Say whether the agent runs a script or reads it: "Run `scripts/x.py`" or
  "See `scripts/x.py` for the algorithm".
- File names describe their content: `test-and-publish-errors.md`, not
  `errors.md`.

## What earns a place

A skill is worth writing when an agent following the documentation alone would
still get it wrong. Three things qualify:

- An ordered procedure the docs spread across several pages.
- A rule the code enforces and no page states.
- A failure whose message does not name its cause.

A skill that restates one documentation page adds nothing and goes stale on its
own schedule. Link the page instead.

## Accuracy

Every command, flag, environment variable, and error string is quoted exactly as
the tool emits it. Check it against the CLI or the API before writing it down. A
skill that is confidently wrong costs more than no skill, because the agent
stops checking.

## Before merging

- Write three prompts a user would type that should load the skill. Run
  each without the skill and note what goes wrong. Run each with the skill and
  check the failure is gone.
- Run the prompts on more than one model. A small model shows where the skill
  is too thin. A large model shows where it over-explains.
- Check every command and error string in the skill against the current CLI.

## Writing

Sentence case. No em dashes. Backticks for identifiers, paths, and flags. No
greetings, no sign-offs, and no commentary on the skill itself.
