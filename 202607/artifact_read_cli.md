---
tier: epic
title: Ship sase artifact as a read CLI, and add three record fields
goal: 'Agents and humans can discover, inspect, resolve, and open any indexed artifact
  from the CLI, and every artifact-file record carries sha256, size_bytes, and mime_type
  with a safe, idempotent backfill.

  '
phases:
- id: core-index-contract
  title: Artifact-file index contract and query API in sase-core
  depends_on: []
  size: medium
  description: 'core-index-contract: extract the index parser into a tolerant full-record
    module in the sase-core repo, accept envelope schema versions 1..=2, add a filterable
    query function reusing plan-search date parsing, and expose artifact_files_query
    plus a wire-schema handshake through the PyO3 bindings.'
- id: record-enrichment
  title: Three record fields, tolerant reader, preserving writer, backfill library
  depends_on: []
  size: medium
  description: 'record-enrichment: add optional sha256, size_bytes, and mime_type
    to the Python ArtifactFile record, populate them at store time by reusing the
    already-computed digest, make the index reader range-tolerant and the writer preserve
    unknown-version rows, and build the inspect/backfill/verify library with tests.'
- id: artifact-read-cli
  title: The sase artifact command group
  depends_on:
  - core-index-contract
  - record-enrichment
  size: large
  description: 'artifact-read-cli: rename the artifact-file group to sase artifact
    with a compatibility alias and add the doctor, list, open, path, and show subcommands
    — list through the new Rust query binding, show/path/open through the landed artifact-reference
    resolver, with chat-list-style Rich output, JSON modes, a strict exit-code contract,
    and full parser and handler tests.'
- id: artifact-skill-docs
  title: Skill template and documentation
  depends_on:
  - artifact-read-cli
  size: small
  description: 'artifact-skill-docs: extend the sase_artifact_file skill template
    from create-only to create-and-read and update the CLI, configuration, agent-images,
    ace, and axe docs to the new command group and record fields.'
create_time: 2026-07-29 17:06:30
status: wip
bead_id: sase-ax
---

