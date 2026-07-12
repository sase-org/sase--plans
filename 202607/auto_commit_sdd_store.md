---
create_time: 2026-07-08 12:44:45
status: wip
prompt: 202607/prompts/auto_commit_sdd_store.md
tier: tale
---
# Plan: Auto-commit & push the separate-repo SDD store (with finalizer fallback)

## Problem

When the SDD store is configured as a **separate repository** (`sdd.storage: separate_repo`, checked out at
`<primary>/.sase/sdd/` as its own git clone of e.g. `git@github.com:sase-org/sdd.git`), changes to that store can
silently pile up locally without ever being committed or pushed to GitHub.

Two concrete gaps cause this:

1. **Mutating bead commands don't commit or push.** `sase bead close` (and `create`, `update`, `open`, `rm`, dependency
   add, work-cleanup) mutate the beads JSONL under `.sase/sdd/beads/` via the Rust mutation facade, but the CLI handlers
   only _write_ the files — they never git-commit or push the SDD repo. Only `sase bead work` (launch) and the
   plan/research archival paths currently commit+push. So a bare `sase bead close` leaves the `sase-org/sdd` clone
   dirty.

2. **The commit finalizer can't see the SDD repo.** The post-completion commit finalizer inspects (a) the main workspace
   via the VCS provider diff and (b) declared _linked repos_. The separate SDD store is neither: `.sase/` is gitignored
   in the main repo, so `.sase/sdd/` never shows up in the main diff, and the SDD store is materialized through a
   mechanism entirely distinct from linked repos, so it never shows up there either. The finalizer's only SDD awareness
   is a narrow _in-tree_ `status: wip → done` plan-frontmatter auto-commit, which does not match the separate-repo
   layout.

Net effect: whatever misses the (few) code paths that call `commit_sdd_store_files` leaves the companion repo un-synced,
and nothing downstream catches it.

## Goals

- Any command that modifies SDD-store files commits the change to the SDD repo and pushes it to GitHub **at the point of
  mutation** (primary mechanism). `sase bead close` is the motivating case.
- The commit finalizer detects a dirty separate-repo SDD store at agent-run completion and commits+pushes it as a
  **fallback**, so anything the per-command hooks miss is still synced.
- Behavior is a no-op for `in_tree` stores (their SDD files ride with the normal code-repo commit workflow) and for
  `local` stores with no remote (commit only, nothing to push).
- Failures never break the user's command or fail the agent run: commit/push errors are logged, the local commit is
  preserved, and the fallback can retry later.

## Non-goals

- No change to the in-tree SDD flow or to `sase bead work` (which already commits+pushes).
- No new Rust/`sase-core` work. Bead JSONL mutation stays in Rust; all SDD **git** orchestration already lives in Python
  (`src/sase/sdd/_commit.py`, `src/sase/bead/sync.py`) and we reuse it. This stays on the presentation/glue side of the
  Rust-core boundary.
- No redesign of the finalizer's existing main-repo/linked-repo dirty-state loop.
- Not attempting to reconcile concurrent writers beyond the existing best-effort push (see Risks).

## Background: the pieces we build on (all already exist)

- `resolve_sdd_store(workspace_dir, workspace_num) -> SddStore` (`src/sase/sdd/store.py`): resolves the effective store.
  For `separate_repo`/`local` it points at `<primary>/.sase/sdd`; for `in_tree` at `<workspace>/sdd`. Primary is
  resolved via the checkout marker, so the same call from any workspace targets the _same_ clone. `SddStore.is_in_tree`
  distinguishes modes.
- `commit_sdd_store_files(store, message, *, auto_commit_type, paths=None, push_after_commit=None)`
  (`src/sase/sdd/_commit.py`): stages only changed files under `paths` (or the whole store), commits with runtime tags,
  and — **only for `separate_repo` stores** — pushes per `sdd.push_after_commit` (default `"async"`, detached + logged).
  No-op when the store dir has no `.git` or nothing changed. Push failure never undoes the local commit. This is the
  single helper both mechanisms call.
