---
tier: epic
title: Expose bead history and make bead completion truthful
goal: 'A closed bead is trustworthy evidence and a bead''s working history is readable:
  every note revision the event streams already hold is reachable from the CLI, notes
  accumulate instead of being overwritten, the ~509 beads whose notes lost earlier
  revisions can be repaired, closing a parent no longer silently manufactures completion
  for descendants nobody worked, every closure records a done/canceled/superseded
  resolution, and reopening a bead reopens the ancestors whose completion claim it
  invalidates.

  '
phases:
- id: history
  title: Read the event streams as a per-bead timeline
  depends_on: []
  size: medium
  description: 'history: add a Rust replay API that turns a bead''s event stream into
    an ordered list of field-level changes, expose it through the PyO3 binding, and
    ship `sase bead history` as its first reader.

    '
- id: resolution
  title: Record and display how a bead was resolved
  depends_on: []
  size: medium
  description: 'resolution: add a stored done/canceled/superseded resolution to the
    bead record end to end — wire, close event, SQLite mirror, Python model, and the
    close CLI — and surface it with the close reason, which no human-readable surface
    prints today.

    '
- id: note
  title: Append to bead notes instead of replacing them
  depends_on:
  - history
  size: small
  description: 'note: add an atomic append-a-note mutation in the Rust core and a
    `sase bead note` verb, so recording progress no longer destroys what an earlier
    writer left behind.

    '
- id: invariant
  title: Stop closing descendants nobody worked
  depends_on:
  - resolution
  size: medium
  description: 'invariant: make the parent-close cascade explicit rather than silent
    — reject a close whose descendants were never closed on their own, require force
    plus a reason and a non-done resolution to override, record which descendants
    were swept, and reopen closed ancestors whenever a bead leaves the closed state.

    '
- id: recover
  title: Find and repair notes lost to overwrite
  depends_on:
  - history
  - note
  size: medium
  description: 'recover: detect beads whose current notes dropped text an earlier
    revision held, report them, and offer a confirmation-gated repair that re-appends
    the dropped revisions with provenance.

    '
- id: integrate
  title: Land the new surfaces as one coherent contract
  depends_on:
  - invariant
  - recover
  size: small
  description: 'integrate: reconcile the two halves into one documented contract —
    bead docs, the generated `sase_beads` skill, the land and phase xprompts, the
    published core version window — and produce end-to-end acceptance evidence against
    the live store.

    '
create_time: 2026-07-27 12:34:17
status: wip
bead_id: sase-a1
---

- **PROMPT:** [202607/prompts/bead_history_truthful_close.md](prompts/bead_history_truthful_close.md)
- **BEAD:** [sase-a1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-a1/README.md)

# Plan: Expose bead history and make bead completion truthful

## Context

This epic implements recommendation 7 of the 2026-07-25 beads-leverage research (findings F8 and F9): _"Expose history,
and make completion truthful."_ Its first listed item — stopping the `COMMIT: <sha>` note overwrite — **already
shipped**: `_post_commit_bead_amend` is now `_amend_bead_changes` in
`src/sase/vcs_provider/plugins/_git_commit_dispatch.py`, and `grep -rn "COMMIT: " --include=*.py src/` returns nothing.
Everything else in that recommendation is still open, and that fix stopped the bleeding without restoring anything.

### Evidence from the live store

Measured from a numbered workspace against `sase/repos/plans/beads/issues.jsonl` (2,120 beads) and
`sase/repos/plans/beads/events/streams/*.jsonl` (367 files, 8,835 events). Re-measure before acting; the store moves.

**History is recorded and unreachable.**

| Measure                                                     |       Count |
| ----------------------------------------------------------- | ----------: |
| Beads with a non-empty `notes` value                        |       1,631 |
| …whose notes still begin with `COMMIT`                      | 1,451 (89%) |
| Beads with at least one note revision in the event streams  |       1,631 |
| Beads with more than one note revision                      |         512 |
| **Beads whose current notes dropped an earlier revision**   |     **509** |
| Approximate dropped characters still present in the streams |    ~222,000 |

