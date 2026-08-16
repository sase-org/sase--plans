---
tier: tale
title:
  Fix the monitor start-ack pid contract and close sase-m9.3.1 so epic sase-m9 can land
goal:
  The one remaining epic-attributable failure from the supervisor-owned proc closeout —
  the start-ack supervisor-pid race in tests/monitor/test_monitor_start_ack.py — is
  fixed at the contract level, the two still-undispositioned DISCOVERED ISSUE notes on
  sase-m9.3.1 are dispositioned under bead policy, an exhaustive lane confirms no
  epic-attributable failures remain, and sase-m9.3.1 plus parent phase sase-m9.3 close
  so the waiting sase-m9.land agent can land epic sase-m9.
size: medium
proposed_by: bbugyi200.athena.03y
bead: sase-m9.3.1
create_time: 2026-08-16 13:38:29
status: wip
---

- **BEAD:**
  [sase-m9.3.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m9/sase-m9.3.1.md)

# Plan: Fix the monitor start-ack pid contract and close `sase-m9.3.1`

## Objective

Epic `sase-m9` (Supervisor-owned procs and the sase shell model) is stalled at its last
phase. Phases `sase-m9.1` and `sase-m9.2` are closed. Phase `sase-m9.3`'s only child is
child epic `sase-m9.3.1`, whose five implementation phases (`.1` through `.5`) are all
closed. Agent `sase-m9.land` is alive and parked on `%w(bead=sase-m9.3)`, so nothing in
this epic can progress until `sase-m9.3.1` closes.

The previous closeout attempt (`plan:202608/proc_ownership_closeout.md`, agent `03b--1`)
landed all three of its remediations as `e38874024` but then hit its own hard-stop rule:
monitored `just check-full xbfsm7s2nb5e` failed 11 / 30981, so it deliberately closed
nothing. Ten of those eleven failures were triaged to standing flake beads owned by
other work. Exactly one was attributed to this epic, and it is a real defect in the
monitor start contract rather than a flake. Fix that one, disposition the leftover
notes, re-verify, and close.

Do not redo `e38874024`'s three remediations. They are on `master` and independently
re-verified (see below).

## Verified starting state

Audited by agent `03y` on `master` `ddef1f0d4` after a successful `just install`.
Re-check each item against current `origin/master` before editing — other epics are
landing concurrently.

- All five child phases `sase-m9.3.1.1` through `.5` are closed, as is parent phase
  `sase-m9.2`. `sase-m9.3` and `sase-m9.3.1` are `in_progress`; `sase-m9` is
  `in_progress`.
