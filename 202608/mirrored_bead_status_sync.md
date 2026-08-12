---
tier: tale
title: Bug state drives mirrored bead status
goal:
  A mirrored task bead follows its tracker issue's open/closed state — the mirror closes
  it when the issue closes and reopens it when the issue reopens — while an in-progress,
  claimed, or parented bead, and any bead that merely references an issue, keeps today's
  note-only behavior, and every transition stays visible in the chop counters, the CLI
  summary, and the dry run.
size: medium
proposed_by: bbugyi200.athena.sase-k2.4
bead: sase-k2.4
create_time: 2026-08-12 12:55:57
status: wip
---

- **PROMPT:**
  [prompts/202608/mirrored_bead_status_sync.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/mirrored_bead_status_sync.md)
- **PARENT:**
  [202608/external_mirror_refinement.md](https://github.com/sase-org/sase--plans/blob/main/202608/external_mirror_refinement.md)
- **BEAD:**
  [sase-k2.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-k2/sase-k2.4.md)

# Plan: Bug state drives mirrored bead status (epic phase `bug_status`, bead sase-k2.4)

Implements the `bug_status` phase of `plans:202608/external_mirror_refinement.md`. The
phase depends on `filters`, which has landed (`6b139a0d4`), as have `spec_repair`
(`d4139e96e`) and `lane` (`fb33e3c1f`). Nothing here may assume the sibling
`patch_status` or `perf` phases have landed.

## Goal

Reverse the `sase-jd` plan's "upstream closes: note-only" decision for the one
relationship the mirror owns. When a tracker issue that a bead **mirrors** closes, close
that bead; when it reopens, reopen it. Keep the attributed note in every case as the
audit record, and keep today's note-only behavior wherever the mirror does not own the
bead's status: a bead that merely references the issue, a bead an agent is working, a
bead with unclosed descendants, and a bead already in the target state.

## Background measured against the current tree

- `src/sase/external_mirror/issues.py` is the single reconciliation path for both the
  `external_issue_mirror` chop and `sase bead sync-external`. `_plan_notes` (line 412)
  already computes exactly the transition set this phase needs: for every ref covered by
  a local bead it compares `state.upstream_states[ref]` to the listed issue's state, and
  separately notes refs that dropped out of the listing entirely (`"absent"`).
- `_apply` (line 328) already holds the whole pass inside one `bead_store_write_lock` →
  `open_bead_project_for_beads_dir` → `commit_external_issue_mirror` →
  `publish_bead_claim` block, rebuilds a live index under the lock for its
  raced-creation check, and charges work against `budget.max_creations`,
  `budget.max_notes`, and a `work_deadline`. Status changes fit this shape with no new
  locking, committing, or publication machinery.
- `applied_note_refs` is already the "only advance `upstream_states` for work that
  actually happened" mechanism: `run_issue_mirror_for_project` merges it into
  `state.upstream_states` only when `deferred == 0`, so a deferred transition is
  re-planned next pass. Status changes reuse it verbatim.
- `BeadProject.close(ids, resolution=..., note=..., author=...)`
  (`src/sase/bead/_project_mutations.py:327`) closes and appends an attributed note in
  one mutation. `BeadProject.open(id)` (line 359) reopens a bead and every closed
  ancestor and takes no note, so a reopen is two calls.
- `bead_read_facade.list_issues(beads_dir)` and `BeadProject.list_issues()` pass no
  status filter, so both already return closed beads. A bead the mirror closed stays in
  `covered` and can therefore be reopened on a later pass, and `upstream_states`' "prune
  to covered refs" step keeps working unchanged.
- `_build_identity_index` (line 487) deliberately conflates the two relationships the
  Beads pane distinguishes: it indexes a bead's `external_ref` **and** every `bug:`
  entry in its `refs`. `beads_data.py:388-405` is the authority on the distinction —
  `external_ref` is `"mirrored"`, a `bug:` ref is `"referenced"`.
- `close` rejects a bead with any unclosed descendant, canonically in `sase-core`
  (`sase/memory/sase_beads.md`, "Closing"). Re-closing an already-closed bead is a no-op
  but a conflicting resolution or reason is refused, so the mirror must not call `close`
  on a bead that is already closed.
- `src/sase/bead/close_gate_settle.py` exists precisely so a close settles the bead's
  pending `TaskTriage`/`BeadSnooze` gate immediately (`875f67b74`); `sase bead close`
  calls it from `cli_crud.py:607`. Gates carry the ProjectSpec **key** as `project`
  (`sase_chop_bead_task_triage.py:477` passes `record.project_name`), which is exactly
  the `project_key` this reconciler already has.
- `_check_error` in `src/sase/scripts/sase_chop_external_issue_mirror.py:20` hand-copies
  the counter dict that `summary_counters` builds, so any new counter silently makes a
  failed run report a different key set than a successful one. This is the same loose
  end the `lane` phase fixed for the PR mirror by routing both paths through
  `MirrorReport.summary_fields()`.
- `drift` in the Beads pane (`beads_data.py:425-439`) is computed only for `"mirrored"`
  links and is true when local-closed disagrees with remote-closed (or titles differ).
  After this change the drift badge and the `bug:drift` filter token narrow to the
  _unreconciled_ set — guard-skipped beads, title drift, and reference-only links —
  which is more useful than today's "every upstream close forever". No code change is
  needed there; it must be verified, not edited.

## Design decisions

### D1. Only a bead that mirrors the issue may have its status written

The relationship the mirror owns is `external_ref`. A bead that merely carries a
`bug:<project>#<n>` entry in `refs` is a human's cross-reference, and its status is the
human's.

`_build_identity_index` becomes a two-pass build returning `dict[str, _CoveredBead]`,
where `_CoveredBead` is `(bead: Issue, mirrored: bool)`:

1. First pass indexes every bead's normalized `external_ref` (`mirrored=True`).
2. Second pass indexes every normalized `bug:` ref not already present
   (`mirrored=False`).

Coverage is the union of both passes, so creation behavior — including
`test_bead_with_only_bug_ref_is_recognized_as_covering` — is unchanged. The one
deliberate behavior change is which bead wins a ref that two different beads claim: the
mirroring bead now wins over the referencing one, instead of whichever came first in
list order. That is strictly more correct now that the winner decides both the note
target and whether a status may be written.

Keep the call shape `bead_read_facade.list_issues(beads_dir)` with no keyword arguments:
`test_conflict_created_between_plan_and_apply_is_detected_under_lock` distinguishes the
unlocked planning read from the locked rebuild by exactly that.

### D2. A transition needs a previous observed state; first sight only seeds

`_plan_notes` skips a ref whose `upstream_states` entry is `None`. Keep that. The first
pass that covers a ref records the issue's current state and changes nothing, so a bead
freshly created for an already-closed issue is not closed a moment later. Whether the
mirror should create beads for already-closed issues at all is a _creation_ policy
question that `external_mirror.issues.filters.states` already answers; do not smuggle it
into this phase. Record it as a `PROPOSED FOLLOW-UP:` note on sase-k2.4 instead.

### D3. Every observed transition still appends exactly one note

The note is the audit record and it stays unconditional, so the note stream a user reads
today does not develop gaps where the mirror acted. What changes is the sentence:

| Situation                                | Note text                                                                                                          |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| Applied close                            | `... changed state: open -> closed (observed <iso> by external_issue_mirror). Closed this mirrored bead to match.` |
| Applied reopen                           | `... Reopened this mirrored bead to match.`                                                                        |
| Guard-skipped                            | `... This bead's status is unchanged (<reason>); reconcile deliberately.`                                          |
| Referenced-only bead, or absent upstream | today's text, verbatim                                                                                             |

`notes_appended` therefore keeps its current meaning — "transitions recorded this pass"
— and `beads_closed` / `beads_reopened` count the subset that also changed status. A
close reports as `notes=1 closed=1`, which is honest: one note was written and one bead
was closed.

### D4. Guards are one pure function, evaluated twice

```python
def _blocked_reason(bead: Issue, *, action: str, blocked_ids: frozenset[str]) -> str
```

- `close`: `""` unless the bead is already `closed` ("the bead is already closed"), is
  `in_progress` or `claimed` ("an agent is working this bead"), or appears in
  `blocked_ids` ("the bead has unclosed descendants").
- `reopen`: `""` only when the bead is `closed`; otherwise "the bead is already open".

`blocked_ids` comes from a helper that walks each non-closed bead's `parent_id` chain
and marks every ancestor, cycle-guarded with a per-walk `seen` set.

The function runs twice against two different bead lists, which is the point:

- at **plan** time over the unlocked `local_beads` read, so `--dry-run` reports what
  would really happen rather than an optimistic upper bound;
- at **apply** time over `project.list_issues()` inside the lock, which is the
  authoritative evaluation. A bead that became `in_progress` between the two reads is
  correctly demoted to note-only, exactly as the existing `conflicts` check demotes a
  raced creation.

No `--force`. The descendant rule belongs to the bead store, and sweeping a user's
unfinished children because a tracker issue closed is not a mutation a chop may make.

### D5. Close is one mutation, reopen is two; both cost one unit of budget

`close([bead_id], resolution=Resolution.DONE, note=<text>, author="external_issue_mirror")`
writes the close and the note together. `open(bead_id)` takes no note, so a reopen is
`open()` then `append_note(..., author="external_issue_mirror")`. Both charge one unit
against `budget.max_notes` and are gated by `work_deadline`, so a large backlog of
transitions converges over passes instead of one enormous commit — the same shape the
creation cap already has.

`resolution=done` is right for the mirror: the tracker says the work is finished.
`canceled` would assert a judgment the mirror cannot make, and it would also be the
resolution `--force` requires, which D4 forbids.

### D6. A close settles that bead's pending gate immediately

A mirrored bead is created `open`, never `ready`, so it usually has no gate. But a human
can triage one to `ready`, and then upstream closes it. Every other close path settles
the gate; the mirror must too, or a `TaskTriage` gate offers to launch an agent against
a closed bead until `bead_task_triage`'s sweep catches it up to five minutes later.

Call `settle_closed_task_bead_gates(project_key, closed_task_ids, source="chop")` after
the lock is released and the commit and publication have run, matching
`cli_crud.py:607`'s placement. It is already best-effort and swallows its own failures,
so it cannot turn a successful pass into a failed one. Only pass ids of beads that were
actually closed **and** are `IssueType.TASK`.

### D7. No `sase-core` change and no wire bump

The mutations, the descendant rule, and the resolution model all already exist in the
Rust core and are reached through `BeadProject`. What this phase adds is mirror policy —
which relationship may write a status and under what guards — which lives with the
mirror, next to the filter and cursor policy it already owns. `CLAUDE.md`'s litmus test
agrees: another frontend consuming the bead store sees only ordinary closes and reopens.

### D8. The chop's failure path reports the same counters as its success path

Replace `_check_error`'s hand-written dict with
`summary_counters(MirrorReport(project="", display_name=""))`. The values are identical
(all zero) and the key set can never drift again — the same fix the `lane` phase applied
to the PR mirror. Fold the two new counters into the chop's `no_changes` condition too,
so the reason stays honest even if `notes_appended` ever stops covering them.

## Implementation steps

1. **`src/sase/external_mirror/issues.py`** — the whole behavior change.
   - `MirrorReport`: add `beads_closed`, `beads_reopened`, and the dry-run/apply detail
     tuples `closed_refs` and `reopened_refs`, each documented like `created_refs`.
   - `summary_counters`: add `beads_closed` and `beads_reopened`.
   - Add `_CoveredBead`; rewrite `_build_identity_index` per D1.
   - Add `_unclosed_ancestor_ids(beads) -> frozenset[str]` and `_blocked_reason(...)`
     per D4.
   - Replace `_NoteCandidate` with
     `_TransitionCandidate(bead_id, ref, new_upstream_state, action, observation)`,
     where `action` is `"close"`, `"reopen"`, or `"none"` and `observation` is the
     transition sentence without its outcome clause. `_plan_notes` becomes
     `_plan_transitions` and sets `action` to `"none"` for a non-mirrored bead and for
     the absent-upstream case, otherwise `"close"`/`"reopen"` from the new state.
   - Add `_transition_note(candidate, *, reason, applied)` building the final text per
     D3.
   - `_apply` returns an `_ApplyOutcome` dataclass instead of its current six-tuple,
     builds the live `blocked_ids` and an id→bead map once under the lock, applies each
     transition per D4/D5, records `applied_note_refs[ref]` for every applied transition
     (guard-skipped included, since the note was written), and calls D6's gate settle
     after publication. A candidate whose bead vanished between plan and apply is
     skipped without being recorded as applied; it self-heals next pass because the ref
     is no longer covered.
   - `run_issue_mirror_for_project`: build the plan-time bead index once, pass it to the
     planner and to the dry-run transition preview, and thread the new counters and ref
     tuples into every `MirrorReport` it returns.
2. **`src/sase/scripts/sase_chop_external_issue_mirror.py`** — D8.
3. **`src/sase/bead/cli_sync_external.py`** — append `closed=<n>` and `reopened=<n>` to
   the per-project line only when non-zero, matching how `filtered=<n>` behaves, so the
   common line is unchanged. Under `--dry-run`, print `would close <ref>` and
   `would reopen <ref>` lines beside the existing `would create <ref>` lines.
4. **`src/sase/bead/_sync_git.py`** — widen `commit_external_issue_mirror`'s docstring
   from "created or noted" to cover closes and reopens. The commit message itself is
   already generic and does not change.
5. **`src/sase/main/parser_bead_store.py`** — the `sync-external` description at 322-328
   currently promises "Upstream state changes append an attributed note and never close
   a bead"; state the new rule and its guards. Widen `--dry-run`'s help (line 346) from
   "planned creations" to creations and status transitions.
6. **`src/sase/default_config.yml`** — the `external_issue_mirror` chop description
   makes the same "never close a bead" promise at **two** places, lines 742 and 795,
   because the `checks`-lane copy of both mirror chops was never deleted when the `lane`
   phase added the `external_mirror` lane. Update **both** occurrences and leave the
   duplication itself alone; it is `lane`'s regression, recorded as a
   `PROPOSED FOLLOW-UP:` note (see Out of scope). Respect the description contract:
   summary line ≤100 characters, blank line, then the body.
7. **Docs.** `docs/axe.md:327-336` (the `external_mirror` lane's issue-mirror paragraph)
   and `docs/beads.md:576-584` (the "External Issue Mirroring" section) each state the
   note-only rule; rewrite both to describe the transition, the four guards, the
   mirrored-vs-referenced distinction, and the fact that guard-skipped links are what
   the `drift` badge now means. Update `docs/beads.md:1481`'s `--dry-run` row to match
   step 5.

## Tests

`tests/test_external_mirror_issues.py` carries most of this. Rewrite
`test_upstream_close_appends_exactly_one_note_across_three_passes`, which asserts
`bead.status == Status.OPEN` after an upstream close and is exactly the behavior this
phase reverses; keep its three-pass shape, since "the transition is applied once and
never re-applied" is still the property under test.

New coverage:

- Upstream `open -> closed` closes the mirrored bead, appends one note naming the
  transition, and reports `beads_closed=1 notes_appended=1`; a third pass over the same
  listing does nothing.
- Upstream `closed -> open` reopens it and reports `beads_reopened=1`.
- A close followed by a reopen followed by a close leaves one bead with three notes and
  a closed status — round-trip, not just one hop.
- A bead that only carries a `bug:` ref (no `external_ref`) gets the note and keeps its
  status, with `beads_closed == 0`.
- Guards, one test each, all asserting note appended + status unchanged + the transition
  not retried on the next pass: `in_progress`, `claimed`, an unclosed child, and a bead
  a human already closed.
- A bead whose status changes to `in_progress` between the planning read and the locked
  apply is demoted to note-only — monkeypatch the module's
  `bead_read_facade.list_issues` the way the existing conflict test does.
- `--dry-run` reports `closed_refs`/`reopened_refs`, writes no event, and leaves
  `upstream_states` untouched.
- Transitions are charged against `max_notes`: with a budget of one and two pending
  transitions, one applies, `deferred == 1`, `checkpoint_advanced is False`, and the
  next pass applies the other.
- The disappeared-issue path still appends its stale note and never closes.

`tests/test_axe_chop_external_issue_mirror.py`: a close is reported in the chop's
counters, and a `check_error` result carries the same counter key set as an `ok` result
(assert the two key sets are equal rather than spelling the keys out twice).

`tests/test_bead_sync_external_cli.py`: `closed=`/`reopened=` appear when non-zero and
are absent when zero; `--dry-run` prints the `would close` line.

## Verification

`just install`, then `just check`. Then `just check-full`, because this change touches
`src/sase/default_config.yml`, which is in the broadening set.

Then confirm this phase's slice of the epic's verification list on this machine, without
mutating the real bead store — the epic's live check ("closing a tracker issue that has
a mirrored bead closes that bead on the next pass") requires closing a real GitHub issue
and is the land agent's to run. What this phase can verify locally:

```bash
sase bead sync-external --project sase --dry-run
```

must still run clean, report the same `issues=`/`created=` numbers as before the change,
and plan no transitions on a tree whose `upstream_states` are current. Also confirm
`sase bead sync-external -h` and `sase doctor -C axe` still describe the chop correctly.

## Out of scope

Left to their own phases even though they touch neighboring code: refreshing an adopted
external Patch from its PR (`patch_status`) and the per-pass ProjectSpec index re-read
(`perf`). The `drift` badge and the `bug:drift` filter token are verified, not changed.

Record as `PROPOSED FOLLOW-UP:` notes on bead sase-k2.4 rather than filing beads:

- `src/sase/default_config.yml` still lists `external_issue_mirror` and
  `external_pr_mirror` in the `checks` lane _and_ in the new `external_mirror` lane, so
  both mirror chops are scheduled twice. `docs/axe.md:241-249` documents the `checks`
  lane with three chops, so the checks-lane copies are a leftover from the `lane`
  phase's land and should be deleted.
- A bead created for an issue that was already closed upstream is created `open` and
  never reconciled, because a transition needs a previously observed state (D2).
