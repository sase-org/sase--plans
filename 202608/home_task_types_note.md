---
tier: tale
title: Generate the task-type memory note for project roots only
goal:
  sase memory init generates sase/memory/task_types.md only for SASE-managed project
  repositories, and retires the copy it previously generated at home and chezmoi-home
  roots.
size: medium
proposed_by: bbugyi200.athena.0bs
create_time: 2026-08-23 10:34:09
status: wip
---

# Stop generating the home `sase/memory/task_types.md` note

## Goal

`sase memory init` (and therefore `sase init`) must generate `sase/memory/task_types.md`
**only** for SASE-managed project repositories. Home and chezmoi-home roots must stop
rendering it, must stop inlining a `Task Bead Types (task_types)` section into their
`AGENTS.md` / provider shims, and must delete a previously generated copy so a single
`sase memory init` pass converges.

## Current behavior (what the implementer is changing)

- `src/sase/main/init_memory/root_rendering_task_types.py` renders two flavors of the
  same note: a project flavor from the committed catalog
  (`_project_task_type_snapshot_entries()`) and a home flavor from
  `_home_task_type_specs()` -> `machine_global_builtin_task_type_specs()`. Both memory
  roots always get a note.
- `generated_memory_note_relative_paths(include_project_memory=False)` in
  `root_rendering_notes.py` lists `sase/memory/task_types.md`, so home also _reserves_
  the path against `sase memory write` and the ACE Memory panel.
- The committed `sase/task_types.json` snapshot is already project-only (guarded by
  `include_project_memory` in `root_rendering.py`), so it needs no change.
- `sase/memory/artifact_relations.md` is the existing example of a short note that is
  rendered only when `include_project_memory` is true — mirror it rather than inventing
  a new shape.
- `root_planning.py` already has the retirement machinery for notes a root no longer
  manages: `_retired_note_paths` (project-only long notes) and
  `_retired_glossary_note_paths`. Their results flow automatically into
  `excluded_note_paths` (so the note is not inlined into `AGENTS.md` in the same pass),
  a `delete` `MemoryFileChange` in the plan, `ignored_paths` for the unreferenced/parent
  checks, and `initialize_memory_root`'s `_delete_retired_note_paths`.
- With `use_chezmoi: true` the home memory root **is** the chezmoi source tree
  (`CHEZMOI_HOME = ~/.local/share/chezmoi/home`), and `~/sase/memory/` is the applied
  copy. `init_memory_handler.py` already forwards `home_result.deleted_paths` to the
  chezmoi deploy, which stages the source deletion — but `chezmoi apply` leaves the
  already-applied target file behind. `_init_skills_apply.py` solves exactly this with
  `ChezmoiDeployBehavior(delete_targets=..., delete_target_root=Path.home())`;
  `sase memory init`'s `_deploy_to_chezmoi` does not pass those yet.

## Step 1 — render the note only for project roots

- `src/sase/main/init_memory/root_rendering_task_types.py`
  - Drop the `include_project_memory` parameter from
    `render_generated_task_types_memory_body()`; always render from
    `_project_task_type_snapshot_entries()`.
  - Delete `_home_task_type_specs()`.
- `src/sase/main/init_memory/root_rendering_notes.py`
  - Move `generated_task_types_memory_relative_path()` out of the always-included tuple
    in `generated_memory_note_relative_paths()` and into the `include_project_memory`
    branch; update the docstring so the project-only list names `task_types.md`.
  - Make `generated_task_types_body` an optional `str | None` in
    `generated_short_notes()` and add the key only when it is not `None`, exactly like
    `generated_artifact_relations_body`.
- `src/sase/main/init_memory/root_rendering.py::render_expected_memory_files`
  - Guard the lazy re-render fallback, the `note_overlay` entry, and the
    `MemoryExpectedFile` for the task-type note on `include_project_memory`, mirroring
    the `generated_artifact_relations_*` handling directly above it.
- `src/sase/main/init_memory/root_planning.py::memory_root_context`
  - Move the `render_generated_task_types_memory_body()` call and its blocker return
    under the existing `if include_project_memory:` block that renders
    `artifact_relations`; pass `None` through to `generated_short_notes()` and
    `render_expected_memory_files()` for every other root.