There is no `sase bead history`, no appending `sase bead note`, and no CLI surface of any kind over the event streams.
Roughly 222 KB of agent-written verification summaries and blocked-state handoffs sits in tracked git files that nothing
can read.

**Completion is manufactured, not merely unverified.** The research framed F8 as "closed beads whose notes say to keep
them open". The mechanism is sharper than that. `close_issues` in `crates/sase_core/src/bead/mutation.rs` closes _every_
descendant of a plan bead before closing the plan itself, silently and with no record. Consequently:

- **Zero** closed beads currently have a non-closed descendant. The invariant already holds — because the cascade
  fabricates it, not because the work was done.
- **28 child beads were closed in the same instant as their parent**, meaning no agent ever closed them on their own.
  All eight phases of `sase-3i` are in that set.
- `sase-5t` is closed while its own notes read _"Keep this epic open until the recovery changes have a durable SASE
  commit and ChangeSpec/PR reachable from origin/master."_ Its phase `sase-5t.5` carries the same instruction and is
  also closed. `sase-31.6` is closed while its notes read _"Parent sase-31 remains open"_; `sase-31` is closed.
- 86 of 2,117 closures (4.1%) carry a `close_reason`, and **no human-readable surface prints it**. Neither
  `render_issue_detail` in `src/sase/bead/cli_detail.py` nor `handle_show` in `crates/sase_core/src/bead/cli.rs` emits a
  close-reason line; the value appears only in `--format json`. The one field that already records honest closure is
  invisible.

### The invariant this epic establishes

> **A closed bead has no descendant that was not itself closed deliberately.**

Today the left side is enforced by sweeping the right side under the rug. This epic keeps the invariant and makes the
sweep explicit and attributable:

- **Closing** a bead whose descendants are not already closed fails, naming them, with no writes.
- `--force` performs today's cascade, but demands a reason and a non-`done` resolution, stamps every swept descendant
  with that resolution and a reason naming the forcing parent, and records the swept IDs on the parent's close event.
- **Reopening** a bead reopens its closed ancestors, because a closed parent above reopened work is the same lie in the
  other direction.

`--force --resolution done` is rejected. That single rule is where the truthfulness lives: you may still close an
unfinished tree, but you cannot call it done.

### Design decisions

- **Resolution is a stored enum, not parsed prose.** `resolution: done | canceled | superseded`, unset when never
  recorded. Normal closes default to `done`; forced closes must state a non-`done` value. `close_reason` stays free text
  and keeps its meaning. This is the typed form of the `close --reason` practice whose adoption went 0 → 86.
- **Historical closures are not backfilled.** Stamping `done` on 2,117 past closures would be exactly the fabrication
  this epic exists to stop. They stay unset, and every surface renders that as unrecorded rather than as done.
- **Notes stay a single text field, appended to.** No notes table, no per-note records. `sase bead note` reads, appends,
  and writes under the existing bead mutation lock, and each append lands as an ordinary `issue_updated` event, so the
  streams keep the full revision chain for free. `sase bead update --notes` keeps replace semantics: it is an explicit
  user field edit, and the bug was never the CLI.
- **History computes in Rust, renders in Python.** A web app, the ACE TUI, and the CLI must agree on what a bead's
  history is, so the replay belongs in the core. Rendering follows `list`/`show`, which already render in Python.
  Neither `history` nor `note` is added to `execute_bead_cli`; both are rare interactive commands, and keeping them off
  the fast path avoids maintaining a second renderer and a second golden-test surface.
- **Recovery is opt-in and reviewed, never automatic.** The repair follows the established
  `sase bead doctor --fix-design-refs` shape: preview, confirm, then repair. A silent rewrite of 509 beads' notes is not
  reviewable.
