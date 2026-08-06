---
tier: epic
title: Ship the close-history reducer so bead reopen provenance actually works
goal: "A bead that is closed and later reopened really does keep its close reason at runtime, not only in tests: the
  sase-core reducer that archives close metadata is released and adopted, and `sase bead search` finds an archived close
  reason.

  "
phases:
  - id: core-search
    title: Make archived close reasons searchable and release the reducer
    depends_on: []
    size: medium
    description: "core-search: add close_history to sase-core's BEAD_SEARCH_FIELD_NAMES and searchable_fields so the
      Rust matcher agrees with the Python snippet map, fold it into the pending close-history branch, land that branch
      on master, and get the release published.

      "
  - id: adopt
    title: Adopt the release and prove close history end to end
    depends_on:
      - core-search
    size: small
    description: "adopt: raise the sase-core-rs window to the release from core-search, refresh uv.lock and the
      declared-minimum assertion, replace the end-to-end test's skip guard with a hard assertion, and add real CLI
      coverage for searching an archived close reason and for the history entry a reopen writes.

      "
  - id: land
    title: Close epic sase-fr and retire its plan
    depends_on:
      - adopt
    size: small
    description:
      "land: close bead sase-fr with the recorded verification and follow-up outcomes, clear anything symvision reports
      once its epic-symbol whitelist expires, and mark the close-history plan file done."
proposed_by: bbugyi200.athena.sase-fr.land
parent_bead: sase-fr
create_time: 2026-08-06 00:19:21
status: wip
---

- **PROMPT:**
  [prompts/202608/close_history_landing.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/close_history_landing.md)
