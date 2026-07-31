---
tier: tale
title: Accept multiple bead IDs in sase bead update as one atomic bulk mutation
goal:
  "`sase bead update` accepts one or more bead IDs and applies the same field changes to every named bead in a single
  all-or-nothing store mutation that produces exactly one bead-store commit and one push."
proposed_by: bbugyi200.athena.qm
create_time: 2026-07-31 14:43:58
status: done
---

- **PROMPT:** [202607/prompts/bead_update_bulk_ids.md](prompts/bead_update_bulk_ids.md)

# Plan: Accept multiple bead IDs in `sase bead update`

## Outcome

`sase bead update <id> [<id2> ...] [flags]` applies the same field changes to every named bead. The batch is
all-or-nothing: every ID is resolved and every resulting issue is validated before anything is written, so an unknown
ID, an ambiguous shorthand, or an invalid field value leaves the store byte-identical. A successful batch writes one
`chore(beads): update <id1> <id2> ...` commit naming exactly the beads that actually changed, and the existing
deferred-push plumbing pushes that commit once.

Single-ID invocations keep their current syntax, output line, commit message, and exit codes. Nothing about
`sase bead update <id> -s ready` changes.

This is a `tale` because the Rust core mutation, the Rust CLI fast path, the Python slow path, the argparse surface, the
docs, and the tests form one bounded change that a single implementation agent can complete and verify coherently.
Splitting it across phase agents would create a window where the Rust fast path and the Python slow path disagree about
whether `sase bead update a1 a2 -s ready` is valid syntax — and which of them silently writes a partial batch. The
`sase bead` shorthand-ID work (`sase/repos/plans/202607/bead_id_shorthand.md`) is the direct precedent for this shape
and scope.

## Why both backends must change

`sase bead update` has two live implementations and users hit both today:

- `src/sase/main/entry.py` dispatches through `src/sase/main/bead_fast_path.py`, which calls the `bead_cli_execute` Rust
  binding. `crates/sase_core/src/bead/cli.rs::handle_update` treats `args[0]` as the single ID and hands the rest to
  `parse_update_fields`.
- When the Rust fast path returns `defer()`, argparse runs `handle_bead_update` in `src/sase/bead/cli_crud.py`, which
  calls `BeadProject.update` → `sase.core.bead_mutation_facade.update` → the `bead_update` binding.

The Python slow path is not vestigial: `parse_update_fields` has no case for `-z/--size` or the `-r` short alias of
`--tier`, so `sase bead update <id> -z medium` already defers to Python on every invocation. Implementing bulk update in
only one backend would make the accepted syntax depend on which flags the user happened to pass.

Today a multi-ID invocation already defers: the second positional token matches no flag form in `parse_update_fields`,
which returns `None`. That is the safe status quo, not a feature — argparse then rejects the extra positional. It also
means an older published core degrades gracefully once the Python side accepts multiple IDs.

## Atomicity contract

The batch is one domain mutation, not a loop of mutations. Implement it as a single core operation so a partial batch
can never reach disk:

1. Resolve every supplied token to a canonical full ID with the existing shorthand resolver, before any write. Unknown
   IDs raise the existing not-found error naming the token the user typed; ambiguous shorthands raise the existing
   ambiguity error.
2. De-duplicate the resolved set while preserving first-seen argument order. Passing a shorthand and its full form, or
   the same ID twice, updates that bead once and is not an error.
3. Compute the post-update issue for every target and run the existing per-issue validation on all of them before
   mutating any of them.
4. Apply every accepted change, append every event, and call `store.save()` exactly once.
5. Report the changed IDs, the unchanged IDs, and the reopened-ancestor IDs so the caller can render truthfully and gate
   the commit.

Beads whose requested fields already hold the requested values are no-ops: they are excluded from the changed set, get
no `issue_updated` event, and never appear in the commit message. A batch in which nothing changed writes nothing and
creates no commit, matching today's single-ID quiet no-op.

### `--status closed` in a batch

`update` keeps the descendant guard that rejects closing a bead with unfinished descendants, and keeps pointing users at
`sase bead close`. Evaluate that guard against the batch's projected end state: a descendant that is itself being closed
by the same invocation counts as closed. This makes `sase bead update a1 a1.1 -s closed` and
`sase bead update a1.1 a1 -s closed` behave identically instead of depending on argument order, and it mirrors how
`close_issues` already treats its requested set. A descendant that is _not_ in the batch still rejects the whole batch,
and the error keeps naming the unfinished beads.

### Reopening ancestors in a batch

`update` moving a closed bead out of `closed` reopens its closed ancestors. Collect reopened ancestors across the whole
batch, reopen each one at most once even when several targets share an ancestor, and return the deduplicated ancestor
IDs in the outcome so the CLI can print them once each.

## Work in the linked `sase-core` repo

