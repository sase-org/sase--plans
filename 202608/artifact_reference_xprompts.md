---
tier: epic
title: Artifact reference xprompts
goal: Define artifact-reference renderers as contextual `#ref/` xprompts, automatically
  synthesize and configure them for sidecar repositories, and enforce one shared path-filter
  contract in resolution and completion.
phases:
- id: core-ref-contract
  title: Add the shared reference and filter contract to sase-core
  depends_on: []
  size: large
  description: 'core-ref-contract: extend the Rust content-layout, artifact-reference,
    catalog, and completion wires with contextual ref sources, path filters, and deterministic
    filtered-path behavior.'
- id: python-ref-registry
  title: Build the Python ref registry and sidecar configuration
  depends_on:
  - core-ref-contract
  size: large
  description: 'python-ref-registry: consume the new core contract, load `sase/refs`,
    synthesize sidecar ref xprompts, ship builtin renderers, and expose validated
    config and catalog metadata.'
- id: artifact-rendering
  title: Route artifact expansion through ref xprompts
  depends_on:
  - python-ref-registry
  size: medium
  description: 'artifact-rendering: make `#ref/` and `@` use one late resolver-renderer
    pipeline while preserving staging, consumption tracking, builtin output compatibility,
    and Jinja safety.'
- id: reference-completion
  title: Unify filtered completion across invocation surfaces
  depends_on:
  - core-ref-contract
  - python-ref-registry
  size: medium
  description: 'reference-completion: drive TUI and LSP completion for both `@kind:`
    and `#ref/kind` from the same filtered artifact inventory and invalidate it on
    all relevant source/config changes.'
- id: integration-and-docs
  title: Prove the end-to-end contract and document it
  depends_on:
  - artifact-rendering
  - reference-completion
  size: medium
  description: 'integration-and-docs: add cross-surface tests, migration/config documentation,
    and full combined-tree verification for artifact reference xprompts.'
proposed_by: bbugyi200.athena.vw
create_time: 2026-08-08 13:31:48
status: wip
bead_id: sase-ho
---

- **PROMPT:** [prompts/202608/artifact_reference_xprompts.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/artifact_reference_xprompts.md)
- **BEAD:** [sase-ho](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ho/README.md)

# Artifact reference xprompts

## Goal

Treat each supported artifact-reference kind as a contextual xprompt named `ref/<kind>`.
Users can continue to write the compact artifact form `@<kind>:<payload>`, or invoke the
same definition explicitly as `#ref/<kind>:<payload>` (including the existing
parenthesized and named-argument xprompt forms). Both spellings must share one resolver,
one renderer, one consumption record, and one completion inventory.

Reference definitions live under `sase/refs/`, never `sase/artifact_refs/`. Enabled
sidecar repositories automatically contribute the appropriate reference xprompt. A
path-backed sidecar defaults to accepting Markdown files and rendering a reference as:

```text
the <file_path> file in the <sidecar> sidecar repo
```

For example, with the standard research sidecar enabled:

```text
@research:202608/artifact_reference_rendering/artifact_reference_rendering.md
```

renders as:

```text
the 202608/artifact_reference_rendering/artifact_reference_rendering.md file in the research sidecar repo
```

The design is informed by
`research:202608/artifact_reference_rendering/artifact_reference_rendering.md`, with two
deliberate changes: the source directory is `sase/refs/`, and the public namespace is
singular `#ref/`.

## Non-goals

- Do not create `sase/artifact_refs/` or expose `#artifact_ref/`.
- Do not let a template file register a new artifact kind or resolver. The artifact
  registry remains authoritative, so adding a file cannot reinterpret ordinary `@word:`
  prose.
- Do not let templates change resolution, checkout materialization, staging, consumption
  tracking, or completion semantics. Templates control only prompt text.
- Do not add recursive artifact-reference expansion. Text emitted by a ref renderer is
  output, not a second artifact-reference program.
- Do not move shared resolution or filtering rules into Python. They are backend
  behavior and belong in `sase-core`.

## User-facing contract

### Names and source locations

The effective project exposes one contextual xprompt per known artifact kind:

```text
#ref/commit
#ref/chat
#ref/bug
#ref/file
#ref/bead
#ref/agent
#ref/plans
#ref/research
...
```

