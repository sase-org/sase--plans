---
tier: tale
title: Resilient, previewed, confirmed agent cleanup for `sase bead work` with `-Y/--yes-to-all`
goal: '`sase bead work` proactively cleans up old agents named the same as (or related
  to) the agents it is about to launch — killing/dismissing them and releasing stale
  registry reservations — after showing an itemized preview and getting y/n confirmation;
  the new `-Y/--yes-to-all` flag overrides both that cleanup prompt and the existing
  launch prompt, while `-y/--yes` still overrides only the launch prompt; and the
  "forced reuse cleanup could not resolve any concrete members" failure no longer
  occurs.'
create_time: 2026-07-23 12:03:01
status: wip
---

- **PROMPT:** [202607/prompts/bead_work_cleanup_confirm.md](prompts/bead_work_cleanup_confirm.md)

# Make `sase bead work` resilient: previewed, confirmed agent cleanup + `-Y/--yes-to-all`

## Problem

`sase bead work <epic>` relaunches an epic's deterministic phase/land agents. Before launching, it runs a "forced reuse
cleanup" (`prepare_bead_work_force_reuse` in `src/sase/bead/cli_work_cleanup.py`) that terminates and removes any prior
owners of the deterministic names it is about to claim — completed, dismissed, or still-live. Two things make this
brittle and opaque today:

1. **It hard-fails on stale/orphaned container reservations.** When a name it intends to reuse (e.g. `sase-8u.4.2`) is
   still reserved in the name registry as a _family container_ but no concrete on-disk member artifacts can be resolved,
   `_wipe_force_reuse_family()` raises:

   ```
   Error: forced reuse cleanup could not resolve any concrete members for agent family 'sase-8u.4.2'
   ```

   That aborts the whole command. This is the exact failure the user reported from `sase bead work sase-8u.4 -y`, and it
   should no longer happen.

2. **The destructive teardown is silent and unconfirmed.** On a real launch the cleanup kills live agents and deletes
   completed/dismissed artifacts with no itemized preview and no dedicated confirmation — only the generic
   `confirm_launch()` "Launch these agents?" prompt, which the existing `-y/--yes` already bypasses.
   `warn_force_reuse_collisions()` exists but is shown only on `--dry-run` and lists only _live_ collisions.

## Goals

1. **Resilience.** Never hard-fail on a stale/orphaned container reservation for a name we intend to reuse: release the
   stale reservation and proceed. Preserve the existing hard failures for genuine problems (wipe raised, wipe reported
   errors, name still reserved after we tried to free it) so we never launch into a guaranteed collision.
2. **Transparency.** Before performing any destructive action, print a clear, itemized list of every agent/reservation
   that will be dismissed, killed, or released, labeled by what will happen to it.
3. **Confirmation.** Add a dedicated y/n confirmation for the destructive cleanup, safe in non-interactive shells (no
   uncaught `EOFError`).
4. **Override semantics.** Add a new `-Y/--yes-to-all` flag that overrides **both** the new destructive-cleanup
   confirmation **and** the existing launch confirmation. The existing `-y/--yes` continues to override **only** the
   launch confirmation and must **not** bypass the destructive-cleanup confirmation.
5. **Don't break automated launches.** Non-interactive/automated callers (the approved-epic launch argv, `--json`, TUI
   plan launch, resume) must not block on the new prompt.

## Design

### Flag & override semantics

- Add `-Y/--yes-to-all` to the `sase bead work` argparse parser.
- Precedence: `yes_to_all` implies `yes`. `-y/--yes` alone does **not** imply `yes_to_all`.
- `--json` is inherently non-interactive; it must imply `yes_to_all` (today it already implies `yes` via
  `yes or json_output`). Fold `--json` into `yes_to_all` at the handler.

Resulting gate behavior in the launch path:

| Flags             | Destructive-cleanup prompt         | Launch prompt |
| ----------------- | ---------------------------------- | ------------- |
| (none, TTY)       | shown                              | shown         |
| `-y/--yes`        | **shown**                          | skipped       |
| `-Y/--yes-to-all` | skipped                            | skipped       |
| `--json`          | skipped (implies `-Y`)             | skipped       |
| (none, no TTY)    | refuse: "re-run with --yes-to-all" | n/a           |

The destructive-cleanup prompt is only shown when there is at least one destructive target (a live agent to kill, a
terminal/dismissed owner to remove, or a stale reservation to release). A fresh epic with no collisions shows neither
the preview nor the destructive prompt. Note: the non-destructive rewrite of `%id:!<n>` → `%id:<n>` in
`prepare_bead_work_force_reuse` always runs regardless — it is not a destructive action.

### Single wiring point

