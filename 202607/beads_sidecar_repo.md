---
tier: epic
title: Split bead state into a dedicated beads sidecar repository
goal: 'Every SASE-managed project stores bead state in its own auto-cloned `<project>--beads`
  sidecar repository, seeded by `sase repo init` with a generated README and infographic,
  lazily cloned by `sase bead` when absent, and adopted from the plans sidecar by
  a reversible migration that has been run against every currently enabled project.

  '
phases:
- id: core
  title: Rust core bead-store path recognition
  depends_on: []
  size: small
  description: 'core: teach sase_core''s bead-store path heuristics about the `sase/repos/beads`
    sidecar root so design/plan reference resolution keeps working once bead state
    leaves the plans repo.'
- id: guide
  title: Beads sidecar README and infographic
  depends_on: []
  size: medium
  description: 'guide: author the generated beads sidecar README template plus its
    GPT-image 1600x900 directory-map infographic and prompt record, and split the
    guide-kind constant from the legacy directory-README constant.'
- id: record
  title: Schema v3 store record and beads kind resolution
  depends_on: []
  size: medium
  description: 'record: add an optional `beads` sidecar to the SDD store record at
    schema version 3, resolve `kind_root("beads")` and a per-kind repo root from it,
    and keep schema-2 records resolving bead state to the plans sidecar unchanged.'
- id: config
  title: Beads sidecar registration and inventory
  depends_on:
  - record
  size: medium
  description: 'config: register `beads` as a managed sidecar role across defaults,
    identity, clone-dir mapping, repo inventory, `sase repo path`, doctor storage
    checks, and the config schema, gating auto-clone on a recorded beads sidecar.'
- id: rootstore
  title: Repo-root bead store layout
  depends_on:
  - core
  - record
  size: medium
  description: 'rootstore: support a bead store whose files live at the repository
    root, covering the beads dirname constant, gitignore patterns, conflict-resolver
    prefix handling, bead location resolution, and the project-root and bead-directory
    heuristics.'
- id: commit
  title: Commit, push, lock, and attribution routing
  depends_on:
  - record
  - rootstore
  size: medium
  description: 'commit: route bead commits, pushes, locks, health checks, agent env
    vars, and commit attribution to the beads repository instead of the plans repository.'
- id: lazyclone
  title: Lazy beads sidecar materialization
  depends_on:
  - config
  - rootstore
  size: medium
  description: 'lazyclone: make `sase bead` clone the beads sidecar on demand when
    `sase/repos/beads` is missing, mirroring how `sase repo open` and `--ensure` materialize
    sidecars.'
- id: init
  title: Beads sidecar initialization and adoption transaction
  depends_on:
  - guide
  - config
  - rootstore
  - commit
  size: medium
  description: 'init: create, seed, and record the beads sidecar from `sase repo init`,
    including the rerunnable record-last transaction that adopts existing bead state
    out of the plans sidecar.'
- id: docs
  title: Documentation refresh
  depends_on:
  - init
  size: small
  description: 'docs: update SDD storage, beads, and init documentation plus the plans
    sidecar README for the new beads repository, its layout, and the adoption flow.'
- id: migrate
  title: Migrate the enabled projects
  depends_on:
  - init
  - commit
  - lazyclone
  - docs
  size: small
  description: 'migrate: run and verify the adoption against the sase, actstat, and
    bob-cli projects, then confirm bead reads, writes, and workspace clones against
    the new sidecar.'
create_time: 2026-07-27 14:42:01
status: done
bead_id: sase-a8
---

- **PROMPT:** [202607/prompts/beads_sidecar_repo.md](prompts/beads_sidecar_repo.md)

# Plan: Split bead state into a dedicated `beads` sidecar repository

## Motivation

Bead state currently lives inside the plans sidecar (`<workspace>/sase/repos/plans/beads/`). Prior research recorded in
the `mg.cld` agent transcript established four concrete problems with that arrangement:

1. **Fault isolation.** `require_sdd_repository_health` fails closed on a `rebase-merge`/`MERGE_HEAD` at the repository
   root and gates every SDD commit and plan acceptance. `_resolve_bead_conflicts` refuses to auto-resolve when any
   non-bead path is conflicted. A wedged bead rebase therefore blocks plan commits and epic approval.
2. **Lock contention.** `store_git_write_lock` is keyed on the repository root, so every plan write serializes behind
   the hot bead writer.