Open the checkout with the `/sase_repo` skill (`sase repo open sase-core -r "..."`); use only the path it prints.

### `crates/sase_core/src/bead/mutation.rs`

Add `update_issues(beads_dir, issue_ids: &[String], fields: BeadUpdateFieldsWire)` next to `update_issue`:

- Keep the existing `is_ready_to_work` rejection at the top.
- Run inside one `with_bead_mutation_lock` and one `MutableStore::load`.
- Resolve indexes for every ID first (`store.issue_index`), so a not-found ID aborts before any mutation.
- Clone `fields` per target (`BeadUpdateFieldsWire` is `Clone`) because `apply_update_fields` consumes it; derive
  `event_fields_from_update_fields` once and reuse it for the per-issue `IssueUpdated` payloads.
- Take `now` once from `fields.now` (or `now_utc()`) so every event in the batch shares one timestamp.
- When `fields.status` is `"closed"`, replace the per-issue `reject_unclosed_descendants` call with a batch-aware check
  that treats the requested set as closed, per the contract above.
- Skip targets whose computed issue equals the current issue; validate and stage the rest; then `store.save()` once.
- Return a `BeadMutationOutcomeWire` with `operation: "update"`, `changed` = "at least one issue changed", `issue_ids` =
  the changed IDs in requested order, `issues` = the resulting issues for every requested target (changed and unchanged,
  in requested order), and `reopened_ancestor_ids` = the deduplicated batch-wide set.
- Add an `unchanged_ids` field to `BeadMutationOutcomeWire` (`#[serde(default)]`, like its siblings) so callers can
  distinguish "updated" from "already had these values" without diffing.

Reimplement `update_issue` as a thin wrapper over `update_issues` with a one-element slice, keeping its current outcome
shape (`issue: Some(...)`, `issue_ids` = the single ID even when unchanged, `changed` = false on a no-op) so every
existing caller and test is unaffected. If preserving that shape through the shared path is awkward, leave
`update_issue` as-is and factor only the per-issue field application into a helper both call — do not change
`update_issue`'s observable outcome either way.

### `crates/sase_core/src/bead/cli.rs`

- Replace `parse_update_fields` with a parser that returns `(Vec<String>, BeadUpdateFieldsWire)`: consume flag/value
  pairs exactly as today, collect bare tokens as IDs, and keep returning `None` for any unrecognized `-`-prefixed token
  so `-z/--size` and `-r` still defer to Python. Values are still taken unconditionally after their flag, so a field
  value that starts with `-` is not mistaken for an ID.
- In `handle_update`: defer when the ID list is empty; resolve every ID with `resolve_cli_issue_id` against the loaded
  issues, returning `issue_ids_resolution_outcome` (as `handle_close` does) when any resolution fails; call
  `update_issues`; print one `✓ Updated issue: <id> — <title>` line per changed bead in requested order, one
  `· Unchanged: <id> — <title>` line per no-op target, and one `○ Reopened ancestor: <id> — <title>` line per reopened
  ancestor.
- Emit a mutation summary whose `issue_ids` are the changed IDs and whose `status_transitions` cover every bead whose
  status actually moved, following `handle_close`'s pattern rather than `mutation_summary`'s single-issue shape.

Note the single-ID output change: a no-op single-ID update currently prints `✓ Updated issue: ...` even though nothing
was written. It will now print `· Unchanged: ...`. This is a deliberate truthfulness fix consistent with `close`'s
`· Already closed` row; the changed-bead line format is untouched.

### `crates/sase_core_py/src/lib.rs`

- Export `update_issues` from `crates/sase_core/src/lib.rs` alongside `update_issue`.
- Add a `bead_update_many(beads_dir, issue_ids, fields) -> dict` PyO3 function mirroring `py_bead_update`, reusing
  `bead_update_fields_from_pydict`, and register it in the module. Document it in the module-level binding list next to
  `bead_update`.

### Core tests

Cover in `mutation.rs` tests: multi-ID success; mixed changed/unchanged batch; unknown ID leaves the store untouched;
invalid field value leaves the store untouched; duplicate and shorthand-duplicate collapse; order-independent
`--status closed` on a parent and its child; unfinished out-of-batch descendant rejects the batch; shared ancestor
reopened once. Cover in `cli.rs` tests: multi-ID fast path renders one row per bead and reports every changed ID in one
mutation summary; a `-z`-bearing invocation still defers. Follow `close_fast_path_accepts_note_and_updates_once` for
style. Add a binding round-trip test in `crates/sase_core_py/src/lib.rs` next to the existing `py_bead_update` test.

## Work in this repo

### Argparse surface — `src/sase/main/parser_bead_lifecycle.py`

In `register_bead_update_parser`, replace the `id` positional with:

