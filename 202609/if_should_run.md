---
tier: tale
size: medium
title: Omit conditional swarm agents with %if(should_run=...)
goal:
  Add a boolean %if mode that removes disabled prompt blocks before launch planning, and
  make research_swarm infographic generation opt-in.
proposed_by: bbugyi200.athena.0lm
create_time: 2026-09-15 19:10:39
status: wip
---

# Conditional swarm agents

## Scope and approach

Implement this as one medium tale across `sase-core`, `sase`, and
`sase-research-artifacts`. One coding agent can make and verify the coherent change; the
work does not need independently shipped epic phases. Open both linked repositories with
`/sase_repo` and use their returned paths. Paths below are relative to the named
repository, never to the planning agent's checkout.

The user-facing contract is:

- `%if(should_run=false)` removes its entire prompt segment from expansion and launch.
- `%if(should_run=true)` includes the segment and removes only the directive.
- The boolean keyword and a Python/Bash condition body are mutually exclusive, with an
  actionable error even when the boolean is false.
- `#research_swarm` gains `should_generate_image`, a `bool` input defaulting to `false`.
  Its existing image segment starts with `%if(should_run={{ should_generate_image }})`.
  Default and explicit-false calls launch the two researchers and lead; explicit true
  also launches the existing image agent after the lead.

This is an expansion-time inclusion decision, not an admission predicate. Implement the
shared grammar and filtering policy in Rust core and expose thin Python adapters.

## Findings that constrain the implementation

1. SASE currently accepts script conditions as `%if::` followed by one fenced Bash or
   Python body. Parenthesized positional `%if(...)` scripts are currently rejected.
   Preserve that supported syntax; this task does not add another script syntax.
2. The current script form always contributes a logical planned unit, but a false
   predicate prevents actual dispatch. Preserve its existing predicate execution, wait
   ordering, skipped-state records, and feature gate. Do not change it to always start a
   model process. The requested distinction is that a false boolean contributes no
   planned unit at all.
3. `typed_launch_units` defaults off. The new boolean mode must work in both flag states
   so installing the updated research plugin does not require enabling an unrelated
   beta. Keep script `%if`, `%proc`, and their code snippets gated. No new feature flag
   is needed for this complete additive change and permanent user-selectable input.
4. Swarm rendering currently calls `expand_single_xprompt`, which substitutes Jinja and
   expands local helpers, then splits segments and qualifies name markers. Swarm
   recursion and caller-prefix attachment happen afterward. Filtering only in agent
   dispatch or the final Rust typed-unit classifier would be too late.
5. `plan_typed_launch_units` in the Python facade currently resolves keyed agent names
   before invoking Rust. Direct-launch code also resolves keys and reserves clan/name
   state after expansion. Both must receive the filtered batch.
6. The research plugin stores four authored segments. Existing plugin loading, wheel,
   and SASE fakey tests often assume four _rendered_ agents and require updates.

## Detailed behavior

### Grammar and diagnostics

Accept a single named `should_run` argument in the parenthesized form, using the
existing argument tokenizer and quoting rules. After normal argument decoding and
whitespace trimming, accept case-insensitive `true` and `false`; this includes Jinja's
rendering of typed booleans as `True` and `False`. Do not interpret arbitrary nonempty
strings as true. Reject empty values, numbers, `null`, misspellings, and unrendered
template expressions at the final evaluation boundary.

Keep one active `%if` per segment. Diagnose duplicate keywords, duplicate `%if`
directives, unknown keywords, malformed parentheses, and mixed forms before dropping the
segment. Validate the entire expanded batch before dispatching any survivor, so a bad
condition in a later segment cannot produce a partially launched swarm. In particular,
both `%if("script", should_run=false)` and `%if(should_run=false)::` followed by a fence
must explain that `should_run=` cannot be combined with a Python/Bash condition body. A
boolean occurrence plus a separate `%if::` in the same segment must also fail clearly,
rather than silently dropping it.

Use the shared directive/literal scanners: occurrences inside inline code, fenced code,
directive-owned code bodies, and xprompt-disabled regions are literal. Support normal
directive placement, whitespace, multiline argument lists, CRLF, and Unicode offsets.
During template expansion, allow the unresolved boolean header to reach Jinja; neither
evaluate it early nor hide its `{{ ... }}` expression as an opaque code body.

### Omission semantics and ordering

Use literal deletion of the rendered segment and its separator as the behavioral test
oracle. After rendering an xprompt's own inputs, inspect its segments and drop false
ones before recursive helper/swarm expansion, name-key qualification/allocation,
fanout/repeat planning, dependency binding, project inference, approval preview/digest
creation, and any launch side effects. A false outer segment containing a nested swarm
must remove that entire subtree. Do not evaluate code conditions to decide inclusion.

