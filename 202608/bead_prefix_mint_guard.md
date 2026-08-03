---
tier: tale
title: Repair a leaked ProjectSpec-key issue prefix before minting top-level beads
goal:
  A bead store whose `issue_prefix` leaked the ProjectSpec key repairs itself to the project's `PROJECT_NAME` the next
  time a top-level bead is minted, so `sase bead work` on an approved epic plan in `bob-cli` creates `bob-cli-6` instead
  of `gh_bobs-org__bob-cli-6`, and reports the repair in its output.
proposed_by: bbugyi200.athena.sf
create_time: 2026-08-03 04:18:59
status: wip
---

- **PROMPT:**
  [prompts/202608/bead_prefix_mint_guard.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/bead_prefix_mint_guard.md)

# Plan: Repair a leaked ProjectSpec-key issue prefix before minting top-level beads

## Observed symptom

Approving an epic in the `bob-cli` project (ProjectSpec key `gh_bobs-org__bob-cli`, `PROJECT_NAME: bob-cli`) still mints
key-prefixed beads, and says nothing about it:

```
❯ sase bead work /home/bryan/.sase/plans/202608/priority_property.md -y
✓ Epic bead       gh_bobs-org__bob-cli-5 — Priority bullet property that rolls a scheduled date
✓ Phase beads     gh_bobs-org__bob-cli-5.1 Config schema for `values: priority` · ...
```

Every downstream artifact inherits the string: agent names, clan name (`gh_bobs-org__bob-cli-5`), TUI rows, and commit
trailers.

## Root cause

This is **not** a regression in prefix derivation. `sase/repos/plans/202607/bead_prefix_project_display_name.md` (commit
`22e78f792`, 2026-07-31) already fixed derivation, and it works today. Verified against the live store:

```
cwd=/home/bryan/projects/github/bobs-org/bob-cli
  infer_project_name_from_cwd()      = 'gh_bobs-org__bob-cli'
  project_display_name_for(key)      = 'bob-cli'
  default_issue_prefix(store root)   = 'bob-cli'          # correct
  stale_key_prefix_report(beads_dir) = ('gh_bobs-org__bob-cli', 'bob-cli')
```

The defect is that the corrected policy is **never consulted again after store creation**:

1. `issue_prefix` is chosen once, at store init, and persisted to the store's `config.json`
   (`src/sase/bead/config.py:61-67`). `load_config` only falls back to `get_default_config` when `config.json` is
   _absent_ (`src/sase/bead/config.py:70-76`), so a stored value wins permanently.
2. `bob-cli`'s store predates the fix. Its `config.json` was imported on 2026-07-28
   (`Import bead state from bobs-org/bob-cli--plans@06c4227c4e25`), three days before `22e78f792` landed, and still
   reads `{"issue_prefix": "gh_bobs-org__bob-cli", "next_counter": 6, ...}`. All 13 beads in that store carry the leaked
   prefix.
3. Every mint reads that stored value: Python's `IdGenerator` (`src/sase/bead/project.py:56-59`) and the Rust core
   (`sase_core`, which reads `config.json` itself) both take `issue_prefix` from config, not from
   `default_issue_prefix`.

So the wrong prefix is durable data, and nothing repairs it in the path that keeps consuming it.

### Why it stayed silent

Detection already exists and is correct — `stale_key_prefix_report` (`src/sase/bead/prefix_policy.py:71-94`) returns
`('gh_bobs-org__bob-cli', 'bob-cli')` for this store. But it is reachable from exactly two places, and neither is on the
path that does the damage:

- a `WARNING:` line appended to `sase bead doctor` output (`src/sase/bead/_project_queries.py:158,169-184`);
- the opt-in `sase bead doctor --fix-issue-prefix` repair (`src/sase/bead/cli_admin.py:168-205`).

`sase bead work` — the command that mints an epic plus five phase beads and immediately commits and pushes the graph —
never asks. The store had a detectable, machine-correctable defect at the exact moment it stamped six permanent
identities, and the user was never told. That gap is what this tale closes.

The prior tale deliberately scoped detection to the doctor because it was reasoning about a _retroactive sweep_ of
unknown stores. That is a fair default for a bulk repair, but not for the mint path: minting is where a wrong prefix
becomes a permanent identity, so it is the one place that must not proceed on a value the system already knows is stale.

### Secondary finding

The prior tale's "Follow-up to file" — a task bead for a full historical bead re-prefix migration — was **never filed**.
`sase bead search` over the sase store finds no such bead. This tale re-files it (see _Follow-up to file_).

## Scope

In scope:

1. Repair a key-leaked `issue_prefix` in place, immediately before minting a **top-level** bead ID.
2. Report the repair in `sase bead work` and `sase bead create` output.
3. Refresh the docs that describe the repair as manual-only.
4. File the missing historical-rename follow-up task bead.