## Step 2 — delete the now-dead machine-global catalog replay

`machine_global_builtin_task_type_specs()` exists only to render the home flavor. Per
`sase/memory/symvision.md`, test references never keep a public symbol alive, so it has
to go rather than be whitelisted.

- Before deleting, confirm no linked repo consumes it. Open each of `sase-github`,
  `sase-telegram`, `sase-nvim`, and `sase-research-artifacts` with `/sase_repo` and grep
  for `machine_global_builtin_task_type_specs`. If a real external consumer exists, keep
  the symbol with a `# symvision: <repo-uri>` pragma instead of deleting it, and say so
  in the handoff.
- `src/sase/task_types/registry.py`: delete `machine_global_builtin_task_type_specs` and
  its `__all__` entry. Re-check whether `BuiltinTaskTypes` /
  `builtin_task_type_provenance` imports are still needed there and drop any that died
  with it.
- `src/sase/task_types/__init__.py`: drop the re-export from both the import block and
  `__all__`.
- `src/sase/task_types/_project_config.py`: drop the `include_local_layer` parameter
  from `apply_project_task_type_config()` and `_effective_raw_task_type_entries()`, plus
  the `if not include_local_layer and layer.name == "local": continue` skip. Nothing
  else passes a non-default value once step 1 lands.

## Step 3 — retire previously generated home copies

Add the recognition predicate and wire a third retirement helper into the existing
mechanism. Do **not** add a new delete path in `root_application.py`; reusing
`retired_note_paths` is what gets the exclusion from AMD sync and the unreferenced check
for free.

- `root_rendering_task_types.py`: add
  `is_generated_task_types_memory_content(text: str) -> bool`.
  - Parse the note with `parse_memory_note_text`; require `type == "short"`.
  - Require the body to contain both heading lines produced by rendering the packaged
    `memory-sase-task-types.template.md` with an empty `task_type_entries` value: its H1
    and its `## Types` heading. `format_generated_memory_markdown` emits heading lines
    verbatim, so both are stable.
  - Rationale for a heading signature rather than a byte comparison against a re-render:
    the previously written home body depended on that machine's `bead.task_types` config
    _and_ on whatever template shipped when it was last generated, so a byte comparison
    would silently refuse to retire notes written by an older SASE. A file that does not
    match the signature is left alone and keeps behaving as an ordinary hand-authored
    note, which matches how `_retired_note_paths` and `_retired_glossary_note_paths`
    already treat human-edited copies.
- `root_planning.py`: add
  `_retired_task_types_note_path(root, *, include_project_memory)` returning
  `(root / generated_task_types_memory_relative_path(),)` when `include_project_memory`
  is false, the file exists and is readable, and
  `is_generated_task_types_memory_content()` accepts it; `()` otherwise. Append it to
  the existing `retired_note_paths` tuple next to the two current helpers.

## Step 4 — remove the applied chezmoi target

- `src/sase/main/init_memory_handler.py::_deploy_to_chezmoi`: add a
  `delete_targets: Sequence[Path] = ()` parameter and forward it to
  `ChezmoiDeployBehavior` together with `delete_target_root=Path.home()`.
- At the `if inputs.use_chezmoi:` deploy site: build the live target list from
  `home_result.deleted_paths`, mapping only paths under `CHEZMOI_HOME` to
  `Path.home() / path.relative_to(CHEZMOI_HOME)` and skipping any that are not. Pass it
  to both `defer_chezmoi_paths(..., delete_targets=..., delete_target_root=Path.home())`
  and `_deploy_to_chezmoi(...)`. Keep passing the existing
  `(*written_paths, *deleted_paths)` union as `written_paths` so the source-side
  deletion is still staged and committed.
- This is a general fix for home-root retirements, not a task-type special case; keep it
  written that way.

## Step 5 — tests

New coverage (put the memory-init cases under `tests/main/`, reusing
`tests/main/init_memory_handler_helpers.py`):

