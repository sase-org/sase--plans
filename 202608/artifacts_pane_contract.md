---
tier: epic
status: done
title:
  One Artifacts contract — every ACE sub-tab, Patch included, behind one declared API
goal: "Every ACE Artifacts sub-tab — Patch included — is driven by one host-owned
  ArtifactsPaneContract whose capabilities are derived from declared data. A sidecar or
  artifact repo declares facts in its ref spec and inherits querying, relations,
  grouping, marks, copy, help and chrome without shipping code, so a new sub-tab feature
  is implemented once and appears in every configured provider's tab — including
  providers belonging to users we will never see.

  "
phases:
  - id: foundation
    title: Live defects, golden fixtures, and the conformance harness
    depends_on: []
    size: medium
    description: "foundation: make provider tabs deterministic and failure-visible, then
      freeze golden query/relation/persistence fixtures from current Patch behavior and
      stand up the conformance harness every later phase extends.

      "
  - id: detail
    title: Detail bands render the provider's declared fields
    depends_on: []
    size: xsmall
    description: "detail: parameterize ordered_plan_property_items by the provider's
      declared ref.detail.fields so a document tab shows its own properties instead of
      plan properties.

      "
  - id: identity
    title: One typed entry target on every pane
    depends_on:
      - foundation
    size: large
    description: "identity: promote ArtifactEntryTarget to a typed value carrying pane
      identity with a canonical token round-trip, give Patch a row target and a real
      navigator, make ArtifactEntryNavigator an ABC, and retire index-based marks and
      jump anchors.

      "
  - id: contract
    title: ArtifactsPaneContract and derived, explainable capabilities
    depends_on:
      - identity
      - detail
    size: large
    description: "contract: introduce ArtifactsPaneContract with a closed PaneCapability
      vocabulary derived from declared data by named auditable rules, collapse every
      ref-prefix dispatch site onto it, and add the synthetic third-party provider
      fixture.

      "
  - id: shell
    title: The shared shell and its visual grammar
    depends_on:
      - contract
    size: large
    description: "shell: build the one chrome every pane renders through, specify and
      implement the five canonical pane states, and replace drifting provider accents
      with a deterministic perceptual palette.

      "
  - id: query
    title: One query engine across every pane and both evaluators
    depends_on:
      - contract
    size: xlarge
    description: "query: replace three query dialects with one profile-parameterized
      engine spanning the Rust parser, the Rust batch corpus and the Python reference
      evaluator, move Patch to the inline filter bar, and namespace slots, history and
      selection memory per pane.

      "
  - id: relations
    title: Relations, reveal, and grouping as contract features
    depends_on:
      - query
      - shell
    size: large
    description: "relations: generalize the ancestors/children/siblings jumpers into
      declared hierarchy, family and link relations rendered by a host-owned panel, make
      reveal a reversible lens, and put every pane's grouping on the shared fold
      registry.

      "
  - id: declare
    title: The declarative ref.pane block
    depends_on:
      - relations
    size: large
    description: "declare: add the Python-side ref.pane block and its presentation
      digest so a sidecar declares row template, sort, facets, grouping and empty state
      as data, and ship it in the research provider.

      "
  - id: keymap
    title: Unified Artifacts keymap with a safe migration
    depends_on:
      - relations
    size: medium
    description: "keymap: unify the six Artifacts verbs whose meanings are inverted
      between Patch and its siblings, shipped together with action aliases, a doctor
      advisory and a one-shot config migration.

      "
  - id: conform
    title: Conformance, diagnostics, docs, and the performance gate
    depends_on:
      - declare
      - keymap
    size: medium
    description:
      "conform: close the epic with a parametrized conformance suite over every resolved
      sub-tab including a synthetic provider, ACE-surfaced provider diagnostics, the
      documented contract, and the navigation performance gate."
proposed_by: bbugyi200.athena.01u
bead_id: sase-m6
create_time: 2026-09-09 19:49:57
---

- **PROMPT:**
  [prompts/202608/artifacts_pane_contract.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/artifacts_pane_contract.md)
- **BEAD:**
  [sase-m6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m6/README.md)

# Plan: One Artifacts contract

## Why this shape

The Artifacts tab is five browser implementations wearing one tab bar. The widget
package is 13,506 lines; four panes each carry their own filter model, row identity,
detail scheduling and footer wiring, and Patch carries a fifth stack that the view
refuses to treat like the others — `ArtifactsView.entry_navigator` raises
`ValueError("Patches use the existing Patch navigation model")`
(`src/sase/ace/tui/widgets/artifacts/view.py:194`). Provider specs already reach the
panes and are then ignored: nothing reads them for presentation.

The consequence is the thing this epic exists to remove. Today a new sub-tab feature is
written five times or excluded from the most capable pane, and a sidecar author gets a
tab that renders rows and nothing else. The fix is not a generic widget and not a plugin
callback API. It is **one behavioral contract with multiple adapters**: providers
declare facts, the host derives capabilities from those facts, and every downstream
surface asks the contract instead of comparing pane ids.

```text
sidecar ref spec          fields, relations, safe presentation hints — data, never code
        ▼
sase-core                 typed rows, profile-driven parse/evaluate, validated spec
        ▼
ArtifactsPaneContract     derived capabilities + shared shell + pane session
        ▼
adapter                   Patch is the richest built-in; providers get defaults free
```

