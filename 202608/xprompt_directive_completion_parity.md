---
tier: epic
title: Complete xprompt directive completion in ACE and external editors
goal:
  Give every supported xprompt directive, syntax form, keyword name, and useful keyword
  value one shared, responsive completion contract across the prompt input widget and
  the xprompt LSP.
phases:
  - id: directive-contract
    title: Canonical directive completion contract in sase-core
    depends_on: []
    description:
      "directive-contract: define the complete directive and argument schema,
      grammar-aware cursor context, value-provider roles, and Python bindings in
      sase-core so frontends no longer maintain divergent hard-coded catalogs."
    size: medium
  - id: external-editor-lsp
    title: Complete contextual directive support in the xprompt LSP
    depends_on:
      - directive-contract
    description:
      "external-editor-lsp: drive LSP names, snippets, keyword names, and contextual
      values from the shared contract, extending the existing cached editor catalog
      bridge for bead and identity targets without blocking editor requests."
    size: medium
  - id: prompt-widget
    title: Complete responsive directive support in the ACE prompt widget
    depends_on:
      - directive-contract
    description:
      "prompt-widget: replace the prompt widget's private directive tables and token
      classifier with the shared contract, wire warm dynamic catalogs for bead,
      identity, model, and path values, and preserve the non-blocking completion path."
    size: medium
  - id: parity-verification
    title: Lock runtime, widget, and LSP parity with tests and documentation
    depends_on:
      - external-editor-lsp
      - prompt-widget
    description:
      "parity-verification: add exhaustive contract and interaction coverage for every
      directive and keyword, verify matching ACE/LSP behavior including failure
      degradation and UTF-16 edits, and document the complete completion matrix."
    size: medium
proposed_by: bbugyi200.athena.08s
bead_id: sase-rj
create_time: 2026-09-09 19:52:10
status: wip
---

- **PROMPT:**
  [prompts/202608/xprompt_directive_completion_parity.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/xprompt_directive_completion_parity.md)
