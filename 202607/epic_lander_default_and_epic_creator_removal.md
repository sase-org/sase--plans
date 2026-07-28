---
tier: tale
title: Point @epic_lander at @default and retire @epic_creator
goal: Epic land agents run on the @default model because the builtin epic_lander override
  tracks "@default", and the dead epic_creator builtin alias is gone from both the
  user's global config and SASE's compatibility scaffolding.
create_time: 2026-07-25 13:18:13
status: done
---

- **PROMPT:** [202607/prompts/epic_lander_default_and_epic_creator_removal.md](prompts/epic_lander_default_and_epic_creator_removal.md)
- **AGENTS:**
  - [bbugyi200.athena.ky](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.ky/README.md)
  - [bbugyi200.athena.ky--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.ky.md#member-code)
- **COMMITS:**
  - [14bf5f1](https://github.com/sase-org/sase/commit/14bf5f15c169579fe947609236c23f7d77ccb6f4) — feat(llm-provider)\!: retire epic_creator model alias

# Plan: Point `@epic_lander` at `@default` and retire `@epic_creator`

## Background

Two independent facts drove this request.

**1. `epic_lander` is pinned to a concrete model in the user's global config.** The chezmoi-managed global SASE config
(`home/dot_config/sase/sase.yml`, applied to `~/.config/sase/sase.yml`) currently has:

```yaml
llm_provider:
  model_aliases:
    builtin:
      claude_coder: gpt-5.6-sol
      # Remaining bead/epic role aliases are listed explicitly for readability;
      # each just tracks the default launch model (@default), the implicit default.
      coder: "@default"
      epic_creator: codex/gpt-5.6-sol
      epic_lander: "gpt-5.6-sol"
```

The comment claims the remaining role aliases just track `@default`, but `epic_lander` is pinned to `gpt-5.6-sol`, so
epic land agents always launch on Codex regardless of what `@default` resolves to. SASE's own implicit fallback for
`epic_lander` is already `@default` (`ROLE_ALIAS_FALLBACKS` in `src/sase/llm_provider/model_alias_policy.py`), and
`src/sase/default_config.yml` already documents `epic_lander: "@default"` in its commented example block. So **no
in-repo default needs to change for part 1** — only the user's global override, which currently shadows that default.

**2. `epic_creator` is dead code.** The `@epic_creator` implicit alias was retired when epic approvals stopped launching
`bd/new_epic` (see the `CHANGELOG.md` "Unreleased" breaking-change bullet: _"Epic approvals no longer launch
bd/new_epic, and the implicit @epic_creator model alias is no longer available"_). Verified with a repo-wide search:
nothing in `src/` reads, emits, or resolves the alias. What survives is a compatibility shim so a _configured_
`model_aliases.builtin.epic_creator` entry still renders as a role row instead of an unknown user alias:

- `src/sase/llm_provider/model_alias_policy.py` — `EPIC_CREATOR_MODEL_ALIAS_NAME`, `LEGACY_BUILTIN_ALIAS_NAMES`
- `src/sase/llm_provider/model_alias_config.py:243` / `:257` — legacy branches in `model_alias_kind()` and
  `model_alias_description()`
- `src/sase/llm_provider/alias_view.py` — import plus a `_ROLE_ALIAS_ORDER` slot
- `src/sase/llm_provider/config.py` — two re-exports
- `src/sase/config/sase.schema.json` — _"Legacy epic_creator entries are accepted but no longer used by SASE."_

The only thing keeping that shim alive is the user's own `epic_creator: codex/gpt-5.6-sol` config line. Remove the
config line and the shim has zero consumers.

**Answer to the question in the request: no, `@epic_creator` is not used anymore.** It resolves to nothing, is excluded
from `special_model_alias_names()` (asserted by `tests/llm_provider/test_config_alias_resolution.py`), and only exists
as presentation-compat for a stale config key. Delete it.

**Boundary note.** Per the Rust core backend boundary rule, this was checked against `sase-core`: that repo has no
model-alias code at all (a repo-wide search for `epic_lander` / `epic_creator` / `model_alias` returns nothing). Model
alias policy lives entirely in Python today, so no Rust change is in scope and none should be introduced.

## Scope

Two repos. Use the `/sase_repo` skill to open `chezmoi`, and use the path it prints for every chezmoi read and write. Do
not guess or clone the chezmoi path any other way.

## Task 1 — Repoint `epic_lander` and drop `epic_creator` in the chezmoi global config

In the chezmoi repo, edit `home/dot_config/sase/sase.yml` under `llm_provider.model_aliases.builtin`:

1. Change `epic_lander: "gpt-5.6-sol"` to `epic_lander: "@default"`.
2. Delete the `epic_creator: codex/gpt-5.6-sol` line entirely.
3. Leave `claude_coder: gpt-5.6-sol` and `coder: "@default"` alone, and leave the explanatory comment in place — it
   becomes accurate for the first time once `epic_lander` tracks `@default`.

Resulting block:

```yaml
builtin:
  # Builtin alias overrides only. User-defined aliases belong under
  # model_aliases.custom so the Models panel and %model completion can show a
  # description.
  # Coder follow-ups: a Claude-authored plan hands coding to Codex and a
  # Codex-authored plan hands coding to Claude Opus (migrated from the retired
  # worker_models lane in epic sase-5d).
  claude_coder: gpt-5.6-sol
  # Remaining bead/epic role aliases are listed explicitly for readability;
  # each just tracks the default launch model (@default), the implicit default.
  coder: "@default"
  epic_lander: "@default"
```

Keeping the explicit `epic_lander: "@default"` line (rather than deleting the key and relying on the implicit fallback)
matches the sibling `coder: "@default"` entry and the stated "listed explicitly for readability" convention. It is
behaviorally identical to omitting the key.

Commit that chezmoi edit separately — it is a different repo from `sase`.

**Note for the implementing agent:** chezmoi source edits do not take effect until applied. After committing, run
`chezmoi apply` (or tell the user to) so `~/.config/sase/sase.yml` picks up the change; otherwise `sase doctor` will
report chezmoi drift and epic land agents keep using the old pinned model. Confirm with
`sase doctor --check config.model_aliases` (or a full `sase doctor` run) that no model-alias problems are reported.

**Behavioral consequence to state in the handoff:** epic land agents below the `bead.big_epic_phase_threshold` phase
count stop launching on `gpt-5.6-sol` and start launching on whatever `@default` resolves to (the autodetected
provider's tier default, since this config sets no `builtin.default` and no `llm_provider.provider`). Large epics are
unaffected — they already route through `@big_epic_lander` → `@smartest`. This is the intended effect of the request,
not a regression.

## Task 2 — Delete the `epic_creator` compatibility shim from `src/`

All edits in the `sase` repo.

`src/sase/llm_provider/model_alias_policy.py`:

- Delete the `EPIC_CREATOR_MODEL_ALIAS_NAME = "epic_creator"` constant and its comment block.
- Delete `LEGACY_BUILTIN_ALIAS_NAMES = {EPIC_CREATOR_MODEL_ALIAS_NAME}` and its comment.
- Leave the module docstring's alias inventory correct (it does not currently name `epic_creator`; re-read it after the
  deletion to confirm it still reads cleanly).

`src/sase/llm_provider/model_alias_config.py`:

- Drop `LEGACY_BUILTIN_ALIAS_NAMES` from the `.model_alias_policy` import list.
- In `model_alias_kind()`, delete the branch
  `if name in LEGACY_BUILTIN_ALIAS_NAMES and name in get_builtin_model_aliases(): return "role"`.
- In `model_alias_description()`, delete the branch returning
  `"Legacy compatibility alias; SASE no longer launches this role."`.

`src/sase/llm_provider/alias_view.py`:

- Remove `EPIC_CREATOR_MODEL_ALIAS_NAME` from the `.config` import list.
- Remove its entry from `_ROLE_ALIAS_ORDER` (it sits between `CODER_MODEL_ALIAS_NAME` and
  `EPIC_LANDER_MODEL_ALIAS_NAME`). Every other entry keeps its relative order.

`src/sase/llm_provider/config.py`:

- Remove the `EPIC_CREATOR_MODEL_ALIAS_NAME as EPIC_CREATOR_MODEL_ALIAS_NAME` re-export.
- Remove the `LEGACY_BUILTIN_ALIAS_NAMES as _LEGACY_BUILTIN_ALIAS_NAMES` re-export.

After these edits, grep the repo for `EPIC_CREATOR` / `LEGACY_BUILTIN_ALIAS_NAMES` and confirm `src/` has zero hits.

## Task 3 — Give `epic_creator` a focused retired-key doctor warning

Once Task 2 lands, `model_alias_kind("epic_creator")` returns `"user"`, so a leftover
`model_aliases.builtin.epic_creator` entry in _anyone's_ config would fall into the generic branch in
`src/sase/doctor/checks_config_model_aliases.py` and produce misleading advice: _"is a custom alias in the
builtin-override map; move it to llm_provider.model_aliases.custom.epic_creator with a description"_. Moving a dead
alias into `custom` is the wrong fix — the right fix is deletion.

Mirror the existing retired-`phase_worker` handling in the same file:

1. In the `for alias in sorted(builtin_aliases):` loop (around line 178), add an `epic_creator` arm alongside the
   `phase_worker` arm, keyed `model_aliases.builtin.epic_creator`, with a message saying the alias is retired — SASE no
   longer launches an epic-creator role — and that the entry should be removed. Keep the arm before the
   `elif model_alias_kind(alias) == "user"` fallback so the generic message never fires for it.
2. In the target-validation loop (around line 267), extend the `phase_worker` skip so a retired `epic_creator` builtin
   entry also skips value validation, for the reason already documented there: the focused migration warning is the
   actionable truth and validating a dead target only adds contradictory follow-on advice.

Factor the two retired names into a small module-level constant if that reads better than repeating string literals;
keep the two messages distinct, since `phase_worker` has a successor alias and `epic_creator` does not.

_Judgment call flagged for the reviewer:_ this task is not strictly required by the request. It exists so the removal
degrades gracefully for any config that still carries the key. If the reviewer prefers a minimal diff, Task 3 can be
dropped without affecting Tasks 1, 2, 4, or 6 — but then update Task 5's doctor test expectations accordingly.

## Task 4 — Update the config schema description

In `src/sase/config/sase.schema.json`, in
`properties.llm_provider.properties.model_aliases.properties.builtin.description`, delete the sentence:

> `Legacy epic_creator entries are accepted but no longer used by SASE.`

Leave the rest of that description (the builtin alias inventory and the value-grammar paragraph) byte-identical. Do not
reformat the file — it is a large JSON document and an incidental reflow would bury the change.

## Task 5 — Update tests

`tests/doctor/test_checks_config_model_aliases.py`:

- `test_model_aliases_warns_on_retired_and_unknown_alias_references` currently seeds `"epic_creator": "@default"` and
  asserts `"model_aliases.builtin.epic_creator" not in by_key`. With Task 3 that inverts: assert the key _is_ present
  and that its message says the alias is retired and should be removed. Keep the existing `coder` / `phase_worker` /
  `epic_lander` assertions in that test unchanged.
- Check the rest of the file for other `epic_creator` seeds (there is a `builtin`/`custom` name list near line 224 —
  confirm whether it includes `epic_creator` and update it only if it does).

`tests/llm_provider/test_config_role_aliases.py`:

- Line ~250 asserts `resolve_model_alias("epic_creator") == "epic_creator"` inside a test about epic execution roles.
  That assertion still passes (an unknown alias resolves to its bare input), but it no longer belongs there. Move it
  into the existing unknown/retired-alias test in `tests/llm_provider/test_config_alias_resolution.py`
  (`test_unconfigured_retired_aliases_resolve_to_bare_input`, which already covers `worker` / `other`) so the epic-roles
  test only covers live roles.

`tests/llm_provider/test_config_alias_resolution.py`:

- The existing `assert "epic_creator" not in names` (line ~172) stays as-is; it is exactly the regression guard we want.

Add a new focused test asserting `model_alias_kind("epic_creator")` is `"user"` and `model_alias_description(...)`
returns `None` even when `model_aliases.builtin.epic_creator` is configured — that is the behavior the shim used to
suppress, and it is what Task 2 changes. Put it next to the other `model_alias_kind` tests.

`tests/llm_provider/test_alias_view_panel_rows.py` and the Models-panel PNG goldens were checked: none of them seed
`epic_creator`, so `_ROLE_ALIAS_ORDER` shrinking by one does not move any existing row and **no PNG golden should need
regeneration**. If `just test-visual` disagrees, stop and investigate rather than passing
`--sase-update-visual-snapshots`.

## Task 6 — Changelog

`CHANGELOG.md` already carries an "Unreleased" breaking-change bullet about `@epic_creator` no longer being available.
Add a second, separate bullet under `## Unreleased` → `### ⚠ BREAKING CHANGES` recording that configured
`llm_provider.model_aliases.builtin.epic_creator` entries are no longer accepted as builtin overrides and should be
deleted. Do not edit the existing bullet or any released section.

No changelog entry is needed for Task 1 — that is a change to the user's personal config, not to SASE.

## Verification

Run from the `sase` workspace, in order:

```bash
just install        # workspace venvs go stale between uses; required before anything else
just check
```

Targeted runs while iterating:

```bash
pytest tests/doctor/test_checks_config_model_aliases.py \
       tests/llm_provider/test_config_role_aliases.py \
       tests/llm_provider/test_config_alias_resolution.py \
       tests/llm_provider/test_alias_view_panel_rows.py \
       tests/test_model_picker_aliases.py \
       tests/test_models_panel_alias_rendering.py \
       tests/test_xprompt_model_completion.py \
       tests/test_config_schema_models.py
just test-visual    # confirms the Models-panel goldens are untouched
```

Then, after `chezmoi apply`:

```bash
sase doctor         # expect no model_aliases problems
```

## Done when

- The chezmoi global config sets `epic_lander: "@default"` and no longer mentions `epic_creator`, and the change is
  committed in the chezmoi repo and applied.
- `grep -rn "epic_creator\|EPIC_CREATOR\|LEGACY_BUILTIN_ALIAS_NAMES" src/` returns nothing.
- `sase doctor` reports a clear "retired, remove it" warning for a config that still sets
  `model_aliases.builtin.epic_creator` (Task 3), and reports no problems for the user's cleaned-up config.
- `just check` passes and the Models-panel PNG goldens are unchanged.
- `CHANGELOG.md` records the removal under "Unreleased" breaking changes.
