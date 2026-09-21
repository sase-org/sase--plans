---
tier: epic
title: Provider-gated
goal: '#research_swarm launches one researcher per enabled LLM provider (codex and
  claude by default, grok and muse opt-in), skips any provider that is temporarily
  disabled, names every input and report suffix after its provider, and the lead consolidates
  whatever set of reports actually ran.

  '
phases:
- id: predicate
  title: Provider-disabled Jinja predicate
  depends_on: []
  size: small
  description: 'predicate: add `provider_disabled` / `provider_enabled` prompt-body
    Jinja filters over the existing Rust-backed provider-disable state, with tests
    and xprompt docs, so `%if(should_run=...)` can gate a segment on provider availability.

    '
- id: swarm
  title: Per-provider research swarm segments
  depends_on:
  - predicate
  size: medium
  description: 'swarm: rewrite the research_swarm xprompt around four provider-gated
    researcher segments plus an always-run lead, rename its inputs to `<provider>`
    / `<provider>_model`, switch report suffixes to provider short names, move the
    clan declaration onto the lead, and update the plugin''s config, tests, and docs.

    '
- id: aliases
  title: Retire the host researcher model aliases
  depends_on:
  - swarm
  size: xsmall
  description: 'aliases: drop the now-unreferenced `sol_or_grok` / `opus_or_grok`
    custom model aliases from the host SASE config in the chezmoi repo, keep `image`,
    and retune the `researchers` bucket description.

    '
- id: fanout
  title: Harden the installed-swarm fan-out regression
  depends_on:
  - swarm
  size: small
  description: 'fanout: make the fakey runner-slot test that plans the installed research
    swarm independent of machine-wide provider-disable state and extend it to cover
    the new opt-in providers.'
proposed_by: bbugyi200.athena.0oi
create_time: 2026-09-20 20:51:27
status: wip
bead_id: sase-14t
---

