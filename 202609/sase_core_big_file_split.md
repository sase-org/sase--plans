---
tier: epic
title: Split The Ten Largest sase-core Rust Files Into <=1500 Line Modules
goal: 'Each of the ten largest Rust files in the sase-core repo is decomposed into
  a module tree whose every file is at most 1500 lines, with no behavior change, no
  public API change, and `just check` green after each phase.

  '
phases:
- id: py_bindings
  title: Split crates/sase_core_py/src/lib.rs
  depends_on: []
  size: medium
  description: 'py_bindings: decompose the 35,166-line PyO3 binding crate root into
    a domain-keyed module tree while keeping the `sase_core_rs` pymodule registration
    intact.'
- id: agent_scan_index
  title: Split crates/sase_core/src/agent_scan/index.rs
  depends_on:
  - py_bindings
  size: medium
  description: 'agent_scan_index: decompose the 13,468-line agent artifact index module
    into wire types, index maintenance, query, and alias/output-variable history submodules.'
- id: bead_mutation
  title: Split crates/sase_core/src/bead/mutation.rs
  depends_on:
  - agent_scan_index
  size: medium
  description: 'bead_mutation: decompose the 11,316-line bead mutation module along
    its create/update/claim/close/link/store seams.'
- id: fleet_contract
  title: Split crates/sase_core/src/fleet_contract.rs
  depends_on:
  - bead_mutation
  size: medium
  description: 'fleet_contract: decompose the 10,548-line fleet wire contract into
    a fleet_contract/ module tree grouped by contract area.'
- id: gateway_routes
  title: Split crates/sase_gateway/src/routes.rs
  depends_on:
  - fleet_contract
  size: medium
  description: 'gateway_routes: decompose the 10,062-line axum router module into
    state, router assembly, and per-area handler submodules.'
- id: lsp_server
  title: Split crates/sase_xprompt_lsp/src/server.rs
  depends_on:
  - gateway_routes
  size: medium
  description: 'lsp_server: decompose the 9,939-line LSP server module into per-LSP-capability
    submodules behind the existing server entry point.'
- id: agent_launch
  title: Split crates/sase_core/src/agent_launch/mod.rs
  depends_on:
  - lsp_server
  size: medium
  description: 'agent_launch: move the 7,978-line agent_launch crate-module body out
    of mod.rs into new siblings so mod.rs becomes a thin facade.'
- id: editor_completion
  title: Split crates/sase_core/src/editor/completion.rs
  depends_on:
  - agent_launch
  size: medium
  description: 'editor_completion: decompose the 7,260-line editor completion module
    by completion source and by ranking/rendering concern.'
- id: xprompt_catalog
  title: Split crates/sase_core/src/xprompt_catalog.rs
  depends_on:
  - editor_completion
  size: medium
  description: 'xprompt_catalog: decompose the 4,850-line xprompt catalog module into
    a xprompt_catalog/ module tree.'
- id: sudo_runner
  title: Split crates/sase_gateway/src/sudo_runner.rs
  depends_on:
  - xprompt_catalog
  size: medium
  description: 'sudo_runner: decompose the 4,595-line gateway sudo runner into a sudo_runner/
    module tree and close out the epic''s file-size invariant.'
proposed_by: bbugyi200.athena.0oh
create_time: 2026-09-20 19:06:02
status: wip
bead_id: sase-14s
---

