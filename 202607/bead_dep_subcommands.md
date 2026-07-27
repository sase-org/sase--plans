---
tier: epic
title: Make the bead dependency graph readable and editable
goal: '`sase bead dep` becomes a complete verb: every edge in the store is visible
  with the provenance that says who added it and when, the blocking graph can be walked
  in either direction with cycles and diamonds rendered honestly, and a wrong edge
  can be removed through an auditable event instead of storage surgery.

  '
phases:
- id: graph
  title: See every dependency edge and where it came from
  depends_on: []
  size: medium
  description: 'graph: add the shared read-side dependency adapter and ship `sase
    bead dep list` as its first consumer, including the fast-path verb classification
    fix that read-only `dep` subcommands need.

    '
- id: remove
  title: Remove a dependency edge as a recorded event
  depends_on: []
  size: medium
  description: 'remove: add the `dependency_removed` event, its replay rules, and
    the `remove_dependencies` mutation in the Rust core, expose them through the binding
    and `sase bead dep rm`, and stop the SQLite mirror from resurrecting a removed
    edge.

    '
- id: tree
  title: Walk the blocking graph in either direction
  depends_on:
  - graph
  size: medium
  description: 'tree: render the dependency graph as a tree over the adapter, with
    cycle, repeat-subtree, depth-truncation, and unresolved-target markers, plus a
    footer that names the longest blocking chain.

    '
- id: land
  title: Land the three verbs as one documented contract
  depends_on:
  - remove
  - tree
  size: small
  description: 'land: reconcile the docs, the generated skill, and `bead onboard`
    around the finished verb, bump the published core version window, and produce
    acceptance evidence against the live store.

    '
create_time: 2026-07-27 13:45:29
status: wip
---

# Plan: Make the bead dependency graph readable and editable

## Context

This epic implements recommendation 9 of the 2026-07-25 beads-leverage research
(`202607/sase_beads_leverage_20260725/`), which is also recommendation 7 of the 2026-07-14 consolidated report. Both
name the same gap in the same words:

> `dep rm` and `dep list` (**verified: `dep` has only `add`**; removing a wrong edge means storage surgery), plus
> `ready --explain` and a `dep tree`/graph check for diagnosing why work isn't ready.

Re-verified in this workspace on 2026-07-27:

```
$ sase bead dep --help
usage: sase bead dep [-h] {add} ...
    add       Add a dependency