- **PARENT:**
  [202608/bead_close_history.md](https://github.com/sase-org/sase--plans/blob/main/202608/bead_close_history.md)

# Plan: Ship the close-history reducer so bead reopen provenance actually works

## Context

Epic `sase-fr` (`~/.sase/plans/202608/bead_close_history.md`) set out to stop a `+1` from destroying the reason a bead
was closed, and to say plainly on every surface that a bead was previously closed, why, and what reopened it.

Its eight phases all closed, and the Python half is genuinely complete. `close_history` is threaded through
`src/sase/bead/model.py`, `close_history_codec.py`, `core/bead_wire.py`, `jsonl.py`, `db.py` (schema column plus
`_migrate_add_close_history`), `work.py`, and `cli_admin.py`'s projection-repair allowlist. `reopen_presentation.py`
holds the shared vocabulary, and `cli_detail.py`, `cli_detail_json.py`, `cli_query.py`, `task_gate.py`,
`sase_chop_bead_task_triage.py`, the three ACE beads-pane modules, `bead_pages/`, and `docs/beads.md` all consume it.
Roughly 160 targeted tests cover it, including a rich-ANSI golden and a PNG snapshot.

Two things stop that work from doing anything for a real user.

### The reducer that populates the field was never released

`sase-fr.1` wrote the sase-core change and opened [PR #86](https://github.com/sase-org/sase-core/pull/86)
(`feat(bead): archive close metadata instead of destroying it on reopen`). Its own phase note records the risk plainly:
"NOT YET MERGED, so no release version exists yet."

That is still true. Verified against the live repository:

```text
gh pr view 86 --repo sase-org/sase-core   → state: OPEN, mergedAt: null
gh release list --repo sase-org/sase-core → v0.18.2 (latest)
grep -rn close_history <sase-core master> → no matches in crates/
pyproject.toml                            → sase-core-rs>=0.18.2,<0.19.0
```

Every CI check on the PR is green (`cargo fmt + clippy + test`, `maturin build + import smoke`, `Cargo version guard`,
`Conventional PR title`), and it is the only open PR on the repo. It is simply waiting.

Because the released reducer still calls `clear_close_metadata`, `close_history` is empty for every bead in every store.
The end-to-end test says so out loud and skips:

```text
SKIPPED [1] tests/test_bead/test_close_history_end_to_end.py:47:
  installed sase-core-rs still clears close metadata on reopen
```

So the badge never renders, the `PREVIOUSLY CLOSED` section never appears, and the TaskTriage gate — the surface the
epic called its highest-value one, where the owner is asked to Launch or Close a freshly reopened task — still shows no
prior-close callout. `sase-fr.2` could not raise the dependency window for the same reason and recorded it as a
`PROPOSED FOLLOW-UP`.

### Archived close reasons are still not findable

The epic's `cli` phase set out to make archived reasons searchable, on the stated grounds that "an archived close reason
that cannot be found by `sase bead search` is only half-recovered."

`src/sase/bead/cli_query.py` duly gained a `close_history` entry in `_search_field_value`. But that map only supplies
the _snippet_ for a field that already matched; **match selection happens in Rust**, and
`crates/sase_core/src/bead/search.rs` has neither `close_history` in `BEAD_SEARCH_FIELD_NAMES` nor a `close_history` arm
in `searchable_fields`. The field name can therefore never appear in `matched_fields`, which makes the Python entry
unreachable. Both `sase-fr.1` and `sase-fr.4` recorded this as a `PROPOSED FOLLOW-UP`; `sase-fr.4`'s test only exercises
`_search_field_value` directly, so nothing caught it.

PR #86 touches `search.rs` by exactly one line — a `close_history: Vec::new()` in a test fixture — so the gap survives
the merge unless it is fixed alongside.

### One follow-up that turned out to be a false alarm

`sase-fr.1` also proposed that `sase bead history` would not surface close-history changes, on the belief that
`crates/sase_core/src/bead/history.rs` "enumerates tracked fields explicitly." It does not: `history.rs::issue_fields`
serializes the whole `IssueWire` with `serde_json::to_value` and diffs the resulting object, and `is_default_field`
already treats an empty array as absent. `close_history` will therefore appear in `sase bead history` on its own once
the field exists on the wire. No work is needed — only a test that pins it, which the `adopt` phase adds.

## Design

Two blocking gaps, one release cycle.

The search fix belongs in the same sase-core branch as the reducer, not in a follow-up PR. A second PR would mean a
second release and a second dependency bump for a five-line change, and it would leave a window in which the Python
snippet map points at a field the matcher cannot produce. PR #86 is unmerged and unreviewed, so extending it costs
nothing.

Everything downstream is already written. Once the release lands, adoption is a version bump plus turning three
currently-unprovable claims into tests: that a real reducer archives the close reason, that `sase bead search` finds it,
and that `sase bead history` shows it.

### Attribution of remaining risk

The dependency-window move is the interaction `sase-fr`'s own Sequencing section predicted. The window is now
`>=0.18.2,<0.19.0`, raised by `7ffd5471a` for the in-flight CI-recovery epic's commit-budget fix. Because release-plz
cuts from master, the release that carries close history is a superset of 0.18.2 and adopting it cannot drop that fix —
but the `adopt` phase confirms it rather than assuming it.

## Phases

### Make archived close reasons searchable and release the reducer

This phase crosses the Rust core backend boundary. Open the sase-core checkout with
`sase repo open sase-core -r "<reason>"` and use only the path it prints. Nothing in the sase repo changes here.

Check the state of the branch first: `gh pr view 86 --repo sase-org/sase-core`. The branch is
`sase-core_bead-close-history-core-model_1` at commit `66011f5`. If it has already been merged by the time this phase
runs, make the search change a small follow-up commit on master instead and skip to the release step; the rest of this
section is unchanged either way.

In `crates/sase_core/src/bead/search.rs`:

- Add `"close_history"` to `BEAD_SEARCH_FIELD_NAMES`, next to `"plus_one_evidence"`.
- Add a matching arm to `searchable_fields`, built unconditionally like the `refs` and `plus_one_evidence` arms so the
  field is always present and simply empty when there is no history. Its text must match what
  `src/sase/bead/reopen_presentation.py::close_history_search_text` flattens on the Python side — for each record, the
  `close_reason`, the `resolution` value, the `closed_at`, and the `reopened_at`, newline-joined, skipping absent
  values. If the two disagree, a query can match in Rust and then render an empty snippet in Python.

The existing `searchable_fields`-versus-`BEAD_SEARCH_FIELD_NAMES` coverage test asserts the two lists are equal, so it
will fail until both are updated; that is the intended tripwire. Add tests that a query matching an archived
`close_reason` returns the issue with `close_history` in `matched_fields`, that a query matching an archived
`resolution` does too, and that a bead with no close history does not match a query that only appears in another bead's
archived reason.

Then land it:

- Rebase on master if needed, push, and confirm every check goes green again.
- Merge to master. The repository's own convention is worth knowing: `sase-org/sase-core`'s master branch is not
  protected and feature work normally lands as a direct push, with release-plz's `chore: release` PR being the only
  thing that routinely goes through review. Approving this plan is the authorization to land this branch.
- release-plz opens a `chore: release` PR on merge. This is an additive `feat` on top of `v0.18.2`, so expect `v0.19.0`.
  That release PR is merged by the project owner, not by an agent: if it is still open once the branch has landed and CI
  is green, say so through `/sase_questions` and wait rather than merging it or proceeding.
- Record the exact published version in the phase notes. `adopt` reads it from there.

### Adopt the release and prove close history end to end

Everything here is in the sase repo.

Raise the window in `pyproject.toml` from `sase-core-rs>=0.18.2,<0.19.0` to the released version — `>=0.19.0,<0.20.0`
for the expected `0.19.0` — and refresh `uv.lock`. Before doing so, confirm the release's tree contains the
commit-log-budget fix that `7ffd5471a` raised the floor to `0.18.2` for; it will, because release-plz cuts from master,
but confirm it rather than assume it. Then update the declared-minimum assertion in
`tests/test_sase_core_rs_telemetry_smoke_tool.py` (line 35 pins `"0.18.2"` today) — a stale pin there is exactly what
`sase-fq.7` had to repair.

Then turn the three unprovable claims into tests.

`tests/test_bead/test_close_history_end_to_end.py` currently guards its assertions behind

```python
if not issue.close_history:
    pytest.skip("installed sase-core-rs still clears close metadata on reopen")
```

Delete the guard. Its purpose was to let the Python storage work land ahead of the release, and leaving it in place
would silently swallow any future regression that stops the reducer archiving. The rest of the test already asserts the
right things: the flat trio nulled, the reason and resolution preserved in `close_history[0]`, `reopened_via`,
`reopened_by`, the `(reporter, timestamp)` join against the `+1` evidence, the `issues.jsonl` projection, the SQLite
mirror, and a reload.

Add CLI-level coverage that the phase-4 unit test could not reach:

- Through a real `BeadProject`, close a task with a distinctive reason, `+1` it, then assert `sase bead search` for a
  phrase from that archived reason returns the bead with `close_history` among its `matched_fields` and renders the
  snippet. This is the end of the chain the epic's `cli` phase could only test halfway.
- Assert `sase bead history` on the same bead reports the `close_history` field transition, pinning the behaviour that
  `history.rs`'s whole-issue serialization gives for free — so a later change to explicit field enumeration cannot
  quietly drop it.

Then check the surfaces now that the data is real rather than hand-built: `sase bead show` on the reopened bead renders
the `[↺1]` badge and the `PREVIOUSLY CLOSED` section with the `↺ reopened this task` marker on the joining `+1`, and
`sase bead show --format json` carries `close_history` plus `reopened_bead`.

One consequence to expect rather than be surprised by: this repo's own bead store will materialize archived records for
beads that were closed and reopened before the reducer changed, since `MutableStore::load` replays events. That is the
retroactive recovery the epic designed for. Rows for beads that were never reopened stay byte-identical because
`close_history` is skipped when empty. If `sase bead admin --fix-projection` reports those rows, the guard already
allows the field, so let it repair them.

Finish with `just install` then `just check`. Treat the known contended-host flakes as known: the ACE TUI nodes tracked
by `sase-ct`, the bead-lock node tracked by `sase-e2`, and `test_contract_set_serial_runtime_stays_within_budget`
tracked on epic `sase-fp`.

### Close epic sase-fr and retire its plan

Close `sase-fr` with `sase bead close sase-fr --note "<verification>"`. The note should state that all eight phases were
verified against the source and the epic's seven commits; that the reducer released and was adopted, which is what made
the feature live; and it must carry the follow-up outcomes already triaged during land verification:

- Rust search index missing `close_history` — resolved by this plan's `core-search` phase (proposed by `sase-fr.1` and
  `sase-fr.4`).
- sase-core-rs floor bump blocked on an unreleased reducer — resolved by this plan's `adopt` phase (proposed by
  `sase-fr.2`).
