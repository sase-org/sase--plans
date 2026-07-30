---
tier: epic
title: Idempotent bead close-and-publish, then repair the projection
goal: 'Closing an already-closed bead is a verified no-op that writes no event, no
  commit and no changed byte; a second close that arrives anyway cannot move a close
  timestamp or erase a close reason; concurrent note appends converge instead of overwriting
  each other; and the 313 bead rows already damaged by past duplicate closes are repaired
  by re-deriving `issues.jsonl` from the canonical event log rather than by rewriting
  history.

  '
phases:
- id: core-close-interval
  title: Closed-interval semantics in the event reducer
  depends_on: []
  size: medium
  description: 'core-close-interval: make `apply_event` treat a close of an already-closed
    bead as an exact no-op and clear `closed_at`/`close_reason` on every transition
    out of closed, so duplicate close events cannot move a close timestamp and the
    projection converges regardless of merge order.

    '
- id: core-close-verified
  title: Verified idempotent close in the mutation layer
  depends_on:
  - core-close-interval
  size: medium
  description: 'core-close-verified: preflight every requested ID into closed / already-closed
    / conflicting, abort the whole batch before any write when an explicit resolution
    or reason disagrees with the recorded close, and report the classification through
    new outcome-wire fields.

    '
- id: core-note-append
  title: A convergent note_appended event
  depends_on:
  - core-close-verified
  size: medium
  description: 'core-note-append: add a `note_appended` event whose payload carries
    only the entry text, move note rendering into the reducer so appends compose instead
    of replacing, keep legacy whole-string note events working unchanged, and turn
    an unknown event kind into an actionable error.

    '
- id: cli-close
  title: An honest sase bead close command
  depends_on:
  - core-close-verified
  size: medium
  description: 'cli-close: stop printing "Closed" for beads that were already closed,
    distinguish cascade closes from requested ones, make the auto-commit message name
    what actually happened, and stop the default resolution from manufacturing a conflict
    on every commit-hook close.

    '
- id: doctor-projection
  title: Projection drift detection and repair
  depends_on:
  - core-close-interval
  size: medium
  description: 'doctor-projection: teach doctor to compare `issues.jsonl` against
    the reduction of the canonical streams, census redundant close events with a recent-window
    rate, and add a guarded `--fix-projection` repair that refuses any diff outside
    the expected shape.

    '
- id: history-notes
  title: Redundant closes and restores in history
  depends_on:
  - core-close-interval
  size: small
  description: 'history-notes: label close events the reducer treated as redundant
    so a timeline reads honestly, and add a non-interactive confirmation flag so lost-note
    restoration can be run under an approval gate.

    '
- id: floor-docs
  title: Core floor bump, docs, and full check
  depends_on:
  - core-note-append
  - cli-close
  - doctor-projection
  - history-notes
  size: small
  description: 'floor-docs: raise the published sase-core-rs window to the release
    containing the reducer and event changes, document the close idempotency contract
    and the projection repair, refresh the beads skill source, and run the full check.

    '
- id: repair
  title: Repair the live store under an approval gate
  depends_on:
  - floor-docs
  size: small
  description: 'repair: run the guarded projection repair against the live beads sidecar,
    verify the diff shape mechanically before committing, then restore the lost note
    revisions the research report deferred until the write path was safe.

    '
create_time: 2026-07-30 13:43:32
status: wip
bead_id: sase-bd
---