The first six names are the existing built-in kinds. `plans` and custom sidecar roles
such as `research` are path-backed document kinds. Enabled sidecars decide which
contextual document names exist; disabling or removing a sidecar removes its generated
entry. The built-in `beads` and `agents` sidecars customize the existing singular `bead`
and `agent` kinds rather than creating misleading plural aliases.

Reference renderer sources use the same project/home/plugin/package model as other
xprompt content, but live in a distinct physical directory:

- project: `sase/refs/<kind>.md`
- home: `~/sase/refs/<kind>.md`
- project-specific home override: `~/sase/refs/<project>/<kind>.md`
- plugin: `refs/<kind>.md`
- packaged defaults: `src/sase/xprompts/refs/<kind>.md`

The filename stem owns the artifact kind. A file in `sase/refs/` must opt in with
`ref: true` frontmatter; ordinary `sase/xprompts/` definitions cannot claim the reserved
`ref/` namespace. Conversely, a `ref: true` definition placed in an ordinary xprompt
directory is rejected with a source-aware diagnostic. The loader also rejects unknown
kinds, duplicate definitions at the same precedence, directory traversal, and files
whose declared metadata conflicts with the filename-derived kind.

The effective renderer precedence is:

1. selected-project `sase/refs/<kind>.md`;
2. an explicit `repos.sidecar.*.<role>.ref.xprompt` value for that project;
3. home and project-specific-home ref files, in the established home xprompt order;
4. plugin ref files, in deterministic plugin order;
5. packaged built-in ref files;
6. the generated default for a path-backed sidecar.

Filters do not follow renderer precedence: the effective sidecar configuration always
owns the path policy for its role, even when a file overrides its wording. Catalog and
doctor output identify the winning source and shadowed sources. Ref definitions are
contextual like memory xprompts: a selected project exposes `#ref/research`, not
`#<project>/ref/research`.

### Canonical input contracts

The registry, not renderer frontmatter, supplies the single canonical input for each ref
xprompt. This prevents a wording customization from silently changing the artifact
grammar or completion behavior. At minimum, use these public argument names and types:

| Kind class                   | Argument      | Xprompt input type | Completion source                                  |
| ---------------------------- | ------------- | ------------------ | -------------------------------------------------- |
| path-backed sidecar/document | `file_path`   | `path`             | filtered files under that sidecar's document roots |
| commit                       | `commit`      | `line`             | existing commit artifact inventory                 |
| chat                         | `file_path`   | `path`             | existing chat artifact inventory                   |
| bug                          | `bug`         | `line`             | existing bug inventory                             |
| file                         | `artifact_id` | `line`             | staged-file artifact inventory                     |
| bead                         | `bead_id`     | `word`             | existing bead inventory                            |
| agent                        | `agent_name`  | `agent`            | existing agent inventory                           |

A ref file may add a description and renderer metadata, but may not redefine this input
contract. `sase xprompt list/show`, the ACE picker, and LSP catalog serialize ref
entries with `kind: ref`, their `ref_kind`, their synthesized input, and their source
provenance. Definition navigation opens the winning file when one exists and the owning
SASE config location for a config-generated renderer.

### Sidecar configuration

Extend each sidecar repo entry with an optional `ref` mapping. A path-backed sidecar
supports:

```yaml
repos:
  sidecar:
    custom:
      research:
        repo: sase--research
        ref:
          xprompt: |
            the {{ file_path }} file in the {{ sidecar }} sidecar repo
          filters:
            path_globs:
              - "**/*.md"
```

`xprompt` is a Markdown/Jinja body, not a path. When it is absent, a path-backed sidecar
uses the exact default shown above. When `filters.path_globs` is absent, it defaults to
`['**/*.md']`. An explicitly empty list means no files are valid; users opt into other
extensions by replacing or extending the positive patterns.

Define `path_globs` once and use it everywhere:

- patterns match normalized, repo-relative POSIX paths;
- matching is case-sensitive;
- `**/` matches zero or more directories, so the Markdown default includes both
  `README.md` and nested `foo/README.md`;
- positive patterns are ORed;
- a matching `!pattern` vetoes a positive match;
- a negative-only list starts from allow-all and then applies vetoes;
- unsafe absolute or parent-traversing payloads remain invalid before glob matching.

