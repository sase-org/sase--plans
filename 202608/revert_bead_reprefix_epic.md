---
tier: epic
title: Revert the historical bead re-prefix epic and hand-fix bob-cli
goal:
  Every code, data, and tracking artifact produced by the sase-ei epic is removed from the sase and sase-core
  repositories, and the single project that actually carries a leaked ProjectSpec-key bead prefix, bob-cli, is corrected
  by a one-off manual rename instead of by shipping a general migration feature.
phases:
  - id: sase-revert
    title: Revert the sase-repo bead re-prefix surface
    depends_on: []
    size: medium
    description:
      "sase-revert: revert the three sase-ei commits in the sase repo, keep the unrelated forward mint guard, resolve
      the three known conflict files, and prove the reverted tree still builds, lints, and passes the bead/agents_sync
      suites."
  - id: core-revert
    title: Remove the Rust alias and re-prefix primitives
    depends_on:
      - sase-revert
    size: large
    description:
      "core-revert: delete the sase-core bead re-prefix module, alias config field, and PyO3 migration bindings, restore
      pre-epic bead ID resolution semantics without regressing the retained single-pass detail read, and cut a patch
      release."
  - id: epic-retire
    title: Retire the sase-ei plans, beads, and store residue
    depends_on:
      - sase-revert
    size: medium
    description:
      "epic-retire: remove the epic and phase plan files from the local and sidecar plan stores, drop the id_aliases
      residue from the sase bead store config, and close sase-ei and its unfinished phases as canceled with an auditable
      reason."
  - id: bob-cli-reprefix
    title: Hand-fix the bob-cli bead and agent identities
    depends_on:
      - sase-revert
    size: large
    description:
      "bob-cli-reprefix: rename the thirteen leaked-prefix bob-cli beads and their derived agent identities to a
      collision-free bob-cli prefix with a one-off throwaway script, rewriting the bead store, plan refs, agent
      artifacts, registries, chats, and the agents sidecar while leaving published commit history untouched."
  - id: final-audit
    title: Audit that the epic left nothing behind
    depends_on:
      - core-revert
      - epic-retire
      - bob-cli-reprefix
    size: medium
    description:
      "final-audit: run the full sase and sase-core verification gates, confirm no re-prefix code surface or
      leaked-prefix bead token survives outside immutable git history, and record the outcome on the tracking bead."
proposed_by: bbugyi200.athena.sy
create_time: 2026-08-03 10:54:49
status: wip
---

# Plan: Revert the historical bead re-prefix epic and hand-fix bob-cli

## Goal

Undo everything the `sase-ei` epic ("Safely re-prefix historical bead identities") produced, in both the `sase` and
`sase-core` repositories, and instead correct the one project that actually has the problem — `bob-cli` — with a
one-off, non-shipping rename performed by hand.

The epic set out to build a general, restartable `sase bead migrate-prefix` command with collision-safe mapping,
persistent old-ID aliases, compatibility pages, agent-identity migration, and a multi-repository transaction. That
machinery is being abandoned. A single project needs a single rename; the generalized feature is not worth its cost or
its ongoing surface area.

## What the epic actually landed

Only phases 1-3 produced commits. Phase 4 (migration CLI) and phase 5 (integration/docs) never landed code, so **no
`sase bead migrate-prefix` command exists** (`sase bead --help` has no `migrate` or `prefix` entry) and **no
documentation changes were made** (`docs/beads.md` has no alias or migrate-prefix content).

### sase repository (all three commits are already on `origin/master`; revert, never rewrite)

| Commit      | Bead      | Subject                                                  |
| ----------- | --------- | -------------------------------------------------------- |
| `b763878d3` | sase-ei.1 | `feat(beads): expose prefix migration facade`            |
| `f7e1fe216` | sase-ei.2 | `feat(beads): rewrite historical bead references`        |
| `b4db947d2` | sase-ei.3 | `feat(agent-names): migrate historical agent identities` |

### sase-core repository

| Commit    | Bead      | Subject                                        |
| --------- | --------- | ---------------------------------------------- |
| `0343b6f` | sase-ei.1 | `feat(beads): add prefix migration primitives` |

`0343b6f` shipped in the released `v0.17.15` tag. The release cannot be unpublished; the revert lands as a new patch
release instead.