- The commit finalizer: `run_commit_finalizer(...)` in `src/sase/llm_provider/commit_finalizer.py` (invoked from
  `src/sase/llm_provider/_invoke.py` after each provider turn). It already has a precedent for _directly_ committing
  without re-prompting the LLM: `_auto_commit_done_plan_status_if_possible(...)`, run right after dirty-state collection
  and before the re-prompt loop. Our fallback slots in the same place.

Because both mechanisms resolve the store through `resolve_sdd_store`, they operate on the identical clone the bead
writes land in — no risk of committing the wrong `.sase/sdd/`.

## Design

### Part A — Commit+push at the mutation site (primary mechanism)

Add one shared helper used by the SDD-mutating CLI handlers:

- New helper (e.g. `auto_commit_bead_store(message: str) -> None` in `src/sase/bead/cli_common.py`, or a small new
  `src/sase/bead/cli_commit.py`). It:
  1. Resolves the store: `store = resolve_sdd_store(Path.cwd(), 1)` (the existing bead-CLI pattern; the checkout marker
     makes the primary resolution robust regardless of the placeholder number).
  2. Returns immediately if `store.is_in_tree` (in-tree bead files are committed by the code-repo workflow; we must not
     create side commits in the user's working tree).
  3. Otherwise calls
     `commit_sdd_store_files(store, message, auto_commit_type="beads", paths=[store.sdd_dir / "beads"])`. Scoping
     `paths` to the beads subtree avoids sweeping in unrelated concurrently-edited SDD files (e.g. a plan the agent is
     mid-write on).
  4. Is wrapped so any exception is logged and swallowed — a git hiccup must never fail `sase bead close`.
     (`commit_sdd_store_files` already tolerates _push_ failures; we additionally guard the _commit_ call.)

Wire it into the handlers that mutate the store, each with a descriptive message (matching the existing
`chore(beads): …` commit style):

- `handle_bead_create` → `chore(beads): create <id>`
- `handle_bead_update` → `chore(beads): update <id>`
- `handle_bead_open` → `chore(beads): reopen <id>`
- `handle_bead_close` → `chore(beads): close <ids>`
- `handle_bead_rm` → `chore(beads): remove <id>`
- `handle_bead_dep` (add-dependency, `src/sase/bead/cli_admin.py`) → `chore(beads): link <a> -> <b>`
- work-cleanup update (`src/sase/bead/cli_work_cleanup.py`) → a matching message

`sase bead work` is left as-is (it already commits+pushes via `commit_successful_work_launch`).

Why per-handler rather than hiding it inside the `get_project()` context manager: it yields precise, reviewable commit
messages, avoids surprising interactions with the work-launch flow's own commit/push and with read-only views, and keeps
mutation intent explicit. The finalizer fallback (Part B) is the safety net for any handler we don't wire up (including
future ones).

### Part B — Finalizer fallback (catch-all)

Teach the finalizer to sync a dirty separate-repo SDD store directly (like the existing in-tree plan-status auto-commit
— a direct commit, not an LLM re-prompt, since the store is a gitignored satellite the agent's `/sase_git_commit` skill
does not operate on).

- Add `_auto_commit_separate_sdd_store_if_possible(project_dir)` in `src/sase/llm_provider/commit_finalizer.py`. It:
  1. Resolves `store = resolve_sdd_store(project_dir, <workspace_num>)` (best-effort; marker-driven).
  2. No-ops unless `store.storage == separate_repo` and `store.repo_root/.git` exists.
  3. Calls `commit_sdd_store_files(store, "chore(sdd): sync uncommitted SDD store changes", auto_commit_type="sdd")` —
     **whole-store** (no `paths` scoping): at run completion, everything dirty in the store should be synced (beads
     _and_ any plan/prompt/research edits a code path forgot to commit).
  4. Swallows/logs exceptions so finalization never fails on a git error.