```

`handle_bead_dep` in `src/sase/bead/cli_admin.py` has one branch and an `Unknown dep action` fallthrough. The Rust fast
path (`handle_dep` in `crates/sase_core/src/bead/cli.rs`) defers unless the arguments are exactly `add <a> <b>`.

The scale that makes this matter: the 2026-07-25 measurement found **1,271 beads carrying at least one dependency edge**
across 86 epic graphs authored in ten days, 48 of them with a parallel wave and 49 with a fan-in. Dependency edges are
the most-used part of the bead model and the only part with no way to inspect or correct them.

### Why the missing half is worse than "no `rm`"

Three concrete consequences, each verified against this checkout:

1. **A wrong edge is unfixable through the CLI.** The store is event-sourced; `dependencies` is projected by replaying
   `dependency_added` events (`apply_event` in `crates/sase_core/src/bead/events.rs`). There is no compensating event,
   so correcting a mistake means editing tracked JSONL by hand — which is exactly the "storage surgery" the research
   names.
2. **Edge provenance is recorded and unreachable.** `DependencyWire` stores `created_at` and `created_by`, and
   `Dependency` in `src/sase/bead/model.py` carries them into Python. No surface prints either. `render_issue_detail` in
   `src/sase/bead/cli_detail.py` prints `DEPENDS ON` and `BLOCKS` as bare title rows; `issue_to_wire_dict` includes the
   timestamps only inside `--format json`. Before you delete an edge you want to know who added it and when, and today
   you cannot find out.
3. **"Why is this not ready?" is a manual join.** `sase bead blocked` prints `[blocked by: <ids>]` — IDs only, one level
   deep, no titles and no statuses. Following a blocking chain means running `sase bead show` repeatedly and building
   the graph in your head.

### Two defects this epic must not walk past

Both were found while reading the code for this plan, and both are load-bearing for `dep rm` specifically.

**The SQLite mirror never deletes a dependency row.** `import_from_jsonl` in `src/sase/bead/jsonl.py` upserts issues and
then loops `for dep in issue.dependencies: db_mod.add_dependency(...)` inside a `try/except: pass`. There is no delete
pass. That mirror is rebuilt lazily by `rebuild_from_jsonl` in `src/sase/bead/_sync_git.py` whenever `issues.jsonl` is
newer than `beads.db`, and `BeadProject._export` falls back to `export_to_jsonl(self._conn, ...)` — reading that mirror
— whenever the Rust export raises. So an edge removed from the canonical event store can survive in `beads.db` and be
written back into the projection by the fallback export. Adding `dep rm` on top of an insert-only mirror would ship a
verb that appears to work and then silently un-works. `remove` fixes the dependency direction and reports on the issue
direction rather than expanding into it.

**`dep` is classified as unconditionally mutating on the fast path.** `_MUTATING_VERBS` in
`src/sase/main/bead_fast_path.py` contains `"dep"`, so `try_handle_bead_fast_path` calls
`assert_bead_store_write_sandboxed` for _any_ `dep` invocation before dispatch. Adding read-only `dep list` and
`dep tree` under that classification would make two read commands assert a write boundary — and fail inside a pytest
process whose sandbox is not published, which is precisely the guard's designed behavior. `graph` fixes the
classification before adding the first read subcommand.

### Design decisions

- **Only `dep rm` crosses into `sase-core`.** Removing an edge is an event-sourced mutation and has no Python-side
  expression; it must be a core change. Reading the graph does not: `bead_list` already returns every issue with its
  `dependencies`, and the reverse index this epic needs is the same one `resolve_issue_detail` already builds in Python
  today (`cli_detail.py`, the `block_ids` comprehension). Following that precedent keeps `list` and `tree` independent
  of a core release, which is what lets `graph` and `remove` run in parallel. The adapter is deliberately a single
  pure-function module so that moving it into Rust later — if the ACE open-bead tree needs the same traversal — is a
  relocation rather than a rewrite. State that boundary call in the module docstring so the next reader sees it was
  chosen, not defaulted into.
- **`dep list` earns its place beside `show`.** It adds three things `show` does not have: edge provenance, a per-edge
  satisfied/blocking verdict, and a store-wide view. Where the two overlap, the vocabulary matches exactly —
  `DEPENDS ON` and `BLOCKS`, the same `→` and `←` arrows, the same status glyphs — so the new command reads as a lens on
  something already familiar rather than a second dialect.
- **`dep rm` mirrors `dep add` argument-for-argument.** Same positional order (`<issue>` first, then what it depends
  on), no confirmation prompt — matching `bead rm`, which is far more destructive and has none — and no `--dry-run`,
  because `dep add` is the exact inverse and the removal is recorded as an event either way. Multiple targets are
  accepted and validated all-or-nothing, the way `remove_issues` already validates every ID before writing.
- **Removal is a new event, never an edited stream.** A `dependency_removed` event keeps the streams append-only, keeps
  merges replay-stable, and makes the removal visible in `sase bead history` for free.
- **No new link kinds.** Both research reports anti-recommend adding relationship taxonomy speculatively. This epic
  makes the one existing edge kind inspectable and correctable; it adds no `discovered-from`, no `validates`, no
  link-kind field.

### Boundary and constraints

Bead storage, mutation, and event replay are core backend behavior and live in the `sase-core` linked repo
(`crates/sase_core/src/bead/`, with the PyO3 surface in `crates/sase_core_py/src/lib.rs`). Open it with the `/sase_repo`
skill and use the path that command prints; never clone or web-fetch it. Core and Python changes land in the same phase,
because `just install` builds `sase_core_rs` from the local checkout — only the published window in `pyproject.toml`
waits for `land`.

New and changed CLI options follow `sase/memory/cli_rules.md`: subcommands and options sorted alphabetically, a short
alias for every public long option, help output that scans cleanly, and color where it improves readability.

**Epic `sase-a1` is in flight and touches the same files.** Its phases `.3` through `.6` are open or in progress, and
they modify `crates/sase_core/src/bead/{events,mutation,cli}.rs`, `crates/sase_core_py/src/lib.rs`,
`src/sase/main/parser_bead.py`, and `docs/beads.md`. Expect mechanical rebase conflicts in sorted lists and match
statements; resolve by keeping both sides and re-sorting. Do not revert or reshape anything `sase-a1` added — in
particular, do not touch `close_issues`, the resolution enum, or the note-append work.

Non-goals:

- No `ready --explain` and no `ready` filters. The research bundles them with this recommendation; they are a separate
  command's surface and this epic does not touch `ready`.
- No wave or critical-path _scheduling_ view. `dep tree`'s footer names the longest chain it walked, which is a
  rendering detail of the tree it already built; reusing `work --dry-run`'s wave planner is out of scope.
- No ACE TUI surface, no open-bead tree, no TUI CRUD.
- No `--json` on `ready`, `blocked`, or `stats`, and no exit-code contract work.
- No new link kinds, priorities, or labels.
- No repair of existing edges, and no bulk edge cleanup against the live store.
- **No edits to `sase/memory/*.md`, `AGENTS.md`, or the generated provider instruction shims** (`CLAUDE.md`,
  `GEMINI.md`, `OPENCODE.md`, `QWEN.md`). Those need explicit user permission in the acting agent's own conversation,
  which this plan does not confer. The generated `sase_beads` skill source under `src/sase/xprompts/skills/` is not a
  memory file and is in scope for `land`.

Every phase runs `just install` before `just check`, because workspaces are ephemeral and dependencies drift.

### The finished surface

Authored across three phases; recorded here once so the phases agree on it.

```
sase bead dep                                            → delegates to `sase bead dep list`
sase bead dep add <issue> <depends-on>                   (unchanged)
sase bead dep list [<id>]
  -c, --color {always,auto,never}                        Color output (default: auto)
  -d, --direction {both,in,out}                          Edges to show (default: both)
  -f, --format {compact,full,json}                       Output format (default: compact)
  -n, --limit N                                          Max beads to print; 0 means unlimited
  -s, --status {claimed,closed,in_progress,open}         Filter by status (repeatable)
sase bead dep rm <issue> <depends-on> [<depends-on2> ...]
sase bead dep tree [<id>]
  -c, --color {always,auto,never}                        Color output (default: auto)
  -d, --direction {both,in,out}                          Direction to walk (default: out)
  -f, --format {compact,full,json}                       Output format (default: compact)
  -L, --levels N                                         Max levels to descend; 0 means unlimited
  -s, --status {claimed,closed,in_progress,open}         Filter by status (repeatable)
```

`--direction out` follows "depends on" (what am I waiting for), `in` follows "blocks" (what is waiting on me), and
`both` shows each. `list` defaults to `both` because a flat listing is naturally bidirectional; `tree` defaults to `out`
because that is the question a tree answers, and `--direction both` renders the two trees in sequence.

`--status` selects which beads may appear. A store-wide invocation defaults to `open`, `claimed`, and `in_progress`,
matching `sase bead list`. An invocation scoped to one ID defaults to every status, because a closed dependency is
exactly what you want to see when asking whether something is blocked, and the scoped root always prints regardless of
the filter.

Adding an exact `list` child means bare `sase bead dep` delegates to it through `_default_list_subcommands` in
`src/sase/main/parser.py` and prints `No subcommand provided for 'sase bead dep'; delegating to 'sase bead dep list'.`
That wiring is central; do not re-implement it, and do not add a `dep list` default that makes the bare form fail — the
positional is optional precisely so the bare form is useful.

## See every dependency edge and where it came from

Two deliverables: the adapter every later phase reads through, and `sase bead dep list` as its first consumer.

### The adapter

Add `src/sase/bead/dep_graph.py`. It is pure: it takes the issue list a caller already fetched and returns frozen
dataclasses, performs no I/O, and imports nothing from the CLI layer.

- `DepEdge` — `issue_id`, `depends_on_id`, `created_at`, `created_by`, and `satisfied`, where satisfied means the target
  bead exists and its status is `closed`. An unresolved target is never satisfied; an edge pointing at a bead that is
  not in the store is a real state the store can reach (see the `IssueRemoved` replay arm) and every surface must render
  it rather than crash.
- `DepGraph` — built once from `list_issues()`, holding the issue index, forward adjacency, reverse adjacency, and the
  edge set, all in deterministic order (source ID, then target ID). Expose lookups for one bead's outgoing and incoming
  edges, and a resolver that returns the `Issue` or `None` for an ID.
- Traversal for `tree`: a depth-bounded walk in either direction that reports, per visited node, whether it is a repeat
  of a subtree already expanded and whether the edge closes a cycle. Cycle detection is per-path (an ID already on the
  current path), repeat detection is global (an ID already expanded anywhere). Both are needed and they are not the same
  test: a diamond is a repeat, not a cycle, and rendering it as one would be a lie.

Nothing in the store forbids a cycle — `add_dependency` in `crates/sase_core/src/bead/mutation.rs` checks only for a
duplicate edge — so the traversal must terminate on a cyclic graph by construction, not by luck. Build a cyclic fixture
and assert termination directly.

Keep the module's public surface to what `graph` and `tree` actually consume; Symvision counts test-only references as
dead (`sase/memory/symvision.md`). If `tree` needs a helper that `list` does not, add it in `tree`.

### The command

Add `src/sase/bead/cli_dep.py` with `handle_bead_dep_list` and its renderers, and move `handle_bead_dep` there from
`src/sase/bead/cli_admin.py` so the whole verb lives in one module — `cli_admin.py` keeps `sync`, `doctor`, `onboard`,
and `resolve-conflicts`. Re-export through `src/sase/bead/cli_basic.py` and `src/sase/bead/cli.py` the way the other
handlers are; `src/sase/main/entry.py` keeps dispatching `"dep"` to `handle_bead_dep`, which grows a `list` branch.
Register the subparser in `src/sase/main/parser_bead.py` so `dep`'s children read `add`, `list`.

Reads go through `get_read_view()` from `src/sase/bead/cli_common.py`, like every other query handler.

**Fast-path classification.** In `src/sase/main/bead_fast_path.py`, replace the unconditional `"dep"` membership in
`_MUTATING_VERBS` with a check that treats `dep` as mutating only when its subcommand is one that writes. Keep the
default closed: an unrecognized or absent `dep` subcommand is treated as mutating, so a future writing subcommand is
guarded before anyone remembers to add it. Add a test that `dep list` does not trigger the write assertion and that
`dep add` still does.

`dep list` and `dep tree` stay off the Rust fast path entirely — `execute_bead_cli`'s `handle_dep` already defers on
anything that is not `add`, and `sase-a1` established the precedent that a Python-rendered read command does not need a
second Rust renderer and a second golden surface.

### Output

`compact`, scoped to one bead, is the primary view:

```
○ sase-a2.3 · Wire dep rm through the CLI   [OPEN]

DEPENDS ON (2)
  → ✓ sase-a2.1 · Record dependency-removal events     [CLOSED]        satisfied
  → ○ sase-a2.2 · Build the dependency graph adapter   [OPEN]          blocking

BLOCKS (1)
  ← ◐ sase-a2.4 · Land the dep verbs as one contract   [IN_PROGRESS]

────────────────────────────────────────────────────────────
Blocked by 1 of 2 dependencies.
```

`compact`, store-wide, groups by source bead and closes with a one-line census:

```
○ sase-a2.3 · Wire dep rm through the CLI
  → ✓ sase-a2.1 · Record dependency-removal events
  → ○ sase-a2.2 · Build the dependency graph adapter

◐ sase-a2.4 · Land the dep verbs as one contract
  → ○ sase-a2.3 · Wire dep rm through the CLI

────────────────────────────────────────────────────────────
3 dependencies · 2 beads · 1 satisfied · 2 active
```

`full` adds the provenance line under each edge — the reason this command exists ahead of `dep rm`:

```
  → ○ sase-a2.2 · Build the dependency graph adapter   [OPEN]          blocking
      added 2026-07-27T14:02:11Z by bryanbugyi34@gmail.com
```

`json` prints one envelope: `scope` (the ID, or `null` store-wide), `direction`, `count`, and `edges`, where each edge
carries `issue`, `depends_on`, `created_at`, `created_by`, `satisfied`, and `direction`. Render the two bead references
with `_ref_to_wire_dict` from `src/sase/bead/cli_detail.py` — promote it to a public helper there rather than
duplicating the shape — so an unresolved target has the same `"resolved": false` form `show --format json` already
emits.

Empty results print `No dependencies found.` (store-wide) or `<id> has no dependencies.` (scoped), and exit 0. An
unknown ID prints `Error: issue not found: <id>` to stderr and exits 1, matching `handle_bead_show`.

**Color.** Reuse `bead_status_presentation` in `src/sase/bead_status_presentation.py`, which already carries a
`cli_style` ANSI code per status; do not introduce a second palette. Color the status glyph by status and the bead ID
bold blue, matching the constants the Rust renderer uses in `crates/sase_core/src/bead/cli.rs`. `--color auto` emits
color only when stdout is a TTY and `NO_COLOR` is unset; `always` and `never` override both. `--format json` is never
colored.

### Tests

`tests/test_bead/test_dep_graph.py` for the adapter: forward and reverse adjacency, a satisfied edge, an edge with an
unresolved target, a diamond producing a repeat and not a cycle, and a cyclic fixture terminating.
`tests/test_bead/ test_cli_dep_list.py` for the command: each format scoped and store-wide, the `--direction` values,
the differing default status filters, `--limit`, the unknown-ID exit code, `--color never` producing no escape sequences
and `--color always` producing them off a TTY, and the empty case. Add a fast-path test for the verb classification. Add
a `dep_list.stdout` golden only if the existing `tests/test_bead/test_cli_golden.py` fixture store already has edges
that exercise both directions; otherwise keep the assertions in the new module rather than growing the golden store.

## Remove a dependency edge as a recorded event

The one phase that crosses into `sase-core`.

### Event

In `crates/sase_core/src/bead/events.rs`, add `BeadEventOperationWire::DependencyRemoved` and a matching
`BeadEventPayloadWire::DependencyRemoved { dependency: DependencyWire }`. Carrying the full edge — not just the target
ID — means the archive records what was dropped, including its original `created_at` and `created_by`. Validate the
payload the way `DependencyAdded` does, rejecting a payload whose `issue_id` disagrees with the event's.

Two replay rules matter more than the rest, and both are about surviving a merge:

- **Ordering.** `event_operation_priority` currently ranks `IssueCreated => 0`, `DependencyAdded => 2`, everything else
  `=> 1`, and that rank is the second component of `merge_key`. Give `DependencyRemoved` a rank above `DependencyAdded`,
  so an add and a remove sharing a timestamp always replay add-then-remove. Getting this backwards resurrects the edge,
  and only on merged stores — the failure mode `4376ec2` ("make event merges replay-stable") was written to prevent.
- **Tolerance.** The `apply_event` arm removes the edge if present and is a **no-op when it is absent**, and unlike
  `DependencyAdded` it does not require the target issue to still exist. Both cases are reachable on a legitimately
  merged store: `IssueRemoved` already purges dependency edges during replay, and two branches can each record a removal
  of the same edge. Erroring here would make a valid store unreadable. Assert both directly.

`import_issues_to_event_streams` needs no change: it synthesizes `dependency_added` events from the projection, and a
removed edge is absent from the projection, so re-import followed by replay is stable. Add a test that says so, because
it is the kind of invariant that holds until someone adds a synthesis pass.

### Mutation

Add `remove_dependencies(beads_dir, issue_id, depends_on_ids, now)` to `crates/sase_core/src/bead/mutation.rs`, beside
`add_dependency` and inside `with_bead_mutation_lock`. Validate everything before writing anything: the issue exists,
and every requested target is currently an edge of it. Deduplicate repeated targets. Then drop the edges, append one
`DependencyRemoved` event per edge, and save. A batch with one bad target leaves the store untouched — the same
all-or-nothing contract `remove_issues` already offers.

Return `operation: "dep_rm"` with `issue_ids` holding the source ID followed by the removed target IDs, and add a
`dependencies: Vec<DependencyWire>` field to `BeadMutationOutcomeWire` (with `#[serde(default)]`, mirroring the existing
`issue`/`issues` pair) carrying the removed edges. Errors distinguish the two failures: a missing issue uses the
existing `not_found` kind so the Python facade's `KeyError` translation applies unchanged, and a missing edge is a
validation error naming both IDs.

### Fast path and binding

Extend `handle_dep` in `crates/sase_core/src/bead/cli.rs` with an `rm` arm taking three or more arguments, returning a
`dep_rm` mutation summary. It must keep deferring for `list`, for `tree`, and for any argument starting with `-`, so the
Python renderers stay authoritative for the read verbs. Expose `bead_dep_remove` in `crates/sase_core_py/src/lib.rs`
following the `bead_dep_add` pattern and register it in the module init.

### Python

`remove_dependencies` in `src/sase/core/bead_mutation_facade.py` returning the removed `Dependency` list;
`BeadProject.remove_dependencies` in `src/sase/bead/project.py` beside `add_dependency`, calling
`_refresh_db_from_jsonl` the same way; an `rm` branch in `handle_bead_dep` using the
`bead_store_mutation(auto_commit_bead_store)` wrapper with commit message `chore(beads): unlink <issue> -> <targets>`;
the matching `dep_rm` case in `_mutation_commit_message` in `src/sase/main/bead_fast_path.py` so the fast path and the
slow path produce the same commit message; and the subparser, placed so `dep`'s children read `add`, `list`, `rm`.

Output states the consequence, because the reason you remove an edge is to change what can run:

```
✗ Removed dependency: sase-a2.3 no longer depends on sase-a2.1
○ sase-a2.3 is now ready (no active blockers).
```

or, when blockers remain, `○ sase-a2.3 still has 1 active blocker: sase-a2.2.` Compute that from the bead's post-removal
edges; do not re-derive readiness rules locally.

### Stop the mirror resurrecting removed edges

Fix `import_from_jsonl` in `src/sase/bead/jsonl.py` to reconcile dependencies rather than only insert them: after
upserting an issue's edges, delete the rows for that issue whose `depends_on_id` is not in the projection. Add the
delete helper to `src/sase/bead/db.py` beside `add_dependency`.

Then measure the blast radius rather than assuming it. `import_from_jsonl` also never deletes _issues_ that left the
projection, so `sase bead rm` has the same latent gap; determine whether a stale mirror can actually reach a user —
through `rebuild_from_jsonl` in `src/sase/bead/_sync_git.py` and the `export_to_jsonl` fallback in `BeadProject._export`
— and report the finding in the phase's completion message. Fix the dependency direction here; do not expand this phase
into the issue direction.

Regression test, end to end and deliberately ugly: build a store, add an edge, force the SQLite mirror to materialize,
remove the edge, force a rebuild from the projection, and assert the mirror no longer has the row and that an export
through the Python fallback does not reintroduce it.

### Tests

Rust, in `mutation.rs` and `events.rs`: a single removal; a batch removal; a batch whose second target is not an edge
failing with no write; removing from an unknown issue; add-remove-add replaying to present; add-remove sharing one
timestamp replaying to absent; a removal event for an edge that is not present replaying as a no-op; a removal whose
target issue was already removed replaying cleanly; and a projection round-trip through
`import_issues_to_event_streams`. Python, in `tests/test_bead/test_cli_dep_rm.py`: removal through the CLI, the
readiness consequence line in both forms, the two error messages and their exit codes, the all-or-nothing batch, the
auto-commit message, and the mirror regression above. Confirm — do not assume — that `sase bead history <id>` shows the
removal as a `dependencies` change; `issue_fields` in `crates/sase_core/src/bead/history.rs` serializes the whole
`IssueWire`, so it should follow for free, and if it does not, that is a finding for `land` to record.

## Walk the blocking graph in either direction

`sase bead dep tree` over the adapter from `graph`. Rendering and traversal only; no new store access patterns.

Scoped to one bead it roots the tree there. Store-wide it renders a forest rooted at the beads that nothing depends on
(with `--direction out`), which puts the top of each blocking chain at the top of the output.

```
○ sase-a2.4 · Land the dep verbs as one contract
└─ ○ sase-a2.3 · Wire dep rm through the CLI
   ├─ ✓ sase-a2.1 · Record dependency-removal events
   └─ ○ sase-a2.2 · Build the dependency graph adapter
      └─ ✓ sase-a2.0 · Spike the graph shape

────────────────────────────────────────────────────────────
5 beads · depth 4 · 2 active blockers
Longest chain: sase-a2.4 → sase-a2.3 → sase-a2.2 → sase-a2.0
```

Four markers, each for a state the graph can actually reach:

- **Repeat** — `⇡ (shown above)` appended to the row, and the subtree is not expanded again. A fan-in graph is a DAG,
  not a tree; 49 of 86 recently authored epic graphs have a fan-in, so re-expanding shared subtrees would make the
  common case unreadable.
- **Cycle** — `↻ (cycle)` in the warning color, the walk stops at that node, and the footer adds
  `Warning: 1 dependency cycle detected: sase-a2.4 → sase-a2.3 → sase-a2.4`. Nothing prevents a cycle from being
  created, so nothing may assume one is absent.
- **Truncated** — with `--levels N`, a node whose children were not walked prints `(+3 more, use --levels 0)`.
- **Unresolved** — a target that is not in the store prints `? <id> (not found)`, matching how `render_issue_detail`
  already handles the same case.

`full` adds the status label and phase size to each row plus the edge's provenance line, the same shape
`dep list --format full` uses. `json` prints `{"scope": ..., "direction": ..., "roots": [...]}` where each node has
`issue` (the shared reference shape), `edge` (provenance and `satisfied`, absent on a root), `repeat`, `cycle`,
`truncated`, and `children`; a repeat, cycle, or truncated node carries an empty `children` list rather than being
omitted, so a consumer can render the marker without inferring it.

Sibling order is the adapter's deterministic order, so two runs against an unchanged store produce identical bytes.
Color follows the `graph` rules exactly, with the cycle marker as the one addition to the palette.

Tests in `tests/test_bead/test_cli_dep_tree.py`: a linear chain; a diamond marked repeat rather than cycle; a cycle
marked and terminating with the footer warning; `--levels` truncating and reporting the remainder; `--direction in`
inverting the walk; `--direction both` rendering two trees; an unresolved target; the forest form picking the right
roots; the JSON node shape; and byte-identical output across two runs.

## Land the three verbs as one documented contract

The fan-in phase. Nothing here is new behavior; all of it is what makes the behavior reachable.

**Docs.** In `docs/beads.md`, replace the four-line `sase bead dep add <issue> <depends_on>` section with a
`sase bead dep` section covering all four subcommands, keeping the CLI Commands list alphabetical. Document the
bare-`dep` delegation to `list`, the differing `--direction` and `--status` defaults and why they differ, and the cycle
and repeat markers. Extend the Dependencies section under Data Model with edge provenance and the fact that removal is
an event rather than an erasure, and extend the Event Log section with `dependency_removed` and its replay-ordering
rule.

**Skill.** `src/sase/xprompts/skills/sase_beads.md` documents `dep add` alone. Replace that with the full verb, in the
diagnostic order an agent will need it: `dep list <id>` to see what a bead waits on and what waits on it, `dep tree` to
follow the chain, `dep rm` to correct a wrong edge. Per `sase/memory/generated_skills.md`, edit the source template, run
`sase skill init --force`, then `chezmoi apply`; never hand-edit a generated provider copy. This is the targeted
addition the research's recommendation 2 skill rewrite would later absorb, not that rewrite.

**Onboard.** `handle_bead_onboard` in `src/sase/bead/cli_admin.py` lists `sase bead dep add` in its quick-start block;
add the three new lines in the same aligned two-column style.

**Version window.** Bump `sase-core-rs` in `pyproject.toml` to the window that includes the release carrying `remove`'s
core changes, following the `build(deps): require sase-core-rs ...` precedent. Dev installs and CI build from the local
checkout and ignore the window, so verify against the actually published release rather than assuming a version number.
`sase-a1` may bump the same line first; take whichever floor is higher.

**Acceptance evidence.** Against the live store, read-only except where stated:

1. `sase bead dep list sase-a1.6 --format full` — the epic's real fan-in edges render with provenance.
2. `sase bead dep tree sase-a1 --direction in` — the dependents of a live epic render as a tree.
3. `sase bead dep` bare — prints the delegation notice and the store-wide listing.
4. `sase bead dep list --format json | python -c 'import json,sys; json.load(sys.stdin)'` — the envelope parses.
5. On a scratch store, not the live one: create three beads and two edges, `dep rm` one of them, confirm the readiness
   line, confirm `sase bead history` shows the `dependencies` change, and confirm `sase bead dep list` no longer reports
   the edge after a mirror rebuild.
6. Count the beads carrying at least one edge in the live store and compare against the 1,271 the research recorded on
   2026-07-25, reporting the current number rather than restating theirs.

Report every result in the phase's completion message, including anything that did not match. Do not remove any edge
from the live store.

## Verification

Every phase, in its ephemeral workspace:

```bash
just install
just check
```

`remove` additionally runs the `sase-core` test suite from the linked checkout, since the Python suite exercises the
binding but not the Rust unit tests.

Expect mechanical conflicts between `graph` and `remove` in `src/sase/main/parser_bead.py`, `src/sase/bead/cli.py`,
`src/sase/bead/cli_basic.py`, and `src/sase/main/bead_fast_path.py`, and between this epic and the in-flight `sase-a1`
in `docs/beads.md`, `crates/sase_core/src/bead/events.rs`, and `crates/sase_core_py/src/lib.rs`. All are additions to
sorted lists or match statements; resolve by keeping both and re-sorting.
