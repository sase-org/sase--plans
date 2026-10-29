---
title: Finish task-backlog epic 2 landing
goal:
  Close sase-1pn after repairing same-turn symbol cleanup, sharing quarantine policy
  through Rust, and reconciling the contract manifest.
tier: tale
size: medium
proposed_by: bbugyi200.athena.sase-1pn.land
proposed_with: codex/gpt-6-astra
bead: sase-1pn
create_time: 2026-10-29 16:30:34
status: wip
---

- **PARENT:**
  [202610/task_backlog_epic_2.md](https://github.com/sase-org/sase--plans/blob/main/202610/task_backlog_epic_2.md)
- **BEAD:**
  [sase-1pn](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1pn/README.md)

# Finish task-backlog epic 2 landing

## Outcome and boundaries

Finish the three remaining gaps in epic `sase-1pn`, verify the combined tree, close its
resolved tasks and the epic, and continue its approved backlog chain. This is one
bounded coding turn; there is no separate land agent for this tale. Do not wait for this
turn's commit SHA, push, or CI before closing the epic. The host commits after the turn
ends.

Read `sase bead read sase-1pn -r "Need landing evidence and current readiness"` and its
linked artifact `plan:202610/task_backlog_epic_2.md` through `sase artifact read`. The
four phases are closed. Preserve the accepted decisions:
`claude_tools=inbound_and_async`, `slot_refusal=quarantine`, and
`toolchain_source=pin_extras`. No new feature choices or memory changes are needed.

The land audit used sase `7e562f58f8` and linked sase-core `1762b33a`. The four epic
commits are `995c451c38` (toolchain), `464125cfa6` (Claude), `8bea2e8414` (close guard),
and `01dcc2e5f5` (quarantine and test repairs). All child notes were read. A 128-test
focused run covering provider settings, stream tripwire, quarantine, allocation, close
guards, formatter setup, and monitor supervisor/ack tests passed. Textual's two named
regression files passed all 32 tests, with Ruff 0.17.3, mypy 2.4.1, and Textual 8.0.1
installed. Seven proposed flake nodes also passed serially. The contract-manifest test
failed as described below. No `just check-full` was run or is requested.

The initial lander examined all seven non-epic commits from `995c451c38^` through
`7e562f58f8`: prepared monitor carry, staged custody adoption, staged questions/pipe,
finalizer import isolation, Monitoring presentation, and the two commit-workflow splits.
The manifest collision is the concrete integration failure. Keep the staged handoff
custody guarantees intact when touching workspace launch adapters.

## 1. Preserve local symbol cleanup while detecting upstream additions

`src/sase/bead/epic_symbols.py:load_epic_symbol_scan` blindly unions local and upstream
rows. The lander reproduced a deadlock in a temporary remote and clone: a row already
exists on the cloned base, the working tree removes that row, but the scan still returns
it with `origins=("origin/master",)`. A normal lander cannot close the bead until its
own cleanup commits, while host completion commits only after its turn. This contradicts
same-turn landing.

Compute the effective merged row set using a proven common Git baseline. A row removed
from the local tree may retire its unchanged upstream counterpart only when the baseline
proves that local deletion. Continue blocking rows newly added upstream after the common
base, rows still present locally, changed/re-keyed upstream rows, and uncertain cases.
Do not interpret mere absence in an old checkout as cleanup. A missing baseline must
preserve the conservative union. Preserve bounded noninteractive fetches, source
diagnostics, and the established warn-and-local/cached fallback on unavailable upstream
reads.

Keep Git discovery and reads in the Python host adapter. Put new shared row
reconciliation policy in the owning Rust core domain and expose it through a thin
binding. Both `epic-symbols` and every close path must use the same effective view:
normal close, phase-only close, forced cancellation, batch `update -s closed`, and the
existing ACE close integration. No bypass flag is needed. Update the relevant CLI
documentation after reading `docs_writing.md` through `/sase_memory_read`.

Add temporary-repository regressions for both cases together: a resolved local removal
permits close before commit, while a new sibling upstream row still blocks. Cover
missing-baseline/fetch-failure behavior and keep existing force/phase/batch checks. Do
not require a push to demonstrate local cleanup.

## 2. Make quarantine decisions shared core behavior

The Rust commit `1762b33a` correctly added `skip_workspaces` to
`WorkspaceClaimRequestWire` and the allocator. However, the new 448-line Python
`src/sase/workspace_provider/slot_refusal.py` owns the refusal record validation,
normalization, first-record decision, stale-record removal, and project skip-set policy.
These are shared domain decisions under the required Rust backend boundary. Moving only
the final skip-set membership into Rust left that boundary unfinished.

Open `sase-core` with `/sase_repo`. Add a small Rust refusal wire/policy API with
binding round-trip tests. It should validate/normalize records, decide record upsert and
removal from supplied observations, preserve one standing refusal per project/slot, and
derive the effective skip set. Python may retain filesystem locks, atomic persistence,
Git cleanliness observations, notifications, and diagnostic rendering; reduce its policy
to calls through the binding. Preserve the on-disk schema or explicitly support the
records this epic has already written. Do not invent a second state store or a Python
fallback.

Route record creation, successful-preservation clearing, clean/gone reconciliation,
pinned-launch refusal, auto-allocation and preview selection, and `workspace repair -C`
through the same policy. Preserve fail-closed nested preservation and one notification
per standing record. Keep the core allocator pure: host-observed state is explicit input
rather than hidden global I/O. Prove that binding consumers get the same decisions and
that the real two-launch scenario still skips the refused slot. Include no-free-slot,
explicit-pin, clean, gone, successful-preservation, and manual-clear cases without
changing their accepted behavior. Avoid stale-observation removal of a newer refusal
record.

Use the existing linked-repo workflow in `docs/contribute/rust-backend.md`. The host
commits a changed linked core first and writes its pushed SHA into
`sase-core-revision.txt` before the primary commit when both are declared. Declare both
repositories; never wait for this turn's own SHA or manually commit early.

## 3. Reconcile the contract manifest with later finalizer work

`01dcc2e5f5` added `tests/test_core_finalizer_handoff.py` and
`tests/test_ratchet_core_window_tool_ref.py` to the contract manifest. But the
intervening finalizer landing `d97902c0df` deliberately removed the former file's
`pytest.mark.contract`: the published core floor 0.42.7 lacks its new bindings. The
lander's focused check now reports 34 passed / 1 failed, with the manifest containing
one extra entry, `tests/test_core_finalizer_handoff.py`.

Refresh `tests/contract_manifest.txt` to the actual selected set, retaining the
ratchet-ref test. Preserve the intentional removal of the finalizer contract marker.
Update stale manifest-budget commentary consistently; do not increase its budget or
claim unmeasured timings. The epic's original Ruff-stub complaint is already fixed in
`01dcc2e5f5`; do not redo it.

## 4. Verify and finish this epic's closeout in this same turn

Run focused tests for the repaired policies, then `just fix` and the required
`sase tool run check` (the wrapped `just check`) in each changed code repository. Follow
each repo's setup and test instructions. Never run `just check-full` without an explicit
new request. Treat any newly caused issue as remaining work; do not hide it in a
follow-up. If a genuinely long check requires `/sase_monitor`, put the entire unfinished
closeout in its follow-up instructions, not a request to wait for this turn's future
commit.

Before closing, reread the epic, all descendants and notes, and the linked plan, and
inspect commits since the audited base. The initial lander already triaged all phase
`PROPOSED FOLLOW-UP` entries; copy the exact dispositions from its `FOLLOW-UP TRIAGE`
note into the close note, including declined duplicates and resolved proposals. Detailed
failure excerpts are at `file:explicit:fa4e166bf4135802647a61a0`. The recorded outcomes
are:

| Proposing phase/note | Disposition                                                                                                                  |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `sase-1pn.2` #2      | Existing `sase-1pg` closed: both monitor stubs fixed by `01dcc2e5f5`, full files pass.                                       |
| `sase-1pn.2` #3      | New ready flake `sase-1po`, large; sudo JSON EOF, related to `sase-15h`, cause unknown.                                      |
| `sase-1pn.3` #2      | No new task: current manifest collision remains epic integration work in step 3.                                             |
| `sase-1pn.3` #3      | AGY corroborated on `sase-19m`; pager flake `sase-1pp` and modal flake `sase-1pq` created ready, large.                      |
| `sase-1pn.4` #1      | Dismissal flake `sase-1pr` created ready, large, related to `sase-1on`; both claim-prepass nodes corroborated on `sase-1oo`. |

Unknown causes were not assigned to an active epic merely because its topic overlaps.
The original epic note #1 formatter regression is already resolved by `01dcc2e5f5`; it
needs no new task or repeated patch.

Confirm every `TASK RESOLVED` claim from phases `sase-1pn.1` through `.4` against the
final tree. Close `sase-163`, `sase-1p9`, `sase-o7`, `sase-1cs`, and `sase-vr` with
evidence if still open. The Claude and toolchain fixes already have landed SHAs above;
the final quarantine and close-guard repairs may close with tested working-tree evidence
before their host-owned commits. Do not close snoozed tasks or tasks belonging to
another still-active epic as opportunistic cleanup.

Use `/sase_repo` to open the plans sidecar and audited `sase artifact read` for plan
context. Honor the original carry-forward: confirm `status: done` in
`plan:202610/task_backlog_epic_1.md`, `plan:202610/rebased_stitch_gate.md`, and
`plan:202610/finish_rebased_stitch_gate.md`, updating any still `wip` as explicitly
authorized by the epic plan. Include changes to that repo in the final declaration.

Run `sase bead epic-symbols sase-1pn`. Resolve each row by wiring, privatizing, adding a
justified non-test pragma, or deleting it under `symvision.md`; re-key only when an
identified still-open later bead actually needs the exemption. Then close normally:

```sh
sase bead close sase-1pn --note "Verified all four phase scopes and notes and the accepted decisions; reviewed the epic commits and post-start drift; repaired same-turn upstream symbol reconciliation and Rust quarantine policy ownership; reconciled the contract manifest with d97902c0df; focused tests and just check passed. Include exact test evidence and every FOLLOW-UP TRIAGE disposition here."
just symvision
```

Replace the example close note with actual final evidence and triage outcomes. If
leftovers are reported, resolve them and retry. If named phases are incomplete,
finish/reopen them instead. Never use `--force` merely to make the close succeed, and
never use it to advance a successful nested landing.

Set `status: done` in the frontmatter of the epic's linked plan,
`202610/task_backlog_epic_2.md`, in the opened plans repo. Also mark this tale done when
its work is complete. Re-read `sase-1pn` for its parent link: the initial audit found
`parent_id: null` and no ancestors, so `sase-1pm` is a historical backlog predecessor,
not a containing parent to close. If that relationship changed, follow the live parent
type: close only a completed parent phase normally, or fully review a directly parented
plan's descendants, notes, linked plan, drift and symbols before closing it and marking
its plan done. Stop at the first incomplete parent with a blocker note.

The original epic also requires continuing the backlog chain. Do that as the last part
of closeout: author and validate the next epic before proposing it, and propose only
after all the closures, symvision, and plan-status edits above are complete. Pass these
original rules verbatim to its land agent:

1. Re-list every non-closed task bead (`sase bead list -t task -n 0`). Close only beads
   you can verify are duplicates (resolution `superseded`, naming the canonical bead,
   with a note on the canonical) or obsolete (resolution `done`, citing the fixing
   commit or the grep that proves absence, or a passing focused test). Never close a
   bead an active epic owns. Never close a snoozed bead early. Beads that are merely
   dormant stay open.
2. Pick the 5 most important remaining task beads. Rank them by fleet-wide cost first
   (this means: breaks every agent's check or CI, breaks or wastes every landing,
   security or data-loss risk, heavy +1 corroboration), then by cost to fix.
3. Write an epic plan with at most 5 phases and at most 5 task beads. One phase may work
   several beads. Use plan decisions sparingly, but always give the owner a way to opt
   out of feature work and memory changes that might not be wanted.
4. That epic's Landing section must carry this same "continue the backlog chain" rule,
   until no non-closed task beads remain.
5. Propose it with `/sase_plan`. Proposing ends the land agent's turn, so do it last.

Use `/sase_plan`'s validate-with-explain, revalidate, then propose loop. If proposal
reports Staged, finish through `/sase_final`; legacy proposal handoff ends the turn
mechanically. There is no need to wait for the next backlog epic to finish before
closing this one.