```python
parser.add_argument(
    "ids",
    nargs="+",
    metavar="ID",
    help="One or more full or shorthand issue IDs to update",
)
```

Give the parser a `description` and an `epilog` with examples, and use
`formatter_class=argparse.RawDescriptionHelpFormatter`, matching `register_bead_close_parser`. Per the CLI rules memory,
keep the help text scannable, keep the option list alphabetized as it already is, and state in the description that
every listed bead receives the same field changes in one commit. Example epilog entries:

```
sase bead update sase-at.1 -s in_progress
sase bead update at.1 at.2 at.3 -s ready
sase bead update sase-at.1 sase-at.2 -a alice -z medium
```

### Facade and project adapter

- `src/sase/core/bead_mutation_facade.py`: add `update_many(beads_dir, issue_ids, **fields) -> tuple[list[Issue], dict]`
  calling `require_rust_binding("bead_update_many")` through `_call_issue_operation`, returning
  `issues_from_list(payload.get("issues", []))`. Add it to `__all__` (keep sorted).
- `src/sase/bead/_project_mutations.py`: add `BeadProjectMutationMixin.update_many(issue_ids, **fields)` that rejects
  `is_ready_to_work` like `update` does, resolves each ID through `self.resolve_id`, applies
  `_normalize_changespec_fields` / `_validate_issue_update` per target against the pre-batch issue, calls
  `rust_beads.update_many`, records the outcome, and refreshes the DB projection. Leave `update` untouched so the ACE
  plans view (`src/sase/ace/tui/actions/artifacts_plans.py`), `cli_work_task.py`, `cli_admin.py`, and
  `cli_work_cleanup.py` keep their current single-bead call.

### Handler — `src/sase/bead/cli_crud.py`

Rewrite `handle_bead_update` to build the same `fields` dict, then:

- call `mutation.project.update_many(args.ids, **fields)` inside the existing `bead_store_mutation` context, so the
  whole batch stays under one store write lock;
- read `mutation.project.last_mutation_outcome` for `issue_ids` (changed), `unchanged_ids`, and `reopened_ancestor_ids`;
- commit with `require_mutation_commit_message("update", changed_ids)` only when at least one bead changed, letting the
  existing `mutation_changed` gate suppress the commit for an all-no-op batch;
- print the same rows the Rust path prints, in the same order and with the same glyphs, so output does not depend on
  which backend served the command. Factor the row rendering the way `_print_close_results` / `_print_close_result_row`
  already do.

Keep the existing `KeyError` → `Error: issue not found: <id>` and `ValueError` → `Error: <msg>` handling, exiting 1
without committing.

### Commit message — `src/sase/bead/mutation_commit.py`

Change the `update` branch of `mutation_commit_message` to join all IDs, exactly like `close` and `rm`:

```python
if operation == "update" and issue_ids:
    return f"chore(beads): update {' '.join(issue_ids)}"
```

A one-element join is byte-identical to today's message, so single-ID commits are unchanged. `bead_fast_path.py` already
routes non-`close` operations through `mutation_commit_message(operation, issue_ids)` with the summary's `issue_ids`, so
the fast path picks this up with no further change — provided the core summary carries the changed IDs as specified
above.

### Push behavior

No plumbing change is needed and none should be made. The fast path commits and pushes through one
`auto_commit_bead_store(message)` call in `_apply_mutation_side_effects`; the slow path commits with
`push_after_commit=False` under the lock and calls `_push_committed_bead_store()` once after releasing it. Both already
yield exactly one commit and one push per invocation. The work here is making sure the batch is one mutation so those
existing single-commit paths stay single-commit — verify it with tests rather than by adding push code.

### Dependency floor

`bead_update_many` is a new `require_rust_binding` name, and CI's `published-core-minimum-smoke` job runs
`tools/check_sase_core_rs_bindings` against the exact minimum in `pyproject.toml`. Land the `sase-core` change first,
let release-please publish it, then raise `sase-core-rs` in `pyproject.toml` to that published version and refresh
`uv.lock`. Read the actual published version from the sase-core release rather than assuming the next number.

Do not guess: run `python3 tools/smoke_sase_core_rs_telemetry --print-minimum pyproject.toml` and
`python3 tools/check_sase_core_rs_bindings` against a venv with that published wheel before opening the sase PR, which
is what CI will do.

Heads up for the implementer: this gate is already red on `master` independently of this plan. `bead_resolve_id` (added
by the shorthand work) first appears in published `sase-core-rs` 0.17.3, while `pyproject.toml` still allows `>=0.17.0`.
The floor bump this plan requires also repairs that skew. Do not expand scope beyond the version bump itself.

### Docs and generated skill source

- `docs/beads.md`: retitle the section to `sase bead update <id> [<id2> ...]`, describe the bulk contract (same fields
  applied to every listed bead, all-or-nothing, one commit, duplicates collapsed), and note the batch-aware
  `--status closed` guard while keeping the existing "prefer `sase bead close`" guidance.