3. **Retention divergence.** Bead churn dominates the plans pack (roughly 43 of 60 MiB), and it duplicates the event
   store, which is the actual history source for `sase bead history`. The two stores cannot be pruned independently
   while they share a repository.
4. **Optionality.** Beads (and epics) cannot be made opt-out while their state is welded into the plans repository.

This epic delivers the split. It explicitly does **not** deliver the beads opt-out: 61 modules outside `src/sase/bead/`
import `sase.bead`, and making the codebase tolerate an absent bead store is separate, larger work. The split is a
precondition for it, not the feature itself.

## Design

### Repository layout

Each managed project gains a public `<owner>/<repo>--beads` sidecar, cloned to `<workspace>/sase/repos/beads/` with
`auto_clone: true`, exactly like `plans`. **Bead state lives at the repository root**, matching how the plans and
research sidecars keep their `<YYYYMM>/` directories at their roots:

```text
<owner>/<repo>--beads/
├── README.md               # generated guide (phase: guide)
├── assets/
│   └── beads-directory-map.png
├── .gitignore              # beads.db, beads.db-shm, beads.db-wal
├── config.json             # bead store config
├── metadata.json           # bead store metadata
├── issues.jsonl            # generated compatibility projection
└── events/
    ├── manifest.json
    └── streams/*.jsonl     # canonical append-only event store
```

Root placement is both what was asked for and the better design. `sase_core`'s bead-store heuristics match on the
_trailing path components_ of the bead directory; `.../sase/repos/beads` is a clean new shape that sits exactly parallel
to the existing `.../sase/repos/plans` plans root, whereas nesting would produce a confusing `sase/repos/beads/beads`.

The cost is that `beads_dir == repo_root` for this layout, which touches five places that assume a non-empty
subdirectory name. Phase `rootstore` handles all of them. Both `pathlib` and Rust's `Path::components()` normalize away
a `.` component, so a `"."` dirname is safe on both sides of the binding.

### The store record is the switch

`.sase/sdd-store.json` remains the single authority. Today it is schema version 2 with `sidecars: {plans, research}`.
This epic adds an **optional** `beads` entry and bumps the maximum supported schema version to 3:

- A record that carries `sidecars.beads` is written at `schema_version: 3` and resolves bead state to the beads sidecar.
- A record without it stays at `schema_version: 2` and resolves bead state to `<plans>/beads`, byte-for-byte as today.

Two consequences follow, and both are load-bearing:

- **The migration is reversible until its final step.** Everything before the record write leaves an unreferenced (but
  pushed and complete) beads repository and a fully working project.
- **Un-migrated projects never see a schema-3 record**, so an older `sase` install keeps working against them. A
  schema-3 record _is_ rejected by older installs with the existing "upgrade sase" error, so the new build must be
  installed on every machine that touches a project before that project is migrated.

`SddStoreRecord.sidecar_for_kind(kind)` becomes an honest lookup that returns `self.beads` for `"beads"` — today it
returns `self.plans`, and no caller passes `"beads"`, so this is safe. The plans fallback moves into `SddStore`
construction, where it belongs, so that "is a beads sidecar recorded?" stays answerable.

### Adoption transaction

`sase repo init` performs the move, rerunnable and idempotent, in this order:

1. Create or adopt `<owner>/<repo>--beads` and clone it to `sase/repos/beads`.
2. Seed `README.md`, `assets/beads-directory-map.png`, and `.gitignore`.
3. If the beads clone has no bead state and the plans clone has `beads/`, copy the bead files to the beads clone root,
   commit as `Import bead state from <plans-repo>@<sha>`, and **push**.
4. Write the schema-3 store record. _(This is the switch. Bead commands now resolve to the beads sidecar.)_
5. Commit and push the plans-side removal of `beads/` and its `beads/*.db` gitignore lines.

Bead history is **not** filtered out of the plans repository. Its git history stays as an archive, which is also why
step 3 seeds rather than rewrites; the event store is the real history source, so the 43 MiB of bead churn is not worth
recreating in the new remote. Failure before step 4 leaves the project on the plans store; failure at step 5 leaves a
harmless duplicate that the next run cleans up.

### Auto-clone gating

`inject_default_linked_repos` will inject a `beads` sidecar for every managed project, including ones that have not been
migrated yet. Their remote does not exist, and an eager `auto_clone: true` would make workspace preparation try to clone
it and fail. **A `beads` sidecar record is only reported with `auto_clone: true` once the store record actually names
it.** Before that it is inventory-visible but not materialized.

## Phases

### Rust core bead-store path recognition

