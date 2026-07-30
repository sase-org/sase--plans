---
tier: epic
title: Persist artifact references on beads and ChangeSpecs
goal: 'A bead and a ChangeSpec each carry a durable, ordered list of canonical artifact
  references that survives machines, workspaces, and store rebuilds: one shared Rust
  codec parses, normalizes, and batch-resolves reference lists for every caller; `sase
  bead ref` and `sase changespec ref` attach and detach them; `sase bead show`, the
  ACE surfaces, bead pages, and the mobile bridge render the stable reference and
  where it currently resolves; and `sase bead doctor` and `sase doctor` report references
  that resolve nowhere instead of silently passing.

  '
phases:
- id: core-refs
  title: Shared reference-list codec and the ChangeSpec REFS section
  depends_on: []
  size: medium
  description: 'core-refs: add the Rust parse/normalize/render/batch-resolve API for
    stored artifact-reference lists, expose it through the PyO3 binding, and teach
    the Rust ChangeSpec parser the new REFS section behind a wire-schema bump.

    '
- id: core-beads
  title: The bead refs field in the Rust core
  depends_on:
  - core-refs
  size: medium
  description: 'core-beads: give beads a `refs` list with its own add and remove events,
    SQLite column and migration, JSONL codec, `sase bead ref` and `sase bead create
    --ref` handling, show rendering, search coverage, and doctor diagnostics.

    '
- id: beads
  title: Python bead refs, show, and doctor
  depends_on:
  - core-beads
  size: medium
  description: 'beads: raise the core floor, add the Python reference-list facade,
    mirror the new bead field across the model, codecs, and JSON wire, render a resolved
    REFS section in `sase bead show`, and thread the reference context into `sase
    bead doctor`.

    '
- id: changespecs
  title: The ChangeSpec REFS section in Python, CLI, and ACE
  depends_on:
  - beads
  size: medium
  description: 'changespecs: parse, format, and atomically persist the REFS section
    in Python, consolidate the duplicated section-boundary tables onto one constant,
    add `sase changespec ref`, render REFS in the ACE CLI and TUI, and add the `sase
    doctor` validation check.

    '
- id: surfaces
  title: Published pages, ACE Plans tab, mobile bridge, and declaration
  depends_on:
  - beads
  size: small
  description: 'surfaces: render bead references on published pages and in the ACE
    Plans detail panel, return them through the mobile bridge, and let `sase artifact
    create` attach the reference it just minted to a bead.

    '
- id: docs
  title: Documentation, skills, and the live-store audit
  depends_on:
  - changespecs
  - surfaces
  size: small
  description: 'docs: document the bead field and the ChangeSpec section, update the
    affected skill sources, regenerate the deployed skills, and audit both live stores
    with the new validators.

    '
create_time: 2026-07-30 10:53:32
status: wip
bead_id: sase-bb
---

