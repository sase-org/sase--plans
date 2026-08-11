---
tier: tale
size: medium
title: Ref contract wire types in sase-core
goal:
  Land the versioned artifact-reference contract primitives in the sibling `sase-core`
  Rust repo — the `@stitch`/`@patch`/path-`@file` kinds, the permanent kind-alias
  registry, quoted-argument grammar, the closed expansion formatter, the provider
  spec/entry/use wire types, a shared numeric Markdown-link allocator, and a bottom-
  anchored `Referenced By` block — plus the Python bindings and the one coordinated
  wire-version bump this repo needs, without changing any behavior sase's current Python
  depends on.
proposed_by: bbugyi200.athena.sase-js.1
bead: sase-js.1
create_time: 2026-08-11 13:43:10
status: wip
---

- **PARENT:**
  [202608/artifact_ref_contract.md](https://github.com/sase-org/sase--plans/blob/main/202608/artifact_ref_contract.md)
- **BEAD:**
  [sase-js.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-js/sase-js.1.md)

# Plan: Ref contract wire types in sase-core

This is phase `core` of epic `sase-js` (bead `sase-js.1`), specified by §4.1 of
`plans:202608/artifact_ref_contract.md`. Read that file's §3.1–§3.8 before starting;
this plan does not restate the design, it records the verified integration constraints
and the exact API surface to build.

Almost all of the work lands in the linked `sase-core` repo. Open it with the
`/sase_repo` skill (`sase repo open sase-core -r "<reason>"`) and use only the path it
prints. The single exception is a five-file coordinated bump in this repo, described in
§7.

## 1. Verified constraints that shape this phase

These were confirmed by reading both trees at their current revisions. They are the
reason several §4.1 bullets are implemented as _opt-in_ core APIs rather than as
in-place behavior changes.

**sase CI builds `sase-core` from master.** `.github/workflows/ci.yml`'s `build-core`
job checks out `sase-org/sase-core` at HEAD and builds the wheel every source lane
consumes; `just install` does the same from the local checkout. So the moment this
phase's commit lands on `sase-core` master, every sase CI run and every agent workspace
picks it up. **Core must keep sase master green on its own.**

**Python hard-gates the wire version, with one constant for three record types.**
`src/sase/artifact_ref_models.py` declares `ARTIFACT_REF_WIRE_SCHEMA_VERSION = 4` and
`check_record_schema()` compares it for parse records (`ArtifactRef.from_wire`), scan
candidates (`ArtifactRefPromptCandidate.from_wire`), and resolution records
(`artifact_ref_operations.py:156`). `_require_artifact_ref_schema()` additionally probes
`artifact_ref_wire_schema_version()` on every parse/canonicalize/resolve/scan call and
raises. Consequences:

- Rust's `ARTIFACT_REF_PARSE_WIRE_SCHEMA_VERSION` and
  `ARTIFACT_REF_RESOLUTION_WIRE_SCHEMA_VERSION` must move **together**, and the Python
  constant must move in the same landing. There is no version-skew tolerance in either
  direction; a mismatch breaks the agent launch path outright.
- `ARTIFACT_REF_CONTEXT_WIRE_SCHEMA_VERSION` and
  `ARTIFACT_REF_LIST_RESOLUTION_WIRE_SCHEMA_VERSION` are **not** touched by this phase.

**Python allow-lists kind, payload, and status strings.** `ArtifactRef.from_wire`,
`ArtifactRefPayload.from_wire`, and `artifact_ref_lists._RESOLUTION_STATUSES` raise on
anything outside their sets. So new kinds and payload types require the coordinated
Python addition in §7, and **this phase must not invent a new resolution `status`
string** — unresolvable new kinds reuse the existing `unknown_kind`.

**Phase `retire` (`sase-js.2`) is running concurrently in both repos.** It deletes and
rewrites `src/sase/xprompts/refs/*.md`, `xprompt/loader_refs.py`,
`artifact_ref_prompt_parsing.py`, `artifact_ref_prompt_rendering.py`,
`prompt_catalog.py`, `xprompt_browser_helpers.py`, `content_layout.py`, and — in
`sase-core` — `xprompt_catalog.rs` and `crates/sase_xprompt_lsp`. **Do not touch any of
those files.** Confine sase-core edits to `artifact_ref/`, `commit_footer.rs`, the two
new top-level modules, `lib.rs`, and `crates/sase_core_py/src/lib.rs`.

**Therefore the aliases ship switched off.** §4.1 asks for `commit:` → `stitch:`
permanently and `plans:` → `plan:` with a diagnostic. Canonicalizing inside the default
`parse_artifact_ref` would change `rendered` for every persisted string and break
`artifact_ref_prompt_rendering.py` (which branches on `kind_type == "commit"` in five
places and does `rendered.removeprefix("commit:")`) — a file `retire` is editing right
now. It would also break `resolve_document`, whose document roots really are configured
with the role `plans`. So this phase lands the alias table plus an explicit
canonicalizing entry point (`parse_artifact_ref_canonical`), leaves the default parse
byte-identical, and phase `builtins` (`sase-js.4`) flips the Python callers over. Record
this handoff in the bead note; it is a deliberate sequencing decision, not an omission.

**release-plz owns `sase-core` versions.** Per `sase-core/AGENTS.md`, never hand-edit
`[workspace.package].version`, crate versions, or path-dependency pins. Use Conventional
Commits and let release-plz compute the bump.

## 2. Kinds, payloads, and aliases

In `crates/sase_core/src/artifact_ref/wire.rs`:

- Bump `ARTIFACT_REF_PARSE_WIRE_SCHEMA_VERSION` and
  `ARTIFACT_REF_RESOLUTION_WIRE_SCHEMA_VERSION` from `4` to `5`.
- `ArtifactRefKindWire` gains `Stitch` and `Patch` (serde tags `stitch`, `patch`).
- `ArtifactRefPayloadWire` gains:
  - `Stitch { repo: Option<String>, sha: String }` — tag `stitch`
  - `Patch { name: String }` — tag `patch`
  - `FilePath { path: String }` — tag `file_path`, used with kind `File`

Grammar and rendering, all of which must round-trip (`render(parse(x)) == x`):

| Authored                | Payload                               | Rendered                |
| ----------------------- | ------------------------------------- | ----------------------- |
| `stitch:<sha>`          | `Stitch { repo: None, sha }`          | `stitch:<sha>`          |
| `stitch:<repo>@<sha>`   | `Stitch { repo: Some(repo), sha }`    | `stitch:<repo>@<sha>`   |
| `patch:<name>`          | `Patch { name }`                      | `patch:<name>`          |
| `file:explicit:<hex24>` | `File { source, digest }` (unchanged) | `file:explicit:<hex24>` |
| `file:<path>`           | `FilePath { path }`                   | `file:<path>`           |

- Stitch SHAs reuse `validate_sha(sha, false)` (7–40 lowercase hex); the repo label
  reuses `validate_repo`. `parse_payload` disambiguates on the presence of `@`, using
  `rsplit_once('@')` exactly as `Commit` does today, so a payload-internal `@` keeps
  working.
- Patch names may contain spaces (that is what §3 quoting is for). Validate only that
  the name is non-empty, contains no control characters, no newline, no `\0`, and does
  not begin or end with whitespace. **Do not invent a project-qualified spelling**: a
  Patch name has no reserved separator, so `patch:<project>/<name>` would not
  round-trip. §4.1 lists the payload as `Patch { project, name }`; project qualification
  instead lives on the resolved entry (`ArtifactEntryWire.project_display_name`, §5) and
  in the `candidates` list of an ambiguous resolution, which is all §4.4 actually needs.
  Note this deviation in the bead note.
- `FilePath` accepts the authored spelling verbatim, including a leading `~/` or `/`.
  Validate only: non-empty, no `\0`, no backslash, no trailing slash. Containment, `~`
  expansion, glob filtering, and traversal rejection belong to phase `files`
  (`sase-js.5`) — do not implement them here, and do not let this payload reach the
  filesystem.
- `file:` disambiguation is total and unchanged from the design: a payload whose first
  colon-separated segment is exactly `explicit` or `default` **and** which has exactly
  one further colon is a digest payload; everything else is a path payload. Today
  `file:notes/x.md` is a parse error, so this is purely additive.
- `kind_rejects_fragments` gains `Stitch` and `Patch` (entity kinds, like `Commit`).
  `FilePath` keeps fragments, as `File` does today.
- `classify_kind` gains `"stitch" => Stitch` and `"patch" => Patch`. Both currently fall
  through to `Document { role }`, and no configured document root uses either name, so
  nothing observable changes.
- `resolve_artifact_ref` gains arms for `Stitch`, `Patch`, and `FilePath` that return
  `resolution("unknown_kind", rendered)` with a `diagnostic` naming the kind and saying
  resolution arrives with the provider registry. **Do not add a new status string.**
  Real resolution is phase `builtins`/`files` work.

New module `crates/sase_core/src/artifact_ref/kinds.rs` — the registry §4.1 calls
"descriptor()", and the home of the aliases:

```rust
pub const ARTIFACT_REF_KIND_CATALOG_WIRE_SCHEMA_VERSION: u64 = 1;

pub enum ArtifactRefKindStatusWire { Live, Alias, Deprecated, Historical }

pub struct ArtifactRefKindDescriptorWire {
    pub schema_version: u64,
    pub kind: String,              // canonical label
    pub display_name: String,
    pub status: ArtifactRefKindStatusWire,
    pub aliases: Vec<String>,      // permanent parse aliases
    pub reserved: bool,            // providers may not claim it
    pub argument_summary: String,  // one line, for completion and docs
    pub offered_in_completion: bool,
    pub accepts_fragment: bool,
}

pub struct ArtifactRefKindAliasWire {
    pub schema_version: u64,
    pub requested: String,
    pub canonical: String,
    pub alias: bool,
    pub status: ArtifactRefKindStatusWire,
    pub diagnostic: Option<String>,
}

pub struct CanonicalArtifactRefWire {
    pub schema_version: u64,
    pub reference: ParsedArtifactRefWire,
    pub alias: Option<String>,        // the label the author actually wrote
    pub diagnostic: Option<String>,
}

pub fn artifact_ref_kind_catalog() -> Vec<ArtifactRefKindDescriptorWire>;
pub fn canonical_artifact_ref_kind(label: &str) -> ArtifactRefKindAliasWire;
pub fn parse_artifact_ref_canonical(value: &str)
    -> Result<CanonicalArtifactRefWire, ArtifactRefError>;
```

Registry contents:

- Reserved, live, offered in completion: `stitch`, `patch`, `bead`, `agent`, `file`.
- `commit` — permanent alias of `stitch`, status `Alias`, no diagnostic, absent from
  completion. It must parse forever; persisted `commit:` strings live in bead `refs:`
  lists, `artifact_consumption.jsonl`, prompt manifests, and Patch files.
- `plans` — deprecated alias of `plan`, status `Deprecated`, with an actionable
  diagnostic (`"@plans: is deprecated; write @plan:<path>"`), absent from completion.
- `chat`, `bug` — status `Historical`: they still parse and render for archives, and are
  absent from completion and from new-prompt validation.
- Any other label is an unregistered document kind: canonical to itself, status `Live`,
  `reserved: false`.

`parse_artifact_ref_canonical` rewrites only the kind label, then delegates to
`parse_artifact_ref`, and carries the alias plus diagnostic. It is the _only_ place a
`commit:`/`plans:` string becomes `stitch:`/`plan:`; `parse_artifact_ref` itself is
unchanged.

## 3. Quoted-argument grammar

In `crates/sase_core/src/artifact_ref/scanner.rs`:

- After `@<kind>:`, if the next byte is `"`, the argument runs to the next unescaped
  `"`. Support `\"` and `\\` escapes and nothing else; a backslash before any other
  character is a literal backslash.
- An unterminated opening quote produces one candidate that ends at the end of the
  current line (never at EOF) with `well_formed: false`. Editors get a usable diagnostic
  instead of a candidate that swallows the rest of the document.
- `text` stays the raw source slice including the quotes. `reference` is the **unquoted,
  unescaped** `kind:argument`, so `parse_artifact_ref(reference)` keeps working
  unchanged.
- `payload_span` covers the argument _inside_ the quotes; `candidate_span` covers the
  whole `@kind:"…"` including them.
- A trailing `#fragment` splits after the closing quote for a quoted argument
  (`@plan:"a b.md"#L3`) and exactly as today for an unquoted one. Trailing-punctuation
  trimming (`trim_candidate_end`) applies only to unquoted arguments — a quoted argument
  ends at its quote, full stop.
- `ArtifactRefPromptCandidateWire` gains `quoted: bool`. Python reads named keys only,
  so an added field is inert there until phase `builtins` wants it.
- Add `pub fn quote_artifact_ref_argument(argument: &str) -> String`, returning the
  quoted-and-escaped spelling when the argument contains whitespace, `"`, `'`, a
  backtick, or a character `trim_candidate_end` would strip from the end; otherwise the
  bare argument. Completion inserts through this.
- Leave the scanner's left-context rule and its zone-agnostic behavior alone. Literal-
  zone protection lives in `rewrite_prompt_artifact_links`/`prompt_literal_zone_ranges`,
  not in the candidate scanner, and changing that would alter `artifact_ref_scan_prompt`
  results for the LSP and completion surfaces that `retire` is mid-edit on.

## 4. The closed expansion formatter

New module `crates/sase_core/src/artifact_ref/expansion.rs`:

```rust
pub const ARTIFACT_REF_EXPANSION_PLACEHOLDERS: &[&str] = &[
    "kind", "argument", "canonical_argument", "display_label",
    "project", "repository", "repo_relative_path", "checkout_path",
    "sidecar_role", "captured_revision", "captured_digest", "logical_path",
];

pub fn validate_artifact_ref_expansion_format(format: &str)
    -> Result<Vec<String>, ArtifactRefError>;   // returns the placeholders used
pub fn render_artifact_ref_expansion(
    format: &str,
    values: &BTreeMap<String, String>,
) -> Result<String, ArtifactRefError>;
```

- `{{` and `}}` escape literal braces. An unknown placeholder, an unbalanced brace, or a
  missing value is a validation error naming the offender.
- Substituted values are inserted verbatim and **never rescanned**: a value containing
  `{kind}` stays literal text. Test this explicitly — it is the injection refusal §4.1
  asks for.
- No filesystem, process, or environment access, by construction: the function takes a
  format and a map and returns a string.

## 5. Provider spec, entry, resolution, and use wire types

New module `crates/sase_core/src/artifact_ref/provider_spec.rs`, modelling §3.4
verbatim, with `schema_version`, `provider`, and a `ref:`-keyed descriptor
(`#[serde(rename = "ref")]`). Every map is a `BTreeMap` so serialization is
deterministic.

```rust
pub const ARTIFACT_REF_PROVIDER_SPEC_WIRE_SCHEMA_VERSION: u64 = 1;

pub fn validate_artifact_ref_provider_spec(
    spec: &ArtifactRefProviderSpecWire,
) -> Result<(), ArtifactRefError>;
pub fn artifact_ref_provider_spec_digest(
    spec: &ArtifactRefProviderSpecWire,
) -> Result<String, ArtifactRefError>;   // sha256 hex over the normalized spec
```

Validation rejects: an unsupported `schema_version`; a provider id or ref kind that is
not `[a-z][a-z0-9_-]*`; a reserved kind (`stitch`, `patch`, `bead`, `agent`, `file`)
claimed by a document provider; an expansion format that fails
`validate_artifact_ref_expansion_format`; a property type outside `string`, `enum`,
`boolean`, `integer`, `number`, `date`, `datetime`, `string_list`; an `enum` field with
no values; a `detail.fields` entry or `identity.property` that names no declared
property; inventory globs that fail `ArtifactPathFilter::compile`; a `publication.link`
outside `vcs_permalink | agents_object | none`; a `publication.referenced_by` outside
`markdown_table | none`; and a `properties.source` outside
`markdown_frontmatter | provider`.

**Spec merge is out of scope.** §3.5's `use:`-plus-inline merge is config merge, which
the Rust core backend boundary explicitly leaves in Python; it is phase `registry`
(`sase-js.3`) work.

New module `crates/sase_core/src/artifact_ref/entry.rs` — §3.3's normalized results:

```rust
pub const ARTIFACT_REF_ENTRY_WIRE_SCHEMA_VERSION: u64 = 1;

pub struct ArtifactEntryWire {
    pub schema_version: u64,
    pub stable_id: String,
    pub ref_kind: String,
    pub canonical_argument: String,
    pub display_label: String,
    pub project_display_name: Option<String>,
    pub repository: Option<String>,
    pub repo_relative_path: Option<String>,
    pub captured_revision: Option<String>,   // full 40-hex when present
    pub captured_digest: Option<String>,     // full 64-hex sha256 when present
    pub logical_path: Option<String>,
    pub properties: BTreeMap<String, String>,
    pub origin: ArtifactEntryOriginWire,     // prompt_ref | agent_artifact | both
}

pub struct ResolvedArtifactRefWire {
    pub schema_version: u64,
    pub raw_ref: String,
    pub canonical_ref: String,
    pub occurrence_span: ArtifactRefSpanWire,
    pub entry: Option<ArtifactEntryWire>,
    pub prompt_text: String,
    pub publication_target: Option<String>,
    pub captured_file: Option<String>,
    pub diagnostics: Vec<String>,
}

pub fn validate_artifact_entry(entry: &ArtifactEntryWire)
    -> Result<(), ArtifactRefError>;
```

`validate_artifact_entry` enforces the split §3.3 depends on: `stable_id` non-empty,
`captured_revision` exactly 40 lowercase hex when set, `captured_digest` exactly 64
lowercase hex when set, `ref_kind` a valid kind label, and property keys matching
`[a-z][a-z0-9_]*`.

New module `crates/sase_core/src/artifact_ref/uses.rs` — §3.7's immutable per-occurrence
record and its JSONL manifest, modelled directly on `prompt_artifact.rs`'s tolerant
parser (skip and report malformed or future-versioned lines rather than failing the
whole file):

```rust
pub const ARTIFACT_REF_USE_WIRE_SCHEMA_VERSION: u64 = 1;

pub struct ArtifactRefUseRecordWire { /* exactly the §3.7 JSON fields */ }

pub fn parse_artifact_ref_use_manifest(bytes: &[u8]) -> Vec<ArtifactRefUseRecordWire>;
pub fn render_artifact_ref_use_record(record: &ArtifactRefUseRecordWire)
    -> Result<String, serde_json::Error>;
pub fn validate_artifact_ref_use_record(record: &ArtifactRefUseRecordWire)
    -> Result<(), ArtifactRefError>;
```

## 6. Shared numeric link allocator and the `Referenced By` block

New top-level module `crates/sase_core/src/markdown_link_refs.rs`. It is not
artifact-ref-specific — commit footers use it too — so it sits beside
`commit_footer.rs`, not under `artifact_ref/`.

- Move `parse_reference_definition`, `allocate_numeric_id`, and `assign_reference_id`
  out of `commit_footer.rs` and have that module import them. Behavior must be
  byte-identical; `commit_footer.rs`'s existing tests are the proof and must pass
  untouched.
- Add the scan §2 of the epic plan says is missing today:

```rust
pub const MARKDOWN_LINK_REFS_WIRE_SCHEMA_VERSION: u64 = 1;

pub struct MarkdownReferenceDefinitionWire { pub label: String, pub destination: String }
pub struct MarkdownReferenceScanWire {
    pub schema_version: u64,
    pub definitions: Vec<MarkdownReferenceDefinitionWire>,  // first-definition-wins order
    pub used_labels: Vec<String>,                           // numeric uses, deduped, sorted
}

pub fn scan_markdown_reference_links(document: &str) -> MarkdownReferenceScanWire;
pub fn allocate_markdown_reference_label(
    scan: &MarkdownReferenceScanWire,
    destination: &str,
    assigned: &BTreeMap<String, String>,   // label -> destination assigned this run
) -> String;
pub fn append_markdown_reference_definitions(
    document: &str,
    definitions: &[MarkdownReferenceDefinitionWire],
) -> String;
```

- The scan collects reference _definitions_ (`[label]: dest`) **and** numeric _uses_ —
  full (`[text][7]`), collapsed (`[7][]`), and shortcut (`[7]`) — skipping ranges
  returned by `crate::prompt_literal_zone_ranges` so fenced blocks and inline code never
  reserve a label. Footnote labels (`[^1]`) live in their own namespace and are ignored.
- Honor CommonMark's first-definition-wins rule: a second `[7]: …` is recorded as a
  duplicate to be preserved verbatim, never re-emitted, and never treated as the live
  destination.
- `allocate_markdown_reference_label` reuses an existing label whose destination already
  matches, otherwise returns the **lowest positive integer** not used by any definition,
  any numeric use, or any label assigned this run.
- `append_markdown_reference_definitions` appends missing definitions in numeric order
  at the bottom, preserves the document's trailing-newline shape, emits nothing for a
  definition that already exists with the same destination, and is byte-identical on a
  second run.

New top-level module `crates/sase_core/src/referenced_by.rs`, mirroring the proven
properties of `plan/artifact_link.rs`'s header block but anchored at the **bottom**:

```rust
pub const REFERENCED_BY_BLOCK_WIRE_SCHEMA_VERSION: u64 = 1;
pub const MAX_RENDERED_REFERENCED_BY_ROWS: usize = 50;

pub struct ReferencedByColumnWire { pub key: String, pub label: String, pub numeric: bool }
pub struct ReferencedByRowWire {
    pub values: BTreeMap<String, String>,
    pub link_targets: BTreeMap<String, String>,   // key -> destination, rendered as [value][N]
}
pub struct ReferencedByTableWire {
    pub schema_version: u64,
    pub columns: Vec<ReferencedByColumnWire>,
    pub rows: Vec<ReferencedByRowWire>,
    pub omitted: usize,
}
pub struct ReferencedByDocumentWire {
    pub schema_version: u64,
    pub table: Option<ReferencedByTableWire>,
    pub body: String,          // the document with the managed block removed
    pub reason: Option<String>,
}

pub fn render_referenced_by_block(table: &ReferencedByTableWire)
    -> Result<String, ArtifactRefError>;
pub fn parse_referenced_by_block(document: &str) -> ReferencedByDocumentWire;
pub fn upsert_referenced_by_block(document: &str, table: &ReferencedByTableWire)
    -> Result<String, ArtifactRefError>;
pub fn remove_referenced_by_block(document: &str) -> String;
pub fn strip_referenced_by_block(document: &str) -> String;
```

- Markers are exactly `<!-- sase:referenced-by:start -->` and
  `<!-- sase:referenced-by:end -->`, wrapping a blank line, a `## Referenced By`
  heading, and a GFM table with an alignment row (right-aligned for `numeric: true`).
- Rows sort deterministically by their column values in declared column order; the
  render caps at `MAX_RENDERED_REFERENCED_BY_ROWS` and emits a trailing `_… and N more_`
  line when `omitted > 0`.
- Cell text is escaped for `|` and newlines. A cell with a `link_targets` entry renders
  as a numbered reference-style link and returns the labels the caller must define; do
  the numbering through `markdown_link_refs`, not with a second allocator.
- `upsert_referenced_by_block` places the block at the very bottom separated by exactly
  one blank line, preserves the document's trailing-newline shape, removes the block
  entirely when the table has no rows, and is byte-identical on a second run.
- Marker recovery, and this is the tested contract: a duplicate block collapses to one;
  a start marker with no end marker is treated as extending to EOF and is replaced; a
  stray end marker with no start is left alone and reported through `reason`.
- `strip_referenced_by_block` is the §3.8 invariant made callable — content digests and
  change detection run on its output, so a citation never counts as a new version of the
  cited artifact.

Register both modules in `crates/sase_core/src/lib.rs` and re-export their public items
alongside the existing `pub use` blocks.

## 7. Bindings and the coordinated bump

In `crates/sase_core_py/src/lib.rs`, add bindings and extend the module doc-comment
inventory at the top of the file (it is a maintained list, not decoration):

```text
artifact_ref_kind_catalog()                      -> list[dict]
artifact_ref_kind_canonicalize(label)            -> dict
artifact_ref_parse_canonical(value)              -> dict
artifact_ref_quote_argument(argument)            -> str
artifact_ref_expansion_placeholders()            -> list[str]
artifact_ref_expansion_validate(format)          -> list[str]
artifact_ref_expansion_render(format, values)    -> str
artifact_ref_provider_spec_validate(spec)        -> None
artifact_ref_provider_spec_digest(spec)          -> str
artifact_ref_provider_spec_wire_schema_version() -> int
artifact_ref_entry_validate(entry)               -> None
artifact_ref_entry_wire_schema_version()         -> int
artifact_ref_use_manifest_parse(bytes)           -> list[dict]
artifact_ref_use_record_render(record)           -> str
artifact_ref_use_wire_schema_version()           -> int
markdown_link_refs_wire_schema_version()         -> int
markdown_reference_links_scan(document)          -> dict
markdown_reference_label_allocate(scan, destination, assigned) -> str
markdown_reference_definitions_append(document, definitions)   -> str
referenced_by_wire_schema_version()              -> int
referenced_by_block_parse(document)              -> dict
referenced_by_block_render(table)                -> str
referenced_by_block_upsert(document, table)      -> str
referenced_by_block_remove(document)             -> str
referenced_by_block_strip(document)              -> str
```

Follow the surrounding conventions: `#[pyo3(name = "…")]`, dict-in/dict-out through the
existing serde helpers, `py.allow_threads` for anything that could be slow, and errors
mapped through the module's existing `ArtifactRefError` converter.

Then the coordinated bump in **this** repo — five small edits, none of them in phase
`retire`'s blast radius:

1. `src/sase/artifact_ref_models.py` — `ARTIFACT_REF_WIRE_SCHEMA_VERSION = 5`; add
   `"stitch"` and `"patch"` to `ArtifactRefKindType` and to the kind allow-list in
   `ArtifactRef.from_wire`; add `"stitch"`, `"patch"`, and `"file_path"` to the payload
   allow-list in `ArtifactRefPayload.from_wire`. The dataclass already carries `repo`,
   `sha`, `name`, and `path`, so no new fields are needed.
2. `tools/validate_sase_core_rs` — `"artifact_ref_wire_schema_version": 5`.
3. `tests/test_validate_sase_core_rs_tool.py` — the same expectation.
4. `tests/artifact_refs/test_parsing.py:54` — `== 5`.
5. `tests/artifact_refs/test_lists.py:55` — the fixture's `"schema_version": 4` → `5`.

Deliberately **not** touched: `BUILTIN_ARTIFACT_REF_KINDS`, completion catalogs, the ref
xprompt surface, and `REQUIRED_BINDINGS` in `tools/validate_sase_core_rs`. Adding the
new binding names to `REQUIRED_BINDINGS` would raise the published-floor requirement
before any sase code calls them; the phases that wire them up add their own names.

## 8. Tests

Rust, in the module under test, matching §4.1's list:

- **Parsing** — quoted, escaped, fragment-after-quote, unterminated-quote, and
  trailing-punctuation cases; `quote_artifact_ref_argument` round-trips through the
  scanner for arguments with spaces, quotes, and trailing punctuation.
- **Kinds** — `stitch:<sha>` versus `stitch:<repo>@<sha>` disambiguation; short-hash
  length bounds; `patch:` names with spaces; both `file:` payload shapes; render/parse
  round-trips for every new form; fragments rejected on `stitch`/`patch` and accepted on
  a file path.
- **Aliases** — `commit:` canonicalizes to `stitch:` with no diagnostic and the original
  label preserved; `plans:` canonicalizes to `plan:` with a diagnostic; `chat:`/`bug:`
  still parse and render unchanged; `parse_artifact_ref` (the default) is byte-identical
  to today for all eight historical kinds. Capture that last one as an explicit golden
  test — it is the regression net for the whole backward-compatibility argument.
- **Formatter** — placeholder validation, brace escaping, unknown-placeholder and
  missing-value errors, and the injection refusal (a substituted value containing
  `{kind}` is not re-expanded).
- **Provider spec** — each validation rejection above; a valid spec round-trips through
  serde; the digest is stable across serialization and changes when any field changes.
- **Entry/use wires** — `validate_artifact_entry` rejections; manifest round-trip that
  skips malformed and future-versioned lines and tolerates unknown fields.
- **Link allocator** — allocation with gaps, pre-existing definitions, same-destination
  reuse, numeric uses with no definition (the dangling `[x][3]` collision §6 of the epic
  plan calls out), labels inside fenced code and inline code, footnote labels, duplicate
  definitions, and second-run idempotence.
- **`Referenced By`** — render/parse round-trip; idempotent repeat upsert; the 50-row
  cap with the omitted line; empty table removes the block; duplicate-block collapse;
  unterminated start marker; stray end marker; `strip_referenced_by_block` leaves a
  document with no block untouched.

Python-side: no new tests. The five edits in §7 are version-constant mechanics and are
covered by the tests they update.

## 9. Verification

In the `sase-core` checkout (its CI runs exactly these three):

```bash
cargo fmt --all
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
```

Then in this repo, after the core edits are in place:

```bash
just install     # rebuilds sase_core_rs from the linked checkout
just check-full
```

`just check-full`, not `just check`: a wire-version bump plus a rebuilt extension is a
whole-repo change, well outside what the diff-scoped lane's import-graph closure can
select.

## 10. Landing and handoff

Two commits, in this order.

1. **`sase-core` first.** Commit through the `sase_git_commit` skill from the linked
   checkout. Conventional Commits with a breaking marker, because the wire version moves
   and public enums gain variants:
   `feat(artifact-ref)!: add ref contract wire types, quoted arguments, link allocator, and Referenced By block`
   plus a `BREAKING CHANGE:` footer naming the parse/resolution wire bump from 4 to 5.
   Do not edit any version in `Cargo.toml`; release-plz computes `0.24.6 -> 0.25.0` from
   the marker.
2. **This repo second**, with the five §7 edits, referencing bead `sase-js.1`.

Release and rollout notes to record in the bead note for the land agent:

- "Release the binding" completes when release-plz's release PR merges and the
  `sase-core-rs` wheel publishes. sase master stays green before that, because
  `build-core` builds the wheel from `sase-core` master.
- The one lane that needs the published wheel is `release-core-floor` in
  `.github/workflows/ci.yml`, which runs only on `release-please--branches--master` PRs
  and installs the declared floor. It fails until `sase-core-rs` 0.25.0 is published,
  and self-heals on the next push, because `sync-release-metadata` runs
  `tools/ratchet_core_window` and `ceiling_specifier_for_floor` widens the window to
  `>=0.25.0,<0.26.0`.
- Sibling phase agents holding an older `sase-core` clone will see "artifact-reference
  wire is stale" from `just check` until they re-run `sase repo open sase-core`, which
  updates the workspace to `origin/master`.
- Phase `builtins` (`sase-js.4`) owns switching the Python callers from
  `artifact_ref_parse` to `artifact_ref_parse_canonical`, adding `stitch`/`patch` to
  `BUILTIN_ARTIFACT_REF_KINDS` and the completion catalogs, and retiring the `commit`
  branches in `artifact_ref_prompt_rendering.py`.

## 11. Out of scope

Resolution of `@stitch`/`@patch`/`@file` paths, `PromptRefContext`, the pluggy provider
registry and `use:`/inline merge, config schema deltas, the content-addressed object
store, prompt link rewriting and `Referenced By` write-back through the publication
outbox, ACE sub-tabs, and all documentation. Those are phases `registry`, `builtins`,
`files`, `linking`, `ace`, and `adopt`. This phase ships the primitives they call.
