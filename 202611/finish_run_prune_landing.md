---
tier: tale
title: Repair run-trash preservation and finish sase-1rd
goal:
  Purge honors every fresh protection, kept runs remain intact, notifications report the
  actual result, and sase-1rd closes after verification.
size: medium
proposed_by: bbugyi200.athena.sase-1rd.land
proposed_with: codex/gpt-6-astra
bead: sase-1rd
create_time: 2026-11-06 13:19:50
status: wip
---

- **PARENT:**
  [202611/actionable_run_prune_notifications.md](https://github.com/sase-org/sase--plans/blob/main/202611/actionable_run_prune_notifications.md)
- **BEAD:**
  [sase-1rd](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1rd/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1rd.land](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1rd.land.md)

# Repair run-trash preservation and finish sase-1rd

Implement the remaining work below, verify it, and close epic `sase-1rd` in this turn.
This tale has no land agent, and the original lander will not resume after you. The
existing preview, CLI, modal, and quarantine implementation are already present; do not
rebuild them. The bounded repair spans the existing Rust trash owner, its Python
adapter, and notification presentation.

## Binding scope and decisions

Read `sase bead read sase-1rd -r "Need current landing notes and descendants"` and the
original design through
`sase artifact read plan:202611/actionable_run_prune_notifications.md "Need approved run-prune contract"`.
Honor the final decisions: `auto_purge = yes`; `decision_record = no`; no memory edits.
The hourly job must still purge eligible, previously confirmed batches. Unattended run
quarantine remains refused by `apply_ace_run_retention`, disk reap, and disk-pressure
housekeeping.

All lifecycle tests use disposable projects roots, trash roots, indexes, notification
stores, and SASE_HOME. Never apply, restore, or purge real host runs. Open the linked
core with `/sase_repo` and
`sase repo open sase-core -r "Repair run-trash landing defects"`; read its AGENTS.md.
Shared policy and path safety belong in Rust. Python gathers observations, calls
bindings, updates indexes, and renders results. Read the required reference memories
through `/sase_memory_read`: `lint_and_test.md`, `symvision.md`, `sase_artifacts.md`,
and, before TUI edits, `ace.md`, `tui.md`, `tui_perf.md`, and `tui_screenshot.md`. Read
`tools.md` before changing binding validation tools and `docs_writing.md` before
documentation edits.

## What the landing audit established

At sase `d44032ad79` and core `28159925`, all four phases are closed. Their source
commits are core `39744416` and sase `37a4c8ed8d`, `950c8aec15`, `e94a52e0ae`, and
`d44032ad79`. The current core pin contains the original core change. No epic commit
edited memory. Sixty-one focused tests across review snapshots, CLI apply/restore, the
preview job, the view model, pilot interaction, and the full fixture round trip pass.
Those tests omit the following confirmed cases:

1. Quarantine an old terminal run whose `done.json` names `protected-agent` and whose
   `agent_meta.json` has that name and `bead_id: demo-task`. Calling
   `purge_ace_run_trash` with only `protected_agent_names={"protected-agent"}` reports
   `purged=1, restored=0`. Repeat with only
   `non_closed_bead_ids_by_project={"demo": {"demo-task"}}`: same deletion. Both
   original and trash paths disappear. The facade forwards only protected dirs and
   timestamps, dropping the other collected protections. It also never computes the
   continuation closure for trashed runs.
2. After a successful quarantine, move `<batch>/runs` to a sibling disposable directory
   and replace it with a symlink to that directory. Explicit restore reports
   `restored=1` and moves the external run. `collect_trash_runs` checks
   project/month/day/run entries but not the `runs` ancestor; restore does not apply
   `safe_trash_item`. `ensure_safe_batch` canonicalizes away an internal symlink before
   checking its components. The protected-item branch of purge uses `restore_one`
   without checking the source containment either.
3. Put an empty child directory inside a run before quarantine. Recreate the original
   run path to cause a restore conflict, then purge with the timestamp protected. The
   result says `kept=1`, but its empty child has disappeared. `cleanup_empty_batch`
   recursively prunes inside retained run contents.
4. `_notify_result` calls `_dismiss_live_previews` even for a blocked `busy` result. The
   modal's completion callback instead tells the user “Preview kept”. Dismissal also
   happens before writing the replacement result notification.

The audit additionally found fixed 14-day promises in the job and modal while the actual
deadline uses configurable `trash_grace_days`, and index errors omitted from the typed
result/success decision used by the TUI.

The lander has already privatized the internal snapshot-directory helper and removed
`--epic-symbol "sase-1rd(snapshot_dir)"` from the Justfile. That change lands through
host completion before this tale starts. This is deliberate: the close guard examines
both local and default-branch Justfiles. Do not create a new dependency on this tale's
own future commit or push before closing.

## 1. Make purge honor the complete fresh protection set

Repair `src/sase/core/agent_artifact_run_trash.py` and
`crates/sase_core/src/agent_artifact_run_trash/` so purge evaluates current references
by original identity even while bytes live in trash. Reuse the existing protection
predicates and continuation retention owner; do not invent a second Python eligibility
policy.

- Carry the fresh named-agent and per-project non-closed-bead facts through the Rust
  contract. Read run metadata from the validated trash location and compare agent
  aliases and bead, phase, and epic IDs using the same semantics as preview. Current
  path and timestamp protections keep working when the original path no longer exists.
- Supply continuation observations for the union of relevant live runs and trash
  candidates, mapping each physical trash location to its original logical run identity.
  A live/recoverable continuation or monitor must protect its ancestry after quarantine.
  Preserve the existing Rust continuation closure rather than treating a missing
  original directory as evidence that it has no metadata.
- Newly protected items restore to their original paths when safe. Conflicts keep the
  complete item in trash. Missing or unreadable required metadata/protection
  observations block destructive work and report why. Check terminal/active markers
  again on the actual stored run before permanent deletion.
- Keep eligibility decisions and validation in Rust, adding a typed observation
  interface if needed. Any new safety evidence is required, with no empty default that
  silently authorizes an older caller to delete. Update PyO3 bindings, schema mirrors
  and their round-trip/validator tests together if the wire changes. Preserve
  compatibility by an explicit versioned contract/refusal, not an unsafe fallback. Do
  not hand-edit package versions.

Add real binding/facade regressions for new `agent:name` protection, reopened
bead/phase/epic protection, newly live continuation ancestry, direct path and timestamp
protection, absent evidence, and unreadable metadata. Check retained bytes and index
re-insertion after auto-restoration, not just counters.

## 2. Constrain every trash effect and preserve kept trees

In the Rust helpers and restore/purge owners, validate lexical components under the
canonical trusted root before reading or moving a batch/run. Reject planted symlinks at
the batch, runs, project, month, day, run, and journal positions as appropriate.
Canonicalization must not erase the evidence being checked. Keep the existing macOS
`/tmp` ancestor convention by canonicalizing the trusted root consistently; links
planted below that root are rejected.

Apply the same source-containment check to explicit restore and purge's protected
restore branch. Neither may traverse a symlink or mutate an external tree. Restrict
empty-batch cleanup to empty structural directories above run roots; never recurse
through a retained run's payload, even when it contains only empty directories. Exercise
a batch containing both successfully handled and kept runs. Do not lose successful
move/restore outcomes if a later journal or cleanup write fails: report partial outcomes
so the adapter can still deindex/reindex those paths. Add fault-injection coverage for
such post-effect errors.

Retain the existing orphan discovery/restore behavior for a crash after rename but
before journal append. Preserve the orphan's `original_path` in the Python listing wire
too: it currently parses the Rust orphan record as a generic item outcome, dropping that
field. New tests should compare directory structure as well as file contents, and assert
that external targets remain unchanged for both explicit restore and auto-purge
restoration.

## 3. Make completion reporting and retention promises accurate

In `src/sase/artifact_cli/prune_runs.py`, only retire previews after a completed apply
and successful result publication. Blocked or error outcomes leave the review available
and publish accurate diagnostics when possible. A notification store failure must not
turn an otherwise successful filesystem operation into a failed operation, and must not
discard the only live preview. Expected kept items are normal successful outcomes;
distinguish them from I/O/index errors.

Include index-follow-up errors in typed operation results and visible completion
reporting. The CLI JSON, textual report, result notification, and modal callback must
agree about complete/partial/blocked behavior and what actually moved. Report the
restore command for any successfully created batch even on partial failure. Test blocked
`busy`, unavailable protections, partial moves, index failure, result-store failure, and
success with kept items using isolated stores.

Replace fixed 14-day promises in `src/sase/scripts/sase_chop_artifact_run_prune.py` and
`src/sase/ace/tui/modals/artifact_run_prune_modal.py` with the configured grace period,
loaded outside the TUI message pump. Reuse the existing configuration knob. Test a
non-default period. Consolidate the duplicated human-size formatting touched by these
repairs, as the original plan required; keep the existing KiB/MiB/GiB output. Use an
unavailable-project label consistently when a project cannot be resolved: the job and
CLI reports currently fall back to the raw key, while the modal correctly says “Project
unavailable”. Refresh only affected visual goldens if rendered output changes.

## 4. Complete integration and verification

Recheck changes since `d44032ad79` before finishing. The lander reviewed all non-epic
commits since the epic's first commit, plus `71fd5b6c2a` which landed between epic
creation and that first commit. Relevant neighbors are hijack recovery/inheritance
(`3fe8d5d34b`, `7896ea657b`, `fb61982a41`, `812ddbc3ae`, `efac88d0b9`, `a7d68a2e71`),
monitor retention and staged adoption (`520962bcba`, `c65055eff6`, `956310b7ad`), and
their later module splits. Their retained predecessor/continuation dependencies must
survive the purge repair. Add a focused fixture proving a recovery/monitor successor can
newly protect a run already in trash. Publication recovery, bead opener attribution,
slow-tool ordering and unread navigation changes have no duplicate prune implementation;
preserve their behavior.

Run focused regression tests and the existing 61-test set, including
`tests/ace/tui/test_artifact_run_prune_modal_e2e.py`. Verify dry-run and unattended
apply refusal remain unchanged. Run `sase tool run check` in the modified core checkout
and in sase; this is the guarded `just check`, not `just check-full`. Read
`/sase_monitor` before commands that require a monitor. Any verification follow-up must
still carry the explicit closeout in step 5; do not silently stop after the code is
tested. Inspect affected PNGs if a targeted screenshot update is needed. The original
design requested a live review screenshot without confirmation; capture it if available,
otherwise record why it could not be obtained and retain the disposable pilot/visual
proof.

The known unrelated isolation failures are owned by active `sase-1ra` and still
reproduce on clean `d44032ad79`; use independent baseline/ToolRun evidence if they
persist. Do not weaken tests or run check-full to make this epic green. Treat new
unexplained failures as yours.

The lander's required `just check` settled with exit 1 in ToolRun
`dee7bd4a2b6bb98b8cc573f669a0e722`: all lint stages passed, including Symvision; the
test stage reported 60,314 passed, 17 skipped, and the same two isolation failures
(`test_no_undo_on_shared_monkeypatch_fixture` and
`test_session_sandbox_seal_is_not_first_test`). Both were independently reproduced
before the lander's three-file symbol cleanup. The automated triage also extracted a
nested relative-path spelling of the first failure as NEW; this is not a third failing
test. The verification command remained failed, not waived or relabeled.

For simultaneous changes in sase-core and sase, `/sase_final` commits both. The host
commits the linked core first and writes its pushed SHA into `sase-core-revision.txt`
before the sase commit (documented under “The CI source revision pin” in
`docs/contribute/rust-backend.md`). Include both repository decisions. Do not wait for
your own commit SHA, push, or CI before step 5; the same-turn close is the intended
flow. Ratchet manually only for an already-landed core dependency, using the existing
recipe.

## 5. Finish this epic's closeout in the same turn

Read every current descendant and note again and verify the linked plan's work is
complete. Do not close with an unresolved preservation or reporting defect. Carry
forward the lander's follow-up dispositions into the close note:

- `sase-1rd.1` note #2, `.2` note #1, `.3` note #2, and `.4` note #1: the isolation
  guard failures were consolidated on active `sase-1ra` with fresh reproduction; no
  duplicate task.
- `.1` note #2 and `.2` note #2 marker-audit portions: exact node now passes; existing
  `sase-1by` received resolution evidence, so no new task or failure +1.
- `.2` note #2 screenshot prepare-error flake: ready task `sase-1rh`, related to
  `sase-1pd`. The current focused run passed; no fresh failure was claimed.
- `.3` note #1 phase-close/symbol proposal: already resolved by its note #3 and close.
  The lander removed the last epic-owned snapshot helper exemption.

Run `sase bead epic-symbols sase-1rd`. Resolve each remaining listed entry by wiring its
actual consumer, privatizing it, adding a justified non-test pragma, or deleting it.
Re-key only when a still-open later bead really needs the exemption. The prior
snapshot-dir exemption should already be gone upstream because the lander's integration
commit precedes this tale. Never evade the guard or add an exemption merely to close.

Close normally with
`sase bead close sase-1rd --note "<source and descendant verification; fixed preservation and notification cases; concurrent-change integration; checks and all follow-up dispositions>"`.
If rejected for a leftover symbol, finish its cleanup and retry. If named phases are
incomplete, finish or reopen them. Never use `--force` merely to succeed or to advance a
successful nested landing.

After close, run `just symvision` to confirm the whitelist is clean. Open the plans repo
through `/sase_repo` and set `status: done` in the frontmatter of
`202611/actionable_run_prune_notifications.md` under the printed checkout path. Its
artifact reference is `plan:202611/actionable_run_prune_notifications.md`; use
`sase artifact read` for any further content read. Include this modified repo in
finalization.

Finally run `sase bead read sase-1rd -r "Need the parent link"`. The lander found no
parent, so no ancestor close is currently needed. If a parent link has been added,
inspect it: close only a complete parent phase and leave its containing epic to its
lander; for a parent plan, re-audit its prior landing note, descendants, linked plan and
drift before any normal close, symbol check and plan-status update. Stop at an
incomplete or ambiguous ancestor and record the blocker there. Finish through
`/sase_final` after all commands exit.