Both entry paths converge on `launch_epic_bead_work` (`src/sase/bead/cli_work_handler.py`), which is the only caller of
`confirm_launch()` and `prepare_bead_work_force_reuse`. The plan-file path (`work_from_plan_file` in
`src/sase/bead/cli_work_from_plan.py:335`) calls it, as do the TUI plan-launch action and the resume path. Thread a new
`yes_to_all: bool = False` parameter into `launch_epic_bead_work` and host the preview + destructive confirmation there,
just before the existing launch confirmation and the `force_reuse_cleanup` stage.

### Cleanup preview (non-destructive)

Add a preview that enumerates exactly what the subsequent `prepare_bead_work_force_reuse` will act on, **without
mutating anything**. It should target the same names the cleanup targets: the `%id:!<n>` force-reuse directive names
parsed from the rendered query (equivalently `expected_agent_names(plan)`) plus `legacy_epic_cleanup_names(plan)`. For
each name, classify using existing read-only helpers:

- **KILL (running):** name owned by a live agent — reuse `find_live_name_collisions(plan)` /
  `get_live_agent_name_subset`.
- **REMOVE (terminal/dismissed):** name owned by a done or dismissed agent — `find_named_agent(name)`.
- **Family members:** `find_agent_family(name)` returns concrete members — list each member and classify it as live-kill
  vs terminal-remove.
- **RELEASE (stale reservation):** name is reserved as a family (or clan) _container_ in the registry but resolves no
  concrete owner/members — `lookup_registered_name` / `is_name_reserved` / `get_reserved_family_names` /
  `get_reserved_clan_names` plus an empty `find_agent_family(name)`. This is the orphaned case behind the reported
  failure.

Return a small structured result (e.g. a `CleanupPreview` dataclass with a list of `(name, current_state, action)`
targets and a `has_destructive_targets` property). Keep the epic clan container itself (`plan.epic_id`) out of the
target set — it is the joinable clan container and is intentionally reused, not wiped (mirrors the existing
`declare_clan` logic and `allow_container_skip=True` for extra-cleanup names).

### Preview rendering + confirmation prompts

- Add a renderer that prints the itemized preview to stderr, matching the command's existing plain-`print` style (like
  `warn_force_reuse_collisions`). Group/label targets by action so it is unambiguous which agents are killed vs removed
  vs released, e.g.:

  ```
  Cleaning up existing agents before relaunching epic sase-8u.4:
    KILL    (running)   sase-8u.4.2      pid 12345, at <artifact-dir>
    REMOVE  (completed) sase-8u.4.3      done, at <artifact-dir>
    RELEASE (stale)     sase-8u.4.2      orphaned family reservation
  ```

- Add `confirm_cleanup()` alongside `confirm_launch()` in `src/sase/bead/cli_work_plan.py`, modeled on the TTY/EOF-safe
  patterns in `src/sase/prompt/cli_maintenance.py` (`_confirm`/`_stdin_is_tty`) and `src/sase/main/update_handler.py`
  (`_confirm_mode_switch`): if stdin is not a TTY, do not call `input()`; return a sentinel that makes the caller refuse
  with a clear message telling the user to re-run with `--yes-to-all`. Catch `EOFError` and treat it as "no". The prompt
  text should read like: `Proceed with dismissing/killing these agents? [y/N] `.
- While here, harden the existing `confirm_launch()` the same way (catch `EOFError`, treat a non-TTY without a yes-flag
  as a clear refusal rather than an uncaught exception). Today it can raise `EOFError` on closed stdin.

### Launch-path control flow (`launch_epic_bead_work`)

Insert between the plan-summary print and the existing launch confirmation:

1. Build the preview from the rendered `query`, `expected_agent_names(plan)`, and `legacy_epic_cleanup_names(plan)`.
2. On `--dry-run`: render the **full** preview (replacing the current
   `warn_force_reuse_collisions(find_live_name_collisions(plan))`, which shows only live collisions), then print the
   multi-prompt and return as today.
3. If `preview.has_destructive_targets`:
   - Render the preview to stderr.
   - If not `yes_to_all`:
     - Non-interactive stdin → print a clear refusal ("re-run with `--yes-to-all` to proceed non-interactively"), abort
       (return `False` / raise the same way an aborted launch does today) without wiping or launching.
     - Interactive → `confirm_cleanup()`; if declined, print `Aborted.` and return `False`.
4. Existing launch confirmation, now gated by `yes or yes_to_all`:
   `if not (yes or yes_to_all) and not confirm_launch(): print("Aborted."); return False`.
5. Existing `force_reuse_cleanup` stage runs `prepare_bead_work_force_reuse` (now resilient).

