---
tier: tale
title: Gate editable sase update on the pinned sase-core revision
goal:
  "`sase update` never fast-forwards sase past the sase-core revision its local core
  checkout can build, verifies the rebuilt sase_core_rs exposes every binding sase
  requires, and never restarts ACE or the scheduler onto a core that fails that check."
size: medium
proposed_by: bbugyi200.athena.0wf
create_time: 2026-10-04 14:10:24
status: wip
---

# Plan: Stop `sase update` from advancing sase past the Rust core it can build

## Problem

On apollo, ACE crashes as soon as `(` is typed in the prompt input:

```
_prompt_text_area_key_pairing.py:186 _try_prompt_text_pair_edit
  -> _argument_syntax_editing.py:78 plan_argument_list_continuation_edit
  -> core/rust.py:74 require_rust_binding("argument_list_continuation_edit")
AttributeError: sase_core_rs is importable but does not expose binding
'argument_list_continuation_edit'; the installed wheel is stale ...
```

`3d318f2d39 feat(ace): continue macro argument lists with parenthesis` started calling
that binding and moved `sase-core-revision.txt` to sase-core `2f16dc4a` ("feat(editor):
continue existing macro argument lists"), the commit that adds it.

## Root cause (verified on apollo)

1. **The primary sase-core checkout is wedged.** `~/projects/github/sase-org/sase-core`
   has uncommitted edits left by an agent at 2026-10-03 18:19–18:24 UTC (`M lib.rs`,
   `M prelude.rs`, `M vcs/mod.rs`, `M vcs/tests.rs`, `?? agent_session_manifest.rs`).
   They are an early draft of upstream `d7f2dbf0`. The checkout is stuck at `f50782f7`,
   5 commits behind `origin/master`, and does not contain the pin `2f16dc4a`. Why agent
   edits land in the primary checkout at all is tracked separately in task bead
   **sase-1g1**. That is out of scope here.
2. **`sase update` advances sase anyway.** `plan_dev_update`
   (`src/sase/dev_update/plan.py`) skips the dirty core root ("checkout has local
   changes"), but it still marks the host `sase` root actionable and fast-forwards it.
   Nothing compares the host's incoming `sase-core-revision.txt` with the core checkout
   the extension will be built from. The apollo journal
   (`~/.sase/logs/dev_update.jsonl`, runs at 16:57 and 17:13 UTC on 2026-10-04) shows
   `sase-core-rs` skipped (`0.36.4+1.gf50782f7e.dirty` vs `0.36.5+1.g2f16dc4a3`) while
   `sase` moved to `21779e1923` and then `aa8b98ffd5`.
3. **The rebuild verifies too little.** `_reconcile_steps` sets
   `rebuild_core = python_changed and dev_core_present`. That means
   `just rust-dev-install-uv-tool` rebuilt `sase_core_rs.abi3.so` (mtime 17:13 UTC) from
   the stale, dirty tree. `_rust_health_check_step` only runs `import sase_core_rs`, so
   the update reported success. ACE then restarted onto the new Python code with an
   extension that lacks `argument_list_continuation_edit`.

ACE loads that call from a keystroke handler and does not guard it, so the stale build
surfaces as a crash. Per the accepted `decisions:rust-core-required` record ("a strict
PyO3 loader with no soft-fail path"), the fix is not a TUI fallback. The fix is to never
put sase code and the core build out of step, and to fail loudly when they are.

## Operational remediation (user-run on apollo, not part of the coding task)

The coding agent must not modify apollo or any primary checkout. The user restores
apollo with:

```bash
cd ~/projects/github/sase-org/sase-core
git stash push --include-untracked -m "orphaned agent-session-manifest draft (superseded by d7f2dbf0)"
sase update            # core is now clean and behind, so it fast-forwards and rebuilds
~/.local/share/uv/tools/sase/bin/python ~/projects/github/sase-org/sase/tools/check_sase_core_rs_bindings
```

Then restart ACE (`Q`).

## Changes

### 1. Core-pin helper — new `src/sase/dev_update/core_pin.py`

Small, subprocess-only host helpers. Use `run_git`/`_git` conventions from
`sase.version._git`, with bounded timeouts.

- `read_core_pin(host_root: Path, ref: str) -> CorePin | None`
  - Resolve the pin file path from the host checkout's project-local linked-repo config.
    Use the entry named `sase-core` that declares `revision_pin` (today
    `sase-core-revision.txt` in `sase/sase.yml`).
  - Reuse `revision_pins_for_project` in `sase/finalizers/commit_revision_pin.py`, or
    the `_linked_repo_config` primitives it wraps. Do not hardcode the filename.
  - Read the pin's content at `ref` via `git -C <host_root> show <ref>:<pin_file>`.
  - Return the pin file and the 40-hex SHA. Return `None` when there is no pin config,
    the file is missing at `ref`, the content is malformed, or git errors. Missing pin
    information never blocks an update.
- `core_contains_revision(core_root: Path, sha: str, ref: str) -> bool | None`
  - Return `False` when the object is absent locally (`git cat-file -e <sha>^{commit}`
    fails) or when `git merge-base --is-ancestor <sha> <ref>` exits 1.
  - Return `True` when it exits 0.
  - Return `None` on any other git error or timeout (fail-open: no gate).
- Format SHAs consistently: a 12-character short form in messages.

### 2. Planner gate — `src/sase/dev_update/plan.py`

In `plan_dev_update`, after the root plans and package plans are built and **before**
`_finish_check_step` and `_reconcile_steps`, apply the gate:

- **Core checkout the build will use:** the git root of the editable core record if one
  exists. Otherwise use `_core_checkout_dir(host_record)` (the stale-wheel restore
  path). If there is neither, skip the gate.
- **Core target ref:** the core root's upstream ref if that root is actionable, else
  `HEAD`.
- **Host target ref:** the host root's upstream ref if the host root is actionable, else
  `HEAD`.
- **If the host root is actionable** and the pin at the host target ref is not contained
  in the core target ref (`False`):
  - Downgrade the host root plan and every package in it to `skipped`.
  - Use a reason such as:
    `needs sase-core <pin12> (sase-core-revision.txt), which the sase-core checkout does not contain (<core skip reason, e.g. checkout has local changes>); clean or update the sase-core checkout, then rerun sase update`.
  - When the core root is actionable but its upstream lacks the pin, say that instead.
  - Because reconcile steps derive from actionable packages, a blocked host no longer
    triggers the uv reinstall or the core rebuild. Plugin roots are unaffected.
- **If the host is not moving** (already current, or skipped for another reason) and the
  host `HEAD` pin is not contained in the core target ref (apollo's current state):
  - Append to the core package's skip reason, for example
    `checkout has local changes; it does not contain sase-core <pin12> required by the installed sase (sase-core-revision.txt), so sase_core_rs may be missing bindings`.
  - This reuses the existing skip-reason surfaces: CLI `_dev_plan_table`, the TUI
    confirm preview (`plugins_browser_comprehensive_update_preview.py`), progress check
    rows, and the journal. No new UI field is needed.
- **If containment is `True` or `None`, or no pin is found:** behavior is unchanged.

### 3. Binding verification after a rebuild — `plan.py`, `models.py`, `reconcile.py`, `execute.py`

- Add the `DevReconcileStepKind` value `"rust_binding_check"`.
- When `rebuild_core` and the host source root exist and
  `<host_root>/tools/check_sase_core_rs_bindings` is a file, plan a step right after
  `rust_health_check`:
  - Label: "Verify sase-core-rs exposes the bindings sase requires".
  - Command:
    `(<tool python>, <host_root>/tools/check_sase_core_rs_bindings, "--src", <host_root>/src/sase, "--remedy", <text pointing at cleaning/updating the sase-core checkout and rerunning sase update>)`.
  - It has no repair command. Restoring a published wheel cannot add newer bindings.
  - Omit the step when the tool is absent. The scan takes about 0.7s and imports only
    `sase_core_rs`.
- In `run_reconcile_steps`, run the binding check whenever the extension ended up
  importable:
  - That includes the existing "warned" health path where the build failed but the old
    extension still imports, and the path where the repair succeeded.
  - Skip it when the import itself is still broken.
  - A non-zero exit is a failure. Join it with any pending failure using the tool's
    stderr tail (it lists the missing names) and finish the step `failed`.
- Return the verification state from `run_reconcile_steps`: verified, failed, or not
  run.
- Add `DevUpdateResult.core_bindings_verified: bool | None = None` (`None` = not run)
  and set it in `execute_dev_update`.
- Record the field in the journal result record (`src/sase/dev_update/journal.py`).

### 4. Never restart onto a core that fails verification

A failed binding check happens after the merge, so `changed` is true. Restarting would
load the new code against the stale extension and reproduce the crash. The running
process still holds the old code and the old in-memory extension, so staying up is safe.

- **TUI.** In `src/sase/ace/tui/actions/update_run.py`, where `result.code_changed`
  triggers `_restart_after_update`: when the SASE leg's dev result reports
  `core_bindings_verified is False`, do not restart.
  - Show an `error` toast that names the stale core and the remedy.
  - Expose the condition through a small property on
    `ComprehensiveSaseUpdateResult`/`ComprehensiveUpdateResult`
    (`src/sase/ace/comprehensive_update.py`) instead of reaching into payload internals
    in the action.
- **CLI.** Wherever a dev result's `changed` feeds `restart_after_update`
  (`src/sase/main/update_handler_live.py`, and `update_handler_mode_switch.py` if the
  same result flows there):
  - Skip the scheduler restart with a distinct `RestartInfo` status, e.g.
    `skipped_core_bindings`, with a clear reason.
  - Update `update_types.py`, `_restart_detail` in `update_handler_support.py`, and the
    JSON renderer (`update_json.py`) for the new status.

### 5. Docs

- `docs/plugins.md` ("Updating sase and plugins"), in the editable-checkout and restart
  bullets, document:
  - the core-pin gate: the host is skipped with a reason when the core checkout lacks
    the pinned revision;
  - post-rebuild binding verification;
  - no restart after a failed verification.
- `docs/rust_backend.md`: in the editable `sase update` paragraph near
  `rust-dev-install-uv-tool`, mention the binding-verification step and that it reuses
  `tools/check_sase_core_rs_bindings`.

## Non-goals

- No soft-fail or `optional_rust_binding` fallback in the prompt key handlers
  (`decisions:rust-core-required`).
- No change to `sase update --to dev` mode-switch planning beyond the restart guard if
  it already consumes `DevUpdateResult`.
- No change to the Justfile or `_refresh-sase-core-checkout`.
- No fix for how agent edits reach the primary checkout (sase-1g1).
- No Rust/sase-core changes. This is host install orchestration, which stays in Python
  per `docs/rust_backend.md`.

## Tests

- **New `tests/dev_update/test_core_pin.py`**, using real temporary git repos:
  - host repo with `sase/sase.yml` (linked `sase-core` entry with `revision_pin`) plus a
    pin file;
  - core repo with commits A→B and pin = B:
    - HEAD at A → `False`;
    - HEAD at B → `True`;
    - unknown SHA → `False`;
  - malformed or missing pin file, and no pin config → `None`.
- **`tests/dev_update/test_plan.py`**, monkeypatching the core-pin helpers on
  `plan_mod`:
  - host behind + core dirty without the pin → host root and package skipped, reason
    names the pin and "checkout has local changes", and no `uv_tool_install`/rust steps
    are planned;
  - host behind + core behind with the pin in its upstream → both actionable;
  - host behind + core clean HEAD containing the pin → host actionable;
  - host current + core dirty without the pin → core reason enriched, nothing
    actionable;
  - pin `None` or containment `None` → existing behavior unchanged;
  - `rust_binding_check` is planned after `rust_health_check` when the tool file exists,
    and omitted when it does not.
- **`tests/dev_update/test_execute_reconcile.py`:**
  - binding check passes → verified `True`;
  - it fails → the failure carries the stderr tail, no repair runs, verified `False`;
  - build fails but the import succeeds → the binding check still runs;
  - the import fails and there is no repair → the check does not run (`None`).
- **`tests/dev_update/test_execute.py` / `test_journal.py`:** the result field is
  propagated and journaled.
- **`tests/ace/tui/test_update_run_actions.py`:** `code_changed` with
  `core_bindings_verified=False` → no restart and an error toast; the verified path
  still restarts.
- **CLI restart tests** (next to the existing `restart_after_update` / update-handler
  tests): the new skip status and its rendering, including JSON.

## Verification

- Run the repo's standard verification per the lint-and-test memory note.
- Add a CLI render test that feeds a gated plan through `_dev_plan_table`. It must
  confirm that `sase update -n` output shows the host skip reason and the enriched core
  reason.
