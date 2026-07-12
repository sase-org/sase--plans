---
create_time: 2026-07-08 23:02:43
status: done
prompt: 202607/prompts/bead_writes_workspace_local_sdd.md
tier: tale
---
# Fix: `sase bead` writes dirty the primary workspace's `.sase/sdd` instead of the current workspace's local clone

## Problem

In `separate_repo` SDD mode, agents running `sase bead` from a secondary/ephemeral workspace write their bead mutations
into the **primary** workspace's `.sase/sdd/beads/` clone rather than the current workspace's own `.sase/sdd` clone.
This:

- Leaves the primary workspace's SDD clone permanently dirty (uncommitted `beads/issues.jsonl` and
  `beads/events/streams/*.jsonl` changes), because the auto-commit and finalizer commit passes look at a _different_
  clone.
- Silently drops the bead changes from version control — the work is written to disk but never committed or pushed.
- Creates cross-workspace write contention: concurrent agents in different workspaces all mutate the same primary clone.

The intended model (already implemented for the commit side and for clone creation) is: **each workspace has its own
`.sase/sdd` clone; agents read/write their own clone; the finalizer commits and pushes that local clone.**

## Root cause

The SDD "separate-repo" model gives every workspace its own git clone at `<workspace>/.sase/sdd`
(`ensure_workspace_sdd_clone` / `src/sase/sdd/_store_link.py`). The authoritative path policy already encodes this —
`_sdd_dir_for_storage` in `src/sase/sdd/store.py:361-370`:

- `in_tree` → `<workspace>/sdd`
- `separate_repo` → `<workspace>/.sase/sdd` _(workspace-local)_
- `local` → `<primary>/.sase/sdd` _(shared, primary-only)_

and `resolve_sdd_dir`'s docstring (`store.py:130-138`) explicitly states that for `separate_repo` "the effective working
tree is workspace-local: `<workspace>/.sase/sdd`."

The **bead-write resolvers were never updated for this three-way split.** They branch only on the boolean
`SddStore.is_in_tree` (`src/sase/sdd/_store_types.py:55-57`, true only for `in_tree`) and treat every other mode — both
`local` _and_ `separate_repo` — identically as "use the primary":

1. **Slow path** — `find_beads_location` in `src/sase/bead/cli_common.py:34-46`:

   ```python
   if resolve_sdd_store(cwd, 1).is_in_tree:
       ... walk up for sdd/beads, else primary ...
   else:
       # Other modes: always primary/.sase/sdd/beads/
       return primary / ".sase" / "sdd", BEADS_DIRNAME_NON_VC   # <-- primary
   ```

   `get_project()` / `get_read_view()` (same file) open the `BeadProject` here, so every slow-path
   create/update/open/close/rm read+write targets the primary.

2. **Fast path** — `_resolve_lightweight_beads_context` in `src/sase/main/bead_fast_path.py:113-125`: mirrors the same
   logic and returns `primary/.sase/sdd/beads` for all non-in-tree modes.

Meanwhile **both commit paths correctly target the workspace-local clone**:

- `auto_commit_bead_store` (`cli_common.py:105-124`) commits `resolve_sdd_store(Path.cwd(), 1).sdd_dir` →
  `<workspace>/.sase/sdd` for `separate_repo`.
- The finalizer `_auto_commit_separate_sdd_store_if_possible` (`src/sase/llm_provider/commit_finalizer.py:325-350`)
  resolves the store from the run's workspace and commits the workspace-local clone with message
  `chore(sdd): sync uncommitted SDD store changes`.

Because a secondary workspace has `primary != <workspace>`, the writer and the committers disagree: mutations land in
the primary clone, both commit passes run against the (unchanged) workspace-local clone and no-op, and the primary is
left dirty forever. This was confirmed empirically — the workspace-local clone is clean at the finalizer's own commit,
while the primary clone carries exactly the reported dirty bead files.

A secondary, related gap: the fast path handles `create` for non-VC stores without any commit at all
(`bead_fast_path.py:84-85` bails `open/update/close/dep` back to the slow path for non-VC, but **not** `create`), and
`_apply_mutation_side_effects` never commits. So even once writes are pointed at the right clone, fast-path `create`
would leave that clone dirty until the finalizer runs.

## Design principle for the fix

Make the bead **write/read resolvers agree with the committers**: for `separate_repo`, resolve to the **current
workspace's own `.sase/sdd` clone**, never the primary. Only `local` (shared, non-cloned) mode may point at the primary.
`in_tree` behavior is unchanged.

To keep the writer and the committer byte-for-byte consistent (and robust to a CWD that is a subdirectory of the
workspace rather than its root), resolve the **current workspace root** once — from the checkout marker
(`find_marker_from_cwd`, the directory that owns `.sase/checkout.json`), falling back to walking up from CWD for an
ancestor containing `.sase/sdd/beads` — and derive `<workspace_root>/.sase/sdd` from it in every place that reads,
writes, or commits the bead store.

## Changes

### 1. Route `separate_repo` bead writes/reads to the workspace-local clone

**`src/sase/bead/cli_common.py` — `find_beads_location` (lines 34-46).** Replace the two-way `is_in_tree` branch with a
three-way split:

- `in_tree`: unchanged (walk up for `sdd/beads`, else primary).
- `separate_repo`: resolve to the current workspace's local clone — `<workspace_root>/.sase/sdd` with
  `BEADS_DIRNAME_NON_VC`. Determine `workspace_root` from the checkout marker / walk-up as described above; do **not**
  return the primary in this branch (in a secondary workspace the primary is never a valid write target).
