---
tier: tale
title: Raise the builtin flake task type's triage bar from +1 to +3
goal:
  The shipped flake task type requires three +1 corroborations before bead_task_triage
  raises its gate, and the one already-raised flake notification below that bar is
  dismissed.
size: small
proposed_by: bbugyi200.athena.07d
create_time: 2026-08-18 20:40:42
status: wip
---

# Plan: Raise the builtin `flake` task type's triage bar from +1 to +3

## Goal

Change the shipped `flake` task-type triage bar (`triage.min_plus_ones`) from `1` to
`3`, so a flaky-test bead needs three independent `+1` corroborations before
`bead_task_triage` raises its `TaskTriage` gate. Then clear the one already-raised flake
notification that sits below the new bar.

## Background

`sase bead task-type show flake` prints `TRIAGE / min_plus_ones: 1` today. That value is
a per-type override of the global `bead.task_triage.min_plus_ones` config field, and it
lives in exactly one place:

- `src/sase/task_types/_builtin.py` — `_flake_spec()` ends with
  `"triage": {"min_plus_ones": 1}`. Every other builtin (`bug`, `ci`, `feature`,
  `memory`) declares `0` or no `triage` block at all, so `flake` is the only builtin
  that raises the bar.

Resolution order is `sase.bead.task_triage_policy.effective_min_plus_ones`: a typed bead
uses its own type's bar, and an untyped or unregistered-type bead falls back to
`bead.task_triage.min_plus_ones` (shipped default `1`, see
`src/sase/bead/config.py:14`). Changing the builtin therefore affects only `flake` beads
and leaves untyped legacy beads alone.

The digest in `sase/task_types.json` is computed by Rust from the spec
(`src/sase/task_types/_validation.py:56-60`), so no digest is hand-maintained and
nothing in the sibling `sase-core` repo needs to change. This is a Python-only edit.

### The notification half is mostly automatic

`bead_task_triage` already reconciles gates against the bar on every five-minute tick.
`src/sase/scripts/sase_chop_bead_task_triage.py:464-484` cancels a pending gate whose
bead has fallen below its bar with reason `task_bead_below_plus_one_threshold`, and
`cancel_gate` → `_settle_gate_notification(..., action="cancelled")`
(`src/sase/notification_gates/executor.py:455`, `:573-601`) dismisses the notification
row. `src/sase/default_config.yml:830-832` documents exactly this.

So once the new bar is live in the _installed_ sase that the chop daemon runs, the
sub-threshold flake notification is withdrawn without anyone touching it. Because the
daemon runs the global install rather than this checkout, the plan still dismisses the
row explicitly so the user sees the effect immediately; the chop's later cancellation is
idempotent on an already-dismissed row.

### Current live state (verified 2026-08-18)

Five `ready` flake beads exist, all in the `sase` project:

| bead      | `+1`s | under bar `1` | under bar `3` |
| --------- | ----- | ------------- | ------------- |
| `sase-pr` | 1     | gated         | withheld      |
| `sase-pu` | 0     | withheld      | withheld      |
| `sase-q8` | 0     | withheld      | withheld      |
| `sase-qk` | 0     | withheld      | withheld      |
| `sase-qm` | 0     | withheld      | withheld      |

Exactly one active flake-tagged notification exists — `sase-pr` at `[+1]`, notification
id `834dd386-7adf-419a-8fb8-48974cb68f17`, gate
`task_triage/bead-task-triage-sase-pr-fcc010b8a5ce-g2`. The four zero-`+1` beads are
already withheld under today's bar and have no notification to dismiss.

## Decisions and assumptions

**Read of "less than +1s".** The request says to dismiss active flake notifications
"with less than +1s". Taken as _below the new bar_ — fewer than three `+1` reports —
because that is the only reading that selects any work: every flake bead with zero `+1`s
is already suppressed and has no notification, so a literal "fewer than one" bar would
select nothing. Under the chosen reading exactly one row is dismissed (`sase-pr`,
`[+1]`). If the user meant something narrower, say so at the approval gate and the
dismissal step drops out; the threshold change is unaffected either way.

**Builtin default, not a project override.** The request names sase's flake task type's
_default_ threshold, and `+1` is precisely the shipped builtin value. So edit
`_builtin.py`. The alternative — a `bead.task_types` entry in `sase/sase.yml` with
`use: builtin@flake` and `triage: {min_plus_ones: 3}` — would change only this project
and leave the shipped default at `1`. Rejected as not what "default threshold" means,
but it is a one-line pivot if the user wants a project-scoped change instead.

**No feature flag.** Per `sase/memory/feature_flags.md` a flag covers a disabled beta,
an early landed path, or a deprecation whose old branch must stay reachable. A threshold
value change is none of those, and the bar is already user-overridable through
`bead.task_triage.min_plus_ones` and `bead.task_types`, which makes it a config field
rather than flagged behavior.

**No memory-file edits.** No file under `sase/memory/`, no `AGENTS.md`, and no provider
shim states flake's bar, so none needs to change and no memory-permission question
arises. If `sase memory init --check` unexpectedly reports a change to any of those
paths, stop and report it rather than writing it.

## Steps

1. **Prepare the workspace.** Run `just install` first; the workspace venv may be stale.

2. **Raise the bar.** In `src/sase/task_types/_builtin.py`, in `_flake_spec()`, change
   `"triage": {"min_plus_ones": 1}` to `"triage": {"min_plus_ones": 3}`. That is the
   only production-code edit.

