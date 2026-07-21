---
tier: tale
title: Load-balanced model aliases and the builtin cheapest alias
goal: Alias values can name a pipe-separated model pool that round-robins across launches
  and skips uninstalled providers, a new builtin @cheapest alias uses that mechanism,
  and small/large phase workers route through @cheapest/@smartest.
create_time: 2026-07-21 09:11:51
status: done
prompt: 202607/prompts/cheapest_load_balanced_alias.md
---

# Plan: Load-balanced model aliases and the builtin `@cheapest` alias

## Context and outcome

SASE model aliases (`src/sase/llm_provider/config.py`) resolve to exactly one target today. Introduce _load-balanced
alias values_: an alias value may list several targets separated by `|` (for example
`claude/opus@medium | codex/gpt-5.5`). Each real agent launch that resolves through such a value takes the next member
of the pool (round-robin), members whose LLM provider CLI is not installed are skipped, and a trailing `@<effort>`
suffix on the selected member threads through as the launch's reasoning effort. On top of that mechanism, add a new
implicit builtin `@cheapest` alias whose default value is `claude/opus@medium | codex/gpt-5.5`, change the implicit
default of `@small_phase_worker` from `@phase_worker` to `@cheapest`, and change `@large_phase_worker` from
`@phase_worker` to `@smartest`. `@medium_phase_worker` keeps its `@phase_worker` fallback, and `@phase_worker` itself
still falls back to `@default`.

Two deliberate behavior changes must be called out in docs and the commit message: overriding the bare `phase_worker`
alias no longer implicitly governs small and large phases (only medium and explicit `@phase_worker` uses), and
`@smartest` is once again selected automatically — via the `large_phase_worker` fallback chain rather than hardcoded
phase-selector logic. Users who want the old uniform behavior configure the size aliases back to `"@phase_worker"`.

The alias layer is Python-only today (no `sase_core` mirror), and this work stays in this repo's
`src/sase/llm_provider/` policy layer with its existing callers; no Rust core changes are expected.

## Load-balanced alias values

- Grammar: split alias values on `|` outside of nothing-special (values are plain config strings), trim whitespace, and
  reject empty members. A single-member value degenerates to today's behavior. Each member uses the existing
  single-target grammar: `provider/model`, a bare known model, or an `@<alias>` reference, optionally carrying a
  trailing `@<effort>` from the canonical vocabulary (`sase.xprompt.effort.split_model_effort`).
- Pools are valid only in _alias values_: `model_aliases.builtin` / `model_aliases.custom` entries (including implicit
  builtin defaults). `%model` directive values, per-launch alias overrides (`%model:<alias>=<target>`), and temporary
  overrides stay single-target; keep the existing directive errors and add doctor/editor validation rather than new
  directive syntax.
- Selection happens where :func:`resolve_model_alias` encounters the pool-valued hop, keyed by the alias that owns the
  pool value (so `@small_phase_worker → @cheapest` and an explicit `%model:@cheapest` share one rotation). Resolution
  first drops members whose resolved provider is unregistered or whose `llm_autodetect_cli_name` is not on `PATH`
  (honoring `SASE_<PROVIDER>_PATH`; providers with no CLI name always count as installed). If every member is filtered
  out, keep the full pool so the launch fails with the normal missing-provider diagnostics instead of silently
  rerouting. Memoize the availability probe per process (mirroring the registry metadata caches, with a test-visible
  cache clear) so Models-panel refreshes never add per-row `PATH` scans.
- An `@<alias>` pool member follows the normal resolution chain. A chain that reaches a _second_ pool is treated like a
  cycle today — resolution falls back to the original input — and doctor flags nested pools as config errors. Cycle and
  depth protection must keep working across pool hops.
- After selection, peel a trailing effort suffix off the final resolved target and thread it into the launch's effective
  effort with precedence: explicit `%effort`/`@effort` (or a `%model:...@<effort>` directive suffix) wins, then the
  alias-target effort, then `llm_provider.default_effort`. Alias-borne effort is config-derived, so it is best-effort
  like `default_effort` (unsupported levels are skipped, never raised). This split must apply to any alias-resolved
  target — today a configured `claude/opus@medium` value silently produces the bogus literal model `opus@medium`, and
  that bug blocks the requested `@cheapest` pool. Launch metadata (`reasoning_effort`) and the `Model:` label must
  reflect the threaded effort.

## Usage tracking: consume vs. peek

- Persist rotation state in a new versioned, machine-global state file next to `llm_override.json` (e.g.
  `~/.sase/llm_lb.json` via `sase_home()`), following the `temporary_override.py` pattern exactly: fcntl lock file,
  atomic temp-file + `os.replace` writes, best-effort self-cleaning reads, and corrupt/missing state treated as a fresh
  rotation so it can never crash resolution. Each entry records the owning alias, a fingerprint of the normalized pool
  members, and the rotation cursor; a fingerprint mismatch (user edited the pool) resets the rotation.