Explicitly out of scope: **renaming bead IDs that already exist**. `bob-cli`'s 13 existing beads keep their
`gh_bobs-org__bob-cli-*` IDs. Those IDs are referenced from canonical event streams, `bead_id` frontmatter in the
separate `bob-cli--plans` sidecar repo, design refs, ChangeSpecs, agent names, chat transcripts, and commit history
(`chore(beads): create gh_bobs-org__bob-cli-3`). Rewriting them needs a rename primitive in `sase-core` plus a
cross-repo reference migration — an epic, not this tale. The forward-only outcome is a mixed-prefix store, which is
verified safe below.

### Why forward-only is safe

`_next_top_level_counter` (`src/sase/bead/_project_store.py:76-82`) returns
`max(self._id_gen.counter, max_top_level_counter(prefix, beads_dir) + 1)`. The repair rewrites `issue_prefix` only and
leaves `next_counter` untouched, so for `bob-cli`: `max_top_level_counter("bob-cli", ...)` is `0` (no `bob-cli-*` beads
exist) giving `1`, but `_id_gen.counter` is `6`. The next bead is `bob-cli-6`. It cannot collide with any
`gh_bobs-org__bob-cli-*` ID, and no counter restarts at 1.

## Implementation

### 1. Extract the repair into `src/sase/bead/prefix_policy.py`

`_repair_issue_prefix` (`src/sase/bead/cli_admin.py:184-198`) already contains the exact write this tale needs. Lift its
body into a reusable function so the doctor and the mint path cannot drift apart:

```python
def repair_stale_key_prefix(beads_dir: Path) -> tuple[str, str] | None:
    """Rewrite a key-leaked issue prefix in place; return (stored, corrected)."""
```

- Return `None` and write nothing when `stale_key_prefix_report(beads_dir)` is `None`.
- Otherwise `load_config`, set `issue_prefix` to the corrected label, `save_config`, and return the report tuple.
- Leave `next_counter` and `owner` untouched — the counter continuity argument above depends on it.
- Do **not** acquire a lock or commit inside this function; both call sites handle that differently (see below).

Rewrite `cli_admin._repair_issue_prefix` to call it inside its existing `bead_store_write_lock` block, keeping the
preview, the confirmation prompt, the post-preview re-check, and the `auto_commit_bead_store` call exactly as they are
today.

### 2. Repair before minting, in `BeadProject.create`

`src/sase/bead/_project_mutations.py:37-85` is the single chokepoint for every top-level mint — `sase bead create`
(`src/sase/bead/cli_crud.py:151`), epic creation (`src/sase/bead/epic_from_plan.py:130`), and the SDD store
materialization/adoption paths all route through it. Patching it once covers them all; patching individual CLI handlers
would not.

In `create`, replace the current guard:

```python
if parent_id is not None:
    parent_id = self.resolve_id(parent_id)
```

with a branch that repairs on the top-level path only:

```python
if parent_id is not None:
    parent_id = self.resolve_id(parent_id)
else:
    self._repair_stale_key_prefix()
```

Gating strictly on `parent_id is None` matters for two reasons: child IDs are `<parent_id>.<N>` and never consult
`issue_prefix`, and `stale_key_prefix_report` does ProjectSpec I/O plus cwd inference, which should not run once per
phase in a five-phase epic.

`_repair_stale_key_prefix` (new, on `BeadProjectStoreMixin` in `src/sase/bead/_project_store.py`) must:

- call `repair_stale_key_prefix(self.beads_dir)`; return immediately on `None`;
- reload the in-memory mirror so the Python `IdGenerator` agrees with what Rust will read — reuse
  `self._refresh_db_from_jsonl()` (`src/sase/bead/_project_store.py:104-116`), which already reloads `self._config` and
  rebuilds `self._id_gen` from it;
- record the result on `self._last_prefix_repair: tuple[str, str] | None` (initialized to `None` in
  `BeadProject.__init__`, `src/sase/bead/project.py:52-59`), exposed as a read-only `last_prefix_repair` property
  alongside `last_mutation_outcome` (`src/sase/bead/project.py:78-81`).

Order matters: the repair must land before `rust_beads.create(...)` is called, because the Rust core reads `config.json`
itself to build the ID.

**Do not wrap this in `bead_store_write_lock`.** That lock is deliberately non-reentrant: `store_git_write_lock` raises
`_StoreWriteLockUsageError` when the same context already holds it (`src/sase/sdd/_git_contention.py:133-138`), and
`create` is normally called from inside `bead_store_mutation`, which already holds it
(`src/sase/bead/cli_common.py:169-186`). Taking it again would turn every top-level create into a hard error. Writing
`config.json` unlocked from inside `BeadProject` is the established pattern — `_save_counter`
(`src/sase/bead/_project_store.py:94-97`) does exactly that. The CLI paths are already serialized by the enclosing
`bead_store_mutation` lock, and its commit covers `beads_dir`, so the rewritten `config.json` rides along in the same
bead-creation commit with no extra commit needed.