The single highest-leverage fact: **`ref.properties` already carries `type`, `values`
and `source`, is already validated in Rust, and is already on the tab descriptor.**
Every declared property becomes queryable, completable and facetable the moment the
query layer reads it — no spec change, no wire change, no work by the sidecar author.
That is the "one implementation, every unknown user's sidecar benefits" payoff, and it
is available at schema version 1 today.

Patch is **contract-in, spec-out**. `patch` is in `RESERVED_KINDS` (`sase-core`
`crates/sase_core/src/artifact_ref/provider_spec.rs:27`), so it can never be a document
provider. It consumes the contract from a built-in Python table and never declares one.
Write that asymmetry into the docs so the next reader does not try to "fix" it.

## Grounding

Verified in this workspace at `357c45c72`, against `sase-core` `4170150` (v0.27.2) and
`sase-research-artifacts` at its linked checkout. Numbers below are measured, not
carried.

| Fact                                               | Evidence                                                                                                                                                                                                   |
| -------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Patch is excluded from stable navigation           | `widgets/artifacts/view.py:189-195`                                                                                                                                                                        |
| `ref:`-prefix dispatch is spread across the TUI    | 24 sites in 12 files under `src/sase/ace/tui/`                                                                                                                                                             |
| Provider accents collide with Plans                | `_PROVIDER_ACCENTS[0]` and `ARTIFACTS_ACCENTS["ref:plan"]` are both `#AF87FF` (`artifact_tabs.py:63,78`)                                                                                                   |
| Provider accents drift when a sidecar is installed | assignment is `index % len(...)` over the sorted kind list (`artifact_tabs.py:333`)                                                                                                                        |
| Accent resolution mutates a module global          | `ARTIFACTS_ACCENTS.setdefault(tab_id, accent)` (`artifact_tabs.py:365`)                                                                                                                                    |
| Provider discovery swallows failures               | four bare `except Exception` blocks at `artifact_tabs.py:380,407,415,483,515`                                                                                                                              |
| Navigator protocol is half-implemented             | `request_entry_target` / `conditional_footer_entries` exist only in `plans_navigation.py` and `beads_navigation.py`; call sites use `getattr(pane, ..., None)` (`actions/artifacts_navigation.py:103,119`) |
| Patch marks are indices                            | `EntryJumpAnchor = int \| PatchBannerJumpAnchor` (`actions/navigation/jump_hints.py:52`); `patch_list.py:64` holds `set[int]`                                                                              |
| Saved slots hard-switch panes                      | `actions/patch/_query.py:58-67`                                                                                                                                                                            |
| Query history is not sub-tab scoped                | `action_prev_query` gates only on `current_tab != "artifacts"` (`actions/patch/_query.py:205`)                                                                                                             |
| Files alone omits `NEGATABLE_KEYS`                 | set in `commit_filter_bar.py:70`, `bead_filter_bar.py:101`, `plan_filter_bar.py:62`; absent in `file_filter_bar.py`                                                                                        |
| A Research filter bar renders Plans-purple         | `plan_filter_bar.py:31` reads `ARTIFACTS_ACCENTS["plans"]` at class-definition time                                                                                                                        |
| `o` is double-booked on Patch                      | `default_config.yml:355` `mark_pr_origin` and `:410` `cycle_grouping_mode`                                                                                                                                 |
| The typed-value gap is real                        | `ArtifactEntryWire.properties` is `BTreeMap<String, String>` (`sase-core` `artifact_ref/entry.rs:40`)                                                                                                      |
| The research provider declares four properties     | `create_time`/`updated_time` datetime, `status` **string**, `tags` string_list; `detail.fields` declared and ignored                                                                                       |
| Additive spec fields are free at v1                | no `deny_unknown_fields` on the provider-spec wire (it appears only in `axe_overrun/wire.rs` and `axe_chop/wire.rs`)                                                                                       |

## Where this plan contradicts the research

The research report `202608/artifacts_query_and_pane_contract/` is the basis for this
plan and most of it is adopted unchanged. Six places are deliberately different, and
each one is a decision a phase worker must not silently revert.

**1. Identity is a typed value, not a bare tuple.** The consolidated report and `__b`
specify L0 as giving Patch a tuple like its siblings —
`patch_row_target(patch) -> ("patch", project, name)` — keeping
`ArtifactEntryTarget = tuple[str, ...]`. That is too weak for what the same reports
require elsewhere. Both demand cross-pane relation targets in the wire _from day one_
("retrofitting pane identity into relation targets later is expensive"), and both
require per-query selection memory persisted as `{pane_id: {canonical_query: target}}`.
A bare `tuple[str, ...]` carries pane identity only by convention and serializes to a
schemaless JSON array. This plan promotes the target to a frozen dataclass with a
canonical token round-trip in the `identity` phase, which is exactly the retrofit both
reports warn against paying for twice. Aligns with `__a` §3.6; departs from the
consolidated §4 (L0).