### Not part of this epic — do not revert

- sase `77ef3953e` `fix(beads): repair stale project-key prefixes before minting`. This is the forward mint guard from
  the separate `202608/bead_prefix_mint_guard.md` plan (agent `bbugyi200.athena.sf`), it carries no `SASE_BEAD` footer,
  and it is what keeps _new_ beads from re-leaking a ProjectSpec key. Keeping it is what makes the one-off bob-cli fix
  durable.
- sase-core `5f39c3d` `perf(bead): add single-pass detail read`, `a6b0d53`, `e153a2e`, and the `v0.17.15` release
  commit.
- sase `7a66461b9` `perf(bead): resolve detail from one core snapshot` and `18d438bc0`
  `test(bead): guard the show speedup's end-to-end budget`, which consume that perf work.

## Observed starting point

### The revert surface in the sase repo is narrow

The three sase-ei commits touched 45 files. Every module they _added_ is referenced only by itself, its own tests, and
the binding validator — nothing in the shipped CLI or TUI imports them, because phase 4 never wired them up:

- `src/sase/core/bead_prefix_migration.py`
- `src/sase/bead/reference_rewriters.py`
- `src/sase/agent/names/_identity_migration*.py` (six modules) plus the `src/sase/agent/names/__init__.py` re-exports

They also _modified_ shared code that must return to its pre-epic form: `src/sase/bead/config.py`,
`src/sase/bead/_project_mutations.py`, `src/sase/bead/_project_queries.py`, `src/sase/bead/cli_pages.py`,
`src/sase/bead_pages/**` (links, refresh, rendering, associations), `src/sase/sdd/plan_header_writes.py`,
`src/sase/agents_sync/**` (publication, rendering, v2_io, v2_manifest_io, v2_models), and `tools/validate_sase_core_rs`.

Only three files were touched again by later commits, so `git revert` should be nearly clean:

| File                                  | Later commits            |
| ------------------------------------- | ------------------------ |
| `src/sase/bead/_project_queries.py`   | `7a66461b9`              |
| `Justfile`                            | `c449ce27c`, `d4be80d3f` |
| `tests/agents_sync/test_rendering.py` | `c449ce27c`              |

### The Rust revert is entangled and needs surgery, not a straight `git revert`

`0343b6f` did two things at once: it added the migration feature _and_ refactored bead ID resolution behind a new
`BeadIdentityResolver` (`crates/sase_core/src/bead/identity.rs`), routing `read.rs`, `history.rs`, `work.rs`,
`mutation.rs`, and `cli.rs` through it.

The later, retained commit `5f39c3d` built `show_issue_detail` / `show_issue_detail_in_issues` /
`issue_ancestors_in_issues` / `resolve_optional_issue_in_issues` **on top of that resolver**. A plain
`git revert 0343b6f` conflicts in `crates/sase_core/src/bead/read.rs`, `mod.rs`, `lib.rs`, and
`crates/sase_core_py/src/lib.rs`, and would delete functions the retained perf work depends on.

### The epic's alias field already leaked into committed data

The sase project's own bead store config now carries the epic's field:

```json
{ "issue_prefix": "sase", "next_counter": 530, "owner": "…", "id_aliases": {} }
```

`BeadConfigWire` has no `deny_unknown_fields`, so a reverted binary silently ignores the stale key and drops it on the
next save. That self-healing is not good enough for a committed sidecar file that 16+ workspace clones read; remove it
explicitly. The `bob-cli` and `actstat` bead configs were never written by the epic and contain no `id_aliases`.

### bob-cli is the only project with a real leaked prefix

- `bob-cli` bead store: `issue_prefix: gh_bobs-org__bob-cli`, `next_counter: 6`, five event streams
  (`gh_bobs-org__bob-cli-1.jsonl` … `-5.jsonl`), thirteen issues in the projection, **all `closed`** (no `claimed` or
  `in_progress` blocker). The `-4` stream ends in `issue_removed`, so it is a valid stream with no projected issue and
  must be carried through the rename anyway.
- `actstat` bead store: `issue_prefix: gh_bbugyi200__actstat` is also leaked, but `next_counter: 1` and there is no
  `events/streams/` directory at all — zero beads exist, so the forward mint guard alone repairs it on first use.
  Nothing to do.
