---
tier: tale
title: ArtifactsPaneContract and derived, explainable capabilities
goal:
  Every Artifacts pane is driven by one host-owned contract whose capabilities are
  derived from declared facts and explainable in the TUI and CLI.
size: medium
proposed_by: bbugyi200.athena.sase-m6.4
bead: sase-m6.4
create_time: 2026-08-14 20:04:21
status: done
---

- **PROMPT:**
  [prompts/202608/artifacts_pane_contract_1.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/artifacts_pane_contract_1.md)
- **PARENT:** [202608/artifacts_pane_contract.md](artifacts_pane_contract.md)
- **BEAD:**
  [sase-m6.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m6/sase-m6.4.md)

# Plan: ArtifactsPaneContract and derived, explainable capabilities

## Goal

Make one immutable, widget-free `ArtifactsPaneContract` the source of truth for every
Artifacts sub-tab. The contract must describe both built-in panes and document-provider
panes, derive a closed set of host capabilities from declared facts through named pure
rules, explain every enabled and disabled verdict, and drive the TUI and CLI surfaces
that currently infer behavior from pane ids or the `ref:` prefix.

This phase preserves the established behavior of Patch, Stitch, Bead, File, and Plan
while making a fully declarative synthetic provider receive the host's generic document
features without provider Python. It declares empty extension points for the later
query, relation, grouping, shell, and presentation phases; it does not implement those
later features or alter the Rust provider-spec schema version.

## Constraints and design decisions

- Keep the contract and its compiler beside `ArtifactsTabDescriptor` in the widget-free
  artifact-tab model/descriptor modules. Textual widgets may consume a contract but the
  contract layer must not import Textual.
- Preserve `ArtifactsTabDescriptor` as the discovery/rendering envelope (provider spec,
  degradation diagnostics, mounted widget id), but store the behavioral declaration in
  exactly one attached contract. Compatibility properties may delegate to the contract
  during migration; duplicated behavioral values may not drift independently.
- Define `PaneCapability` as a closed enum of host-owned verbs. Include the currently
  shipped generic behaviors (entry navigation/open, filter/query session, refresh,
  project scope, stable marks, detail scrolling, stable-reference copy, saved/history
  query state, and version or mutation operations where the pane actually supports them)
  plus dormant enum members/contract fields needed by the later relation, grouping,
  status, and shell phases. A capability enables registered host behavior; it never
  contains callbacks, commands, widgets, colors, or provider code.
- Compile a small immutable declared-facts record for each pane. Built-in facts come
  from one host-owned adapter table. Provider facts come only from the normalized
  schema-v1 `ref` declaration (`inventory`, `properties`, `identity`, `publication`,
  `detail`, and the optional suppression block). In particular, inventory plus fields
  earns filter/history/saved-view behavior, stable reference facts earn copy-reference,
  revision facts earn versions, and mutation is available only to built-in adapters.
- Implement each derivation as a named pure rule returning an auditable verdict with the
  capability, ON/OFF state, rule name, concrete fact/reason, and optional suppression
  reason. Preserve verdicts for both states rather than retaining only the enabled set.
  A provider may suppress an earned capability only through a normalized
  `ref.capabilities.suppress` mapping whose values are non-empty reasons; it may never
  assert a capability. Unknown capabilities, non-string/blank reasons, or attempted
  assertions must produce a visible provider diagnostic/degraded descriptor instead of
  disappearing or being silently ignored.
- Keep provider-spec wire `schema_version: 1`. Treat the suppression block as additive
  Python-owned metadata that survives the existing permissive wire validator, and add it
  to the inline sidecar ref-key allowlist so inline and `use:` provider paths behave
  consistently. Do not add `ref.pane`; that belongs to the later declaration phase.
- Populate all requested contract fields now: id, label, icon, accent, order, digit, ref
  kind, target prefix, project scoping, presentation digest, capabilities and verdicts,
  query schema, relations, grouping, detail fields, status counters, empty state, and
  copy targets. Later-phase fields may use typed empty/default values, but their
  representation and digest contribution must already be deterministic.
- Compile contracts once while resolving/caching descriptors. Hot action, render,
  completion, footer, and navigation paths may only perform in-memory contract lookup;
  they must not rediscover providers, read files, stat paths, parse config, or invoke
  provider code.
- Preserve current user-facing key assignments and semantics. The later `query`,
  `relations`, `shell`, and `keymap` phases own query unification, relation UI, visual
  redesign, and key migration.

## Implementation

### 1. Contract model, compiler, and descriptor integration

- Add the enum and immutable contract/fact/verdict records in
  `src/sase/ace/tui/_artifact_tab_model.py`, with convenience predicates for capability
  checks and a deterministic serializable explanation payload.
- Add a focused widget-free compiler module if needed to keep descriptor construction
  readable. Define the built-in fact/adaptor table there, extract provider facts from
  normalized specs, validate suppressions, run every named rule for every enum member,
  and calculate the presentation digest from normalized host presentation inputs plus
  the provider spec digest.
- Change fixed and provider descriptor construction to attach contracts, then update
  digit assignment with `dataclasses.replace` on the contract as well as the envelope.
  Export contract lookup APIs from `artifact_tabs.py`, including an exact lookup for
  diagnostics/CLI use that does not silently normalize an unknown id to Stitches.
- Keep degraded panes present. Their contract should expose only safe host behavior and
  retain the discovery/compiler error so other panes are unaffected.
- Add unit coverage for every named derivation rule, all built-in contract snapshots,
  provider fact extraction, deterministic digests, valid suppression, invalid
  suppression degradation, and digit/contract synchronization.

### 2. Shared snapshot lifecycle and contract-aware panes