- `e38874024` ("fix(proc): isolate SASE*PROC*\* tests, skip malformed monitors, overlay
  session workers") landed all three closeout remediations. Each was re-verified
  independently:
  - **Env hermeticity.** `tests/_conftest_environment.py` now scrubs every inherited
    `SASE_PROC_*` variable by prefix and keeps a `leaked_proc_keys` teardown guard. From
    a live agent shell with all six variables set and **no** outer `env -u`, the nine
    file families named by task `sase-ml` passed: `137 passed, 3 skipped in 19.68s`.
  - **Malformed monitor rows.** `sase monitor list --all` now exits 0 against the host
    artifact `20260815145837` that used to poison enumeration.
  - **Session-worker overlay.** `src/sase/ace/tui/actions/proc_actions.py` has
    `_session_overlay_rows()` and composes it through
    `compose_proc_projection(durable, ...)`.
- Task bead `sase-ml` is **already closed** by `03y` with that verification recorded in
  its close note. Do not reopen it, re-verify it, or treat its disposition as
  outstanding work; the earlier closeout plan's section 1 is complete.
- The stale `Justfile` `--epic-symbol 'sase-m9.3.1.2(compare_inventory_to_source)'`
  exemption that was reported five separate times is gone.

## The defect to fix

`tests/monitor/test_monitor_start_ack.py::test_startup_sigterm_settles_stopped_without_running_command`
failed on `gw4` of `just check-full xbfsm7s2nb5e` with
`supervisor pid was never recorded`, and passed in 1.75s on an isolated rerun. It is not
a flake to file away — it is a genuine mismatch between what the test documents as the
contract and what `start_monitor` actually does.

`_wait_for_recorded_supervisor_pid` documents the contract as: `start_monitor`
"overwrites this on the new member's `agent_meta.json` with the real supervisor pid
immediately after spawning it — well before it blocks on the startup acknowledgement".
That is false against the current code. In `src/sase/monitor/start.py`, the real pid is
written by `update_meta_field(artifacts_dir, "pid", supervisor_pid)` inside `after_ack`,
which the submit path invokes only _after_ `wait_for_start_acknowledgement`.

The test sets `SASE_PROC_BOOTSTRAP_IMPORT_DELAY_SECONDS=1.0` so the ack cannot land
during the first second, then polls only 10s (`timeout=10.0`) for a pid that differs
from `os.getpid()` before delivering `SIGTERM`. Under the loaded full lane that 10s
window expires while `start_monitor` is still blocked on ack, the poll raises, and the
SIGTERM-during-startup race the test claims to exercise never runs at all. The member
briefly carries the _caller's_ pid as a placeholder from `create_monitor_member`, which
is why a bare "some pid is present" check cannot paper over it.

**Prefer the contract fix over the timeout fix.** Record the spawned supervisor pid on
the member's `agent_meta.json` before blocking on the acknowledgement, so the documented
behavior becomes true and an external caller really can signal into the startup window.
That is the behavior the test is trying to prove and the behavior an operator needs when
a monitor wedges during bootstrap. Keep `after_ack`'s authoritative write: the
acknowledgement's pid remains the source of truth, so the pre-ack write is a placeholder
upgrade, not a replacement. Make sure the pre-ack write can never leave the caller's own
pid recorded as if it were the supervisor's, and that a spawn failure before ack does
not strand a bogus pid on a member that never started.

If — and only if — the pre-ack write proves genuinely unsafe (for example, the spawned
pid is not knowable before the acknowledgement in the current submit path), fall back to
raising the test's poll timeout to at least the ack timeout, correct the misleading
docstring so it describes what the code does, and record in the phase note why the
contract fix was rejected. Do not do both.

Add a regression that fails against the old ordering: assert the real supervisor pid is
observable on disk _while_ `start_monitor` is still blocked on the acknowledgement, with
the bootstrap import delay long enough that the assertion is meaningful. Keep it robust
under a loaded parallel lane — derive any poll bound from the ack timeout rather than
hard-coding a smaller constant, since a hard-coded 10s against a 20s ack default is the
exact bug being fixed.

## Remaining note disposition

Two `DISCOVERED ISSUE` notes on `sase-m9.3.1` are still undispositioned. Both explicitly
declined to create a standalone task bead on the grounds that this epic owns them, so
they cannot simply be left behind when it closes. Neither is in scope to _implement_
here.

1. **Dead in-process launch body and the vestigial `proc_callable` parameter** (note by
   `03a`, 2026-08-16T13:30Z). Confirmed still true on `ddef1f0d4`:
   `src/sase/ace/tui/actions/agent_workflow/_launch_procs.py` accepts `proc_callable`
   only to `del` it, `_cleanup_procs.py` does the same, and
   `run_single_agent_launch_body`, `_launch_repeat_agents`, `_launch_bulk_agents`,
   `_launch_multi_prompt_agents` and `_launch_multi_model_agents` have no production
   callers while tests still drive them. That divergence already hid one real bug (the
   stale `_prompt_context` leak fixed by `2aa8ba26f`). File it as a task bead through
   `/sase_new_task`, sized on the retirement of both the dead bodies and the parameter,
   including migrating the test doubles onto the durable submit path production uses.
2. **`sase monitor start` fails from inside a live agent with "no agent artifacts found
   for agent '<name>'"** (note by `03a`, 2026-08-16T13:44Z). The malformed-monitor half
   of that report is fixed by `e38874024`; the artifact-lookup half was never addressed.
   Re-run `sase monitor start` from the implementing agent's own context first — this
   plan requires a monitored `check-full` anyway, so the reproduction is free. If it now
   succeeds, record that on `sase-m9.3.1` and disposition the note as resolved. If it
   still fails, file it through `/sase_new_task`, which will check it against ready task
   `sase-ll` (in-agent `sase monitor start` without an explicit `--lane` resolving the
   wrong epic family parent) for semantic duplication before creating anything.

Before closing, re-read every note on `sase-m9.3.1` and its five child phases and
confirm each `DISCOVERED ISSUE` / `PROPOSED FOLLOW-UP` is fixed, filed, or explicitly
recorded as intentionally out of scope.

## Verification

Run `just install` first — workspaces are ephemeral and dependencies drift.

1. Focused first: the whole `tests/monitor/` tree plus the proc/ops suites the closeout
   touched — `tests/test_proc_submission_static_invariants.py`, the producer inventory
   test, `tests/ace/tui/test_proc_actions_session_workers.py`, and the proc observer and
   Procs-pane tests. The start-ack file must pass both in isolation **and** under
   contention; run it with `-n` parallelism and repeat it, because a single serial pass
   is exactly the evidence that misled the previous attempt.
2. `just check`.
3. `just check-full` through `/sase_monitor` with an explicit `--next` action, never
   inline. Do not sanitize the environment with an outer `env -u` — `e38874024` made the
   suite hermetic and a passing lane under inherited `SASE_PROC_*` is now part of what
   is being proven.

**Closure bar, deliberately different from the previous attempt.** The old plan treated
any red full lane as an absolute hard stop, which is why a finished epic has sat blocked
behind four unrelated flake beads. The bar here is: **zero epic-attributable failures.**
Classify every failure against a named owner and cite the evidence. Known standing
classes at time of writing, all with their own ready beads: `sase-mv` / `sase-j7`
(config-cache leak past `patch('sase.config.core.CONFIG_DIR')`), `sase-n5` (Models panel
`test_panel_mixed_bucket_sections_title_and_restore`), `sase-n6` (fakey runner-slot
timeout `test_child_is_exempt_while_repeat_roots_stay_capped`), `sase-nc` and `sase-nd`
(run-supervisor timeout flakes). A failure you cannot attribute to an existing bead is
**not** automatically unrelated — reproduce it in isolation, and if it is new, either
fix it or file it through `/sase_new_task` before closing. Corroborate rather than
duplicate: add `+1` evidence to an existing bead when the class already has one.

## Closure

Close these beads **only if** the associated work above is actually complete — the
start-ack contract fixed with a regression that fails against the old ordering, both
leftover notes dispositioned, and an exhaustive lane whose every failure is attributed
to a bead other than this epic. If any of that is unfinished, close nothing, record
exactly what blocked on `sase-m9.3.1`, and hand off.

- **`sase-m9.3.1`** — close as `done` with a note naming the start-ack fix and its
  commit, the disposition of both leftover notes (with any task-bead IDs created), the
  focused suite counts, and the monitored `check-full` id plus its per-failure
  attribution table.
- **`sase-m9.3`** — its only child is `sase-m9.3.1`, so closing the child should settle
  it. Confirm the status afterward and close it explicitly with a note linking the
  child's verification if it did not settle on its own.

Do **not** use `--force`, and do **not** close `sase-m9` — its land agent `sase-m9.land`
is alive and waiting on `%w(bead=sase-m9.3)` and owns that close. Confirm both statuses
after closing so that agent can continue.

Beads already closed by `03y` before this plan, which this plan must not revisit:
`sase-ml` (verified fixed by `e38874024`), and — from the unrelated sibling epic
`sase-m6` — `sase-m6.7.1` and `sase-m6.7`.