- **The 28 historically cascade-closed beads are not repaired.** Nothing in the record distinguishes "swept" from
  "finished, then closed together", so inventing that distinction now would repeat the original error. The `integrate`
  phase reports the count; it does not mutate them.

### Boundary and constraints

Bead storage, mutation, and event replay are core backend behavior: the CLI, the ACE TUI, the mobile bridge, and any
future frontend must agree on what a bead's history is and on when a close is legal. That logic belongs in the
`sase-core` linked repo — `crates/sase_core/src/bead/` and the PyO3 surface in `crates/sase_core_py/src/lib.rs`. Python
keeps argparse, rendering, confirmation prompts, agent-identity resolution, and auto-commit.

Open `sase-core` with the `/sase_repo` skill and use the path it prints; do not clone or web-fetch it. Wire, event, and
binding changes land there first, in the same phase as the Python callers that need them, since `just install` builds
`sase_core_rs` from the local checkout. The published `sase-core-rs` window in `pyproject.toml` is bumped once, in
`integrate`.

New and changed CLI options follow `sase/memory/cli_rules.md`: alphabetically sorted, a short alias for every public
long option, help output that scans cleanly.

Non-goals:

- No `sase bead stats --flow` and no flow metrics (research recommendation 11). This epic gives that work its data
  source; it does not build it.
- No `doctor` checks beyond what `integrate` needs (research recommendation 4), no `ready` filters, no `--json` on
  `blocked`/`ready`/`stats`, no `dep rm`, no ACE open-bead tree.
- No rewrite of the generated `sase_beads` skill (research recommendation 2). `integrate` documents the new verbs and
  the new close contract in the existing template; it does not restructure it.
- **No edits to `sase/memory/*.md`, `AGENTS.md`, or the generated provider instruction shims.** Those require explicit
  user permission in the acting agent's own conversation, which this plan does not confer.
- No change to the `notes` storage shape, to `update --notes` semantics, or to `rm` cascade behavior.

Every phase runs `just install` before `just check`, because workspaces are ephemeral and dependencies drift.

## Read the event streams as a per-bead timeline

Add `crates/sase_core/src/bead/history.rs`, re-exported from `bead/mod.rs`, owning one operation: replay the event store
and return the ordered, field-level change history for one issue.

Read the streams with the existing `jsonl::read_event_store`, and merge them with the same deterministic k-way ordering
`events::reduce_event_streams` uses — reuse it rather than reimplementing the comparator, so history order and
projection order can never disagree. Replay events in that order while maintaining the reduced issue map; for each event
whose `issue_id` matches the target, snapshot the issue before and after applying it and emit one entry.

The entry wire record carries `event_id`, `timestamp`, `actor`, `operation`, and a list of `{field, from, to}` changes,
with its own `BEAD_HISTORY_WIRE_SCHEMA_VERSION` constant following the convention in `crates/sase_core/src/bead/`. Emit
changes for every field that actually differs, so a no-op update produces an entry with an empty change list rather than
a fabricated one. `issue_created` emits the creating snapshot as `from`-less changes for its non-default fields;
`issue_closed`, `issue_opened`, `ready_marked`, `ready_unmarked`, `dependency_added`, and `epic_work_preclaimed` emit
their status and flag effects so the timeline is complete rather than update-only. `issue_removed` ends the timeline.

Cover it with Rust unit tests: a single-update timeline, a multi-revision `notes` chain reporting each `from`/`to` pair,
a close-then-reopen-then-close sequence, an issue whose events live in a parent's stream (phases share the root plan's
stream), a merged stream whose appended events predate earlier entries, and an unknown issue ID producing a not-found
error rather than an empty list.

Expose it as `bead_history` through `crates/sase_core_py/src/lib.rs` following the existing `bead_show` binding pattern,
and register it in the module init.

In this repo, add `src/sase/bead/cli_history.py` with the renderer and `handle_bead_history`, wire it through
`src/sase/bead/cli_basic.py` and `src/sase/bead/cli.py` the way the other handlers are, and register the parser in
`src/sase/main/parser_bead.py` between `dep` and `init` so the subcommand list stays sorted. Reads resolve through the
same `cli_common` read view the other query handlers use, so a workspace with several readable stores behaves
consistently.