- Introduce `ArtifactsSnapshotPane` in the Artifacts widget package as the shared
  off-thread, coalesced, last-request-wins snapshot loader used by Documents, Beads, and
  Files. Centralize worker ownership, loading/error state, pending force/full requests,
  cancellation, and terminal rescheduling. Give subclasses narrow hooks to build,
  validate, accept, and render their typed snapshots so Files can retain paging,
  generations, and detail workers and Documents can retain deep-archive scheduling.
- Pass the resolved contract into every mounted pane, including Patch, and let pane
  identity, label, accent, provider kind/spec, detail fields, empty state, and copy
  declarations come from it. Keep specialized renderers/adapters; this is a behavioral
  contract, not a forced generic widget.
- Re-run the existing Bead, File, and Document loading tests with explicit cases for
  coalescing, cancellation, stale project/generation results, and refresh. Assert the
  shared base never performs synchronous data collection on the event loop.

### 3. Replace pane-id/ref-prefix behavioral dispatch

- Add one cheap active-contract accessor on `ArtifactsView`/the app and migrate every
  `ref:` prefix dispatch site under `src/sase/ace/tui/` to contract identity,
  capabilities, target prefix, declared copy group/targets, or adapter metadata. Cover
  action availability, command-palette filtering/context, query editing, project scope,
  detail scrolling, stable-reference capture/resolution, copy-mode routing/previews, and
  keybinding-mode context.
- Replace the document pane's inherited Plan naming in generic runtime paths. Keep Plan
  approval/rejection and bead-link operations as built-in Plan-only capabilities;
  unknown providers receive only behavior earned from their facts. Compatibility aliases
  may remain at module/API boundaries where tests or persisted keymaps require them, but
  generated labels, warnings, groups, and help must use the active contract's provider
  label/ref kind.
- Make copy groups and registered targets contract-driven. Plan retains its richer
  built-in targets, while an unknown stable-ref document provider receives the generic
  reference/link/path/title/body/JSON/handoff/snapshot targets under its own pane group.
  Availability must be the intersection of contract declarations and selected-row data,
  not a `ref:` shortcut.
- Generate Artifacts pane help sections from the resolved contracts and capability-to-
  action metadata, leaving Patch-specific workflow sections specialized. Extend the
  conformance harness to assert every contract-declared action/copy target has a
  registered host implementation and every unavailable action is explained by an OFF
  verdict.
- Add a source-level regression assertion that behavioral modules contain no
  `startswith("ref:")`/equivalent provider dispatch. Parsing or rendering a canonical
  artifact ref string may still use `ref:` only outside behavioral pane dispatch and
  must be documented in the assertion's allowlist.

### 4. Explainability CLI and synthetic third-party conformance fixture

- Add the alphabetically registered nested command `sase artifact pane show <pane_id>`,
  plus `-j/--json`, through `parser_artifact.py`, `artifact_handler.py`, and a focused
  `artifact_cli/pane.py` handler. The text view should identify the pane/contract and
  list every closed enum capability as ON or OFF with its named rule, declared fact, and
  suppression reason. JSON should expose the same stable information. Unknown pane ids
  must fail clearly and list configured ids rather than falling back to another pane.
- Update artifact group help/examples and parser/handler tests, keeping nested commands
  and options alphabetized and every public long option paired with a short alias.
- Add a declarative synthetic provider fixture under
  `tests/ace/tui/artifacts_contract/fixtures/`: schema-v1 provider data and sample
  Markdown only, with no SASE plugin class, callback, or fixture-side Python provider.
  Feed it through normal sidecar normalization/descriptor compilation and include it in
  conformance cases alongside built-ins, Plan, degraded providers, and resolved
  providers.
- Assert the synthetic pane earns its query-session, navigation/refresh, marks,
  detail-scroll, project-scope, stable-reference copy, generated-help, and generic copy
  declarations from facts; assert it cannot earn mutation, versions, Plan approval, or
  undeclared later-phase relation/grouping behaviors.

## Verification

1. Run `just install` before any test or lint command.
2. Run focused model/descriptor, provider discovery, sidecar normalization, parser/CLI,
   copy registry/palette, help modal, action availability, conformance, and the three
   snapshot-pane loading suites while iterating.
3. Run the existing Patch contract goldens and the Artifacts navigation, marking,
   copy-reference, copy-mode, command-catalog, and keymap tests to prove compatibility.
4. Run the Artifacts j/k benchmark with `SASE_TUI_PERF=1` for every available pane,
   including the synthetic provider, and keep cached navigation p95 below 16 ms. Check
   that contract lookup and capability checks allocate no provider/discovery work on a
   keystroke.
5. Run `just check`. Because this phase touches broad Artifacts behavior, run
   `just check-full` only through `/sase_monitor` with a `--next` action, inspect its
   terminal result, and fix/re-run until green.
6. Re-run the focused conformance and CLI tests after the full-suite fixes, then verify
   the worktree diff contains no unintended generated files or unrelated edits.

## Completion criteria

- Every resolved Artifacts descriptor owns one deterministic contract and every closed
  capability has an auditable named ON/OFF verdict.
- `sase artifact pane show` explains those verdicts in text and JSON, including valid
  provider suppressions and their reasons.
- Behavioral TUI dispatch no longer infers provider behavior from `ref:` or labels an
  unknown provider as Plan; availability, help, footer/copy surfaces, and detail routing
  consume the contract.
- Documents, Beads, and Files use the shared non-blocking `ArtifactsSnapshotPane`
  lifecycle without losing coalescing, stale-result rejection, paging, or detail
  behavior.
- A no-code synthetic schema-v1 provider participates in the conformance suite and gains
  only the capabilities its declared facts earn.
- Focused verification, performance checks, `just check`, and monitored
  `just check-full` all pass.
