---
tier: epic
title: One profile-driven query engine for every Artifacts pane
goal:
  Replace the pane-specific Artifacts query dialects with one profile-parameterized
  syntax and evaluation contract shared by the Rust parser and batch corpus, the Python
  reference evaluator, and every Artifacts filter bar; migrate Patch to that inline bar
  and scope saved slots, history, and remembered selection by pane without losing legacy
  user state.
phases:
  - id: rust_profile
    title: Profile-driven Rust syntax and corpus evaluation
    depends_on: []
    size: medium
    description:
      "rust_profile: introduce the compiled query profile in sase-core, pass it per call
      through parsing and persistent-corpus evaluation, support declared fields,
      searchable values, closed sigils and zero-argument predicates, and extend Rust and
      binding parity tests without persisting the profile as a versioned wire."
  - id: python_reference
    title: Python schema compiler and reference evaluator parity
    depends_on:
      - rust_profile
    size: medium
    description:
      "python_reference: compile each ArtifactsPaneContract query schema into a stable
      profile and digest, generalize the Python reference evaluator, canonicalization,
      highlighting and completion metadata to the same semantics, and prove parity with
      the Rust parser and evaluator over the golden corpus."
  - id: pane_persistence
    title: Pane-scoped query persistence with safe legacy migration
    depends_on: []
    size: medium
    description:
      "pane_persistence: namespace saved slots, history stacks and per-query
      ArtifactEntryTarget selection memory by pane id, store source, canonical form and
      profile digest, preserve compatibility readers, and migrate only after a
      successful write and reread while treating malformed state as empty."
  - id: token_panes
    title: Migrate the four token-filter panes without semantic drift
    depends_on:
      - python_reference
      - pane_persistence
    size: medium
    description:
      "token_panes: configure Plans, Beads, Stitches and Files from their pane
      contracts, route them through the shared engine with boolean mode disabled,
      preserve their existing canonical token behavior byte-for-byte, enable declared
      provider properties and negation, and extend the synthetic-provider conformance
      fixture."
  - id: patch_cutover
    title: Move Patch onto the shared inline query surface
    depends_on:
      - python_reference
      - pane_persistence
      - token_panes
    size: medium
    description:
      "patch_cutover: profile Patch with boolean mode enabled, route its persistent Rust
      corpus through the shared profile, replace QueryEditModal with a persistent inline
      FilterBar, move slot submission and project-scope rewriting into shared actions,
      bind f consistently, and preserve Escape rollback, match counts, coverage and
      legacy query behavior."
  - id: query_conformance
    title: Cross-evaluator conformance, visuals, docs and performance gate
    depends_on:
      - patch_cutover
    size: medium
    description:
      "query_conformance: run every golden case through Rust one-row and
      persistent-corpus evaluation plus the Python reference evaluator for all five
      panes and the synthetic provider, verify persistence migrations and pane
      isolation, update reviewed visual snapshots and query documentation, and
      demonstrate navigation p95 below 16 ms with no keystroke-path I/O."
proposed_by: bbugyi200.athena.sase-m6.6
parent_bead: sase-m6.6
create_time: 2026-09-09 20:00:34
status: wip
---

- **PROMPT:**
  [prompts/202608/unified_artifacts_query.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/unified_artifacts_query.md)
