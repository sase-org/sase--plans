---
tier: epic
title: Fix the popped-pane catalog reload and ship sase v0.18.0
goal:
  The plugins pane does not start a catalog load after it has been popped, Master Gate
  and Full CI are green on a master tip that contains that fix, release PR 299 merges,
  and `pip install sase==0.18.0` works from PyPI.
parent_bead: sase-1io.7.6
phases:
  - id: mount-race
    title: Stop a popped plugins pane from reloading its catalog
    depends_on: []
    size: small
    description:
      "mount-race: stop the unchanged update-completion path from starting a catalog
      load when the plugins pane has already been popped, and note the fix on sase-1ja."
  - id: ship
    title: Ship sase v0.18.0 once the blocking epics have landed
    depends_on:
      - mount-race
    size: medium
    description:
      "ship: after sase-1j6.10 and sase-1jc have closed, prove Master Gate and Full CI
      green on the tip, merge PR 299, publish v0.18.0, and verify the PyPI install."
proposed_by: bbugyi200.athena.sase-1io.7.6.land
create_time: 2026-10-10 09:29:16
status: wip
---

- **PROMPT:**
  [prompts/202610/finish_v0_18_0_ship.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/finish_v0_18_0_ship.md)

# Plan: Fix the popped-pane catalog reload and ship sase v0.18.0

This child epic finishes the unfinished landing of epic `sase-1io.7.6`. That epic's
three code commits are already on `origin/master` and were re-verified on 2026-10-10.
`sase==0.18.0` is not on PyPI. The land agent's note on `sase-1io.7.6` is the triage
record. Do not re-file those follow-ups. Do not close `sase-1io.7.6`, `sase-1io.7`, or
`sase-1io` from this child epic. The `parent_bead` link resumes that landing after this
child lands.

## What is already done

Re-read `sase bead read sase-1io.7.6` before acting. As of tip `35a97a0e1e`:

- `ready_gate=known_miss` is in the tree. `Justfile` `bead-perf-scale-gate` allows
  `ratio:ready`. `docs/perf_runbook.md` states the measured miss. Full CI `38029274494`
  `perf-floors` succeeded. Do not put `ratio:ready` back on the blocking list.
- The Reply-card fix `a36b5c90ca` is on master. `show_reply_card` is the one-press
  helper. Do not add press retries or longer timeouts.
- `reconcile_prompt_with_live_auto_state` is public. Do not rename it back to a private
  helper.
- `sase bead epic-symbols sase-1io.7.6` was empty. Do not add an `--epic-symbol` line
  for `sase-1io.7.6` or for this child unless Symvision forces one, and then key it to
  the phase bead that introduces the symbol.

## Escalation

If you get stuck, or you cannot close your assigned phase bead, hand off with the
`/sase_handoff` skill to `opus/opus@xhigh`. The ancestor release epics carry the user's
authorization for that handoff. If `sase pipe` rejects the spec, retry with
`--model 'claude/opus@xhigh'`. A non-zero exit means no handoff happened. Fence literal
percent signs in the successor prompt. Waiting on CI is not being stuck. Wait with
`/sase_monitor`.

## Guardrails

- Fix root causes. Never weaken an assertion, skip a test, add an xfail or retry, or
  raise a timeout to get green.
- Never hand-edit release-owned files: `CHANGELOG.md`, any `version` field,
  `.release-please-manifest.json`, or the `sase-core-rs` window in `pyproject.toml`.
- Do not cut a `sase-core-rs` release. Missing auto-restart bindings are `sase-1j6.10`'s
  work.
- Do not run `just install` or `just install-dev`. Use `just install-venv`.
- Do not run `just check-full`. Verify with `just fix` and then `sase tool run check`.
- While `sase-1j6.10` or `sase-1jc` is open, do not edit auto-restart, the feature-flag
  registry, or the agents live-query removal. Note new evidence on that epic and wait.
- The only PR you may merge is sase PR 299, and only in `ship`.
- Land code only through the host finalizer. `mount-race` cannot watch CI for its own
  commit. `ship` runs after that commit is on master.

## Phase mount-race: Stop a popped plugins pane from reloading its catalog

`sase-1ja` is a ready flake. Master Gate run `37992380866` failed
`tests/ace/tui/test_plugins_browser_pane_cached_open.py::test_mutation_completion_after_unmount_invalidates_memo`
twice (the run and its one allowed failed-job rerun): `assert 3 == 2` on catalog
`len(calls)`. Seven isolated local runs passed. No release commit touches this test. The
latest completed Master Gate `38053026930` did not fail this node. The code path is
unchanged.

The test opens the plugins pane (1 catalog call), `pop_screen`s it, waits until no
modal, then calls `_handle_code_update_completion` with `success=True` and
`payload=None`. It expects the inventory memo cleared and exactly one more catalog call
on the next open (`len(calls) == 2`).

In `src/sase/ace/tui/modals/plugins_browser_sase_update_procs.py`,
`_handle_code_update_completion` always invalidates the inventory, then on the
unchanged-success path calls `_start_load` when `self.is_mounted and not self._loading`.
Under shard load `is_mounted` can still be true after `pop_screen`, so the popped pane
records the extra catalog call. The reload is only useful while the pane is still the
presented screen. `invalidate_inventory` must stay unconditional so the next open
refetches.

Steps:

1. Read
   `sase memory read tui.md sase_beads.md lint_and_test.md -r "Need TUI, bead, and check rules before the mount-race fix"`.
2. Note on `sase-1ja` that this phase is fixing the root cause. Do not file a duplicate.
3. Change the unchanged-success reload so it runs only when this pane is still the
   presented screen. `pop_screen` removes it from the screen stack before unmount
   finishes, so a stack or current-screen check is the right signal. `is_mounted` alone
   is not. Keep the failure path and the real-change restart path as they are.