The same matcher controls exact resolution, drift resolution candidates,
canonicalization, `@` completion, and `#ref/` argument completion. A filtered file is
not treated as merely missing: return a stable `filtered` resolution status and a
diagnostic that identifies the kind, normalized payload, and active filter without
leaking absolute local paths into prompt text.

The same `ref.xprompt` field may customize the renderer associated with the built-in
`beads` and `agents` sidecars. Their artifact kinds remain `bead` and `agent`, and
`ref.filters` is rejected for these non-document kinds rather than silently ignored.

### Rendering and expansion

`#ref/<kind>` is a special catalog entry: xprompt parsing validates its one canonical
argument and rewrites the invocation to the corresponding canonical artifact token. It
does not render early with ordinary xprompts. The existing late artifact pass then:

1. scans and parses the artifact token;
2. resolves it through `sase-core`, including sidecar filters;
3. performs any checkout or file materialization required by the existing kind;
4. stages and records the structured artifact using the raw and canonical reference;
5. selects the effective `ref/<kind>` renderer;
6. renders the prompt text once with strict undefined-variable checking.

This makes `#ref/research:foo.md` and `@research:foo.md` observably equivalent after
parsing: the same status, canonical reference, resolved file, consumption entry, filter
decision, and final prompt text.

Each renderer receives its canonical input variable plus a primitive `ref` mapping. Do
not expose mutable model objects to templates. The stable initial context is:

```yaml
ref:
  raw: "@research:foo.md"
  canonical: "@research:foo.md"
  kind: research
  kind_type: document
  payload: "foo.md"
  fragment: null
  occurrence_index: 0
  resolved_path: "foo.md"
  checkout: null
  url: null
  project: sase
  sidecar: research
```

For a document kind, `file_path` and `ref.resolved_path` are the normalized resolved
sidecar-relative path (including an accepted drift repair), while `ref.payload`
preserves the authored payload. The convenience variable `sidecar` is the configured
role. Built-in templates may use additional registry-defined convenience variables, but
they must be derived from the same primitive mapping and documented in tests.

Ship packaged templates for the six built-in kinds that preserve the existing output
byte-for-byte. Path-backed sidecars intentionally use the new relative-path sentence
instead of exposing `@<absolute_path>`. Renderer identity and a digest may be added to
the staging manifest for provenance, but prompt consumption continues to record the
artifact rather than treating the template as a newly consumed file.

Move VCS file-reference materialization out of `_artifact_ref_replacement` before making
the renderer pure. Protect rendered artifact spans from the later top-level Jinja pass
so literal `{{` or `{%` emitted by a ref template cannot accidentally turn the entire
prompt into a new Jinja template. Preserve the established subsequent `@file` handoff
where a built-in renderer deliberately emits one, while preventing another `@kind:` or
`#ref/` expansion pass.

## Phase 1: Add the shared reference and filter contract to sase-core

**Slug:** `core-ref-contract`

Update the sibling `sase-core` repository first because content placement, artifact
validity, canonicalization, and editor inventories are shared backend behavior.

### Work

- Extend the content-layout wire and helpers with `refs` source records, the physical
  `refs` segment, the singular reserved `ref` namespace, contextual
  `ref_reference_name`, and placement validation. Keep memory and skill behavior
  unchanged and bump the content-layout schema version.
- Extend the native xprompt catalog model with ref-kind/source metadata and synthesized
  canonical inputs so the Rust catalog fallback and Python loader serialize identical
  entries.
- Add `path_globs` to `ArtifactRefDocumentRootWire`, with the Markdown default supplied
  by the caller's effective sidecar config. Bump the artifact context schema and reject
  old/new wire mismatches explicitly.
- Implement the documented POSIX glob semantics once in Rust. Apply the matcher before
  accepting exact paths, to every drift candidate, and before canonicalizing an absolute
  file back to a document reference.
- Add a `filtered` artifact resolution status and stable diagnostics/wire encoding.
  Ensure filtered paths cannot fall through to another root in a way that bypasses a
  role's policy.
- Filter document payloads in the native artifact inventory used by `@` completion.
  Expose a batch binding or equivalent shared API for Python callers that already own a
  bounded candidate list, so they do not reimplement glob rules.
- Teach native xprompt-argument completion to recognize a catalog entry with `ref_kind`
  and use the artifact payload inventory instead of generic filesystem completion.
