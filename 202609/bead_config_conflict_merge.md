---
tier: tale
title: Merge config.json next_counter conflicts in the bead conflict resolver
goal:
  Concurrent bead-id mints across sidecar clones no longer wedge bead-store publication;
  epic sase-wh's diverged store can integrate, relocate, and launch.
size: medium
proposed_by: bbugyi200.apollo.f
create_time: 2026-09-04 09:37:16
status: wip
---

# Plan: Merge `config.json` `next_counter` conflicts in the bead conflict resolver

## Problem

Epic `sase-wh` ("Initialize projects from the Admin Center Projects tab") cannot launch.
`sase bead work sase-wh -Y` fails during pre-spawn checkpoint publication:

```
Error: epic launch checkpoint publication failed before agent launch for sase-wh:
git rebase failed: ... Could not apply 4ffc62506... chore(beads): checkpoint approved
epic graph sase-wh; unsupported bead conflicts: config.json
```

Root cause, confirmed by inspecting the primary checkout's beads sidecar clone
(`sase/repos/beads` under the primary host checkout, remote
`git@github.com:sase-org/sase--beads.git`, currently 3 ahead / 18 behind `origin/main`):

1. Two clones concurrently minted top-level bead counter 1170 → id `sase-wh`. An agent
   land run created a flake-task bead `sase-wh` and pushed first; the primary clone
   created the epic `sase-wh` and lost the push race.
2. Concurrent duplicate mints are exactly what the resolver's relocation machinery
   (`merge_event_streams_with_relocation` + `_RelocationIds` in
   `src/sase/bead/conflict_resolver.py`) handles: the local side relocates to a fresh
   id, upstream keeps the id (see
   `test_duplicate_top_level_creations_report_typed_relocation`).
3. But the two sides bumped `config.json`'s `next_counter` **unequally** (base 1169 →
   local 1170; → upstream 1172, which minted `sase-wh`, `sase-wi`, `sase-wj`). Git
   therefore reports a content conflict on `config.json`, and `_is_mergeable_bead_path`
   does not include `config.json`, so `resolve_bead_conflicts` returns "unsupported bead
   conflicts: config.json" and the whole integration aborts — publication is wedged.

Note the existing relocation test never hits this because both sides mint the same
_number_ of beads there, producing byte-identical `config.json` on both sides (no git
conflict). Unequal mint counts — the common real-world case — always conflict, so any
concurrent-mint collision wedges the losing clone until someone hand-repairs it. This
will keep recurring until the resolver can merge `config.json`.

## Fix

Teach `_resolve_bead_conflicts` in `src/sase/bead/conflict_resolver.py` to semantically
merge a conflicted store `config.json` when — and only when — the divergence is confined
to `next_counter`:

1. **Classify `config.json` as mergeable.** Add the store `config.json` path to
   `_is_mergeable_bead_path` (via `_store_path(bead_prefix, "config.json")`).

2. **Merge it before any other resolution work.** `_RelocationIds.__init__` calls
   `sase.bead.config.load_config`, which `json.load`s the worktree `config.json`; while
   that file holds git conflict markers the load raises. So, immediately after the
   unsupported-conflict check and before `_load_worktree_streams` / `_RelocationIds`:
   - Read the three conflict stages with the existing `_unmerged_stages` /
     `_upstream_and_local_stages` / `git show :<stage>:<path>` helpers (stage 1 = base
     may be absent; treat it as `{}`).
   - Parse each stage as JSON. Any parse failure → return the existing "unsupported bead
     conflicts: config.json" failure.
   - If local and upstream differ on **any key other than `next_counter`** (including
     keys missing on one side), or any present `next_counter` is not an `int` →
     unsupported, same message. `issue_prefix`, `owner`, and unknown future keys must be
     byte-equal semantically; we only ever merge the counter.
   - Merged content = the common keys plus `next_counter = max(base, local, upstream)`
     (missing values excluded).
   - Write the merged config to the worktree so `load_config` succeeds.

3. **Account for relocation allocations in the final counter.** After the stream-merge
   loop, when `config.json` was among the conflicts, rewrite it with
   `next_counter = max(merged value, the highest counter `_RelocationIds` allocated + 1)`
   so a relocated id can never be re-minted by the next local creation. Expose the
   allocator's final counter with a small accessor on `_RelocationIds` (it already
   tracks `self._counter`). Write the file with `sase.bead.config.save_config` (its
   `json.dump(..., indent=2)` + trailing newline matches the store's existing on-disk
   encoding — verify against the Rust store writer's output in a test rather than
   assuming), then stage it with `_git_add`, and include the path in `resolved_paths`
   exactly once.

