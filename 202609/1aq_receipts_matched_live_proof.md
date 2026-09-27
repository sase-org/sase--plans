---
tier: epic
title: Deploy the settled-receipt contract to both hosts and prove it live
goal: Apollo and Athena run the same sase and sase-core build, one that contains the
  exact_ops_receipts contract. From Athena, a live exact stop and retry against Apollo
  settle certainly, a fresh dispatch can be stopped before the owner snapshot rebuilds,
  and a killed row keeps its retry context.
parent_bead: sase-1aq.10.7.5
phases:
- id: receipt_cleanup
  title: Land the settled-receipt type and kill-wrapper fixes
  depends_on: []
  size: small
  description: 'receipt_cleanup: fix the mypy arg-type error in dispatch mutations
    and the over-broad TypeError fallback in the mobile kill wrapper that exact_ops_receipts
    introduced, update the stale test fakes, and verify.'
- id: matched_deploy
  title: Bring Apollo and Athena to the same build carrying the receipt contract
  depends_on:
  - receipt_cleanup
  size: medium
  description: 'matched_deploy: reconcile the dirty primary sase checkouts on both
    hosts, run the supported sase update so both hosts match origin/master for sase
    and sase-core, restart the gateway stack, repair the Athena agent index, and record
    identities.'
- id: receipt_live_proof
  title: Prove settled stop, fresh-row stop, and retry-after-kill live from Athena
  depends_on:
  - matched_deploy
  size: medium
  description: 'receipt_live_proof: drive Athena over SSH against Apollo on the matched
    build to prove the exact_ops_receipts contract live, and attach audited requirement-to-evidence
    notes to the original beads.'
proposed_by: bbugyi200.apollo.sase-1aq.10.7.5.land
create_time: 2026-09-27 02:20:27
status: wip
bead_id: sase-1aq.10.7.5.7
---