- **BEAD:**
  [sase-rj](https://github.com/sase-org/sase--beads/blob/main/pages/sase-rj/README.md)

# Complete xprompt directive completion in ACE and external editors

## Outcome

Make directive assistance complete and trustworthy in both editing surfaces. A user who
types a directive name, an allowed syntax delimiter, a keyword fragment, or a value for
a structured keyword should see the same valid, well-described choices in ACE's prompt
input and in an LSP client. Completion must never advertise syntax the runtime
interprets differently, and adding a runtime directive or keyword must fail a parity
test until the shared completion contract is updated.

This is an epic because the canonical editor/domain behavior belongs in `sase-core`,
while the xprompt LSP and ACE prompt widget need separate integrations and dynamic
catalog adapters. The two frontend phases can proceed in parallel after the shared
contract lands.

## Audited baseline

The runtime vocabulary is:

| Directive           | Alias                                                            | Supported keyword arguments                                                                              |
| ------------------- | ---------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `%alt`              | advertised as `%{...}`; legacy `%(...)` remains parse-compatible | none; `id=value` inside a branch names that branch and is not a directive keyword                        |
| `%auto`             | `%a`                                                             | none; `plan`, `tale`, and `epic` are compatibility argument suggestions, not a closed runtime vocabulary |
| `%clan`             | `%c`                                                             | `summary=`, `summary_script=`, `tribe=`                                                                  |
| `%effort`           | `%e`                                                             | none                                                                                                     |
| `%hide`             | `%h`                                                             | none                                                                                                     |
| `%id`               | `%i`                                                             | `bead=`, `clan=`, `family=`, `tribe=`                                                                    |
| `%model`            | `%m`                                                             | every currently configured model-alias name, used as a launch-family override key                        |
| `%repeat`           | `%r`                                                             | none                                                                                                     |
| `%wait`             | `%w`                                                             | `bead=`, `priority=`, `runners=`, `time=`                                                                |
| `%xprompts_enabled` | none                                                             | none; colon values are `false` and `true`                                                                |

The audit found these concrete failures:

- ACE and the Rust completion engine both omit `%wait(bead=...)`, even though the Python
  runtime accepts it and the canonical xprompt documentation describes it.
- ACE deliberately omits `%xprompts_enabled` and therefore also omits its `false` /
  `true` values; the LSP already includes them.
- ACE has no `%repeat` value suggestions and recognizes fixed `%auto` / `%effort` values
  only in a subset of the runtime's accepted argument syntax.
- The LSP does not classify parenthesized `%model` clauses independently, so configured
  model aliases are not offered as `alias=` override keys and model values are not
  completed correctly after those keys. ACE has a separate implementation of this
  feature, which is another drift point.
- Both surfaces offer keyword names but generally stop completing after `=`. They lack
  contextual bead IDs, clan/family/tribe targets, time/integer examples, and executable
  paths. They also re-offer already-selected or mutually exclusive keywords in several
  `%id` and `%clan` contexts.
- Both surfaces currently offer `%wait` keyword names in colon-form wait lists, although
  the runtime treats colon-form entries as positional agent names; structured wait
  keywords are valid only in `%wait(...)`.
- Directive names, aliases, descriptions, argument hints, keyword tables, ordering, and
  cursor parsing are duplicated between Python and Rust, so no test makes drift a build
  failure.

## Design

### One shared directive contract

Extend `sase-core`'s editor directive metadata into a serializable contract that covers:

- canonical name, advertised alias or shorthand, description, multiplicity, and the
  exact allowed syntax forms;
- positional argument role and any finite suggested values;
- keyword name, description, value role, whether it can recur within one directive, and
  conflicts with other keywords;
- deterministic display order; and
- dynamic provider roles rather than frontend-specific callbacks.

Value roles should cover at least `model`, `model_alias_key`, `agent`, `clan`, `family`,
`tribe`, `bead`, `path_or_executable`, `bool`, `non_negative_int`, `positive_int`,
`wait_time`, `free_text`, and `gate_owned`. Keep free-form values editable: suggestions
are assistance, never an accidental allowlist.

The shared grammar classifier must return the canonical directive, syntax form, active
clause range, clause kind (positional, keyword name, or keyword value), active keyword,
and already-selected names/values. It must handle aliases, incomplete parenthesized
calls, comma clauses on either side of the cursor, quotes/text blocks where allowed, and
LSP UTF-16 positions. It must not surface keywords in invalid syntax forms.

Expose the contract and classifier through `sase_core_rs` so ACE consumes the same
source as the Rust LSP. Retain thin Python presentation adapters for Rich/Textual rows
and Python-owned live catalogs; do not copy the static directive schema back into ACE.

### Contextual value providers

Use existing bounded data sources wherever possible:

- `%model` positional and override values use the existing model completion catalog;
  override keys derive from configured alias entries, and self-references are excluded.
- `%wait(bead=...)` and `%id(..., bead=...)` use open bead candidates with IDs, titles,
  status, task type, and project context. Move shared filtering/ranking into
  `sase-core`, as anticipated by `src/sase/ace/tui/models/wait_bead_catalog.py`, while
  keeping store reads in the Python host adapter.
- `%id(..., clan=...)`, `%id(..., family=...)`, `%id(..., tribe=...)`, and
  `%clan(..., tribe=...)` filter the existing kind-aware agent catalog to the required
  target kind. Existing values remain suggestions so users can author a new tribe.
- `%clan(..., summary_script=...)` delegates to existing path completion and labels the
  value as an executable/script path. `summary=` remains free text but receives clear
  inline documentation and a safe quoted/text-block snippet.
- `%wait(time=...)`, `%wait(runners=...)`, `%wait(priority=...)`, and `%repeat` offer a
  small documented set of useful examples while accepting arbitrary runtime-valid
  values. Include the semantic distinctions (duration versus wall clock, drain barrier,
  default priority, positive repeat count) in candidate documentation.
- `%xprompts_enabled:` offers exactly `false` and `true`, including the closing marker
  while the cursor is inside a disabled region.

Suppress keywords already present in the same directive when the parser rejects a
duplicate. Enforce completion-time conflict filtering for `%id`'s mutually exclusive
`clan=` / `family=` / `tribe=` choices and `%clan`'s mutually exclusive `summary=` /
`summary_script=` pair without preventing manually typed text from reaching runtime
validation.

### Responsiveness and degradation

ACE's keystroke path must remain read-only and free of disk access, subprocesses, or
unbounded locks. Reuse the existing agent snapshot and model caches immediately; load
beads and path-backed data off the Textual event loop with coalesced, mtime-keyed
warmers, then revalidate prompt/cursor state before repainting. A cold or unavailable
dynamic catalog should leave static keyword completion working and may show a
lightweight loading/unavailable row; it must not freeze or close the menu.

The LSP should reuse its catalog cache and helper-host worker path. Extend the existing
completion catalog response compatibly with bounded bead rows rather than adding an
interactive or per-keystroke subprocess path. Helper failure or a mixed-version host
must degrade to static names and examples with a bounded warning, not fail completion.

## Phase details

### `directive-contract` — Canonical directive completion contract in sase-core

1. Expand `crates/sase_core/src/editor/directive.rs` and its wire types to represent the
   complete audited matrix, allowed syntaxes, keyword conflicts, ordering, and value
   provider roles. Include `%wait bead=` and the special `%xprompts_enabled` marker.
2. Refactor `crates/sase_core/src/editor/completion.rs` so one grammar-aware context
   classifier handles every directive and alias. Correct colon-versus-parenthesis
   behavior and represent keyword-value positions explicitly instead of overloading a
   generic directive-argument context.
3. Add shared candidate builders for static values and raw dynamic inventories,
   including bead ranking/filtering and kind-filtered identity targets. The builders
   must preserve free-form entry and return stable descriptions and replacement ranges.
4. Add `sase_core_rs` bindings for the directive contract/context and JSON-shaped
   candidate results needed by ACE. Update binding documentation and Python-wire parity
   fixtures.
5. Cover all canonical names, aliases, syntax forms, keyword names, conflicts,
   selected-clause suppression, malformed/incomplete calls, and UTF-16 ranges with Rust
   and binding tests. Run `just check` in `sase-core`.

### `external-editor-lsp` — Complete contextual directive support in the xprompt LSP

1. Replace LSP-specific directive dispatch and keyword lists with the shared context and
   contract. Render appropriate LSP kinds, documentation, filter text, text edits, and
   snippets for every directive and structured keyword.
2. Complete `%model(...)` override-key and override-value behavior from the materialized
   model catalog, including partial alias keys, multiple clauses, aliases,
   self-reference suppression, and cursor edits in an earlier clause.
3. Extend the existing editor helper agent-catalog wire compatibly with bounded
   open-bead rows and reuse its family/clan/tribe entries for keyword values. Keep the
   host bridge read-only, cached, and backward-compatible when either side lacks the new
   optional fields.
4. Route `%wait(bead=...)`, `%id(..., bead=...)`, identity membership values, script
   paths, and static numeric/time examples to the correct provider. Do not fetch agent
   targets while completing a keyword value of another kind.
5. Add core completion tests and `sase_xprompt_lsp` JSON-RPC tests for every directive,
   alias, keyword name, keyword value role, snippet-capable and non-snippet client,
   catalog failure, mixed-version payload, mid-clause cursor, and Unicode range. Run
   `just check` in `sase-core` and `just check` in `sase` for helper-bridge changes.

### `prompt-widget` — Complete responsive directive support in the ACE prompt widget

1. Replace `_USER_FACING_DIRECTIVES`, argument hints/descriptions, keyword tuples, and
   the bespoke directive clause parser under `src/sase/ace/tui/widgets/` with adapters
   over the `sase_core_rs` contract and context result.
2. Preserve ACE-specific presentation metadata while making names, aliases, allowed
   syntax, ordering, keyword descriptions, duplicate/conflict filtering, and replacement
   ranges originate in the shared core.
3. Wire value roles to the existing live model catalog, agent completion snapshot,
   kind-filtered clan/family/tribe rows, file completion, and wait-bead catalog. Move
   any shared bead ranking to the core adapter and reuse the wait modal's mtime-keyed
   raw catalog instead of adding another store scan.
4. Warm bead-backed completion off-thread with coalescing and stale-result guards. Keep
   the prompt keystroke, refresh, Tab, and acceptance paths free of synchronous I/O and
   preserve the current menu selection when a warm catalog arrives.
5. Add pure candidate/context tests plus real prompt-widget interactions for all names,
   aliases, fixed values, every keyword, contextual values, earlier-clause editing,
   automatic opening, Tab acceptance, loading/failure states, and menu refresh. Include
   a regression that reproduces the supplied `%w(..., )` screenshot and asserts a
   documented `bead=` row. Run `just check` in `sase`.

### `parity-verification` — Lock runtime, widget, and LSP parity

1. Add a machine-readable parity test in `sase` that compares the Python runtime's
   canonical directives, aliases, special disabled-region marker, fixed values, and
   supported keyword sets with the exported core contract. Configured `%model` aliases
   should be checked as a dynamic-key role rather than frozen names.
2. Add table-driven cross-surface fixtures that run every directive/alias and keyword
   context through the ACE adapter and the LSP JSON-RPC path and compare insertions,
   order, descriptions, and active replacement text. Include an assertion that colon
   `%wait:` never advertises structured keywords.
3. Verify invalid duplicates/conflicts remain typable but are not suggested, free-form
   values remain accepted by completion, dynamic-catalog failures retain static rows,
   and no completion path performs synchronous bead-store I/O.
4. Update `docs/xprompt.md`, `docs/ace.md`, and the external-editor/LSP documentation
   with the complete directive/keyword/value matrix, syntax restrictions, ordering, and
   graceful-degradation behavior. Do not edit SASE memory files without a separate,
   explicit user request.
5. Run each repository's full required verification (`just check` in `sase-core`; in
   `sase`, install the workspace dependencies first and run `just check-full` through a
   SASE monitor because this is an epic combined-tree landing). Capture any intentional
   visual change with the dedicated visual snapshot workflow only if the completion
   panel rendering itself changes.

## Acceptance criteria

- `%wait(` completion includes `bead=`, `priority=`, `runners=`, and `time=` in one
  documented, deterministic order in ACE and LSP clients; after `bead=`, matching open
  bead IDs and titles are available without blocking.
- Every row in the audited directive matrix completes in both surfaces, including
  `%xprompts_enabled:false|true`, and every advertised alias resolves to the same
  canonical metadata and argument behavior.
- Every runtime keyword is suggested only where syntactically valid, has useful
  documentation, and enters a correctly classified value context. Model alias keys and
  model values work in both surfaces.
- Duplicate and mutually exclusive keywords are omitted from suggestions according to
  runtime rules, while manually typed free-form or invalid values remain available for
  normal runtime validation.
- ACE uses only warm snapshots on the keystroke path; cold bead data loads off-thread,
  stale results do not repaint a changed prompt, and unavailable catalogs do not remove
  static completion.
- LSP completion stays functional with a missing helper, stale/mixed schema, or empty
  dynamic catalog, and all edits use correct UTF-16 ranges.
- A single exhaustive parity fixture fails whenever the Python runtime, shared core
  contract, ACE adapter, or LSP surface gains or loses a directive or keyword without
  the others being updated.