**2. There are two evaluators to generalize, not one engine.** The consolidated report's
first correction states that the Patch query engine "already exists in the right place"
and frames the query phase as de-Patch-ifying it. That is right about the _parser_ and
wrong about evaluation. `src/sase/core/query_facade.py` documents in its own module
docstring that `evaluate_query`, `evaluate_query_with_context` and `build_query_context`
"remain Python-owned host logic," while the TUI hot path uses a third route — the Rust
`QueryCorpus` batch evaluator, compiled once per stable patch-list identity
(`actions/patch/_loading.py:161`, `_state_init_navigation.py:167`). So the query phase
must profile-parameterize a Rust parser, a Rust batch corpus, _and_ a Python reference
evaluator, kept honest by `crates/sase_core/tests/query_evaluator_parity.rs`. The report
also overstates the Rust size: `crates/sase_core/src/query/` is 2,289 lines including
595 of tests, plus a 313-line parity test — not 3,428. This is why `query` is sized
`xlarge` and its worker authors a child epic rather than attempting it in one pass.

**3. Derived capabilities need an explainer.** Both passes conclude capabilities should
be derived from declared data rather than declared by the provider, and this plan agrees
— it is the decision that makes a future host feature light up in a sidecar written
before that feature existed. But neither report gives derivation any introspection.
Invisible derivation fails precisely the audience this epic serves: a third-party author
whose `~` does nothing has no way to learn why. The `contract` phase therefore requires
each capability to come from a **named rule that records its verdict and the declared
fact behind it**, surfaced by a CLI and in the help modal, with opt-out requiring a
reason string.

**4. The keymap unification is severable and lands late.** The consolidated report folds
the six-key rename into the Patch cutover inside the query phase. This plan splits it
out. Only `f` (open the filter bar) is load-bearing for the modal-to-inline migration
and lands with `query`; `y`, `R`, `s`, `L` and `o` are taste decisions on the owner's
primary surface that would otherwise be able to stall the contract. As its own phase the
rename is independently revertible and cannot block L0–L4.

**5. The synthetic third-party provider arrives at L1, not at the end.** Both reports
put the synthetic no-SASE-Python provider in the final conformance phase. By then every
layer has been designed against two built-ins and one first-party sidecar, and will have
absorbed their assumptions. It is the _only_ fixture that tests "providers belonging to
users we will never see," so it lands with the contract and every later phase extends
it.

**6. Beauty is a specified deliverable.** Neither report designs anything visual; "same
look and feel" is asserted and never defined. The `shell` phase owns a written visual
grammar and the five canonical pane states, including a _designed_ degraded state —
because a degraded tab is a third-party author's first encounter with their own mistake.

Two smaller corrections, adopted without changing the plan: the report attributes
`deny_unknown_fields` to `runner_limit_override.rs` (it is in `axe_overrun/wire.rs` and
`axe_chop/wire.rs`; the conclusion that additive spec fields are free at v1 still
holds), and it counts 26 dispatch sites in 13 files where the tree now has 24 in 12.
Stitches also has no `commit_row_target` helper — it builds targets inline in
`commits_timeline.py:89` — so the `identity` phase has one more call site than the
reports imply.

## Non-negotiable constraints

These hold in every phase.

- **Providers declare facts, never UI.** No callbacks, widgets, colours, keybindings,
  command strings, shell interpolation or Python entry points in sidecar config, ever.
  Anything needing those is a built-in pane: a `sase` change, not a plugin change.
- **No provider code runs during render, navigation, completion or query evaluation.**
  Declarations are compiled at discovery so the host always knows which path can block.
- **The Rust provider-spec wire stays at schema version 1.** `provider_spec.rs:18-25`
  documents why: CI installs the published floor `sase-core-rs` and runs it against the
  checkout, so a v2 spec is rejected before the core carrying the bump ships. Bead
  `sase-lm` is a live instance of that failure. Additive optional fields are free at v1;
  making a field required takes two releases (core ships the validator, floor ratchets,
  specs emit it). `ref.pane` is Python-side only — a Rust-modelled block would digest
  against whichever core is installed, making the content digest core-version-dependent.
- **A broken provider degrades visibly, never silently.** One invalid spec must never
  remove a tab, and must never remove _another_ tab. A malformed document degrades per
  entry. Diagnostics travel with the snapshot.
- **Digits and accents are host-assigned.** Providers never allocate either.
- **Performance.** Read `sase/memory/tui_perf.md` before implementing any phase. Build
  typed values, searchable text, completions and relation indexes once per snapshot, off
  the event loop. Cache compiled profiles by digest and results by
  `(pane_id, generation, profile_digest, canonical_query)`. A keystroke must never
  resolve providers, glob files, call Git, parse Markdown or stat the filesystem. Hold
  `SASE_TUI_PERF=1` navigation p95 under 16 ms on every converted pane, measured after
  each conversion.

## Phases

### foundation — Live defects, golden fixtures, and the conformance harness

Three defects make the surface non-deterministic today, and no later phase can be
trusted on top of them. Fix them first; each is independently user-visible.

_Accents._ Derive a provider accent deterministically from a hash of the `ref_kind`,
resolved against the pinned built-in set so a provider can never draw a built-in's
colour, and return it on the descriptor. Never write to `ARTIFACTS_ACCENTS`. This kills
all three symptoms at once: the `#AF87FF` collision with Plans, the repaint-on-install
drift (installing a `design` sidecar currently moves `research` from `#5FAFFF` to
`#5FD7AF`), and the module-global leak that `reset_artifacts_subtabs_cache()` does not
undo. Delete the hand-rolled `try/finally` key-popping workaround in
`tests/ace/tui/test_artifact_tab_digits.py`; if that workaround is still needed, the fix
is incomplete. The palette itself is designed in `shell` — here, produce the mechanism
and keep the current colour values.