4. **Config-only conflicts must still resolve.** If `config.json` is the only store
   conflict (no conflicted streams), the current `if not store_conflicts:` early-return
   must not swallow it: ensure the flow still writes + stages the merged config and
   reports it resolved. Keep the derived `issues.jsonl`/`events/manifest.json` rewrite
   behavior unchanged.

5. **Update user-facing text.** The `sase bead resolve-conflicts` subcommand help in
   `src/sase/main/parser_bead_store.py` currently enumerates the mergeable files ("Only
   sdd/beads/issues.jsonl, events/manifest.json, and events/streams/\*.jsonl conflicts
   are auto-merged"); add `config.json` (next_counter-only) to that enumeration. The
   module docstring of `conflict_resolver.py` needs no change.

Boundary note: this stays in Python deliberately. The Rust core owns the event-stream
merge semantics (`sase.core.bead_conflict_facade` bindings); the resolver's git-stage
plumbing, store writing (`_write_resolved_store`), and id allocation (`_RelocationIds`)
are already Python in this repo, and the config merge is more of that same git-conflict
plumbing. No other frontend consumes it except through this module. No Rust/wire change
is needed.

## Tests

All in `tests/test_bead/test_conflict_resolver.py`, following the existing
`_init_repo`/`_git` scenario style:

1. **Regression for the production wedge** (the important one): seed a store, branch; on
   the "other" (upstream) branch mint three top-level beads; on master mint one (its id
   collides with upstream's first). Merge → conflicts on `config.json` (unequal
   `next_counter`) _and_ the colliding stream. Assert: resolution ok; exactly one
   relocation whose `old_id` is the local bead; relocated id collides with none of
   upstream's ids; merged `config.json` parses with `next_counter >= ` both sides'
   values and `> ` the relocated id's counter; `config.json` appears once in
   `resolved_files`; the reduced store shows both titles under distinct ids.
2. **`next_counter`-only conflict merges to the max** even without a stream collision
   path exercised (construct the conflict directly, as the existing
   `test_resolve_bead_conflicts_rejects_nonmergeable_bead_file` does): assert ok, merged
   value, file staged (no longer listed by `git diff --name-only --diff-filter=U`).
3. **Non-counter divergence stays unsupported**: the existing
   `test_resolve_bead_conflicts_rejects_nonmergeable_bead_file` (owner differs) must
   keep passing unchanged; add a variant where both `next_counter` _and_ `owner` diverge
   → still "unsupported bead conflicts: config.json".
4. **Malformed stage stays unsupported**: one side's `config.json` stage is invalid JSON
   → unsupported, no crash.

Also confirm the encoding assumption in test 1 or 2: after resolution, the merged
`config.json` bytes must round-trip against what `sase.bead.config.save_config` writes
for the same dict (guarding the byte-churn concern documented in
`_write_resolved_store`'s docstring).

## Verification

- `just check` (agent default; scoped lane must pick up
  `tests/test_bead/test_conflict_resolver.py`).
- `just check-full` runs only via the landing monitor per the two-speed rule; do not run
  it inline.

## Post-landing operational unwedge (host-side; NOT part of this tale's implementation — the implementing agent must not execute any of this, and must not touch any `sase/repos/beads` clone)

Documented here so the whole remediation is in one place. After this tale lands on
`origin/master`, the host operator (Bryan, from the primary checkout
`~/projects/github/sase-org/sase` — the `sase` CLI is a uv-tool _editable_ install of
that checkout, so `git pull` there activates the fix):

1. `git -C ~/projects/github/sase-org/sase pull` (fast-forward; makes the fixed resolver
   live).
2. In the wedged sidecar clone `~/projects/github/sase-org/sase/sase/repos/beads`:
   squash the 3 unpushed local commits into one, content-preserving
   (`git fetch origin && git reset --soft $(git merge-base HEAD origin/main) && git commit -m "chore(beads): checkpoint approved epic graph sase-wh"`).
   Rationale: rebase integrates commit-by-commit; with 3 stacked commits the colliding
   `sase-wh` stream would conflict in successive rounds against already-relocated
   content, risking a second spurious relocation. One squashed commit = one conflict
   round = exactly the scenario the resolver and its tests cover.
3. `sase bead sync` from the primary checkout — the fixed resolver max-merges
   `config.json`, relocates the local epic off `sase-wh` to a fresh id, and pushes. Note
   the printed "relocated duplicate beads: sase-wh -> <new-id>".
4. Launch the epic under its **new** id: `sase bead work <new-id> -Y`. Do **not** re-run
   `sase bead work sase-wh -Y` verbatim after the sync: post-merge, `sase-wh` names the
   _upstream flake-task bead_, not the epic.
