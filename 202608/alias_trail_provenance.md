---
tier: tale
title: Record the alias resolution trail and its origin at launch
goal:
  Every SASE launch persists the ordered list of model-alias hops it actually traversed
  (`model_alias_trail`) plus how the entry alias entered the launch
  (`model_alias_origin`) into `agent_meta.json` and prompt-step markers, at every write
  site — preview, authoritative reconcile, runner re-exec, workflow step marker, and
  accepted-plan follow-up — so a later phase can answer "which agents ran on this alias,
  and how did they get here?" from stored data instead of re-resolving today's config.
size: medium
proposed_by: bbugyi200.athena.sase-n8.1
bead: sase-n8.1
create_time: 2026-08-16 11:53:10
status: done
---

- **PROMPT:**
  [prompts/202608/alias_trail_provenance.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/alias_trail_provenance.md)
- **PARENT:** [202608/launch_control_alias_history.md](launch_control_alias_history.md)
- **BEAD:**
  [sase-n8.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-n8/sase-n8.1.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-n8.1](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-n8.1.md)
- **COMMITS:**
  - [96b48d0](https://github.com/sase-org/sase/commit/96b48d0abbe9acec0f8037a08c388fc7c291edf8)
    — feat: record alias launch provenance

# Plan: Record the alias resolution trail and its origin at launch

This is phase `provenance` of epic bead `sase-n8` (Agent history for a model alias in
Launch Control). It is the launch-side half: it produces the two new metadata fields
that every later phase of that epic reads. It ships **no** Rust, no wire mirror, no
config field, no adapter, and no TUI.

## Why

`agent_meta.json` records `model_alias` today, but that is only the **first hop** — the
bare alias named by `%model`, or the alias named by the `llm_provider.default_model`
launch setting. Two facts are unrecoverable afterwards:

- **Indirection is lost.** A launch with `%model:@coder`, where `coder` resolves to
  `@large`, records `model_alias: "coder"`. Nothing ties that run to `@large`.
  Re-resolving `@coder` at display time answers "where would this go today", not "where
  did it go".
- **Origin is lost.** A directive-driven `@large` and a no-`%model` launch that landed
  on `@large` write the identical `model_alias: "large"`.

So resolution must return the trail it traversed, `LaunchSelection` must carry that
trail plus the request's origin, and every metadata writer must persist both.

## Pinned data contract (from the epic plan; do not change)

Both fields are additive and optional. Absence is meaningful and must never be
normalized away.

| Field                | Type        | Meaning                                                                                                                          |
| -------------------- | ----------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `model_alias_trail`  | `list[str]` | Ordered bare alias names traversed to reach the concrete target. Index 0 is the entry alias. Omitted when no alias was involved. |
| `model_alias_origin` | `str`       | How the entry alias entered the launch: `directive`, `default_model`, or `none`.                                                 |

Invariants this plan implements:

- A trail records **alias hops only** — never concrete targets, effort suffixes, or
  provider names.
- A trail never repeats a name; the resolver's existing `seen` guards already fail
  closed on cycles, and the trail is accumulated at exactly those guard points.
- A selector (`A | B` / `A || B`) contributes the alias that owns the selector plus any
  alias-valued member that was **actually selected** — never the members that were not.
- A failed resolution (cycle, depth overflow, malformed selector) yields an **empty**
  trail. Attributing partial hops to a failed chain would file a run under an alias it
  never used.
- Origin and trail stay self-consistent: an empty trail always pairs with origin `none`,
  and a non-empty trail never pairs with `none`.
- `model_alias` keeps its current meaning and its current writers. Nothing that reads it
  today changes behavior.

### One deliberate, documented deviation

The epic plan also states `model_alias_trail[0] == model_alias` whenever the trail is
non-empty. That holds for every alias reference SASE itself generates, because
`format_model_directive_value()` always writes a known alias as `@<name>` and
`_directive_extract` only sets `directives.model_alias` when the `@` prefix was present.

It does **not** hold for a hand-typed `%model:large` (a known alias written without
`@`): `directives.model_alias` is `None`, yet resolution really did traverse `@large`.
This plan records the truthful trail in that case rather than dropping a real hop,
because the downstream projection, query, and adapter key off the trail and
`alias_position`, never off `model_alias`. The result is strictly more complete history,
never less. This is recorded as a `PROPOSED FOLLOW-UP:` note on bead `sase-n8.1` so the
epic's land agent and the `core`/`adapter` phases see it.

## Design

### 1. Resolution returns the trail

`src/sase/llm_provider/model_alias_resolution.py`

- Add `alias_trail: tuple[str, ...] = ()` as the **last** field of
  `_ResolvedModelAlias`, so the existing positional constructions keep working.
- `_with_suspended_override()` forwards `result.alias_trail`.
- Add a `trail: tuple[str, ...]` parameter to the inner `resolve()` closure, threaded
  like `seen`: rebound (`trail = trail + (bare,)`) on the linear path, and passed by
  value into every recursive call so a discarded branch discards its hops.
- Append a hop for `bare` at each point the resolver commits to `bare` being an alias.
  Those are the five sites that call `seen.add(bare)` today, plus one that does not:

  | Site                                                                           | Hop |
  | ------------------------------------------------------------------------------ | --- |
  | launch-override redirect (`launch_target is not None`)                         | yes |
  | temporary override paused by a provider disable → underlying configured target | yes |
  | temporary override paused by a provider disable → implicit fallback reference  | yes |
  | configured/implicit target (`target is not None`), including its selector arm  | yes |
  | implicit fallback reference (`fallback_reference is not None`)                 | yes |
  | **applied temporary override** (terminal `f"{provider}/{model}"` return)       | yes |

  The last row is the one with no `seen.add()` today. It must still be a hop: a launch
  through `@large` that a temporary override redirected really did use `@large`, and the
  panel must list it there. Adding it cannot introduce a duplicate, because that branch
  is evaluated before the `target`/`fallback` branches and therefore terminates the
  walk.

- `fail()` keeps its default empty trail; the dangling-reference/concrete terminal
  return carries whatever the walk accumulated.
- Selector arms already recurse per member with a copied `seen`; pass the same `trail`
  into each member and keep only `member_results[index]`'s. Availability probing of the
  losing members therefore cannot leak hops.
- `resolve_model_alias()` still returns a bare string;
  `resolve_model_alias_with_effort()` carries the richer result through unchanged.

`src/sase/llm_provider/registry.py`

- Add `resolve_model_provider_with_trail()` returning
  `(provider_or_none, model, effort, alias_trail)`. Reimplement
  `resolve_model_provider_with_effort()` as a thin wrapper that drops the trail, so the
  two cannot drift and every existing caller and test is untouched.

### 2. The no-`%model` default path threads its trail out too

A no-directive launch resolves through the launch-model-setting helpers, not through
`directives.model`. Without threading, such a launch would record only the setting's
first hop instead of the full `@large → …` chain.

`src/sase/llm_provider/model_launch_settings.py`

- Add `alias_trail: tuple[str, ...] = ()` to `LaunchModelSettingSnapshot` and populate
  it in each branch of `build_launch_model_setting_snapshot()`:
  - a temporary override **on the setting key** (`setting:default_model`) short-circuits
    to a concrete provider/model → empty trail (it already reports
    `referenced_alias=None`);
  - the branch where `referenced_alias` owns a selector → the trail from
    `resolve_model_provider_with_trail(raw_value, …)`;
  - the branch where `raw_value` is itself an inline selector expression → the
    **selected** member's `alias_trail` (the setting is not an alias, so it contributes
    no hop);
  - the plain branch → the trail from `resolve_model_provider_with_trail(raw_value, …)`.