```
sase bead history [<id>]
  -F, --field FIELD                  Restrict to events changing FIELD (repeatable)
  -f, --format {compact,full,json}   Output format (default: compact)
  -n, --limit N                      Newest N entries; 0 means unlimited
```

`compact` prints one line per entry — timestamp, actor, operation, changed field names. `full` prints each change with
its `from` and `to` values, which is the surface that makes an overwritten note readable again. `json` prints one
envelope: `{"issue_id": ..., "schema_version": ..., "entries": [...]}`. The optional positional and the two remaining
flags (`--lost-notes`, `--restore`) arrive in `recover`; author the positional as `nargs="?"` now, erroring cleanly when
no ID is given, so that phase adds behavior rather than re-shaping the parser.

Python tests in `tests/test_bead/test_cli_history.py`: each format against a fixture store, `--field notes` isolating a
revision chain, `--limit` taking the newest entries, an unknown ID exiting non-zero with a clear message, and the JSON
envelope's shape. Document the command in `docs/beads.md` under CLI Commands, and extend the Event Log section to say
that the streams are now readable rather than merely canonical.

## Record and display how a bead was resolved

Add `resolution` to the bead record end to end, following the `tier` column precedent exactly.

In `sase-core`: add `BeadResolutionWire { Done, Canceled, Superseded }` to `bead/wire.rs` and an
`Option<BeadResolutionWire>` field on `IssueWire`, skipped when serializing so existing fixtures and streams stay
byte-identical. `IssueWire::validate` rejects a resolution on a non-closed issue. Add the field to
`BeadIssueUpdateEventFieldsWire` and to the `IssueClosed` payload in `bead/events.rs`, and apply it in
`apply_update_event_fields` and the `IssueClosed` arm of `apply_event`. Add `resolution` to `BEAD_SQLITE_SCHEMA` and a
`needs_resolution_migration` / `resolution_migration_sql` pair in `bead/schema.rs` mirroring `needs_tier_migration`; the
migration adds the column with its `CHECK` constraint and backfills nothing.

`close_issues` gains a `resolution` parameter, defaulting to `Done` when the caller passes nothing, and stamps it
alongside `closed_at` and `close_reason` in `MutableStore::close_one`. The upward delegated-parent close in
`close_one_and_delegated_parent` — which fires only when every child of that phase is already closed — records `Done`
with its existing reason. `update_issue` accepts `resolution` in `BeadUpdateFieldsWire` and clears it whenever a status
change moves an issue out of `closed`.

In this repo: add `Resolution` to `src/sase/bead/model.py` and the field to `Issue` with its validation, then thread it
through `src/sase/core/bead_wire.py`, `src/sase/bead/jsonl.py`, `src/sase/core/bead_mutation_facade.py`,
`src/sase/bead/project.py`, and `src/sase/bead/db.py` — the last needs a `_migrate_add_resolution` alongside
`_migrate_add_tier`, using the Rust migration policy the way `_migrate_relax_size_check` does.

Add the flag to `sase bead close`, in both `parse_close_args` in `crates/sase_core/src/bead/cli.rs` (close is on the
fast path) and `src/sase/main/parser_bead.py`:

```
  -R, --resolution {canceled,done,superseded}   How this bead was resolved (default: done)
```

Then fix the display gap. For a closed bead, both `render_issue_detail` in `src/sase/bead/cli_detail.py` and
`handle_show` in `crates/sase_core/src/bead/cli.rs` print a `RESOLUTION` block with the resolution, the close reason,
and `closed_at`, rendering an unset resolution as `(unrecorded)` so the 2,117 historical closures read honestly rather
than silently. Update the golden tests in both places together so the parity implementations do not drift. Add
`resolution` to `issue_to_wire_dict`, to `_render_list_json`, and to the ACE Plans detail rows in
`src/sase/ace/tui/widgets/artifacts/plans_detail.py`, which already renders `Close reason` and is the natural home for
the value beside it.

