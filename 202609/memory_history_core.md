---
tier: epic
title: Memory history semantics, cache, and query bindings in sase-core
goal: 'sase-core can answer, from git alone, what each memory note, web, strand, instruction
  file, and asset was at any committed revision: its subject identity across renames,
  whether a provider shim aliased or diverged, what kind of change it was, who committed
  it, and which memory or config edits landed in the same instruction render. A disposable
  per-scope snapshot makes that answer incremental, and GIL-releasing query bindings
  expose it to Python without Python reimplementing lineage, classification, or diffing.

  '
phases:
- id: subjects
  title: Subject identity, shim aliasing, and the fixture corpus
  depends_on: []
  size: medium
  description: 'subjects: derive note, web, strand, instructions, and asset identity
    from file_history lineages, alias each shim version into its instructions subject
    by blob equality, and build the fixture corpus later phases assert against.'
- id: classify
  title: Version classes, summaries, and commit provenance
  depends_on:
  - subjects
  size: medium
  description: 'classify: assign every committed version a class, a hidden-by-default
    bit, a summary, a sparkline volume, and footer provenance from prose_diff stats
    and the commit message.'
- id: causes-feed
  title: Instruction causes, changesets, and the merged feed
  depends_on:
  - classify
  size: medium
  description: 'causes-feed: attribute instruction versions to co-changed memory,
    config, and renderer paths, then group versions into changesets and a time-merged
    feed.'
- id: cache-queries
  title: Snapshot cache, upstream marker, and query bindings
  depends_on:
  - causes-feed
  size: medium
  description: 'cache-queries: persist the per-scope snapshot, report how far origin
    is ahead, and expose sync, subjects, resolve, timeline, version, compare, and
    feed through GIL-releasing memory_history_ bindings.'
proposed_by: bbugyi200.apollo.sase-1dr.4
parent_bead: sase-1dr.4
create_time: 2026-09-30 20:38:40
status: done
bead_id: sase-1dr.4.1
---