- **PARENT:**
  [202608/artifacts_pane_contract.md](https://github.com/sase-org/sase--plans/blob/main/202608/artifacts_pane_contract.md)

# Plan: One profile-driven query engine for Artifacts

## Outcome

Every Artifacts pane consumes the query schema already carried by its
`ArtifactsPaneContract`. The host compiles that schema into an immutable profile used by
the Rust tokenizer/parser and persistent batch corpus, the Python reference evaluator,
and Python-owned completion/highlighting UI. Sidecar providers gain boolean filtering
over declared properties without executing provider code, while Patch retains its full
language and joins the same inline filter interaction as the other panes.

The final system has one semantic oracle exercised in three execution modes: Python
reference evaluation, Rust single/batch evaluation, and Rust persistent-corpus
evaluation. Query persistence is pane-scoped and target-based, so loading a slot or
navigating history can never switch or mutate a hidden pane.

## Constraints and invariants

- Query profiles are compiled host data passed as call arguments. They are not persisted
  as a new provider-spec field or versioned Rust wire, and the provider spec remains at
  schema version 1.
- The profile vocabulary is closed and host-validated. Providers may select declared
  fields but cannot invent punctuation, callbacks, matchers, commands, or executable
  code.
- `boolean=False` reproduces the current flat-token behavior for Plans, Beads, Stitches
  and Files. Patch uses `boolean=True` and remains the compatibility oracle for boolean
  expressions, parentheses, literals, sigils and canonicalization.
- Searchable text is derived once per snapshot from fields marked `searchable`; neither
  rendering nor typing may resolve providers, touch disk, invoke subprocesses, or
  rebuild the corpus.
- Compiled profiles are cached by profile digest. Query results are cacheable by pane
  id, snapshot generation, profile digest and canonical query.
- Persisted selections use canonical `ArtifactEntryTarget` tokens, never Patch names or
  row indices. Invalid or obsolete targets fail closed and fall back to normal stable
  selection.
- Existing public Patch-query callers outside ACE keep compatibility defaults until a
  deliberate later migration; this phase must not silently change CLI or Axe semantics.
- A malformed provider field or persisted record degrades locally and visibly. It does
  not remove another pane or corrupt the persistence file.

## Phase details

### rust_profile — Profile-driven Rust syntax and corpus evaluation

In the linked `sase-core` repository, define the minimal immutable compiled-profile
types required by the grammar: boolean mode, field definitions and aliases, searchable
fields, a closed sigil mapping, zero-argument predicates, and legacy macro groups.
Replace tokenizer references to the static property allowlist and hard-coded sigil
emissions with profile lookups while retaining an explicit Patch compatibility profile
for existing entry points.

Thread the profile through query compilation and both evaluator routes. Extend
`QueryCorpus` so searchable values and predicate inputs are compiled once with the
snapshot rather than flattened on each query. Expose profile-aware functions through the
Python binding while retaining compatibility wrappers. Add unit and wire tests for
unknown fields, invalid aliases, unsupported sigils, boolean-disabled parsing, stable
digests, and persistent-corpus reuse.

### python_reference — Python schema compiler and reference evaluator parity

Add a Python `ArtifactQuerySchema`/compiled-profile adapter at the contract boundary.
Compile Patch's built-in schema and provider `ref.properties` into the same normalized
shape, including temporary coercion of declared datetime and string-list values from the
current string-valued wire. Derive completion keys, values, negatable keys, searchable
fields, highlighting and canonicalization from that profile rather than filter-bar class
constants.

Generalize the Python reference evaluator without removing it: it remains the readable
semantic oracle for tests and non-batch call sites. Extend the shared golden corpus so
each case declares its profile and expected canonical form, parse result and matched
targets. Compare those expectations against both the Python implementation and the new
Rust binding, covering flat and boolean modes, quoted/case-sensitive literals, field
negation, macros, sigils, predicates and provider-declared properties.

### pane_persistence — Pane-scoped query persistence with safe legacy migration

Introduce typed persistence records for saved queries, history, and selection memory.
Each saved query records source text, canonical text and profile digest; history owns
independent previous/next stacks per pane; selection maps pane plus canonical query to a
canonical `ArtifactEntryTarget` token. Keep deterministic JSON and atomic writes.

Readers accept the existing flat Patch files and project them into the Patch namespace.
Do not delete or overwrite legacy state until the namespaced representation has been
atomically written and read back successfully. Corrupt, partially migrated, unknown-pane
or unknown-version records produce an empty local result and a diagnostic rather than an
exception on the UI thread. Add fixtures for first migration, repeat migration,
interrupted writes, invalid targets and isolation between every pane.

### token_panes — Migrate the four token-filter panes without semantic drift

Replace the Plans, Beads, Stitches and Files filter parsers with shared profile-driven
configuration obtained from the pane contract. Begin with `boolean=False`, compare old
and new canonical forms and result sets against the foundation goldens, and remove each
old route only after its pane is equivalent. Ensure Files gains declared negatable keys
through the shared configuration.

Make the synthetic provider prove the extensibility claim: its declared properties must
automatically appear in completion, highlighting and filtering without provider Python
code or a provider release. Build typed/searchable row inputs when the provider snapshot
is loaded off-thread, and keep filter keystrokes limited to compiled in-memory data.

### patch_cutover — Move Patch onto the shared inline query surface

Compile the built-in Patch schema with boolean mode, legacy status macros, sigils and
state predicates enabled. Route the existing stable-identity `QueryCorpus` cache through
that profile and verify that repeated filtering does not rebuild the corpus or profile.

Replace `QueryEditModal` for Patch with the shared persistent inline `FilterBar`. The
bar supplies profile-derived completion and highlighting, live match count, coverage,
Escape restoration and visible parse errors. Move `#N` save handling into its submit
path and preserve the `0`-prefixed slots and `*` picker because bare digits select
sub-tabs. Make `f` open/focus the bar on Patch and all sibling panes, and update the
help modal. Implement Patch project scope as a rewrite of only the `project:` token;
choosing all projects removes that token and retains the rest of the committed query.

Slot loads, previous/next history and selection restoration operate only on the current
pane session. Remove the hard switch to Patch and the hidden-pane history mutation. Keep
compatibility adapters for non-TUI Patch query entry points and delete modal-only logic
only when no production caller remains.

### query_conformance — Cross-evaluator conformance, visuals, docs and performance gate

Parametrize the Artifacts conformance harness over Patch, Plans, Beads, Stitches, Files
and the synthetic provider. For each profile, compare tokenization, canonicalization,
parse failures and matches across the Python evaluator, Rust evaluation over rows, and
the persistent Rust corpus. Exercise provider property completion, zero-argument
predicates, boolean-disabled rejection, profile-digest changes and malformed-value
degradation.

Verify all three persistence stores against their golden files, including a slot with
the same digit in multiple panes, independent history stacks, stale profile digests and
selection targets that disappear after refresh. Add an interaction test proving that
typing `^` on Beads never changes Patch state and loading a provider slot never changes
the active sub-tab.

Update `docs/query_language.md` to describe profiles, common syntax, per-pane fields and
legacy compatibility. Update help content and reviewed PNG snapshots for the persistent
Patch bar and saved-query picker. Run install and repository checks in each modified
repository; use monitored exhaustive checks where required. Measure navigation and
filter interaction with `SASE_TUI_PERF=1`, require p95 below 16 ms on every pane, and
inspect stall logs to confirm parsing, persistence and corpus construction never reach
the event loop or Textual message pump.

## Verification strategy

Each implementation phase starts with the repository's install step and runs focused
tests while iterating. Rust changes run the query unit suite, evaluator parity suite,
binding tests, formatting and clippy/test gates. Python phases run the query goldens,
core facade tests, persistence tests, Artifacts contract suite, filter-bar interaction
tests and selection tests.

Before a phase closes, run `just check` in the SASE repository. Because this work
touches the Artifacts broadening set, run `just check-full` through the SASE monitor
with a concrete next action before landing the combined tree. Review any changed PNG
snapshot artifacts before accepting them; exact visual changes are expected only for
Patch's modal-to-inline transition and related help/picker chrome.

## Follow-up boundary

Do not widen `ArtifactEntryWire.properties` in this epic. Coercion at the Python
snapshot boundary is intentional temporary compatibility; a future core task can add a
typed value wire after the provider-spec release floor permits it. Do not migrate the
Agents tab's separate `agent_query` dialect, remove legacy persistence readers, or
perform the broader Artifacts keymap unification assigned to later phases of the parent
epic. Phase workers record any such discoveries as `PROPOSED FOLLOW-UP:` notes on their
assigned bead rather than creating tasks.