- `sase` bead store: `issue_prefix: sase`, already correct.

bob-cli bead inventory to be renamed:

```text
gh_bobs-org__bob-cli-1     closed task   Remove dead getNonDuplicateCopyablePomodoroBullet…
gh_bobs-org__bob-cli-2     closed plan   Capture sub-bullets onto existing Obsidian tasks
gh_bobs-org__bob-cli-2.1-2.4  closed phase (4 phases)
gh_bobs-org__bob-cli-3     closed task   Fix Pomodoro block-ID usage error wording
gh_bobs-org__bob-cli-4     (removed; stream only)
gh_bobs-org__bob-cli-5     closed plan   Priority bullet property that rolls a scheduled…
gh_bobs-org__bob-cli-5.1-5.5  closed phase (5 phases)
```

### The obvious mapping collides — this is the plan's central finding

The natural rename `gh_bobs-org__bob-cli-N` → `bob-cli-N` is **not safe**. Before the ProjectSpec key leaked, this
project already used the `bob-cli` prefix and reached counter 9, and the derived agent identities from that era are
still published in the bob-cli agents sidecar:

```text
bbugyi200.athena.bob-cli-1 … bob-cli-9      (top-level, all nine counters occupied)
bbugyi200.athena.bob-cli-2.1 … bob-cli-2.5  (collides with today's -2.1 … -2.4)
bbugyi200.athena.bob-cli-5.1 … bob-cli-5.3  (collides with today's -5.1 … -5.3)
```

The current bead store cannot see this history: it was created on 2026-07-28 by
`Import bead state from bobs-org/bob-cli--plans@06c4227c4e25`, which imported an **empty** `issues.jsonl` and reset
`next_counter` to 1. So the bead ID space looks free while the derived agent-name space is not. Renaming into
`bob-cli-1` … `bob-cli-5` would duplicate published agent pages and mis-associate today's beads with June/July agents
that worked entirely different changes.

**Mapping decision.** Continue the project's real counter sequence instead of restarting it. Base36 counters `a`-`e` are
10-14, immediately after the historical maximum of 9, and every target is free in both the agents sidecar and
`~/.sase/agent_name_registry.json`:

| Old bead ID                               | New bead ID               |
| ----------------------------------------- | ------------------------- |
| `gh_bobs-org__bob-cli-1`                  | `bob-cli-a`               |
| `gh_bobs-org__bob-cli-2` (+ `.1`-`.4`)    | `bob-cli-b` (+ `.1`-`.4`) |
| `gh_bobs-org__bob-cli-3`                  | `bob-cli-c`               |
| `gh_bobs-org__bob-cli-4` (removed stream) | `bob-cli-d`               |
| `gh_bobs-org__bob-cli-5` (+ `.1`-`.5`)    | `bob-cli-e` (+ `.1`-`.5`) |

`next_counter` becomes `15`, which also repairs the counter reset the import introduced.

_Rejected alternative:_ keep the numbers (`bob-cli-1` … `bob-cli-5`) and skip the agent rename, leaving today's agents
permanently at `bbugyi200.athena.gh_bobs-org__bob-cli-*`. That preserves nothing worth preserving — the numbers are
opaque base36 counters — and leaves exactly the leaked-key strings the whole exercise is meant to delete. If the owner
prefers numeric continuity over agent-name consistency, this is the single decision to reverse before `bob-cli-reprefix`
starts; nothing else in the plan changes.

### Tracking artifacts left by the epic

- Beads: `sase-ei` (in_progress, epic) with `sase-ei.1`-`.3` closed done and `sase-ei.4`, `sase-ei.5` in_progress. Bead
  pages exist for all six under the beads sidecar `pages/sase-ei/`.
- Plan files in the plans sidecar: `202608/historical_bead_reprefix.md` (epic), `202608/rust_bead_reprefix_primitive.md`
  (phase 1).
- Plan files in the local store `~/.sase/plans/202608/`: the same two plus `historical_agent_identity_migration.md`
  (phase 3, never published) and `bead_prefix_migration_cli.md` (phase 4, never implemented).
- No ChangeSpec in the sase ProjectSpec or its archive references `sase-ei` or any re-prefix work, so there is no
  ChangeSpec cleanup.