- **PROMPT:** [prompts/202609/1aq_receipts_matched_live_proof.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/1aq_receipts_matched_live_proof.md)
- **PARENT:** [202609/1aq_close_original_gates.md](https://github.com/sase-org/sase--plans/blob/main/202609/1aq_close_original_gates.md)
- **BEAD:** [sase-1aq.10.7.5.7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1aq/sase-1aq.10.7.5.7.md)

# Deploy the settled-receipt contract to both hosts and prove it live

## Why this plan exists

Epic `sase-1aq.10.7.5` (plan `plan:202609/1aq_close_original_gates.md`) closed all six
phases. Its land audit found that the epic's own feature was never exercised live:

- Phase `exact_ops_receipts` (`sase-1aq.10.7.5.2`) landed two commits. In sase, commit
  `700b37b384` added the same-key acceptance-window poll and fleet stop without
  dismissal (`retain_for_retry`). In sase-core, commit `b57cd21` added settled mutate
  receipts, the index-backed catalog overlay for fresh launches, and retention of killed
  rows with retry and fork advertised.
- Phase `live_matrix` (`sase-1aq.10.7.5.3`) was required to prove "exact stop and
  same-key retry, now with settled receipts" and "retry-after-kill according to the
  exact_ops_receipts contract" on matched builds. It did not install matched builds. Its
  note #2 records sase skew: Apollo `0.17.1+1549.g64fae010f.dirty` and Athena
  `0.17.1+1540.g63d2bdcea.dirty`, with sase-core `0.34.73+46.ge44af7d40` on both.
- Neither host's sase contains `700b37b384`, or even `fb0b91edce`. Neither host's
  sase-core contains `b57cd21`. The Apollo gateway process started before `b57cd21`
  existed.
- The live-matrix evidence it recorded on `sase-xe.16.11.7.14.6.7.6` (note #3) reused
  pre-epic stop and retry runs from `sase-1aq.10.7.1`. It also credited "row still DONE
  2.5h post-kill" to the .5.2 contract, which was not deployed. Its live
  retry-after-kill attempt was refused for a missing `lifecycle.retry` capability, which
  is old-contract behavior.
- `sase update -n` on both hosts shows the blocker. Each host's primary sase checkout
  (`~/projects/github/sase-org/sase`) is dirty, so the update skips its fast-forward and
  would move only sase-core. Apollo has modified
  `src/sase/integrations/mobile_gateway.py`, `src/sase/ops/commands/machine.py` and
  `tests/test_machine_agent_command.py`. Athena has modified
  `src/sase/ops/commands/machine.py` and `tests/test_machine_agent_command.py`. These
  look like hand-applied copies of commits already landed on master: `29f8240df5` (bare
  sase bridge command) and `fb0b91edce` (exact stop/retry lookup).
- The Athena agent artifact index also reports "repair recommended: no such column:
  gate_shell_id" (`sase agent index status`), a symptom of the same build skew.

The land audit also found two small defects that `700b37b384` introduced. The land agent
could not commit them before this handoff, so phase `receipt_cleanup` owns them.

This plan finishes only that remaining work. It does not close `sase-1aq.10.7.5`,
`sase-1aq.10.7`, `sase-1aq.10` or `sase-1aq`: their land agents resume through
`parent_bead` after this plan lands. It does not reopen the original beads that
`sase-1aq.10.7.5` closed. Evidence goes onto them as notes.

## Operating rules

- **Hosts.** Apollo owns the agents and runs these workers. Athena is the dispatch
  controller and viewer. Reach Athena through `ssh athena` (Tailscale SSH), and run
  every viewer-side step there. Read the `tailnet.md` and `dispatch.md` reference memory
  with `/sase_memory_read`, and read `docs/remote_dispatch.md`, before the live phases.
- **Code placement.** Shared fleet and dispatch behavior belongs in sase-core; open it
  with `sase repo open sase-core`. A binding change needs a pin move in
  `sase-core-revision.txt` (see `docs/rust_backend.md`). Verify changed trees with
  `sase tool run check`. Do not run `just check-full`.
- **Stale workspace binding.** If focused tests fail with
  `agent scan wire schema mismatch: got 9, expected one of [10, 11]`, the workspace's
  `sase_core_rs` is stale. Run `just install` first; it takes about 20 minutes, so give
  it a long foreground timeout. Do not record this as a product failure.
- **Known clean-base reds at master `19abe261d4`.** Do not count these as this plan's
  work, and do not file duplicates:
  - 4 mypy errors. `_tree.py:622-629` is owned by `sase-19i.7.3.3.3.3`.
    `_agent_display_hint_sections.py:74` (`LEGACY_NAMED_PROC_SECTION_ID`) is owned by
    `sase-1ab`.
  - 13 symvision unused-public symbols, owned by task `sase-1ay`.
  - About 48 test-scoped failures from the shell-to-turn and proc renames, owned by
    `sase-1ab` and `sase-18s`. Examples: the marker audits, `test_parser_proc`,
    `test_launch_proc_runtime`, the keybinding footer "shell" digits, and
    `proc_wire_schema_version`.
  - `test_dispatch_federation` `ipc_client` `AF_UNIX path too long` from long workspace
    tmp paths.
- **Evidence.** Keep redacted payloads, receipts and pane captures as audited artifacts
  (`sase artifact create`) with UTC timestamps. Put requirement-to-evidence rows on the
  beads named below. Use sleep-free prompts for exact-output assertions.
- **Safety.** Preserve enrollment (`sase machine list` on Athena must still show
  `apollo` pinned) and unrelated work. Never replay an uncertain operation under a new
  key. Never close or reopen a bead with `--force`.

## Phase receipt_cleanup

Make these exact changes in this sase checkout:

1. `src/sase/dispatch/mutations.py`, `_submit_remote_mutation`. Mypy reports
   `Argument 1 to "float" has incompatible type "object"` at the line
   `acceptance_window = float(request["acceptance_window_seconds"])`. Compute
   `acceptance_window = max(timeout_seconds or config.request_timeout_seconds, 30.0)`
   before building `request`, and use it for `"acceptance_window_seconds"`. Delete the
   `float(request[...])` line and the redundant `mutate_timeout` alias, and pass
   `timeout_seconds=acceptance_window` to `facade.mutate_sync`.
2. `src/sase/integrations/_mobile_agent_deps.py`, `kill_named_agent`. Remove the
   `try: ... except TypeError: return func(name, exact_name=exact_name)` fallback.
   Always call the override or the real function with
   `(name, exact_name=exact_name, retain_for_retry=retain_for_retry)`. The fallback
   catches a `TypeError` raised inside a real kill and re-runs the kill without
   `retain_for_retry`, which dismisses the row the fleet stop meant to keep.
3. Update the test fakes that still have the pre-contract signature to accept
   `retain_for_retry: bool = False`. These are the two
   `lambda name, *, exact_name: _KillResult(...)` fakes in
   `tests/test_mobile_agent_kill_retry.py`, and `fake_kill` in
   `tests/test_mobile_agent_bridge_smoke.py`.
   `test_kill_mobile_agent_bridge_returns_success_for_stale_cleanup` patches
   `lifecycle.kill_named_agent` directly and currently fails with
   `<lambda>() got an unexpected keyword argument 'retain_for_retry'`.

Verify with these focused tests, which must all pass:
`pytest tests/test_dispatch_mutations.py tests/test_mobile_agent*.py tests/test_kill_named_agent_dismiss.py`.
Also run `mypy src/sase/dispatch/mutations.py` (clean) and `sase tool run check`, where
only the known clean-base reds above may remain.

## Phase matched_deploy

On each host, Apollo first and then Athena over `ssh athena`:

1. Record the current state: `sase version`, `sase update -n`, the running gateway,
   federation-worker, AXE and ACE processes (with start times), and
   `git -C ~/projects/github/sase-org/sase status --short` plus the matching `git diff`.
2. Reconcile the dirty primary sase checkout. Save the full `git diff` to a timestamped
   backup patch outside the checkout, and register it as an audited artifact. Check
   every hunk against the landed commits (`29f8240df5`, `fb0b91edce`, and anything else
   on `origin/master`). If every hunk is already on master, discard it with
   `git stash push -m "pre-matched-deploy <UTC>"` so it stays recoverable. If any hunk
   is not on master, stop and record it on this phase bead instead of discarding it. Do
   the same check for the primary sase-core checkout.
3. Check `sase agent list` and the WAITING land agents. The update changes the editable
   code that running agents import, so do not update in the middle of another agent's
   commit or landing on that host. Waiting and parked agents are not a blocker.
4. Run the supported update path: `sase update`, dev mode, as already configured.
   Confirm that both hosts now report the same sase commit, which contains `700b37b384`
   and this plan's `receipt_cleanup` commit. Also confirm the same sase-core-rs build,
   which contains `b57cd21` (`sase update -n` should show nothing pending).
5. Restart the gateway and federation worker through the service host or the documented
   supported path, so the running gateway is the new build. Restart AXE and ACE where
   the update requires it. Record each host's gateway version and protocol, the fleet
   schema, and the AXE and ACE identities with process start times after the restart.
6. Health checks. On Athena, run `sase machine list` (apollo still pinned),
   `sase machine status apollo` (ok, no version skew) and `sase doctor` (dispatch OK).
   On both hosts, run `sase agent index status`. If a host still reports "repair
   recommended", run `sase agent index verify` and then the recommended
   `sase agent index gc`, and confirm that the status is clean.

Record a note on this phase bead that lists the versions, identities, backup-patch
artifact refs, and health results. Close nothing else.

## Phase receipt_live_proof

On the matched build, drive Athena over SSH against Apollo. For each item, capture the
receipts, the `sase machine agent` output, and the Apollo owner state as audited
artifacts.

1. **Fresh-row exact stop.** Dispatch a fresh long-running Apollo agent from Athena.
   Exact-stop it by full dispatch key before the owner's full fleet snapshot rebuilds
   (well inside the old ~7 minute catalog lag). The stop must find the row through the
   index-backed overlay and settle.
2. **Settled receipts.** The stop receipt must settle as `applied` inside the acceptance
   window (at least 30 s), not as `uncertain` on the old 5 s request timeout. Replaying
   the same operation key must return the original settled receipt (`already_settled`)
   with no second effect on Apollo.
3. **Killed row keeps retry.** After the stop, the killed row must still be served to
   Athena with `lifecycle.retry` and `lifecycle.fork` advertised, and it must not be
   dismissed on the owner.
4. **Retry after kill.** Retry the killed row from Athena, and repeat the same-key retry
   three times. There must be exactly one `.r0` execution on Apollo, and each receipt
   must be settled.
5. **Guards still hold.** A stale or replaced locator is refused. Cross-project
   isolation holds. An uncertain operation is never replayed under a new key. Reuse
   `sase-1aq.10.7.5.1`'s accepted fencing evidence where it still applies, rather than
   redoing it.
6. **Gateway restart.** Restart the Apollo gateway, then confirm from Athena that the
   retained killed row and the settled receipts survive the restart.

If an item fails, find the cause and fix it in this phase, in sase-core or sase as
appropriate. Add focused regressions, move the pin if the binding changes, and run
`sase tool run check`. After it lands, redeploy through `matched_deploy`'s steps 4-6 and
rerun the failed items.

Then record the evidence:

- a requirement-to-evidence note on `sase-xe.16.11` that answers its DISCOVERED ISSUE
  items (a) uncertain receipts, (b) catalog lag and (c) reaped killed rows, with the
  live artifact refs;
- a note on `sase-xe.16.11.7.14.6.7.6` that corrects note #3: its "no-reap retention
  (.5.2 contract)" row ran on a pre-contract build, and this note supersedes it with the
  new live evidence;
- a short summary note on `sase-1aq.10.7.5.2`.

Stop or dismiss the test agents you created, and leave both hosts' Agents panes clean.
Do not close any of these beads; they are already closed. Do not close
`sase-1aq.10.7.5`.
