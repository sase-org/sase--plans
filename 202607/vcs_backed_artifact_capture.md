---
tier: epic
title: Make artifact capture mean authorship and stop copying what version control
  stores
goal: 'Automatic artifact capture keeps bytes only for files an agent authored that
  version control cannot reproduce. Files that version control can reproduce get a
  byte-free record carrying repo, commit, and path provenance, and every reader —
  CLI, Files pane, and prompt `@`-ref expansion — can still get their exact content
  on demand.

  '
phases:
- id: core-record
  title: 'Rust core: VCS-backed records and on-demand materialization'
  depends_on: []
  size: medium
  description: 'core-record: add the three VCS-provenance fields and an optional stored
    path to the artifact-file wire, admit byte-free rows, resolve them to a new `vcs_backed`
    status, and add a content-verified git materialization primitive exposed through
    `sase_core_rs`.

    '
- id: capture-policy
  title: Capture policy — the authorship and version-control decision
  depends_on: []
  size: medium
  description: 'capture-policy: add a pure decision module that classifies every capture
    candidate as store, reference, or skip using a git probe, the repo inventory,
    the run window, and a per-run byte-copy cap, plus its configuration.

    '
- id: py-record
  title: Python record, doctor, and read surfaces
  depends_on:
  - core-record
  size: medium
  description: 'py-record: mirror the new record fields in Python, make ids and dedupe
    keys work without a stored path, teach doctor that byte-free rows are healthy,
    and make every read surface materialize VCS-backed content on demand.

    '
- id: capture-wiring
  title: Wire the policy into finalization capture
  depends_on:
  - capture-policy
  - py-record
  size: small
  description: 'capture-wiring: label capture candidates with their origin, route
    them through the policy at finalization, write reference rows instead of byte
    copies, enforce the cap, and cover the decision matrix end to end.

    '
- id: docs-and-skill
  title: Docs, skill, and configuration reference
  depends_on:
  - capture-wiring
  size: small
  description: 'docs-and-skill: document VCS-backed artifact files in the `sase_artifact_file`
    skill source, the artifact docs, and the new configuration block, and regenerate
    the deployed skill.

    '
create_time: 2026-07-30 08:52:41
status: done
bead_id: sase-b7
---

- **PROMPT:** [202607/prompts/vcs_backed_artifact_capture.md](prompts/vcs_backed_artifact_capture.md)
- **BEAD:** [sase-b7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-b7/README.md)

# Plan: Make artifact capture mean authorship and stop copying what version control stores

This implements recommendation 2 of `research:202607/artifact_capture_and_retention/artifact_capture_and_retention.md`
("Make capture mean authorship, and never copy what version control stores"), including the §3.3 resolution that the
artifact _record_ — not the reference grammar — carries VCS provenance.

## 1. Context and measurements

All numbers below were re-derived on 2026-07-30 against the live store (`~/.sase/artifacts/index.jsonl`, 4,293 rows) and
a fetched clone of the primary repo. They confirm the research report and correct it in two places that change the
design.

**Confirmed.** 220 rows are explicit (agent-declared); 4,073 are automatic. Of the automatic rows, **3,985 (97.8%) have
a `source_path` that is tracked in the primary repo today**, 81 are workspace files not tracked now, and only 7 are
outside any workspace. Kinds are image 4,053, file 25, markdown 215 — so _every_ automatic row is an image or video, and
`persist_default_artifact_files()` in `src/sase/core/artifact_file_defaults.py:154` is the single write site that
produces them. The 219 explicit rows that live outside a workspace are the declared markdown cohort and are untouched by
this plan.

**Correction 1 — the volume driver is the diff/commit scan, not the prompt regex.** Report `__b` diagnosed capture as "a
regex sweep over prompt text", which is a true code fact but not the cause of the bytes. Comparing each row against its
run's `done.json` for the 2,450 rows whose run directory still exists: 2,449 came from the `image_paths`/`video_paths`
lists that `collect_agent_image_paths()` builds from `git diff HEAD`, untracked files, and `HEAD~1..HEAD`; exactly 1
came from prompt-regex discovery alone. The signature — one agent producing 179 rows, 403 distinct labels across the
snapshot cohort — is an agent regenerating tracked media (visual-snapshot goldens, demo GIFs, docs images, sidecar
infographics) and having every revision copied. Those files _are_ authored by the run, so an authorship gate alone does
not touch them. The version-control gate is what does. Size the phases accordingly: the authorship gate is a correctness
fix with a small measured volume effect; the version-control gate carries essentially the whole byte win.

