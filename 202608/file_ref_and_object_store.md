---
tier: tale
title: The @file ref and the content-addressed store
goal:
  "@file:<path> becomes a tracked citation: an artifact_refs.file.roots allow-list
  bounds it, Rust resolves containment/globs/file-type/size, launch capture reads the
  bytes once and hashes those bytes, the prompt expands to the captured copy,
  publication writes one object per digest into the agents sidecar at
  files/objects/sha256/<ab>/<64hex>, and a durable (logical_path, sha256) version index
  is promoted at publication time."
size: medium
proposed_by: bbugyi200.athena.sase-js.5
bead: sase-js.5
create_time: 2026-08-11 16:37:11
status: wip
---

- **PARENT:**
  [202608/artifact_ref_contract.md](https://github.com/sase-org/sase--plans/blob/main/202608/artifact_ref_contract.md)
- **BEAD:**
  [sase-js.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-js/sase-js.5.md)

# Plan: The `@file` ref and the content-addressed store

Phase `files` of epic `sase-js` (bead `sase-js.5`), whose plan is
`@plan:202608/artifact_ref_contract.md` §3.5, §3.7 and §4.5. Depends on `registry`
(`sase-js.3`), which is landed. Siblings `builtins` (`sase-js.4`), `linking`
(`sase-js.6`), `ace` (`sase-js.7`) are in flight in parallel workspaces; this plan stays
inside the `@file` and object-store surface so the land agent has minimal overlap to
reconcile.

## 1. Goal

Make `@file:<path>` a first-class tracked citation:

- an opt-in `artifact_refs.file.roots` allow-list is the exfiltration boundary,
- resolution enforces containment, globs, file-type, and size in Rust,
- launch-time capture reads the bytes exactly once and hashes those bytes,
- the prompt expands to the immutable captured copy,
- publication writes one object per unique byte sequence into the agents sidecar at
  `files/objects/sha256/<ab>/<64hex>`,
- a durable `(logical_path, sha256)` version index is promoted at publication time.

Non-goals for this phase: the `ref-uses` manifest (phase `builtins`), reference-link
rewriting and `Referenced By` (phase `linking`), the Files pane (phase `ace`), and the
documentation rewrite (phase `adopt`, which explicitly owns `docs/configuration.md` and
`docs/agents_sidecar.md` for these keys). Completion for `@file:<path>` is also out of
scope; the kind catalog already advertises the argument shape.

## 2. Verified current state

Read in this workspace at `f53e43ab1`, and in the linked `sase-core` checkout at
`c0f1ca4`. Open the core repo with `/sase_repo` (`sase repo open sase-core`); dev
installs build the extension from `sase/repos/linked/sase-core`, so unreleased core
commits are expected here and the published version window in `pyproject.toml` is
ratcheted separately at release time.

**Rust already parses the path payload but refuses to resolve it.**
`crates/sase_core/src/artifact_ref/wire.rs` has
`ArtifactRefPayloadWire::FilePath { path }` alongside `File { source, digest }`;
`parse_payload` in `artifact_ref/mod.rs` disambiguates on "first segment is
`explicit`/`default` and there is exactly one colon". `render_artifact_ref` round-trips
it through `validate_file_path_payload` (rejects empty, NUL, backslash, trailing slash).
`resolve_artifact_ref` returns `unresolved_kind_resolution("file", ...)` — status
`unknown_kind` with the diagnostic "file references resolve through the provider
registry, not this crate". Replacing that arm is the core of this phase.

**`ArtifactRefContextWire` has no file roots.** It carries `document_roots`,
`chats_root`, `artifact_index_path`, `repositories`, `projects`, `bead_stores`,
`agent_roots`, `agent_owner`, at `ARTIFACT_REF_CONTEXT_WIRE_SCHEMA_VERSION = 1`.
`validate_artifact_ref_context` requires exact version equality.

**The filter vocabulary already exists.** `artifact_ref/filter.rs`'s
`ArtifactPathFilter::compile/allows/summary` is what `resolve_document` uses, and
`filtered_resolution` already builds a diagnostic that names the kind, the repo-relative
payload, and the filter summary without echoing an absolute local path. That is exactly
the non-leaking behavior `@file` needs.

**`artifact_refs` is not a config key.** `src/sase/config/sase.schema.json` has
`additionalProperties: false` and no `artifact_refs` property. `artifacts` exists and is
unrelated (capture/retention budgets).

**Staging is a workspace-local pool, publication is a per-month directory.**
`src/sase/core/prompt_artifact_staging.py::stage_prompt_artifact` hashes with
`hash_file(source)` and then copies to
`<workspace>/.sase/artifacts/pool/<12hex>-<basename>` (Rust
`prompt_artifact_pool_filename`), appending a row to
`.sase/artifacts/prompt-artifacts.jsonl` under `fcntl`.
`src/sase/agents_sync/prompt_archive/preparation.py::_publish_linked_artifacts` copies
each pooled file to `artifacts/<yyyymm>/<same basename>` in the agents repo and
`_ArtifactTargetResolver.__call__` links it as `../../artifacts/<yyyymm>/<name>`. Both
duplication vectors named in the epic plan §2 are here: identical bytes under different
basenames get different pool names, and identical bytes cited in different months land
in different directories.

**Prompt expansion order.** `sase/llm_provider/preprocessing.py::preprocess_prompt_late`
runs artifact refs (step 3) before plain `@path` refs (step 4), passing
`staged_file_paths` so a generated `@<path>` is not re-staged.
`artifact_ref_prompt.py::_expand_artifact_references` resolves every candidate, renders
a replacement, and only afterwards calls `_stage_artifact_references`. For
`@file:<path>` the capture has to happen _before_ the replacement text is built, because
the replacement must name the captured copy.

**Preprocessing runs inside the agent process.** `preprocess_prompt` is called from
`sase/llm_provider/_invoke.py::invoke_agent`, i.e. after the launch-approval preview
(`sase/agent/launch_preview.py`) has already been built host-side from the unexpanded
prompt. The approval preview therefore cannot show capture records without re-resolving
refs host-side; see §6.

**Sidecar visibility is configured.** `repos.sidecar.builtin.<role>.visibility` defaults
to `"public"` (`src/sase/sdd/_sidecar_init.py`, `src/sase/_linked_repo_config.py`).

**Precedent to copy.** `src/sase/config/file_hooks.py` is the fail-soft-per-entry loader
(layer replay for provenance, `logger.warning` per bad entry, memoized on
`current_config_token()`). `crates/sase_core/src/artifact_ref/uses.rs` is the tolerant
JSONL wire pattern (skip malformed and future-versioned lines). The `sase_core_py`
binding list lives in `crates/sase_core_py/src/lib.rs`, mirrored by `REQUIRED_BINDINGS`
in `tools/validate_sase_core_rs`.

## 3. Design decisions

### 3.1 The store is the agents sidecar; the workspace pool stays staging

Epic plan §3.7 places the content-addressed store "in the agents sidecar" at
`files/objects/sha256/<ab>/<64hex>`, and §4.5 warns not to build the store on
`artifact_pool_filename`. This plan honors both by moving _publication_ to the sidecar
CAS while leaving the workspace `.sase/artifacts/pool/` as ephemeral staging.

- The published store gets exactly one path per unique byte sequence, killing both
  duplication vectors, because the sidecar path is derived only from the full digest.
- The staging copy keeps the original basename (and therefore the file extension), which
  the prompt's `@<path>` expansion and the downstream attachment handling need. A
  `<64hex>` extension-less path in the prompt would change how an agent CLI treats a
  `.md` versus a `.png` citation.
- The existing pool budget, pruning, publish, and digest-verification machinery is
  reused rather than duplicated.

Every byte-backed prompt artifact — not just `@file` ones — publishes into the CAS from
this phase on. Historical archives keep their `artifacts/<yyyymm>/` links; the epic plan
§5 explicitly forbids rewriting historical pages for link style, so
`artifacts_month_dir` and `relative_artifact_link` stay for reading old archives.

### 3.2 Resolution is Rust; capture and publication are Python

Per `CLAUDE.md`'s Rust core backend boundary and epic plan §3.3, grammar, root
containment, glob filtering, and file-type policy are cross-frontend behavior and go in
`sase_core`. Byte copying, config merge, manifests, and the publication outbox stay in
Python, matching the existing split where `prompt_artifact_pool_filename` is a Rust
binding but the copy is Python.

### 3.3 One new resolution status

`denied` is added alongside the existing statuses. `filtered` keeps its established
meaning — the configured allow-list said no (outside every root, or a glob miss) — and
its non-leaking diagnostic. `denied` covers policy rejections about the file itself
(directory, special file, symlink escape, unreadable, over the size ceiling), so the
diagnostic can be actionable without pretending the allow-list was the cause. `missing`
still means "inside a root, matches the globs, does not exist".

### 3.4 The logical path is the published identity

A resolved `@file:<path>` yields `locator = "<root-name>:<root-relative-posix-path>"`
(for example `bob:gtd.md`) and `resolved_path = <canonical absolute path>`. Only the
logical path, the authored spelling, and the digest reach published metadata. The
physical `/home/<user>/...` path stays in the workspace-local manifest, which is not
published.

### 3.5 The version index is a sibling, keyed by `(logical_path, sha256)`

Epic plan §3.9 permits "the durable index (or a sibling `ref_files` index) keyed by
`(logical_path, sha256)`". A sibling is chosen: the existing
`~/.sase/artifacts/index.jsonl` is keyed by artifact id, is read by
`query_artifact_files` through a versioned Rust query wire, and is exactly what the
parallel `ace` phase is about to widen (`explicit` boolean to an origin enum). Adding a
second identity axis to it would collide with that phase and force a schema migration of
every historical row. The sibling index `~/.sase/artifacts/ref-files.jsonl` is new,
additive, and gives the Files pane the `(one logical row, N versions)` shape it wants
directly.

### 3.6 Payload shapes accepted

`@file:explicit:<hex24>` and `@file:default:<hex24>` keep working exactly as today.
`@file:<path>` accepts an absolute path or a `~/`-rooted path only. A bare relative path
is rejected with an actionable diagnostic, because epic plan §3.6 forbids inferring
context from `cwd` and a relative `@file` would do exactly that. The logical spelling
(`@file:bob:gtd.md`) is not accepted as an input form in this phase; it would be
ambiguous with paths containing colons.

## 4. Implementation

### 4.1 sase-core: file roots and path resolution

Open the repo with `/sase_repo` first. All Rust below lands in `crates/sase_core`.

**`artifact_ref/wire.rs`**

- Add
  `ArtifactRefFileRootWire { name: String, path: String, #[serde(default)] path_globs: Option<Vec<String>> }`,
  with the same `Option` contract as `ArtifactRefDocumentRootWire::path_globs` (`None` =
  allow all, `Some(vec![])` = allow nothing).
- Add to `ArtifactRefContextWire`:
  `#[serde(default)] file_roots: Vec<ArtifactRefFileRootWire>`,
  `#[serde(default)] home_dir: Option<String>`,
  `#[serde(default)] file_capture_max_bytes: Option<u64>`.
- Bump `ARTIFACT_REF_CONTEXT_WIRE_SCHEMA_VERSION` to `2` and update `Default`.

**`artifact_ref/file_roots.rs` (new)**

`pub fn resolve_artifact_file_path(path: &str, context: &ArtifactRefContextWire, rendered: String) -> Result<ArtifactRefResolutionWire, ArtifactRefError>`,
in this order:

1. `validate_file_path_payload` (already exists).
2. Reject a relative payload: diagnostic "`@file:` needs an absolute or `~/` path".
3. Expand a leading `~/` (and a bare `~`) using `context.home_dir`, falling back to
   `std::env::var("HOME")`; if neither is available, return `denied` with a diagnostic
   rather than guessing.
4. If `context.file_roots` is empty, return `filtered` with a diagnostic naming
   `artifact_refs.file.roots` as the thing to configure.
5. Canonicalize both the candidate and each root with `std::fs::canonicalize`. A
   candidate that does not exist is canonicalized by canonicalizing its nearest existing
   ancestor and re-joining the remainder, so a missing file inside a root is still
   attributed to that root. Containment is decided on the _canonical_ paths, which is
   what defeats a symlink that points out of the root.
6. Collect every root that contains the canonical candidate. Zero roots is `filtered`
   (diagnostic lists the configured root names, never the absolute candidate). More than
   one root is `ambiguous`, with the root names as `candidates`.
7. Apply the root's `ArtifactPathFilter` to the root-relative POSIX path. A miss is
   `filtered` with `filter.summary()` — never `missing`.
8. `symlink_metadata` on the candidate: a symlink whose canonical target left the root
   is `denied`. Then `metadata`: a directory, FIFO, socket, block or character device is
   `denied` with the kind named. A path that does not exist is `missing`.
9. Probe readability by opening the file; `denied` on `PermissionError`.
10. If `context.file_capture_max_bytes` is set and the file exceeds it, `denied` with
    both sizes in the diagnostic.
11. Success: `exact`, `locator = Some("<root>:<relpath>")`,
    `resolved_path = Some(<canonical absolute path>)`, `candidates = vec![]`.

Wire this into `resolve_artifact_ref`'s `(File, FilePath)` arm, replacing
`unresolved_kind_resolution`. Export from `artifact_ref/mod.rs`.

**`artifact_object_store.rs` (new, top level of the crate)**

- `pub const ARTIFACT_OBJECT_STORE_DIR: &str = "files/objects/sha256";`
- `pub fn artifact_object_relpath(sha256: &str) -> Result<String, ArtifactRefError>` →
  `files/objects/sha256/<first two hex>/<64 hex>`, rejecting anything that is not
  exactly 64 lowercase hex characters.
- `pub fn artifact_object_prompt_link(relpath: &str) -> Result<String, _>` → the
  prompt-relative link `../../<relpath>`, matching the existing
  `prompts/<yyyymm>/<name>.md` depth that `relative_artifact_link` already assumes, and
  validating the relpath shape rather than accepting arbitrary input.

**`artifact_ref/ref_files.rs` (new)**

Modelled line-for-line on `uses.rs`.

- `ARTIFACT_REF_FILE_INDEX_WIRE_SCHEMA_VERSION: u64 = 1`
- `ArtifactRefFileVersionRowWire { schema_version, logical_path, root_name: Option, authored_path: Option, artifact_id: Option, sha256, size_bytes, mime_type: Option, first_seen_at, origin, object_relpath, sidecar_repo: Option, agents: Vec<String>, projects: Vec<String> }`
  where `origin` is `ref | created | capture`.
- `validate_artifact_ref_file_row` — non-empty `logical_path`, 64 lowercase hex
  `sha256`, known `origin`, `object_relpath` equal to `artifact_object_relpath(sha256)`.
- `render_artifact_ref_file_row` and `parse_artifact_ref_file_index` (tolerant: skip
  malformed and future-versioned lines with a stderr note, exactly as `uses.rs`).
- `fold_artifact_ref_files(rows) -> Vec<ArtifactRefLogicalFileWire>` where
  `ArtifactRefLogicalFileWire { schema_version, logical_path, root_name, origin, versions: Vec<ArtifactRefFileVersionWire> }`;
  versions are deduplicated on `sha256`, keep the earliest `first_seen_at`, union
  `agents` and `projects`, and are sorted by `(first_seen_at, sha256)`. Logical files
  sort by `logical_path`.

**`prompt_artifact.rs`**

Bump `PROMPT_ARTIFACT_MANIFEST_SCHEMA_VERSION` to `2` and add, all `#[serde(default)]`
so v1 rows still parse: `logical_path`, `root_name`, `authored_path`, `origin`,
`object_relpath`, `sidecar_visibility`. Keep the parser accepting version 1 rows
(historical manifests in long-lived workspaces).

**`crates/sase_core_py/src/lib.rs`**

Register and document: `artifact_object_relpath`, `artifact_object_prompt_link`,
`artifact_ref_file_index_parse`, `artifact_ref_file_row_render`,
`artifact_ref_file_row_validate`, `artifact_ref_files_fold`,
`artifact_ref_file_index_wire_schema_version`. Extend `artifact_ref_context_from_pydict`
for the three new context fields.

**Rust tests** (in-module, per crate convention): `~` expansion with and without
`home_dir`; absolute and `~` spellings resolving to the same logical path; relative
rejection; zero roots; one root; two overlapping roots giving `ambiguous`; glob miss
giving `filtered` and not `missing`; a symlink inside a root pointing outside giving
`denied`; `..` traversal that escapes; directory, FIFO, and unreadable rejection; size
ceiling; missing file inside a root; the diagnostic never containing the absolute
candidate for `filtered`; `artifact_object_relpath` validation and prefixing; ref-files
row round-trip, tolerant parse, and fold dedupe/ordering.

### 4.2 sase: config allow-list

**`src/sase/config/sase.schema.json`** — new top-level `artifact_refs`:

```yaml
artifact_refs:
  file:
    roots:
      - name: bob
        path: ~/bob
        path_globs: ["**/*.md"]
```

`additionalProperties: false` at every level; `name` matches `^[a-z0-9][a-z0-9_-]*$`;
`name` and `path` required; `path_globs` an array of strings.

**`src/sase/config/artifact_ref_files.py` (new)** — modelled on `file_hooks.py`:

- `@dataclass(frozen=True) ArtifactFileRoot { name, path: Path, path_globs: tuple | None, source_layer: str }`.
- `get_artifact_file_roots() -> tuple[ArtifactFileRoot, ...]`, memoized on
  `current_config_token()`, replaying `load_config_layers()` so a layer with
  `list_strategy == "replace"` resets the accumulated roots and each root keeps the
  layer that supplied it. Later layers with the same `name` replace earlier ones.
- Fail soft per entry: a bad root is `logger.warning`-ed and skipped; a bad
  `artifact_refs` block never raises on the launch path.
- Paths are `expanduser()`-ed and resolved non-strictly; a root that does not exist is
  still returned (resolution reports it; `sase doctor` complains).
- Export through `src/sase/config/__init__.py`.

**Doctor** — `src/sase/doctor/checks_config_artifact_refs.py` (new), registered in
`src/sase/doctor/checks_config.py` the same way `check_config_repos` is: findings for a
root path that is missing or not a directory, a duplicate or non-slug root name, a root
nested inside another root, an unparsable `path_globs`, and an `artifact_refs` block
that parsed to zero usable roots. Each finding names the config file that supplied the
entry.

### 4.3 sase: context, models, resolution

**`src/sase/artifact_ref_models.py`**

- `ARTIFACT_REF_CONTEXT_WIRE_SCHEMA_VERSION = 2`.
- `@dataclass(frozen=True, slots=True) ArtifactRefFileRoot { name, root: Path, path_globs: tuple | None }`
  with `to_wire()`.
- `ArtifactRefContext` gains `file_roots: tuple[ArtifactRefFileRoot, ...] = ()`,
  `home_dir: Path | None = None`, `file_capture_max_bytes: int | None = None`, all
  emitted by `to_wire()`.
- `ArtifactRefResolutionStatus` gains `"denied"`; mirror the literal in
  `artifact_ref_lists.py` and `artifact_ref_operations.py`.

**`src/sase/artifact_ref_context.py`** — populate `file_roots` from
`get_artifact_file_roots()`, `home_dir` from `Path.home()`, and `file_capture_max_bytes`
from `get_artifact_capture_max_file_size_bytes()`. Wrap the config read in the same
best-effort `try` used for the repo inventory so a broken config cannot break every
other ref kind.

**`src/sase/artifact_ref_prompt_resolution.py`** — `artifact_resolved_path` returns the
_captured_ path for a `file_path` payload (see §4.4) and the existing behavior for
digest payloads. `artifact_ref_resolution_hint` gains a `denied` branch that returns the
resolver diagnostic verbatim.

### 4.4 sase: launch-time capture

**`src/sase/core/prompt_artifact_staging.py`**

- New
  `capture_prompt_file_ref(*, source: Path, logical_path: str, root_name: str, authored_path: str, raw_ref: str, expanded_ref: str, workspace_root=None, agent_artifacts_dir=None) -> PromptArtifactRecord | None`.
- Single pass: open the source once, stream it through `hashlib.sha256` while writing to
  a `tempfile` in the pool directory, then `os.replace` into
  `pool/<prompt_artifact_pool_filename(sha256, source.name)>`. The digest is of the
  bytes actually written, never of a later re-read, so a file changed during capture
  cannot produce a manifest row whose digest disagrees with the captured copy. If the
  destination already exists, verify its digest and reuse it instead of rewriting.
- Enforce `capture_file_exceeds_size_limit` again here (the Rust resolver checks the
  pre-read size; this catches growth between resolution and capture) and abort the
  capture with a `too-large` skip row.
- Fill the new manifest fields: `logical_path`, `root_name`, `authored_path`,
  `origin="ref"`, `object_relpath = artifact_object_relpath(sha256)`,
  `sidecar_visibility` read from the agents sidecar config.
- Extend `_pool_files` / `_prune_prompt_artifact_pool` unchanged in behavior — the
  captured copies live in the same `pool/` directory and are already covered.

**`src/sase/artifact_ref_prompt.py`**

- In `_expand_artifact_references`, after a successful resolution whose payload type is
  `file_path`, call the capture before building the replacement and use the captured
  path as the resolved path. A capture that returns `None` (no artifacts dir, i.e. no
  staging available, which is also the home-mode case) falls back to the physical
  resolved path; this is the same rule that already governs plain-file staging, and it
  keeps home mode working without inventing a second store.
- Record captured refs in the existing `_stage_artifact_references` bookkeeping so the
  following plain-file pass does not double-stage the captured path, and skip the
  generic `stage_prompt_artifact` call for a ref that was already captured.
- After expansion, emit one grouped notice (stderr) listing each captured `@file`:
  authored spelling, logical path, digest prefix, size, and the target sidecar repo plus
  its configured visibility. This is the launch-time surfacing of the exfiltration
  boundary required by epic plan §4.5.

### 4.5 sase: publication into the sidecar CAS

**`src/sase/agents_sync/prompt_archive/preparation.py`**

- `_ArtifactTargetResolver.__call__`: when a record has `sha256` and a pooled source,
  return `artifact_object_prompt_link(artifact_object_relpath(sha256))` instead of
  `relative_artifact_link(...)`. Every other branch (VCS blob, locator, commit, bug,
  agent) is unchanged.
- `_publish_linked_artifacts`: write to `repo / artifact_object_relpath(digest)`. Reuse
  `_copy_content_addressed`, which already refuses a digest collision and re-verifies
  after the copy. An object that already exists at that path is verified byte-wise and
  left alone.
- Keep `artifacts_month_dir` and `relative_artifact_link` exported for reading
  historical archives; add a module comment saying new publications no longer use them.

### 4.6 sase: durable version index

**`src/sase/core/artifact_ref_files_index.py` (new)**

- `default_ref_files_index_path()` → `~/.sase/artifacts/ref-files.jsonl` (sibling of the
  existing `index.jsonl`, same root as `default_artifact_files_root()`).
- `upsert_ref_file_versions(records, *, index_path=None)` — takes the published manifest
  rows, builds Rust-validated rows via `artifact_ref_file_row_validate` /
  `artifact_ref_file_row_render`, and appends under the existing `locked_file` pattern.
  Repeats of the same `(logical_path, sha256)` append a provenance row rather than
  rewriting history; the fold collapses them.
- `query_ref_file_versions(*, index_path=None)` — reads the file and returns the folded
  `ArtifactRefLogicalFile` projection through `artifact_ref_files_fold`.
- Rows for a cited `sase artifact create` output (a `file:explicit:<hex24>` or
  `file:default:<hex24>` ref that published bytes) are written with `origin="created"`,
  `artifact_id=<the id>`, and `logical_path` taken from the artifact's recorded
  `source_path` when known, else `artifact:<id>`. That is the compatibility alias epic
  plan §4.5 asks for: the old id still names the new object/version record.

**Call site** — in `agents_sync/prompt_archive/publish.py::_publish_prompt_archive`,
after `prepare_prompt_archive` has written the sidecar objects and inside the same
publication step, so only published runs contribute versions and the launch path stays
cheap. Failures here are logged and never fail the publication.

### 4.7 Binding validation

Add every new binding name to `REQUIRED_BINDINGS` in `tools/validate_sase_core_rs` and
keep `just check` green.

## 5. Tests

Rust tests are listed in §4.1. Python tests, mirroring the epic plan §4.5 matrix:

- `tests/artifact_refs/test_file_roots_config.py` — layered merge including a `replace`
  layer; per-entry fail-soft on a bad root; duplicate name resolution; `~` expansion;
  memoization on the config token.
- `tests/artifact_refs/test_file_ref_resolution.py` — absolute and `~` spellings give
  one logical path; outside every root; two roots; glob miss is `filtered`; symlink
  escape, `..` traversal, directory, FIFO, unreadable, over-size all rejected with the
  right status; missing file inside a root; both digest payload shapes still resolve
  exactly as before; the failure message never contains the absolute path for a
  `filtered` result.
- `tests/artifact_refs/test_file_ref_capture.py` — the prompt expands to the captured
  copy, not the source; the captured bytes equal the source bytes; a source rewritten
  after capture does not change the captured copy or its digest; capture is a single
  read; duplicate content cited twice under different names produces one manifest
  digest; capture falls back to the physical path when no artifacts dir is set.
- `tests/agents_sync/test_prompt_archive_object_store.py` — one object per full digest
  across differing basenames, agents, months, and origins; the published link is the
  prompt-relative CAS path; an existing object with matching bytes is reused; an
  existing object with different bytes raises; historical `artifacts/<yyyymm>/` archives
  still render.
- `tests/core/test_artifact_ref_files_index.py` — upsert and fold produce one logical
  row with N versions ordered by first-seen; a repeated capture with an unchanged digest
  adds provenance without adding a version; a `created`-origin row carries its artifact
  id alias; a malformed line is skipped.
- `tests/doctor/test_checks_config_artifact_refs.py` — each finding.
- `tests/test_config_schema.py` — the new `artifact_refs` block validates and a
  misspelled key is rejected.

Run `just install` first (ephemeral workspaces), then `just check`. Because this change
touches the Rust binding and the publication path, finish with `just check-full`.

## 6. Risks and open items

| Risk                                                                            | Mitigation                                                                                                                                                                            |
| ------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Context wire bump to 2 breaks a caller built against 1                          | Both repos ship together in dev installs; `validate_artifact_ref_context` fails loudly rather than silently resolving with no roots, which is the failure this bump exists to prevent |
| Publishing every byte-backed artifact to the CAS changes prompt-archive tests   | Historical readers keep working; only new publications move, per epic plan §5                                                                                                         |
| Overlap with parallel phases `builtins`, `linking`, `ace`                       | Changes are confined to the `@file` payload arm, the new config key, the new index, and the publication destination; no ref-uses manifest, no link numbering, no ACE                  |
| A symlinked root (for example `~/bob` itself a symlink) making containment fail | Both the root and the candidate are canonicalized before comparison                                                                                                                   |
| An unreadable or slow root directory stalling the launch path                   | Resolution touches only the candidate path and the configured roots; there is no tree scan                                                                                            |

**Deferred, to be recorded as a `PROPOSED FOLLOW-UP:` on `sase-js.5` rather than
silently dropped:** showing captured `@file` targets and their sidecar visibility in the
_launch-approval preview_. The preview is built host-side before the agent process
starts, and prompt preprocessing (where refs resolve) runs inside the agent process, so
this needs a host-side resolve-only pass that does not belong in this phase. The
launch-time notice in §4.4 delivers the same information at the moment the bytes are
captured.