- **PROMPT:** [202607/prompts/bead_close_integrity.md](prompts/bead_close_integrity.md)
- **BEAD:** [sase-bd](https://github.com/sase-org/sase--beads/blob/main/pages/sase-bd/README.md)

# Idempotent bead close-and-publish, then repair the projection

Implements recommendation 1 of `202607/sase_beads_close_integrity_and_capture/` in the `research` sidecar: _"Make
close-and-publish idempotent, then repair the historical duplicates."_

## Problem

`sase bead close` is not idempotent in the one place it matters. When a second close reaches a bead that is already
closed, the store silently rewrites that bead's `closed_at` forward and replaces its `close_reason` with whatever the
second call carried — usually `null`. Measured against the live store on 2026-07-30 (2,417 beads, 418 streams, 10,754
events):

| Signal                                                               |                                 Value |
| -------------------------------------------------------------------- | ------------------------------------: |
| Bead rows whose `closed_at` is later than their real first close     |                               **312** |
| Bead rows whose `close_reason` was erased by a later redundant close |                                **36** |
| Redundant `issue_closed` events in the log                           |                               **450** |
| Largest single-bead error                                            | **16 h 2 min** (p50 129 s, p90 508 s) |
| Redundant closes recorded since 2026-07-08                           |                                 **2** |
| Merge commits in the beads sidecar                                   |                          **0 of 773** |

`sase-b8.8` is the clean live reproduction, from 2026-07-30:

```text
16:10:17Z  issue_closed   closed_at := 16:10:17Z    ← real completion
16:22:45Z  note_appended-equivalent (issue_updated · notes)
16:22:45Z  issue_closed   closed_at := 16:22:45Z    ← rewrites the close 12 minutes forward
```

Two separate `chore(beads): close sase-b8.8` commits, twelve minutes apart, for one piece of work.

Every cycle-time metric in the store terminates on `closed_at`. While a third of closed beads carry a close time that is
a rewrite of the real one, phase duration, lead time and throughput are not measurable — which is why research items 7
and 9 are blocked behind this one.

## Root cause

The research report attributes this to close re-entrancy and rules out cross-clone divergence because the sidecar has
zero merge commits in 773. Re-measurement supports the first half and refutes the second.

**Re-entrancy is already fixed.** `MutableStore::close_one` (`crates/sase_core/src/bead/mutation.rs`) returns `None`
when the target is already closed, so a repeat close inside one clone writes no event. The guard is live and provable:
on 2026-07-30 a second `sase bead close sase-b8.6 --note …` appended its note and emitted **no** `issue_closed`. The
per-day duplicate rate collapsed from ~85% of closes before 2026-07-08 to 2 events in the three weeks since.

**The residue is cross-clone, and zero merge commits is not evidence against it.** Each of 21 workspaces holds its own
clone of the `beads` sidecar. The managed sync worker _rebases_, and `src/sase/bead/conflict_resolver.py` resolves the
resulting conflicts by a semantic three-way merge of the event streams (`merge_bead_event_streams` →
`reduce_event_streams` → rewrite `issues.jsonl`). Two clones that each observe a bead as open will each mint a
legitimate `issue_closed`; the merge correctly keeps both; the history stays perfectly linear. Zero merge commits is the
_expected_ signature of this architecture, not a refutation of concurrency.

**So the damage is done by the reducer, not by the writer.** `apply_event` (`crates/sase_core/src/bead/events.rs`)
applies every `IssueClosed` unconditionally:

```rust
BeadEventPayloadWire::IssueClosed { close_reason, resolution, .. } => {
    let issue = existing_issue_mut(issues, &event.issue_id)?;
    issue.status = StatusWire::Closed;
    issue.closed_at = Some(event.timestamp.clone());   // last close wins
    issue.close_reason = close_reason.clone();         // null overwrites a real reason
    issue.resolution = resolution.clone();
    issue.updated_at = event.timestamp.clone();
}
```

This is the load-bearing fact for the whole plan: **`issues.jsonl` is a pure derived projection.** `read_store_issues`
(`crates/sase_core/src/bead/read.rs`) and `MutableStore::load` both reduce the streams on every read and every mutation;
`export_jsonl` is literally _load-then-write_. Nothing reads `issues.jsonl` as truth. So a reducer that is idempotent
under duplicate closes does not merely stop the bleeding — it retroactively repairs every row that was ever damaged,
without deleting or rewriting a single event.

The same replace-not-append shape damages notes. A note append is stored as
`issue_updated { notes: <whole new string> }`, so when the merge keeps two concurrent appends the later one's snapshot
silently drops the earlier one's text. That is the mechanism behind the lost note revisions the report attributes to
snapshot-shaped storage.

## Approach

Four properties, in dependency order. Each is a structural guarantee, not a guard that a future caller can forget.

**P1 — the projection is idempotent under duplicate closes.** Introduce the notion of a _closed interval_: the span
between a bead becoming closed and its next transition out of closed. The first `issue_closed` of an interval defines
`closed_at`, `close_reason` and `resolution`; every later one in the same interval is an exact no-op. Reopening — via
`issue_opened`, via `issue_updated` with a non-closed status, or via `epic_work_preclaimed` — ends the interval and
clears `closed_at`/`close_reason`, so a genuine reopen-then-reclose still records its new close time.

The rule keys entirely off the projected `status` and `closed_at`, so it needs no extra reducer state, and it is
order-insensitive: replaying two closes equals replaying one. That is what makes the derived state _convergent_ across
clones, which is the property the store actually needs and does not have.

**P2 — the command is a verified no-op, not a silent one.** A close of an already-closed bead exits 0 and reports
`Already closed`, but only after checking that the request agrees with the record. Asking for `--resolution canceled` on
a bead already closed `done` is a conflict, not a no-op: the batch aborts before any write and points at
`sase bead open`. This is the "verified" half of the research item — idempotency must never be a synonym for silently
discarding what the caller asked for.

**P3 — note appends converge.** Add a `note_appended` event carrying only the entry text; the reducer renders
`[timestamp · actor] entry` from the event's own metadata and appends. Two concurrent appends survive a merge as two
events and reduce to two entries in timestamp order. Legacy `issue_updated { notes }` keeps replace semantics forever,
because the log is immutable and `sase bead update --notes` genuinely _is_ a rewrite.

**P4 — publish writes nothing when nothing changed.** A no-op close produces no event, so no store bytes change, so
`mutation_changed` is false and no commit is created. When only a note landed, the commit says `note`, not `close`.
Bead-page publication is already content-compared and stays as-is; the phase pins that with tests.

Then the repair: because of P1, re-deriving `issues.jsonl` from the streams _is_ the repair. Simulating the new reducer
over the live store gives an exact, checkable expectation:

| Property of the repair diff                                        | Simulated value                                          |
| ------------------------------------------------------------------ | -------------------------------------------------------- |
| Rows changed                                                       | 313 of 2,417                                             |
| Rows whose `status` changes                                        | **0**                                                    |
| Fields that change                                                 | `closed_at` (313), `close_reason` (36), `updated_at` (5) |
| `closed_at` changes that move _later_                              | **0** — every one moves earlier, or to null              |
| Beads currently reopened but still advertising a stale `closed_at` | 1 (`sase-9n.3`)                                          |

Those five lines become the repair command's refusal contract, so the repair cannot do anything surprising even if the
store has moved on by the time it runs.

### Decisions

- **Fix the reducer, not the merge.** The merge could be taught to drop a second close, but that would destroy the
  record of two agents both believing they finished the work, and it would be unable to distinguish a redundant close
  from a genuine reopen-and-reclose. The log stays an honest account of what was attempted; the projection becomes the
  single place that decides what it means. This is the append-only-correct answer and it is what makes the historical
  repair free.
- **Repair by re-projection, never by rewriting events.** No `issue_closed` event is deleted, renumbered or amended.
  `event_id` embeds a per-stream ordinal (`sase-aq:000029:…`), so any rewrite would cascade; more importantly, a store
  whose history can be edited to make a metric look right is not a store anybody should trust.
- **Reopen clears `closed_at`.** Today a reopened bead keeps advertising its old close time, which is both wrong on its
  face and the thing that makes "is this interval already closed?" ambiguous. Clearing it is a correctness fix that the
  interval rule also needs; `sase bead history --format full` will show the `closed_at → null` transition, which reads
  correctly.
- **`--resolution` defaults to nothing, not to `done`.** The argparse default of `"done"` makes it impossible to tell
  "the caller did not care" from "the caller asked for done", which would turn P2's conflict check into a false alarm on
  the commit hook's bare `sase bead close <id>`. The Rust layer keeps defaulting an absent resolution to `Done` when it
  actually closes.
- **A new event kind, not a new optional field.** Adding `notes_append` to the existing `issue_updated` fields struct
  would make old binaries fail the "issue_updated event has no fields" validation anyway, so the additive-field escape
  hatch buys nothing. A distinct `note_appended` operation is honest in the log, lets `history` show a real note
  timeline, and is the natural first typed entry for the append-only journal in research item 11.
- **Rollout is a loud error, not silent divergence.** A workspace running a `sase_core_rs` older than this epic will
  refuse to read a store containing `note_appended`. That is correct — the alternative is two binaries writing divergent
  projections of the same store. Phase `core-note-append` turns that failure into a message that names `just install` as
  the fix, and phase `floor-docs` raises the published floor so it cannot happen through a normal install.
- **The repair is a product feature, not a script.** `sase bead doctor --fix-projection` and its read-only drift warning
  outlive this epic: any future reducer correction, or any bad merge, is detected and repaired the same way.

### Non-goals

- **Actor attribution on close** (research item 4). `issue_closed` is 0% agent-attributed and stays that way here;
  fixing the mutation plumbing so the acting agent reaches every event is its own item and touches every mutation, not
  just close. This epic only makes the close _time_ trustworthy.
- **Append-only journal entries** (research item 11). `note_appended` is deliberately minimal — one text field, no typed
  kinds, no content-derived identity. The typed-entry design belongs with the completion-evidence work.
- **Preventing a second close from being attempted.** `src/sase/workflows/commit/commit_hooks.py` closes the linked bead
  on every `sase commit`, and phase agents close their own bead explicitly. That redundancy is now harmless and stays;
  narrowing it is a prompt-and-workflow question, not an integrity one.
- **`sase bead ready` / `open` semantics, the capture prohibition, `--format json` on `ready`/`blocked`/`stats`, the
  remaining `doctor` gaps, and `sase bead rm --dry-run`** are research items 2, 3, 6 and 7. Only the projection-drift
  and redundant-close diagnostics are added to `doctor` here.
- **No change to `bd/*` prompts** in `src/sase/default_config.yml`. They already instruct agents to
  `sase bead close <id> --note "…"`, which becomes strictly safer.

## Phases

Rust work lands in the linked core repo; open it with `sase repo open sase-core -r "<reason>"` and use the printed path
as the only path for reads and writes. Commit there with Conventional Commits so release-plz computes the version, and
never hand-edit `[workspace.package].version`. `just install` in a sase workspace builds `sase_core_rs` from that linked
checkout, so a Python phase can develop against core changes as soon as they are on the core repo's canonical branch.
This repo's `CHANGELOG.md` is generated by release-plz from commit subjects; do not hand-write entries, and remember
`just install` before `just check` in an ephemeral workspace.

### Phase `core-close-interval` — closed-interval semantics in the event reducer

**Depends on:** nothing.

In `crates/sase_core/src/bead/events.rs`, `apply_event`:

- `BeadEventPayloadWire::IssueClosed`: when the issue is already `Closed` **and** already carries a `closed_at`, change
  nothing at all — not `closed_at`, not `close_reason`, not `resolution`, not `updated_at`. Otherwise apply as today.
  The `closed_at.is_some()` half is load-bearing: `sase bead update --status closed` sets the status without stamping a
  close time, and that path must still accept the `issue_closed` that follows it.
- Every transition out of closed clears `closed_at` and `close_reason` alongside the `resolution` that is already
  cleared: `BeadEventPayloadWire::IssueOpened`, `apply_update_event_fields` when `fields.status` is present and not
  `Closed`, and `BeadEventPayloadWire::EpicWorkPreclaimed`.

`MutableStore::load` and `read_store_issues` both reduce through this function, so the mutation layer inherits the rule
with no further change. Confirm `close_one`'s existing already-closed guard still short-circuits first, so a same-clone
repeat close remains event-free rather than relying on the reducer to absorb it.

Tests in `events.rs` and `crates/sase_core/tests/bead_event_parity.rs`:

- close → close reduces to one close time, one reason and one resolution; the second event is inert.
- close → open → close records the _second_ close time, and `closed_at` is null while the bead is open.
- close → `issue_updated{status: in_progress}` → close records the second close time.
- close → `epic_work_preclaimed` → close records the second close time.
- `issue_updated{status: closed}` → `issue_closed` stamps `closed_at` from the close event.
- close carrying a reason, then a redundant close carrying `close_reason: None`, keeps the reason.
- reducing `[close_a, close_b]` and `[close_b, close_a]` yields identical issues (order insensitivity), which is the
  property the cross-clone merge depends on.

Check `crates/sase_core/tests/bead_storage_parity.rs` and `bead_read_parity.rs` for fixtures that assert a `closed_at`
this change corrects, and update them deliberately rather than by regeneration.

**Done when:** the core suite passes with duplicate closes proven inert and reopen-reclose proven intact, and the commit
is on the core repo's canonical branch.

### Phase `core-close-verified` — verified idempotent close in the mutation layer

**Depends on:** `core-close-interval`.

In `crates/sase_core/src/bead/mutation.rs`, `close_issues_with_note`:

- Preflight every requested ID before any mutation, alongside the existing unclosed-descendant check, classifying each
  as `close`, `already_closed` or `conflict`. A conflict is an already-closed bead where the caller supplied an explicit
  `resolution` differing from the recorded one, or an explicit non-empty `reason` differing from the recorded
  `close_reason`. The whole batch aborts on any conflict, with a `BeadError` naming the bead, the recorded resolution
  and close time, the requested resolution, and both remedies (`sase bead open <id>`, or `sase bead note <id> '…'`).
  This preserves the existing all-or-nothing batch contract — see `batch_close_preflights_every_request_before_writing`.
- Change the `resolution` parameter so an absent resolution is distinguishable from an explicit `done`. Keep defaulting
  an absent resolution to `Done` when the bead actually closes, so no existing caller changes behavior.
- Extend `BeadMutationOutcomeWire` with `already_closed_ids`, `noted_ids` and `cascade_closed_ids` (delegated-parent and
  forced-descendant closes), all `#[serde(default)]`. `closed_ids` stays the set this invocation actually closed. Stop
  pushing already-closed issues into `issues`, which is what makes the CLI report them as closed today; return them
  through `already_closed_ids` and keep their `IssueWire` available for rendering.
- `changed` stays honest: true when anything was closed or noted, false otherwise.

The note-append loop keeps running for already-closed beads — evidence about a completed bead is welcome — but it is now
reported separately from the close.

Tests in `mutation.rs`: repeat close writes no event and no changed byte; repeat close with a note writes the note only
and leaves `closed_at` untouched; conflicting resolution aborts a mixed batch with nothing written; conflicting reason
aborts likewise; an absent resolution against a bead closed `canceled` is not a conflict; cascade closes appear in
`cascade_closed_ids` and not in the requested set.

**Done when:** a second `close` of a closed bead is provably write-free, a disagreeing close is refused before any
write, and the outcome wire distinguishes closed, already-closed, cascaded and noted beads.

### Phase `core-note-append` — a convergent `note_appended` event

**Depends on:** `core-close-verified`.

In `crates/sase_core/src/bead/events.rs`:

- Add `BeadEventOperationWire::NoteAppended` and `BeadEventPayloadWire::NoteAppended { entry: String }`, with
  `validate_for` pairing them and rejecting a blank entry. Give it the same merge priority as other issue-scoped
  operations in `event_operation_priority`.
- Move the note formatter (`appended_note_text`, currently in `mutation.rs`) into `events.rs` and have `apply_event`
  render `[{event.timestamp} · {event.actor}] {entry}` and append it to `notes`, setting `updated_at`. The reducer
  becomes the single owner of note rendering, so stored text is a function of the event rather than of whichever writer
  produced it.
- Leave `BeadEventPayloadWire::IssueUpdated { fields: { notes } }` replace semantics exactly as they are. Existing
  streams are immutable and `sase bead update --notes` is a deliberate rewrite.

In `crates/sase_core/src/bead/mutation.rs`, `append_note_to_store` appends a `NoteAppended` event with the author as the
event actor and recomputes `notes` through the shared formatter, so `append_issue_note` and `close --note` both emit the
new kind.

In `crates/sase_core/src/bead/jsonl.rs`, `read_event_stream_file`: when a line fails to deserialize, re-parse it as a
generic value and, if its `operation` is not a kind this build knows, return an error that names the unknown kind, the
stream file, and `just install` as the remedy. A stale binary must fail with an instruction, not with a serde message.

Tests in `events.rs` and `crates/sase_core/tests/bead_event_parity.rs`: a `note_appended` reduces to the same text the
legacy path produced; two appends compose; merging two branches that each appended a different entry keeps both, in
timestamp order, with no text lost; merging two branches carrying a byte-identical append collapses to one entry; a
stream mixing legacy `issue_updated{notes}` and `note_appended` reduces correctly; an unknown operation produces the
actionable error.

**Done when:** concurrent note appends survive a three-way merge with no text lost, legacy streams reduce
byte-identically to before, and an unknown event kind reports how to fix it.

### Phase `cli-close` — an honest `sase bead close` command

**Depends on:** `core-close-verified`.

`src/sase/main/parser_bead.py`: change `--resolution`'s default from `"done"` to `None` so an unspecified resolution
reaches the core as absent. Keep the `-R` short alias, the choices, and update the help text to say that the default is
recorded as `done` on a real close and is not compared on an already-closed bead. Keep the option list alphabetical.

`src/sase/bead/cli_crud.py`, `handle_bead_close`: render one row per affected bead from the new outcome fields, instead
of printing `✓ Closed:` for everything in `issues`:

```text
✓ Closed          sase-ba.5 — Retention policy docs
↳ Closed          sase-ba   — Artifact retention (delegated parent)
· Already closed  sase-b8.6 — Lane-based plan and bead agent associations (2026-07-30T16:06:06Z · done)
+ Noted           sase-b8.6 — Lane-based plan and bead agent associations
```

Colored per the CLI conventions; `↳` marks closes the caller did not request (delegated parents and forced descendants),
which are invisible today. Exit 0 whenever every requested bead ends closed, whether or not this invocation is what
closed it; exit 1 on conflict, unknown ID, or unclosed descendants, with the core's message.

`src/sase/bead/mutation_commit.py`: derive the close message from what happened rather than from the request. Add a
close-specific builder used by `handle_bead_close`: at least one real close → `chore(beads): close <closed ids>`; zero
closes with at least one note → `chore(beads): note <id>`; nothing → no commit at all (`mutation_changed` is already
false, so no message is needed). Keep `mutation_commit_message` intact for the other operations.

`src/sase/workflows/commit/commit_hooks.py`, `handle_beads`: the bare `sase bead close <bead_id>` on every commit is now
conflict-free because it passes no resolution — assert that with a test rather than changing the call. Replace the
unconditional `Closing bead {bead_id}...` status line with one that reports the outcome, so a commit that closed nothing
does not claim it did.

Tests in `tests/test_bead/test_cli_close_note.py`, `test_cli_close_resolution.py`, `test_cli_auto_commit.py` and
`tests/test_bead/test_cli_close_phases.py`:

- re-closing a closed bead prints `Already closed`, exits 0, and creates **no** git commit in the store.
- re-closing with a note prints both `Already closed` and `Noted`, and commits `chore(beads): note <id>`.
- `--resolution canceled` against a bead closed `done` exits 1, prints the remedy, and leaves the store byte-identical.
- a bare re-close (no `--resolution`) is never a conflict, covering the commit hook.
- `--phases` over a partly-closed epic reports each phase honestly in one invocation.
- delegated-parent closes render with `↳` and are not double-counted in the commit message.

**Done when:** `sase bead close` never claims to have closed a bead it did not close, a repeat close leaves no commit
behind, and a disagreeing close is refused.

### Phase `doctor-projection` — projection drift detection and repair

**Depends on:** `core-close-interval`.

In `crates/sase_core/src/bead/read.rs`, `doctor_impl` — the reduced issues are already computed there:

- Compare the reduction against the on-disk `issues.jsonl` and report drift per row, e.g.
  `WARNING: issues.jsonl is 313 row(s) stale versus the canonical event streams; run 'sase bead doctor --fix-projection'`.
  This is a permanent integrity check: it catches this epic's repair backlog today and any future bad merge or partial
  write afterwards.
- Census redundant closes while replaying:
  `NOTE: 450 redundant close event(s) across 312 bead(s); 2 in the last 7 days`. The recent-window number is the
  research item's success test ("the duplicate-close count stops growing") expressed as a command instead of an analysis
  script.
- Expose the drift set through a wire struct the Python layer can render and act on: for each row, the ID, the changed
  field names, and the current and reduced values.

In `src/sase/main/parser_bead.py`, add to `sase bead doctor`, keeping options alphabetical and each long option given a
short alias:

- `-P, --fix-projection` — preview and, after confirmation, rewrite `issues.jsonl` from the canonical streams.
- `-y, --yes` — skip the confirmation prompt; required for non-interactive use and mirrors the existing design-ref
  repair prompt, which returns false when stdin is not a TTY.

In `src/sase/bead/cli_admin.py`, `handle_bead_doctor`: render the drift preview grouped by field, then **refuse** to
apply if the diff is outside the expected shape — any row appearing or disappearing, any `status` change, any changed
field other than `closed_at`, `close_reason` or `updated_at`, or any `closed_at` moving later. Print exactly which row
tripped the refusal. On confirmation, re-check the drift under the store mutation lock (the design-ref repair's
recheck-after-preview pattern), write through the existing export path, and commit
`chore(beads): reproject bead state from canonical events`.

Tests in `tests/test_bead/test_cli_doctor.py` and the core's read tests: a store with a duplicate close reports drift
and the redundant census; `--fix-projection --yes` repairs it and reports clean afterwards; a second run is a no-op with
no commit; a hand-corrupted `issues.jsonl` whose row set differs is refused, not silently rewritten; a store with no
drift prints nothing new.

**Done when:** `sase bead doctor` names projection drift and redundant closes, and `--fix-projection` repairs drift of
the expected shape while refusing anything else.

### Phase `history-notes` — redundant closes and restores in history

**Depends on:** `core-close-interval`.

`sase bead history` replays through the same `apply_event`, so a redundant close now produces an entry with an empty
change list. In `src/sase/bead/cli_history.py`, render that case explicitly rather than as a blank row — for example
`issue_closed (redundant — already closed <first close timestamp>)` in compact and full output, and a boolean on the
JSON entry so a chop can count them. Reopen entries will now show `closed_at → null`; confirm the full-format renderer
prints that cleanly.

Add `-y, --yes` to `sase bead history`, keeping options alphabetical, so `--lost-notes --restore` can be confirmed
non-interactively. `_confirm_lost_note_restore` returns false whenever stdin is not a TTY, which makes the restore
unrunnable from any agent or gate today. `--yes` without `--restore` is an error, mirroring the existing `--restore`
requires `--lost-notes` check.

Tests in `tests/test_bead/test_cli_history.py`: a duplicate-close timeline labels the second close redundant in all
three formats; a reopen shows the `closed_at` clear; `--restore --yes` restores without a TTY; `--yes` alone is
rejected.

**Done when:** a duplicate close is visible as redundant in every history format and lost-note restoration can run under
an approval gate.

### Phase `floor-docs` — core floor bump, docs, and full check

**Depends on:** `core-note-append`, `cli-close`, `doctor-projection`, `history-notes`.

- Raise the `sase-core-rs` window in `pyproject.toml` (currently `>=0.14.0,<0.15.0`) to the release-plz-published
  version containing all three core phases. This is what guarantees an ordinary install can never read a store
  containing `note_appended` with a binary that does not understand it. If that version is not published yet, this phase
  waits on it.
- `docs/beads.md`: document the close idempotency contract (what a repeat close does, what a conflicting one does, the
  exit codes), the closed-interval rule and that reopening clears `closed_at`, the `note_appended` event and why note
  appends now converge, and `sase bead doctor --fix-projection` as the way any projection drift is repaired.
- `src/sase/xprompts/skills/sase_beads.md`: state that closing an already-closed bead is safe and free, that
  `--resolution` on an already-closed bead is refused rather than applied, and that `sase bead note` is the way to add
  evidence to a bead that is already closed. Regenerate per the generated-skills workflow — commit the template first,
  then `sase skill init --force` from a clean, merged tree; never hand-edit the deployed provider copies.
- Run `just install`, then `just check`. Read the Symvision long-term memory before touching any Symvision failure the
  new wire fields or CLI helpers produce.

**Done when:** `just check` passes, the floor bump is in place, and the documented contract matches the shipped
behavior.

### Phase `repair` — repair the live store under an approval gate

**Depends on:** `floor-docs`.

This phase mutates the live `beads` sidecar. Open it with `sase repo open beads -r "<reason>"` and work only through the
printed path.

1. `just install` first, so the corrected reducer is the one that runs.
2. Record the before-state: `sase bead doctor` output and the sidecar's `HEAD`.
3. `sase bead doctor --fix-projection` and read the preview. Expect ~313 changed rows, zero `status` changes, changes
   confined to `closed_at`/`close_reason`/`updated_at`, and every `closed_at` moving earlier or to null. The command
   refuses anything else; if it refuses, **stop and report** rather than overriding — a refusal means the store diverged
   in a way this epic did not anticipate.
4. Confirm and let it commit `chore(beads): reproject bead state from canonical events`. Verify afterwards that
   `sase bead doctor` reports no drift and that the redundant-close census still shows the same historical count with a
   recent-window count that is not growing.
5. Preview lost notes with `sase bead history --lost-notes` and report the revision and bead counts (~301 beads at the
   time of the research). Present that count to the user through the `/sase_gate` skill for explicit approval — this
   appends text to hundreds of beads and is the user's call, not the agent's. Only on approval, run
   `sase bead history --lost-notes --restore --yes`, then re-run `--lost-notes` to confirm the set is empty.
6. If approval is declined, land steps 1–4 and say plainly that the restore was deferred.

**Done when:** the live store's `closed_at` values match the canonical event log, `sase bead doctor` is clean of
projection drift, and the lost-note restore has either run or been explicitly deferred by the user.

## Verification

After the epic lands:

1. `sase bead close <closed-id>` prints `Already closed`, exits 0, and `git -C <beads sidecar> status` is clean — no
   event, no commit, no changed byte. Repeat it five times and confirm the sidecar `HEAD` never moves.
2. `sase bead close <closed-id> --note "second look"` appends the note, still reports `Already closed`, and commits
   `chore(beads): note <id>` — not `close`.
3. `sase bead close <done-closed-id> --resolution canceled` exits 1, names the recorded resolution and close time,
   suggests `sase bead open`, and leaves the store unchanged.
4. `sase bead open <id>` then `sase bead close <id>` records the _new_ close time, proving the reopen path is intact,
   and `sase bead history <id> --format full` shows `closed_at → null` on the reopen.
5. In two workspace clones of the sidecar, close the same open bead from each without syncing in between, then let the
   managed sync integrate. The stream carries both `issue_closed` events; `sase bead show` reports the **earlier** close
   time; `sase bead history` labels the second one redundant.
6. Same two-clone exercise with `sase bead note` on each side: both entries appear in the merged notes, in timestamp
   order, with no text lost. Before this epic, one of them disappears.
7. `sase bead doctor` reports no projection drift, and its redundant-close recent-window count stays at zero across a
   week of normal epic traffic — the research item's success test.
8. `sase bead stats` cycle times computed off `closed_at` no longer include the 16-hour outlier, and the 36 beads whose
   close reasons were erased read their real reasons again in `sase bead show`.