4. Keep the existing assertion `len(calls) == 2`. Add a regression beside that test that
   forces the old signal (the pane still reports mounted, or whatever condition the old
   code trusted) after `pop_screen` and asserts that no catalog load starts. No sleeps
   and no shard luck.
5. Run the plugins-browser cached-open file and `sase tool run check` after `just fix`.

**Done when** a popped pane cannot start that catalog load, `sase-1ja` has a note that
this phase fixed the root cause, and `sase tool run check` passes.

## Phase ship: Ship sase v0.18.0 once the blocking epics have landed

`mount-race` must be on `origin/master` before this phase dispatches CI. The release is
also blocked by work this child must not absorb:

- In-progress epic `sase-1j6.10` owns the auto-restart bindings missing from published
  `sase-core-rs` 0.37.2. PR 299 `release-core-floor-smoke` job `114186958378` (run
  `38043045593`) failed on those seven names. A note on `sase-1j6.10` records them.
- In-progress epic `sase-1jc` owns flag-registry rule 7 (closed bead `sase-s7` still
  defines `typed_launch_units`, Full CI `38029274494`) and the unused
  `ArtifactIndexProjection` Master Gate lint. A note on `sase-1jc` records both.

Steps:

1. Read
   `sase memory read sase_beads.md lint_and_test.md -r "Need bead close and check rules before shipping v0.18.0"`.
2. `git fetch`. Confirm the `mount-race` commit is on `origin/master`.
3. If `sase-1j6.10` or `sase-1jc` is still open, do not edit their areas and do not
   merge PR 299. Wait with `/sase_monitor` until both beads are closed. Use a bounded
   timeout of at least three hours and a `--next` that says to re-read both beads and
   continue this phase. Re-check after each wake. Master moves while you wait.
4. When both are closed, list the failed jobs of the newest completed Master Gate and
   Full CI on the current tip. Dispatch a fresh Full CI with
   `gh workflow run full.yml --repo sase-org/sase` unless a completed run already covers
   a tip that contains `mount-race` and both epics' final commits. Dispatch
   `gh workflow run publish.yml --repo sase-org/sase -f publish_existing=false` so PR
   299 regenerates from that tip.
5. Monitor Master Gate for the current tip, that Full CI run, and PR 299 until they
   settle. Full CI can queue for about an hour and run for about an hour. Use a monitor
   timeout of at least three hours.
   - A known flake (`sase-1ib`, `sase-1g8`, `sase-1j4`, `sase-1ht`, `sase-1j9`,
     `sase-18t`) may be retried once with `gh run rerun <run-id> --failed`. Record the
     evidence on its bead. Never rerun a deterministic failure. Do not rerun
     `sase-1ja`'s old run `37992380866`.
   - `ratio:ready` inside a green `perf-floors` job is the known miss. It is not a
     failure.
   - Any other deterministic red is yours once `sase-1j6.10` and `sase-1jc` are closed.
     Reproduce it at `origin/master`, attribute it with `git log`, and fix the root
     cause. If the fix is in this phase, land it and close this bead with a note that
     starts `RELEASE NOT SHIPPED:`, naming the failures and the runs to re-dispatch. The
     parent land finishes the release. Do not raise timeouts or edit goldens to hide a
     dropped Reply-card press. If `test_agents_decks_single_main_paged_png_snapshot`
     times out again, fix the product or the readiness wait. It did not fail Full CI
     `38029274494`.
   - If `release-core-floor-smoke` still fails because the published core lacks
     bindings, do not cut a core release. Note the missing names on the epic that added
     the imports and close this bead with `RELEASE NOT SHIPPED:` only after any code fix
     you landed. If you landed no fix, hand off instead of closing the phase as done.
6. Merge only when Master Gate is green on the current tip, Full CI was green within the
   last 6 hours on a tip that contains `mount-race`, and PR 299's checks are green,
   including `release-core-floor-smoke`. Let `ci_watch` merge PR 299. If it has not
   merged within about 30 minutes, merge with
   `gh pr merge 299 --repo sase-org/sase --merge`.
7. Publish with
   `gh workflow run publish.yml --repo sase-org/sase -f publish_existing=false`. Monitor
   `build`, `install-smoke`, `install-smoke-core-floor`, and `publish`. If the tag
   exists but publishing failed, fix the cause and dispatch with
   `-f publish_existing=true`.
8. Verify `https://pypi.org/pypi/sase/json` reports `0.18.0` with a wheel and an sdist.
   In a fresh `uv venv`, run `uv pip install sase==0.18.0`, then confirm `sase version`
   and `sase core health --json` both succeed.
9. Note the green Master Gate and Full CI run ids on `sase-1i5.9.1.2.1.7`. Do not close
   that bead. Send a `sase notify create` summary with the PyPI URL, the core version,
   and the failures this release train fixed.

**Done when** `sase==0.18.0` installs from PyPI and passes the health check, or this
bead closes with a `RELEASE NOT SHIPPED:` note after landing a further fix.

## Landing

Do not close parent epic `sase-1io.7.6` in this child. After this child lands, that
parent's land agent resumes. It should treat the follow-up triage already written on
`sase-1io.7.6` as settled, re-check that `0.18.0` is on PyPI, resolve any
`--epic-symbol` entries, close `sase-1io.7.6` without `--force`, run `just symvision`,
and mark `plan:202610/ship_v0_18_0_after_full_ci_fixes.md` done. Then it continues with
parent `sase-1io.7` only if this child actually shipped.