- **PROMPT:** [prompts/202609/research_swarm_providers.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/research_swarm_providers.md)
- **BEAD:** [sase-14t](https://github.com/sase-org/sase--beads/blob/main/pages/sase-14t/README.md)

# Plan: Provider-gated #research_swarm researchers

## Context

`#research_swarm` is defined by the `sase-research-artifacts` plugin at
`src/sase_research_artifacts/xprompts/research_swarm.md`. Today it is a fixed
four-segment swarm:

1. `research.{@1}.cdx` — "researcher A", `%m:{{ primary_model }}` (default
   `@sol_or_grok`), writes `<stem>__a.md` via `#research(suffix=a)`. This segment also
   carries the `%clan(research.{@1}, tribe=research, summary=...)` declaration for the
   whole clan.
2. `research.{@1}.cld` — "researcher B", `%m:{{ second_opinion_model }}` (default
   `@opus_or_grok`), writes `<stem>__b.md` via `#research(suffix=b)`.
3. `research.{@1}.final` — the lead, `%m:{{ lead_model }}` (default `@xlarge`), waits on
   both researchers, reads their registered reports, researches further, and writes the
   consolidated report.
4. `research.{@1}.image` — guarded by `%if(should_run={{ should_generate_image }})`.

The agent-id suffixes `cdx` / `cld` are already the SASE provider short names
(`llm_provider_short_name`: codex→`cdx`, claude→`cld`, grok→`grk`, muse→`mus`), but the
report filenames use `__a` / `__b`, and the provider is only implied by an alias
default.

### Mechanics this plan relies on

- **Static `%if` omission.** `%if(should_run=true|false)` is consumed by
  `filter_conditional_launch_segments` in `sase-core` before directives are collected: a
  `false` segment is dropped whole, a `true` segment keeps its body with the directive
  stripped. `sase.xprompt.processor._filter_conditional_xprompt_segments` applies it
  right after the xprompt body renders (with `preserve_segment_separators=True` on the
  swarm path), and `plan_typed_launch_units` applies it again before planning. Because
  only `%if::` fences and `%proc` require the `typed_launch_units` beta flag, the static
  `%if(...)` form works with that flag off — this is exactly how today's
  `should_generate_image` guard already works. **Do not** introduce a feature-flag
  dependency for this work.
- **Render order.** `expand_xprompt_swarms_with_metadata` substitutes the xprompt body
  first and only then splits the result on top-level `---`, so `%if` gating happens
  before the split, and call-site leading directives (e.g. a `#gh:` / `#git:` workspace
  ref) attach to the first _surviving_ sub-segment.
- **`should_run` parsing.** `parse_should_run` lowercases before matching, so Jinja's
  `True` / `False` render correctly; anything else is a hard error.
- **Clan attributes.** `resolve_clan_tribe` / `resolve_clan_summary` select the _latest
  explicit declaration_ for a clan generation across all members, and are
  order-independent. Members join with `%id(<suffix>, clan=<clan>)` without redeclaring
  tribe or summary.

### Scope decisions

- **`lead_model` keeps its name.** The rename to `<provider>_model` covers the
  per-provider researcher models. The lead is a role, not a provider, and its `@xlarge`
  default is a cross-provider size alias, so `<provider>_model` has no meaning for it.
  If Bryan wants the lead pinned per provider too, that is a follow-up, not part of this
  epic.
- **This is a breaking input rename.** `primary_model=` / `second_opinion_model=` stop
  working. SASE feature flags are a `sase`-project concern and this xprompt lives in a
  plugin, so no flag is created; the break is carried by a `BREAKING CHANGE:` commit
  trailer in the plugin repo (release-please builds its changelog from commits).
- **No SASE memory changes.** `sase/memory/xprompts.md` documents directives and xprompt
  structure, not the prompt-body Jinja filter registry (it does not mention the existing
  `plan_ref_path` filter either). Nothing in this epic changes SASE memory; if a later
  worker believes a memory note must change, route it through `/sase_memory_write`
  first.
- **Rust core boundary.** Provider-disable state — its wire schema, expiry,
  `hard`/`soft` modes, and self-cleaning read — already lives in `sase-core` behind the
  `provider_disable_get` binding. The new predicate is a thin Python render-time adapter
  over that binding and needs no `sase-core` change.

## Provider-disabled Jinja predicate

**Repo:** `sase` (this workspace's checkout).

Add two complementary prompt-body Jinja filters in `src/sase/xprompt/jinja_filters.py`,
registered from the existing `register_prompt_filters` hook so they reach both xprompt
bodies and workflow prompt parts:

```python
env.filters["provider_disabled"] = _provider_disabled
env.filters["provider_enabled"] = _provider_enabled
```

`_provider_disabled(provider, mode="any")`:

- Coerce `provider` with `str(...).strip().lower()`; a non-string, empty, or
  non-provider-id value returns `False` (not disabled) rather than raising, so a
  templating slip cannot silently claim a provider is down.
- Read the authoritative snapshot with
  `sase.llm_provider.provider_disable.get_active_provider_disables()` (lazy import,
  keeping the LLM package out of the low-level xprompt import graph, matching how
  `_plan_ref_path` imports `sase.sdd.plan_refs`). Absent record → `False`.
- `mode="any"` (default) is true for any active disable; `mode="hard"` uses
  `record.is_hard`; `mode="soft"` uses `record.is_soft`. Any other `mode` raises
  `ValueError` so an authoring typo fails loudly at render instead of quietly
  defaulting.
- Catch `ProviderDisableStateError` and `OSError` and return `False`. Fail open: a
  corrupt machine-wide state file must not break every prompt that uses the filter.

`_provider_enabled(provider, mode="any")` returns
`not _provider_disabled(provider, mode)`. Both forms are registered because the swarm
needs the positive guard inside a `%if(should_run=...)` argument, where an extra
`not (...)` is both noisier and easier to get wrong.

**Why a filter and not a Jinja global.** `jinja_inspect.unknown_variables` lints
undeclared variables against `RESERVED_GLOBAL_NAMES` / `BUILTIN_RUNTIME_NAMES`, and
`jinja2.meta.find_undeclared_variables` does not know about `env.globals` — a global
named `provider_disabled` would be reported as an unknown variable by prompt diagnostics
and the LSP. Filters need no lint change and are already surfaced to editor completion
by `jinja_inspect.jinja_filter_names()`.

**Do not** use `peek_active_provider_disables`. Its module-level cache only re-checks
the state path after a 0.5s monotonic floor, so a test that monkeypatches `SASE_HOME`
can be served a stale snapshot from a previous test. The authoritative read is also the
only one that self-cleans expired records.

**Tests** (new module, e.g. `tests/test_xprompt_jinja_provider_filters.py`), each with
`SASE_HOME` pointed at a `tmp_path`:

- no disable state at all → `provider_disabled` false, `provider_enabled` true;
- a hard disable → disabled under `any` and `hard`, not under `soft`;
- a soft disable → disabled under `any` and `soft`, not under `hard`;
- an already-expired record → not disabled;
- blank / non-string / unknown-provider input → not disabled;
- a corrupt `llm_provider_disables.json` → not disabled, no exception;
- an invalid `mode` raises `ValueError`;
- an end-to-end expansion assertion: an xprompt body containing
  `%if(should_run={{ "grok" | provider_enabled }})` on one of two `---` segments keeps
  both segments when grok is enabled and drops that segment when grok is disabled.

**Docs:** extend the "Jinja2 Integration" section of `docs/xprompt.md` with the two
filters and a worked `%if(should_run=...)` example, and cross-reference it from the
`%if(should_run=true|false)` documentation later in that same file.

**Verification:** `sase tool run check` (or `just check`) in the `sase` repo.

## Per-provider research swarm segments

**Repo:** `sase-research-artifacts` — open it with `/sase_repo`
(`sase repo open sase-research-artifacts -r "..."`) and use only the path it prints.

The plugin's `just install` routes its `sase` requirement at the coordinated local sase
checkout, so this phase tests directly against the filter from the `predicate` phase; no
sase release is needed first.

### Inputs

Replace the model inputs with a per-provider set. Keep `prompt` first so
`#research_swarm:: <text>` and `#research_swarm(<text>)` keep working; every new input
is named-only in practice.

| Name                    | Type | Default                                 | Meaning                       |
| ----------------------- | ---- | --------------------------------------- | ----------------------------- |
| `prompt`                | text | required                                | unchanged                     |
| `wait`                  | word | `null`                                  | unchanged                     |
| `priority`              | int  | `null`                                  | unchanged                     |
| `runners`               | int  | `null`                                  | unchanged                     |
| `codex`                 | bool | `true`                                  | request the codex researcher  |
| `claude`                | bool | `true`                                  | request the claude researcher |
| `grok`                  | bool | `false`                                 | request the grok researcher   |
| `muse`                  | bool | `false`                                 | request the muse researcher   |
| `codex_model`           | word | `codex/gpt-5.6-sol@xhigh`               | model for `<clan>.cdx`        |
| `claude_model`          | word | `claude/opus@xhigh`                     | model for `<clan>.cld`        |
| `grok_model`            | word | `grok/grok-4.6@xhigh`                   | model for `<clan>.grk`        |
| `muse_model`            | word | `muse/muse-spark-1.3-contributor@xhigh` | model for `<clan>.mus`        |
| `lead_model`            | word | `@xlarge`                               | model for `<clan>.final`      |
| `should_generate_image` | bool | `false`                                 | unchanged                     |

The defaults are concrete provider models rather than the old `||` fallback aliases:
with a dedicated grok researcher, an alias that silently falls back to grok would launch
two grok agents. Note in the input description that `muse-spark-1.3-contributor` carries
SASE's `warn` model advisory ("trains on your data"), which is part of why `muse`
defaults off.

### Segment structure

Six authored top-level segments: four researchers, the lead, the image agent. The raw
body must keep literal `---` separator lines — `xprompt_has_segment_separators` reads
the unrendered content to classify the xprompt as a swarm, so do not generate the
separators from a Jinja loop.

Compute the surviving researcher set once at the top of the body, side-effect free, with
whitespace-control so nothing leaks in front of the first segment's directives:

```jinja
{%- set researchers =
  ([{"short": "cdx", "provider": "codex", "model": codex_model}] if codex and ("codex" | provider_enabled) else [])
+ ([{"short": "cld", "provider": "claude", "model": claude_model}] if claude and ("claude" | provider_enabled) else [])
+ ([{"short": "grk", "provider": "grok", "model": grok_model}] if grok and ("grok" | provider_enabled) else [])
+ ([{"short": "mus", "provider": "muse", "model": muse_model}] if muse and ("muse" | provider_enabled) else [])
-%}
```

Each researcher segment then opens with its own guard, using the same predicate so the
guard and the peer list can never disagree:

```
%if(should_run={{ codex and ("codex" | provider_enabled) }}) %id(cdx, clan=research.{@1})
%m:{{ codex_model }} {% if wait %}%wait:{{ wait }} {% endif %}%q(w=0.25 ...)
```

Keep `%q(w=0.25, ...)` and the existing `runners=` / `priority=` conditionals
byte-identical in shape on every segment; the weight is load-bearing for the fan-out
regression test.

### Clan declaration moves to the lead

The `%clan(research.{@1}, tribe=research, summary=[[...]])` declaration currently sits
on the `cdx` segment, which can now be omitted. Move it onto the lead, which is the only
unconditional agent:

```
%clan(research.{@1}, tribe=research, summary=[[[bold]RESEARCH PROMPT:[/bold] {{ prompt }}]]) %id:research.{@1}.final %m:{{ lead_model }}
```

and give every researcher segment the plain `%id(<short>, clan=research.{@1})` member
form. This mirrors the shape the `cdx` segment uses today, just on a different member.

**Verify this rather than assume it.** `resolve_clan_tribe` / `resolve_clan_summary`
pick the latest explicit declaration for a generation and are documented as
order-independent, so a declaration on the last segment is expected to work. Confirm it
with a real dry run — plan a default `#research_swarm` through `plan_typed_launch_units`
and assert the clan name, `clan_tribe`, and `clan_summary` land on the planned units. If
declaring the clan on a trailing segment turns out not to resolve, fall back to emitting
the full `%clan(...)` form on `researchers[0]` (and on the lead only when `researchers`
is empty) from Jinja, and say so in the phase's completion notes.

### Researcher bodies

Replace the hard-coded "researcher A / researcher B" prose with a peer list derived from
`researchers`, keeping the existing no-peeking rules verbatim:

- name the peers as `` `research.{@1}.<short>` `` for every entry whose `short` differs
  from this segment's;
- state this researcher's own report suffix, e.g. "Your report will end in `__cdx.md`";
- when there are no peers, drop the peer paragraph entirely and say the researcher is
  the only independent researcher in this swarm;
- end with `{{ prompt }} #research(suffix=<short>)` — `suffix` is a free-form `word`
  input on `#research`, so `cdx` / `cld` / `grk` / `mus` need no change there.

### Lead body

- Waits: emit one `%wait:research.{@1}.<short>` per surviving researcher. With none, the
  lead emits no researcher waits and simply runs as a solo researcher — do not error and
  do not leave a dangling wait on an agent that was never launched.
- Step 1 becomes suffix-generic: identify exactly one report per expected suffix in
  `{{ researchers | map(attribute='short') | join(', ') }}`, matching by `wait_name` and
  the canonical research label's `__<suffix>.md`; never reassign suffixes from list
  order; stop and report if the registered reports do not identify exactly one report
  per expected suffix. Keep the existing `/sase_repo` + `sase artifact read`
  instructions and the "do not read predecessor chat transcripts" rule.
- Step 3 moves each report to `<name>/<name>__<suffix>.md`, preserving its existing
  suffix.
- Step 4's merge wording generalizes from "both reports and your own research" to "every
  report above and your own research"; with zero reports it is plainly the lead's own
  research.
- The "Final layout" fenced block is generated from `researchers` so it lists the actual
  `<name>__<short>.md` files plus `<name>.md`.
- Leave the `{% raw %}` `wait.artifacts` block exactly as it is — it is runtime-rendered
  during the lead's own run, not at expansion time.

### Image segment

Unchanged in behavior: still `%if(should_run={{ should_generate_image }})`, still
`%wait:research.{@1}.final`, `#fork:research.{@1}.final`, `%model:@image`.

### Plugin config, tests, and docs

- `src/sase_research_artifacts/default_config.yml`: remove the now-unreferenced
  `sol_or_grok` and `opus_or_grok` custom aliases, keep `image`, and rewrite the
  `researchers` bucket description for the new roles. If Bryan would rather keep the two
  aliases as a hand-override convenience, that is a cheap reversal — but leaving
  unreferenced multi-provider pools in place keeps producing doctor alias-pool
  advisories for no benefit.
- `tests/test_xprompt_loading.py`: update the input-name/type/default assertions and the
  indexed default assertions, change `test_research_swarm_has_four_top_level_segments`
  to the new authored segment count, and rework the dependency-graph assertions for the
  moved clan declaration and the `%m:{{ <provider>_model }}` bindings. Point `SASE_HOME`
  at a `tmp_path` in any test that renders the swarm, so a real machine-wide disable
  cannot change the rendered segment set.
- New plugin test coverage: defaults expand to codex + claude + lead; `grok=true` and
  `muse=true` each add their segment; `codex=false` drops `cdx`; a disabled provider
  drops its segment even when its boolean input is true; all four off still yields
  exactly the lead; reports use `#research(suffix=cdx|cld|grk|mus)`.
- `tests/test_default_config.py`: drop the retired alias assertions, keep `image`.
- `README.md`, `docs/xprompts.md`, `docs/configuration.md`, and `AGENTS.md`: update the
  input table, the segment walkthrough, the alias inventory, and the `__a` / `__b`
  references.
- `pyproject.toml`: **leave the `sase>=0.17.2` floor alone.** That floor is already
  ahead of the sase repo's own version, so inventing another speculative bound repeats
  an existing mistake. Instead, note in `AGENTS.md` and `README.md` that provider gating
  needs a host sase that ships the `provider_enabled` / `provider_disabled` prompt
  filters, and revisit the floor when a release actually carries them.
- Commit the break with a `BREAKING CHANGE:` trailer naming the `primary_model` →
  `codex_model` and `second_opinion_model` → `claude_model` renames and the `__a`/`__b`
  → `__cdx`/`__cld` report-suffix change.

**Verification:** `just check` inside the plugin checkout, plus the
`plan_typed_launch_units` dry runs described above. This phase changes files in a repo
opened through `/sase_repo`, so that repo is part of the turn's final declaration and
needs its own `commit` decision.

## Retire the host researcher model aliases

**Repo:** `chezmoi` — open it with `/sase_repo` and edit `home/dot_config/sase/sase.yml`
through the printed path only. Never edit the deployed `~/.config/sase/sase.yml`
directly.

Under `llm_provider.model_aliases.custom`, remove `sol_or_grok` and `opus_or_grok`: once
the swarm's defaults are concrete provider models, nothing references them, and each is
a two-member `||` pool that keeps drawing doctor advisories. Update the
`model_aliases.buckets.researchers.description`, which still says "primary researcher,
second-opinion researcher", to describe the per-provider researchers and the lead. Leave
`llm_provider.default_effort: xhigh`, the `research` tribe display config, and
everything else untouched.

Sanity-check afterwards with `sase doctor` (the model-alias and config checks) and
`sase xprompt show research_swarm`, confirming the rendered defaults no longer name a
removed alias. Because this repo is chezmoi-managed, the edit lands in the source tree;
the deployed copy follows from Bryan's normal chezmoi apply.

## Harden the installed-swarm fan-out regression

**Repo:** `sase`.

`tests/fakey/test_runner_slots_e2e.py::test_installed_research_swarm_quarter_weights_fill_one_fakey_capacity_unit`
plans the _installed_ `#research_swarm` and asserts exact unit counts (3 by default, 4
with `should_generate_image=true`) and a `0.25` weight on every unit. The default count
is unchanged by this epic — codex + claude researchers plus the lead is still three —
but the plan now depends on machine-wide provider-disable state, which the test reads
from the real `~/.sase` because it plans before the fakey harness monkeypatches
`SASE_HOME`.

- Set `SASE_HOME` to an empty `tmp_path` subdirectory _before_ the first
  `expand_prompt_for_typed_launch` / `plan_typed_launch_units` call, so a genuine codex
  or claude disable on the developer's machine cannot turn this into a phantom failure.
  Keep the fakey harness's own `SASE_HOME` separate from the planning one if the harness
  needs its own root.
- Keep the existing weight, `wait_runners`, `wait_priority`, and wait-graph assertions.
- Extend the test with the new gating: `grok=true` adds a fourth unit, `muse=true` a
  fifth, `codex=false` drops one, and a written-in provider disable for codex drops the
  codex unit while leaving the rest at weight `0.25`.
- The test already `importorskip`s `sase_research_artifacts`, so it stays a local-only
  guard; do not make it a hard CI requirement.

**Verification:** `sase tool run check` (or `just check`) in the `sase` repo.

## Landing notes

- Phases `aliases` and `fanout` both depend only on `swarm` and can run in parallel.
- After `predicate` lands, Bryan's installed sase (the uv tool install, not this
  checkout) needs a `sase update` before the real `#research_swarm` can render the new
  guards — the plugin resolves its host sase from the installed environment at runtime
  even though its tests use the local checkout.