- `sase bead history` not surfacing close history — declined as a false alarm; `history.rs::issue_fields` serializes the
  whole `IssueWire`, so the field is tracked for free, and `adopt` adds a test pinning it (proposed by `sase-fr.1`).
- Contended-host test flakes — corroborated on the existing umbrella beads during land verification rather than filed as
  new tasks: `+1` on `sase-e2` for the bead-mutation lock node, `+1` on `sase-ct` for the three ACE TUI nodes, and a
  `DISCOVERED ISSUE` note on in-progress epic `sase-fp`, which introduced
  `test_contract_set_serial_runtime_stays_within_budget` in `ab955c9ca` (proposed by `sase-fr.3` through `sase-fr.8`).

After the close, run `just symvision`. `sase-fr`'s epic-symbol whitelist entries expire on close; every phase already
removed its own entries and the Justfile currently has none, so a clean run is the expected outcome — but if it reports
anything newly unused, remove the stale entries and the dead code behind them.

Finally set `status: done` in the frontmatter of `~/.sase/plans/202608/bead_close_history.md`.

## Risks and non-risks

- **Not a risk: adopting the release drops the commit-budget fix.** release-plz cuts from master, and `0.18.2` is on
  master, so the close-history release contains it. `adopt` confirms it anyway.
- **Not a risk: old stores break.** `IssueWire` does not use `deny_unknown_fields` and `close_history` is skipped when
  empty, so unaffected rows stay byte-identical and older sase builds ignore a key they do not know.
- **Managed: the release PR is the owner's to merge.** `core-search` surfaces it through `/sase_questions` rather than
  merging it or stalling silently.
- **Watch: Python and Rust search text drifting.** The two flatteners are written separately and only agree by
  convention. `core-search` mirrors `close_history_search_text` deliberately, and `adopt`'s CLI-level search test is
  what would catch a divergence, because a Rust-only match renders an empty snippet.

## Out of scope

- **Bead event actors are wrong for close, open, and ready-marking.** Still the original epic's stated out-of-scope
  item, and still true: `close_issues`, `open_issue`, and `set_ready_to_work` pass `&issue.created_by` as the event
  actor, which is why `reopened_by` is populated only for `+1` reopens. Unchanged here.
- **Folding the current close into `close_history`.** Same reasoning as the original epic.
- **Deflaking the contended-host tests.** Tracked on `sase-e2`, `sase-ct`, and epic `sase-fp`.