_Discovery._ Narrow the bare `except Exception` blocks in the provider-record loader,
render a **degraded tab that carries its own error** instead of no tab, and stop caching
the `("unavailable",)` sentinel. Today, with `sase_core_rs` unimportable, resolution
returns four tabs — the built-in Plans tab vanishes too — with no error anywhere in the
UI, and the degraded answer survives an explicit cache reset. Also stop discarding the
`missing_ref_provider` diagnostic that `sase doctor` already raises: ACE currently
builds a tab from an empty default spec and nothing looks wrong.

_Grouping key._ Bead `sase-m5` (ready) covers `o` being double-booked on the Patch pane,
where `mark_pr_origin` shadows `cycle_grouping_mode` and forward grouping-cycle is
unreachable. Do not file a duplicate. If it is still open when this phase runs, fix it
here and close it with a note, because `relations` builds on that binding.

Then freeze the oracle. Capture golden fixtures **from current Patch behavior before any
code moves** — this is what keeps the query profile shape honest later:

- query fixtures: precedence, quoting, case-sensitive literals, every sigil, implicit
  AND, invalid queries and their messages, and selection results;
- relation fixtures: parent chains, cycles, missing parents, families, cross-kind edges;
- persistence fixtures: real `saved_queries.json`, `query_history.json` and
  `query_selections.json` files.

Finally, stand up the conformance harness any adapter can run. It starts nearly empty
and every later phase extends it; its existence from day one is what makes each phase's
exit condition checkable rather than asserted.

### detail — Detail bands render the provider's declared fields

`ordered_plan_property_items` ignores `ref.detail.fields`, so the research provider
declares `status`, `create_time`, `updated_time`, `tags` and its detail band shows plan
properties instead. Parameterize the function by the provider's declared fields, falling
back to today's plan ordering when none are declared.

No spec change, no Rust change, no new data path — and both research passes
independently rate it the highest value-per-line change available. It is deliberately
unblocked so it can land in parallel with `foundation`.

### identity — One typed entry target on every pane

Everything above this depends on being able to name a row. Patch already has a stable
identity outside the TUI — `builtin_entry_patch.py` mints `patch:{project}/{name}` — and
the TUI does not use it.

Promote `ArtifactEntryTarget` from `tuple[str, ...]` to a frozen dataclass carrying
`pane_id` and `parts`, with a canonical `token` round-trip (`from_token` / `to_token`)
so a target survives JSON persistence and can name a row on a pane other than the one
asking. See contradiction 1: this is the deliberate departure from the research, and it
is what stops `query`'s selection memory and `relations`' cross-pane edges from each
paying for the same retrofit. Accept the legacy tuple at construction during the
migration so panes convert one at a time.

Then:

- add `patch_row_target`, and a named `commit_row_target` for Stitches, which builds
  targets inline today;
- make `ArtifactsPatchesPane` implement the navigator and delete the `raise` in
  `entry_navigator`;
- migrate `marked_indices` to the shared stable-target mark set and remove the pane
  branch from `action_toggle_mark`; migrate the `app.marked_indices` reads in
  `src/sase/testing/ace_page.py` and `ace_page_group.py` with it;
- widen `EntryJumpAnchor` to `ArtifactEntryTarget | PatchBannerJumpAnchor`;
- convert `ArtifactEntryNavigator` from a `Protocol` to an ABC and give Files and
  Stitches the two methods they are missing, deleting both `getattr(pane, ..., None)`
  paper-overs. Cross-pane deep links and conditional footer hints silently do nothing on
  two of four panes today;
- give Patches an `@patch:` copy target — a live ref kind with a resolver that cannot be
  copied from the pane that displays it.

Only user-visible change: marks survive refresh and re-sort, and deep links start
working on Files and Stitches. Both are fixes.

### contract — ArtifactsPaneContract and derived, explainable capabilities

`ArtifactsPaneContract` lives beside `ArtifactsTabDescriptor` in the widget-free
`artifact_tabs.py` and carries id, label, icon, accent, order, digit, `ref_kind`,
`target_prefix`, `project_scoped`, `presentation_digest`, `capabilities`,
`query_schema`, `relations`, `grouping`, `detail_fields`, `status_counters`,
`empty_state` and `copy_targets`. The later-phase fields are declared here and populated
by their phases.

`PaneCapability` is a **closed enum of verbs the host already implements**. A capability
enables a registered action; it never carries executable code. It is the vocabulary
every downstream surface reads instead of comparing pane ids: the footer, the
action-availability chain, the help modal's hand-written per-pane sections, the command
palette, the copy registry and the conformance suite. All 24 `ref:`-prefix dispatch
sites across 12 files collapse onto it, including the `plans-detail-scroll` fallback
that every provider pane currently borrows.

**Capabilities are derived from declared data, never declared by the provider.** An
inventory plus fields implies filter, history and saved views; a hierarchy relation
implies `<`/`>`; a family relation implies `~`; stable refs imply copy-reference;
revision metadata implies versions; only built-in adapters can expose mutation.
Derivation gives three things declaration cannot: a provider cannot claim a feature its
data cannot support, provider config stays free of boolean soup, and **a new host
capability lights up for providers written before it existed** — which is the whole
point of the epic.

