---
tier: epic
title: Repair the snooze close path and finish landing epic sase-gn
goal:
  Closing a snoozed task bead succeeds, drops the snooze record, and leaves the store
  readable, including stores already bricked by the defect; the bead event log can no
  longer be persisted ahead of the state it derives; the dead wake-due-snooze selector
  is gone from the Rust core; the two rival snooze parsers are one; `sase bead list`
  shows snoozed beads by default like `sase bead search` already does; and epic sase-gn
  is closed with its plan file marked done.
phases:
  - id: snooze-close-core
    title: Stop a close from bricking a snoozed bead's store
    depends_on: []
    size: medium
    description:
      "snooze-close-core: clear the snooze record on every transition out of snoozed in
      both the mutation and the reducer, validate derived issues before the event log is
      written, and delete the orphaned wake-due-snooze selector."
  - id: snooze-close-regression
    title: Non-mocked close regression coverage and the core pin bump
    depends_on:
      - snooze-close-core
    size: medium
    description:
      "snooze-close-regression: pin the sase-core release carrying the fix and cover
      both close call sites against a real bead store, closing the mocking gap that let
      the corruption ship."
  - id: snooze-parser-merge
    title: One snooze wake-time parser, not two
    depends_on: []
    size: small
    description:
      "snooze-parser-merge: collapse snooze_time.py and snooze_duration.py into a single
      parser so the CLI and the gate/ACE surfaces cannot accept different wake-time
      forms."
  - id: snooze-list-default
    title: Snoozed beads stay visible in the default listing
    depends_on: []
    size: small
    description:
      "snooze-list-default: add snoozed to `sase bead list`'s default status set and
      correct the help text and docs that still enumerate the pre-snooze status list."
  - id: snooze-gn-land
    title: Close epic sase-gn
    depends_on:
      - snooze-close-core
      - snooze-close-regression
      - snooze-parser-merge
      - snooze-list-default
    size: small
    description:
      "snooze-gn-land: close epic sase-gn with a close note covering the whole landing,
      run symvision, and mark the sase-gn plan file done."
proposed_by: bbugyi200.athena.sase-gn.land
parent_bead: sase-gn
status: done
bead_id: sase-gn.10
create_time: 2026-09-09 19:51:34
---

- **PROMPT:**
  [prompts/202608/snooze_close_corruption.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/snooze_close_corruption.md)