**Correction 2 — there is no stable "canonical repo checkout", so the commit sha is a hint and the content digest is the
identity.** §3.3 assumes the resolver can run `git show <sha>:<relpath>` "against the canonical repo checkout".
`sase repo list --json` shows the primary repo's inventory path _is_ a numbered workspace, and every numbered workspace
is cleaned and reset on open; only the `agents` sidecar has a path outside a workspace. Two consequences:

- Resolution must try _several_ checkouts of a repo identity (the recorded `workspace_dir` first, then any existing
  registered clone), because they share a remote and any fetched clone can serve a pushed object.
- Reachability must be measured from remote-tracking refs, not from a local HEAD. Measured over the 4,067 rows whose
  source path lies inside a numbered workspace (3,970 distinct relpath/content-digest pairs, 464 distinct paths):
  **4,035 of 4,067 (99.2%) have their exact content reachable from `origin/master` history**, while only 420 match the
  content at that branch's _tip_. The content is durable, but the revision holding it is historical and the sha that
  produced it may be squash-rewritten. The design therefore records `vcs_sha` as provenance and makes the resolver
  verify by content digest, with a bounded history search as a fallback when the recorded sha is gone.

**Already true and de-risking.** `_collect_default_artifacts()` is called from `finalize_loop()` in
`src/sase/axe/run_agent_exec_finalize.py`, which runs _after_ the workflow loop and therefore after the commit finalizer
— the "evaluate at finalization" requirement needs no re-sequencing. Every automatic row already carries `sha256`,
`size_bytes`, and `mime_type` at 100% coverage, so a byte-free row loses no integrity metadata. The index envelope
already declares `ARTIFACT_FILE_INDEX_SUPPORTED_SCHEMA_VERSIONS = {1, 2}` on the Python side and `1..=2` on the Rust
side, so raising the written version to 2 needs no reader change.

## 2. The decision matrix