- Cover root-level and nested Markdown defaults, positive/negative/negative-only
  patterns, explicit empty filters, unsafe paths, exact and drift resolution,
  canonicalization, duplicate roots, schema rejection, catalog parity, and both
  completion entry points.
- Publish/release the compatible `sase-core` update before raising the Python pin.

### Exit criteria

- Rust is the only authority for whether a document payload passes its sidecar filter.
- Content layout and catalog can represent a contextual `ref/<kind>` without inventing a
  project-prefixed public name.
- Resolution, canonicalization, and native completion return the same allowed path set.
- The Rust repository's full required checks pass and a consumable release exists.

## Phase 2: Build the Python ref registry and sidecar configuration

**Slug:** `python-ref-registry`

### Work

- Raise the `sase-core` pin and update Python wire adapters atomically for both schema
  bumps; fail clearly if an old extension remains installed.
- Extend `config/sase.schema.json`, typed config parsing, config merge/normalization,
  examples, and doctor validation with `ref.xprompt` and `ref.filters.path_globs`.
  Preserve unset versus explicitly empty filters.
- Attach the effective filters to every document root constructed from enabled sidecars,
  including implicit built-in sidecars and fallback roots. Do not attach a document
  filter to bead or agent resolution.
- Add ref source discovery for project, home, project-specific home, plugins, and
  packaged defaults using the core content-layout records. Reserve `ref/`, enforce
  `ref: true`, reject unknown kinds, and implement the stated precedence and shadowing
  diagnostics.
- Synthesize an effective ref xprompt for every known built-in artifact kind and every
  enabled sidecar role. Map built-in `beads`/`agents` to `bead`/`agent`; give each
  path-backed role a `file_path: path` input and the exact default renderer text.
- Add `ref_kind`, `kind: ref`, canonical input, source/provenance, filter summary, and
  shadow information to the Python `XPrompt`/workflow/catalog models. Keep ordinary
  xprompt behavior and names unchanged.
- Ship packaged renderer files for commit, chat, bug, file, bead, and agent whose
  expected output fixtures are byte-identical to the current hard-coded replacements.
- Include ref source directories and relevant sidecar config in cache keys, refresh
  paths, and source diagnostics. A disabled sidecar must disappear after refresh.
- Add loader/config/catalog tests for every source class, precedence edge, invalid
  placement, unknown kind, implicit sidecars, custom sidecars, singular built-in
  mapping, defaults, explicit empty filters, and schema/doctor errors.

### Exit criteria

- A default project catalog contains all built-in `ref/<kind>` entries and one entry for
  each enabled path-backed sidecar.
- `research` requires no renderer config to expose `#ref/research` with a Markdown-only
  filter and the required default wording.
- Renderer wording is independently customizable without weakening the configured path
  policy.
- Python and native catalog snapshots agree on names, inputs, kinds, and source
  metadata.

## Phase 3: Route artifact expansion through ref xprompts

**Slug:** `artifact-rendering`

### Work

- Add the special early rewrite for parsed `#ref/<kind>` calls. Validate argument
  count/name/type using the catalog, preserve source spans for diagnostics, and emit a
  canonical artifact token for the existing late pass.
- Refactor the artifact prompt pipeline into explicit resolve/materialize/stage/render
  steps. Keep `_artifact_ref_replacement` pure with respect to checkout creation and
  filesystem materialization.
- Select and render the effective ref xprompt with strict undefined variables and the
  documented primitive context. Surface renderer syntax/undefined-variable errors with
  kind and source provenance.
- Preserve byte-for-byte final output for commit, chat, bug, file, bead, and agent
  fixtures. Assert the exact required relative-path sentence for plans and custom
  sidecars.
- Preserve raw/canonical artifact identity and consumption tracking for both invocation
  spellings. If provenance is stored, record renderer scope/source/digest in staging
  metadata without adding the renderer file to prompt consumption.
- Introduce span protection between artifact rendering and top-level Jinja processing,
  then unprotect after that pass. Verify literal Jinja syntax emitted by a renderer is
  not evaluated and ordinary surrounding top-level Jinja still behaves as before.
- Add failures for filtered, missing, ambiguous, invalid, and unknown references in both
  syntaxes and prove that renderer output is not recursively scanned for another
  artifact reference.

### Exit criteria

- `@kind:payload` and `#ref/kind:payload` produce identical resolved artifacts,
  tracking, errors, and prompt text.