Per contradiction 3, derivation is implemented as a set of **named rules**, each a pure
function from declared facts to a verdict plus the fact that produced it. The rule set
is the single source of truth for capability state, and its output is introspectable:
add a `sase artifact pane show <pane_id>` surface that prints every capability, ON or
OFF, and the named rule and declared fact behind the verdict. A provider may _suppress_
a capability it does not want, never assert one it has not earned, and suppression
requires a reason string that appears in that output.

Also land here: the `ArtifactsSnapshotPane` base that absorbs the loader written three
times, per-provider copy groups and generated help sections, and — per contradiction 5 —
the **synthetic third-party provider fixture** carrying no SASE-specific Python. Every
later phase extends it.

Exit condition: a provider pane no longer calls itself Plans anywhere, and no downstream
surface branches on a pane id.

### shell — The shared shell and its visual grammar

The contract makes panes describable; this phase makes them look like one product. It
owns a written visual grammar in `docs/` that later phases and future panes conform to,
not just code.

_Chrome._ One header, one filter-bar position, one status/count treatment, one footer
grammar, one split behavior, one detail-scroll region. A pane's identity comes from its
accent, icon and rows — never from bespoke chrome.

_Five canonical states_, each designed rather than incidental: **loading** (cached
content renders immediately with a progress affordance, never a blank pane), **empty**
(the provider's declared empty state, distinguishing "nothing exists" from "nothing
matches this query," the latter offering the way back), **results**, **stale** (a badge
over live content while a refresh coalesces in the background — a stale badge beats a
frozen UI), and **degraded** (the tab is present, named and navigable, showing the
provider kind, its config source and the validation problem). The degraded state is a
design deliverable, not an error string: it is a third-party author's first encounter
with their own mistake, and it is what stops a bad spec from looking like a bug in SASE.

_Accent palette._ Replace the six-colour cycling list with a curated, perceptually
spaced palette, assigned by the deterministic hash mechanism from `foundation` and
collision-resolved against the pinned built-in accents. The properties that matter: a
kind's colour never changes when an unrelated sidecar is installed, a provider can never
draw a built-in's colour, and the tab row stays legible with eight or more providers
configured on every theme ACE supports.

Fix `plan_filter_bar.py:31` here — it reads `ARTIFACTS_ACCENTS["plans"]` at
class-definition time, so a Research filter bar renders Plans-purple. It disappears once
the bar takes its accent from the contract.

Expect `tests/ace/tui/visual/snapshots/png/` goldens to move. Review those diffs; do not
accept them blind.

### query — One query engine across every pane and both evaluators

**This phase is `xlarge` and its worker is expected to author a child epic plan before
implementing.** It is the whole plan's risk: three dialects, five panes, three
persistence files, one modal-to-inline UI change and a Rust refactor across two
evaluators. Do not attempt it in one pass.

_What exists._ Three languages, and every column of the comparison has at least one
implementation that is right. `ace/query` (Patch) has the boolean grammar, the sigils,
the highlighting, the slots and the history, and its parser is already in Rust.
`ace/agent_query` is a self-declared fork of it that added the one thing the unified
design needs — a typed property-key registry. `FilterBar` has the completion menu, the
live match count, the coverage label and the inline interaction, and is already
class-var driven (`ACCENT`, `KEY_COMPLETIONS`, `NEGATABLE_KEYS`, `PERSISTENT`, …) — 80%
of a field schema written as class attributes. The unified layer is a **merge**, not a
rewrite, and it makes the strongest surface strictly better rather than levelling down
to the weakest.

_Where it lives._ Keep the syntax layer in Rust and make it **profile-parameterized,
with the compiled profile passed as a call argument rather than a persisted wire.** A
per-call argument freezes nothing, so the field-schema shape can churn across releases
without a `schema_version` bump. Authoring stays in Python — an `ArtifactQuerySchema`
per pane, compiled to a profile at pane construction — so the discipline of proving the
shape across five panes before committing to it is preserved in full. What crosses the
binding is a profile, not a schema. Completion, highlighting and the Textual bar stay
Python.

Per contradiction 2, three things must move together: the Rust tokenizer's
`VALID_PROPERTY_KEYS` allowlist and inline sigil emissions, the Rust `QueryCorpus` batch
evaluator that the TUI actually filters through, and the Python reference evaluator that
`query_facade` documents as host logic. `query_evaluator_parity.rs` plus `foundation`'s
golden fixtures are the oracle for all three.

_How it lands safely._ Three properties make incremental migration possible, and they
are load-bearing:

1. **It is a strict superset of all three dialects.** `boolean=False` reproduces today's
   flat token language exactly, so each pane migrates independently and revertibly, then
   opts into `OR`, parens and case-sensitive literals.
2. **Every canonical form is preserved**, so saved queries and history files stay
   readable across the migration.
3. **Free text stops being special** — `searchable=True` flags replace the hand-written
   corpus flattener.

Migrate the four token panes first at `boolean=False` (byte-identical behaviour, one
bead each), then cut Patch over at `boolean=True` last, because it is the owner's
primary surface and the most expensive rollback. Design against Patch throughout; cut it
over at the end.