- `local`: unchanged (`<primary>/.sase/sdd`).

**`src/sase/main/bead_fast_path.py` — `_resolve_lightweight_beads_context` (lines 106-135).** Apply the identical
three-way split so fast-path reads (`search`) and any fast-path writes target the workspace-local clone for
`separate_repo`, and the primary only for `local`.

### 2. Keep the committer consistent with the new writer

**`src/sase/bead/cli_common.py` — `auto_commit_bead_store` (lines 105-124).** Commit the same clone
`find_beads_location` now writes to. The simplest robust approach: derive the store directory from the shared
workspace-root resolution (or directly from `find_beads_location()`), rather than `resolve_sdd_store(Path.cwd(), 1)`
with a hardcoded `workspace_num=1`, so a subdirectory CWD cannot split the writer and committer onto different clones.
Continue to skip in-tree stores and to pass `paths=[<store>/beads]`.

Factor the workspace-root + store resolution into a single shared helper used by `find_beads_location`, the fast path,
and `auto_commit_bead_store` so the three can never drift apart again.

### 3. Give fast-path `create` a commit (close the non-VC create gap)

For non-VC (`separate_repo`/`local`) stores, make `create` behave like the other mutations: either route it through the
slow path (extend the `bead_fast_path.py:84-85` bail set to include `create`, so `handle_bead_create` performs the write
**and** `auto_commit_bead_store`), or call the shared auto-commit from the fast path after a non-VC mutation. Routing to
the slow path is preferred — it reuses the existing commit-with-bead-id message and keeps all non-VC write commits in
one place; the fast path then serves only reads (`list`/`show`/`search`) for non-VC stores, consistent with how
`open/update/close/dep` already behave.

### 4. Finalizer backstop (verify; likely no code change)

The finalizer already commits the whole workspace-local `separate_repo` store (no path filter), so once writes land
there it will commit both the bead JSONL changes and any other dirty SDD files (e.g. research/plan artifacts). Confirm
via test that after routing writes to the local clone, an agent run that mutates beads ends with a clean workspace-local
clone and a committed change. Note as an out-of-scope observation that the finalizer's SDD fallback is gated to
`separate_repo` only and does not cover `local` mode; that is acceptable for this bug but worth a follow-up.

## One-time remediation of the already-dirty primary

The primary workspace's `.sase/sdd` currently carries real, uncommitted bead state (phase status/notes updates for
in-flight epics) plus an untracked SDD research markdown file. This is legitimate agent work that must not be lost. As a
**manual, one-time step** (separate from the code change, performed from the primary workspace): review the pending
changes and commit/push them to the SDD repo (e.g. `sase bead sync` + a commit, or the standard SDD commit path) so the
history is preserved. The code fix prevents recurrence but cannot retroactively move changes already written into the
primary clone.

## Related investigation (fold in if confirmed)

The untracked `research/<yyyymm>/*.md` file in the primary's `.sase/sdd` is **not** produced by `sase bead`; it is a
non-bead SDD artifact. Audit the non-bead SDD write paths (plan/research/prompt materialization and any
`sase sdd`/companion refresh that writes files) for the same primary-vs-workspace-local conflation. If a non-bead writer
also resolves to `<primary>/.sase/sdd` from a secondary workspace, apply the same "workspace-local for `separate_repo`"
correction there. If instead those files are written workspace-locally already, the finalizer's whole-store commit
(Change 4) will capture them once beads stop diverting to the primary — in which case no additional change is needed.

## Testing

- **`tests/test_bead/test_cli_resolution.py`**: this file currently encodes the buggy behavior via
  `test_find_beads_location_non_vc_prefers_primary_workspace` _without pinning a storage mode_. Re-scope it: add
  explicit coverage that
  - `separate_repo` mode + managed marker with `primary != workspace` resolves to the **workspace-local**
    `<workspace>/.sase/sdd/beads` (regression test for this bug), including when CWD is a workspace subdirectory;
  - `local` mode still resolves to `<primary>/.sase/sdd/beads`;
  - `in_tree` mode is unchanged. Pin the storage mode explicitly (patch `resolve_sdd_store` /
    `get_configured_sdd_storage`) so tests do not depend on ambient repo config.
- **`tests/main/test_bead_fast_path.py`**: mirror the three-way resolution assertions for
  `_resolve_lightweight_beads_context`, and assert that non-VC `create` no longer resolves+writes to the primary (and
  now commits / bails to the slow path).
- **`tests/test_bead/test_cli_auto_commit.py`**: add a round-trip test — in a simulated secondary workspace under
  `separate_repo`, a `sase bead create`/`update` writes to the workspace-local clone and `auto_commit_bead_store`
  commits it, leaving the workspace-local clone clean and the primary untouched.
- **`tests/llm_provider/test_commit_finalizer_auto_sdd_status.py`**: confirm the finalizer commits the workspace-local
  clone containing the routed bead changes.
- Run `just check` (after `just install`).

## Risks / notes

- Must preserve `local` (non-separate) mode's primary-targeted behavior and the existing `in_tree` walk-up; only
  `separate_repo` changes.
- The write and commit resolutions must be derived from one shared helper to avoid re-introducing a writer/committer
  split under subdirectory CWDs.
- Reads used by the ACE TUI / cross-project views (`get_project_beads_dirs` → `_canonical_project_beads_dir` in
  `src/sase/bead/workspace.py`) intentionally point at the primary/canonical clone and are **not** changed here;
  per-workspace clones reconcile with that view through the existing commit→push and `git pull` refresh path. Flag but
  do not alter in this change.