- **PROMPT:** [202607/prompts/spec_artifact_references.md](prompts/spec_artifact_references.md)
- **BEAD:** [sase-bb](https://github.com/sase-org/sase--beads/blob/main/pages/sase-bb/README.md)

# Plan: Persist artifact references on beads and ChangeSpecs

This implements recommendation 7 of `research:202607/artifact_capture_and_retention/artifact_capture_and_retention.md`
("Persist artifact references on beads and ChangeSpecs"), which asks to rerun the `sase-9z` playbook — canonical scheme,
shared resolver, persist on new records, render reference plus resolved path, validate with a doctor — against the
generalized reference grammar rather than against `plans:` alone.

It is sequenced after `sase-b7` ("Make artifact capture mean authorship and stop copying what version control stores"),
which is landing now. That epic owns the `sase-core-rs` release that raises the artifact-reference wire from 2 to 3.
This workspace currently fails every `parse_artifact_ref` call with
`artifact-reference wire is stale: expected 3, got 2`, so `sase-b7.5` must land and its release must publish before this
epic's first Rust phase can build on a green tree. Nothing in this plan changes capture, retention, or the reference
grammar itself.

## 1. Context and measurements

All figures below were derived on 2026-07-30 from the live `sase` bead store (`sase/repos/beads/issues.jsonl`, 2,389
records), the ChangeSpec stores under `~/.sase/projects/` (11 project files), and the current working tree.

### What is durable today, and what is not

`sase-9z` proved the pattern on exactly one pointer. 387 beads carry a `design` value; 341 are already canonical
`plans:` references and 46 remain legacy absolute or `sdd/plans/...` paths that the repair pass could not recover. One
field, one kind, one bead each. The grammar now covers `commit`, `chat`, `bug`, `file`, `bead`, `agent`, and any
document role configured as an SDD sidecar (`plans`, `research`), yet **no bead can hold any reference other than its
single plan link, and no ChangeSpec can hold any reference at all.**

The demand is visible but has nowhere to go. 16 beads carry reference-shaped tokens inside free-text `description` or
`notes` — `research:`, `file:`, `agent:`, `bead:` — where nothing parses, resolves, or validates them. The research
report itself is referenced from the `sase-b7` plan body as prose. A reference in prose is a string; the point of the
grammar is that a reference in a field is a resolvable pointer.

### The ChangeSpec store already stores pointers, and stores them as paths

ChangeSpecs are not reference-free by design; they persist three kinds of pointer today, all as machine-local paths, on
COMMITS sub-lines:

| Sub-line   | Occurrences | Resolve now | Dead |
| :--------- | ----------: | ----------: | ---: |
| `\| DIFF:` |          60 |          60 |    0 |
| `\| CHAT:` |          46 |          44 |    2 |
| `\| PLAN:` |           8 |           5 |    3 |
|            |             |             |      |

Three of eight plan pointers are already dead in a store that is 24 days old (the oldest ChangeSpec is the 2026-07-06
entry, matching the `~/.sase` rebuild the research report flags in its open question 3). That is the `sase-9z` pathology
reproducing in a second store. It is _not_ this epic's job to fix — see §7 — but it is the evidence that a ChangeSpec
needs a field whose contents are references rather than paths.

The corpus is small in absolute terms: 42 ChangeSpecs across 11 project files, 38 of them in the `sase` archive. That
matters for sizing, not for value. Beads are durable memory; ChangeSpecs are the in-flight record of one change, and the
reference a reviewer wants — "which research note justifies this?" — has no home in either.

### The grammar accepts more than it should, and only resolution can tell

`parse_artifact_ref` in `crates/sase_core/src/artifact_ref/mod.rs` validates the kind label against `[a-z][a-z0-9_-]*`
and then routes anything unrecognized to `ArtifactRefKindWire::Document { role }`. So `reserch:202607/foo.md` parses
cleanly and renders cleanly; only _resolution_ against `context.document_roots` reports `unknown_kind`. A writer that
validates by parsing therefore cannot catch a mistyped kind.

This plan resolves that the way `design` already does: **writers validate by parsing, and `doctor` validates by
resolving.** Resolution needs the full `ArtifactRefContext` — document roots, chats root, artifact index path, the repo
inventory, project records, bead stores, agent roots — which `artifact_ref_context()` assembles from
`collect_repo_inventory()` and `list_project_records()`. That is far too expensive to build inside every `sase bead`
invocation, which is dispatched through the Rust fast path in `src/sase/main/bead_fast_path.py` precisely to stay cheap.
Keeping resolution out of the write path is a requirement, not a simplification.

### Resolving a list one reference at a time is quadratic

`resolve_file()` calls `read_artifact_index(...)` — a full read and parse of `~/.sase/artifacts/index.jsonl`, currently
4,300 rows — **once per reference**. A bead with five `file:` references costs five whole-index reads in
`sase bead show`; a doctor pass over a store where a hundred beads carry one costs a hundred. This is why the shared API
in this plan is a _list_ codec with a batched resolve, not a loop over the existing single-reference call.

### Adding a ChangeSpec section is currently a five-file hazard

`CHANGESPEC_SECTION_ORDER` in `src/sase/ace/deltas/persistence.py` is the canonical section list. Four other modules
carry hand-maintained copies of it, and **DELTAS is missing from all four** — it was added by `sase-13` and never
propagated:

- `src/sase/ace/hooks/formatting.py::_CHANGESPEC_FIELD_HEADERS` (also missing `PARENT:`, the review-URL prefixes, and
  `BUG:`)
- `src/sase/status_state_machine/field_updates.py::_FIELD_HEADERS`
- `src/sase/workflows/commit_utils/modifiers.py` (three inline tuples)
- `src/sase/workflows/accept/renumber.py` (one inline tuple); `src/sase/workflows/rewind/renumber.py` mirrors its shape

Each of these tables answers "where does the current section end?". A section absent from the table is silently absorbed
into the preceding one. A new REFS section placed before COMMITS would be absorbed into DESCRIPTION by
`field_updates.py` on the next reword. The `changespecs` phase must consolidate onto the single constant rather than add
a sixth copy.

### Already true and de-risking

- The bead store is event-sourced, and `DependencyAdded`/`DependencyRemoved` in `crates/sase_core/src/bead/events.rs`
  are an exact precedent for per-element add and remove events on a multi-valued bead field. That shape also avoids the
  lost-update that whole-list replacement would cause when two agents attach different references to the same bead.
- `ALTER TABLE issues ADD COLUMN` migrations are routine: `model_migration_sql()`, `is_ready_to_work_migration_sql()`,
  and the size and changespec migrations in `crates/sase_core/src/bead/schema.rs` are all the same three-function shape.
- `design_reference_diagnostics()` in `crates/sase_core/src/bead/read.rs`, with its
  `PlanRootMode::NotRequested / Unavailable / Available` tri-state, is the exact template for reference diagnostics that
  degrade cleanly when the context cannot be built.
- The ACE Plans tab already renders "stable reference first, resolved path second" for `design`
  (`plans_detail.py::_plan_reference_properties`) off a precomputed snapshot, so both the wording and the
  off-render-path resolution pattern already exist.
- `sase bead dep` is an existing `add`/`list`/`rm`/`tree` group that bare-invokes to `list`, so `sase bead ref` needs no
  new CLI convention.
- ACE already renders a canonical reference for a selected row through the copy-as palette
  (`src/sase/ace/tui/actions/clipboard/_artifact_reference_resolution.py`). This epic gives the reference it produces a
  place to live.

## 2. Design

### The stored value

A reference list is an **ordered, deduplicated sequence of canonically rendered artifact references**. Every entry is
the output of `render_artifact_ref(parse_artifact_ref(value))`, so a stored list is byte-stable regardless of which
surface wrote it. Deduplication is by rendered form and preserves first-write order.

There is no role, label, or annotation on an entry in this version. The kind already carries most of the semantics
(`research:` is a report, `file:` is a declared artifact, `agent:` is a producer), and a role vocabulary is the shape
the research report reserves for the _consumption_ edge in its item 5, not for this one. The extension path stays open
because an entry is a single token today: a later `role=` prefix is additive and needs no migration.

**Beads** gain `refs: list[str]`, alongside `design` and independent of it. `design` keeps its exclusive meaning — the
one plan that produced this bead — and this phase does not touch it. Storage:

- JSONL: `"refs":["research:...","file:default:..."]`, omitted entirely when empty, matching the conditional-emission
  style `tier` and `resolution` already use in `_issue_to_dict`. Omitting keeps `issues.jsonl` from churning 2,389 rows
  on first write after upgrade and keeps the Rust and Python exporters byte-identical, which `doctor`'s projection-drift
  check compares directly.
- SQLite: `refs TEXT NOT NULL DEFAULT ''`, newline-joined, added by an `ALTER TABLE` migration.
- Events: `ReferenceAdded` and `ReferenceRemoved`, each carrying one rendered reference.

**ChangeSpecs** gain a `REFS:` section, one two-space-indented reference per line, positioned between `STATUS:` and
`COMMITS:` in `CHANGESPEC_SECTION_ORDER`:

```
STATUS: Draft
REFS:
  research:202607/artifact_capture_and_retention/artifact_capture_and_retention.md
  file:default:0f3a9c2b
COMMITS:
```

The section is declarative metadata about the change, so it reads before the activity log, and its placement keeps every
activity section (COMMITS, DELTAS, HOOKS, COMMENTS, MENTORS, TIMESTAMPS) contiguous.

`REFS` rather than `REFERENCES` is a deliberate choice: it matches the `refs` bead field, the `artifact_ref_*` module
family, and the `sase bead ref` / `sase changespec ref` command groups. Naming consistency across the whole feature is
worth more than matching the longest existing section header.

### The shared codec

Reference-list semantics are core backend behavior under the `rust_core_backend_boundary` litmus test: the bead CLI, the
ChangeSpec parser, the TUI, the mobile bridge, and any future frontend must agree on what a stored list means. The codec
lives in `sase-core` and owns four operations, mirroring `crates/sase_core/src/plan/refs.rs`:

- **Parse** a stored list into typed entries, rejecting a malformed entry with the grammar's own error rather than
  silently dropping it.
- **Normalize** a caller-supplied list: parse, render canonically, dedupe by rendered form, preserve order.
- **Render** a normalized list back to its stored form for each store (JSON array for beads, indented lines for
  ChangeSpecs).
- **Resolve** a whole list against one `ArtifactRefContextWire` in a single call, returning a per-entry outcome and
  reading any shared corpus — the artifact index above all — at most once.

Python gets one facade module so nothing outside it invents its own list handling, exactly as `sase.sdd.plan_refs` is
the single entry point for plan references.

### Where resolution may and may not happen

- **Yes**: `sase bead show`, `sase bead doctor`, `sase doctor`, `sase bead ref list --resolve`,
  `sase changespec ref list --resolve`, and the ACE Plans-tab snapshot loader.
- **No**: any TUI render path, any keystroke path, and the `sase bead` fast path. `tui_perf` rules 8 and 11 are
  categorical — render paths never stat or glob, keystroke paths never call side-effectful resolvers — and building an
  `ArtifactRefContext` walks the repo inventory and project records. The ACE ChangeSpec detail panel therefore renders
  the **stored reference verbatim**. That is not a degraded rendering: a canonical reference is already the
  human-legible, machine-stable form, which is the entire argument for storing references instead of paths.

Adding resolved paths to a TUI panel later is a snapshot-loader change, not a render change, and `plans_data.py`'s
`linked_plan_documents` is the pattern to copy.

### Write-time validation

A writer accepts a reference when it parses, stores the rendered form, and does not check resolvability. This is exactly
how `design` behaves and it is the correct trade-off for a durable pointer: a reference may legitimately not resolve on
this machine — an `agent:` from another host, a `research:` document not yet pushed to the sidecar — and refusing to
store it would make the field useless for the cross-machine case it exists to serve. `doctor` reports what does not
resolve, including the mistyped-kind case that parsing cannot catch.

## 3. Shared reference-list codec and the ChangeSpec REFS section

Open the core repo the sanctioned way and use the path it prints:
`sase repo open sase-core -r "Add the artifact-reference list codec and the ChangeSpec REFS section"`. release-plz owns
the version; do not hand-edit `Cargo.toml` versions. The ChangeSpec wire bump makes this a breaking change for older
`sase` installs, so use a `feat!:` commit so release-plz computes a breaking release.

**New `crates/sase_core/src/artifact_ref/list.rs`**, re-exported from `artifact_ref/mod.rs`:

- `parse_artifact_ref_list(entries) -> Result<Vec<ParsedArtifactRefWire>, ArtifactRefError>` — parse each entry through
  the existing `parse_artifact_ref`, failing on the first malformed entry with an error naming the offending value and
  its position.
- `normalize_artifact_ref_list(entries) -> Result<Vec<String>, ArtifactRefError>` — parse, render, dedupe by rendered
  form, preserve first-occurrence order.
- `resolve_artifact_ref_list(entries, context) -> ArtifactRefListResolutionWire` — one entry per input, each carrying
  the rendered reference and its `ArtifactRefResolutionWire`. Read the artifact index at most once for the whole call
  and share it across every `file:` entry, and share any other corpus a kind loads more than once. This is the phase's
  one real performance requirement; a naive loop over `resolve_artifact_ref` is not an acceptable implementation.
- An entry that fails to parse resolves to a synthetic `unknown_kind` outcome rather than aborting the batch, so a
  doctor pass over a store with one bad row still reports every other row.

Give the resolution wire its own `ARTIFACT_REF_LIST_RESOLUTION_WIRE_SCHEMA_VERSION` constant following the convention in
`crates/sase_core/src/artifact_ref/`, and do not change `ARTIFACT_REF_PARSE_WIRE_SCHEMA_VERSION` or
`ARTIFACT_REF_RESOLUTION_WIRE_SCHEMA_VERSION`, which `sase-b7` has just moved to 3.

**`crates/sase_core/src/parser.rs` and `crates/sase_core/src/wire.rs`**:

- `ChangeSpecWire` gains `refs: Vec<String>` with `#[serde(default)]`, defaulting to empty so every existing consumer
  keeps working.
- `try_section_header` learns `REFS:` beside the existing six; `ParserState` gains `in_refs` and a `refs` accumulator,
  reset by `reset_section_flags`; `parse_section_content` collects trimmed non-empty lines while `in_refs`. Store the
  raw text of each line — normalization belongs to the writer, and the parser must round-trip whatever is on disk so a
  hand-edited malformed entry is visible to `doctor` rather than erased.
- Raise `CHANGESPEC_WIRE_SCHEMA_VERSION` from 4 to 5 in both `crates/sase_core/src/wire.rs` and `src/sase/core/wire.py`,
  and add 5 to `SUPPORTED_CHANGESPEC_WIRE_SCHEMA_VERSIONS` in `src/sase/core/wire.py` alongside 2, 3, and 4. The
  Python-side constant and dataclass field move in this phase because `crates/sase_core/tests/python_wire_parity.rs`
  pins the two shapes against each other; the Python _behavior_ that consumes the field lands in `changespecs`.
- Add `refs` to `get_searchable_text` in `crates/sase_core/src/query/searchable.rs` and to its Python mirror
  `src/sase/ace/query/searchable.py` in the same commit, since that function is documented as an exact mirror and has
  parity tests.

**`crates/sase_core_py/src/lib.rs`** — expose `artifact_ref_list_normalize(entries: list[str]) -> list[str]`,
`artifact_ref_list_parse(entries: list[str]) -> list[dict]`,
`artifact_ref_list_resolve(entries: list[str], context: dict) -> dict`, and
`artifact_ref_list_resolution_wire_schema_version() -> int`; register them in the module init and document them in the
header comment list beside `artifact_ref_resolve`.

Tests: round-trip normalize/parse for every kind including a document role and a line fragment; dedupe preserving order;
rejection of a malformed entry, of an entry carrying the `@` sigil, and of an empty entry; a batch resolve over a mix of
kinds asserting the index is read once (assert on a counting fixture, not on wall time); a batch containing one
unparseable entry reporting `unknown_kind` for it and real outcomes for the rest; ChangeSpec parsing of a REFS section
in canonical position, of a REFS section at end-of-spec, of a spec with no REFS section, and of a REFS entry that is not
a valid reference.

## 4. The bead refs field in the Rust core

Second `sase-core` release. Depends on the codec from the previous phase.

**`crates/sase_core/src/bead/schema.rs`** — add `refs TEXT NOT NULL DEFAULT ''` to `BEAD_SQLITE_SCHEMA`, plus
`needs_refs_migration()` and `refs_migration_sql()` copying `needs_model_migration()` / `model_migration_sql()` exactly,
and register the migration where the others run.

**`crates/sase_core/src/bead/jsonl.rs` and the issue wire** — `IssueWire` gains `refs: Vec<String>` with
`#[serde(default, skip_serializing_if = "Vec::is_empty")]`. The skip is required: without it the next export rewrites
all 2,389 rows, and `doctor`'s `export_issues_to_jsonl` projection-drift comparison would flag every store that has not
yet been rewritten.

**`crates/sase_core/src/bead/events.rs`** — add `ReferenceAdded` and `ReferenceRemoved` to `BeadEventOperationWire` and
`BeadEventPayloadWire`, each carrying one rendered reference string, with stable ordering ranks after the dependency
operations. Reduction appends on add (ignoring an entry already present, so replay is idempotent) and removes on remove
(ignoring an absent entry). Do **not** add `refs` to `BeadIssueUpdateEventFieldsWire`: whole-list replacement through
`update` would let two concurrent agents clobber each other's attachment, which is the same reason dependencies got
their own events.

**`crates/sase_core/src/bead/mutation.rs`** — a create request gains an optional reference list; creating a bead with
references emits the `IssueCreated` event followed by one `ReferenceAdded` per normalized entry in the same batch, so
there is exactly one representation of list state in the event stream. Add `add_bead_references` and
`remove_bead_references` mutations that normalize through the codec, reject a malformed entry with the grammar's error
message, and report which entries actually changed so the caller can skip a no-op commit.

**`crates/sase_core/src/bead/cli.rs`**:

- `sase bead ref` with `add`, `list`, and `rm`, following the `dep` group exactly, including bare-invocation delegation
  to `list`. `add` and `rm` take a bead id followed by one or more references. `list` takes an optional bead id and
  supports `-j, --json`; `-r, --resolve` is accepted here but resolution is performed by the Python caller (see the
  `beads` phase), because the Rust CLI has no reference context.
- `sase bead create` gains `-R, --ref REF`, repeatable. `-r` is already `--tier`, so `-R` is the available short alias.
- `sase bead show` renders a `REFS` section listing each stored reference, after `PLAN`. In this phase it prints the
  reference alone; the `beads` phase adds resolution.
- Follow `sase/memory/cli_rules.md`: alphabetically sorted subcommands and options, a short alias for every public long
  option, help output that scans cleanly.

**`crates/sase_core/src/bead/search.rs`** — add `refs` (joined) to the indexed field list beside `design`, so
`sase bead search research:202607` finds the beads that cite a report.

**`crates/sase_core/src/bead/read.rs`** — add `reference_diagnostics(issues, context)` beside
`design_reference_diagnostics`, taking an optional `ArtifactRefContextWire` through the same tri-state
`NotRequested / Unavailable / Available` shape. Resolve every bead's list in **one** batched call, not per bead, and
report three grouped, counted messages rather than one line per bead:

- `WARNING: artifact references with unknown kinds (N): <bead> [<ref>], ...` for `unknown_kind`, `unknown_repo`, and
  `unknown_project` — these are namespace errors, and a mistyped kind is never legitimate.
- `WARNING: unresolvable artifact references (N): ...` for `missing`.
- `WARNING: ambiguous artifact references (N): ...` for `ambiguous`.

`vcs_backed`, `exact`, and `drifted` are all healthy.
`NOTE: bead artifact reference validation skipped: reference context unavailable` is the `Unavailable` message, matching
the existing design-reference wording. Reporting stays strictly read-only and must never fail the health check when the
context cannot be built.

Tests: SQLite migration from a pre-`refs` database; JSONL round-trips with and without references, asserting a row
without references serializes byte-identically to today; event replay for add, remove, duplicate add, and remove of an
absent entry; the golden CLI contract tests in `cli.rs` for every new `ref` verb, for `create --ref`, and for `show`;
search matching a reference substring; and reference diagnostics over a fixture store covering each grouped message plus
the skipped-context branch.

## 5. Python bead refs, show, and doctor

Depends on the `core-beads` release. Start by raising the `sase-core-rs` pin in `pyproject.toml` to that release and
running `just install`; `sase core health` must pass before anything else. This phase owns the epic's single pin bump.

**New `src/sase/artifact_ref_lists.py`** — the one Python entry point for stored reference lists, exported from
`src/sase/artifact_refs.py` so callers keep using the stable facade:

- `ARTIFACT_REF_LIST_RESOLUTION_WIRE_SCHEMA_VERSION` with the same runtime equality assertion
  `sase.sdd.plan_refs.resolve_plan_reference_from_roots` makes, so a stale binding fails fast and visibly.
- `normalize_artifact_ref_list(entries) -> tuple[str, ...]` raising `ValueError` with the grammar's message.
- `ArtifactRefListEntry` (rendered reference plus its `ArtifactRefResolution`) and
  `resolve_artifact_ref_list(entries, *, context) -> tuple[ArtifactRefListEntry, ...]`.
- `artifact_ref_list_display_lines(entries)` returning the shared "reference, then where it resolves, or an explicit
  unresolved marker" rendering, so `sase bead show`, `sase bead ref list`, and `sase changespec ref list` cannot drift
  apart. Reuse the existing unresolved wording from `plans_detail.PLAN_REFERENCE_MISSING_LABEL` rather than inventing a
  second phrase.

**Bead model mirrors** — `Issue` in `src/sase/bead/model.py` gains `refs: list[str] = field(default_factory=list)`;
`src/sase/bead/jsonl.py` reads it through a tolerant helper and emits it only when non-empty, in the same key position
Rust serializes it, because the two exporters are compared byte-for-byte; `src/sase/bead/db.py` reads and writes the new
column; `issue_to_wire_dict` in `src/sase/bead/cli_detail.py` gains `refs`.

**`sase bead show`** — `render_issue_detail` prints a `REFS` section after `PLAN`, one line per reference showing the
stable reference and where it currently resolves, and saying so plainly when it resolves nowhere. Build the context once
per invocation and resolve the whole list in one batched call. `show` and `list` are excluded from the fast path in
`src/sase/main/bead_fast_path.py`, so this is the Python renderer and the Rust renderer from the previous phase must
stay in step — update the golden tests on both sides together, as `sase-9z` required for `display_design_path`.

**`sase bead ref list --resolve`** — the Python slow path performs resolution and prints the same shared rendering. Add
`ref` to the mutating-verb classification in `src/sase/main/bead_fast_path.py`: `_MUTATING_VERBS` covers bare verbs and
`dep` is special-cased by its action, so `ref add` and `ref rm` need the same treatment or they bypass
`assert_bead_store_write_sandboxed`. Getting this wrong lets a read-only context write to a bead store, so cover it with
a test that asserts the guard fires.

**`sase bead doctor`** — thread an optional reference context into `sase.core.bead_read_facade.doctor` and the
`py_bead_doctor` binding beside the existing plan roots, with `src/sase/bead/project.py` and
`src/sase/bead/cli_admin.py` supplying it from `sase.artifact_ref_context.artifact_ref_context`. Building the context
must be best-effort: any failure degrades to the `Unavailable` note, never to a traceback. Update the `sase bead doctor`
help text, which currently reads "Validate bead-store health and design references".

Tests: model, JSONL, and SQLite round-trips including the byte-identity assertion for a reference-free bead;
`sase bead show` against a bead with a resolving reference, an unresolvable one, and none; `ref add`/`rm`/`list`
including `--resolve` and `--json`; the write-sandbox guard on `ref add`; and doctor output for each grouped message and
for the unavailable-context branch.

## 6. The ChangeSpec REFS section in Python, CLI, and ACE

**Parsing and models** — `ChangeSpec` in `src/sase/ace/changespec/models.py` gains `refs: list[str] | None`;
`_ParserState` in `src/sase/ace/changespec/parser.py` gains `in_refs` and a `refs` accumulator wired through
`reset_section_flags`, `_parse_section_header`, `_parse_section_content`, and `build_changespec`;
`changespec_wire_from_dict` in `src/sase/core/wire_conversion.py` reads `refs`. The Python and Rust parsers must agree
exactly — the Rust parser is the production path through `parse_project_bytes`, and the Python one is the fallback and
the parity reference.

**Section-order consolidation.** Add `"REFS:"` to `CHANGESPEC_SECTION_ORDER` between `"STATUS:"` and `"COMMITS:"`, then
**delete the four hand-maintained copies** identified in §1 and have each module import the canonical constant.
`_CHANGESPEC_FIELD_HEADERS` in `src/sase/ace/hooks/formatting.py` and `_FIELD_HEADERS` in
`src/sase/status_state_machine/field_updates.py` become aliases of it; the inline tuples in
`src/sase/workflows/commit_utils/modifiers.py`, `src/sase/workflows/accept/renumber.py`, and
`src/sase/workflows/rewind/renumber.py` become slices of it. Where a copy deliberately excludes a header, express that
as an explicit exclusion from the canonical constant, not as a re-listing. Adding DELTAS to these tables is a behavior
change: it stops those scanners from absorbing a DELTAS section into whatever precedes it, which is the correct
behavior, but assert it with a test per call site rather than assuming it. If any call site turns out to depend on the
omission, say so in the phase's completion message rather than silently keeping a divergent copy.

**Persistence** — new `src/sase/ace/changespec/refs_format.py` and an `apply_refs_update` helper modeled directly on
`src/sase/ace/deltas/persistence.py`: replace an existing REFS section, insert one in canonical position when absent,
remove it when the list becomes empty, all under `changespec_lock` with `write_changespec_atomic`. Normalize through
`sase.artifact_ref_lists` before writing.

**CLI** — add `sase changespec ref` with `add`, `list`, and `rm`, bare-invoking to `list` through
`_default_list_subcommands()`. Every verb takes `-c, --changespec NAME`, defaulting to the ChangeSpec for the current
VCS checkout the way `sase changespec current` resolves it; `add` and `rm` take one or more references as positionals;
`list` supports `-j, --json` and `-r, --resolve`. An optional positional name before a variadic reference list is
ambiguous in argparse, which is why the target is an option. Follow `sase/memory/cli_rules.md`.

**ACE display** — render a `REFS:` section in `src/sase/ace/display.py`, in the TUI via a new `refs_builder` following
`deltas_builder`, in `src/sase/ace/tui/widgets/changespec_detail.py`, in the clipboard helper at
`src/sase/ace/tui/actions/clipboard/_helpers.py`, and in `src/sase/main/search_handler.py`. **Render the stored
reference verbatim; do not resolve anywhere on a TUI path.** REFS is a short, flat list, so it needs no fold level of
its own — but if you add one, register it beside the DELTAS fold in `src/sase/ace/tui/app.py` and document it in the `?`
help modal, which `src/sase/ace/CLAUDE.md` requires for any `sase ace` behavior change.

**Syntax highlighting** — add `syn match saseGpFieldLabel /^REFS:/` and a value match for reference lines in
`syntax/sase_gp.vim` in the `sase-nvim` linked repo. Note that `src/sase/ace/CLAUDE.md` still points at
`home/dot_config/nvim/syntax/saseproject.vim` in the chezmoi repo; that file no longer exists there and the ChangeSpec
syntax now lives in `sase-nvim`. Open the repo with `sase repo open sase-nvim` and use the printed path. Do **not** edit
`src/sase/ace/CLAUDE.md` or any other memory file to correct the stale pointer — report it in the phase's completion
message instead, because memory edits need explicit user permission.

**Validation** — add `src/sase/doctor/checks_changespec_refs.py` registering a `project.changespec_refs` check that
parses every ChangeSpec in the current project's store, batch-resolves all their references in one call, and reports the
same three grouped categories the bead doctor uses. Skip cleanly when no project store or no reference context is
available. Follow the `DiagnosticCheck` shape in `src/sase/doctor/checks_beads.py`, including `next_steps`.

Tests: parser round-trips against the Rust parser for a spec with, without, and with a malformed REFS section; the
persistence helper for replace, insert, remove, and a spec that ends at two blank lines; one test per consolidated
section-boundary call site; `sase changespec ref` verbs including the current-checkout default; ACE CLI and TUI
rendering; the searchable-text mirror; and the doctor check across healthy, unresolvable, and skipped states.

## 7. Published pages, ACE Plans tab, mobile bridge, and declaration

**Bead pages** — `src/sase/bead_pages/rendering_identity.py` renders a `**Plan:**` fact through `PlanLinkResolver`. Add
a `## References` section listing each stored reference, hosted-linked where the existing resolver can produce a URL and
plain-escaped otherwise, and keep it inside the same injection-safe rendering the prose sections use. Published pages
are public artifacts of this repo, so a reference that resolves nowhere renders as plain text, never as a broken link.

**ACE Plans tab** — `src/sase/ace/tui/widgets/artifacts/plans_detail.py` renders `Plan reference` and `Resolved plan`
rows for `design`. Add a `References` row listing the stored references for the selected epic or phase. Resolution, if
added at all, belongs in the `PlansSnapshot` built by `plans_data.py`, never in the detail renderer; rendering the
reference alone is an acceptable and preferred v1. `plans_filtering.py` includes `issue.design` in its filter corpus —
decide whether references join it, test the choice, and state it in the completion message.

**Mobile bridge** — `src/sase/integrations/_mobile_helper_beads.py` returns bead fields to the mobile client. Return the
stored references, resolved where resolution succeeds and stored-form otherwise, matching the treatment `design`
received in `sase-9z`.

**Declaration** — `sase artifact create` mints a `file:` reference that today the agent must copy by hand into prose.
Add `-b, --bead [ID]` to `src/sase/artifact_cli/create.py` and `src/sase/main/parser_artifact.py`: given bare, attach
the new reference to the bead named by `SASE_BEAD_ID` (`src/sase/bead/work.py::SASE_BEAD_ID_ENV`); given an id, attach
to that bead. `-b` is unused on this subcommand. This is opt-in on purpose — a bead write auto-commits the bead store,
and `sase artifact create` must not start committing to a sidecar without being asked. Fail loudly with a clear message
when the flag is passed bare and no bead id is in the environment, and when the named bead does not exist; never
silently create the artifact without the attachment the caller asked for.

Tests: page rendering with zero, one, and unresolvable references, including the escaping assertion; the Plans-detail
row; the mobile helper's resolved and unresolved shapes; and `sase artifact create --bead` for the bare, explicit,
missing-environment, and unknown-bead cases.

## 8. Documentation, skills, and the live-store audit

- `docs/change_spec.md` — add `REFS` to the canonical order block in "Format Overview" and a field specification section
  describing the entry format, the accepted kinds, and that entries are stored canonically and validated by
  `sase doctor`.
- `docs/beads.md` — document the `refs` field, the `sase bead ref` group, `sase bead create --ref`, and what
  `sase bead doctor` now reports. Keep `design` and `refs` clearly distinguished: one plan, many references.
- Skill sources under `src/sase/xprompts/skills/` — `sase_beads.md` gains the `ref` group and the new doctor output;
  `sase_changespecs.md` gains the REFS section; `sase_artifact_file.md` gains `sase artifact create --bead` and a
  pointer to where a minted reference should be persisted. Read the `generated_skills` long-term memory first, edit the
  templates and never a deployed copy, and follow its commit-then-deploy sequence.
- Audit both live stores with the new validators and include the output in the completion message: `sase bead doctor`
  over the `sase` store (2,389 beads, which today carry no references, so the expected result is a clean reference
  report and unchanged design-reference findings) and `sase doctor` for `project.changespec_refs` over the 11 project
  files. Confirm that `issues.jsonl` has not churned: a store with no references must still export byte-identically to
  its pre-epic form.

## 9. Explicitly out of scope

- **Migrating the `| CHAT:`, `| DIFF:`, and `| PLAN:` sub-lines** on COMMITS entries to canonical references. The
  pathology is real and measured in §1, but `DIFF:` has no kind in the grammar, and converting per-commit pointers is a
  ChangeSpec-history migration with its own renumber and rewind interactions. This epic gives ChangeSpecs a reference
  field; converting the existing path pointers is the natural follow-up and this plan deliberately leaves the field
  empty for them.
- **Scraping references out of prose.** 16 beads mention reference-shaped tokens in `description` or `notes`. Guessing
  intent from prose is exactly the silent rewrite `sase-9z` refused for legacy design paths, and there is no repair
  index here comparable to plan `bead_id:` frontmatter.
- **Repairing the 46 remaining legacy `design` values.** They belong to `sase-9z`'s residue, not to this epic, and this
  plan does not touch `design`.
- **Roles or labels on a reference entry.** Deferred by design (§2); the storage format keeps the extension additive.
- **Any change to the reference grammar**, to the seven kinds, or to fragment anchors. The research report is explicit
  that the grammar should be spent on consumers, not widened, and this epic is one of those consumers.
- **Resolution on any TUI render or keystroke path**, and any new artifact-reference caching layer for the TUI.
- **Retention protection.** The research report's item 3 wants pruning to skip anything referenced by a bead, plan, or
  ChangeSpec. This epic makes that query answerable for the first time; consuming it is retention's job.

## 10. Risks

- **Two parsers drifting.** The ChangeSpec format has a Rust production parser and a Python fallback, and the bead
  detail view has a Rust and a Python renderer. Both pairs have existing parity tests
  (`crates/sase_core/tests/python_wire_parity.rs`, the `cli.rs` golden tests); every phase that touches one side updates
  the other in the same commit.
- **Byte-stability of `issues.jsonl`.** The bead doctor compares the event projection against the legacy JSONL export
  byte-for-byte. Emitting `refs` unconditionally would flag every store until it is rewritten, which is why the field is
  omitted when empty on both sides. The audit in the `docs` phase verifies this against the live 2,389-record store.
- **Section-boundary consolidation regressions.** Folding four divergent tables onto one constant changes behavior at
  each call site, because all four are missing DELTAS today. This is a fix, but it is a behavior change in the reword,
  renumber, rewind, and hook-formatting paths. Cover each call site with a test and report anything that turns out to
  depend on the omission.
- **Resolution cost.** `resolve_file` reads the whole artifact index per call. Every batched path in this plan exists to
  amortize that; a loop over the single-reference API anywhere in `doctor`, `show`, or a snapshot loader reintroduces
  the quadratic cost the codec was built to remove.
- **Cross-repo release coupling.** Two `sase-core` releases land in sequence and one pin bump consumes them, in the
  `beads` phase. `CHANGESPEC_WIRE_SCHEMA_VERSION` moves 4 to 5 on both sides at once; Python asserts equality at
  runtime, so a mismatch fails fast rather than corrupting data.
- **Depending on an epic that is still landing.** `sase-b7.5` is open and this workspace's installed core is one wire
  version behind what the tree expects. The `core-refs` phase must start from a tree where `just install` and
  `sase core health` pass; if they do not, that is `sase-b7`'s residue and must be resolved before this epic's first
  commit, not worked around.
- **A field nobody writes.** Storage without a writer is dead weight. `sase artifact create --bead` in the `surfaces`
  phase and the skill updates in `docs` are what make the field populate in normal operation; treat them as part of the
  deliverable rather than as documentation polish.

## 11. Verification

Each phase runs `just install` first, because ephemeral `sase_<N>` workspaces drift, then `just check`. In `sase-core`,
run that repo's own check target before committing. New public Python symbols that a later phase consumes get
`--epic-symbol "<this epic's bead id>(<symbol>)"` entries in the `_lint-symvision` recipe in the `Justfile`, removed by
the phase that adds the consumer; per `sase/memory/symvision.md`, do not reach for pragmas when the consumer is a later
phase of this epic.

End-to-end checks after the `surfaces` phase, against the live store:

```bash
sase bead ref add sase-b7 research:202607/artifact_capture_and_retention/artifact_capture_and_retention.md
sase bead show sase-b7                      # REFS section, reference plus resolved path
sase bead ref list sase-b7 --resolve --json # machine-readable resolution outcomes
sase bead search artifact_capture           # the reference is searchable
sase bead doctor                            # clean reference report
sase bead ref rm sase-b7 research:202607/artifact_capture_and_retention/artifact_capture_and_retention.md
sase changespec ref add research:202607/artifact_capture_and_retention/artifact_capture_and_retention.md
sase changespec current                     # REFS section renders
sase doctor                                 # project.changespec_refs passes
```