### 3. Report the repair

Add to `src/sase/bead/cli_work_from_plan_render.py`, matching the 16-column label field the neighbouring lines use:

```python
def render_prefix_repair(stored: str, corrected: str) -> None:
    Console().print(
        f"[green]✓[/green] Issue prefix    {stored} → {corrected} "
        "(repaired from ProjectSpec key; existing bead IDs keep the old prefix)"
    )
```

Call it from `src/sase/bead/epic_from_plan.py` immediately after the epic `proj.create(...)` at line 130, guarded on
`proj.last_prefix_repair`. Calling it there rather than threading the value through `_EpicFromPlanResult` and
`PlanFileWorkResult` into `render_created_beads` keeps the change to one line at the site that already has `proj` in
scope, and still lands the output in the right place: `render_created_beads` runs later, from
`src/sase/bead/cli_work_from_plan.py:418`, so the new line prints just above `✓ Epic bead`, in chronological order.

Also print a one-line notice in the `sase bead create` handler (`src/sase/bead/cli_crud.py:151`) when
`proj.last_prefix_repair` is set, so an interactive top-level create is equally honest.

The resume path (`src/sase/bead/cli_work_from_plan_resume.py:96`) re-renders an already-created epic and mints nothing,
so it needs no change.

### 4. Docs

- `docs/beads.md:716-718` currently says to _use_ `sase bead doctor --fix-issue-prefix` to forward-repair a pre-fix
  store. Update it: the repair now happens automatically the next time the store mints a top-level bead, and the doctor
  flag remains for repairing a store on demand, before its next bead. Keep the forward-only caveat.
- `docs/beads.md:673,681` and the `sase bead doctor` section: note that the flag is no longer the only path.
- The quick-start block in `src/sase/bead/cli_admin.py:419` keeps its existing line unchanged.

No `sase/memory/*.md` file documents the prefix policy (checked), so no memory edits are needed — and none may be made
without explicit user permission.

## Tests

Extend `tests/test_bead/test_prefix_policy.py`, whose fixtures already use the `gh_bobs-org__bob-cli` / `bob-cli` pair:

- `repair_stale_key_prefix` rewrites `issue_prefix`, preserves `next_counter` and `owner`, and returns the tuple.
- It returns `None` and leaves `config.json` byte-identical when the prefix is a deliberate custom value (`beads`), when
  it is already the project name, and when no project can be inferred.

New coverage for the mint path (extend `tests/test_bead/test_cli_doctor.py` or add a focused module):

- **The regression test for this bug.** A store with `issue_prefix: gh_bobs-org__bob-cli`, `next_counter: 6`, and
  existing `gh_bobs-org__bob-cli-1..5` beads: `proj.create(...)` with `parent_id=None` returns id `bob-cli-6`, rewrites
  `config.json`, leaves every existing bead ID untouched, and sets `last_prefix_repair`.
- Creating a **child** bead (`parent_id` set) on the same unrepaired store does not rewrite `config.json` and leaves
  `last_prefix_repair` as `None` — the phase path must stay inert.
- A store with a deliberate custom prefix is untouched by `create`, and the minted ID keeps that prefix.
- After an auto-repair, the `sase bead doctor` stale-prefix `WARNING:` is gone.
- An end-to-end check through the `sase bead work` from-plan path in a project whose key differs from its
  `PROJECT_NAME`: the epic bead is `<PROJECT_NAME>-<n>` and the `✓ Issue prefix` line is emitted. This is the exact
  scenario from the bug report.

## Verification

- `just install` first — ephemeral workspace directories may carry stale dependencies — then `just check`.
- If the full suite is killed by the environment rather than failing (exit 144), fall back to targeted subsets:
  `pytest tests/test_bead/test_prefix_policy.py tests/test_bead/test_cli_doctor.py tests/test_bead/test_config.py`, plus
  `just lint`.
- Manual sanity check in a scratch project whose key and `PROJECT_NAME` differ: create a store with a key-shaped prefix,
  run `sase bead create`, and confirm the ID uses the project name, `config.json` was rewritten, the notice printed, and
  the store is git-clean afterwards.

## Follow-up to file

Use `/sase_new_task` to file the historical bead re-prefix migration task the prior tale promised but never created: a
rename primitive in `sase-core` plus rewriting `bead_id` plan frontmatter, design refs, ChangeSpec links, artifact refs,
and agent-name registry entries, so a store like `bob-cli`'s can be made uniformly `bob-cli-*` instead of mixed. Note in
the task that this tale's forward-only repair leaves `bob-cli` with beads 1–5 on the old prefix and 6+ on the new one.