Work in the `sase-core` linked repo (open it with `/sase_repo`); this is core backend behavior per the repository's Rust
boundary rule, so it lands there and not in a Python shim.

`crates/sase_core/src/bead/cli.rs` resolves plan roots and a storage root by matching the reversed trailing components
of the bead directory. `design_plan_roots` and `design_storage_root` both currently recognize three shapes:

| Reversed components                | Meaning                       |
| ---------------------------------- | ----------------------------- |
| `["beads","plans","repos","sase"]` | split sidecar (bead-in-plans) |
| `["beads","sdd",".sase"]`          | local / legacy separate repo  |
| `["beads","sdd"]`                  | in-tree                       |

Add a fourth, checked **before** the existing sidecar shape so it cannot be shadowed:

- `["beads","repos","sase"]` — the dedicated beads sidecar root.
  - `design_storage_root` returns `beads_dir.ancestors().nth(3)` (the workspace root).
  - `design_plan_roots` returns the sibling plans clone: `beads_dir.parent().join("plans")`.

Keep the existing `["beads","plans","repos","sase"]` arm; un-migrated projects still use it.

Verify `bead::mutation::init_store` tolerates `beads_dirname == "."`: `root_dir.join(".")` yields `root/.`, which
`create_dir_all` and the subsequent `join` calls handle, and `Path::components()` normalizes the `.` away so the new
shape match still fires. Add a Rust unit test for each of the two functions covering the new shape, and one covering
`init_store(root, ".")` writing `config.json`, `issues.jsonl`, and `beads.db` directly into `root`.

Land this in `sase-core` first; the Python phases depend on the published behavior but not on the binding signature,
which is unchanged.

### Beads sidecar README and infographic

Produce the generated guide bundle for the new role, following the plans/research precedent exactly.

**Constant split (do this first, it is a correctness fix, not cosmetics).** `SDD_SIDECAR_KINDS` in
`src/sase/sdd/_init_files.py` is used for two different things: choosing which roles get an illustrated sidecar guide,
and generating `sdd/<kind>/README.md` directory READMEs for the legacy in-tree/local layout. Adding `beads` to the
single constant would write a `README.md` _inside_ legacy bead stores. Split it:

- `SDD_SIDECAR_GUIDE_KINDS = ("plans", "research", "beads")` — drives `expected_sdd_sidecar_files`,
  `_read_sdd_sidecar_directory_map_bytes`, and `_validate_sidecar_kind`.
- `SDD_SIDECAR_KINDS = ("plans", "research")` — unchanged, keeps driving `expected_sdd_directory_readmes`.

Re-export both from `src/sase/sdd/files.py`.

**Files to add.**

- `src/sase/sdd/templates/sidecar-beads-README.md`. Mirror the tone and structure of `sidecar-plans-README.md`. It must
  cover: that this public sidecar stores durable bead state for its SASE-managed source repository and is auto-cloned
  into each workspace; the root layout table from the Design section above; that `events/**` is the canonical
  append-only source of truth while `issues.jsonl` is a generated compatibility projection and `beads.db*` is a
  gitignored local SQLite cache; and the commands `sase bead list`, `sase bead ready`, `sase bead show`,
  `sase bead history`, `sase bead work`, and `sase repo path beads`. Embed the infographic as
  `![Beads directory map](assets/beads-directory-map.png)`.
- `src/sase/sdd/assets/beads-directory-map.png` — 1600×900, 8-bit sRGB.
- `src/sase/sdd/assets/beads-directory-map.png.prompt.md` — the provenance record.
- `SDD_SIDECAR_DIRECTORY_MAP_FILENAMES["beads"] = "beads-directory-map.png"`.

**Infographic pipeline.** Follow `plans-directory-map.png.prompt.md` verbatim as the process template; that file
documents the exact four-step method and its ImageMagick equivalent. This phase needs an agent with GPT image generation
available.

1. Generate a **text-free** structural base with the image tool. The prompt must forbid all readable words, letters,
   numbers, pseudo-text, logos, and watermarks, and must request the same visual language as the existing maps: warm
   off-white technical-documentation canvas with subtle paper grain, crisp flat vector-like architecture diagram, thin
   dark-slate strokes, small-radius rounded panels, soft shadows, restrained teal/blue/green/amber/slate accents.
   Composition: a left-to-right flow from a blank command/input card into a large central repository container; inside
   it, an append-only event-stream stack feeding a derived projection document and a faded local cache chip; at right, a
   fan-out of workspace containers each receiving a clone; along the bottom, an automation rail with commit and push
   nodes returning into the repository; a separate, clearly _disconnected_ sibling repository container at upper right
   to communicate isolation from the plans repository. Leave large blank label zones in every card.