Tests: Rust round-trips through create/update/close events and the projection; a resolution on a non-closed issue
rejected; the SQLite migration adding the column to a pre-existing table; Python `close --resolution` end to end; `show`
rendering both the recorded and the unrecorded case; and a JSON-envelope assertion in `tests/test_bead/test_cli_show.py`
and `tests/test_bead/test_cli_list.py`. Document the field in `docs/beads.md` under Data Model and `sase bead close`.

## Append to bead notes instead of replacing them

Add `append_issue_note(beads_dir, issue_id, entry, author, now)` to `crates/sase_core/src/bead/mutation.rs`. It runs
inside `with_bead_mutation_lock`, reads the current `notes`, appends, writes, and emits one `issue_updated` event
carrying the new full value — the same shape `update_issue` emits, so the streams, the projection, and `bead history`
all keep working unchanged. Doing the read-modify-write inside the lock is the point: the equivalent Python
read-then-update would be exactly the check-then-act race upstream `beads` documents as its most common bug class.

The appended entry is `[<timestamp> · <author>] <text>`, separated from existing content by a blank line, or written
alone when `notes` is empty. Reject an empty or whitespace-only entry. Use the store's configured `owner` as the event
actor when the caller supplies no author.

Expose it as `bead_append_note` in `crates/sase_core_py/src/lib.rs`, add `append_note` to
`src/sase/core/bead_mutation_facade.py` and `BeadProject.append_note` in `src/sase/bead/project.py`, and add
`handle_bead_note` to `src/sase/bead/cli_crud.py` beside `handle_bead_update`, using the same
`bead_store_mutation(auto_commit_bead_store)` wrapper and a `chore(beads): note <id>` commit message.

```
sase bead note <id> <text>
  -a, --author NAME   Author recorded on the entry (default: current agent, else the store owner)
```

Python resolves the default author with `discover_agent_identity()` from `src/sase/agent/identity.py`, falling back to
the store owner when it returns nothing, so a note written by a phase agent is attributed to that agent while a note
written by hand is attributed to the store owner. Register the parser in `src/sase/main/parser_bead.py` between `list`
and `onboard`. `note` is deliberately absent from `_MUTATING_VERBS` in `src/sase/main/bead_fast_path.py`, because it is
not routed through `execute_bead_cli` at all; confirm the slow path handles it and that the store still auto-commits.

Tests: appending to an empty and to a non-empty `notes`; two appends producing two entries and two stream events; an
empty entry rejected without writing; the author defaulting from `SASE_AGENT_NAME` and from the store owner; and an
assertion that `sase bead history <id> --field notes --format full` shows both revisions, which is the cross-phase
contract this phase and `history` share. Document the verb in `docs/beads.md`, next to `sase bead update`, stating
plainly that `note` appends and `update --notes` replaces.

## Stop closing descendants nobody worked

Change `close_issues` in `crates/sase_core/src/bead/mutation.rs` from silent cascade to explicit sweep.

Before any write, collect for each requested ID every descendant whose status is not `closed`. If that set is non-empty
and `force` is false, return a validation error naming the requested bead and up to a bounded number of unresolved
descendants with their statuses, plus the count of any remainder, and write nothing. Because every requested ID is
checked before the first mutation, a batch close either fully applies or leaves the store untouched, matching how
`remove_issues` already validates.

With `force` true, perform today's cascade, and require both a reason and a resolution other than `Done`. Each swept
descendant is closed with the forcing resolution and a close reason naming the forcing parent; the parent's
`IssueClosed` event payload gains a `forced_descendant_ids` list recording exactly what was swept, so the archive
answers "what was not done" without prose parsing. Extend `BeadEventPayloadWire::IssueClosed` for that field with a
`#[serde(default)]` so existing streams keep parsing.