Ordering rationale: authorize the teardown first (it is what makes the relaunch possible), then the launch itself. With
`-y`, the user still sees and answers the destructive prompt but the launch prompt is auto-confirmed — exactly the
requested separation.

### Resilience fix (the reported failure)

In `src/sase/bead/cli_work_cleanup.py`, change the orphaned-container behavior so it releases the stale reservation
instead of raising:

- `_wipe_force_reuse_family(name)`: when `member_names` is empty (no concrete members to wipe), do **not** raise. Treat
  the family reservation as stale/orphaned and **release it** so the name becomes claimable, then
  `rebuild_name_registry()` and verify. Only if `name` is _still_ reserved after the release attempt (i.e. it was not
  truly orphaned and we could not free it) raise a clear `ForcedReuseCleanupError` — preserving correctness (never
  launch into a guaranteed collision).
- Apply the same "release if orphaned, else keep failing" treatment to the clan-container branch of
  `_wipe_force_reuse_owner` for an expected/legacy name that is a stale clan reservation with no live/concrete members
  (rare, but symmetric).
- **Determine the exact release mechanism by first reproducing the orphaned state** (see Implementation, step 0).
  Candidate mechanisms, to be chosen after reproduction: (a) a targeted registry-reservation removal for an entry with
  no backing artifacts/members, followed by `rebuild_name_registry()`; or (b) extending `wipe_agent_name_for_reuse` with
  an opt-in (e.g. `allow_stale_container`) that removes a container reservation and any residual base-named artifact
  when the container has no live/concrete members. Prefer the smallest change that makes the reproduced failure pass
  while leaving the genuine-failure guards intact.

**Invariant:** after a successful cleanup, none of the expected/legacy names remain reserved in the registry. If that
invariant cannot be met, fail loudly (as today). The resilience only suppresses the _specific_ "no concrete members to
wipe" abort when the reservation is genuinely orphaned and can be released.

### Automated / non-interactive callers

- `src/sase/bead/epic_launch.py`: change `build_epic_launch_argv` to emit `--yes-to-all` instead of `--yes`. This is the
  approved-epic launch command (run detached with stdin set to `DEVNULL`); it already carries user approval and must
  auto-confirm the cleanup. Update the parser epilog examples in `parser_bead.py` accordingly if they imply `--yes` is
  enough for an unattended relaunch.
- `handle_bead_work`: `--json` implies `yes_to_all` (fold `json_output` into `yes_to_all`).
- Thread `yes_to_all` through `work_from_plan_file` (`src/sase/bead/cli_work_from_plan.py`, including its internal
  helper hops at lines ~191/209/288/335/407/418) into `launch_epic_bead_work`.
- Other `launch_epic_bead_work` callers — `src/sase/ace/tui/actions/artifacts_plans.py:392` (TUI plan launch) and
  `src/sase/bead/cli_work_from_plan_resume.py:96` (resume) — are automated/approved launches; pass `yes_to_all=True` so
  they do not block. Verify each and set the argument explicitly.

## Files to change

- `src/sase/main/parser_bead.py` — add `-Y/--yes-to-all`; refine `-y/--yes` help to say it skips only the launch
  confirmation (not the cleanup confirmation); adjust epilog examples.
- `src/sase/bead/cli_work_handler.py` — `handle_bead_work` reads `yes_to_all`, computes
  `yes_to_all = yes_to_all or json_output`, passes `yes` and `yes_to_all` to both the plan-file and epic-id paths;
  `launch_epic_bead_work` gains `yes_to_all: bool = False` (thread through its self-recursive call), hosts the preview +
  destructive confirmation, and gates the launch confirmation on `yes or yes_to_all`.
- `src/sase/bead/cli_work_cleanup.py` — resilience fix in `_wipe_force_reuse_family` and the clan branch of
  `_wipe_force_reuse_owner`; add the non-destructive preview function + `CleanupPreview` type; keep
  `warn_force_reuse_collisions` or supersede it with the richer renderer.
- `src/sase/bead/cli_work_plan.py` — add `confirm_cleanup()` and the preview renderer; harden `confirm_launch()` for
  TTY/EOF.
- `src/sase/bead/cli_work_from_plan.py` — thread `yes_to_all` through `work_from_plan_file` and helpers into
  `launch_epic_bead_work`.
- `src/sase/bead/epic_launch.py` — `build_epic_launch_argv` emits `--yes-to-all`.
- `src/sase/ace/tui/actions/artifacts_plans.py`, `src/sase/bead/cli_work_from_plan_resume.py` — pass `yes_to_all=True`.
- `src/sase/bead/cli_work.py` and `src/sase/bead/cli.py` — re-export any new public symbols (`confirm_cleanup`, preview
  helpers) if they follow the existing facade pattern.