- **BEAD:** [sase-ax](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ax/README.md)
- **PROMPT:** [prompts/artifact_read_cli.md](prompts/artifact_read_cli.md)
- **AGENTS:**
  - [bbugyi200.athena.ov](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.ov/README.md)
  - [bbugyi200.athena.sase-ax.1](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-ax.1/README.md)
  - [bbugyi200.athena.sase-ax.2](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-ax.2/README.md)
  - [bbugyi200.athena.sase-ax.3--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-ax.3.md#member-code)
  - [bbugyi200.athena.sase-ax.3--plan](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-ax.3.md#member-plan)
  - [bbugyi200.athena.sase-ax.4](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-ax.4/README.md)
  - [bbugyi200.athena.sase-ax.land](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-ax.land/README.md)

# Ship `sase artifact` as a read CLI, and add three record fields

## Motivation

Research report `202607/artifact_refs_and_inspector/artifact_refs_and_inspector.md` (in the `sase--research` sidecar),
§5 item 3: the artifact-file store holds ~3,985 files / 622 MB, yet `sase artifact-file` has exactly one subcommand
(`create`, agent-gated). Neither an agent nor a human can discover, read, resolve, or open an artifact from the CLI. The
index record (`schema_version: 1` rows in `~/.sase/artifacts/index.jsonl`) carries provenance and both locations but
lacks **`sha256`**, **`size_bytes`**, and **`mime_type`** — the only genuine schema gaps the research identified.
Meanwhile the kind-tagged artifact-reference substrate the research called "item 2" has since fully landed:
`src/sase/artifact_refs.py` is a complete Python facade over the Rust `artifact_ref_*` bindings (parse / render /
canonicalize / resolve / scan, kinds `commit`, `chat`, `bug`, `file`, and document roles such as `plans` / `research` /
`designs`, plus `#L`/`#page=`/`#t=` fragments). This epic spends that resolver in a read CLI and completes the record.

## Design summary

One command group, six subcommands, three new optional record fields, one doctor, zero new stores.

```
sase artifact                    # bare group delegates to `list` (central default-list convention)
sase artifact create ...         # existing behavior, moved from `sase artifact-file create`; now also prints the ref
sase artifact doctor [-f] [-V]   # index health report; --fix backfills sha256/size_bytes/mime_type; --verify re-hashes
sase artifact list [filters]     # pretty Rich table (chat-list style) or -j JSON, backed by a Rust core query API
sase artifact open <ref>         # resolve any artifact ref and open it with the right terminal viewer
sase artifact path <ref>         # print the one resolved absolute path (script/agent composable)
sase artifact show <ref>         # full metadata + resolution report for any artifact ref, or -j JSON
```

`sase artifact-file` remains as a compatibility alias for the group so existing agents and docs keep working.

### Key decisions (and why)

1. **The three new fields are additive and optional at envelope `schema_version: 1` — deliberately NOT a hard bump to
   v2.** Both existing readers pin exact equality — `_read_index_unlocked` at
   `src/sase/core/artifact_file_explicit.py:252` and `read_artifact_index` in the sase-core repo at
   `crates/sase_core/src/artifact_ref/mod.rs` (there is even a Rust test asserting v2 rows are ignored). The index is
   one global mutable file (`~/.sase/artifacts/index.jsonl`) rewritten wholesale on every upsert and shared by many
   workspace venvs of mixed ages. If we hard-bumped, one stale-venv agent running `sase artifact-file create` would
   reread the index, silently skip every v2 row, and rewrite the file without them — destroying all migrated rows. With
   additive optional fields, a stale writer at worst strips the enrichment fields (recoverable by re-running
   `sase artifact doctor --fix`); no row is ever lost. Both readers become range-tolerant (schema versions 1..=2) and
   the writer learns to preserve rows with unrecognized schema versions verbatim, so a future coordinated flip to
   writing v2 envelopes becomes a trivial follow-up once no pre-change writers remain. This honors the research's intent
   (new fields + backfill, no broken readers) while removing a data-loss hazard the research did not examine.
2. **Backfill everything, not just explicit artifacts** (research open question 2): a one-time sequential hash of 622 MB
   is cheap on local NVMe, and uniform fields keep `list`/`show`/`open` honest for all rows. New rows get the fields for
   free at store time — `_store_file` already computes the full SHA-256 and throws away all but 12 hex chars; we thread
   the full digest out instead of re-hashing.
3. **The list/query API lives in the Rust core** per the `rust_core_backend_boundary` memory litmus test: the mobile
   gateway and a future ACE Files sub-tab need list semantics identical to the CLI. sase-core already parses the index
   (id + path) for `file:` ref resolution; extending that to a tolerant full-record parse plus one filter function is a
   small, additive step — not a rebuild of the deleted artifact graph. Python keeps ownership of writes (store, upsert,
   doctor) and of presentation.
4. **`show` / `path` / `open` accept any artifact reference**, not just `file:` — an agent holding `chat:…`, `plans:…`,
   `research:…`, `commit:…`, or `bug:…` can resolve it through one verb. `path` is restricted to kinds that resolve to a
   filesystem path (`file`, `chat`, document roles) and fails with a clear hint otherwise; `open` additionally opens
   `bug:` refs in the browser and points `commit:` refs at `show`.
5. **CLI conventions** (`cli_rules` memory): subcommands and options registered alphabetically, every public long option
   gets a short alias, colored Rich output modeled on `sase chat list` (Panel + table, cyan border), bare group
   delegates to `list` via the central `_default_list_subcommands()` wiring (document the default in the group help).
   `--since` accepts the same DATE forms as `sase plan search` (`YYYY-MM-DD`, `YYYY-MM`, `YYYYMM`, `Nd`/`Nw`/ `Nm`) with
   the real parsing in the Rust core for cross-command consistency.
6. **Project names, never keys** (`gotchas` memory): filters accept a display name, alias, or key; all human-facing
   output renders the configured display name. Keys remain identity in JSON output and storage.

### Out of scope

TUI changes (copy-mode gates, Files sub-tab, `PreviewPanelModal`, prompt-bar completion — research items 1, 4, 5, 8);
any new artifact store, registry, or second index (research §6); re-minting artifact ids; flipping the written envelope
version to 2 (explicit follow-up once fleet saturation makes it safe).

## Phases

### Phase `core-index-contract` — artifact-file index contract and query API in sase-core

**Repo:** the `sase-core` linked repo (open with `/sase_repo`). **Depends on:** nothing.

1. Introduce an artifact-file index module (e.g. `crates/sase_core/src/artifact_file.rs`) that owns the index contract
   currently embedded in `crates/sase_core/src/artifact_ref/mod.rs` (`ArtifactIndexEnvelope`, `ArtifactIndexRow`,
   `read_artifact_index`):
   - Full-record row struct: required `id` and `path`; optional `label`, `kind`, `source_path`, `workspace_dir`,
     `created_at`, `agent_artifacts_dir`, `project`, `workflow`, `raw_timestamp`, `agent_name`, `explicit` (default
     false), and the new optional enrichment fields `sha256`, `size_bytes`, `mime_type`. Unknown JSON fields are ignored
     (serde default), malformed lines are skipped — matching today's tolerance.
   - Accept envelope schema versions in a supported range `1..=2` (replace the exact-equality filter). Update the
     existing test that asserts v2 envelopes are ignored (`indexed_file_resolution_uses_schema_one_envelopes`) to the
     new contract: v1 and v2 rows are both read; versions outside the range are skipped.
   - Refactor `artifact_ref`'s `read_artifact_index` / `read_artifact_index_entries` to consume this module so `file:`
     resolution and canonicalization share one parser. Behavior of `resolve_file` is unchanged.
2. Add a query function `query_artifact_files(index_path, filters) -> Vec<row>` with filters: `kinds` (multi), `project`
   (exact key match), `agent` (exact match on `agent_name`), `since` (reuse the existing plan-search date parsing so
   `sase plan search --since` and `sase artifact list --since` accept identical forms; compare against `created_at`,
   treating rows without a parsable timestamp as non-matching when a `since` filter is set), `explicit_only`, `query`
   (case-insensitive substring over label, path, and source_path), `limit`. Results sort newest-first by `created_at`
   (missing timestamps last).
3. Expose PyO3 bindings in `crates/sase_core_py/src/lib.rs` following the `artifact_ref_*` conventions:
   `artifact_files_query(index_path, filters_dict) -> list[dict]` where each row dict carries a `schema_version` field,
   plus an `artifact_file_query_wire_schema_version()` handshake binding (start at 1), mirroring
   `artifact_ref_wire_schema_version`.
4. Tests: envelope range acceptance (v1 row, v2 row, v3 skipped, malformed line skipped, unknown fields ignored); every
   filter individually and combined; date-form parity with plan search; sort order and limit; wire schema handshake. Run
   the crate test suites per the repo's standard workflow.
5. Ship per sase-core release conventions so the sase repo can pick the binding up with a routine `sase-core-rs` floor
   bump (the current pin is `sase-core-rs>=0.12.12,<0.13.0`; this is an additive 0.12.x change).

**Acceptance:** new bindings importable from `sase_core_rs`; all sase-core tests pass; `file:` ref resolution behavior
unchanged for v1 indexes.

### Phase `record-enrichment` — three record fields, tolerant reader, preserving writer, backfill library

**Repo:** sase (this repo). **Depends on:** nothing (pure Python domain work; parallel with `core-index-contract`).

1. `src/sase/core/artifact_file_types.py`: add `sha256: str | None = None`, `size_bytes: int | None = None`,
   `mime_type: str | None = None` to `ArtifactFile`; round-trip them in `artifact_file_to_dict` /
   `artifact_file_from_dict` (absent in old rows → `None`). Add
   `ARTIFACT_FILE_INDEX_SUPPORTED_SCHEMA_VERSIONS = frozenset({1, 2})` alongside the existing write-version constant
   (which stays 1 — see design decision 1); document both constants' roles where they are defined.
2. New helper `artifact_file_mime_type(path) -> str` in `src/sase/core/artifact_file_helpers.py`: an explicit suffix map
   for SASE-known kinds first (e.g. `.md` → `text/markdown`, `.png` → `image/png`) so results are deterministic across
   Python versions, then `mimetypes.guess_type`, falling back to `application/octet-stream`.
3. `src/sase/core/artifact_file_explicit.py`:
   - `_store_file` returns the full SHA-256 it already computes (today it truncates to 12 chars for the filename and
     discards the rest); `store_explicit_artifact_file` and `store_default_artifact_file` populate `sha256`,
     `size_bytes` (stat of the stored file), and `mime_type` on every new row.
   - Reader (`_read_index_unlocked`): accept any version in `ARTIFACT_FILE_INDEX_SUPPORTED_SCHEMA_VERSIONS`.
   - Writer (`_upsert_index_row` / `_write_index_unlocked`): preserve verbatim any line whose schema version is outside
     the supported set instead of dropping it (fixes the latent whole-index destruction hazard on rewrite).
4. Backfill + health library (new module, e.g. `src/sase/core/artifact_file_doctor.py`), consumed by the CLI in the next
   phase but fully tested here:
   - `inspect_artifact_file_index(...)` → report: total rows, rows missing enrichment fields, stored paths missing,
     source paths missing (informational — expected for recycled workspaces), duplicate ids, rows preserved with
     unrecognized schema versions.
   - `backfill_artifact_file_index(...)` → for every row whose stored path exists, compute any missing `sha256` /
     `size_bytes` / `mime_type`; rewrite atomically under the existing index lock; idempotent (second run is a no-op);
     rows whose stored file is missing are reported, never guessed.
   - `verify_artifact_file_index(...)` → re-hash stored files and report mismatches against recorded `sha256`
     (corruption detection, report-only).
5. Tests (`tests/` mirroring existing artifact-file test layout): round-trip of new fields; old v1 rows load with `None`
   enrichment; unknown-version lines survive an upsert rewrite byte-for-byte; store-time population (including digest
   equality with an independently computed hash); backfill idempotency; verify catches a tampered stored file; mime
   determinism for known suffixes; byte-identical files from two agents keep distinct ids and equal `sha256`.

**Acceptance:** `just check` passes; new artifacts carry all three fields; a v1 index with a foreign v3 line survives
create/backfill round-trips with the v3 line intact.

### Phase `artifact-read-cli` — the `sase artifact` command group

**Repo:** sase (this repo). **Depends on:** `core-index-contract`, `record-enrichment`.

1. Bump the `sase-core-rs` floor in `pyproject.toml` to the release from `core-index-contract`.
2. Parser: evolve `src/sase/main/parser_artifact_file.py` into the `sase artifact` group (rename module to
   `parser_artifact.py`) registered with `aliases=["artifact-file"]` for compatibility. Subcommands registered
   alphabetically: `create`, `doctor`, `list`, `open`, `path`, `show`. Every long option gets a short alias; help text
   documents that bare `sase artifact` delegates to `list` (the central `_default_list_subcommands()` wiring picks the
   group up automatically — add a test asserting the delegation notice). Update the dispatch in `src/sase/main/entry.py`
   and evolve `src/sase/main/artifact_file_handler.py` into `artifact_handler.py`, following the `sase chat` pattern of
   thin dispatch plus per-subcommand presentation modules (new package, e.g. `src/sase/artifact_cli/`).
3. Thin query facade (e.g. `src/sase/core/artifact_file_query.py`): calls `require_rust_binding("artifact_files_query")`
   with the wire-schema handshake, maps wire rows into `ArtifactFile` values. This is the query path for `list`;
   Python's `read_artifact_file_index` remains the writer-side reader. Add a parity test: one fixture index read by both
   paths yields identical records.
4. Subcommand behavior:
   - **`create`**: unchanged semantics and gating (`SASE_AGENT=1`, `SASE_ARTIFACTS_DIR`); additionally prints
     `ref: file:<id>` so the artifact is born with its durable, copyable name.
   - **`list`**: filters `-a/--agent`, `-e/--explicit` (explicit artifacts only), `-j/--json`, `-k/--kind` (repeatable,
     choices from `ARTIFACT_FILE_KINDS`), `-l/--limit` (default 50), `-p/--project` (accepts display name, alias, or
     key; resolved to the stored key before querying), `-q/--query`, `-s/--since` (plan-search DATE forms, validated
     syntactically in argparse like `plan_date_arg`). Pretty output: Rich Panel + table in the `sase chat list` style —
     columns KIND, REF (the copyable `file:<id>`), LABEL, PROJECT (display name), AGENT, SIZE (humanized), CREATED
     (minute-precision ISO). JSON output: full records including `sha256`, `size_bytes`, `mime_type`, and the rendered
     `ref`, stable schema.
   - **`show <ref>`**: accepts a full artifact ref of any kind, or a bare index id (`default:<hash>` /
     `explicit:<hash>`) as sugar for `file:<id>`. For `file:` refs: metadata panel with every record field, the rendered
     ref, and liveness (stored path exists ✓; source path labeled live/missing). For other kinds: parsed kind, canonical
     rendered ref, resolution status, resolved path or locator, and candidates when ambiguous. Resolution context comes
     from `src/sase/artifact_refs.py` (reuse the launch-context assembly used by `process_artifact_references`, exported
     as a public helper). `-j/--json` emits the record plus resolution.
   - **`path <ref>`**: prints exactly one absolute path on stdout. Exit 0 on success; exit 1 with stderr diagnostics
     (status plus candidate list) for missing/ambiguous/unknown; exit 2 with a hint to use `show` for kinds without a
     filesystem identity (`commit:`, `bug:`).
   - **`open <ref>`**: resolves like `path`, then dispatches on kind/mime, reusing the existing viewer command builders
     from `src/sase/ace/tui/graphics/` (they are plain functions, importable without the TUI app): text-like →
     `artifact_text_viewer_command` (bat, or the safe dump-module fallback); image → `kitten_icat_command` when kitty
     graphics are available, otherwise a clear message naming the path; video → the bounded mpv builder; pdf/unknown →
     `xdg-open` when available. `bug:` refs open the issue URL in the browser (reuse `issue_url_for_number`); `commit:`
     refs error with a pointer to `show`. Paths are always passed with `--` argument boundaries.
   - **`doctor`**: presentation over the `record-enrichment` library — colored report by default (exit 1 when problems
     are found, 0 when healthy), `-f/--fix` runs the backfill and reports what changed, `-V/--verify` re-hashes stored
     content. Read subcommands (`doctor`, `list`, `open`, `path`, `show`) are **not** agent-gated.
5. Tests: parser registration (alphabetical order, alias resolves, delegation notice fires on bare group, help snapshots
   if conventional); handler tests over tmp indexes and fixture refs for every subcommand — exit-code contract, JSON
   schema stability, display-name rendering, bare-id sugar, ambiguous (multiple candidates) and missing refs, fragment
   echo on `show`; `open` command construction for each kind including a host without `bat` and filenames beginning with
   `-` or containing spaces; research §7 edge cases where CLI-reachable.

**Acceptance:** `just check` passes; `sase artifact` / `sase artifact-file` both work; an agent can round-trip
`create → list → show → path → open` entirely from the CLI.

### Phase `artifact-skill-docs` — skill template and documentation

**Repo:** sase (this repo). **Depends on:** `artifact-read-cli`.

1. Extend `src/sase/xprompts/skills/sase_artifact_file.md` from create-only to create-and-read: update the frontmatter
   description; document `sase artifact create` (with the new `ref:` output line), `list`, `show`, `path`, and `open`
   with agent-oriented examples (find prior artifacts for a project, resolve a ref someone handed you to a concrete
   path, hand a ref to another agent); note that read subcommands work outside agent runs. Keep the skill name
   `sase_artifact_file`. Deployment of the regenerated skill happens after this phase lands on the canonical branch
   (`sase skill init --force` from the clean merged tree, per the generated-skills workflow) — do not deploy from a
   phase workspace.
2. Docs: update `docs/cli.md` (replace the single `sase artifact-file create` row with the `sase artifact` group, noting
   the alias and the bare-`list` default), `docs/configuration.md` (§ `sase artifact-file` → `sase artifact`, new
   subcommand/option table), and the `sase artifact-file create` spellings in `docs/agent_images.md`, `docs/ace.md`, and
   `docs/axe.md` to the new canonical `sase artifact create` (mentioning the alias once). Document the three new record
   fields and the doctor in the explicit-artifact contract section of `docs/agent_images.md`.
3. Run the repo docs/lint checks via `just check`.

**Acceptance:** `just check` passes; skill template and all five doc files teach the new surface; no generated provider
instruction files or canonical memory files are touched.

## Verification (epic level)

- `just install && just check` green in the sase repo after each sase phase; sase-core test suite green after
  `core-index-contract`.
- Manual smoke on real data: `sase artifact doctor` reports the expected enrichment gap before `--fix` and a clean bill
  after; `sase artifact list -k image -l 5` renders display names and refs; `sase artifact path` on a `plans:` ref
  prints the same path `sase bead show` renders for that ref; `sase artifact open` on a markdown artifact pages through
  bat.
- Research §7 validation cases covered by tests where CLI-reachable: recycled-workspace source paths (labeled, never
  emitted as bare relative paths), byte-identical artifacts with distinct provenance, missing/corrupt stored objects
  (doctor), `-`-prefixed and space-containing filenames, no-`bat` hosts.