- Resolution gets an explicit launch/peek distinction, with peek the default so no existing caller starts consuming
  accidentally. Only the authoritative launch lanes advance the cursor: the axe launch-metadata resolution in
  `src/sase/axe/run_agent_directives.py` and `invoke_agent`'s own `%model` lane in `src/sase/llm_provider/_invoke.py`
  (which resolves only when no explicit provider was passed, so one launched agent consumes exactly once). Display and
  analysis surfaces — the Models panel/alias views, model picker, completions, doctor, dry-run and plan/bead previews —
  peek: they show the _next_ selection without advancing it.
- Multi-agent launches (swarm segments, `%repeat`, alt fan-outs) advance once per launched agent, which is exactly how
  round-robin spreads a fleet across the pool. Resumptions reuse the provider/model recorded in agent metadata and must
  not re-consume.

## The `@cheapest` alias and phase-size rewiring

- Add `CHEAPEST_MODEL_ALIAS_NAME = "cheapest"` to the implicit alias policy with default value
  `claude/opus@medium | codex/gpt-5.5`, a description ("cheap load-balanced pool for high-volume agents" spirit), role
  kind, completion-catalog entry, and a Models-panel role row ordered after `smartest`. It joins no bucket. A
  user-configured `builtin.cheapest` (pool or single target) shadows the implicit value; temporary and launch-scoped
  overrides behave like any other role alias, and an active temporary override suspends load balancing for its duration
  (the override short-circuits before the pool hop, which the docs should state).
- Change `_ROLE_ALIAS_FALLBACKS`: `small_phase_worker` → `@cheapest`, `large_phase_worker` → `@smartest`. Phase-launch
  routing in `src/sase/bead/work.py` already emits the size aliases and needs no selector change; verify the rendered
  fallback chains and effective models everywhere they surface (bead renderer, dry-run, plan previews, Models panel).
- Update the example alias block in `src/sase/default_config.yml`, the JSON schema (`src/sase/config/sase.schema.json`)
  builtin/custom value descriptions to document the `|` pool syntax and the new `cheapest` name, and doctor's known
  alias enumeration.

## Surfaces, docs, and generated skills

- Models panel (`src/sase/llm_provider/alias_view.py` and `src/sase/ace/tui/modals/models_panel*`): an `AliasView` for a
  pool-valued alias keeps its raw pool string as the configured/implicit value and shows the peeked next selection as
  the effective provider/model; expose the parsed members (with availability) on the view so the panel's detail/edit
  rendering can present them, and make the persistent alias editor accept and validate pool values (reject empty members
  and nested pools with actionable messages). Consult the `tui_perf` memory before touching panel refresh paths.
- Doctor (`src/sase/doctor/checks_config_model_aliases.py`): validate pool values — empty members, members that resolve
  to nothing, nested pools, and an informational note when a member's provider CLI is currently unavailable (that is the
  fallback feature working, not necessarily an error).
- Docs: `docs/llms.md` (model aliases + reasoning-effort sections: pool grammar, rotation semantics, availability
  fallback, effort threading, consume-vs-peek), `docs/configuration.md` (schema/examples), `docs/ace.md` (Models panel),
  and `docs/beads.md` / `docs/sdd.md` (new small/large phase defaults and the `phase_worker`-override behavior change).
- Generated skill source `src/sase/xprompts/skills/sase_plan.md` currently states that each size alias falls back to the
  shared `phase_worker` alias; update it to the new chains (`small → @cheapest` pool, `medium → @phase_worker`,
  `large → @smartest`), adjust its source-contract tests, then follow the generated-skills workflow
  (`sase skill init --force`, `chezmoi apply`, linked-repo audit) — never hand-edit installed `SKILL.md` files.

## Verification

- Unit coverage in `tests/llm_provider/`: pool parsing/normalization; round-robin order across consecutive consuming
  resolutions; peek stability (repeated peeks do not advance); availability filtering with a mocked CLI probe, including
  the all-unavailable full-pool fallback and the single-installed-provider case always selecting the available member;
  fingerprint reset on pool edits; corrupt/locked state never crashing; shared rotation between `@small_phase_worker`
  chains and explicit `@cheapest`; nested-pool and cycle protection; effort threading precedence (explicit directive
  beats pool effort beats `default_effort`) and the general single-target `provider/model@effort` alias fix.
- Launch-lane tests: axe metadata resolution and `invoke_agent` each consume exactly once per launched agent; override
  precedence (launch-scoped, temporary, configured) still wins over pool selection; resumption does not re-consume.
- Surface tests: alias-view/panel rows for `cheapest` and the rewired size chains, completion catalog order and
  descriptions, schema acceptance of pool values, doctor findings for malformed pools, and refreshed Models-panel PNG
  goldens via the dedicated visual suite.
- Run `just install` first, iterate with focused test modules, and finish with `just check` (lint, typing, full suite,
  visual snapshots) plus the generated-skills regeneration and audit.