The xprompt's input validation and ordinary Jinja rendering still occur to obtain the
boolean. After that decision, a dropped segment must not expand its nested xprompts, run
nested argument command substitutions, validate unrelated launch directives, reserve
identities, create agent/proc/admission rows, consume queue capacity, select a workspace
or remote target, or contribute a wait. Validate its own `%if` grammar and mode
exclusivity even when false.

Apply the same Rust policy to already-rendered literal multi-prompts and direct core
planning callers, not just the research template. Retain an explicit empty-batch result:
a false sole segment or an all-disabled swarm is a successful no-op with zero launches,
never a fallback blank agent. A true directive with no remaining prompt content must not
manufacture a blank segment.

For surviving segments:

- Preserve order and align each segment with its template group, swarm provenance, extra
  environment, and one-shot first-slot metadata.
- In sole/embedded swarms, caller directives and surrounding prose attach to the first
  surviving segment. In multiple embedded invocations, leading prose attaches to the
  first emitted segment even if the first invocation emits nothing. Preserve inherited
  VCS references. For a wholly empty embedded expansion, preserve substantive caller
  prose using the existing empty-expansion behavior without launching directive-only or
  workspace-reference-only remnants.
- Bind bare `%wait` to the preceding surviving unit and generate contiguous logical IDs
  from the surviving batch. Named waits and explicit logical-unit references keep their
  ordinary meaning, exactly as if the removed block had never been authored; do not
  invent a completed/skipped target or silently rewrite a named dependency.
- Do not transfer declarations, dependencies, or fork references from a removed block.
  Surviving references to a removed declaration receive the same existing resolution
  behavior as a manually deleted declaration.
- Retain repeat/alternative behavior within included segments. A false segment creates
  none of its repeats or alternatives; a boolean introduced by an expanded branch must
  be resolved before that branch receives identity or dependency state.

## Implementation steps

### 1. Rust grammar, filtering, and binding

In `sase-core`, add a focused shared parser/filter module alongside
`crates/sase_core/src/agent_launch/mod.rs`, reusing its argument and literal handling
and `crates/sase_core/src/fenced_code.rs` where appropriate. Avoid growing another
Python implementation of the grammar.

Expose a pure, versioned result through `crates/sase_core_py/src/lib.rs` that identifies
kept source segments, cleaned prompt text, and actionable diagnostics. Preserve source
indexes/spans so Python can retain parallel metadata and report which block failed. Keep
this static inclusion result separate from `LaunchConditionWire`; no synthetic script or
admission record should represent a boolean.

Use the same policy at the common fanout/typed-plan boundary before allocating slots or
validating the survivor graph. Review `plan_agent_launch_fanout`'s `auto` fallback,
which currently manufactures a single slot for otherwise empty input. Ensure filtered
empty input remains empty for every relevant launch kind. Keep existing script-mode
wire/admission semantics intact. Add PyO3 binding tests as well as pure Rust tests.

### 2. Python expansion and launch integration

Add a thin typed adapter for the Rust result. Wire it into:

- `src/sase/xprompt/processor.py`: perform inclusion after own-input substitution and
  before local-helper expansion, preserve segment separators, and recheck newly expanded
  segments before they enter launch planning. Keep normal prompt-part expansion's
  first-surviving-segment behavior consistent.
- `src/sase/agent/_xprompt_swarm_rendering.py` and `xprompt_swarm.py`: recursively
  propagate survivors, handle zero/one/many results, and attach caller context to the
  correct survivor. Filter raw segments before recursion as well as template output.
- `src/sase/agent/launch_request_planning.py`, `launch_cwd_agents.py`, and
  `src/sase/core/agent_launch_facade.py`: ensure direct, typed, preview/approval, and
  already-expanded launch-unit entry points all filter before key allocation, project
  selection, hard-disabled checks, or clan/name prepasses. Preserve metadata alignment
  and zero-launch results. Already-cleaned prompts must pass through unchanged.
- `src/sase/xprompt/code_value.py`, `_directive_collect.py`, `_directive_extract.py`,
  and `_directive_scan.py`: distinguish static boolean syntax from gated code syntax,
  share Rust diagnostics, strip an included boolean from model-facing text, and ensure
  boolean-only prompts do not trigger typed admission. Do not simply remove the flag
  check for every `%if` form. Update static launch-directive extraction/serialization
  only where the existing call graph needs the parsed inclusion result.

Review existing callers rather than adding a second launch route. Make sure the original
submitted prompt's typed-directive guard does not reject a boolean already consumed by
expansion. No predicate subprocess should execute for either boolean value.

### 3. Completion and documentation

Update the shared Rust editor metadata/completion contract in
`crates/sase_core/src/editor/directive.rs` and related feature-flag/snippet helpers.
Offer `%if(should_run=...)`, `should_run=`, and `true`/`false` without the beta flag;
offer code-form recipes only with the flag enabled. Exercise the shared Python/TUI
consumer; a separate editor-plugin implementation should not be necessary.