- Built-in output compatibility is locked by golden text fixtures.
- The research example expands exactly as requested without placing an absolute local
  path in prompt text.
- A renderer cannot cause late accidental top-level Jinja evaluation.

## Phase 4: Unify filtered completion across invocation surfaces

**Slug:** `reference-completion`

This phase may proceed alongside Phase 3 after the registry is available.

### Work

- Filter the Python/TUI artifact completion catalog through the shared Rust matcher,
  retaining existing bounded scans and cancellation behavior. Never duplicate the glob
  implementation in Python.
- Route TUI `#ref/<kind>` argument assistance to the same warm artifact catalog used by
  `@<kind>:` payload completion. Preserve quoting/insertion behavior for the xprompt
  syntax while sharing candidate identity and filtering.
- Route native LSP `#ref/<kind>` argument completion to its artifact inventory and
  ensure `@` completion consumes the same filtered document roots.
- Add `sase/refs`, home ref sources, plugin ref sources, and sidecar config changes to
  LSP watchers and TUI/native cache invalidation. Switching the selected project or
  enabling/disabling a sidecar must rebuild the contextual ref names and payloads.
- Show ref entries in the ACE xprompt picker and CLI catalog with their `ref` kind,
  source, sidecar, and argument. Keep the result set bounded and avoid walking a sidecar
  twice merely because both invocation syntaxes are active.
- Add parity tests that compare sorted payloads for `@research:` and `#ref/research:`
  under the default filter, custom positive/negative patterns, an empty filter, a
  disabled sidecar, multiple roots, and refresh after config/file changes. Include root
  and nested Markdown plus excluded non-Markdown files.

### Exit criteria

- For every kind and project context, the two invocation syntaxes offer the same payload
  identities after filtering.
- A path excluded from resolution is absent from both completion surfaces.
- Ref source and sidecar config changes refresh without restarting the editor or TUI.
- Existing latency/cancellation tests show no duplicate unbounded filesystem work.

## Phase 5: Prove the end-to-end contract and document it

**Slug:** `integration-and-docs`

### Work

- Add end-to-end preprocessing tests for direct `@research:...` and `#ref/research:...`,
  including exact output, relative-path rendering, staged artifact identity,
  consumption, completion, and project-context isolation.
- Add regression coverage for all six built-in kinds, project/home/plugin/config
  precedence, explicit renderer overrides, unknown ref files, disabled sidecars,
  Markdown defaults, custom non-Markdown opt-in, negated filters, drift resolution, and
  literal Jinja output.
- Document `sase/refs/`, `ref: true`, `#ref/`, the sidecar `ref` config schema, exact
  glob semantics, defaults, precedence, canonical inputs, template context, and the fact
  that definitions customize known kinds rather than register resolvers.
- Update xprompt, artifact-reference, configuration, sidecar, completion, and plugin
  authoring documentation and examples. Explicitly state that `sase/artifact_refs/` is
  unsupported.
- Run the SASE repository's install and exhaustive combined-tree verification, using
  `just install` followed by `just check-full`. Run the corresponding full checks in
  `sase-core` before integration. Resolve any native/Python catalog snapshot or schema
  mismatch before landing.

### Exit criteria

- The documented research example, sidecar default, customization, and filter behavior
  are exercised end to end.
- There is one normative table for ref source precedence, one normative description of
  filter semantics, and no documentation suggesting the superseded directory or
  namespace.
- Full checks pass in both repositories, with the released core pin and generated
  schema/catalog fixtures in sync.

## Cross-phase invariants

- `sase-core` owns shared layout, grammar-adjacent validity, filtering,
  canonicalization, and native completion behavior; Python owns project/config assembly
  and prompt rendering.
- A template changes prose only. Resolver availability and filter policy are derived
  from the artifact registry and effective sidecar config.
- `#ref/` is contextual and singular; all physical definitions live under `refs`.
- Sidecar defaults never expose an absolute checkout path in prompt text.
- Filtered files are invalid artifacts and unavailable for completion, not merely hidden
  suggestions.
- Existing built-in artifact references preserve output, staging, and consumption
  behavior unless this plan explicitly changes path-backed sidecar wording.
- Every wire/schema version bump is paired with a clear mismatch error and tests; the
  Python pin moves only after the Rust release is available.