- Call it in `run_commit_finalizer` right after `_collect_dirty_state(...)`, alongside
  `_auto_commit_done_plan_status_if_possible(...)`, and again after each re-prompt pass (the passes can themselves
  produce new SDD changes). Because the SDD store is invisible to `collect_dirty_state`, this runs independently of —
  and even when — the main dirty-state is clean (it must run _before_ the existing `dirty_state.is_clean` early return).

### Interaction between A and B

Both go through `commit_sdd_store_files`, which is idempotent: if Part A already committed+pushed, the store is clean
and Part B's call is a no-op. Async pushes that race simply find nothing new. No double commits of the same change.

### Configuration

Reuse the existing `sdd.push_after_commit` knob (`true` | `false` | `"async"`, default `"async"`) — it already governs
whether/how `commit_sdd_store_files` pushes for separate-repo stores, so both the per-command hook and the finalizer
fallback honor it for free. Optionally gate the finalizer fallback behind the existing `commit.finalizer.enabled` flag
it already lives under; no new config is required.

## Files to change (approximate)

- `src/sase/bead/cli_common.py` (or new `src/sase/bead/cli_commit.py`) — the shared `auto_commit_bead_store` helper.
- `src/sase/bead/cli_crud.py` — call it from create/update/open/close/rm handlers.
- `src/sase/bead/cli_admin.py` — call it from the add-dependency handler.
- `src/sase/bead/cli_work_cleanup.py` — call it from the cleanup update.
- `src/sase/llm_provider/commit_finalizer.py` — `_auto_commit_separate_sdd_store_if_possible` + wiring into
  `run_commit_finalizer`.

## Edge cases & risks

- **In-tree stores:** helper returns early; finalizer fallback gated on `separate_repo`. In-tree bead changes keep
  riding with the code-repo commit workflow — behavior unchanged.
- **`local` store (no remote):** commits happen; `commit_sdd_store_files` only pushes for `separate_repo`, so there is
  simply nothing to push. Fine.
- **No changes / no `.git`:** `commit_sdd_store_files` is already a safe no-op.
- **Push needs interactive credentials:** async push detaches with stdin closed and logs failure (existing behavior).
  SSH-remote setups push non-interactively. The local commit is always preserved; a later command or the finalizer
  retries.
- **Concurrent workspaces writing the same primary `.sase/sdd`:** a push may be rejected as non-fast-forward. This is
  already tolerated (logged, commit kept); a subsequent materialize/`git pull --ff-only` or push reconciles, and the
  bead-store rebase conflict resolver exists for JSONL. Documented as a known limitation, not addressed here.
- **Performance:** each mutating bead command gains one bounded `git add`+`commit` (local, timeout- guarded) and a
  detached async push — negligible for interactive CLI use.

## Testing

- Unit tests for `auto_commit_bead_store`: `separate_repo` → commit created + push invoked (mocked); `in_tree` → no-op
  (no side commit in the code repo); no-change → no-op; commit raising → swallowed, command still succeeds.
- Bead-handler tests (extend the existing split SDD/bead handler suites): `sase bead close` on a `separate_repo` store
  leaves the store committed and triggers the push path with a `chore(beads): close …` message; same spot-check for
  create/update/open/rm/dep.
- Finalizer tests: dirty `separate_repo` store → committed+pushed even when the main repo is clean;
  `in_tree`/`local`-without-remote → no push; clean store → no-op; commit failure → finalization still succeeds. Verify
  placement before the `is_clean` early return and re-run across passes.
- Run `just check` (after `just install`) before finishing.

## Rollout

Pure additive behavior guarded by existing config. Default `sdd.push_after_commit: "async"` means users see background
pushes after SDD-mutating commands; set `false` to keep commits local-only.

```

```