3. **Update the test that pins the old value.** `tests/task_types/test_builtin.py:91`,
   in `test_flake_requires_corroboration_and_ci_does_not`, asserts
   `specs["flake"]["triage"]["min_plus_ones"] == 1`. Change it to `3`. Leave the
   neighboring `ci == 0` and `"triage" not in ...` assertions alone. Nothing else in the
   suite pins the builtin flake bar: `tests/test_bead/test_cli_task_type.py` and
   `tests/test_axe_chop_bead_*` build synthetic registries whose `min_plus_ones` is a
   fixture parameter, so they stay green untouched.

4. **Update the doc that names the value.** `docs/beads.md:199` reads "among the
   builtins only `flake` raises it (to `1`)". Change `1` to `3`. Same character width,
   so the prettier wrap is unchanged and `just fmt-md-check` stays quiet. Leave
   `docs/beads.md:377`, `docs/beads.md:649`, `docs/axe.md:262`, and
   `docs/notifications.md:318` alone — they say "`0` for most builtins" without naming
   flake's value, which stays true.

5. **Regenerate the committed catalog snapshot.** `sase/task_types.json` is the
   git-committed snapshot written by `sase memory init`
   (`src/sase/task_types/_snapshot.py:3`), and the flake entry there carries both
   `"min_plus_ones": 1` (line 280) and a spec digest that the change invalidates.
   - Dry-run first: `sase memory init --check --diff`.
   - Confirm the only planned write is `sase/task_types.json`, and that its diff touches
     only the `flake` entry's `triage.min_plus_ones` and `digest`. Plugin entries such
     as `github` must not appear or disappear — if they do, the local plugin set is
     wrong and the snapshot must not be written.
   - Apply with `sase memory init --no-commit`. Use `--no-commit`: committing in this
     repo goes through the normal commit flow, not this command's built-in sequence.

6. **Dismiss the sub-threshold flake notification.**
   - Re-derive the id rather than trusting the one recorded above; the chop may have
     regenerated the gate: `sase notify list -t flake -l 100 -j`.
   - For each returned row whose bead has fewer than three `+1`s (today: `sase-pr`
     alone), run `sase notify apply-state <notification-id> dismiss`.
   - **Do not answer the gate.** `sase gate show -k task_triage -i <request-id>` offers
     `launch`, `close`, and `snooze`; `close` would close `sase-pr` itself, which is not
     what was asked. Dismissing the notification hides the row and leaves the bead
     untouched, and the chop later cancels the orphaned pending gate on its own.

7. **Verify.**
   - `sase bead task-type show flake` prints `min_plus_ones: 3` under `TRIAGE`.
   - `sase notify list -t flake -l 100` returns no active rows.
   - `sase bead list -T flake -s ready` still lists all five beads: raising the bar
     withholds gates, it must not change any stored status.
   - `git status` shows edits confined to `src/sase/task_types/_builtin.py`,
     `tests/task_types/test_builtin.py`, `docs/beads.md`, and `sase/task_types.json`.

8. **Gate the change.** Run `just check`. Hand it to `/sase_monitor` with a `--next`
   action if it runs long. `just check-full` is not required here — the change is one
   literal plus a regenerated snapshot, and the scoped lane covers `tests/task_types/` —
   but run it through `/sase_monitor` (never inline) if the scoped run escalates or
   reports an unusual selection.

## Consequences worth stating

- **Three `+1`s is a high bar.** Per `sase/memory/sase_beads.md`, each reporter
  contributes at most one `+1` and the creator does not count, so a flake bead now needs
  four independent agents to have seen the same flake before it reaches the user. That
  is the point of the change — flakes are the type most often misfiled — but it does
  mean most flake beads will sit `ready` and ungated indefinitely.
- **Stale cleanup eventually notices.** `bead_stale_cleanup` sweeps ready task beads
  that sit below their effective bar for `bead.task_triage.stale_after_days` (7) once
  `stale_cleanup_min_beads` (10) of them have accrued, and offers to close them. All
  five flake beads are below the new bar, and `sase-pr` newly joins that set. Nothing
  fires today — every flake bead was created 2026-08-18 — but the backlog will trend
  toward a `BeadStaleCleanup` gate offering to close uncorroborated flakes. That is
  existing designed behavior, not a regression; it is called out so the first such gate
  is not a surprise.
- **The running daemon lags this checkout.** The chop runs the globally installed sase,
  so the new bar governs gate creation only after this change lands and that install is
  refreshed. Step 6 covers the interim for the one row that exists now.

## Non-goals

- Do not change `bead.task_triage.min_plus_ones` (`src/sase/bead/config.py:14`,
  `src/sase/default_config.yml:1190`). That is the untyped-bead fallback and applies to
  the 47 ready untyped task beads; the request is scoped to the flake type.
- Do not change any other builtin type's bar.
- Do not close, snooze, or `+1` any bead.
- Do not edit `sase/memory/*.md`, `AGENTS.md`, or any provider shim.
- Do not commit or push; the user has not asked for a commit.

## Acceptance criteria

- `sase bead task-type show flake` reports `min_plus_ones: 3`.
- `sase/task_types.json`'s flake entry carries `"min_plus_ones": 3` and a refreshed
  digest, regenerated by `sase memory init` rather than hand-edited.
- `docs/beads.md` no longer claims flake's bar is `1`.
- No active flake-tagged notification remains for a flake bead with fewer than three
  `+1`s.
- All five flake beads remain `ready` and unclosed.
- `just check` passes.