Every capture candidate resolves to exactly one of three outcomes. `origin` distinguishes candidates the run _changed_
(from `image_paths`/`video_paths`, i.e. `git diff HEAD`, untracked files, or the run's own commit) from candidates
merely _mentioned_ in a prompt.

| #   | Condition (first match wins)                                                                                              | Outcome                                                             |
| :-- | :------------------------------------------------------------------------------------------------------------------------ | :------------------------------------------------------------------ |
| 1   | Content is reproducible from version control (see invariants below)                                                       | `reference` — byte-free row with `vcs_repo`/`vcs_sha`/`vcs_relpath` |
| 2   | Authored by this run: inside the agent's own artifacts directory, `origin == changed`, or mtime at or after the run start | `store` — copy bytes, as today                                      |
| 3   | Mentioned only, and not inside any known repo working tree                                                                | `store` — the user-supplied input case                              |
| 4   | Otherwise (mentioned only, inside a known repo, not reproducible)                                                         | `skip` — no row                                                     |

Rule 3 is a deliberate softening of a strict authorship reading. A screenshot the user hands an agent lives outside
every repo, is not authored by the run, and is exactly what prompt-media discovery exists to surface in the Files pane.
The cohort is tiny — 7 automatic rows historically — so the rule costs nothing and keeps a working affordance rather
than quietly removing it. Rule 4 is the narrow case the authorship gate actually removes: a repo file the run neither
authored nor can reproduce.

Explicit declarations (`sase artifact create` → `store_explicit_artifact_file()`) always store bytes and never enter
this matrix.

### Invariants (non-negotiable)

- **A — no silent substitution.** A `reference` outcome is only produced after the candidate's exact bytes have been
  reproduced from `<vcs_sha>:<vcs_relpath>` and the reproduction's SHA-256 verified equal to the candidate's. A tracked
  file with uncommitted modifications therefore fails rule 1 and falls through to rule 2, which is what the research
  asks for.
- **B — durability.** `vcs_sha` is reachable from a remote-tracking ref at capture time. An unpushed local commit is not
  durable: the workspace that holds it is reset on next open and the object becomes unreachable.
- **C — fail-safe.** Any git failure, timeout, ambiguity, or unknown repo downgrades a would-be `reference` to `store`.
  The change must never be able to lose bytes.

## 3. Rust core: VCS-backed records and on-demand materialization

Open the core repo the sanctioned way and use the path it prints:
`sase repo open sase-core -r "Add VCS-backed artifact-file records and materialization"`. release-plz owns the version;
do not hand-edit `Cargo.toml` versions. The wire-schema bumps below make this a breaking change for older `sase`
installs, so use a `feat!:` commit so release-plz computes a breaking release.

**`crates/sase_core/src/artifact_file.rs`**

- `ArtifactFileWire.path` becomes `Option<String>`; add `vcs_repo`, `vcs_sha`, `vcs_relpath`, each
  `#[serde(default)] Option<String>`.
- Row admission in `parse_artifact_file_index()`: keep a row when it has an `id` and _either_ a non-empty `path` _or_
  all three `vcs_*` fields. Skip anything else, as today.
- Add `pub fn artifact_file_is_vcs_backed(row: &ArtifactFileWire) -> bool` (path absent and the triple present) and use
  it where behavior differs.
- `query_artifact_files()`: the `-q` needle must also match `vcs_relpath`; the tie-break sort on `path` must tolerate
  `None` (sort byte-free rows after byte-backed ones at equal timestamp and id, deterministically).
- Raise `ARTIFACT_FILE_QUERY_WIRE_SCHEMA_VERSION` from 1 to 2.

**New `crates/sase_core/src/artifact_file/vcs.rs`** (or a `vcs` submodule beside `artifact_file.rs`) —
`materialize_vcs_artifact_file(request) -> MaterializationWire`, where the request carries `cache_root`,
`checkout_paths: Vec<String>`, `vcs_sha`, `vcs_relpath`, `sha256`, `suffix`, and `max_history_scan`. Algorithm:

1. `cache = cache_root/<sha256[..2]>/<sha256><suffix>`. If it exists and re-hashes to `sha256`, return
   `{status: "cached", path}`. If it exists and does not, treat it as absent and overwrite.
2. For each checkout in order: `git -C <checkout> cat-file blob <vcs_sha>:<vcs_relpath>`. On success, hash the bytes; if
   they match `sha256`, write them to a temp file in the cache directory, `fsync`, `rename` into place, and return
   `{status: "materialized", path, checkout, sha: vcs_sha}`.
3. Fallback for a rewritten or pruned commit — for each checkout, walk at most `max_history_scan` commits that touch the
   path on durable refs (`git -C <checkout> rev-list --max-count=<n> --remotes=origin -- <vcs_relpath>`), hashing
   `<commit>:<vcs_relpath>` until one matches. On a match, materialize as above and return the _discovered_ sha so
   callers can report drift.
4. Otherwise `{status: "missing"}` with the checkouts tried.

Follow the existing `Command::new("git")` precedent in `crates/sase_core/src/bead/mutation.rs`. Never write outside
`cache_root`. Bound every git invocation with a timeout and surface failures as `missing`, never as a panic.

**`crates/sase_core/src/artifact_ref/`**

- `ArtifactRefRepositoryWire` gains `#[serde(default)] pub checkout_paths: Vec<String>` — every existing checkout of
  that repo identity, most likely first.
- `resolve_file()`: when the matched index row is VCS-backed, return `status: "vcs_backed"`,
  `locator: "<vcs_repo>@<vcs_sha>:<vcs_relpath>"`, `resolved_path: None`. Resolution stays pure — it must not shell out
  to git or write to disk, so read-only callers (`sase lsp`, `@`-completion, TUI hover) stay cheap. Byte-needing callers
  materialize explicitly.
- Bump `ARTIFACT_REF_PARSE_WIRE_SCHEMA_VERSION` and `ARTIFACT_REF_RESOLUTION_WIRE_SCHEMA_VERSION` from 2 to 3 together;
  `py_artifact_ref_wire_schema_version()` `debug_assert_eq!`s that they match.

**`crates/sase_core_py/src/lib.rs`** — expose `artifact_file_materialize_vcs(request: dict) -> dict`, register it in the
module, and document it in the header comment list alongside `artifact_files_query`.

Tests: a `tempfile` git repo fixture covering the fast path, the cached path, a cache entry with wrong content, a
rewritten sha found by the bounded walk, a pruned/unknown sha, a checkout list where the first entry lacks the object,
and index parsing of byte-free rows, `path`-only rows, and rows with a partial `vcs_*` triple.

## 4. Capture policy — the authorship and version-control decision

New module `src/sase/core/artifact_capture_policy.py`. It has no dependency on the record schema, so it can land in
parallel with the core-record phase.

- `CaptureCandidate(path: str, origin: Literal["changed", "mentioned"], sha256: str, size_bytes: int)`.
- `CaptureDecision(outcome: Literal["store", "reference", "skip"], reason: str, vcs_repo: str | None, vcs_sha: str | None, vcs_relpath: str | None)`
  — `reason` is a short stable slug used in log output and asserted by tests.
- `decide_captures(candidates, *, artifacts_dir, workspace_dir, project, workspace_num, run_started_at, probe, limits) -> list[CaptureDecision]`,
  implementing §2's matrix in order and applying the byte-copy cap.
- `VcsProbe` protocol plus a `GitVcsProbe` implementation providing: repo toplevel for a path; identity lookup for a
  toplevel; durable candidate commits touching a relpath; and blob content digest for `<commit>:<relpath>`.

`GitVcsProbe` details:

- Toplevel: `git -C <dir> rev-parse --show-toplevel`, memoized per containing directory so a run with 179 candidates in
  three directories makes three calls.
- Identity: match the realpath'd toplevel against `collect_repo_inventory(project=<project>)` records — each record's
  `clone_for_workspace(workspace_num).path` and its `path`. The inventory is already ordered primary, sidecar, linked,
  external, so first match wins; the result is `record.name`, which is what the resolver later looks up. An unmatched
  toplevel means "unknown repo": invariant C applies and the candidate cannot be a `reference`.
- Durable commits: resolve the upstream and default remote refs once per repo
  (`git rev-parse --abbrev-ref --symbolic-full-name @{upstream}` and `origin/HEAD`), then
  `git rev-list --max-count=<max_history_scan> <refs...> -- <relpath>`. Using named remote-tracking refs rather than
  `--remotes` keeps the walk bounded on a repo with many remote branches, and satisfies invariant B by construction.
- Blob digests: one `git cat-file --batch` process per repo fed every `<commit>:<relpath>` spec, so the whole run costs
  a couple of git processes rather than hundreds.
- Every invocation gets a timeout; any non-zero exit, timeout, or decode error returns "unknown" and the caller
  downgrades to `store`.

Configuration — new top-level `artifacts` block, following the `tasks.history_limit` vocabulary the research names:

```yaml
artifacts:
  capture:
    # Maximum byte-copying automatic captures per agent run. Version-control-backed
    # reference rows cost no bytes, are not counted, and are never capped.
    max_stored_per_agent: 50
    # Bounded number of commits searched per file when looking for a durable commit
    # that holds its exact content.
    max_history_scan: 20
```

Add the block to `src/sase/default_config.yml` and to `src/sase/config/sase.schema.json` (`additionalProperties: false`,
integer minimums, descriptions written to read well in `sase config` output), plus
`DEFAULT_ARTIFACT_CAPTURE_MAX_STORED_PER_AGENT = 50`, `DEFAULT_ARTIFACT_CAPTURE_MAX_HISTORY_SCAN = 20` and matching
`get_artifact_capture_*()` accessors in `src/sase/config/core.py`, exported from `src/sase/config/__init__.py`. Copy the
fail-open, type-checked shape of `get_task_history_limit()` exactly.

The cap counts `store` outcomes per run. Once reached, remaining `store` candidates become `skip` with reason
`capture_cap`, and the wiring phase prints one summary line. Reference rows stay uncapped; row-count noise is
retention's problem, not capture's.

Symvision will flag this module's public API as unused until the wiring phase consumes it. Add
`--epic-symbol "<this epic's bead id>(<symbol>)"` entries to the `_lint-symvision` recipe in the `Justfile` for each new
public symbol, and delete them in the wiring phase. Do not reach for pragmas — the consumer is a later phase of this
epic.

Tests: table-driven cases over real `tempfile` git repos (see `init_test_git_repo()` in `tests/_sdd_commit_helpers.py`)
covering a clean tracked file whose commit is pushed (`reference`), the same file with uncommitted edits (`store`), a
committed-but-unpushed file (`store`, invariant B), an untracked workspace file (`store`), a file inside the artifacts
directory (`store`), a mentioned tracked file (`reference`), a mentioned untracked repo file (`skip`), a mentioned file
outside any repo (`store`), a file in a repo absent from the inventory (`store`, invariant C), a probe that raises or
times out (`store`, invariant C), and cap exhaustion.

## 5. Python record, doctor, and read surfaces

Depends on the core-record release. Start by raising the `sase-core-rs` pin in `pyproject.toml` to that release and
running `just install`; `sase core health` should pass before anything else.

**`src/sase/core/artifact_file_types.py`**

- `ArtifactFile.path` becomes `str | None`; add `vcs_repo`, `vcs_sha`, `vcs_relpath` as `str | None`.
- Add a `is_vcs_backed` property (path falsy and all three set).
- `artifact_file_from_dict()` must read `path` through `_optional_str()` — today it does `str(data["path"])`, which
  would turn a null into the literal string `"None"` — and must reject a row with neither a path nor a complete triple.
- Raise `ARTIFACT_FILE_INDEX_SCHEMA_VERSION` from 1 to 2 for all writes. `ARTIFACT_FILE_INDEX_SUPPORTED_SCHEMA_VERSIONS`
  already contains 2 on both sides, so no reader changes. Note the known, precedented hazard: a pre-change `sase` that
  rewrites the index would strip the new fields, exactly as it would have stripped `sha256`/`size_bytes`/`mime_type`
  when the prior tranche added them. Accepted; the epic lands as one unit.

**`src/sase/core/artifact_file_helpers.py`**

- `artifact_file_id()` currently hashes `path_key(path)`. Byte-free rows have no path, and an empty path would collapse
  every such row onto one id. Give the identity string a VCS branch that substitutes
  `f"vcs:{vcs_repo}:{vcs_relpath}:{sha256}"` for the path component. It is stable across reruns (idempotent capture),
  distinct per revision (different content, different digest), and mirrors the byte-backed case, whose stored filename
  already embeds a digest prefix.
- Add `artifact_file_dedupe_key(row)` returning the path key for byte-backed rows and the VCS identity for byte-free
  rows, and use it in `dedupe_artifact_files()` — which today keys on `path_key(artifact_file.path)` and would otherwise
  collapse all byte-free rows into one.

**`src/sase/core/artifact_file_explicit.py`** — `store_default_artifact_file()` gains a reference mode: given a decision
with VCS provenance, write an index row with `path=None`, the three `vcs_*` fields, and the already-computed `sha256`,
`size_bytes`, and `mime_type`, without touching `_store_file()`. Keep the existing signature working for byte copies.

**`src/sase/core/artifact_file_doctor.py`** — byte-free rows are healthy, not broken:

- `missing_stored_path_ids` must exclude VCS-backed rows; add a `vcs_reference_rows` count and a
  `vcs_provenance_incomplete_ids` tuple for rows with a partial triple.
- `backfill_artifact_file_index()` must not treat a byte-free row as missing; enrichment for those rows is already
  complete at capture time, so leave them untouched.
- `verify_artifact_file_index()` should verify a VCS-backed row by materializing it and comparing digests, reporting the
  same `ArtifactFileDigestMismatch` shape on failure and a new "unresolvable" bucket when materialization returns
  `missing`. That makes the reference contract continuously auditable rather than assumed.
- Surface the new counts in `src/sase/artifact_cli/doctor.py` output.

**`src/sase/core/artifact_file_query_facade.py`** — raise `ARTIFACT_FILE_QUERY_WIRE_SCHEMA_VERSION` to 2 to match Rust.
`_validate_wire_row()` currently requires `path` to be a non-empty string; change it to require a non-empty `path` _or_
a complete `vcs_*` triple, and add the three fields to the optional-string list.

**Materialization bridge** — new `src/sase/core/artifact_file_vcs.py`:
`materialize_artifact_file(row, *, repositories) -> Path | None` builds the request (cache root
`default_artifact_files_root() / "vcs-cache"`, checkout paths for `row.vcs_repo`, suffix from `vcs_relpath`,
`max_history_scan` from config), calls the `artifact_file_materialize_vcs` binding via `require_rust_binding`, and
returns the materialized path or `None`. Keep it the single Python entry point so no caller invents its own git call.

**Resolver context** — `src/sase/artifact_ref_models.py` `ArtifactRefRepository` gains
`checkout_paths: tuple[Path, ...]` and emits it from `to_wire()`; `src/sase/artifact_ref_context.py` populates it from
`record.clones` (every clone whose `exists` is true, current workspace first) plus `record.path`. The existing singular
`checkout_path` stays, since `artifact_ref_prompt.py`'s `commit:` flow uses it. `record.clones` already carries
`exists`, so this adds no filesystem stats. Add `"vcs_backed"` to `ArtifactRefResolutionStatus` and to
`_RESOLUTION_STATUSES` in `src/sase/artifact_ref_operations.py`, and raise `ARTIFACT_REF_WIRE_SCHEMA_VERSION` from 2
to 3.

**Read surfaces.** `vcs_backed` deliberately fails every existing `status not in {"exact", "drifted"}` gate, so nothing
silently misbehaves; update exactly the four call sites that need bytes:

- `src/sase/artifact_cli/path.py` — on `vcs_backed`, materialize and print the cache path; on failure print the
  provenance-bearing diagnostic and exit 1.
- `src/sase/artifact_cli/open.py` — materialize, then continue into the existing viewer selection unchanged.
- `src/sase/artifact_ref_prompt.py` — `_artifact_ref_replacement()` materializes for `vcs_backed` so an `@file:` ref in
  a prompt still expands to a real path; a materialization failure becomes an `_ArtifactRefFailure`, which already fails
  the launch loudly rather than handing an agent a dangling path.
- `src/sase/ace/tui/widgets/artifacts/files_detail.py` and `src/sase/ace/tui/actions/artifacts_files.py` — `row.path` is
  now optional. Render a `PROVENANCE` section (`repo`, `commit`, `path`, and whether content is cached) instead of a
  missing "Stored" path, and materialize off-thread in the same worker that already loads previews. Never materialize on
  the UI thread; see the `tui_perf` long-term memory before touching the Files pane load path.

`src/sase/artifact_cli/show.py` and `src/sase/artifact_cli/listing.py` need no materialization — `show` reports
`stored_path_status` as `vcs-backed` plus the locator, and `list` already renders metadata only.
`src/sase/ace/tui/widgets/artifacts/files_data.py`'s `artifact_file_view_mode(row.path, ...)` must fall back to
`vcs_relpath` for the suffix.

Tests: index round-trips for byte-free rows, id stability and non-collision across many byte-free rows in one run,
dedupe across mixed rows, doctor counts and verify behavior, query-facade validation of both row shapes,
`sase artifact path`/`open`/`show` against a byte-free row backed by a temp git repo, and a `@file:` prompt-expansion
test.

## 6. Wire the policy into finalization capture

- `_MediaCandidate` in `src/sase/core/artifact_file_defaults.py` gains `origin: Literal["changed", "mentioned"]`.
  `_media_candidates()` labels `image_paths`/`video_paths` entries `changed` and `_discover_prompt_media_candidates()`
  output `mentioned`. This is the authorship signal, and it is already structurally present: those lists come from
  `collect_agent_image_paths()`/`collect_agent_video_paths()`, which derive from `git diff HEAD`, untracked files, and
  the run's own commit.
- `persist_default_artifact_files()` hashes each existing candidate once, calls `decide_captures()`, then writes byte
  copies for `store`, reference rows for `reference`, and nothing for `skip`. It keeps returning the persisted
  `ArtifactFile` list and stays idempotent: a rerun over the same workspace must produce the same rows and the same ids.
- Derive `run_started_at` from the artifacts directory name (`raw_timestamp`, `%Y%m%d%H%M%S`), falling back to the
  directory's own mtime. There is no run-start field in `agent_meta.json`.
- Thread `workspace_num` and `project` from `AgentExecContext` through `_collect_default_artifacts()` in
  `src/sase/axe/run_agent_exec_finalize.py` so identity lookup works. That call site already sits after the commit
  finalizer, so no re-sequencing is needed.
- Print one summary line — stored / referenced / skipped counts and whether the cap fired — beside the existing
  `[artifacts]` output. The existing `except Exception` around the call already degrades a total failure to
  `default_artifacts_persisted=False`; keep that.
- Remove the `--epic-symbol` entries added in the policy phase.

Tests: extend `tests/artifact_file_facade/test_persistence.py` with a temp git repo where one candidate is clean and
pushed (reference row, no bytes on disk), one is dirty (byte copy), one is untracked (byte copy), one is
mentioned-only-and-tracked (reference row), one is mentioned-only-outside-any-repo (byte copy), and one is
mentioned-only-untracked-in-repo (no row). Assert idempotency on rerun. Add an end-to-end case to
`tests/test_artifact_file_e2e.py` that captures, then resolves the reference row back to verified content.

## 7. Docs, skill, and configuration reference

- `src/sase/xprompts/skills/` `sase_artifact_file` source: document that automatic capture now keeps bytes only for
  authored files version control cannot reproduce; that a `file:` reference may be VCS-backed, in which case `show`
  reports a locator and `path`/`open` materialize on demand; and that `create` is unaffected. Read the
  `generated_skills` long-term memory first and regenerate the deployed skill the way that note prescribes.
- Add a "VCS-backed artifact files" section to the artifact documentation under `docs/` covering the decision matrix,
  the three record fields, the `vcs-cache` directory, and how to diagnose an unresolvable reference.
- Document the `artifacts.capture` block wherever config options are documented, and make `sase artifact doctor --help`
  mention the new counts. Follow the `cli_rules` long-term memory: alphabetical options, a short alias for every long
  option, scannable help.

## 8. Explicitly out of scope

- **Recommendation 1, copy-by-default for `sase artifact create`.** Still unfixed: `src/sase/artifact_cli/create.py:36`
  hardcodes `move=True`, and the skill instructs agents to create a file and register it — a sequence that deletes their
  work. It is a two-line change plus docs and it is the precondition that makes declaration usable, but it is a
  different recommendation and is not bundled here. Worth doing first or alongside.
- **Recommendation 3, retention and the retroactive migration** of the rows already in the store. This plan makes that
  migration possible and hands it one measured constraint: it cannot use HEAD. Only 420 of the 4,067 in-workspace rows
  match the content at the primary repo's current tip, while 4,035 are findable somewhere in `origin/master` history —
  so the migration must use the bounded content-addressed search this plan builds in Rust, which it can call directly.
- Content-addressed blob storage for the whole store. The `vcs-cache` directory is a transient, content-keyed
  materialization cache, not a storage-layout migration; deleting it must only cost re-materialization.
- Any change to the reference grammar, the synthesized chat/plan/PDF artifact rows (which store no bytes today), or the
  2.49 GB agent-directory tier.

## 9. Risks

- **Wrong bytes.** Mitigated structurally by invariant A: a reference row is only written after the exact content has
  been reproduced and digest-verified, and `doctor --verify` re-checks it later.
- **Unresolvable references.** Mitigated by invariant B, by multi-checkout resolution, and by the bounded
  content-addressed fallback for squash-rewritten shas. Residual failure is a loud `missing` status naming repo, commit,
  path, and digest — never a silent empty file.
- **Finalization latency.** Bounded by memoized toplevel lookups, one `cat-file --batch` process per repo, a capped
  history walk, and timeouts. Worst observed run is 179 candidates.
- **Cross-repo release coupling.** Three wire constants move in lockstep: `ARTIFACT_FILE_QUERY_WIRE_SCHEMA_VERSION` 1→2,
  `ARTIFACT_REF_*_WIRE_SCHEMA_VERSION` 2→3, and the Python mirrors. Python asserts equality at runtime, so a mismatch
  fails fast and visibly rather than corrupting data. The core-record phase must be released before the py-record phase
  raises the pin.
- **Effort versus the report's estimate.** The research sized this "S–M". It is larger, because the report did not
  account for the `sase-core` release boundary, the blast radius of an optional stored path across ids/dedupe/doctor, or
  the fact that no stable canonical checkout exists. The phase sizes here reflect the real work.

## 10. Verification

Each phase runs `just install` first (ephemeral workspaces drift), then `just check`. In `sase-core`, run that repo's
own check target before committing. Sanity checks after the wiring phase, on a real run:

```bash
sase artifact list -k image -l 5 -j          # rows present; VCS-backed rows carry no stored path
sase artifact show file:default:<id>          # locator + vcs-backed stored status
sase artifact path file:default:<id>          # materializes and prints a cache path
sase artifact doctor                          # 0 missing stored paths; vcs reference rows counted
sase artifact doctor --verify                 # byte-free rows verify by materialization
du -sh ~/.sase/artifacts                      # growth flattens on subsequent runs
```
