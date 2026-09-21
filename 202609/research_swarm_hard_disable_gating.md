---
tier: tale
title: Gate research swarm researchers on hard provider disables only
goal:
  "#research_swarm skips a researcher only when its provider is hard-disabled, and
  xprompt swarm authors have tested, documented soft-vs-hard provider-disable gating."
size: small
proposed_by: bbugyi200.apollo.1e.w0
create_time: 2026-09-21 08:45:47
status: wip
---

# Research swarm: gate researchers on hard provider disables only

## Problem

`#research_swarm` (in the linked `sase-research-artifacts` plugin repo) drops a
per-provider researcher segment whenever that provider has _any_ active machine-wide
disable, because every gate uses the bare `provider_enabled` Jinja filter, whose default
mode is `"any"`:

```jinja
... if codex and ("codex" | provider_enabled) else [] ...
%if(should_run={{ codex and ("codex" | provider_enabled) }}) %id(cdx, clan=research.{@1})
```

That is wrong for **soft** disables. A soft disable only steers alias/pool routing away
from the provider; it never refuses an explicit launch (see
`src/sase/agent/launch_guard.py`, which blocks only hard-disabled providers, and
`docs/llms.md` / `docs/beads.md`). Each research-swarm researcher pins an explicit
per-provider model (`%m:{{ codex_model }}` etc.), so a soft-disabled provider would
still launch fine — the swarm should only skip a researcher when its provider is
**hard** disabled (the case where the launch guard would refuse the segment).

## Current host capability (already present — verify, do not reimplement)

The sase host already lets prompt-body/swarm Jinja distinguish the two modes:
`src/sase/xprompt/jinja_filters.py` registers `provider_disabled` / `provider_enabled`,
both taking an optional mode argument `"any"` (default), `"hard"`, or `"soft"`, backed
by `TemporaryProviderDisable.is_hard` / `.is_soft`. Unit tests live in
`tests/test_xprompt_jinja_provider_filters.py` and the filters are documented in
`docs/xprompt.md` (Jinja2 Integration table, ~line 891, and the Static conditional
segments section, ~line 1747). What is missing on the host side is:

- a swarm-level regression test proving mode-qualified gates behave correctly through
  real xprompt-swarm expansion (only plain `_filter_conditional_xprompt_segments` and
  mode-less gates are covered today), and
- documentation steering swarm authors to the `"hard"` mode when a segment pins a
  provider's model explicitly.

No Rust-core change is needed: the disable record and its mode already come from the
Rust-backed `provider_disable` facade; the filter is thin Python template glue.

## Changes

### 1. sase (this repo)

1. **Swarm-level test.** In `tests/test_xprompt_jinja_provider_filters.py` (or
   `tests/test_xprompt_swarm_expansion.py` if its fixtures make a real swarm easier to
   build), add tests that use a small xprompt swarm (or a rendered multi-segment body
   pushed through the same filtering used for swarm expansion) with two segments, the
   second gated by `%if(should_run={{ "grok" | provider_enabled("hard") }})`, isolated
   under a `tmp_path` `SASE_HOME`:
   - no disable → gated segment survives;
   - `disable_provider("grok", 900.0, source="test", mode="soft")` → gated segment
     **survives**;
   - `disable_provider("grok", 900.0, source="test")` (default hard) → gated segment is
     dropped;
   - symmetric check that `provider_disabled("soft")` is true only for the soft record
     when rendered through `get_jinja_env()` (template-call syntax, not just the Python
     helper), so the mode argument is proven to parse in real templates.
2. **Docs** (`docs/xprompt.md`):
   - In the Static conditional segments section (near the existing
     `%if(should_run={{ "grok" | provider_enabled }})` example), add a short paragraph
     and example explaining that `provider_enabled` defaults to `"any"`, and that a
     segment which pins an explicit provider model should normally gate on
     `provider_enabled("hard")`, because a soft disable never refuses explicit launches
     (only hard disables trip the launch guard). Mention `provider_disabled("soft")` for
     authors who want to react to soft disables specifically (e.g. adjust prose).
   - Keep the Jinja2 Integration table consistent (it already documents the mode
     argument; only touch it if wording needs to reference the new guidance).
   - Keep lines wrapped/formatted the way the surrounding Markdown is (the repo runs a
     Markdown formatter via its lint recipe).

### 2. sase-research-artifacts (linked repo)

Open it with the `/sase_repo` skill
(`sase repo open sase-research-artifacts -r "<reason>"`) and make all edits/tests in the
path that command prints.

1. **Swarm template** `src/sase_research_artifacts/xprompts/research_swarm.md`: change
   every `provider_enabled` gate to `provider_enabled("hard")` — the five entries in the
   `researchers` list prelude (codex, claude, grok, muse, and `agy` for gemini) and the
   five matching `%if(should_run=...)` segment headers. The `researchers` list and the
   `%if` headers must stay in lockstep so peer counts, the layout listing, and the
   lead's `%wait` list agree with the segments that actually launch. Grep afterwards to
   confirm no bare `provider_enabled` (without `("hard")`) remains.
2. **Tests** `tests/test_xprompt_loading.py`:
   - Keep `test_research_swarm_disabled_provider_drops_segment` and
     `test_research_swarm_disabled_agy_drops_gemini_segment` but make the hard mode
     explicit (`mode="hard"`) and rename them to say `hard_disabled`, so intent is
     obvious.
   - Add `test_research_swarm_soft_disabled_provider_keeps_segment`: soft-disable
     `codex`, render with defaults, assert `%id(cdx, clan=research.{@1})` is present,
     the lead's final segment still has `%wait:research.{@1}.cdx`, the researcher count
     prose says `2-researcher swarm`, and each segment still has exactly one queue
     directive (reuse `_assert_each_segment_has_one_queue`).
   - Add the gemini/`agy` soft analogue (soft-disable `agy`, `gemini=true` → 4 segments
     including `%id(gem,`).
3. **Docs** — replace the "temporarily disabled drops its researcher" wording with
   "hard-disabled", and add that a soft-disabled provider still runs its researcher
   (soft disables never refuse explicit model launches):
   - `README.md` (~line 104, the `#research_swarm` bullet),
   - `docs/xprompts.md` (~line 73),
   - `AGENTS.md` (~line 35) and the "Provider gating needs a host sase..." sentences in
     `README.md` / `docs/xprompts.md`: note the host must ship the mode-aware
     `provider_enabled("hard")` filter. The existing `sase>=0.17.2` floor already
     targets the first release carrying these filters (the mode argument landed in the
     same host commit as the filters), so do not change `pyproject.toml`.
4. Run that repo's checks per its `Justfile` (e.g. `just check` or the lint/test recipes
   it defines) and make them pass.

## Verification

- sase: read the `lint_and_test` reference memory (via `/sase_memory_read`) and run the
  verification recipe it prescribes (`just check`), plus the targeted
  `pytest tests/test_xprompt_jinja_provider_filters.py`.
- sase-research-artifacts: `pytest tests/test_xprompt_loading.py` and the repo's check
  recipe, all green.
- Sanity: in both repos, `grep -rn 'provider_enabled' ` shows no bare mode-less gate in
  `research_swarm.md`.

## Out of scope

- Changing the default mode of `provider_enabled` / `provider_disabled` (other callers
  may legitimately want `"any"`).
- Alias-based researcher models (e.g. `codex_model=@codex`): gating remains keyed on the
  researcher's nominal provider, as today.