- **PARENT:**
  [202608/bead_snooze_and_notification_indicator.md](https://github.com/sase-org/sase--plans/blob/main/202608/bead_snooze_and_notification_indicator.md)
- **BEAD:**
  [sase-gn.10](https://github.com/sase-org/sase--beads/blob/main/pages/sase-gn/sase-gn.10.md)

# Repair the snooze close path and finish landing epic sase-gn

## Context

Epic sase-gn (`@plan:202608/bead_snooze_and_notification_indicator.md`) shipped the
snoozed task-bead status, the `BeadSnooze` wake gate, and the per-tab notification
indicator across nine phases. Its land review confirmed every phase delivered what its
notes claim, and confirmed that the one unrelated commit that landed mid-epic — the bead
SQLite layer split — already absorbed the epic's snooze columns, codecs, migration, and
row hydration, so no integration work is outstanding.

What is outstanding is a severe defect the epic's own verification phase found, plus
three smaller inconsistencies the epic introduced. They are epic work, not follow-ups,
so sase-gn cannot close until they are done.

### The defect: closing a snoozed task bead permanently bricks the bead store

Reproduced directly against the Rust binding, in a scratch store, three times:

```python
m.bead_update(beads, iid, {"status": "ready"})
m.bead_snooze(beads, iid, "2099-01-01T00:00:00Z", None, "r", "tester@example.com", None)
m.bead_close(beads, [iid], "why", "canceled")
# ValueError: validation: Only snoozed issues can carry snooze metadata
m.bead_list(beads)
# ValueError: validation: Only snoozed issues can carry snooze metadata   <- forever, in any process
```

Two independent faults combine:

1. **The close does not clear the snooze record.** `MutableStore::close_one` sets
   `status`, `closed_at`, `close_reason`, `resolution`, and `updated_at`, and leaves
   `snooze` populated. The `IssueClosed` reducer arm does the same.
   `IssueWire::validate` rejects any non-snoozed issue that still carries snooze
   metadata, so the derived record is invalid on both paths. `apply_update_fields` and
   `apply_update_event_fields` already do this correctly for an ordinary status update,
   with a comment explaining exactly why — the close path simply never got the same
   treatment.

2. **The event log is persisted before the state it derives is known to be valid.**
   `MutableStore::save` calls `write_event_store` first and `write_issues_jsonl` — which
   validates — last. So the `issue_closed` event lands durably on disk, validation then
   fails, and `issues.jsonl` keeps the pre-close snapshot. Every later load replays the
   poison event and re-derives the same invalid record, so the store never recovers.
   Confirmed on disk: the stream file carries the `issue_closed` event while
   `issues.jsonl` still reads `snoozed`.

Blast radius. Both callers go straight to `project.close()` with no snooze pre-clear:

- `close_bead_snooze` in `src/sase/bead/snooze_gate.py` — the **primary, default**
  option on the `BeadSnooze` wake gate this epic shipped. Every default "Close" on a
  woken snoozed bead bricks that project's bead store.
- `handle_bead_close` in `src/sase/bead/cli_crud.py` — plain `sase bead close <id>`,
  pre-existing, whenever the target happens to be snoozed.

Both run inside `bead_store_mutation(auto_commit_bead_store, ...)`, so a corrupted event
stream sits in the beads sidecar working tree where a later commit could publish it.

Why the tests missed it: `tests/test_bead/test_snooze_gate.py` builds the whole project
as a `MagicMock` via `_mutation_double()`, so `project.close()` is never exercised
against a real store on any close, ready, or re-snooze test.

Scope of the same class elsewhere, from a probe of every mutation reachable from
`snoozed`:

| Transition                    | Today                                                                                    |
| ----------------------------- | ---------------------------------------------------------------------------------------- |
| `bead_close`                  | **bricks the store**                                                                     |
| `bead_open` (reopen)          | raises the same validation error; store survives, because it validates before persisting |
| `bead_claim_for_agent_launch` | raises the same validation error; store survives                                         |
| `bead_claim_for_agent_wait`   | correctly refuses (`changed=false`)                                                      |
| `bead_update --status ...`    | correct — clears the record                                                              |
| `bead_plus_one`               | correct — stays snoozed below target                                                     |
| `bead_remove`                 | correct                                                                                  |

### Three inconsistencies the epic introduced

- **A dead selector.** `wake_due_task_snoozes` and its `bead_wake_due_snoozes` binding
  have no caller anywhere. Design decision D2 made a snoozed bead's gate born snoozed
  and resurfaced by notification snooze expiry, so the reconciler never polls for due
  wakes. sase-gn.7 flagged it, sase-gn.8 deleted the Python facade wrapper, and the Rust
  half was left for the landing to decide. It should go: keeping an unreachable
  mutation-shaped entry point in the core invites a future caller to use the wrong wake
  mechanism.
- **Two rival wake-time parsers.** `src/sase/bead/snooze_time.py` (CLI only) and
  `src/sase/bead/snooze_duration.py` (gate + ACE modal) both parse the same
  `30m`/`2h`/`3d` vocabulary, and **both docstrings claim to be the single shared
  parser** — `snooze_time.py` explicitly names "the gate options that carry a duration
  in their feedback field, and the ACE modal's custom field" as its callers, which is
  false. They already disagree: a naive ISO-8601 timestamp gets the configured timezone
  attached by one and is rejected as unrecognized by the other, and their error text
  differs.
- **`sase bead list` hides snoozed beads.** The design reasoned explicitly that a
  snoozed task must not vanish like a black hole, and that reasoning shipped for
  `DEFAULT_BEAD_FILTER_QUERY` and for `sase bead search`, which passes no status filter
  and so includes snoozed. The separate hardcoded default in `handle_bead_list` was not
  updated.

## Phases

### snooze-close-core

Fix the corruption and remove the dead selector in the sibling Rust core repo, in one
change, so a single release carries both. Open that repo with `/sase_repo` and use only
the path it prints.

Correctness:

- `MutableStore::close_one` (`crates/sase_core/src/bead/mutation.rs`) clears `snooze`
  when it sets `Closed`.
- The `BeadEventPayloadWire::IssueClosed` arm of the reducer
  (`crates/sase_core/src/bead/events.rs`) clears `snooze` too. This is what makes an
  already-bricked store heal itself: `issues.jsonl` still holds the valid pre-close
  record, so once replay derives `closed` without a snooze record, the next load
  succeeds and the close finally takes effect.
- Give the two transitions that raise the same cryptic validation error the same
  treatment, in both the mutation and the matching reducer arm, rather than leaving a
  snoozed bead un-reopenable and un-launchable: the reopen path and
  `claim_for_agent_launch` (`IssueOpened` and `EpicWorkPreclaimed` on the reducer side).
  Follow the precedent and the comment already in `apply_update_fields`: moving off
  `snoozed` drops the record, exactly as moving off `closed` archives the close fields.

Durability ordering:

- `MutableStore::save` must not write the event store before the derived issue set is
  known to be valid. Validate the issues first — or write `issues.jsonl` first — so the
  next mismatch of this class is a clean rejection with nothing persisted, instead of a
  permanent brick. This guard is the reason the defect was unrecoverable rather than
  merely wrong, so do not skip it in favor of the one-line snooze fix.

Deletion:

- Remove `wake_due_task_snoozes` and its crate test from
  `crates/sase_core/src/bead/mutation.rs`, its re-exports from
  `crates/sase_core/src/bead/mod.rs` and `crates/sase_core/src/lib.rs`, and, in
  `crates/sase_core_py/src/lib.rs`, the `py_bead_wake_due_snoozes` function, its import,
  its `wrap_pyfunction!` registration, its module doc-comment inventory line, and both
  of its entries in the binding-name inventory test.

Tests, all in the crate:

- Closing a snoozed task succeeds, the closed record carries no snooze, and reloading
  the store returns it.
- The same store replayed purely from its event stream derives the identical record —
  the reducer and the mutation must not drift, which is the invariant that broke.
- A store already carrying a `task_snoozed` event followed by an `issue_closed` event
  loads clean. Build the fixture from raw event records, not by calling the fixed close,
  so it stays a real recovery test.
- A deliberately invalid derived state leaves the event streams on disk untouched (the
  ordering guard).
- Reopen and agent-launch claim from `snoozed` behave as decided above.

Gates: `cargo fmt --all --check`,
`cargo clippy --workspace --all-targets -- -D warnings`, `cargo test --workspace`. Then
commit and push to the core repo's master so release-plz cuts the next release, and
record the released version in a bead note for the next phase.

### snooze-close-regression

Depends on `snooze-close-core`.

- Bump the `sase-core-rs` requirement in `pyproject.toml` from `>=0.18.4,<0.19.0` to
  require the release carrying the fix. This also discharges two follow-ups sase-gn.1
  and sase-gn.2 left open: a published wheel resolving to 0.18.4 lacks
  `classify_notification_tabs` entirely and would raise from `require_rust_binding`, and
  the aggregated `NotificationTabWire.color` stays `None` there so sender-declared tab
  colors never reach the indicator. Dev checkouts build the binding from the local core
  checkout and were never affected, which is why this went unnoticed.
- Add regression coverage that would have caught the corruption, against a **real** bead
  store with no project mock:
  - `close_bead_snooze` on a genuinely snoozed bead — the gate's primary action — closes
    it and leaves the store readable afterwards.
  - `sase bead close <id>` on a snoozed bead does the same.
  - Both assert the reload, not just the call. The call raising is not the failure mode
    that matters; the store dying afterwards is.
- `tests/test_bead/test_snooze_gate.py`'s `_mutation_double()` may stay for the tests
  that only assert wiring, but the close, ready, and re-snooze side effects need at
  least one real-store test each, since mocking all three is what hid this.
- Verify with `just check-full`: this phase changes a dependency pin and the epic's
  broadening set is already in play.

### snooze-parser-merge

No dependencies.

Collapse `src/sase/bead/snooze_time.py` and `src/sase/bead/snooze_duration.py` into one
module with one error type and one accepted-forms string. Keep `parse_snooze_request`'s
`"<duration> [+<N>]"` handling, since the gate and ACE modal need the `+N` target and
the CLI takes it as a separate flag.

Decide the one behavior for a naive ISO-8601 timestamp — attach the configured timezone
as `snooze_time` does, or reject it as `snooze_duration` does — and state the reason in
the module docstring. Attaching the zone is the friendlier answer and matches what
someone typing `2026-08-09T09:00` means, but either is defensible; what is not
defensible is the current split.

Callers to update: `src/sase/bead/cli_crud.py`, `src/sase/bead/snooze_gate.py`,
`src/sase/bead/task_gate.py` (two function-local imports),
`src/sase/ace/tui/modals/bead_snooze_modal.py`, and the tests in
`tests/test_bead/test_cli_snooze.py` and `tests/test_bead/test_snooze_gate.py`. Make
sure the surviving module's docstring describes its real callers; both current
docstrings claim a reach they do not have.

Preserve the seconds-resolution wake time sase-gn.8 fixed — sub-second precision is
noise against the reconciliation tick, and dropping it would resurrect that bug.

### snooze-list-default

No dependencies.

- Add `Status.SNOOZED` to the default status list in `handle_bead_list`
  (`src/sase/bead/cli_query.py`), between `Status.READY` and `Status.IN_PROGRESS` to
  match the status ordering used elsewhere.
- Update the command help in `src/sase/main/parser_bead_queries.py`, which still reads
  "List open, claimed, ready, and in-progress beads by default."
- Update `docs/beads.md`: the `sase bead list` example comment, and the descendant-close
  sentence that enumerates "open, claimed, ready, or in progress" — a snoozed descendant
  blocks a close too, since the guard rejects any descendant that is not closed.
- Cover the new default with a test, and check no existing test asserts the four-status
  default.

### snooze-gn-land

Depends on every other phase. **Read this phase's description in full before acting: it
closes an epic that is not this plan's parent.**

`sase-gn` is a separate, already-finished epic — all nine of its phase beads are closed,
and this plan exists only because its landing uncovered the work above. Closing it is
therefore not the forbidden "close your own parent epic" action, and the
descendant-close guard does not apply: `sase-gn` has no unclosed descendants, and this
plan's phases are not its descendants. Do not close this plan's own epic bead; its land
agent does that.

1. Confirm every phase above is closed and `just check-full` is green on the combined
   tree.
2. Close sase-gn:

   ```bash
   sase bead close sase-gn --note "<close note>"
   ```

   The note should record: all nine phases verified against source and commits; the only
   mid-epic commit (the bead SQLite layer split) already integrated the epic's snooze
   work, so no integration remained; the store-corrupting close, the
   event-before-validation ordering, and the dead wake-due selector were fixed as epic
   work by this plan, along with the duplicate parsers and the `sase bead list` default;
   and the disposition of every non-epic follow-up (below).

3. Run `just symvision` and remove any stale sase-gn epic-symbol whitelist entries and
   unused code it reports. There are currently no `--epic-symbol` entries in the
   Justfile at all — sase-gn.8 removed the last of them — so a clean run is the expected
   result, not a reason to skip the step.
4. Set `status: done` in the frontmatter of
   `@plan:202608/bead_snooze_and_notification_indicator.md`. Resolve the plans repo with
   `sase repo path plans`.

Non-epic follow-ups, already dispositioned by the sase-gn land agent, for the close
note:

- The four ACE TUI parallel-lane flakes reported across sase-gn.1, .2, .6, and .8 were
  corroborated onto the existing task `sase-ct` as a +1 rather than filed again.
- `sase-go` — `test_contract_set_serial_runtime_stays_within_budget` flakes despite its
  calibration probes.
- `sase-gp` — the mobile core knows neither bead gate kind, so both render as
  "Unsupported action"; carries the open question of whether bead gates should be
  mobile-priority.
- `sase-gq` — Telegram `/bead` is read-only, so snoozing from Telegram works only
  through a gate button.
- `sase-gr` — `sase/memory/sase_beads.md`'s status list is missing `snoozed`; needs the
  owner's explicit approval because it is a memory file.
- Two sase-gn.6 notes were design records, not follow-ups, and need no action:
  `GateAdapter.validate_selection` exists because an option command cannot see feedback
  text, so it is the only place a mistyped re-snooze duration can be rejected while
  leaving the gate pending; and the shared duration parser layers days onto
  `parse_duration`, which has no day unit.

## Verification

- The reproduction at the top of this plan must close cleanly and leave the store
  readable.
- A store bricked by the old code must load again without hand repair.
- `just check-full` green, and the crate's own `cargo fmt --check` /
  `clippy -D warnings` / `cargo test --workspace` green in the core repo.
- `sase bead list` shows a snoozed bead with no flags.
- Exactly one module parses snooze wake times.
