---
tier: tale
title: Prefix epic phase descriptions with the phase slug ID
goal: '`sase plan validate --explain` and the `/sase_beads` skill teach agents to
  start every epic phase description with `<slug_id>: ` instead of quoting the phase''s
  full section title, and the repo''s own fixtures follow the new convention.'
---

- **AGENTS:**
  - [bbugyi200.athena.lf](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.lf/README.md)
  - [bbugyi200.athena.lf--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.lf.md#member-code)
- **COMMITS:**
  - [4b6c2c3](https://github.com/sase-org/sase/commit/4b6c2c3cadf2ea71e9ff1d9f99df0f8ca5cb0a53) — docs: standardize epic phase description prefixes

# Plan: Prefix epic phase descriptions with the phase slug ID

## Problem

The authoring guidance printed by `sase plan validate <epic> --explain` currently tells planning agents to name a
phase's plan-body section inside its `description`. Agents follow it literally, so epic phase descriptions read like:

```
'One critical section for bead mutation, commit, and integration' section: put the bead-store worktree materialization
and its git commit inside the same store-write-lock critical section that SDD integration already uses, ...
```

The quoted title is pure duplication: the phase's `title` sits directly above the `description` in the plan frontmatter,
and ACE already renders the title on the row above the description (see the epic detail pane, where the phase's slug
`serialize` is displayed under its title). The wasted prefix crowds bead descriptions, `sase bead show` output, and
every ACE panel that renders them.

The convention should instead be the phase's own slug ID:

```
serialize: put the bead-store worktree materialization and its git commit inside the same store-write-lock critical
section that SDD integration already uses, ...
```

The slug is short, unambiguous, and resolvable: an agent that reads `serialize: ...` finds the matching `phases:` entry
in the plan frontmatter and, from its `title`, the corresponding section of the plan body.

## Background: where this convention lives

Nothing in the codebase parses or generates the `'<title>' section: ` prefix — it exists purely as instruction text plus
copies of it in test fixtures:

- `src/sase/main/plan_explain.py` — `EPIC_PLAN_EXPLANATION`, the text `--explain` prints (both the worked YAML example
  and the prose rule on line 50).
- `src/sase/bead/epic_from_plan.py:127` — copies each authored `description` verbatim into the phase bead (falling back
  to `generated_phase_description()` when a phase omits one). No transformation, so no code change is needed here.
- `src/sase/xprompts/skills/sase_beads.md` — the `/sase_beads` skill source; today it documents `--description` without
  mentioning any phase-description convention.
- Test fixtures that hand-author epic frontmatter in the old style: `tests/test_plan_validate.py`,
  `tests/test_plan_command_handler.py`, `tests/plan_validation_helpers.py`,
  `tests/test_bead/test_cli_work_epic_summary.py`.

Because the prefix is convention rather than schema, this change is instruction text plus fixtures. Deliberate
non-goals: do **not** add a `plan validate` diagnostic that enforces the prefix, do **not** change
`generated_phase_description()`, do **not** strip or reformat descriptions in ACE rendering, and do **not** rewrite
descriptions in historical plan files under `sase/repos/plans/`.

## Step 1 — Rewrite the epic authoring guidance

In `src/sase/main/plan_explain.py`, inside `EPIC_PLAN_EXPLANATION`:

1. Change the three example `description` values to use their phase's slug ID as the prefix, leaving the summaries and
   every other field untouched:
   - `description: "core: implement workspace selection and safety guards."`
   - `description: "cli: add the CLI flow and progress reporting."`
   - `description: "smoke: exercise successful and guarded cleanup."`
2. Replace the prose sentence that currently reads:

   > Give every phase a `description` that names its section in the plan body and briefly summarizes that section; do
   > not reference the plan file itself because `sase bead show` already displays it.

   with guidance for the new convention. Keep it in the same paragraph and preserve the surrounding sentences about
   unique slugs, dependency ordering, and `size:`. Use wording equivalent to:

   > Give every phase a `description` that starts with that phase's own `id` followed by `: `, then briefly summarizes
   > the phase's section of the plan body. Do not quote or repeat the section title — the phase's `title` already names
   > that section — and do not reference the plan file itself because `sase bead show` already displays it.

Keep the module's existing line wrapping style (the file wraps prose near 120 columns inside the string literal; ruff's
line-length rule does not apply to these lines, as the current example lines already exceed 88 characters).

## Step 2 — Teach the convention in the `/sase_beads` skill

Edit `src/sase/xprompts/skills/sase_beads.md` (the skill source — never the generated `SKILL.md` copies). Add a new
`## Phase Bead Descriptions` section immediately after the existing `## Types` section, before `## Commands`, so it sits
next to the plan/phase type discussion. It must convey:

- An epic phase bead's description comes from the matching `phases:` entry in the epic plan file's frontmatter.
- Convention: a phase description starts with that phase's slug ID followed by `: `, then a short summary — for example
  `serialize: put the bead-store worktree materialization inside the store-write-lock critical section.`
- The prefix identifies the phase without repeating its `title`, which already names the phase's section in the plan
  body; resolve a slug to its section by looking the ID up in the plan frontmatter that `sase bead show` displays.
- Authoring or editing a phase bead description by hand uses the same prefix. Include short `sase bead create` and
  `sase bead update` examples with a slug-prefixed `--description`, e.g.
  `--description "login: add the endpoint and its auth checks."`
- The prefix applies to phase beads only — plan/epic bead descriptions (which carry the plan's goal) do not take it.

Match the file's existing markdown conventions: 120-column prose wrapping, fenced `bash` blocks, and `sase bead` (not
`.venv/bin/sase bead`) in examples.

## Step 3 — Regenerate and deploy the generated skill files

The `/sase_beads` SKILL.md files are generated from the source edited in Step 2, per the `generated_skills` long-term
memory note (read it with `/sase_memory_read` before this step):

```bash
sase skill init --force
chezmoi apply
```

`sase skill init --force` renders the per-provider skill files into their managed locations; `chezmoi apply` deploys the
chezmoi-managed copies. If you need to inspect or edit anything inside the chezmoi repo itself, open it with the
`/sase_repo` skill first and use only the path it prints. Do not hand-edit any generated `SKILL.md`.

## Step 4 — Update fixtures and skill-content assertions

Bring the repo's own epic fixtures onto the new convention so future agents copying them do not reintroduce the old
prefix.

1. `tests/test_plan_validate.py`
   - In `VALID_EPIC`, change the two descriptions to `"core: build the shared validation engine."` and
     `"cli: wire the validator into the command."`.
   - `test_facade_rehydrates_normalized_epic_phases` strips the `cli` description by exact string match; update that
     literal to the new text (`    description: "cli: wire the validator into the command."\n`) so the phase still ends
     up description-less.
2. `tests/test_plan_command_handler.py` — in `VALID_EPIC`, use
   `description: "implementation: build and verify the complete capability."`.
3. `tests/plan_validation_helpers.py` — in `VALID_EPIC_PLAN`, use
   `description: "implementation: deliver and verify the approved implementation."`.
4. `tests/test_bead/test_cli_work_epic_summary.py` — build `phase_descriptions` from the phase IDs instead of the
   titles, keeping the descriptions distinct per phase (the test asserts each one appears in the rendered clan detail):

   ```python
   phase_descriptions = tuple(
       f"{phase_id}: deliver {title.lower()} from the committed plan."
       for phase_id, title in zip(phase_ids, phase_titles, strict=True)
   )
   ```

   Pass `strict=True` to satisfy ruff's `zip`-without-strict rule.

5. `tests/main/test_init_skills_sources.py` — the parametrized `sase_beads` entry asserts required phrases in the skill
   source. Add one or two phrases drawn verbatim from the Step 2 text so the convention is regression-protected (for
   example the `login: add the endpoint and its auth checks.` example line, plus a distinctive fragment of the prose
   rule). The phrases must match the authored markdown exactly, including punctuation.

No test currently asserts on the prose of `EPIC_PLAN_EXPLANATION` itself — `tests/test_plan_validate.py` compares
`--explain` output against the constant — so Step 1 needs no assertion updates beyond the fixtures above.

## Verification

Run from the workspace checkout:

```bash
just install          # ephemeral workspace: dependencies may be stale
just check
```

Then confirm the guidance renders as intended end to end:

```bash
sase plan validate tests/... --explain    # or any scratch epic plan file
```

Expect the printed epic example to show `description: "core: implement workspace selection and safety guards."` and the
prose rule to describe the `<id>: ` prefix. Also spot-check that the deployed `/sase_beads` skill file picked up the new
section after Step 3 (`sase skill status` reports generated skill source/target status).

## Done when

- `sase plan validate <epic> --explain` instructs agents to prefix each phase `description` with `<slug_id>: ` and no
  longer asks them to name the section title.
- `/sase_beads` documents the convention for reading and hand-authoring phase bead descriptions, and its generated
  copies are regenerated and deployed.
- Every in-repo epic fixture uses slug-prefixed descriptions, and `just check` passes.