Apply the same guard to `update_issue` when a status change lands on `closed`: reject it with the same error rather than
performing a cascade, since a field edit is not a close. This closes the bypass that `sase bead update --status` and the
ACE Plans status cycle in `src/sase/ace/tui/actions/artifacts_plans.py` would otherwise offer; that action surfaces the
error as a TUI notification.

Make the reverse direction symmetric. Whenever a mutation moves an issue out of `closed` — `open_issue`, or
`update_issue` with a non-closed status — walk up `parent_id` and reopen every closed ancestor, emitting one
`issue_opened` event each and clearing their resolution. Extract that walk as one helper both call sites use, so no path
can reopen a child under a closed parent. Report the reopened ancestors in the mutation outcome so the CLI can print
them.

`handle_bead_open` in `src/sase/bead/cli_crud.py` currently calls `project.update(id, status="open")`, so it emits an
`issue_updated` event rather than `issue_opened`. Route it through the `open_issue` mutation instead, matching the Rust
fast path, then print the reopened ancestors.

Add the flag in both `parse_close_args` in `crates/sase_core/src/bead/cli.rs` and `src/sase/main/parser_bead.py`:

```
  -f, --force   Close despite descendants that were never closed on their own;
                requires --reason and a --resolution other than 'done'
```

Rewrite the affected Rust unit tests in `mutation.rs` rather than deleting them:
`close_plan_cascades_open_children_before_parent` and `close_plan_cascades_through_nested_child_epics` become the
forced-close cases and additionally assert the recorded `forced_descendant_ids`;
`closing_child_epic_closes_completed_parent_phase`, `open_sibling_delegated_work_keeps_parent_phase_open`, and
`nested_delegation_closes_only_phase_parents` must stay green unchanged, because the upward delegated-parent close only
fires when every child is already closed and is therefore invariant-respecting by construction. Add: an unforced close
with one open descendant failing and writing nothing; a batch where the second ID fails leaving the store untouched;
`--force --resolution done` rejected; `--force` without a reason rejected; and reopening a grandchild reopening both
ancestors and clearing their resolutions.

Then update the prompts, whose current text assumes the old behavior. In `src/sase/default_config.yml`:

- `bd/land_epic` step 3 keeps `sase bead close {{ bead_id }}` as the normal path and gains a sentence for the new
  failure: if the close is rejected, the named phases were never completed — finish or reopen them, or record the
  outcome deliberately with `--force --reason ... --resolution canceled|superseded`. Never force merely to make the
  command succeed.
- `bd/work_phase_bead` and `bd/next` gain nothing new about creating beads (that is a separate recommendation), but both
  state that closing a phase bead is unaffected, so a phase agent's existing instruction stays literally correct.

Update `docs/beads.md`'s `sase bead close` section, which today documents the silent cascade as intended behavior, and
its Status Lifecycle section, which today says status "can transition freely between any values".

## Find and repair notes lost to overwrite

Detect and repair the 509 beads whose current notes dropped text an earlier revision held.

Add the detection to `crates/sase_core/src/bead/history.rs`, reusing the replay from the `history` phase: for each
issue, walk its ordered `notes` revisions and collect every earlier revision whose non-empty trimmed text is absent from
the current value. Substring containment is the test — it is conservative in the right direction, treating a revision
that was appended to as retained and only flagging text that genuinely vanished. Return one wire record per affected
issue carrying the issue ID, the current value, and the dropped revisions with their timestamps and actors. Expose it as
`bead_lost_notes` beside `bead_history`.

Extend the `sase bead history` parser with the remaining two flags:

```
  -l, --lost-notes   Report beads whose current notes dropped an earlier revision
  -R, --restore      With --lost-notes, re-append the dropped revisions (asks first)
```

`--lost-notes` with no positional ID scans the whole store; with an ID it reports that bead alone. `--restore` previews
every planned append, prompts once for confirmation, and then re-appends each dropped revision through the `append_note`
mutation from the `note` phase, marking provenance so a restored entry is never mistaken for a fresh one:

```
[<original timestamp> · <original actor>] (restored <date>) <original text>
```

Restoring through the same append path means each repair lands as an ordinary event, so the repair is itself visible in
`bead history` and reversible by reading. Follow the confirmation shape `sase bead doctor --fix-design-refs` already
uses in `src/sase/bead/cli_admin.py`, including its non-interactive behavior, so the two repairs feel like one tool.
`--restore` without `--lost-notes` is a usage error.

Restoration is idempotent: a revision already present in the current notes is not reported as dropped, so a second
`--restore` run finds nothing. Assert that directly.

Tests: an overwrite detected; an append-only chain producing no finding; a revision restored and a re-run finding
nothing; a declined confirmation writing nothing; `--restore` without `--lost-notes` erroring; and a fixture with
several affected beads reporting them in a stable order. Document both flags in `docs/beads.md`.

Do not run the repair against the live store here — that is `integrate`'s call, once every surface that displays the
result exists.

## Land the new surfaces as one coherent contract

The fan-in phase. Reconcile the two halves and produce the acceptance evidence that shows the epic delivered.

**Contract.** Read `docs/beads.md` end to end and make the new surfaces read as one story rather than five additions:
the event streams are the archive, `history` reads it, `note` writes to it without destroying it, `resolution` and the
close guard make the archive's completion claims true, and reopening keeps them true. Reconcile the Status Lifecycle,
Event Log, `close`, `open`, `update`, `history`, and `note` sections against each other; the CLI Commands list stays
alphabetical.

**Skill.** Update `src/sase/xprompts/skills/sase_beads.md` to document `history`, `note`, `close --force` and
`--resolution`, and the reopen-ancestors behavior — the skill is what agents actually read, and none of this exists for
them until it lands there. Follow `sase/memory/generated_skills.md`: edit the source template, run
`sase skill init --force`, then `chezmoi apply`; never hand-edit a generated provider copy. This is a targeted addition,
not the full skill rewrite recommendation 2 calls for.

**Version window.** Bump `sase-core-rs` in `pyproject.toml` to the window that includes the release carrying this epic's
core changes, following the `build(deps): require sase-core-rs ...` precedent. Dev installs build from the local
checkout and ignore the window, so verify the constraint against the actual published release rather than assuming.

**Acceptance evidence.** Run and record, against the live store:

1. `sase bead history sase-5t.5 --field notes --format full` — the multi-revision handoff note is readable.
2. `sase bead history --lost-notes` — re-measure the affected-bead count against the 509 recorded here and explain any
   difference, then run `--restore` and re-run to show it converges to nothing.
3. `sase bead show sase-5t` — the `RESOLUTION` block renders, with the unrecorded historical case rendering as such.
4. On a scratch store: closing a plan with an open phase is rejected and writes nothing; `--force --resolution done` is
   rejected; `--force --resolution canceled --reason ...` succeeds and records the swept IDs; reopening the phase
   reopens the plan and clears its resolution.
5. Re-run the two structural queries from the Context section: closed beads with a non-closed descendant (expect 0, now
   because the invariant is enforced rather than faked) and child beads closed in the same instant as their parent
   (expect the historical 28, unchanged — this epic does not rewrite history).

Report each result in the phase's completion message, including anything that did not match. Do not repair the 28
historical cascade closures.

## Verification

Every phase, in an ephemeral workspace:

```bash
just install
just check
```

Phases touching `sase-core` also run its test suite from the linked checkout (`just rust-test` from this repo, or
`cargo test` in the core repo), since the Python suite exercises the binding but not the Rust unit tests.

Expect mechanical rebase conflicts between the two wave-one phases: `history` and `resolution` both append to
`crates/sase_core/src/bead/mod.rs`, `crates/sase_core_py/src/lib.rs`, `src/sase/main/parser_bead.py`, and
`docs/beads.md`. They are additions to sorted lists; resolve by keeping both and re-sorting.