Update `docs/xprompt.md`'s directive matrix, completion matrix, and conditional-launch
description, plus the existing flag-summary claims in `docs/configuration.md` and
`docs/editor.md` that currently say all `%if` forms are gated. Explain static omission,
script admission, strict booleans, mutual exclusion, bare-wait rebinding, and empty
swarms with short examples. Do not change memory files as part of this tale.

### 4. Research plugin opt-in image generation

In `sase-research-artifacts`, append this input after the existing inputs in
`src/sase_research_artifacts/xprompts/research_swarm.md` to preserve positional
ordering:

```yaml
- name: should_generate_image
  type: bool
  default: false
  description: Generate an infographic after the lead researcher finishes.
```

Place `%if(should_run={{ should_generate_image }})` at the start of the existing image
segment. Retain its image model, clan membership, wait on `.final`, quarter-weight queue
settings, `#fork`, and `#research/image`. Update the xprompt description and README to
say three agents by default and four when enabled, and show
`#research_swarm(prompt="A research topic", should_generate_image=true)`.

Update `tests/test_xprompt_loading.py` and `tests/test_wheel_contract.py`: distinguish
four authored blocks from three default expanded agents; use explicit true in tests
whose subject requires the image segment. Cover omitted, false, and true inputs through
the actual public expansion and planning paths. Keep model overrides, external waits,
queue capacity/priority, and raw-protected runtime `wait.artifacts` templates intact.

Update SASE's `tests/fakey/test_runner_slots_e2e.py`: assert the default three-agent
graph, and explicitly opt into image generation for the four-quarter-weight capacity
test so it continues proving a total capacity load of one.

## Verification and acceptance

Add focused behavioral coverage, principally in Rust unit/binding tests and these
existing SASE suites: `tests/test_typed_launch_units_code_contract.py`,
`tests/test_xprompt_swarm_expansion.py`,
`tests/test_multi_prompt_launcher_xprompt_groups.py`, and typed-plan/admission tests.

Required cases:

1. True/false including Jinja booleans; invalid, unknown, repeated, and mixed inputs;
   clear mode-exclusivity errors with both true and false; literal-zone protection; no
   partial dispatch when a later segment contains an invalid condition.
2. False first/middle/last/sole/all segments; true retaining content; nested disabled
   subtrees; multiple embedded invocations; non-ASCII and CRLF. Compare the resulting
   graph to a manually deleted-block control with deterministic name allocation.
3. Bare waits, explicit waits, repeats/alternatives, caller prefix/prose/VCS
   inheritance, and per-segment metadata stay consistent with literal deletion. Include
   an all-disabled `auto` plan and a direct-launch no-op to prevent blank fallback
   agents.
4. A disabled segment containing a nested expansion with an observable callback never
   invokes it. Launch spies show no name reservation, workspace claim, dispatch,
   admission/predicate execution, or queue entry for the removed image agent.
5. Both flag states accept boolean mode. Flag-off still rejects script `%if` and
   `%proc`; flag-on preserves script-condition units and current false-predicate
   skipped-state behavior. Completion exposes only the forms allowed in each state.
6. Installed plugin: omitted/false produce exactly `.cdx`, `.cld`, `.final` with the two
   lead dependencies; true additionally produces `.image` waiting on `.final` and
   retaining its fork/model/queue directives. No phantom image unit appears in preview,
   approval payload, or dispatch. Invalid boolean input reports a useful argument error.

Rebuild/install the modified local Rust binding into the implementation checkout's
environment using SASE's `just rust-install` with the opened core path, and verify the
plugin against that same local SASE/core combination. Confirm tests import the opened
plugin source when doing cross-repository integration; do not silently test an older
installed plugin. Run:

- Core `just check` (includes workspace and PyO3 tests; core-only cargo tests are not
  sufficient).
- SASE targeted tests above, then `just check` as required by `lint_and_test.md`.
- Plugin `just check` and `just test-wheel`, supplying the new local dependencies using
  the repository's existing wheel-test/install mechanism. Set
  `SASE_RESEARCH_ARTIFACTS_SASE_SOURCE_DIR` to the implementation agent's SASE checkout
  and `SASE_RESEARCH_ARTIFACTS_SASE_CORE_SOURCE_DIR` to its opened core checkout; the
  plugin Justfile propagates these sources into the fresh wheel-test environment.

Use `/sase_monitor` for long checks and for any required `just check-full` before
landing. Do not launch real research or image-generation agents to test this feature.
Keep release-plz-owned Cargo versions unchanged. Record the dependency order as core,
SASE, then research plugin; ensure published consumer dependency floors require the
releases that contain this support when those release versions are known, without
inventing versions or bypassing the existing release workflow.

The work is complete when grammar, expansion, both launch routes, completion, plugin
packaging, and the default/opt-in acceptance cases pass, with script-based admission
behavior preserved. Use the host-owned final declaration for all repositories changed by
implementation; do not manually commit, branch, or publish.