_What generalizes._ Sigils become a per-profile `sigil → field` map from a
**host-validated closed vocabulary** — providers name a field, never invent punctuation.
The state markers `!!!`, `@@@`, `$$$` become **declared zero-arg predicates** (name,
sigil, label, matcher), and candidates exist everywhere the moment the mechanism does:
`@@@` on Beads is "has a launched agent," `!!!` on Stitches is "has a failed hook."
Patch's `%` status aliases start as a legacy profile macro group.

_Patch's UI joins its siblings._ The inline `FilterBar` replaces `QueryEditModal`, which
is most of what "same look and feel" means, and delivers four things Patch lacks: a
completion menu over its own keys and values, a live match count, a coverage label, and
Escape-restores-last-committed. The `#N` save grammar moves out of the modal's dismiss
callback — where it is undiscoverable and untestable — into the shared bar's submit
handler, where completion can advertise it. `f` opens the filter bar on every pane
including Patch; this is the one keymap change that is load-bearing here, and the rest
waits for `keymap`. Patch's project scope becomes symmetric by copying the pattern
Stitches already ships: `p` rewrites the `project:` token while preserving every other
committed token, and "All projects" removes it. Declare Patch's bar `PERSISTENT` — its
query is already always visible; make that a declaration rather than an accident.

_Persistence._ Namespace all three stores per pane with a read-time migration and a
silent fallback to empty: `saved_queries.json` becomes `{pane_id: {slot: …}}`,
`query_history.json` becomes `{pane_id: {prev, next}}`, and `query_selections.json`
stores entry targets rather than Patch names. Store the source text **plus** the
canonical form **plus** the profile digest, so a renamed provider field yields a visibly
invalid saved view with an edit path rather than silent reinterpretation. Do not delete
the legacy files until the new store has been written and re-read successfully. Slots
stay behind the `0` prefix and the `*` picker — bare digits are spent on sub-tab
selection. Two live bugs die here: slot loading no longer hard-switches to Patches, and
`^` on the Beads pane no longer rewinds the Patch query behind a hidden pane.

Fix `file_filter_bar.py`'s missing `NEGATABLE_KEYS` while here — `-key:` silently works
on three panes and not the fourth. It disappears when the bar is configured from the
contract.

_The typed-value gap._ `ArtifactEntryWire.properties` is `BTreeMap<String, String>`, so
a spec may declare `type: datetime` or `string_list` and the wire flattens it. `tags` on
the research provider is the concrete casualty. Coerce on read in Python for now;
widening the wire is `sase-core` follow-up work, scheduled after this epic, not inside
it.

Exit condition: one engine, five panes, every golden fixture green, and `ref.properties`
driving completion on a provider tab with no provider release.

### relations — Relations, reveal, and grouping as contract features

Patch's jumpers are the richest feature in the tab and the least obviously
generalizable. `ancestors_children_panel.py` (612 lines) computes three families off a
prebuilt `PatchGraphIndex`; `navigation/_tree.py` (346 lines) drives `<`/`>`/`~` and the
multi-key child buffer. The structural insight from the research, adopted whole: when a
target is outside the current result set, navigation **rewrites the query** — so the
jumper and the query language are one feature, which is why this phase cannot precede
`query`.

