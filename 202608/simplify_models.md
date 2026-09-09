---
tier: epic
title: Simplify built-in model routing and redesign the Models panel
goal:
  SASE exposes only five size aliases, routes launch roles through explicit config
  fields, and presents every model-related setting in one clear and polished panel
phases:
  - id: core_model_routes
    title: Define shared size and epic-land model routing primitives
    depends_on: []
    size: medium
    description:
      "core_model_routes: add and bind the provider-agnostic Rust contract for size
      alias selection and explicit/configured epic-land model precedence."
  - id: alias_config_contract
    title: Replace legacy role aliases with the compact config contract
    depends_on:
      - core_model_routes
    size: medium
    description:
      "alias_config_contract: reduce built-ins to five direct size aliases, add the
      three model config fields, migrate runtime routing and state, and diagnose retired
      names."
  - id: models_panel_redesign
    title: Redesign Models around launch settings and flat size aliases
    depends_on:
      - alias_config_contract
    size: medium
    description:
      "models_panel_redesign: build a unified navigable panel for model settings,
      effort, runner limit, five built-ins, and responsive user-owned aliases and
      buckets."
  - id: migration_docs_and_verification
    title: Complete migration coverage, documentation, and end-to-end verification
    depends_on:
      - alias_config_contract
      - models_panel_redesign
    size: medium
    description:
      "migration_docs_and_verification: sweep public surfaces for retired aliases,
      document the new contract, update intentional goldens, and run exhaustive
      verification."
proposed_by: bbugyi200.athena.02n
status: done
bead_id: sase-mf
create_time: 2026-09-09 19:51:32
---

- **PROMPT:**
  [prompts/202608/simplify_models.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/simplify_models.md)