2. Resize the base to exactly 1600×900.
3. Composite a transparent 1600×900 SVG label overlay using DejaVu Sans Bold/Book and DejaVu Sans Mono Bold/Book, dark
   slate `#17243a`, secondary slate `#536175`, opaque white label panels, and `#d4dbe5` borders. Every visible character
   comes from this overlay, never from the model. Labels: title `BEADS SIDECAR`; subtitle
   `Append-only bead state — isolated, auto-cloned everywhere`; repository `BEADS REPOSITORY`, `sase-org/sase--beads`,
   `public linked sidecar`; contents `EVENT STORE`, `events/streams/*.jsonl`, `append-only source of truth`,
   `PROJECTION`, `issues.jsonl`, `generated`, `LOCAL CACHE`, `beads.db (gitignored)`; clone behavior `AUTO-CLONE`,
   `auto_clone: true`, `EVERY WORKSPACE`, `under sase/repos/beads`; isolation `PLANS REPOSITORY`,
   `sase-org/sase--plans`, `separate repo, separate lock`; rail `SDD machinery`, `commit`, `push`.
4. Strip metadata, then inspect the full-size raster **and** a 900px-wide reduction. Confirm `auto_clone: true`,
   `events/streams/*.jsonl`, `issues.jsonl`, and `sase-org/sase--beads` are legible at 900px.

Write `beads-directory-map.png.prompt.md` with the same section structure as the plans prompt file: Target (path, size,
intended use, alt text), Final GPT Image Prompt, Deterministic Labels, Post-Processing, and the equivalent ImageMagick
pipeline. State plainly that no model-rendered text is present in the final raster.

Extend `tests/sdd_store/test_sidecar_init_files.py` to cover the `beads` kind, and add a check that
`expected_sdd_directory_readmes` still returns exactly the plans and research entries.

### Schema v3 store record and beads kind resolution

Make the store record able to name a beads sidecar, and make store resolution honor it.

**`src/sase/sdd/_store_types.py`**

- Add `beads: SddSidecar | None = None` to `SddStoreRecord`.
- `sidecar_for_kind` becomes a plain lookup: `plans` → `self.plans`, `research` → `self.research`, `beads` →
  `self.beads`; unknown kinds still raise. Delete the `kind in {"plans", "beads"}` special case.
- Add `SddStoreRecord.has_split_beads` returning `self.beads is not None`.
- Add `beads_dir: Path | None = None` to `SddStore`.
- `SddStore.kind_root("beads")` returns `self.beads_dir` when it is set (the beads sidecar root **is** the bead
  directory), and otherwise keeps returning `self.sdd_dir / "beads"` for sidecar storage and `self.sdd_dir / "beads"`
  for every other storage mode. This is where the plans fallback lives now.
- Add `SddStore.repo_root_for_kind(kind) -> Path` returning the owning git repository root for a kind: the beads sidecar
  root for `beads` when split, `research_dir` for `research`, otherwise `repo_root`. Phase `commit` uses it.

**`src/sase/sdd/_store_records.py`**

- `_MAX_SUPPORTED_SCHEMA_VERSION = 3`.
- `_sidecars_from_raw` also coerces `sidecars.beads` through `_coerce_sidecar` (so the remote policy check applies).
- `_record_to_json` emits `sidecars.beads` only when present, and derives `schema_version` from content: 3 when a beads
  sidecar is recorded, otherwise 2. Do not blanket-bump un-migrated projects.
- `_load_sdd_store_record` and `_coerce_sdd_store_record`: sidecar storage still requires `plans` and `research`; a
  `beads` entry requires `schema_version >= 3`. `_is_materialized_record` is unchanged — a beads sidecar is optional and
  its absence must not make a record non-materialized.

**`src/sase/sdd/_store_resolution.py`**

- In `resolve_sdd_store`, when the record has a beads sidecar, set
  `beads_dir = Path(sidecar_repo_clone_dir(workspace_dir, "beads"))`. Leave it `None` otherwise.

**Tests.** Round-trip a v3 record; assert a v2 record loads unchanged and still resolves `kind_root("beads")` to
`<plans>/beads`; assert a record with `sidecars.beads` at `schema_version: 2` is rejected; assert writing a record with
no beads sidecar still produces `schema_version: 2`; assert `repo_root_for_kind` for all three kinds in both layouts.