_Three primitives, not one._ **Hierarchy** is directed and transitive (parent/child).
**Family** is an equivalence class over a grouping key — Patch "siblings" are a
normalized base name including reverted variants, _not_ graph siblings, and collapsing
both into a parent pointer would lose Patch semantics. **Link** is an ordinary typed
edge (produced-by, owns, documents), including cross-kind. Providers declare edges by
naming properties; the core derives inverses and detects cycles; anything needing a real
traversal (Beads' dependency graph) stays a built-in `RelationSource` gated by
capability. **Allow cross-pane relation targets in the wire from day one** even if the
first UI only opens them — the typed target from `identity` is what makes this free
rather than a retrofit.

_Reveal is a reversible lens._ Keep the query rewrite as the underlying mechanism, but
present it as a temporary identity lens with a visible "return to query" affordance.
Jumping through a graph must not destroy a composed query. Missing targets render as
dangling links with a diagnostic and must never invalidate the pane.

_The host owns the UI._ `AncestorsChildrenPanel` becomes `RelationPanel` in the shared
shell; key assignment, the multi-key buffer, hint rendering and the fallback are all
host-owned. Providers supply only edges.

Relations that exist in the data today and are unreachable, all of which light up here:
Beads epic→phases, `deps` and plan link; Documents proposal→active→archive and
plan↔bead; Files logical→versions; Stitches commit→parent and commit→patch.

_The first non-Patch family already exists._ The research sidecar encodes one in
filenames: a swarm bundle is `<name>/<name>__a.md`, `<name>__b.md`, `<name>.md`, where
the consolidated report is the parent and the drafts are the family — structurally
identical to Patch's `__<N>` siblings. It needs no frontmatter, no schema change and no
work from the sidecar author, which makes it the ideal first conformance case on a real
third-party provider rather than a built-in.

_Grouping._ `GroupFoldRegistry` is already generic and already shared by Agents and
Patches, keyed on `tuple[str, ...]`. It needs a third consumer, not a design. Declare
`grouping` on the contract, let providers declare `group_by`, and render banners through
the one registry — collapsed banners stay first-class jump targets. Files' flat date
separators, Documents' section headers, Beads' epic tree and Stitches' day headings
converge here.

**Hot-path warning.** `update_relationships_from_index` exists precisely so that 100
selections do not rebuild the graph 100 times. The relation panel is the specific
performance risk in this epic; preserve the prebuilt-index discipline rather than
rebuilding per selection, and measure.

### declare — The declarative ref.pane block

The payoff phase: a sidecar author writes data and gets a tab.

`ref.pane` carries row template, `group_by`, `default_sort`, facets, empty state, label,
description, order, and the query and relation hints from the two previous phases. It is
Python-side at `DOCUMENT_REF_PROVIDER_SPEC_SCHEMA_VERSION`; the Rust wire stays at 1.
Add `REF_PANE_CONFIG_KEY` to `_KNOWN_REF_CONFIG_KEYS` in `sidecar_ref_config.py` or
inline `ref: pane:` will fail validation while a `use:`-provided plugin spec silently
succeeds — the two paths already diverge. Compute the Python-side `presentation_digest`
over the normalized block and fold it into the descriptor and every pane cache key.

Constraints that keep the plugin surface bounded: presentation refers only to declared
properties; list fields are priority hints and detail stays lossless; facets are typed;
identity is stable across content edits; a malformed document degrades per entry and
never removes the tab; no colours, keybindings, command strings, Python entry points,
mutation or approval flows. Unknown optional hints fall back to host defaults; unknown
_required_ constructs produce a visible disabled pane, never a silent disappearance.

Then prove it end to end in `sase-research-artifacts`: ship its `pane` block and flip
`status` from `string` to `enum` with values. That four-character edit is the cheapest
possible demonstration of the entire thesis — a sidecar author gains completion, a facet
and a grouping mode without touching SASE. Note honestly that today's `type: string`
declaration yields substring matching only; declaring the enum is what buys the quality.

By the end of this phase, `ref.kind` plus `inventory.globs` alone earns: a tab with a
stable digit and non-colliding accent; lazy off-thread coalesced loading; shared project
scope; `j`/`k`/`enter`/`f`/`R`; marks, hint-jump and detail scrolling; a boolean query
language over declared properties with completion, negation, canonicalization,
highlighting, saved slots, history and per-query selection memory; relation jumpers over
any declared parent or family property; grouping and folding; a copy-mode group; a
generated help section; palette entries; `@kind:<path>` round-tripping; and
`Referenced By` write-back.

### keymap — Unified Artifacts keymap with a safe migration

Patch and its siblings assign the same keys to inverted meanings, and this is the real
cost of Patch inclusion. Per contradiction 4 it lands as its own phase so it can be
deferred, revised or reverted without unwinding the contract.

| Key     | Patch today                         | Siblings today            | Contract verb                             | Resolution                                                                                                                  |
| ------- | ----------------------------------- | ------------------------- | ----------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `y`     | refresh                             | copy sha / copy reference | `artifacts_copy_reference`                | Patch refresh moves to `R`; `y` copies `@patch:`                                                                            |
| `R`     | `start_rewind`                      | refresh                   | `artifacts_refresh`                       | `start_rewind` to bang mode or leader                                                                                       |
| `s`     | `change_status` (mutates)           | cycle a facet (filters)   | —                                         | Genuine semantic conflict; Beads already sides with Patch. Keep `s` = mutate where `MUTATE` is declared; move facet-cycling |
| `l`/`h` | fold in/out                         | expand/collapse           | `artifacts_expand` / `artifacts_collapse` | Same concept; unify                                                                                                         |
| `L`     | `expand_all_folds`                  | open bead / open plan     | `artifacts_link_jump`                     | Fold-snap moves under the `z` fold prefix Patch already owns                                                                |
| `o`     | `mark_pr_origin` (shadows grouping) | open external             | `artifacts_open_external`                 | `mark_pr_origin` to bang mode                                                                                               |
| `d`     | `show_diff`                         | `stitches_toggle_sdd`     | —                                         | Legitimately pane-local; leave                                                                                              |

`f` already moved in `query`. `m`, `u`, `'`, `enter`, `j`, `k`, `g`, `G` already agree.

**Ship all three mitigations or none:** action-name aliases in the keymap loader so
existing `~/.config/sase/sase.yml` overrides keep working, a `sase doctor` advisory
naming each replacement, and a one-shot `sase config` migration. `y` flipping from
refresh to copy on the owner's primary surface is the single most jarring change in this
plan and additionally needs a `CHANGELOG.md` line and a first-run toast. Update
`src/sase/default_config.yml`, which is where keymap defaults live.

### conform — Conformance, diagnostics, docs, and the performance gate

Close the epic by making the contract enforceable rather than documented.

Parametrize the conformance suite over `resolve_artifacts_subtabs()` so every pane —
built-ins, the research provider and the synthetic third-party provider from `contract`
— runs the same assertions. The suite must assert that **every contract-declared key
resolves to the action the contract names, on every pane.** `o` was double-booked and
nobody noticed; that is the honest signal about how untested this surface is at the
binding level, and this assertion is what stops the next one.

Surface provider diagnostics in ACE rather than discarding them, so the
`missing_ref_provider` case `sase doctor` already detects is visible where the user is
looking.

Document the contract: the capability vocabulary and its derivation rules, the
declarative `ref.pane` reference for sidecar authors, the visual grammar from `shell`,
and the Patch **contract-in, spec-out** asymmetry so the next reader does not try to fix
it.

Refresh visual snapshots, and hold the performance gate: `SASE_TUI_PERF=1` navigation
p95 under 16 ms on every pane.

## Verification

Every phase runs `just install` first — workspaces are ephemeral and dependencies drift.

`identity` through `declare` touch `tests/ace/tui/test_artifacts_*` broadly, so
`just check`'s scoped lane will escalate. Run `just check-full` through `/sase_monitor`
before landing each of those phases, with a `--next` action so the follow-up agent acts
on the result; it routinely outruns a single agent turn and must never be run inline.
`detail` and `keymap` are narrow enough for `just check` inline.

Expect PNG goldens under `tests/ace/tui/visual/snapshots/png/` to move in `shell`,
`query` and `declare`. Inspect `.pytest_cache/sase-visual/` artifacts and review the
diffs; use `--sase-update-visual-snapshots` only for changes you have actually looked
at.

Measure `SASE_TUI_PERF=1` navigation p95 after each pane conversion, not only at the end
— a regression found in `conform` is a regression with six phases of suspects.

## Risks

- **`query` is the whole plan's risk.** Mitigation is structural: the `boolean=False`
  degradation path makes each pane migration byte-identical and independently
  revertible, `foundation`'s golden fixtures plus the existing parity harness give the
  Rust work a hard oracle, and the phase is sized `xlarge` so its worker plans it as a
  child epic rather than improvising.
- **Two evaluators can drift.** Parameterizing the Rust corpus and the Python reference
  evaluator separately invites divergence. `query_evaluator_parity.rs` must be extended
  in lockstep, not after.
- **Keymap churn is user-visible and unavoidable.** It is severed into its own late
  phase precisely so it cannot block the contract, but it still ships with all three
  mitigations or not at all.
- **Providers gaining a query language enlarges the plugin blast radius.** A malformed
  `properties` block must degrade per entry, never remove a tab. Load-bearing from
  `query` onward.
- **The relation panel is the hot-path risk.** Preserve
  `update_relationships_from_index` rather than rebuilding a graph per selection.
- **Derived capabilities can become opaque.** The named-rule explainer is the mitigation
  and is not optional; a capability whose state cannot be explained is a support burden
  for every sidecar author we never meet.

## Rejected alternatives

**Keep Patch permanently exempt** — the prior report's position, overruled by the owner
and wrong regardless. Cheap now; guarantees two navigation, query, persistence, footer,
help and action-availability systems forever. Every future cross-pane feature either
excludes the most capable pane or is built twice, and the exemption becomes permanent
because each new feature widens the gap. It also removes the best stress test from the
contract.

**Bring Patch in but keep three query languages.** Avoids the biggest refactor; leaves
Patch with a modal editor and no completion and providers with a flat dialect and no
`OR`. "Unified" would be skin-deep.

**Adopt the flat token language everywhere.** Simplest, and a strict downgrade for the
owner's primary surface — loses `OR`, parens, case-sensitive literals and every sigil.

**Adopt the Patch language everywhere as-is.** Keeps the grammar, but its keys are a
hard-coded allowlist and its corpus a hand-written flattener; a provider can extend
neither, and it loses `FilterBar`'s completion and live count.

**Force every pane through the current document/Plan widget.** Optimizes class count
rather than behavior; Patch mutation and graph UX, File versions, Bead hierarchy and
Stitch detail would turn the generic widget into a pile of pane-id conditionals. Shared
shell plus specialized renderers is the smaller abstraction over time.

**Let providers ship Textual widgets or Python callbacks.** Maximum flexibility;
forfeits portability, security, performance guarantees, failure isolation, host keymap
and help consistency, and future-frontend parity — every property the epic exists to
buy.

**Let providers declare their own capabilities.** Rejected in favour of derivation. A
provider could claim a feature its data cannot support, provider config fills with
boolean soup, and — decisively — a new host capability would require a provider release
to reach anyone, which is exactly the outcome this epic exists to prevent.

**Keep provider values as strings.** Recreates today's duplicated matchers and makes
numeric and date sorting inconsistent. Types must survive the wire if the engine is
genuinely shared; deferred to `sase-core` follow-up, not abandoned.

**Save raw query strings without profile identity.** Works until a provider renames a
field or changes a type. Silent reinterpretation is worse than a visibly invalid saved
view; source plus canonical plus profile digest is the minimum durable record.

**Bump the Rust provider-spec wire to schema 2.** Rejected on documented evidence: CI
installs the published floor core, which would reject a v2 spec before the core carrying
the bump ships. Bead `sase-lm` is a live instance. Everything this epic needs from the
spec is additive and optional, so it proceeds at v1.

## Follow-up, explicitly out of scope

- Typed `ArtifactValueWire` in `sase-core`, widening `ArtifactEntryWire.properties`
  beyond `BTreeMap<String, String>` so `datetime` sorting and `string_list` matching are
  expressible rather than coerced on read.
- Retiring the pane-local token parsers and the legacy saved-query readers after a
  measured deprecation window.
- Folding `ace/agent_query` — the Agents tab's self-declared fork — onto the unified
  engine. It is the third dialect and the accelerator for the design, but the Agents tab
  is outside the Artifacts contract and converting it is its own change.