- **PROMPT:** [prompts/202609/memory_history_core.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/memory_history_core.md)
- **PARENT:** [202609/memory_history.md](https://github.com/sase-org/sase--plans/blob/main/202609/memory_history.md)
- **BEAD:** [sase-1dr.4.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1dr/sase-1dr.4.1.md)

# Plan: Memory history semantics, cache, and query bindings in sase-core

## 1. Summary

Phase `memory-history-core` of epic `sase-1dr` (design of record:
`plan:202609/memory_history.md`, especially §3, §4.10, §5.2, §5.3, and §9) builds
`sase_core::memory_history` on top of the `file_history` and `prose_diff` modules that
have already landed. This epic is that phase's implementation plan. Python presentation,
the CLI, the pager, and the `sase-core-revision.txt` ratchet stay in later phases of
`sase-1dr` (`history-cli` onward).

Work only in the linked sase-core checkout from `sase repo open sase-core`. Read that
repo's `AGENTS.md` before editing. Do not edit the sase Python repo.

Where this plan and the parent design name different existing APIs, follow this plan. It
is bound to the code that landed:

- Lineage, incremental git walks, blob reads, path status, and atomic snapshot IO
  already exist as `sase_core::file_history` (`build_index`, `sync_index`, `read_blobs`,
  `path_status`, `persist_index`, `load_index`).
- `compare_prose` returns `ProseStatsWire` (`reflow_only`, `whitespace_only`,
  `frontmatter_only`, word counts) and `type_change` of `"promoted"` or `"demoted"` only
  for `type: reference` ↔ `type: core`.
- `parse_commit_footer` exposes canonical tag keys with the `SASE_` prefix stripped, so
  `SASE_AGENT` and `SASE_BEAD` arrive as `AGENT` and `BEAD`.
- `sase_core::vcs_log::classify_commit_types` returns provenance labels (`manual`,
  `stitch`, `automatic`, …). It does not parse conventional-commit prefixes. Parse
  `feat` / `chore` / `fix` from the subject line locally.

## 2. Boundaries

In scope for every phase:

- A multi-file `crates/sase_core/src/memory_history/` module. `mod.rs` is a facade of
  `mod` and `pub use` lines only, and its first line is a `//!` one-line summary so
  `just modules` lists it. Keep each file at or under 1,500 lines. No `macro_rules!`.
  Free functions over `*Wire` serde structs. Errors are a `thiserror` enum
  `MemoryHistoryError`.
- `pub mod memory_history;` in `crates/sase_core/src/lib.rs`, in alphabetical order
  (between `markdown_link_refs` and `migration`). Do not add items to the root `pub use`
  list.
- Tests beside the code, built on temporary git repos the way `file_history/tests.rs`
  does (`init --initial-branch=master`, fixed `user.name` / `user.email`,
  `commit.gpgsign=false`). Library code calls the `file_history` runner
  (`run_git_checked` / `run_git_unchecked`, `--no-optional-locks`, argv arrays, `--`
  separators). Tests may spawn `git` the same way `file_history/tests.rs` does.
- No new workspace dependencies are expected. If one is added, regenerate hakari the way
  `AGENTS.md` describes. Never edit a crate `version`, a path-dependency pin, or a
  `CHANGELOG.md`.

Out of scope, for every phase:

- Anything under the sase Python tree: facade, wire dataclasses, CLI, vocabulary, pager,
  flags, and `sase-core-revision.txt`. `history-cli` owns the ratchet.
- Strand keyword and alias lookup from frontmatter. Core `resolve` accepts a subject id,
  a repo-relative path (current or historical), or a unique basename. The selector
  grammar (`glossary:stitch`, dates, `--since` parsing) is Python.
- An in-process cache of query results. The only process-local memo is the snapshot
  file's `(path, mtime, len)` parse cache in `cache-queries`. The Python history service
  adds its own memo later.
- Real-repo performance budgets (cold build ≤ 1.5 s, warm step ≤ 30 ms). Those are
  measured by `history-cli` and `launch`. This epic proves behavioral equality on the
  fixture corpus.
- Restoring a version, fetching from a remote, writing a commit-graph, or storing file
  bodies in the snapshot.
- Closing `sase-1dr`, `sase-1dr.4`, or any ancestor. Each phase worker closes only its
  own phase bead. `sase plan propose` parents this epic under the phase that authored
  it; landing the child epic is what closes that phase.
- Creating beads. Record a discovered follow-up as `PROPOSED FOLLOW-UP:` via
  `sase bead note` on the phase bead being closed.

Inner loop for every phase: `just fast`, then `just test -p sase_core memory_history`.
Before closing a phase, run `sase tool run check` in the sase-core checkout (about five
minutes; allow at least ten). A targeted crate test does not replace that gate. A
failure that reproduces on the clean base tree is a `PROPOSED FOLLOW-UP:` note, not a
reason to leave the phase open.

## 3. Shared contracts

The `subjects` phase creates these types in `wire.rs`. Later phases fill fields; they do
not rename them or bump the schema unless a serialized meaning changes. Both version
constants start at `1`:

- `MEMORY_HISTORY_WIRE_SCHEMA_VERSION`
- `CLASSIFIER_VERSION` (bump only when class rules change; it is part of the cache key)

`MemoryHistoryScopeWire` fields, matching the parent design:

| Field               | Meaning                                                                                                                                                      |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `scope_key`         | Caller identity, for example `project:sase` or `home`. Embedded in subject ids.                                                                              |
| `scope_kind`        | `project` or `home` (`snake_case`).                                                                                                                          |
| `repo_root`         | Checkout passed to `file_history`.                                                                                                                           |
| `memory_roots`      | Directory pathspecs, longest-prefix wins. Example: `["sase/memory", "memory"]`.                                                                              |
| `instruction_files` | `{dir, agents_path, shim_paths, template, managed}`.                                                                                                         |
| `generated_notes`   | Repo-relative or memory-root-relative paths. A subject is generated when any of its paths, or that path stripped of a matching memory root, equals an entry. |
| `renderer_prefixes` | Opaque path prefixes. Empty except for the sase repo.                                                                                                        |
| `config_paths`      | Repo-relative files whose co-change marks an instruction version as config-driven.                                                                           |
| `cache_dir`         | Caller-supplied snapshot root. Ignored by identity and classification.                                                                                       |

Pathspecs passed to `file_history` are the memory roots plus every `agents_path` and
shim path, sorted and deduped. They must pass `safe_pathspec` (no globs, no colons, no
`..`). Config paths and renderer prefixes stay out of the walk. Widening the walk with
`--full-diff` is forbidden; cause attribution uses a separate batched diff-tree (§6).

Subject ids, from the lineage's **latest** path. Historical paths are aliases, not extra
subjects:

| Kind         | Id                                | When                                                                                                             |
| ------------ | --------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| instructions | `instructions:{scope_key}/{dir}`  | The lineage's paths include an `agents_path`. `dir` is `.` for the repo root.                                    |
| strand       | `strand:{scope_key}/{web}/{slug}` | Memory-root-relative path is `{web}/{slug}.md` and `{web}.md` is some lineage's latest path under the same root. |
| web          | `web:{scope_key}/{slug}`          | Latest path is `{slug}.md` and some latest path under the same root is `{slug}/{file}.md`.                       |
| note         | `note:{scope_key}/{name}`         | Any other `*.md` or `*.md.tmpl` under a memory root. `name` is the root-relative path without the final suffix.  |
| asset        | `asset:{scope_key}/{path}`        | Anything else under a memory root. Listed only: no `prose_diff`.                                                 |

Shim lineages never become their own subjects. Display name is the filename for a note
or asset, the slug for a web or strand, and `AGENTS.md` for instructions. `generated`
and `managed` and `template` are subject flags. `managed` and `template` are copied from
the matching instruction entry; notes, webs, strands, and assets are `managed: false`
and `template: false`.

`MemoryHistoryClassWire` (`snake_case`), one primary class per committed version. First
match wins:

1. `deleted` — file-history kind is deleted.
2. `created` — file-history kind is created, including a re-add after a gap.
3. `moved` — file-history kind is moved and the blob is unchanged, or prose stats are
   `reflow_only` or `whitespace_only`. `hidden_by_default`.
4. `promoted` / `demoted` — `type_change` is `promoted` or `demoted`.
5. `frontmatter` — `frontmatter_only` and no type promotion or demotion.
6. `reflow` — `reflow_only`. `hidden_by_default`.
7. `whitespace` — `whitespace_only`. `hidden_by_default`.
8. Instruction subject (`managed` is irrelevant until the next step) whose file-history
   kind is not a pure move:
   - `hand_edited` when the instruction entry is `managed: false`.
   - `rendered` when a non-generated note, web, or strand also has a version in this
     commit.
   - `config` when a `config_paths` entry or a `renderer_prefixes` entry matches a path
     from this commit's diff-tree and no memory source matched. The cause record
     distinguishes config paths from renderer paths.
   - `regen_only` when none of those match.
9. `regenerated` — subject is generated.
10. `authored` — every other content edit, including a rename that changes words. The
    rename similarity and both paths stay on the version; the class is not `moved`.

`uncommitted` and `staged` exist on the enum for query-time pseudo-versions only. They
are never stored in the snapshot and never participate in the priority list.
`hidden_by_default` is true only for `moved`, `reflow`, and `whitespace`.

Each committed version also carries:

- ordinal (`1` is oldest across canonical and diverged rows), commit, parent SHAs,
  committer time, author time, author name and email, path, `blob_oid`, `prev_blob_oid`,
  file-history kind, similarity, `gap_before`
- `diverged: bool` and `source_path` (the shim path when the row is a diverged shim
  version; otherwise the path)
- `aliased_paths`: shim paths whose blob at this commit equals this version's blob. A
  matching shim does **not** create its own row
- summary: `section_paths` (deduped hunk breadcrumbs, first-seen order), `words_added`,
  `words_removed`, `frontmatter_phrase` (`"promoted reference → core"` style when
  `type_change` is set; otherwise `key: before → after` joined for remaining frontmatter
  entries), `created_words` (set on `created`), `volume`
- provenance: `agent` and `bead` from tags `AGENT` and `BEAD`, every footer tag as
  `{key, label}`, the commit subject, `conventional_type` (leading `[A-Za-z]+` before
  `:` or `(`, lowercased, else absent), and `commit_types` from `classify_commit_types`
  on `subject + "\n\n" + body` with `is_merge` when there is more than one parent
- `boilerplate`: subject starts with `chore: run sase init` or equals
  `chore: initialize sase memory` after trimming
- `cause`: instruction versions only.
  `{sources: [{subject_id, ordinal}], config_paths, renderer_paths, regen_only}`. Empty
  for every other kind. Present these as related changes, not as proof

Volume is a raw non-negative integer. Log scaling is a renderer concern. Created
versions use the new document's whitespace-delimited word count. Deleted versions use
the previous document's word count. Pure moves, reflow, whitespace, and assets use `0`.
Every other class uses `words_added + words_removed`.

Feed types (filled in `causes-feed`, declared now):

- A changeset is one commit in one scope: commit metadata, provenance, `boilerplate`,
  `regen_only`, `authored` entries, and `consequences`.
- An entry points at `(subject_id, ordinal)` plus class, summary, and path.
- A version is a **consequence** when its subject is generated, or when it is an
  instruction version whose class is `rendered`, `config`, or `regen_only`. Role decides
  folding, not class: a created generated note still folds. `hand_edited` instruction
  versions stay authored. Promotions of ordinary notes stay authored.
- A changeset is `regen_only` when, after the hidden filter, it has no authored entries
  and every remaining consequence is class `regen_only` or `regenerated`.

Query response types (declared now, implemented in `cache-queries`) all carry
`schema_version`. `compare` embeds a `ProseComparisonWire` rather than replacing its
schema.

## 4. Phase: Subject identity, shim aliasing, and the fixture corpus (`subjects`)

Create the module, `wire.rs` with every type in §3, and `subjects.rs` with
`derive_subjects(index, scope) -> Vec<MemoryHistorySubjectWire>`.

Derivation:

1. Ask `file_history` for lineages over the scope pathspecs. Do not reimplement rename
   folding.
2. Claim instruction subjects from lineages that touch an `agents_path`.
3. Fold each shim lineage into that directory's instruction subject.
   - At each shim version's commit, compare its blob to the `AGENTS.md` blob at that
     same commit (`read_blobs`, or the lineage blob when the commit already touched
     `AGENTS.md`).
   - Equal blobs: append the shim path to that `AGENTS.md` version's `aliased_paths`. If
     this commit did not change `AGENTS.md`, attach the alias to the latest `AGENTS.md`
     version whose blob equals the shim, and do not add a row.
   - Unequal blobs, or no `AGENTS.md` blob yet: append a `diverged` version row with the
     shim's own blob, path, and commit. `diverged_count` is the number of such rows.
4. Classify every remaining lineage as web, strand, note, or asset from the latest-path
   rules in §3. Stable ordering: instruction subjects by `dir`, then other subjects by
   id. Version rows within a subject are newest-first, matching `file_history`, with
   ordinals assigned oldest-first. A diverged shim row shares the commit's place;
   tie-break equal committer times by path so a rebuild is deterministic.
5. Build the path-alias map from every historical path, including shim paths, to the
   subject id.

Leave class as a temporary `authored` only when a test needs a filled enum before
`classify` exists — prefer not storing a class until `classify` runs. The wire field can
default to an `unclassified` variant that `classify` overwrites and that `cache-queries`
refuses to persist. Do not run `prose_diff` in this phase.

### Fixture corpus

Put one shared builder in `tests/corpus.rs`, reused by later phases. Fix committer and
author dates with `GIT_AUTHOR_DATE` and `GIT_COMMITTER_DATE` so order does not depend on
the clock. One project repo, initial branch `master`, scope:

- `scope_key: project:fixture`, `scope_kind: project`
- `memory_roots: ["sase/memory", "memory"]`
- instruction entry
  `{dir: ".", agents_path: "AGENTS.md", shim_paths: ["CLAUDE.md"], template: false, managed: true}`
- `generated_notes: ["sase/memory/roster.md"]`
- `renderer_prefixes: ["src/sase/amd/"]`
- `config_paths: ["sase/sase.yml"]`

Commits, oldest first:

1. **Boilerplate init.** Message `chore: initialize sase memory`. Add
   `memory/build_and_run.md` (`type: reference`, a 60-line body using the same shape as
   `file_history/tests.rs`), `memory/policies.md` (`type: reference`),
   `memory/legacy.md` (`type: core`), `memory/scratch.md`, `AGENTS.md`
   (`project agents v1`), and `CLAUDE.md` with different bytes (`claude era`).
2. **Promotion plus a body word change** on `policies.md` (`reference` → `core`).
   Footer: `SASE_AGENT=athena.sase-1au.5` and `SASE_BEAD=sase-1au.5`.
3. **Demotion** of `legacy.md` (`core` → `reference`) with no body change.
4. **Reflow only** on `build_and_run.md`: rewrap one paragraph, same words.
5. **Whitespace only** on `scratch.md`.
6. **Frontmatter only** on `scratch.md`: `title: Old` → `title: New`.
7. **Pure directory rename** `git mv memory sase/memory`. No content edits.
8. **Content rename** `sase/memory/build_and_run.md` → `sase/memory/lint_and_test.md`,
   changing 22 of 60 lines the way `file_history/tests.rs` does. Assert rename
   similarity is in `55..=75`.
9. **Delete** `sase/memory/scratch.md`.
10. **Recreate** `sase/memory/scratch.md` with new body (same subject, gap).
11. **Web.** Add `sase/memory/glossary.md` and `sase/memory/glossary/stitch.md`, then a
    later commit that `git mv`s the strand to `sase/memory/glossary/bead.md` with no
    content change.
12. **Co-rendered note.** One commit edits one word of `lint_and_test.md` and edits
    `AGENTS.md`. Footer `SASE_AGENT=athena.sase-1bc.12` and `SASE_BEAD=sase-1bc.12`.
    Message `feat(memory): edit lint note`.
13. **Config.** One commit edits `sase/sase.yml` and `AGENTS.md` only.
14. **Renderer.** One commit edits `src/sase/amd/render.rs` and `AGENTS.md` only.
15. **Regen-only.** One commit edits `AGENTS.md` only.
16. **Generated consequence.** One commit adds `sase/memory/roster.md` and edits
    `lint_and_test.md`. A later commit edits only `roster.md`.
17. **Shim convergence.** One commit rewrites `CLAUDE.md` to the current `AGENTS.md`
    bytes without editing `AGENTS.md`. A later commit makes `CLAUDE.md` differ again.
18. **Asset.** Add `sase/memory/diagram.png` (non-markdown bytes).
19. **Stays deleted.** Add `sase/memory/gone.md` and delete it in a following commit.

A second, smaller home repo (one note commit dated between two project commits) is built
by the same module for the feed-merge test. It is not required to pass the subject
tests.

### Tests in this phase

- Subject ids after the full history: `note:project:fixture/lint_and_test`,
  `web:project:fixture/glossary`, `strand:project:fixture/glossary/bead`,
  `instructions:project:fixture/.`, `asset:project:fixture/sase/memory/diagram.png`.
- `build_and_run.md` and `memory/build_and_run.md` alias to the lint note.
  `glossary/stitch.md` aliases to the bead strand. The directory move does not split the
  subject. Delete-and-recreate keeps `scratch`'s id and sets `gap_before` on the re-add.
- The hand-written `CLAUDE.md` era is `diverged` with its own blob. The convergence
  commit adds `CLAUDE.md` to `aliased_paths` and does not add a row. The later
  divergence adds another diverged row. `diverged_count` matches the diverged rows.
- The asset is `asset`, not a note. The shim path is not its own subject.
- `generated` is true only for `roster.md`. `managed` is true only for the instructions
  subject.

Do not add Python bindings in this phase.

## 5. Phase: Version classes, summaries, and commit provenance (`classify`)

Add `classify.rs` with a pure function over subjects, blob text, and the scope. Blob
text comes from `read_blobs` (OID LRU is already in `file_history`). Compare each
version to its previous blob through `compare_prose` with `format: markdown` and
`context_lines: 3`. Assets skip `compare_prose`. Missing blobs produce a version that
still classifies from the file-history kind, with empty prose stats and the OID recorded
for the caller; they do not fail the classify call.

Apply the priority list in §3. Set summary, volume, provenance, and `boilerplate`
exactly as specified there. `frontmatter_phrase` for the promotion fixture is
`promoted reference → core`. Demotion is `demoted core → reference`.

### Tests in this phase

On the project corpus, assert the class (and `hidden_by_default`) of:

| Commit              | Subject                | Class                                                       | Hidden |
| ------------------- | ---------------------- | ----------------------------------------------------------- | ------ |
| init                | lint note's oldest row | `created`                                                   | no     |
| init                | instructions           | `created` (the file appeared; later renders use later rows) | no     |
| promotion           | policies               | `promoted`                                                  | no     |
| demotion            | legacy                 | `demoted`                                                   | no     |
| reflow              | lint note              | `reflow`                                                    | yes    |
| whitespace          | scratch                | `whitespace`                                                | yes    |
| title edit          | scratch                | `frontmatter`                                               | no     |
| directory `git mv`  | lint note              | `moved`                                                     | yes    |
| 22-of-60 rename     | lint note              | `authored`                                                  | no     |
| delete              | scratch, then gone     | `deleted`                                                   | no     |
| recreate            | scratch                | `created`                                                   | no     |
| strand `git mv`     | bead strand            | `moved`                                                     | yes    |
| co-rendered note    | lint note              | `authored`                                                  | no     |
| generated-only edit | roster                 | `regenerated`                                               | no     |
| asset add           | diagram                | `created`                                                   | no     |

Also assert: promotion summary phrase and non-zero `words_added` / `words_removed`;
reflow volume `0` and empty word ops' counts; created `created_words` equals the
document word count; the promotion commit's provenance agent is `athena.sase-1au.5` and
bead is `sase-1au.5`; `conventional_type` on the lint edit is `feat`; `commit_types`
contains `stitch` when `SASE_AGENT` is present (that is what `classify_commit_types`
returns today); the init commit is `boilerplate`.

A content-changing rename must not be hidden. A pure move must not be `authored`.

## 6. Phase: Instruction causes, changesets, and the merged feed (`causes-feed`)

Add `causes.rs` and `feed.rs`.

Cause attribution runs only for instruction versions. Memory sources are the other
non-generated note, web, and strand versions in the same commit (already in the index).
Config and renderer matches are **not** expected to appear in the file-history walk,
because those paths are outside the pathspec. For the set of commits that produced an
instruction version, run one batched

```text
git diff-tree --stdin --root -r --name-only -z
```

through the `file_history` runner (budgets, `--no-optional-locks`, no fetch). A path
matches `config_paths` by equality. A path matches a renderer prefix when it equals the
prefix or starts with the prefix plus `/`. Fill `cause.sources`, `cause.config_paths`,
and `cause.renderer_paths`. Set `cause.regen_only` when all three are empty. Do not
treat a generated subject or another instruction subject as a source.

Feed:

- Group classified versions by `(scope_key, commit)`.
- Split entries into `authored` and `consequences` using the role rule in §3.
- Set changeset `regen_only` and `boilerplate` from §3.
- `build_feed(scopes, since, limit, include_hidden)` merges changesets by committer time
  descending, then `scope_key` ascending, then commit ascending. `since` is an optional
  inclusive epoch-second lower bound. `limit` applies after filtering.
  `include_hidden: false` drops `hidden_by_default` versions and drops `regen_only`
  changesets, counting the dropped changesets in `hidden_changeset_count`.
  `include_hidden: true` keeps both.
- Several scopes are an input. One scope is the one-element case.

### Tests in this phase

- The co-rendered lint commit: the note is authored, `AGENTS.md` is a consequence of
  class `rendered`, and `cause.sources` names the lint note's ordinal. It is not
  `regen_only`.
- The config commit: class `config`, `cause.config_paths == ["sase/sase.yml"]`, empty
  sources and renderer paths.
- The renderer commit: class `config`, `cause.renderer_paths` contains
  `src/sase/amd/render.rs`, empty config paths.
- The regen-only `AGENTS.md` commit: class `regen_only`, empty cause lists, changeset
  `regen_only`, omitted from the default feed and present with `include_hidden`.
- The roster-only edit folds as a consequence and the changeset is `regen_only`. The
  commit that edits the lint note and adds roster keeps the note authored and the roster
  as a consequence, and is not `regen_only`.
- Init boilerplate is flagged and still present in the default feed.
- Hidden reflow and pure-move versions disappear from the default feed and return with
  `include_hidden`.
- Merging the home repo with the project repo orders a home changeset between the two
  project changesets whose committer dates bracket it. `since` and `limit` cut that
  merged list.

## 7. Phase: Snapshot cache, upstream marker, and query bindings (`cache-queries`)

Add `cache.rs`, `upstream.rs`, `query.rs`, and `crates/sase_core_py/src/memory_history/`
(`mod.rs` plus `tests.rs`). Register `register_memory_history` from
`crates/sase_core_py/src/lib.rs` next to the other domain registers. Bindings follow
`prose_diff`: `#[pyfunction]`, `#[pyo3(name = "memory_history_…")]`,
`fn py_memory_history_…`, dict → `*Wire` via serde, `py.allow_threads` around the core
call, `serialize_to_py`, core errors mapped to `PyValueError`.

### Cache

Snapshot path: `{cache_dir}/v{schema}/{sha256-of-canonical-key}.json`. `cache_dir` is
the scope's `cache_dir` (Python passes `~/.sase/cache/memory_history`). The hashed key
contains `scope_key`, `repo_common_dir`, the sorted pathspecs, sorted `generated_notes`,
`renderer_prefixes`, `config_paths`, and `memory_roots`, the canonical instruction-file
list, `MEMORY_HISTORY_WIRE_SCHEMA_VERSION`, and `CLASSIFIER_VERSION`. It does not
contain `repo_root` or `cache_dir`.

The snapshot stores the semantic subjects (no bodies) plus the embedded
`FileHistoryIndexWire` plus the key. Writes use a writer-only advisory lock on a sibling
`.lock` file and the same temp-file-plus-rename pattern as `persist_index`. Readers
never open the lock. A corrupt, unreadable, or wrong-schema file is `Ok(None)`: sync
rebuilds and does not surface a cache error.

`sync`:

1. Resolve the common dir and HEAD through `file_history` (do not shell out separately).
2. Load the snapshot. A memo keyed by `(path, mtime, len)` skips reparsing inside this
   process.
3. Key mismatch, missing file, or corrupt file: `build_index`, classify, attribute
   causes, status `rebuilt`.
4. Key match and cached tip equals HEAD: status `fresh`, no classify.
5. Key match and cached tip is a first-parent ancestor of HEAD: `sync_index` folds the
   git walk, then classification runs again over the **full** folded lineages (blob
   reads hit the OID cache). Status `folded`. Do not hand-merge old class rows;
   reclassifying the folded index is what makes the equality test hold.
6. Anything else (rewrite, missing tip, not an ancestor): status `rebuilt`.
7. Persist the result. A persist failure still returns the in-memory sync result.

`elapsed_ms` is the wall time of that sync. Counts are subject count, version count, and
hidden-version count. Health (`shallow`, `shallow_boundary_time`, `truncated`,
`missing_objects`, `complete`) is copied from the embedded file index.

### Upstream

`upstream_ahead` is `Option<u64>`. Resolve the local remote-tracking default with
`git symbolic-ref --quiet --short refs/remotes/origin/HEAD` through the bounded runner.
If that ref is absent, return `None`. Otherwise
`git rev-list --first-parent --count HEAD..<that-ref> -- <pathspecs>`. `Some(0)` means
the checkout is not behind. Never fetch. A git failure on this marker becomes `None` and
does not fail sync.

### Queries

Each query syncs first (so a fresh cache is a stat check plus a memo hit).

| Python name                          | Behavior                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `memory_history_wire_schema_version` | Returns the `u32` constant. No git.                                                                                                                                                                                                                                                                                                                                                                                                         |
| `memory_history_sync`                | The sync record: status `fresh` / `folded` / `rebuilt`, tip, counts, health, `upstream_ahead`, `elapsed_ms`.                                                                                                                                                                                                                                                                                                                                |
| `memory_history_subjects`            | Subject list without version bodies.                                                                                                                                                                                                                                                                                                                                                                                                        |
| `memory_history_resolve`             | Selector plus optional `at_commit`. Resolution order: exact subject id, exact repo-relative path, unique basename. Ambiguous basenames error with the candidate ids. `at_commit` means "as of this revision": one `git log --first-parent -1 --format=%H <rev> -- <every historical path>`, then the version for that commit. No such commit yields the subject with `existed: false` and no version. An unsafe revision token is an error. |
| `memory_history_timeline`            | Committed versions (hidden ones dropped unless `include_hidden`), plus aliasing (`diverged_count`, `aliased_paths`). Pseudo-versions come from `path_status` on the current path and are listed before committed rows: `uncommitted` when `worktree_oid` differs from `head_oid`, and `staged` only when `index_oid` differs from both. Path state (`tracked`, `untracked`, `ignored`, `no_vcs`) is on the response.                        |
| `memory_history_version`             | One version by ordinal (`7` or `v7`), by `~N` (`~1` is the newest committed version, `~2` the one before it), or by a unique commit SHA prefix. `include_body` fetches the blob inside core. A deleted version's body is the tombstone blob. `now` reads the worktree file. A missing object returns an empty body and `body_missing: true` rather than failing.                                                                            |
| `memory_history_compare`             | Fetches both blobs inside core and returns `compare_prose` embedded in the memory-history envelope. Assets use `format: plain`. Base and target accept the same selectors as `version`.                                                                                                                                                                                                                                                     |
| `memory_history_feed`                | `build_feed` over the given scopes after each scope is synced. `since` is epoch seconds.                                                                                                                                                                                                                                                                                                                                                    |

### Tests in this phase

- Cache round trip: sync, drop the process memo, sync again, status `fresh`, subject ids
  unchanged.
- Corrupt snapshot bytes: sync status `rebuilt`, and the file on disk parses.
- Key mismatch: change `generated_notes`, sync, status `rebuilt`.
- Incremental equality: sync at an intermediate commit (the co-rendered note commit),
  then sync at final HEAD. Compare subjects, ordinals, classes, summaries, causes, and
  the default feed to a cold rebuild at HEAD. Exclude `elapsed_ms`.
- `upstream_ahead` is `None` with no origin ref, and `Some(1)` after a second clone's
  `refs/remotes/origin/master` is pointed one in-scope commit ahead without fetching.
- Resolve `build_and_run.md` to the lint note. Resolve a commit that predates the file
  to `existed: false`. Timeline of a dirty worktree includes `uncommitted` and does not
  invent a committed ordinal for it. Compare of the promotion version against its parent
  reports `type_change: promoted`.
- Binding test in `sase_core_py`: build a tiny repo, call `memory_history_sync` and
  `memory_history_subjects` through the registered module, and assert the schema version
  and the note id. Model the harness on `crates/sase_core_py/src/prose_diff/tests.rs`.

Run `sase tool run check` in sase-core. Do not ratchet sase's `sase-core-revision.txt`
and do not add a Python caller; until `history-cli` moves the pin, sase's CI will not
see these bindings, and that is expected.