## Implementation steps

0. **Reproduce the failure first.** Write a focused test/fixture that seeds the exact broken state: a name registry that
   reserves an expected phase name (e.g. `<epic>.2`) as a family container while `find_agent_family(<epic>.2)` resolves
   no concrete members. Confirm it currently produces
   `forced reuse cleanup could not resolve any concrete members for agent family '<epic>.2'`. Use this to pin down
   precisely what keeps the reservation alive after `rebuild_name_registry()` (residual base artifact vs planned
   reservation vs registry entry), which determines the release mechanism.
1. Add the `-Y/--yes-to-all` argparse option and refine `-y/--yes` help + epilog.
2. Thread `yes_to_all` through `handle_bead_work` → `launch_epic_bead_work` and `work_from_plan_file`, plus the
   TUI/resume callers; fold `--json` into `yes_to_all`.
3. Implement the non-destructive preview + renderer + `confirm_cleanup()`; harden `confirm_launch()`.
4. Wire the preview/confirmation into `launch_epic_bead_work` (including the `--dry-run` full-preview change) with the
   gate behavior in the table above.
5. Implement the resilience fix in `cli_work_cleanup.py`, keeping the genuine-failure guards.
6. Update `build_epic_launch_argv` to `--yes-to-all`.
7. Tests (below).
8. Run `just install` then `just check` (ruff + mypy) and `just test`; fix all findings.

## Testing

Add/extend tests under `tests/test_bead/` (e.g. `test_cli_work_epic_launch_cleanup.py` and a new
`test_cli_work_cleanup_confirm.py`). Update the shared `make_args` helper to accept `yes_to_all` (default `False`).

- **Regression (the reported failure):** with the seeded orphaned family-container reservation for an expected phase
  name, `handle_bead_work(make_args(epic, yes_to_all=True))` releases the stale reservation, does **not** raise
  `ForcedReuseCleanupError`, and proceeds to launch. Assert the name is no longer reserved afterward.
- **Genuine failures still abort:** preserve the existing `test_work_force_reuse_cleanup_failure_aborts_before_mutation`
  cases (`wipe-raises`, `wipe-reports-errors`, `name-still-reserved`) — they must still exit non-zero with "forced reuse
  cleanup" in stderr and launch nothing. Set `yes_to_all=True` in those cases so the assertion targets the cleanup, not
  the prompt.
- **Override matrix** (monkeypatch `sys.stdin.isatty` and `builtins.input`):
  - No flags + TTY + destructive targets, answer `n` → aborted, nothing wiped or launched.
  - `-y` (yes=True, yes_to_all=False) + destructive targets → the destructive prompt **is** invoked (assert `input`
    called); answering `y` proceeds; the launch prompt is **not** shown.
  - `-Y` (yes_to_all=True) → neither prompt invoked; proceeds.
  - No flags + **non-interactive** stdin + destructive targets → clear refusal mentioning `--yes-to-all`, exit non-zero,
    nothing wiped or launched (no uncaught `EOFError`).
  - No destructive targets (fresh epic) → destructive prompt never shown; only the launch prompt applies (and is skipped
    by `-y`).
- **Preview content:** rendered preview lists the correct names grouped as KILL (live), REMOVE (terminal/dismissed), and
  RELEASE (stale reservation).
- **`--json` non-interactive:** `handle_bead_work(make_args(epic, json=True))` with destructive targets present neither
  prompts nor raises `EOFError`.
- **Argv:** `build_epic_launch_argv(path)` ends with `--yes-to-all`; update any existing test asserting `--yes` (grep
  for `build_epic_launch_argv` / `"--yes"` in `tests/`).

## Non-goals / scope guardrails

- Do not broaden "related agents" beyond the names the launch actually claims (the `%id:!` force-reuse directive names +
  `legacy_epic_cleanup_names`) and their family members / stale container reservations. Do **not** sweep arbitrary hood
  neighbors or unrelated members of the epic clan — that would risk killing agents the relaunch is not replacing.
- Do not change the actual agent-termination primitive (`wipe_agent_name_for_reuse` / `os.killpg` SIGTERM) beyond the
  minimal opt-in needed for stale-container release.
- No Rust `sase-core` change: the entire name-registry / wipe / bead-work cleanup subsystem is already Python with no
  `sase_core` counterpart, and this is CLI-presentation + Python-only cleanup logic. (Litmus check: no other frontend
  needs this specific bead-work confirmation flow to match a TUI behavior.) The implementer should confirm no core
  wire/API is touched.
- No memory-file or `default_config.yml` keymap changes are involved.