- `docs/configuration.md`: change the `sase bead update` flag table's `id` row to `ids` /
  `One or more full or shorthand issue IDs`, matching the `sase bead close` table just below it.
- `src/sase/xprompts/skills/sase_beads.md`: add a bulk example to the `update` section and one line stating that
  multiple IDs update in one commit. Check `sase/memory/generated_skills.md` with the `/sase_memory_read` skill before
  editing, and follow its regeneration and deployment steps.
- `src/sase/bead/cli_admin.py`: the `onboard` quick-start block lists `sase bead update <id> --status=in_progress`.
  Leave it single-ID; the quick start is deliberately minimal.

Do not edit `sase/memory/*.md`, `AGENTS.md`, or the generated `CLAUDE.md` / `GEMINI.md` / `OPENCODE.md` / `QWEN.md`
shims. Nothing in this change requires a memory update, and those files need explicit user permission.

## Tests in this repo

Add `tests/test_bead/test_cli_update_bulk.py`:

- multi-ID update applies the same field to every bead and returns the changed issues in argument order;
- exactly one `auto_commit_bead_store` call whose message is `chore(beads): update <id1> <id2>` with the IDs in argument
  order (patch `sase.bead.cli_crud.auto_commit_bead_store`, as `test_cli_auto_commit.py` does);
- a mixed batch commits only the IDs that actually changed and prints `· Unchanged` rows for the rest;
- an all-no-op batch makes no commit;
- an unknown ID in the middle of the list exits 1, writes no commit, and leaves `issues.jsonl` byte-identical;
- an invalid field value for one target leaves every other target unmodified;
- shorthand and full forms of the same bead collapse to one update and one ID in the commit message;
- `--status closed` across a parent and its only child succeeds in either argument order, while an unfinished
  out-of-batch descendant rejects the whole batch and writes nothing.

Extend existing suites:

- `tests/test_bead/test_cli_auto_commit.py`: update the `argparse.Namespace(id=...)` fixtures to `ids=[...]`, and add a
  fast-path case asserting a multi-ID summary produces one joined `chore(beads): update ...` message.
- `tests/test_bead/test_cli_mutation_push.py`: assert one deferred push for a multi-ID update.
- `tests/test_bead/test_project_rust_delegation.py`: add `update_many` to the delegation coverage.
- `tests/test_bead/test_cli_id_shorthand.py`: add a shorthand-multi-ID case.
- Sweep for any other `Namespace(id=...)` construction aimed at `handle_bead_update`
  (`tests/test_bead/test_cli_changespec.py` has several) and switch them to `ids=[...]`.

## Verification

1. In the `sase-core` checkout: `cargo fmt`, `cargo clippy --all-targets`, `cargo test -p sase_core`, and
   `cargo test -p sase_core_py`.
2. In this repo: `just install` first — it rebuilds `sase_core_rs` from the linked `sase-core` checkout, and this
   workspace may be stale. Then `just check`.
3. Manual round trip in a scratch bead store: create three beads, run `sase bead update <a> <b> <c> -s in_progress`, and
   confirm `git log` in the bead store shows exactly one `chore(beads): update <a> <b> <c>` commit and that the store's
   remote received exactly one push.
4. Repeat step 3 with `-z medium` to exercise the Python slow path, and confirm the output rows, commit message, and
   push count are identical to the fast path's.
5. Confirm a single-ID `sase bead update <id> -t "New title"` still prints one `✓ Updated issue:` line and commits
   `chore(beads): update <id>`.

## Landing order

1. `sase-core` PR: core mutation, fast path, binding, core tests.
2. Wait for the release-please release that publishes it.
3. This repo's PR: argparse, facade, project adapter, handler, commit message, floor bump + `uv.lock`, docs, generated
   skill source, tests.

The sase-side work can be written and fully verified locally before step 2 completes, because `just install` builds
`sase_core_rs` from the linked checkout. Only the PR's CI depends on the published release.

## Out of scope (deliberate)

- **`-P/--no-push` on `update`.** `sase bead close` has it and it would be a natural companion for bulk updates, but it
  is not part of this request. Adding it later is a self-contained follow-up.
- **Per-bead differing values.** Every listed bead receives the same field values; there is no per-ID field syntax.
- **`--phases`-style epic expansion.** `sase bead close -p 1-3` selects phase beads of an epic. Extending that selector
  to `update` is a separate feature.
- **Teaching the Rust flag parser `-z/--size` and `-r`.** Those still defer to Python, which now serves them atomically
  through `bead_update_many`. Closing that gap is a separate cleanup.
- **Bulk `sase bead note` / `open` / `create`.** Only `update` changes here.