- Add `resolve_default_launch_provider_model_with_trail()` returning
  `(provider, model, effort, alias_trail)`, and reimplement
  `resolve_default_launch_provider_model_with_effort()` on top of it.

`src/sase/llm_provider/temporary_override_defaults.py`

- Add `resolve_effective_default_provider_model_with_trail()` alongside the existing
  `_with_effort` sibling, and reimplement `_with_effort` on top of it. Re-export the new
  name from `sase/llm_provider/temporary_override.py`'s import block and `__all__` next
  to its siblings (`__all__` is keep-sorted).

If `symvision` then reports either `_with_effort` wrapper as unused, delete that wrapper
and its re-exports rather than whitelisting it — the `_with_trail` form is a strict
superset. Do not touch `resolve_effective_default_provider_model` (no `_with_effort`
suffix); the ACE override indicator still consumes it.

### 3. `LaunchSelection` carries the trail and the origin

`src/sase/llm_provider/launch_selection.py`

- Add `alias_trail: tuple[str, ...] = ()` and `alias_origin: str = "none"` to
  `LaunchSelection`.
- Define the three origin values as module constants
  (`ALIAS_ORIGIN_DIRECTIVE = "directive"`,
  `ALIAS_ORIGIN_DEFAULT_MODEL = "default_model"`, `ALIAS_ORIGIN_NONE = "none"`),
  exported through `__all__`, so the persistence sites and tests never spell the
  literals.