This phase is deliberately behavior-preserving on its own: nothing writes a v3 record yet.

### Beads sidecar registration and inventory

Register `beads` as a first-class managed sidecar role everywhere `plans` and `research` are enumerated.

- `src/sase/_linked_repo_config.py`: add
  `DEFAULT_BEADS_DESCRIPTION = "Durable SASE bead state: the append-only event store and its projections."` Add
  `("beads", f"{project_name}--beads", DEFAULT_BEADS_DESCRIPTION, True)` to the `defaults` tuple in
  `inject_default_linked_repos`.
- `src/sase/_linked_repo_identity.py`: `_store_sidecar_identity` accepts `beads` in its role set.
- `src/sase/_linked_repo_paths.py`: `_sdd_sidecar_repo_dirnames` iterates `("plans", "research", "beads")` for both the
  store-record mapping and the project-name fallback.
- `src/sase/repo_inventory.py`: iterate `("plans", "research", "beads")`; select `DEFAULT_BEADS_DESCRIPTION` for the new
  role. **Auto-clone gating:** the default for `auto_clone` becomes `kind in {"plans", "beads"}`, but the beads record
  is only emitted at all when `store_record.sidecar_for_kind("beads")` is not `None`, so an un-migrated project never
  advertises a beads clone target. The config-entry loop must not synthesize a materializable beads record either — the
  injected default entry is reported with `auto_clone=False` until the store record names it.
- `src/sase/sdd/_sidecar_init.py`: `SPLIT_SIDECAR_KINDS = ("plans", "research", "beads")`.
- `src/sase/main/_repo_init_config.py`: add a `beads` entry (`name: beads`, `auto_clone: true`) to the `entries` map in
  `explicit_sidecar_config_update`; `materialized_compatibility_roles` iterates all three roles.
- `src/sase/main/repo_handler_path.py`: `sase repo path beads` and `sase repo path beads --ensure` work — extend the
  `{"plans", "research"}` legacy-fallback set to include `beads`.
- `src/sase/doctor/checks_config_sdd.py`: `_regressed_split_sidecar_paths` currently detects a split layout by testing
  `(plans / "beads").is_dir()`. Make it tolerate a migrated project by also accepting a `beads` clone directory.
