---
tier: tale
title: Fix closed-bead submission time and finish sase-1li landing
goal:
  Launch claims use the real submission instant, preserve queued-close refusal, and epic
  sase-1li is verified and closed with its original plan marked done.
size: medium
proposed_by: bbugyi200.athena.sase-1li.land
bead: sase-1li
create_time: 2026-10-16 18:23:13
status: wip
---

- **PARENT:**
  [202610/closed_bead_launch_attach.md](https://github.com/sase-org/sase--plans/blob/main/202610/closed_bead_launch_attach.md)
- **BEAD:**
  [sase-1li](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1li/README.md)

# Finish the closed-bead launch landing

This is the remaining work from the landing audit of epic `sase-1li`. Implement it
directly and finish the landing in this turn. This tale has no land agent or automatic
return to the interrupted lander. Do not wait for this turn's commit, SHA, push, or CI:
the host commits only after the turn ends. Closing the epic with the final code still
uncommitted is the intended sequence.

## Verified context and scope

Read `sase bead read sase-1li -r "Need landing audit and follow-up dispositions"` and
the original plan through
`sase artifact read plan:202610/closed_bead_launch_attach.md "Need original accepted requirements"`.
The audit reviewed every child and note, the actual source and commits, and later
changes through `f3087c86ac`, which matched freshly fetched `origin/master`. The
retained audit and reproduction are `file:explicit:7216f9267aba3db812ee92eb`; read them
through `sase artifact read` if more evidence is needed.

The landed commits are core `594faf31d80af79a148cb67968318b2bfb1a5770`, exactly the
current `sase-core-revision.txt` pin; runner `881f444778`; and guidance `4b000ca2c6`.
The Rust comparison and atomic note mutation are implemented, including preservation of
close metadata/ancestors, legacy missing-context refusal, equality attaching,
launching-agent note authorship, and replay parity. The runner's metadata, notice,
distinct commit subject and normal/force-reuse forwarding exist. Guidance exists in the
prompts, docs and authorized memory template. Keep the accepted decisions:
`healer_done_only = no` (all restart paths may attach regardless of resolution) and
`beads_memory = yes` (already completed). This repair needs no memory edits.

The lander ran 69 focused tests successfully in 11.62s, but found a real R2 defect and
omitted planned integration coverage. Do not implement the separately triaged feature
proposals below as part of this repair.

## 1. Carry the actual submission instant

`src/sase/axe/run_agent_runner_launch.py::_promote_bead_claim` currently derives
`submitted_at` from `launch_timestamp_to_utc_iso(state.timestamp)`. `state.timestamp` is
an allocated unique launch ID. `reserve_launch_timestamp_batch` in
`src/sase/core/agent_launch_facade.py` and Rust's `allocate_launch_timestamp_batch`
advance these IDs past earlier reservations and by one second per fanout slot. They are
not reliable wall-clock submission instants.

The lander reproduced this against an isolated temporary store and the real binding:

| Actual submission | Recorded close   | Allocated launch ID | Current result      | Correct result      |
| ----------------- | ---------------- | ------------------- | ------------------- | ------------------- |
| 12:00:00Z         | 12:00:01Z        | 260101_120002       | attaches            | closed_while_queued |
| 12:00:00.900000Z  | 12:00:00.100000Z | 260101_120000       | closed_while_queued | attaches            |

All timestamps are on 2026-01-01. The second case explicitly supplies a supported
fractional RFC3339 close through the facade; normal current close clocks store whole
seconds. Preserve recorded-time equality and historical missing-close-time behavior; do
not invent precision for legacy records or broaden this into a close-clock rewrite.

Implement a dedicated UTC RFC3339 submission stamp, preserving fractional precision,
captured at launch ingress before admission, dependency or capacity waits. Carry it
through the existing launch request/environment and durable runner metadata. A suitable
transport is an internal `SASE_LAUNCH_SUBMITTED_AT` key plus `launch_submitted_at`
metadata; use one coherent transport and avoid a new user-facing option.

Inspect the existing funnels before wiring it:

- `launch_cwd_agents.py`, `launch_cwd_bead_work.py`, fanout planning/execution, and
  `launch_admission_engine.py` carry per-launch/per-segment context.
- `launch_provenance.py` demonstrates environment transport, but its ambient
  human-authorship inheritance must not be copied blindly for submission time.
- `launch_spawn.py::spawn_agent_subprocess` is the common process boundary, including
  direct retry callers. Do not stamp only here if admission has already waited.
- `axe/run_agent_runner_setup_meta.py`, bootstrap/state and
  `run_agent_runner_refresh.py` preserve run identity through refresh re-exec.

Rules:

1. One submitted launch keeps its stamp through fanout transport, queue waits, workspace
   preparation retries and refresh re-execs. Never replace it with the promotion time or
   derive it from the unique launch/artifact ID.
2. A new manual restart, `,x`, retry handoff, update-skew restart, or new typed launch
   gets its own submission stamp. A child must not inherit its parent's old timestamp
   accidentally from the ambient process environment. Internal transport for the same
   already-submitted request preserves the value. A multi-slot submission must stamp its
   context before dispatching the first slot, so a later slot's delay cannot make an
   intervening close appear pre-submission.
3. Pass the trusted persisted stamp, plus the unchanged artifacts run ID, through both
   `_promote_bead_claim` branches into the existing Rust claim API. The host notice and
   disposition metadata must still reach the provider.
4. A legacy launch with no trustworthy stamp retains the existing no-context closed
   error. Do not silently use the synthetic ID as a fallback. Malformed supplied context
   must fail clearly rather than becoming an attach permission; ordinary open claims
   must retain their existing behavior.
5. Keep the domain comparison/mutation in Rust. This is host clock capture and transport
   glue, not a second Python implementation of claim policy. No core API change should
   be necessary. If one is genuinely needed, open `sase-core` with `/sase_repo`, follow
   its instructions and verify the binding/pin contract.

Remove or privatize `launch_timestamp_to_utc_iso` if this leaves it without a real
consumer; do not keep it alive solely with tests or add an epic whitelist for it.

## 2. Complete the behavioral tests

Use deterministic clocks, real temporary bead stores and the real claim facade. Keep
sleeps, live launches and mutable production state out of regression tests.

- Add a future-ID regression using batch allocation at 12:00:00 and the third ID
  12:00:02; a close at 12:00:01 must refuse before the model callback, with no note or
  close-metadata mutation. Prove the persisted real stamp reaches the Rust boundary.
- Cover fractional timestamps from the second reproduction, equal timestamps and non-UTC
  environments. UTC provenance must not depend on reparsing a local ID.
- Prove preservation through waiting/refresh and freshness for a newly submitted
  restart/retry. Exercise both ordinary and force-reuse paths and guard against ambient
  parent-stamp leakage. Cover the preplanned bead-work bypass as well as the generic
  launch funnel.
- Finish the original plan's missing
  `tests/test_bead/test_epic_jit_claim_integration.py` matrix. Its
  `test_closed_rendered_bead_fails_before_model_callback` still tests only the legacy
  no-submission error. For both phase and epic targets, cover a visible close attaching
  and executing the callback, a close after submission refusing before it, and missing
  submission retaining the legacy refusal. Verify the returned status, unchanged
  resolution/close fields, exactly one correctly authored note on attach, and no writes
  on refusal.
- Add the omitted real helper case: a closed bead assigned to the prior owner attaches
  under force reuse without changing that assignee. Retain the active-other-owner
  rejection. The current promotion test mocks the claim and does not prove this.
- Keep the host notice idempotent and ordinary open launches free of closed-attach
  disposition keys.

## 3. Integrate and verify

Review commits since the audit HEAD for changes at these launch seams and update the
repair accordingly. The original audit found later docs splits, generated directives,
instruction projections, release-watcher changes, visual recapture and artifact-link
caching compatible with the epic. The post-core `181072b5` change only affects
content-layout. Do not reimplement or undo those changes.

Run the relevant launch/provenance/queue/refresh tests and the original focused set:

```sh
.venv/bin/python -m pytest -q tests/test_bead/test_epic_jit_claim_integration.py tests/test_run_agent_runner_bead.py tests/test_run_agent_runner_launch.py tests/test_core_facade/test_bead_mutation.py tests/core/test_agent_launch_timestamps.py tests/test_bead_macro_tags.py
```

Read `lint_and_test.md` with `/sase_memory_read`; run formatting and
`sase tool run check` (the required `just check`). Do not run `just check-full`. Fix new
or unexplained failures caused by this repair. If a long check needs a monitor, use
`/sase_monitor` with explicit follow-up instructions to finish this tale's closeout;
never leave the check running when ending the turn. Do not use prepared completion that
skips the required close note, post-close symvision and plan status update.

The independent home-shim drift is already recorded on `sase-1lj` with the causal
97825eee1d diff; it is not a reason to alter unapproved memory. Existing phase test
failures and their owners are in the triage record below. Apply the repository's
KNOWN/FLAKY evidence policy without claiming a failed check passed or waiving a new
failure. Do not expand this repair to unrelated test infrastructure.

## 4. Close sase-1li in this coding turn

Re-read the epic, its three children (`sase-1li.1`, `.2`, `.3`) and any new descendants
or notes; confirm completion and current readiness. Its original plan is
`plan:202610/closed_bead_launch_attach.md`, under the plans repository at
`202610/closed_bead_launch_attach.md`. Use `/sase_repo` to open `plans` before editing
and use only the returned checkout; read artifacts with audited `sase artifact read`.

Carry all eight already-completed triage outcomes into the close note:

- `.1 #1`: corroborated `sase-1gz` (gateway listener flake).
- `.2 #1`: current home shim drift recorded as `DISCOVERED ISSUE` on `sase-1lj`;
  declined reopening the superseded `sase-i7` or creating a duplicate task.
- `.2 #2`: corroborated `sase-xo` (blocked-render flake).
- `.3 #1`: corroborated `sase-193` (premature provider-turn bead close); no new task.
- `.3 #2`: created READY medium feature `sase-1lu` (restart attachment hint).
- `.3 #3`: created READY large feature `sase-1lv` (stood-down outcome).
- `.3 #4`: corroborated `sase-1ic` (baseline import budget).
- `.3 #5`: corroborated exact-node `sase-1k3` (SIGTERM settlement SQLITE_BUSY); declined
  another task or redundant corroboration on related `sase-18t`.

Then perform this sequence, without waiting for this turn's future commit:

1. Run `sase bead epic-symbols sase-1li`. Resolve every listed symbol by wiring,
   privatizing, a justified non-test consumer pragma, or deletion. Re-key only when an
   identified still-open later bead needs the exemption. The audit found none.
2. Close normally with `sase bead close sase-1li --note "<verification>"`. The note must
   identify the repaired timestamp contract, integration/tests and actual check result,
   decisions honored, complete descendant review, and all triage outcomes. If rejected
   for symbols, clean them and retry. If a phase is incomplete, finish or reopen it.
   Never force merely to succeed, and never force a successful nested landing.
   Cancellation/supersession requires a deliberate outcome and reason.
3. Run `just symvision` and address any whitelist leftovers.
4. Set `status: done` in the frontmatter of the original plan
   `202610/closed_bead_launch_attach.md` in the opened plans checkout. Include that
   repository obligation in finalization. Mark this tale done as normal.
5. Read `sase bead read sase-1li -r "Need the parent link"`. The audit confirms no
   `parent_bead`, so finish normally. If the graph changed, follow the original land
   prompt's parent policy: normally close only a verified parent phase, or re-audit
   descendants, previous landing notes, linked plan and post-child drift for each
   directly parented plan before its normal close, symvision and done update. Stop with
   a concrete note at the first incomplete parent; never force advancement.

Submit `/sase_final` only after all commands finish. Report the actual verification and
closeout result, with no promise to wait for a commit from this turn.