- **BEAD:**
  [sase-mf](https://github.com/sase-org/sase--beads/blob/main/pages/sase-mf/README.md)

# Simplify Built-in Models and Redesign the Models Panel

## Outcome

SASE will have one small, legible built-in model vocabulary:

| Built-in alias | Direct shipped target                                                                               |
| -------------- | --------------------------------------------------------------------------------------------------- |
| `@xsmall`      | `claude/sonnet@medium \| codex/gpt-5.5@medium \| grok/grok-4.6@medium \| agy/gemini-3.7-flash-high` |
| `@small`       | `claude/sonnet@high \| codex/gpt-5.5@high \| grok/grok-4.6@high`                                    |
| `@medium`      | `codex/gpt-5.5@xhigh \| claude/sonnet@xhigh \| grok/grok-4.6@xhigh`                                 |
| `@large`       | `claude/opus@xhigh \| codex/gpt-5.6-sol@xhigh`                                                      |
| `@xlarge`      | `claude/opus@max \|\| codex/gpt-5.6-sol@max \|\| grok/grok-4.6@max`                                 |

These are concrete selector owners, not fallbacks to a second layer of `smart*`/`cheap*`
aliases. The `@large` member order above follows the explicit example in the request;
retain it as an observable part of deterministic fresh-pool selection. Users may
override these five built-ins under `llm_provider.model_aliases.builtin`, and may
continue defining arbitrary described aliases and display buckets under `custom` and
`buckets`.

Remove every other implicit model alias: `@default`, `@epic_lander`, `@big_epic_lander`,
all five `@<size>_worker` names, and `@smart`, `@smarter`, `@smartest`, `@cheap`,
`@cheaper`, and `@cheapest`. Remove the automatic `worker` built-in bucket as well. A
user-created alias or bucket named `worker`, `smart`, or any other now-unreserved name
remains valid custom configuration; it gains no special behavior.

## Configuration and Routing Contract

Add these scalar fields under `llm_provider`, using the same model expression grammar as
alias targets (concrete models, provider-qualified models, alias references, effort
suffixes, round-robin pools, and ordered fallbacks):

| Field                   | Shipped default | Purpose                                       |
| ----------------------- | --------------- | --------------------------------------------- |
| `default_model`         | `@large`        | A launch with no explicit `%model`            |
| `epic_lander_model`     | `@large`        | An epic below `bead.big_epic_phase_threshold` |
| `big_epic_lander_model` | `@xlarge`       | An epic at or above that threshold            |

This intentionally restores `llm_provider.default_model` as a supported field with new,
explicit semantics. The other two settings live beside it because they are model routing
choices; `bead.big_epic_phase_threshold` remains the independent threshold. All
accessors must validate defensively and fall back to these shipped values if a caller
somehow bypasses schema/doctor validation.

Routing precedence must be unambiguous:

1. An explicit prompt, plan, phase, task, or approval-picker model wins.
2. A currently active temporary override of the selected launch setting wins.
3. Otherwise the selected merged config field is resolved through the normal alias,
   effort, selector, provider-disable, and launch-family override machinery.
4. Missing or malformed config fails safely to the shipped field default.

Size-derived tale, task, and epic-phase launches select `@xsmall`, `@small`, `@medium`,
`@large`, or `@xlarge` through the shared core mapping. Epic landing selects one of the
two config fields through the existing authored-phase threshold, including closed phases
on resume. No launch should synthesize one of the removed public aliases.

For metadata, previews, and logs, record the actual referenced alias when the chosen
field value is an alias (for example `large` for the shipped default), and no alias for
a concrete target. Do not preserve a fictitious `default`, `epic_lander`, or
`big_epic_lander` alias merely as provenance. Internal temporary-setting keys must be
stable and namespaced from public aliases so a user-defined `@default` cannot collide
with the default-launch setting.

## Migration and Compatibility

This is a deliberate public simplification, not a silent compatibility-alias layer.
Completions, model pickers, directive validation, generated catalogs, and the Models
panel must expose only the five built-ins plus configured custom aliases.

`sase doctor -C config.model_aliases` and inline directive diagnostics must identify
retired config keys and references with exact destinations:

- `model_aliases.builtin.<size>_worker` becomes `model_aliases.builtin.<size>`.
- `model_aliases.builtin.default` becomes `llm_provider.default_model`.
- `model_aliases.builtin.epic_lander` becomes `llm_provider.epic_lander_model`.
- `model_aliases.builtin.big_epic_lander` becomes `llm_provider.big_epic_lander_model`.
- `cheaper`, `cheap`, `smart`, `smarter`, and `smartest` map conceptually to `xsmall`,
  `small`, `medium`, `large`, and `xlarge`; diagnostics must explain how to preserve a
  customized target rather than blindly replacing a reference.
- `cheapest` has no automatic replacement; users who still want it should define a
  described custom alias.
- Built-in `worker` bucket metadata without custom `worker` members becomes orphaned
  metadata and receives the same actionable warning as any empty custom bucket.

Do not auto-edit user config. Do migrate ephemeral machine state on read/write so an
upgrade does not unexpectedly discard a live choice:

- Map temporary setting overrides from `default`, `epic_lander`, and `big_epic_lander`
  to namespaced setting keys.
- Map direct `<size>_worker` overrides to `<size>`; when both a direct worker override
  and its old `cheap*`/`smart*` fallback override exist, the direct worker value wins.
- Migrate compatible round-robin cursor entries from `cheaper`, `cheap`, `smart`, and
  `smarter` (and configured `<size>_worker` selector owners) to their new size owner
  when the selector fingerprint still matches. Collision handling is deterministic,
  idempotent, lock-safe, and covered by tests.
- Drop or ignore retired state that has no meaningful route, without allowing corrupt
  state to block a launch.

Keep the reserved `@default` **agent panel/tribe** concept untouched. Only the model
alias named `default` is retired.

## Models Panel Design

Replace the title's two-line settings summary and the oversized built-in alias catalog
with one navigable hierarchy:

```text
 Models
 ── Launch settings ─────────────────────────────────────────────
 launch model        @large   → CLAUDE(opus) @ xhigh   shipped/configured
 epic lander         @large   → CLAUDE(opus) @ xhigh   below 5 phases
 big epic lander     @xlarge  → CLAUDE(opus) @ max     5+ phases
 default effort      provider default
 running agents      10
 ── Built-in size aliases ───────────────────────────────────────
 @xsmall  ...
 @small   ...
 @medium  ...
 @large   ...
 @xlarge  ...
 ── Your aliases ────────────────────────────────────────────────
 ▸ research   3 aliases
 @fast        ...
```

The exact typography may adapt to Textual constraints, but preserve this information
hierarchy and vocabulary:

- Launch behavior appears first and shows both each raw configured value and its
  effective provider/model/effort. The big-epic row includes the current threshold.
- Default effort and max running agents are first-class rows in the same section, with
  configured/effective/temporary provenance rather than special text in the title.
- The five built-ins are top-level, in size order, and never folded into a built-in
  bucket.
- User buckets retain drill-in navigation and may mix only user aliases because there
  are no built-in buckets. Empty custom configuration gets a concise configuration hint
  without dominating the panel.
- Use a quiet cyan section structure, warm custom-ownership accent, provider-aware model
  colors, purple temporary-state accents, aligned columns, and stable row height.
  Ellipsize secondary details before hiding the setting or alias identity.

Use one predictable action vocabulary. Model setting rows and alias rows support
temporary override/clear plus persistent edit/reset using the existing picker, selector
builder, preview, surgical config edit, chezmoi, and commit-offer workflows. Effort and
runner rows retain their specialized value cards but participate in the same row
selection and context-sensitive footer; keep existing `ctrl+e` and `ctrl+r` shortcuts as
compatibility accelerators. Bucket rows expose only open/back/navigation actions. Enter
performs the primary action for the selected row.

Build one immutable, display-ready snapshot off the event loop containing launch
settings, effective resolutions, effort, runner limit, aliases, buckets, provider
disable state, and temporary overrides. Rendering, highlight changes, and the five
second countdown tick must remain disk-free. Preserve the programmatic-highlight guard,
selection by stable row ID across refreshes, worker cancellation on unmount, and the
existing background edit/override patterns.

## Phase Details

### 1. `core_model_routes`: shared routing primitives

Open the linked `sase-core` repository through `/sase_repo`. Add a small pure domain
module and PyO3 bindings that:

- map every `PhaseSizeWire` to the canonical bare/public `@<size>` alias;
- select an explicit epic land model or the normal/big configured target from phase
  count and threshold;
- reject invalid sizes/counts/thresholds at the binding boundary rather than letting
  Python and future frontends diverge;
- return simple wire values with no provider, filesystem, or Textual dependency.

Add Rust unit tests, binding tests, and Python-facing parity probes. Update only current
canonical fixtures whose expected produced alias changes; retain deliberately historical
archive fixtures when they are testing old metadata readability. Run the linked
repository's `just check` before handing off.

### 2. `alias_config_contract`: alias policy, config, routing, and state

In the SASE repository:

- consume the new Rust routing functions from the bead/tale/task/plan paths instead of
  maintaining Python size maps;
- reduce `model_alias_defaults.yml` and its parser/constants to five direct targets and
  descriptions, deleting implicit-fallback machinery that no longer has a caller;
- add typed defensive config accessors and schema/default-config entries for the three
  fields, and centralize resolution into a display/runtime snapshot that reports raw
  value, effective provider/model/effort, selector details, and provenance;
- route no-`%model` launches and epic land segments through these fields while
  preserving explicit model, effort, provider-disable, selector-consumption, retry,
  resume, and launch-family override precedence;
- namespace and migrate temporary override and load-balancing state as described above,
  including old state-version fixtures, collisions, corrupt input, locks, and idempotent
  rewrites;
- restrict `model_aliases.builtin` semantically to the five size names while preserving
  arbitrary described custom aliases and custom buckets;
- replace retired-name validation/doctor advice, and remove implicit names from model
  completions and pickers.

Cover default/explicit/temporary/configured routing, selectors, effort overlays,
disabled providers, metadata, preview-versus-consuming selection, family overrides, epic
threshold/resume behavior, schema validation, config edit paths, and migrations with
focused tests. Run `just install` before the SASE checks, then `just check`.

### 3. `models_panel_redesign`: unified visual management surface

Introduce explicit row/view types for model launch settings, scalar settings, aliases,
and user buckets. Rework the panel composition and rendering around the hierarchy and
interaction contract above. Reuse the established model picker, effort picker, selector
builder, duration/until flow, config edit preview, commit offer, and provider styles
rather than duplicating their semantics.

Persistent model-setting writes target the new scalar paths; reset removes a user
override and reveals the shipped default. Temporary model-setting overrides use the
namespaced state keys but render human labels, never fake `@` aliases. Ensure successful
writes refresh the panel and agent/top-bar summaries through the existing fast path, and
ensure failed/cancelled writes preserve selection and state.

Update navigation, rendering, edit/override outcome, warning, responsive-layout, and
top-bar indicator tests. Add visual fixtures and PNG snapshots for at least:

- shipped defaults with all five launch-setting rows visible;
- configured and temporary model-setting provenance;
- five flat built-ins and an empty user section;
- custom bucket collapsed and drilled in;
- mixed custom aliases with active/suspended selector states;
- effort and runner-limit temporary states;
- a narrow viewport proving containment, readable identities, and stable footer/layout.

Run the focused Models-panel suite, `just test-visual` (updating goldens only for the
intentional redesign), and `just check`.

### 4. `migration_docs_and_verification`: public sweep and integration proof

Search all source, tests, demos, templates, and docs for the retired model names and
classify every hit. Update public examples and explanations in the LLM, ACE, beads, SDD,
xprompt, agent-family/provider, and configuration documentation; update the generated
alias table renderer and run `just fmt-docs`. Update the install-time
`memory-sase-sizes` source template and its tests to teach `@<size>` routing.

Do **not** edit canonical files under `sase/memory/`, generated `AGENTS.md`, or provider
instruction shims without explicit user authorization in the implementing conversation.
If that authorization is present, update `sase/memory/sase_sizes.md` and run
`sase memory init` as required. Otherwise, leave those protected files untouched and
surface the stale canonical memory as a permission-gated follow-up; approval of this
plan alone is not permission to edit memory.

Add end-to-end tests proving:

- only the five built-ins appear in completions, catalogs, pickers, panel rows, and
  accepted `%model` values;
- custom aliases and custom buckets still resolve, validate, render, edit, and override;
- every shipped config field displays its raw and effective current value in `,m`;
- no-model launches, normal epic landers, big epic landers, tales, phases, and tasks
  choose the documented routes and metadata;
- stale config/directives receive actionable migration diagnostics rather than silently
  resolving or falling through;
- unrelated reserved `@default` agent-panel behavior is unchanged.

Run `just install`, `just check`, and the complete visual suite. Because the work
changes broad model-routing behavior and combines multiple phase trees, run
`just check-full` only through `/sase_monitor` with a follow-up action, as required by
the project verification policy. Re-run the linked `sase-core` `just check` against the
final binding consumer if integration changed its API.

## Acceptance Criteria

- The implicit alias catalog contains exactly `xsmall`, `small`, `medium`, `large`, and
  `xlarge`, with the direct targets listed above and no built-in buckets.
- The three new config fields default to `@large`, `@large`, and `@xlarge`; runtime,
  preview, metadata, doctor, documentation, and panel agree on their meaning.
- User-defined aliases and buckets remain fully supported, including alias references,
  effort suffixes, pools/fallbacks, persistent edits, and temporary overrides.
- The Models panel makes launch behavior understandable at a glance, remains fully
  keyboard-driven, fits narrow terminals, and has intentional pixel snapshots.
- Upgrade diagnostics and ephemeral-state migration preserve user intent without
  retaining hidden compatibility aliases.
- Focused tests, visual tests, `just check`, monitored `just check-full`, and the linked
  Rust core gate all pass.