- Home root: after `run_handler()`, `home_root/sase/memory/task_types.md` does not
  exist, the home `AGENTS.md` contains no `Task Bead Types` section, and a second
  `plan_memory()` reports no actions.
- Retirement: seed a home root with a generated-shaped `task_types.md`, then assert
  `plan_memory()` shows a `delete` action for it, `run_handler()` removes it, the home
  `AGENTS.md` does not inline it, and no unreferenced-memory blocker fires.
- Non-retirement: a hand-authored home `sase/memory/task_types.md` with a different H1
  survives `run_handler()` and is inlined as an ordinary short note.
- Chezmoi: extend `tests/main/test_init_memory_chezmoi.py` so a retired home source path
  produces the corresponding `~/sase/memory/...` live delete target.
- Project root unchanged: the project still generates both `sase/memory/task_types.md`
  and `sase/task_types.json`.

Updates and deletions:

- `tests/memory/test_mutation_generated.py`: drop `sase/memory/task_types.md` from the
  home generated set and keep it in the project set.
- `tests/ace/tui/test_memory_panel_catalog.py::test_home_snapshot_does_not_mark_project_only_generated_notes`:
  `sase/memory/task_types.md` is now _not_ in the home `generated_paths`.
- `tests/test_bead/test_task_type_end_to_end.py`: drop the `include_project_memory=True`
  kwarg from `render_generated_task_types_memory_body()` calls, and delete
  `test_home_task_type_note_omits_a_machine_global_disabled_builtin`.
- Delete `tests/task_types/test_machine_global_specs.py`.
- `tests/task_types/test_project_config.py`: delete
  `test_machine_global_replay_skips_the_local_layer`.
- Re-check `tests/main/test_init_memory_plan.py`, `test_init_memory_handler_outputs.py`,
  and `test_init_memory_committed_drift.py` for home README/AGENTS assertions that shift
  by one note.

## Step 6 — docs

- `docs/init.md`: in the memory-init bullet list, say the generated short
  `sase/memory/task_types.md` note is project-only alongside `sase/task_types.json`; and
  extend the retirement paragraph (currently about project-only long notes) to cover the
  task-type note.
- `docs/memory.md`: remove `task_types.md` from the "initialization always generates"
  sentence, add it to the SASE-managed-project list, and drop "the home-root note
  renders from the builtin catalog only".
- `docs/beads.md`: state that `sase memory init` generates the note for SASE-managed
  project repositories only, and correct the machine-global `bead.task_types` paragraph
  that currently claims the override reaches "the home-level instruction files".
- `docs/ace.md`: move `sase/memory/task_types.md` from the always-generated list into
  the project-only group of generated notes.

## Out of scope / needs the user

- Do **not** edit `sase/memory/*.md`, `AGENTS.md`, or the provider shims in this repo.
  The project flavor of the note is unchanged, so no memory-file edit is required.
- The user's own `~/sase/memory/task_types.md`, `~/sase/memory/README.md`, and
  `/home/bryan/CLAUDE.md` live in the chezmoi repo. They change only when the user runs
  `sase memory init` on this machine after this lands, which also produces a chezmoi
  commit. Flag that as the follow-up step in the handoff rather than doing it.

## Risks to surface in the handoff

- Agents that run outside a SASE-managed project (for example in a linked chezmoi repo
  workspace) lose the `Task Bead Types` section and the `/sase_new_task` pointer from
  home instructions. That is the intended narrowing, but it is a real behavior change
  and the user should confirm it.
- `#memory/task_types` stops resolving in home scope; it remains valid in a SASE-managed
  project scope.

## Verification

- `just install` first (ephemeral workspace), then `just check`.
- `just _lint-symvision` explicitly after the step 2 deletions.
- Run `just check-full` through `/sase_monitor` before landing — this change touches
  `init_memory`, the task-type registry, and the ACE memory panel, which is exactly the
  broadening set `just check`'s scoped lane is weakest on:

  ```bash
  sase monitor start --command 'just check-full' \
    --start-status TESTING --stop-status TESTED --next '...'
  ```