- Agent bundles for `bbugyi200.athena.sase-ei.*` are published agent history and stay exactly as they are.
- `sase-ei` carries a `DISCOVERED ISSUE:` note about `just check` failing
  `tests/test_check_sase_core_rs_bindings_tool.py::test_dev_extension_exposes_every_collected_name` because a stale dev
  extension lacked the four new bindings. This revert removes the cause.

## Ordering constraint

`test_dev_extension_exposes_every_collected_name` scans **Python** sources for `require_rust_binding("…")` and asserts
the installed extension provides each name; extra Rust bindings are harmless, missing ones are fatal. Therefore
`sase-revert` (which drops the four `require_rust_binding` call sites and the `tools/validate_sase_core_rs` entries)
must land **before** `core-revert` removes those bindings from Rust. Landing them the other way round breaks
`just check` in every workspace in between.

## Phases

### 1. sase-revert — Revert the sase-repo bead re-prefix surface

Work in the sase repository.

Revert the three commits, newest first, as three separate revert commits:

```bash
git revert --no-commit b4db947d2
git revert --no-commit f7e1fe216
git revert --no-commit b763878d3
```

Commit each one separately through the normal sase commit workflow so each revert is independently bisectable. Do not
squash them into the reverts of unrelated work, and do not touch `77ef3953e`.

Expected conflicts and their resolutions:

- `src/sase/bead/_project_queries.py` — `7a66461b9` reshaped this file to resolve detail from one core snapshot. Keep
  the perf structure; remove only the alias-aware lookup the epic added.
- `Justfile` — keep the timezone-related recipe changes from `c449ce27c` / `d4be80d3f`; remove the recipe additions the
  epic made.
- `tests/agents_sync/test_rendering.py` — keep the timezone assertions; remove the alias-page rendering tests.

Also confirm as part of this phase:

- `tools/validate_sase_core_rs` no longer lists `bead_validate_issue_prefix`, `bead_rewrite_id_tokens`,
  `bead_prefix_migration_preview`, `bead_prefix_migration_apply`.
- `grep -rn 'bead_prefix_migration\|reference_rewriters\|identity_migration\|id_aliases' src tests tools` returns
  nothing.
- The six `src/sase/agent/names/_identity_migration*.py` modules, `src/sase/bead/reference_rewriters.py`,
  `src/sase/core/bead_prefix_migration.py`, and their test files are gone.

Verification: `just install`, then `just lint`, then targeted pytest over `tests/test_bead/`, `tests/agents_sync/`,
`tests/test_core_facade/`, `tests/test_check_sase_core_rs_bindings_tool.py`, and `tests/test_sdd/`. Run `just check`; if
the test phase is killed by the sandbox (exit 144 / SIGTERM under load), say so explicitly and fall back to the static
gates plus the targeted subsets rather than reporting a pass you did not observe.

### 2. core-revert — Remove the Rust alias and re-prefix primitives

Open the linked repository with `sase repo open sase-core -r "<reason>"` before any read or edit, and use only the path
it prints.

Remove the epic's feature surface completely:

- Delete `crates/sase_core/src/bead/reprefix.rs` and its `mod.rs` export.
- Remove `id_aliases` from `BeadConfigWire` and `default_config` in `crates/sase_core/src/bead/config.rs`, including the
  test fixtures that construct it.
- Remove the four PyO3 bindings `bead_validate_issue_prefix`, `bead_rewrite_id_tokens`, `bead_prefix_migration_preview`,
  `bead_prefix_migration_apply` from `crates/sase_core_py/src/lib.rs` and the corresponding `pub use` re-exports in
  `crates/sase_core/src/lib.rs`.

Restore pre-epic ID resolution semantics. A straight `git revert 0343b6f` will conflict and would delete
`show_issue_detail` and friends, which `5f39c3d` added and which sase `7a66461b9` calls — that is not acceptable. Choose
between:

- **(a)** Revert `identity.rs` and the `read.rs`/`history.rs`/`work.rs`/`mutation.rs`/`cli.rs` plumbing to the pre-epic
  form, then reapply `5f39c3d`'s single-pass detail read on top of the restored resolution helpers; or