- In `resolve_launch_selection()`:
  - the `%model` branch takes its trail from `resolve_model_provider_with_trail()`;
  - the no-`%model` default branch takes its trail from
    `resolve_effective_default_provider_model_with_trail()`;
  - derive the origin **from the resolution outcome**, not from the directive alone: an
    empty trail is always `none`; a non-empty trail is `directive` when it came from
    `directives.model` and `default_model` when it came from the launch setting. That
    keeps the two pinned consistency rules true by construction, including the edge case
    where a caller supplies `provider_name` alongside a `%model` directive and no alias
    resolution happens at all.

### 4. Persistence — every writer

`src/sase/axe/run_agent_directive_metadata.py`

- Add `model_alias_trail: list[str]` and `model_alias_origin: str | None` to
  `AgentMetadataInputs`, immediately after `model_alias`. The dataclass has no defaults
  today; keep it that way and update its single test construction.
- `build_agent_meta()` writes `model_alias_trail` only when non-empty (matching the
  neighboring "omit when falsy" convention) and `model_alias_origin` whenever it is a
  non-empty string — which includes `"none"`, because "recorded, no alias" is a
  different and useful fact from "predates provenance recording".
- `preserved_agent_metadata()` gains both keys. `model_alias_origin` joins the existing
  non-empty-string loop; `model_alias_trail` needs its own guard, preserving only a list
  whose entries are all non-empty strings.

`src/sase/axe/run_agent_directives.py`

- Preserved-metadata branch: reuse the preserved trail/origin **exactly**, with the same
  `isinstance` discipline as the neighboring fields. A re-exec must never recompute —
  that would advance a pooled alias cursor or re-resolve against config that has since
  moved.
- Preview branch: take both from the preview `LaunchSelection`.
- Pass both into `AgentMetadataInputs`.

`src/sase/axe/run_agent_helpers_artifacts.py` and its
`src/sase/axe/run_agent_helpers.py` facade

- Give `update_meta_fields()` an optional keyword-only `remove_keys: Sequence[str] = ()`
  that pops those keys in the same read/modify/atomic-write pass. The authoritative
  reconcile needs to be able to _clear_ a stale trail written by the preview, and today
  the helper can only set. Existing callers are unaffected.

`src/sase/xprompt/workflow_executor_steps_prompt.py` — the reconciliation point of
record

- Pass `model_alias_trail=list(launch_selection.alias_trail)` and
  `model_alias_origin=launch_selection.alias_origin` into the prompt-step marker write.
- In the anonymous-workflow branch, add `model_alias_origin` to `root_meta_fields`
  unconditionally, add `model_alias_trail` when the trail is non-empty, and pass
  `remove_keys=("model_alias_trail",)` when it is empty, so the stored trail always
  describes the selection that actually answered.

`src/sase/xprompt/workflow_executor.py`

- `_save_prompt_step_marker()` gains `model_alias_trail: list[str] | None = None` and
  `model_alias_origin: str | None = None`, preserved from an existing marker exactly as
  `model_alias` already is (the trail needs a `list` type check when reading the
  existing marker).
- The marker always writes every key, so write `list(model_alias_trail or ())` — never
  `null` — because the phase-`core` `PromptStepMarkerWire` field is a defaulted
  `Vec<String>` that would reject a JSON `null`. `model_alias_origin` is an
  `Option<String>` there and may be `null`.
- Mirror the two new parameters on the `_save_prompt_step_marker` stub in
  `src/sase/xprompt/workflow_executor_steps_script.py` so the mixin signatures stay
  identical.

`src/sase/axe/run_agent_exec_plan_accept.py` and
`src/sase/axe/run_agent_exec_plan_accept_models.py`

- `FollowupModel` gains `model_alias_trail: tuple[str, ...] = ()` and
  `model_alias_origin: str | None = None` (defaults keep the existing keyword
  constructions and their tests valid).
- Add one helper that resolves a `%model` directive value to `(trail, origin)` for a
  follow-up: split the effort suffix, resolve non-consuming through
  `resolve_model_alias_with_effort()`, use an empty trail when the resolution is
  invalid, and report `directive` for a non-empty trail and `none` otherwise. Follow-ups
  always carry an explicit `%model` prefix, so `default_model` is unreachable here.