- **PROMPT:** [prompts/202609/sase_core_big_file_split.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/sase_core_big_file_split.md)
- **BEAD:** [sase-14s](https://github.com/sase-org/sase--beads/blob/main/pages/sase-14s/README.md)

# Plan: Split The Ten Largest sase-core Rust Files Into <=1500 Line Modules

Every phase of this epic does the same kind of work on a different file: take one
oversized Rust source file in the sase-core repo and split it into a module tree in
which no single file exceeds 1500 lines. The phases are chained strictly in sequence
(each `depends_on` the previous one) so that only one agent is ever editing the
sase-core working tree, and so each phase starts from a tree that already passed
`just check`.

## Shared Rules For Every Phase

Every phase agent MUST follow all of the rules in this section. They are not repeated in
the per-phase sections below.

### Opening The Repository

sase-core is a linked repo, not your workspace checkout. Use the `/sase_repo` skill
first:

```bash
sase repo open sase-core -r "Split <target file> into <=1500 line modules"
```

Use the path it prints as the only path for reads and writes. Do not clone, locate, or
web-fetch sase-core any other way. Because you will modify it, sase-core becomes a
repository obligation in your `/sase_final` declaration and needs a `commit` decision.

### Think Hard About The Split

This is the core of the work, and it is a design task, not a mechanical one. Before
moving a single line:

1. Read the entire target file. Build an inventory of its top-level items (types, free
   functions, `impl` blocks, constants, test modules) with their line ranges.
2. Identify the _conceptual seams_: which items form a cohesive unit that a reader would
   expect to find together, and which items are only adjacent by accident of append
   order. Prefer seams that follow the file's own domain vocabulary over arbitrary "part
   1 / part 2" cuts.
3. Map the internal call graph. A good seam minimizes the number of items that must
   change from private to `pub(crate)`/`pub(super)` visibility to keep compiling.
4. Sketch at least two candidate decompositions and write down, in your final response,
   which one you chose and why you rejected the other. Name any candidate split you
   considered and discarded because it would have cut a tightly coupled cluster in half.
5. Only then start moving code.

Numbered, generic module names (`part1.rs`, `impl_a.rs`, `misc.rs`, `helpers.rs`,
`utils.rs`, `common.rs`) are a sign the seam analysis was skipped. Do not use them.
Every new module's name should state what lives in it.

### The Size Invariant

- Every `.rs` file in the target's module tree must end at **1500 lines or fewer** after
  the phase, including generated-looking registration blocks and test modules.
- Test code counts. If the file's `#[cfg(test)] mod tests` block is itself larger than
  1500 lines, split it too, and split it along the same seams as the production code so
  each test file sits next to the code it covers.
- Aim comfortably below the ceiling (roughly 600-1200 lines per file) so the next
  routine edit does not immediately breach it again. Do not create a swarm of 50-line
  files either; cohesion beats line-count optimization.
- If some genuinely indivisible unit cannot be brought under 1500 lines without harming
  the code (a single large `match`, one long derive-heavy type), leave it, and state
  explicitly in your final response which file is still over and why.

### Follow The Repository's Existing Module Convention

sase-core already has the convention this epic wants; do not invent a new one. Look at
`crates/sase_core/src/query/`, `crates/sase_core/src/gate_decision/`,
`crates/sase_core/src/provider_usage/`, `crates/sase_core/src/axe_chop/`, and
`crates/sase_core/src/bead_action/` for the established shape:

- A `foo.rs` becomes `foo/mod.rs` plus sibling submodules.
- `mod.rs` keeps the module's doc comment, declares the submodules, and re-exports the
  module's public surface with `pub use`.
- File-based test modules are declared as `#[cfg(test)] mod tests;` with the body in
  `tests.rs` (or a `tests/` directory with its own `mod.rs` when the tests themselves
  need more than one file).

Do not use `#[path = "..."]` attributes to fake a flat layout.

### Pure Refactor, No Behavior Change

- **No public API change.** Every path that external callers use today must still
  resolve. Re-export moved items from the original module path with `pub use` so that
  other crates in the workspace, and the Python `sase_core_rs` binding, keep compiling
  and importing unchanged.
- Move code **verbatim** wherever possible. Adjust `use` statements, visibility
  (`pub(crate)`, `pub(super)`), and nothing else. Resist the urge to rename, reorder,
  simplify, or "improve" the code you are moving; it makes the diff unreviewable and
  hides real regressions. Genuine cleanups belong in a follow-up, not here.
- No test may be deleted, skipped, weakened, or left behind. The test count before and
  after the phase must match. If a test moves, it moves whole.
- Do not edit `[workspace.package].version`, crate `[package].version`, or local
  path-dependency version pins. release-plz owns those (see sase-core `AGENTS.md`).

### Verification

Run the repo's own gate from the sase-core repo root:

```bash
just check
```

`just check` runs `./scripts/check.sh all` (fmt, clippy, test) and is the same gate CI
runs. Per sase-core `AGENTS.md`, never verify with `cargo test -p sase_core` alone: it
excludes the `sase_core_py` binding tests. The phase is not done until `just check`
passes. Use `/sase_monitor` if the run is slow enough to block your turn.

Also confirm the invariant directly before finishing, from the sase-core repo root:

```bash
find crates -name '*.rs' -not -path './target/*' -exec wc -l {} + \
  | sort -rn | awk '$1 > 1500' | head -40
```

Your target file (and every file you created from it) must be absent from that list, or
explicitly justified as an unavoidable exception.

### Reporting

In your final response, state: the chosen decomposition and the rejected alternative,
the resulting file list with line counts, the before/after test count, and the
`just check` result.

## Split crates/sase_core_py/src/lib.rs

**Target:** `crates/sase_core_py/src/lib.rs` — 35,166 lines (largest file in the repo).
Roughly 21,284 lines of production code, with a single `mod tests` block from line
21,285 to the end (~13,880 lines).

This is the PyO3 binding crate root: ~819 `#[pyfunction]` definitions, 5 `#[pyclass]`
types, and a `#[pymodule] #[pyo3(name = "sase_core_rs")]` function at line 19,927 whose
body performs ~849 `add_class`/`add_function` registrations.

Specific constraints on top of the shared rules:

- The crate is the `sase_core_rs` extension module consumed by the Python sase repo. The
  `#[pymodule]` function, its `#[pyo3(name = "sase_core_rs")]` attribute, and the
  complete set of registered names must survive unchanged. A dropped registration is a
  silent Python-side `AttributeError`, not a compile error — verify the registration
  count before and after and report both numbers.
- The registration body alone is ~1,355 lines. Split it into per-domain
  `pub(crate) fn register_<domain>(m: &Bound<'_, PyModule>) -> PyResult<()>` helpers
  that live beside the functions they register, so the module root's registration
  function becomes a short list of `register_*(m)?` calls. Keeping registration next to
  definition is what stops the two from drifting.
- Group the `#[pyfunction]` wrappers by the `sase_core` domain they bind (beads, agents,
  fleet, notifications, editor, tool runs, and so on), mirroring the `sase_core` module
  names wherever the mapping is obvious.
- Split the test module along the same domain seams.

Given the file's size, this phase will produce on the order of 25-35 new files. Budget
the work accordingly and do the seam analysis before touching anything.

## Split crates/sase_core/src/agent_scan/index.rs

**Target:** `crates/sase_core/src/agent_scan/index.rs` — 13,468 lines, of which ~6,677
are production code and ~6,790 are the `mod tests` block.

The file already has visible seams worth evaluating:

- Wire types for the artifact index and the alias-history query surface
  (`AgentArtifactIndexQueryWire`, `AgentAliasHistoryQueryWire`, and friends).
- Index maintenance: `rebuild_agent_artifact_index`, upsert/delete row helpers,
  stale-row terminalization, hidden-terminal retention, abandoned-row repair.
- Dismissal reconciliation (`replace_agent_artifact_index_dismissed_agents`,
  `reconcile_agent_artifact_index_dismissed_family_members`).
- Read/query: `query_agent_artifact_index`, `load_agent_artifact_records`,
  `find_gate_shell_by_gate_id`, index status and vacuum.
- Alias history: `query_agent_alias_history` plus its candidate refresh, SQL clause
  building, row decoding, and prompt-snippet truncation helpers.
- Output-variable history: `query_agent_output_variable_history` plus the occurrence
  accumulators, filtering, and glob matching.

Treat that list as input to your own analysis, not as the answer. Note that the
`#[cfg(test)]` thread-local instrumentation near the top of the file
(`LAST_INDEX_SQL_STATEMENTS`, `LAST_GATE_SHELL_LOOKUP_RECORDS_DECODED`) is read by the
tests and written by production paths, so decide deliberately where it lives and keep
its visibility correct under both `cfg(test)` and normal builds.

The target becomes `agent_scan/index/mod.rs` plus siblings, alongside the existing
`agent_scan` modules (`context.rs`, `layout.rs`, `scanner.rs`, `selector.rs`,
`wire.rs`). Avoid names that collide confusingly with those existing siblings.

## Split crates/sase_core/src/bead/mutation.rs

**Target:** `crates/sase_core/src/bead/mutation.rs` — 11,316 lines, of which ~3,816 are
production code and ~7,500 are the `mod tests` block. Note the inverted ratio: the test
module is the larger half, so the test split matters at least as much as the production
split here.

Seams visible in the production half, to evaluate rather than adopt blindly:

- Mutation request/outcome wire types (`BeadCreateRequestWire`, `BeadUpdateFieldsWire`,
  `BeadLinkProjectionRequestWire`, `BeadPreclaim*Wire`, `BeadMutationOutcomeWire`).
- Create and task-plus-one / snooze / unsnooze flows, including the note-rendering
  helpers (`plus_one_wake_note`, `snooze_note`, `deferral_length_label`).
- Update and note editing (`update_issue`, `update_issues`, append/edit/remove note).
- Agent claim lifecycle (`claim_for_agent_launch`, `claim_for_agent_wait`,
  `release_agent_claim`, `preclaim_epic_work_plan`).
- Open/close/remove, including the batch close machinery (`CloseBatch`, `CloseEvent`,
  `close_one_and_delegated_parent`, descendant validation, ancestor reopen).
- Dependencies, references, and bead links, including link projection
  (`PreparedLinkProjection`, `apply_prepared_link_projection*`, canonicalization and
  matching helpers).
- The `MutableStore` type and its shared tree helpers (`sorted_children`,
  `sorted_descendants`, `collect_descendants`, `next_top_level_counter`).

`MutableStore` is used by nearly every other group, so it is the natural shared
substrate; place it where the dependency edges point rather than wherever it currently
sits. Bead status lifecycle semantics must not shift — this is a file move, not a
semantics change.

## Split crates/sase_core/src/fleet_contract.rs

**Target:** `crates/sase_core/src/fleet_contract.rs` — 10,548 lines, ~7,438 of
production code and ~3,110 of tests. Contains ~57 public functions, ~24 `impl` blocks,
and well over a hundred `*Wire` types.

This is a wire-contract module, so the seams are contract areas rather than call chains.
Candidate groupings to evaluate:

- `FleetContractError` and shared primitives.
- Installation identity (record, ensure, load, rotate, migrate).
- Locators (`OriginLocatorWire`, `ProjectLocatorWire`, `LogicalAgentLocatorWire`,
  `AgentInstanceLocatorWire`) and owner display names.
- Status vocabulary enums (`FleetRowKindWire`, `FleetFamilyRoleWire`,
  `FleetLifecycleWire`, `OwnerLivenessWire`, `ConnectionHealthWire`,
  `ObservationFreshnessWire`, `FleetStatusBucketWire`).
- Content handles, metadata, revisions, and capability sets.
- Agent resolution and projection (`OwnerResolutionFactsWire`,
  `ResolvedAgentSummaryWire`, `ResolvedAgentDetailWire`, `HumanDisplayLabelsWire`).
- Snapshot and summary responses.
- Catalog: query, paging, continuation, reset, and accumulation state machine.
- Batch lookup, detail, content read, project eligibility, and counts.
- Follows: records, tombstones, promotion, activation, diagnostics, reconciliation.

Because these types are serialized across the gateway boundary, serde attributes, field
order within each struct, default functions (such as `default_row_kind`), and enum
variant names must all be preserved exactly. Any change there is a wire break, not a
refactor. Re-export the full set from `fleet_contract/mod.rs` so
`sase_core::fleet_contract::X` continues to resolve for the gateway crate and for the
Python bindings.

## Split crates/sase_gateway/src/routes.rs

**Target:** `crates/sase_gateway/src/routes.rs` — 10,062 lines, ~4,567 of production
code and ~5,495 of tests. The test half is the larger one.

Structure to work with:

- `GatewayState`, `GatewayStateOptions`, and the supporting runtime types
  (`PairingChallenge`, `EventHub` and its inner/subscription types,
  `AttachmentTokenStore` and its record/mint types, `FleetEnrollmentRateLimiter`).
- Router assembly: `app`, `app_with_state`, `fleet_v1_routes`, and the defaults
  (`default_sase_home`, `default_host_label`, `default_machine_selector`).
- `/api/v1/*` handlers: health, session pairing, events, agents (list, resume options,
  launch, launch-image, kill, retry), changespec/patch tags, xprompt catalog, beads,
  update jobs, notifications, attachments, and gate actions.
- Fleet v1 handlers: enroll, hello, summary, catalog, batch, detail, content, project
  eligibility, events, launch (including settlement recovery and the spawned settlement
  task), mutate, attention read/inventory/resolve, and credential rotate/revoke.

`app_with_state` and `fleet_v1_routes` must keep registering exactly the same paths and
methods. Enumerate the registered routes before and after and report that the two lists
are identical — a lost `.route(...)` is a runtime 404, not a compile error. Handlers
stay `pub(crate)` at most unless they are already public.

The gateway crate already has area modules (`fleet_attention.rs`, `fleet_auth.rs`,
`fleet_launch.rs`, `fleet_mutations.rs`, `fleet_reads.rs`). Check whether some handler
bodies belong with their existing area module rather than in a new `routes/` sibling,
but do not move business logic across that boundary in this phase if doing so would turn
the diff into a behavior change.

## Split crates/sase_xprompt_lsp/src/server.rs

**Target:** `crates/sase_xprompt_lsp/src/server.rs` — 9,939 lines, ~3,520 of production
code and ~6,420 of tests. Again, tests are the larger half and need their own split.

The crate already separates `catalog_cache.rs`, `logging.rs`, `lsp_convert.rs`, and
`semantic_tokens.rs`, so this phase continues an existing decomposition rather than
starting one. Split the server along LSP capability boundaries — document lifecycle and
state, completion, hover, definition, diagnostics/publishing, code actions and commands,
formatting, semantic token plumbing, and the initialize/shutdown handshake — and keep
the `LanguageServer` trait `impl` itself as a thin dispatch layer in the module root
that delegates to the capability submodules. Preserve capability registration in the
initialize result exactly; a dropped capability degrades the editor silently.

## Split crates/sase_core/src/agent_launch/mod.rs

**Target:** `crates/sase_core/src/agent_launch/mod.rs` — 7,978 lines, ~5,441 of
production code and ~2,536 of tests.

This one is different from the others: the module directory already exists, with
siblings `admission.rs`, `condition.rs`, `conditional.rs`, `launch_hold.rs`, and
`proc_runtime.rs`. The problem is that `mod.rs` carries a large body of its own instead
of acting as a facade.

The goal is that `mod.rs` ends up as a thin facade: module declarations, the module doc
comment, and `pub use` re-exports, with the body distributed to new siblings. Decide
each moved item's home by asking which existing sibling it already collaborates with —
some of the body may belong in `admission.rs` or `proc_runtime.rs` rather than in a new
file, and folding it there is better than creating a near-duplicate module. Keep every
`agent_launch::*` path that callers use today resolving through the facade.

## Split crates/sase_core/src/editor/completion.rs

**Target:** `crates/sase_core/src/editor/completion.rs` — 7,260 lines, ~3,693 of
production code and ~3,566 of tests, with 27 public functions.

`editor/` is already a wide flat module (`argument_spans.rs`, `at_reference.rs`,
`definition.rs`, `diagnostics.rs`, `directive.rs`, `frontmatter.rs`, `fuzzy.rs`,
`hover.rs`, `placeholder.rs`, `token.rs`, `wire.rs`, `xprompt_args.rs`, and more), so
the natural move is `editor/completion/mod.rs` plus siblings rather than more top-level
`editor/` files.

Evaluate two orthogonal axes and pick deliberately between them (or a blend), and
justify the choice: split by **completion source** (directives, xprompt names and
arguments, `@` references, model aliases, placeholders, frontmatter keys, file paths),
or split by **pipeline stage** (trigger/context detection, candidate collection,
filtering and fuzzy ranking, item rendering and wire conversion). Note where
`completion.rs` already delegates to `fuzzy.rs`, `token.rs`, and `at_reference.rs`, and
do not duplicate logic that already lives there.

## Split crates/sase_core/src/xprompt_catalog.rs

**Target:** `crates/sase_core/src/xprompt_catalog.rs` — 4,850 lines, ~3,088 of
production code and ~1,760 of tests.

Becomes `xprompt_catalog/mod.rs` plus siblings. Likely seams to evaluate: catalog wire
types, on-disk discovery and parsing of xprompt sources, catalog construction and
indexing, lookup/query, and any caching or invalidation logic. The xprompt LSP crate
(`catalog_cache.rs`) and the Python bindings both consume this module, so the public
surface must be re-exported unchanged from the module root.

## Split crates/sase_gateway/src/sudo_runner.rs

**Target:** `crates/sase_gateway/src/sudo_runner.rs` — 4,595 lines with 8 `impl` blocks.
Unlike the other nine targets, it has no `mod tests` block of its own, so this is a pure
production-code split; check whether its coverage lives in `sudo_runner_main.rs` or in a
`tests/` integration target before assuming anything about where tests should go.

Becomes `sudo_runner/mod.rs` plus siblings. This is privileged-execution code: the split
must not alter the validation, argument construction, or privilege-boundary ordering in
any way. If the seam analysis suggests a change that would reorder a check, reject that
seam.

### Epic Close-Out

This is the last phase, so also re-run the repo-wide audit from the sase-core repo root:

```bash
find crates -name '*.rs' -not -path './target/*' -exec wc -l {} + \
  | sort -rn | awk '$1 > 1500' | head -40
```

Report the remaining over-1500-line files. All ten of this epic's targets, and every
file created from them, should be absent. Anything still listed is either a file this
epic never targeted (expected — the repo has more of them below the top ten) or a
justified exception from an earlier phase; say which. Do not start splitting untargeted
files.
