---
tier: tale
title: Every task-bead snooze leaves a durable note naming its wake conditions
goal:
  Snoozing a task bead — from the CLI, the ACE Beads pane, a TaskTriage gate, or a BeadSnooze
  re-snooze — atomically appends one attributed note recording the wake time, the deferral length,
  any +1 target, and the reason, so the "why and until when" survives the wake that erases the
  snooze record.
proposed_by: bbugyi200.athena.un.w0
create_time: 2026-08-07 10:40:07
status: done
---

- **PROMPT:**
  [prompts/202608/snooze_note.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/snooze_note.md)
- **AGENTS:**
  - [bbugyi200.athena.un.w0](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.un.w0.md)
- **COMMITS:**
  - [bfdc411](https://github.com/sase-org/sase-core/commit/bfdc4112a42b724957c081a500ef5eacf1cdb131)
    — feat(bead): append a snooze note recording wake conditions

# Plan: Always note a task bead's snooze conditions

## Outcome

Every successful `snooze` mutation on a task bead appends exactly one note to that bead, in the same
store mutation that sets `status: snoozed`. The note reads like the confirmation line the CLI
already prints:

```text
[2026-08-07T13:21:54Z · bryanbugyi34@gmail.com] Snoozed until 2026-08-10T09:21:53-04:00 (in 3d). Also wakes at 2 more +1s (25 total). Reason: waiting on the upstream fix
```

Four frontends get this from one change, because all four call the same core mutation:

| Surface                                        | Entry point                                                                   |
| ---------------------------------------------- | ----------------------------------------------------------------------------- |
| `sase bead snooze <id> -u 3d [-p N] [-r TEXT]` | `src/sase/bead/cli_crud.py::handle_bead_snooze`                               |
| ACE Beads pane snooze modal                    | `src/sase/ace/tui/actions/_artifacts_beads_mutations.py::_submit_bead_snooze` |
| `TaskTriage` gate → **Snooze**                 | `src/sase/bead/_task_gate_actions.py::snooze_task_triage`                     |
| `BeadSnooze` wake gate → **Snooze again**      | `src/sase/bead/snooze_gate.py::resnooze_bead_snooze`                          |

Nothing else changes: the CLI confirmation line, the snooze record, the `snoozed` status, the commit
message (`chore(beads): snooze <id>`), the gate lifecycle, and the exit codes are all untouched.

This is a `tale`. The change is one core mutation, one note renderer, its Rust tests, a dependency
floor bump, Python tests, and a docs paragraph — a bounded unit that a single agent can implement
and verify coherently. Splitting it would leave a window where the note's wording is defined in one
repo and asserted in another.

## Problem

The snooze record is the only place a deferral's "why" and "until when" live, and it is destroyed
the moment the bead wakes. `clear_snooze_record` (`crates/sase_core/src/bead/events.rs:1435`) is
called from every path out of `snoozed`:

| Path out of `snoozed`           | Where                                      | Leaves a note?             |
| ------------------------------- | ------------------------------------------ | -------------------------- |
| +1 target reached               | `mutation.rs:365` (`plus_one`)             | yes — `plus_one_wake_note` |
| `snooze --cancel` / pane cancel | `mutation.rs:547` (`cancel_task_snooze`)   | no                         |
| wake gate → **Ready**           | Python `snooze_gate.py::ready_bead_snooze` | yes — a second mutation    |
| wake gate → **Close**           | `close_issues`                             | close reason only          |
| claimed for launch              | `mutation.rs:788`                          | no                         |
| `update --status`, `open`       | `mutation.rs:1056`, `1082`, `1128`         | no                         |

So a task deferred on Monday "until the upstream fix lands" and woken on Thursday shows, on Friday,
no trace that it was ever snoozed, for how long, by whom, or why. `sase bead show` renders the
`SNOOZE` block only while the record exists; afterwards the deferral is recoverable only by
replaying `sase bead history`, which nobody does during triage.

The gap is visible while the bead is still snoozed, too. The `BeadSnooze` wake gate's preview
(`render_bead_snooze_preview`, fed `notes=issue.notes` by
`src/sase/scripts/sase_chop_bead_task_triage.py`) is exactly the surface where the reviewer decides
Close / Ready / Snooze again — and today it shows the bead's pre-existing notes with nothing about
the deferral being reviewed beyond the raw wake conditions in the payload.

Core already treats "a snooze lifecycle event deserves a readable note" as a domain rule: the +1
wake path writes `"Reopened by +1 threshold: reached {target} +1s while snoozed until {until}."`
through `append_note_to_store`, inside the same mutation
(`crates/sase_core/src/bead/mutation.rs:386-398`). This plan applies that same rule to the event
that starts the deferral.

## Where the note is written

In the Rust core, inside `snooze_task` — not at the four Python call sites.

- **Atomicity.** `append_note_to_store` mutates the loaded store before `store.save()`, so the
  status change, the `task_snoozed` event, the `note_appended` event, and the note text land in one
  write under one `with_bead_mutation_lock`. A Python-side second mutation could half-apply, and
  would bump `updated_at` twice and emit a second bead-store commit.
- **No caller can forget.** A fifth snooze caller (a future gateway action, a web frontend, a Rust
  CLI fast path) inherits the note for free. `sase/memory/rust_core_backend_boundary` is explicit
  that behavior another frontend would have to match belongs in core.
- **Consistency.** The wake half of this rule already lives in core. Splitting the pair across repos
  would make the snooze note a Python convention and the wake note a core guarantee.

No new binding is added. `bead_snooze`'s signature, the `BeadSnoozeWire` record, `snooze_codec.py`,
the SQLite mirror, and `issues.jsonl`'s schema are all unchanged — only the bead's `notes` field and
its event stream gain content.

## The note text

Rendered by a new `snooze_note` helper beside `plus_one_wake_note` in
`crates/sase_core/src/bead/mutation.rs`. Grammar, in clause order:

1. **Wake clause** — always.
   - First snooze: `Snoozed until <until> (in <length>).`
   - Re-snooze (the bead was already `snoozed`):
     `Re-snoozed until <until> (in <length>), replacing the wake time <old_until>.`
2. **+1 clause** — only when `plus_ones` was requested. `Also wakes at <n> more +1<s>.`, or
   `Also wakes at <n> more +1<s> (<target> total).` when the bead already carries +1s
   (`plus_one_baseline > 0`). Singular `+1` for `n == 1`, `+1s` otherwise.
3. **Reason clause** — only when the trimmed reason is non-empty. `Reason: <reason>` with no added
   trailing punctuation, so `"Deferred from triage."` and `"waiting on the upstream fix"` both read
   correctly.

`<until>` and `<old_until>` are the stored strings, trimmed and echoed verbatim — the same treatment
`plus_one_wake_note` gives `snooze.until`. Core has no configured timezone and must not invent one.

Worked examples:

```text
Snoozed until 2026-08-10T09:21:53-04:00 (in 3d). Reason: waiting on the upstream fix
Snoozed until 2026-08-08T09:00:00-04:00 (in 21h). Also wakes at 2 more +1s.
Re-snooze: Re-snoozed until 2026-08-17T09:00:00-04:00 (in 7d), replacing the wake time 2026-08-10T09:21:53-04:00. Also wakes at 2 more +1s (25 total). Reason: Deferred from triage.
Snoozed until 2026-08-10T09:21:53-04:00 (in 3d).
```

The last form is the floor: a bare `sase bead snooze <id> -u 3d` with no reason and no +1 target
still records what was deferred and until when. That is the "always informative" requirement — the
reason enriches the note, it is not what makes it exist.

### The deferral length

`<length>` is `until - snoozed_at`, computed from the two timestamps `snooze_task` already parses
(`parse_snooze_timestamp`), rendered with a small `deferral_length_label` helper. It is a snapshot
of the length that was chosen, not a countdown, so it never goes stale.

The ladder mirrors `_snooze_remaining_label` in `src/sase/bead/snooze_presentation.py`, so the note
reads in the same vocabulary as the confirmation line and the `◈ in 3d` chip:

| Delta     | Rendering    |
| --------- | ------------ |
| `< 60s`   | `<secs>s`    |
| `< 60m`   | `<minutes>m` |
| `< 24h`   | `<hours>h`   |
| `< 30d`   | `<days>d`    |
| `< 365d`  | `<months>mo` |
| otherwise | `<years>y`   |

Each bucket rounds to nearest, half away from zero, using integer arithmetic (`(n + d / 2) / d`) —
no floats. Two deliberate departures from the Python ladder, both because this is a length and not a
remaining time: the sub-minute bucket renders `<secs>s` rather than `now` (`snooze_task` rejects a
past wake time, so the delta is at least one second), and there is no `due now` case.

Do not add a parity test between this helper and the Python label. They render different quantities
— total deferral versus time remaining from the reader's "now" — and the Python side never
re-derives the note text. Rounding is cosmetic here: the exact wake time is in the same sentence.

### Whitespace

Collapse runs of whitespace in the reason to single spaces before embedding it. The stored
`SnoozeRecord.reason` keeps the raw text; only the note's copy is flattened. A multi-line reason
would otherwise inject a blank line into `notes`, whose entries are separated by `\n\n`
(`appended_note_text`, `crates/sase_core/src/bead/events.rs:1439`), and split one note into two on
every surface that renders them.

## sase-core changes

Open the checkout with `sase repo open sase-core -r "<reason>"` and work only in the path it prints
(the workspace-relative `sase/repos/linked/sase-core`).

`crates/sase_core/src/bead/mutation.rs`:

1. Add `fn snooze_note(...) -> String` and `fn deferral_length_label(seconds: i64) -> String` next
   to `plus_one_wake_note` (~line 412), with the same doc-comment density as their neighbors.
2. In `snooze_task`, after the existing `store.append_issue_event(... TaskSnoozed ...)` and before
   `store.save()`, call `append_note_to_store(&mut store, index, &note, &actor, &timestamp)?`. Order
   matters: the `task_snoozed` event must precede the `note_appended` event so a replay renders the
   note after the status change.
3. **Re-capture the issue after the note append.** `snooze_task` currently clones the issue into
   `let issue = issue.clone();` _before_ appending its event; a clone taken there has empty notes
   and the stale `updated_at`, and it is what `result.issue` returns to every Python caller —
   including the ACE pane row that repaints straight from it. Follow the `plus_one` path, which
   re-reads `store.issues[index].clone()` after `append_note_to_store`.
4. The re-snooze branch needs the previous record: `current` (the pre-mutation clone taken at the
   top of the closure) already carries `current.snooze`, which is `Some(_)` exactly when this is a
   re-snooze.
5. `actor` is the note author, matching the `snoozed_by` attribution; `timestamp` is the note's
   timestamp, matching `snoozed_at`.

The mutation stays deliberately non-idempotent — re-snoozing to identical conditions appends a fresh
event today, and it will append a fresh note too. That is the documented behavior of `snooze_task`
("appends a fresh event rather than editing the old one so the history stays readable"), not a
regression to fix here.

### Rust tests

In `mutation.rs`'s `mod tests`:

- Extend `snooze_task_records_wake_conditions_and_replays_from_events` to assert the note text.
  `snoozed_task_fixture` snoozes at `2026-01-01T00:02:00Z`, so the
  `("2026-01-04T09:00:00-05:00", Some(2))` case is a 3d13h58m deferral that renders `in 4d` — assert
  the value the ladder actually produces, do not reverse-engineer the ladder to make it read `3d`.
  The fixture's reason is `" needs the upstream fix first "`, which exercises the trim.
- The whole-issue equality assertions in that test (`reduces_to_store`, `import_issues_from_jsonl`)
  are the replay coverage: they prove the note survives the event round trip and reaches the
  generated projection. They need no change beyond the fixture now carrying notes.
- Add a case asserting a snooze with no reason and no `+1` target still writes a one-sentence note.
- Add a re-snooze case: two `snooze_task` calls leave two notes, the second naming the replaced wake
  time, with `\n\n` between them.
- Add direct unit tests for `deferral_length_label` at each bucket boundary (59s, 60s, 59m, 60m,
  23h, 24h, 29d, 30d, 364d, 365d) and for the half-away-from-zero rounding.
- Check the other `snoozed_task_fixture` consumers still pass —
  `cancel_task_snooze_returns_the_bead_to_ready`,
  `closing_a_snoozed_task_drops_the_record_and_the_store_reloads`,
  `a_store_bricked_by_a_close_over_a_snooze_loads_again`,
  `reopening_and_launch_claiming_a_snoozed_task_drop_the_record`,
  `plus_one_below_the_target_leaves_a_snoozed_bead_snoozed`, and the +1-wake test at line ~3028.
  That last one now runs against a bead whose notes already contain a snooze note; it asserts with
  `contains` rather than equality, so it should stay green — keep it that way rather than pinning
  the whole `notes` string anywhere.

Use a Conventional Commit `feat(bead):` subject so release-plz computes the right version.

## Dependency floor

CI's `build-core` job builds `sase_core_rs` from `sase-org/sase-core` **master**, so this repo's PR
turns green only after the core PR has merged. The published-wheel lane
(`published-core-minimum-smoke`) runs the smoke tools and `tools/check_sase_core_rs_bindings`, which
this change does not affect — no new binding is introduced.

Bump the window anyway. Without it, `pip install sase` can resolve a published core that silently
writes no note, which makes the word "always" in this plan's goal false for anyone not building from
source. This matches the precedent set by
`5b3f3494b build(deps): raise sase-core-rs floor to 0.19.0`, a behavior-only floor bump.

In `pyproject.toml`, raise **both** ends of `"sase-core-rs>=0.19.0,<0.20.0"`. A `feat` on a `0.x`
crate bumps the minor (`0.18.5 → 0.19.0` was exactly this shape), so the release that carries this
change is expected to be `0.20.0` — which the current upper bound excludes. Leaving `<0.20.0` in
place while raising the floor produces an unsatisfiable requirement.

Do not guess the number. Read the version release-plz actually published, then:

```bash
python3 tools/smoke_sase_core_rs_telemetry --print-minimum pyproject.toml   # confirms the parsed floor
```

Refresh `uv.lock` in the same commit (`just install` does it; the released lock was last synced in
`bfdc6ca25`).

## This repo's changes

No source change is required in this repo — the note arrives through the existing
`BeadProject.snooze` → `bead_mutation_facade.snooze` → `bead_snooze` path. The work here is the
floor bump, the tests that pin the contract from the Python side, and the docs.

### Tests

`tests/test_bead/test_cli_snooze.py` (already runs `handle_bead_snooze` against a real store with
both clocks pinned — the right home for all of these):

- `sase bead snooze <id> -u 3d -p 2 -r "waiting on the upstream fix"` leaves exactly one note whose
  text names the wake time, the deferral length, the +1 target, and the reason, attributed to the
  snoozing actor.
- A bare `-u 3d` with no reason and no `-p` still leaves one note naming the wake time and length.
- Re-snoozing an already-snoozed bead appends a second note naming the replaced wake time; the first
  note is still present.
- `--cancel` appends no note. This pins the scope boundary below, so a later wake-side change is a
  deliberate edit to a red test rather than a silent behavior drift.
- A snooze whose reason contains a newline yields a single note (no blank line inside it), while
  `project.show(id).snooze.reason` still holds the raw text.

`tests/test_bead/test_snooze_lifecycle.py`: the existing round-trip test already walks the store,
the `issues.jsonl` projection, and the SQLite mirror. Add the note to the fields it asserts on each
surface, so a codec change cannot drop it.

`tests/test_bead/test_snooze_gate.py`: one case building a `BeadSnooze` gate from a bead snoozed
through the real store, asserting the rendered preview's `## Notes` section contains the snooze
note. This is the payoff path — the wake reviewer sees the conditions and the reason at the moment
they choose Close / Ready / Snooze again.

Re-run `tests/test_bead/test_snooze_close_regression.py` and
`tests/ace/tui/test_artifacts_beads_mutations.py` unchanged; both should stay green. The chop's
`_presentation_fingerprint` already hashes `issue.notes`, and the note lands in the same mutation as
the status change, so the wake gate is created once from post-snooze state — no gate churn, and no
`_PRESENTATION_FORMAT_VERSION` bump (no renderer's output shape changed).

### Docs

`docs/beads.md`, "Snoozing a Task Bead" (~line 312): after the paragraph documenting `-r/--reason`,
state that every snooze appends one attributed note recording the wake time, the deferral length,
any +1 target, and the reason, and that this is what preserves the deferral after a wake clears the
record. Give one example note. In the `sase bead snooze` reference table (~line 1217), extend the
`-r, --reason` row to say the text is embedded in that note.

Nothing in `src/sase/xprompts/skills/sase_beads.md` mentions snoozing, so no generated skill source
changes and no skill regeneration is needed.

Do not edit `sase/memory/*.md`, `AGENTS.md`, or the generated `CLAUDE.md` / `GEMINI.md` /
`OPENCODE.md` / `QWEN.md` shims. `sase/memory/sase_beads.md` documents notes and closing but never
mentions snoozing, so nothing there goes stale — and those files need explicit user permission that
this plan does not carry.

## Verification

1. In the sase-core checkout: `cargo fmt`, `cargo clippy --all-targets`, `cargo test -p sase_core`,
   `cargo test -p sase_core_py`.
2. In this repo: `just install` first — it rebuilds `sase_core_rs` from the linked checkout and the
   workspace may be stale — then `just check`. Run `just check-full` before landing; this change
   touches the bead store's write path.
3. Manual round trip in a scratch bead store:
   - `sase bead snooze <id> -u 3d -r "waiting on the upstream fix"`, then `sase bead show <id>` —
     one Notes entry, `SNOOZE` block unchanged, confirmation line unchanged.
   - `sase bead snooze <id> -u 7d -p 2` on the same bead — a second note naming the replaced wake
     time.
   - `sase bead snooze <id> --cancel`, then `sase bead show <id>` — the `SNOOZE` block is gone and
     both notes remain, which is the whole point.
   - `git log` in the bead store shows one `chore(beads): snooze <id>` commit per snooze, not two.
4. Snooze a bead from the ACE Beads pane and confirm the detail pane shows the new note without a
   manual refresh (this is what step 3 of the core changes protects).

## Landing order

1. `sase-core` PR: `snooze_note`, `deferral_length_label`, the `snooze_task` wiring, the re-captured
   outcome issue, and the Rust tests.
2. Wait for the release-plz release that publishes it; read the published version.
3. This repo's PR: floor and upper-bound bump, `uv.lock`, Python tests, docs.

Steps 1 and 3 can be written and verified locally in one sitting, because `just install` builds
`sase_core_rs` from the linked checkout. Only this repo's CI depends on the core PR having merged.

## Out of scope (deliberate)

- **Notes on the wake side.** A snooze can end six ways (table above) and only two of them write a
  note today. Making the whole lifecycle narrate itself — a note on `cancel_task_snooze`, on a close
  over a snooze, on a launch claim — is a coherent follow-up, but it also requires reconciling the
  Python-side `ready_bead_snooze` note so a wake gate's **Ready** does not produce two notes. That
  is a different change with a different blast radius; file it as a task bead if it is wanted.
- **Making `-r/--reason` required.** Tempting, but it would break `TaskTriage`'s snooze option,
  whose feedback field carries a duration and not a reason, and the ACE modal's deliberately
  optional reason field. The conditions are what make the note informative; the reason enriches it.
- **Changing the CLI confirmation line or the modal.** Output stays byte-identical.
- **Notification snoozing.** `mark_snoozed` and the notification store are a separate subsystem that
  defers a _row_, not a bead. Untouched.
- **Reworking the `+1` wake note's wording.** It is the precedent this plan follows, not its
  subject.
- **Backfilling notes onto beads snoozed before this lands.** The event log has the `task_snoozed`
  records, so a backfill is possible, but rewriting historical notes is a store repair, not a
  feature.