- Populate the new fields at all four construction sites:
  `resolve_tale_size_followup()`, both branches of `resolve_followup_model()`, and the
  custom-coder-prompt branch in `run_agent_exec_plan_accept.py`.
- `_write_followup_model_alias_meta()` takes the trail and origin alongside the alias
  and sets or **pops** all three together, so a stale trail can never outlive the alias
  it described.

Deliberately **not** changed: `create_followup_artifacts()`'s inherited-key allowlist.
It does not inherit `model_alias` today, so it must not inherit the trail or origin
either — the follow-up's own authoritative reconcile is what fills them in.

## Verification

New tests, placed to keep files small (`toobig`: 1000-line hard cap).

**`tests/llm_provider/test_model_alias_trail.py` (new)** — resolver unit tests,
following the `mock_provider_config` / `@patch("…config.get_llm_provider_config")`
pattern already used by `tests/llm_provider/test_config_alias_resolution.py`:

- a direct alias → one hop;
- a two-hop chain and a three-hop chain, in order;
- a round-robin selector whose selected member is itself an alias → owner hop + member
  hop, and no losing member's hops;
- an ordered-fallback selector;
- a temporary override short-circuit → the overridden alias is still hop 0;
- an override paused by a provider disable that falls back through the underlying
  target;
- a launch alias override (`%m(@medium, medium=…)`);
- a concrete `%model:opus` → empty trail;
- a cycle → empty trail;
- a depth-limit overflow → empty trail.

**`tests/llm_provider/test_launch_selection_alias_origin.py` (new)** — `directive` for a
`%model:@alias` launch, `default_model` for a no-directive launch (with the full setting
chain in the trail, not just its first hop), `none` + empty trail for a concrete
`%model`, and the trail/origin consistency invariant asserted on all three.

**`tests/test_reasoning_effort_metadata_persistence.py` (extend)** — end-to-end through
`extract_directives_and_write_meta`: the two fields land in `agent_meta.json` for an
alias launch, the no-`%model` default launch records origin `default_model`, a concrete
model launch omits the trail and records origin `none`, and a re-exec preserves both
without re-resolving (extending the existing preserved-selection test rather than
duplicating its fixture). Also extend the existing step-marker test so the marker stores
and preserves both fields across a rewrite.

**`tests/test_run_agent_directive_metadata.py` (extend)** — `preserved_agent_metadata()`
keeps a well-formed trail and origin and rejects a malformed trail (non-list, or a list
containing a non-string); update the one `AgentMetadataInputs(...)` construction.

**`tests/test_pooled_alias_single_consumption.py` (extend)** — the composed
preview-then-real-invocation harness already in that file asserts that the reconcile
writes the trail and origin that the _consuming_ resolution produced, into both
`agent_meta.json` and `prompt_step_main.json`.

**`tests/test_axe_run_agent_exec_plan_followup_metadata.py` (extend)** — a follow-up
routed through a size alias records trail + origin `directive`; a concrete follow-up
model pops all three fields, including a trail inherited in the file's existing
stale-planner-metadata fixture.

Then, from this workspace:

```bash
just install
just check
```

`just check` is the required gate after file changes here (whole-repo lint gates —
including `symvision`, `toobig`, `keep-sorted`, `ruff`, and `mypy` — plus the
diff-scoped test lane). Hand it to `/sase_monitor` with a `--next` action if it runs
long. Escalate to `just check-full` through `/sase_monitor` if the scoped run reports an
unusual selection or escalates.

## Out of scope

- Anything in the `sase-core` repo: the wire fields, the `agent_artifact_model_aliases`
  projection, schema 22, its migration, and `query_agent_alias_history` are phase
  `core`.
- The Python wire mirror, the facade call, and the `tools/validate_sase_core_rs` skew
  probe (phase `wire`).
- `llm_provider.model_alias_history_limit` (phase `config`).
- The adapter, the panel, its `H` keymap, PNG goldens, and the `sase-core-rs` floor
  bump.
- **Documentation.** `docs/llms.md`'s alias section and `docs/ace.md` are edited by
  phases `config` and `panel`; adding a paragraph there now would only manufacture a
  merge conflict when the epic's trees are combined. The two new fields are documented
  as provenance labels by phase `panel`.