- **(b)** Keep `BeadIdentityResolver` as a pure shorthand-canonicalization helper with the alias map and all
  alias-chain/cycle handling stripped out, so its constructor takes only the issue list.

Prefer (a) if it lands cleanly, since it removes more epic code; fall back to (b) if reapplying the perf work would
require rewriting it. Either way the contract is the same and must be demonstrated, not assumed: **bead ID resolution
behavior must match `0343b6f^` exactly** for full IDs, suffix shorthand, ambiguous shorthand (which must still fail and
list candidates), and unknown IDs. Write or restore tests that pin those four cases, and state in the phase note which
option was taken and why.

Verification: `cargo fmt --all -- --check`, `cargo clippy --workspace --all-targets -- -D warnings`,
`cargo test --workspace`. Then cut a patch release (`v0.17.16`) so the removal is consumable. The sase pin
`sase-core-rs>=0.17.14,<0.18.0` already admits it and needs no change. Rebuild the extension in the sase workspace with
`just install` and rerun the sase bead suites to confirm the reverted Python and reverted Rust agree.

### 3. epic-retire — Retire the sase-ei plans, beads, and store residue

Three independent cleanups; all are data, not code.

**Plan files.** Remove the epic's plan files from the plans sidecar (`202608/historical_bead_reprefix.md`,
`202608/rust_bead_reprefix_primitive.md`) and from the local plan store `~/.sase/plans/202608/`
(`historical_bead_reprefix.md`, `rust_bead_reprefix_primitive.md`, `historical_agent_identity_migration.md`,
`bead_prefix_migration_cli.md`). Resolve the sidecar path with `sase repo path plans`; never hand-build sidecar paths.
Commit the sidecar removal through the normal sase commit workflow.

**Bead store residue.** Delete the `"id_aliases": {}` key from the sase bead store's `config.json` in the beads sidecar
and commit it. Confirm afterwards that `sase bead list` and `sase bead show sase-ei` still work against the reverted
core.

**Beads.** `sase-ei.1`-`.3` are already closed `done` and stay that way — the work genuinely happened. Append a note to
each recording that its commit was reverted and naming the revert commit. Then close the unfinished phases and the epic:

```bash
sase bead close sase-ei --force -R canceled --reason "<why the epic was abandoned>"
```

`--force` sweeps the unfinished `sase-ei.4` and `sase-ei.5`; a resolution other than `done` is required for `--force`,
and `canceled` is the accurate one. Closing an epic bead directly is normally the land agent's job — this is a
deliberate exception made at the owner's explicit request to abandon the epic, and the close reason must say so. Do not
use `sase bead rm`; the epic's history is worth keeping.

Verification: `sase bead show sase-ei` reports closed with the canceled resolution and no unclosed descendants;
`sase bead doctor` reports no new malformed plan references for the removed plan files.

### 4. bob-cli-reprefix — Hand-fix the bob-cli bead and agent identities

This is a one-off data migration, **not** a product feature. Write a throwaway script outside the sase package (a
scratch file under `/tmp` or an untracked scratch path), run it once, and do not commit it to `src/`. Nothing about this
phase may add a command, a module, or a test to the shipped surface.

Open every bob-cli repository through the skill first —
`sase repo open <name> -p gh_bobs-org__bob-cli -w 0 -r "<reason>"` — and use only the printed paths. Note that `agents`
is ambiguous for this project (there is a state-side sidecar and a workspace-side one); resolve both explicitly.

Apply the mapping from **The obvious mapping collides** above (`1→a`, `2→b`, `3→c`, `4→d`, `5→e`, preserving every
`.<n>` child suffix verbatim), and set `next_counter: 15`.

Preconditions to assert before writing anything:

- Every affected bead is `closed` (all thirteen are today) and no bob-cli agent is live.
- Every target ID and every derived target agent name is free in the bead store, the agents sidecar, and
  `~/.sase/agent_name_registry.json`.
- Every repository to be written is clean, on its expected branch, and has its upstream configured.
- Take a full backup of every target — a `git bundle` or tarball per repository plus a copy of the non-Git state files —
  and record where it is, so the whole phase is reversible.

Targets to rewrite:

1. **bob-cli beads sidecar** — `config.json` (`issue_prefix` → `bob-cli`, `next_counter` → 15, no `id_aliases` key);
   rename the five `events/streams/*.jsonl` files, including the removed `-4` stream; rewrite every ID-bearing field
   inside the event records (creation snapshots, parents, dependencies, removal entries, bead refs) and exact bead-ID
   tokens in stored prose; regenerate `issues.jsonl` from the renamed events and verify the projection is isomorphic to
   the preimage under the mapping (same thirteen issues, same statuses, same order, only IDs changed); rename the
   `pages/` directories and files and regenerate page content.
2. **bob-cli plans sidecar** — the three plan files whose frontmatter names a bead (`bead: gh_bobs-org__bob-cli-2`,
   `bead_id: gh_bobs-org__bob-cli-2`, `bead_id: gh_bobs-org__bob-cli-5`), plus the generated `BEAD` header links in
   those files.
3. **Agent identities** — `~/.sase/agent_name_registry.json` entries and their `canonical_global_name` / `name` /
   `project_name`-adjacent fields; the run artifacts under `~/.sase/projects/gh_bobs-org__bob-cli/artifacts/`; dismissed
   bundles under `~/.sase/dismissed_bundles/`; chat transcripts and the chats catalog under `~/.sase/chats/` (including
   filenames that embed a derived agent name, where the destination filename is free); notifications and prompt history
   that name a renamed agent.
4. **bob-cli agents sidecar** — regenerate bundles, manifests, hood/family snapshots, and indexes from the migrated
   local sources through the normal agent-sync path rather than editing sidecar text by hand. The fourteen
   `bbugyi200.athena.gh_bobs-org__bob-cli-*` agent directories and the one family file
   `bbugyi200.athena.gh_bobs-org__bob-cli-2.land.md` are what move.

Explicitly out of scope: the bob-cli primary repository's commit subjects, bodies, and footers. Those are published,
immutable history. They will keep naming `gh_bobs-org__bob-cli-*` beads and agents, and the resulting hosted links will
404 — that is the accepted cost of dropping the alias machinery, and the phase note must state it plainly rather than
quietly leaving it undiscovered.

Do not use the leaked ProjectSpec key as a blind string replacement anywhere. Match whole bead-ID tokens
(`gh_bobs-org__bob-cli-<base36>(.<decimal>)*`) with boundary awareness, so that the project _key_ itself — which stays
`gh_bobs-org__bob-cli` in `~/.sase/projects/gh_bobs-org__bob-cli/`, in workspace registries, and in chat filename
prefixes — is never rewritten. Only bead IDs and bead-derived agent names change.

Verification: from the bob-cli workspace, `sase bead list --status closed --limit 0` shows the thirteen beads under
their new IDs; `sase bead show bob-cli-b` renders with its four phases and correct lineage; `sase bead doctor` is clean;
the agents sidecar renders the renamed agents with intact neighbor links; and a token audit finds
`gh_bobs-org__bob-cli-<counter>` only inside `.git` history and the primary repo's commit messages.

### 5. final-audit — Audit that the epic left nothing behind

Run the complete gates and prove the revert is total.

- sase: `just install` then `just check`. If the test phase is SIGTERM-killed under sandbox load, report that honestly
  and substitute the static gates plus targeted suites; do not claim a green full suite you did not see.
- sase-core: `cargo fmt --all -- --check`, `cargo clippy --workspace --all-targets -- -D warnings`,
  `cargo test --workspace`.
- `sase validate` and `sase bead doctor` for both the sase and bob-cli projects.
- Audit the sase working tree for surviving epic surface: no `reprefix`, `migrate-prefix`, `id_aliases`,
  `bead_prefix_migration`, `reference_rewriters`, or `identity_migration` identifiers in `src/`, `tests/`, `tools/`, or
  `docs/`; no `id_aliases` key in any bead store `config.json`; the four migration bindings absent from the installed
  `sase_core_rs`.
- Audit that the forward mint guard survived: `77ef3953e`'s behavior is still present and
  `tests/test_bead/test_prefix_mint_guard.py` still passes.
- Record the outcome, including anything deliberately left behind (immutable bob-cli commit footers, the published
  `sase-core v0.17.15` release, the `bbugyi200.athena.sase-ei.*` agent history), as a note on this epic's bead.