- `src/sase/config/sase.schema.json`: update the `default_linked_repos` description (currently "the default plans and
  intrinsically hidden agents sidecar repositories") to name beads, and the `repos.sidecar` description if it enumerates
  roles. `src/sase/default_config.yml`: update the header comment at lines 11–12 accordingly.
- `sase/sase.yml`: add `- name: beads` with `auto_clone: true` to `repos.sidecar`.

Tests: extend `tests/main/test_repo_init_plan.py` and the repo-inventory tests for the three-role set, and add a test
asserting an un-migrated (schema-2) project reports no materializable beads sidecar.

### Repo-root bead store layout

Support a bead store whose files sit at the repository root.

- `src/sase/bead/project.py`: add `BEADS_DIRNAME_ROOT = "."` with a docstring explaining that it makes
  `beads_dir == root_dir` for the dedicated beads sidecar. `BeadProject.__init__` needs no change (`Path(x) / "."`
  normalizes to `x`), but assert that in a test.
- `src/sase/sdd/_bead_ignore.py`: parameterize the patterns. Add
  `bead_store_gitignore_patterns(prefix: str) -> tuple[str, ...]` producing `beads/beads.db…` for the plans-embedded
  layout and bare `beads.db…` for the root layout, and give `ensure_bead_store_gitignore` a `prefix: str = "beads"`
  keyword. Callers that target the beads sidecar pass `prefix=""`.
- `src/sase/bead/conflict_resolver.py`:
  - `resolve_beads_dir` accepts `BEADS_DIRNAME_ROOT` as a canonical relpath. `Path.relative_to` yields `Path(".")`, so
    normalize `"."` to an empty prefix.
  - Introduce `_BEAD_STORE_ENTRIES = frozenset({"events", "issues.jsonl", "metadata.json", "config.json"})`.
  - `_is_bead_path(path, prefix)`: with a non-empty prefix, unchanged. With an empty prefix, a path is a bead path only
    when its first component is in `_BEAD_STORE_ENTRIES`. This is what keeps a conflicted `README.md`, `assets/…`, or
    `.gitignore` correctly classified as a non-bead conflict rather than being swept into the semantic merge.
  - `_is_event_stream_path` and `_is_mergeable_bead_path` build their comparison paths through a small
    `_store_path(prefix, rest)` helper that omits the separator when the prefix is empty.
- `src/sase/bead/cli_common.py`: in `resolve_beads_location`, when `store.storage == SDD_STORAGE_SIDECAR_REPOS` and the
  store has a `beads_dir`, return a `_BeadsLocation(root=<beads sidecar>, beads_dirname=BEADS_DIRNAME_ROOT)`; otherwise
  keep today's `store.kind_root("plans")` + `BEADS_DIRNAME_NON_VC`. `_BeadsLocation.beads_dir` already derives
  correctly. Apply the same branch to `_resolve_checkout_record_beads_location` so a plain checkout with a schema-3
  record reads from the beads clone. `init_beads` grows a `BEADS_DIRNAME_ROOT` arm that inits the git repo, writes the
  root-prefixed gitignore, and calls `BeadProject.init(root, beads_dirname=BEADS_DIRNAME_ROOT)`.
  `_canonical_storage_plan_path` must keep resolving plans through `location.store.kind_root("plans")`, which now points
  at a different repository than the bead store — verify that path still canonicalizes `plans:` references.
- `src/sase/sdd/_link_files.py`: `_looks_like_project_root` returns `False` for `<…>/sase/repos/beads` (extend the
  `{"plans", "research"}` check) and its `(path / "beads").is_dir()` guard must not misfire now that the plans clone no
  longer contains `beads/`.
- `src/sase/bead/sync.py`: confirm `_is_in_tree_beads_dir` stays `False` for a path ending `sase/repos/beads` (it
  requires a trailing `("sdd", "beads")`, so it does) and add a regression test pinning that.
- `src/sase/workflows/commit/commit_hooks.py`: the `has_bead_dir` probe at line 131 checks `cwd/sdd/beads` — leave the
  in-tree behavior alone but make sure it is not the only signal for sidecar projects.
- `src/sase/doctor/checks_beads.py` and `src/sase/agent/bead_display.py`: both walk parents looking for `sdd/beads` or
  `.sase/sdd/beads`. Add the `sase/repos/beads` candidate and the corresponding dirname mapping in
  `bead_display._beads_root_and_dirname`.

Tests: a `BeadProject` round-trip at a root-level store; conflict-resolution unit tests for the empty-prefix classifier
covering a conflicted `events/streams/x.jsonl` (mergeable), a conflicted `README.md` (refused), and a mixed case
(refused).

### Commit, push, lock, and attribution routing

Point every bead write, push, lock, and label at the beads repository.

- `src/sase/sdd/_commit_store.py`:
  - `sdd_commit_targets` becomes a three-way partition. Build the target list from the store's kind roots: paths under
    `research_dir` go to a research-rooted store, paths under `beads_dir` go to a beads-rooted store, everything else
    (including relative paths, which keep their historical plans-rooted interpretation) goes to plans. Preserve the
    existing `paths is None` behavior by returning one target per materialized repository.
  - `push_sdd_store_after_commit` currently pushes `store.kind_root("beads")` regardless of which repository was
    committed — a latent quirk that becomes a real bug once the roots diverge. Push the target store's own root instead;
    `push_bead_work_launch` already derives the git root from the path it is given.
- `src/sase/sdd/env.py`: `SASE_SDD_BEADS_DIR` now resolves to the beads sidecar root for migrated projects. The existing
  `store.kind_root("beads")` call does this once phase `record` lands; add an explicit test for both layouts.
- `src/sase/bead/workspace.py:206`, `src/sase/bead/cli_work_from_plan_store.py`,
  `src/sase/axe/run_agent_runner_bead.py`, and `src/sase/llm_provider/commit_finalizer.py:415` all already go through
  `store.kind_root("beads")` and need no logic change — add coverage confirming they follow the split.
- Attribution and TUI surfaces that enumerate `("plans", "research")` must include `beads` so bead commits are labelled
  against the right repository: `src/sase/workflows/commit/commit_tracking.py:409`,
  `src/sase/llm_provider/commit_finalizer.py:461`, `src/sase/ace/tui/widgets/prompt_panel/_agent_commits.py:300`, and
  `src/sase/ace/tui/widgets/file_panel/_linked_deltas.py:154`.
- Confirm the isolation win in a test: `store_git_write_lock` is keyed on the repository root, so a held lock on the
  beads clone must not block a plans commit, and a simulated wedged `MERGE_HEAD` in the beads clone must not fail
  `require_sdd_repository_health` for the plans clone.

### Lazy beads sidecar materialization

`sase bead` must work when `sase/repos/beads/` does not exist, by cloning it — the same guarantee `sase repo open` and
`sase repo path --ensure` already provide.

The existing mechanism is
`materialize_linked_repo_workspace(primary_dir=…, workspace_dir=…, workspace_num=…, expected_remote_url=…)`, which for a
sidecar delegates to `ensure_sidecar_sdd_clone(target, remote_url, strict=True)`. Reuse it rather than writing a second
clone path.

- Add `ensure_beads_sidecar_clone(workspace_dir, workspace_num) -> Path | None` next to the other store helpers. It
  reads the store record; returns `None` when no beads sidecar is recorded (un-migrated project — nothing to do); and
  otherwise calls `ensure_sidecar_sdd_clone` against `sidecar_repo_clone_dir(workspace, "beads")` with the recorded
  remote. It must be safe to call concurrently — take the same materialization lock the sidecar init path uses.
- Wire it into `resolve_beads_location(materialize=True)`, which is the single funnel `get_project()` already uses when
  the warm location is missing or unusable. `resolved_beads_location_is_usable` must return `False` when the recorded
  beads clone is absent or its origin does not match, which routes the caller into the materializing branch.
- Read-only commands go through `get_read_view()` → `get_project()`, so they materialize too. That is correct: a bead
  read cannot be served from a clone that does not exist.
- Failure must be a clear, actionable error naming the repository and the remote, not a `FileNotFoundError` from
  `BeadProject`.

Tests: with a schema-3 record and the beads clone deleted, `sase bead list` clones and succeeds; with a bad remote it
fails with the actionable message; with a schema-2 record nothing is cloned and behavior is unchanged.

### Beads sidecar initialization and adoption transaction

Make `sase repo init` (and `sase init`, which dispatches to it) create, seed, record, and adopt the beads sidecar.

Registration from phase `config` already makes `configured_sidecar_specs` yield a `beads` spec and
`run_configured_sidecars` create/adopt/clone/seed/push it through the existing `initialize_sidecars` transaction. Three
things remain.

**1. Record the beads sidecar.** `_write_compatibility_store` builds the record from `plans` and `research` only. Add
`beads`, sourced from this run's sidecars or from the existing record, and pass it to `SddStoreRecord`. Keep the
existing invariant that a missing plans or research sidecar aborts the record write; a missing beads sidecar must
instead write the schema-2 record exactly as today. `_existing_compatibility_sidecar` follows `SPLIT_SIDECAR_KINDS`, so
it picks up `beads` automatically.

**2. Seed with the root layout.** In `_seed_sidecars`, the `spec.role == "plans"` branch that calls
`ensure_bead_store_gitignore(root)` becomes role-aware: `plans` keeps its `beads/`-prefixed patterns while a project is
un-migrated, and `beads` gets the root-prefixed patterns from phase `rootstore`. After migration, the plans-side
patterns are removed in step 5 below.

**3. Adopt existing bead state.** Add `adopt_bead_state(plans_root, beads_root) -> AdoptionOutcome` and call it from
`initialize_sidecars` between the seeding step and `_write_compatibility_store`. Ordering and idempotency, exactly:

1. No-op when `plans_root/beads` is absent, or when `beads_root` already contains bead state (`config.json` or
   `events/`). Both make reruns free.
2. Copy every entry of `plans_root/beads/` to `beads_root/`, **excluding** `beads.db`, `beads.db-shm`, and
   `beads.db-wal` — those are the gitignored local cache and are rebuilt on demand. Two of the three projects being
   migrated have only `config.json` + `issues.jsonl` with no `events/` directory; that is a valid, minimal store and
   must copy cleanly.
3. Commit to the beads clone as `Import bead state from <plans-repo>@<short-sha>` (resolve the sha from the plans clone
   `HEAD`) with `auto_commit_type="beads"`, then push. **A failed push aborts the adoption** — the record must not be
   written against bead state that exists only locally.
4. Return to `initialize_sidecars`, which writes the schema-3 record. This is the switch.
5. Only after the record write: `git rm -r --cached`-equivalent removal of `beads/` from the plans clone, drop the
   `beads/beads.db*` lines from the plans `.gitignore`, commit as `Move bead state to the beads sidecar`, and push.
   Failure here is logged as a warning and leaves a duplicate that the next `sase repo init` removes; it must **not**
   fail the command, because the authoritative switch has already happened.

Do **not** filter bead history out of the plans repository. Its history stays as an archive.

Surface the adoption in `sase repo init --check` / `plan_sidecar_actions` as a distinct planned action
(`"adopt bead state from the plans sidecar"`) so a dry run tells the user a data move is pending.

Tests: full transaction against local bare remotes covering a fresh project (no bead state), a migration (bead state in
plans), a rerun after a successful migration (no-op), a rerun after a failure at step 3 (retries cleanly, record still
schema 2), and a failure at step 5 (record is schema 3, warning emitted, next run cleans up).

### Documentation refresh

- `docs/sdd_storage.md`: add `beads` to the kind-resolution table (`beads` → `<workspace>/sase/repos/beads`); correct
  the statement that beads live at `${SASE_SDD_PLANS_DIR}/beads`; document schema version 3, the optional `beads`
  sidecar, the schema-2 fallback, and the adoption transaction; add `sase repo path beads` to the command list; note
  that the plans clone no longer owns bead state.
- `docs/beads.md`: update the storage section (currently "split sidecar storage uses `beads/` at the root of the active
  workspace's auto-cloned `--plans` repository") and the `sase bead init` description; document the root layout and the
  lazy clone behavior.
- `docs/init.md`: document beads sidecar creation and the adoption step.
- `src/sase/sdd/templates/sidecar-plans-README.md`: drop the `beads/` bullet from the directory layout and the
  `sase bead` line from the commands list; point at the beads sidecar instead. Regenerate the plans infographic's prompt
  record only if the image itself is regenerated — otherwise leave the existing asset and note the staleness of its
  `beads/events/**` label in the prompt file.
- Do not touch `sase/memory/*.md`, `AGENTS.md`, or the generated provider shims. `sase/memory/glossary.md` names the
  sidecar repos and `sase/memory/build_and_run.md` references `sdd/beads/`; both will want updating, but memory edits
  require explicit user permission in the conversation that makes them. Report this as a follow-up instead.

### Migrate the enabled projects

Run the adoption against every currently enabled project. As of planning there are exactly three, all on schema-2
`sidecar_repos` records:

| Project   | Primary workspace                               | Bead state in plans clone               |
| --------- | ----------------------------------------------- | --------------------------------------- |
| `sase`    | `/home/bryan/projects/github/sase-org/sase`     | full store (`events/`, `metadata.json`) |
| `actstat` | `/home/bryan/projects/github/bbugyi200/actstat` | minimal (`config.json`, `issues.jsonl`) |
| `bob-cli` | `/home/bryan/projects/github/bobs-org/bob-cli`  | minimal (`config.json`, `issues.jsonl`) |

Re-derive the list with `sase project list -j` rather than trusting this table; it is a planning-time snapshot.

Procedure, per project, from its primary workspace:

1. `sase repo init --check` and read the planned actions. Confirm the beads sidecar creation and the adoption action are
   both listed.
2. `sase repo init`, confirming the interactive creation prompt for `<owner>/<repo>--beads`.
3. Verify: `sase repo path beads` prints the clone; `.sase/sdd-store.json` is `schema_version: 3` with a `beads` entry;
   the beads remote has the imported commit; the plans clone no longer has `beads/`.
4. Verify bead reads and writes: `sase bead list`, then `sase bead show <id>` on a known bead, then a write
   (`sase bead note`) and confirm the commit lands in the beads repository and pushes.
5. Verify lazy clone: delete `sase/repos/beads`, run `sase bead list`, confirm it re-clones and succeeds.

Order matters. Migrate `actstat` and `bob-cli` first — they are low-traffic and their stores are minimal, so they are
the cheap rehearsal. Migrate `sase` last.

**Migrating `sase` is the hazardous one.** It had 25 active claims at planning time, and its own agents run bead
commands continuously from ephemeral workspaces. Before migrating it: confirm no SASE agents are running against the
project, and confirm the new build is installed on every machine that touches it — a schema-3 record is rejected by an
older install with the existing "upgrade sase" error, which would hard-fail bead commands there. Re-check both
immediately before step 2.

Rollback for any project: delete `sidecars.beads` from `.sase/sdd-store.json` and set `schema_version` back to `2`. Bead
state resolves to `<plans>/beads` again. This only works before step 5 of the adoption has removed `beads/` from the
plans clone; after that, restore it from the plans repository's git history (which is why the history is deliberately
preserved).
