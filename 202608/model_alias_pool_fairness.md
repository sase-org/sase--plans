---
tier: tale
title: Route the no-%model launch default through the alias chain so pools rotate fairly
goal:
  Every launch lane that picks a model through a load-balanced alias pool advances the
  shared round-robin cursor exactly once, so pool members are selected in equal shares —
  including the no-%model default lane, which today bypasses the pool entirely.
size: medium
proposed_by: bbugyi200.athena.z7
create_time: 2026-08-13 08:29:24
status: wip
---

# Plan: Route the no-`%model` launch default through the alias chain

## Problem

SASE already has working round-robin load balancing for `A | B` model alias pools, with
rotation state persisted across sessions and machine restarts. It is not reaching the
most-used launch lane.

`src/sase/llm_provider/load_balancing.py` keeps a per-alias cursor in
`sase_home()/llm_lb.json`, keyed by the pool-**owning** alias name plus a membership
fingerprint, serialized with an `flock` on `llm_lb.lock` and written atomically. It
degrades to a fresh rotation rather than failing a launch when the state file is
missing, corrupt, or locked. `select_model_alias_pool_member(...)` advances the cursor
only when `consume=True`, and skips unavailable members while still advancing past them.

That machinery demonstrably works on the phase-worker lane: recent launches recorded
with `model_alias=medium_worker` alternate cleanly between the two `@smart` pool members
(`claude/sonnet` ↔ `codex/gpt-5.5`), i.e. exactly one cursor advance per launch.

The no-`%model` lane never enters the pool at all. On a machine with no
`llm_provider.model_aliases.builtin` section configured:

```
resolve_effective_default_provider_model_with_effort(consume=False)
    -> ('codex', 'gpt-5.6-sol', None)      # launch default lane
resolve_model_provider_with_effort('@default', consume=False)
    -> ('codex', 'gpt-5.6-sol', 'xhigh')   # the alias chain
```

The missing `xhigh` is the tell: the launch lane is not traversing the alias chain at
all. It returns the configured provider's tier default, which on this machine coincides
with `@smarter`'s member 0 — so the result looks plausible while never rotating to the
other member. Across roughly 4000 indexed agents, launches with no recorded
`model_alias` show `codex/gpt-5.6-sol` 2517 vs `claude/opus` 769 — the two `@smarter`
pool members, badly skewed.

## Root cause

`resolve_default_alias_provider_model()` and
`resolve_default_alias_provider_model_with_effort()` in
`src/sase/llm_provider/registry.py` route through the alias resolver only when
`get_model_aliases().get(default_model_alias_name())` is truthy. `get_model_aliases()`
(`src/sase/llm_provider/model_alias_config.py`) returns **configured** aliases only —
the merged `builtin` + `custom` maps from user config. It does not include the shipped
implicit aliases from `src/sase/llm_provider/model_alias_defaults.yml`. When `default`
is not user-configured, both functions fall through to
`get_provider(provider_name).resolve_model_name(model_tier)`.

`src/sase/llm_provider/_invoke.py` carries a second, independent gate with the same bug
(`elif default_model_alias_name() in get_model_aliases():`). When that is false, neither
`provider_name` nor `model_override` is set and the code falls through to plain tier
resolution. Fixing `registry.py` alone does not fix the `invoke_agent` lane.

This gate used to be correct. Commit `012e1a88b` ("feat: add smarter model alias
routing") added `default: fallback: "@smarter"` to `model_alias_defaults.yml`; before
it, `default` declared no fallback, so consulting only configured aliases was right.
That commit did not update `registry.py` or `_invoke.py`.

The documented behavior already matches the intended fix: `docs/llms.md` describes
`@default` as "Model used when a prompt has no %model directive; delegates to @smarter
unless configured." This is a code/doc divergence, not a design change.

`src/sase/axe/run_agent_directives.py` calls
`resolve_effective_default_provider_model_with_effort(..., consume=True)`
unconditionally, so the axe agent-runner lane is fixed for free once `registry.py` is
fixed.

## Non-goals

- Cross-machine cursor sharing. Rotation state is intentionally per-machine.
- Changing the reset-to-member-zero behavior when a pool's membership fingerprint
  changes.
- Adding per-member selection **counts** to `llm_lb.json` (a state schema bump). Real
  launch data plus the provenance recorded in step 4 is enough to verify fairness; if
  the implementer still wants counters afterward, file a task bead with
  `/sase_new_task`.

## Implementation

### 1. Re-verify the findings before changing anything

Confirm the two resolutions still disagree, so the fix is aimed at a live defect and not
at something already repaired upstream:

```bash
python -c "
from sase.llm_provider.registry import resolve_model_provider_with_effort
from sase.llm_provider.temporary_override import resolve_effective_default_provider_model_with_effort
print('default lane ->', resolve_effective_default_provider_model_with_effort(consume=False))
print('@default     ->', resolve_model_provider_with_effort('@default', consume=False))
"
```

The launch-default tuple should carry no alias-borne effort while `@default` does. If
they already agree, stop and reassess — the rest of this plan assumes the divergence.

### 2. Route the launch default through the alias resolver unconditionally

In `src/sase/llm_provider/registry.py`, change
`resolve_default_alias_provider_model_with_effort()` so it resolves
`f"@{default_model_alias_name()}"` through `resolve_model_provider_with_effort(...)`
regardless of whether `default` is user-configured, passing `consume` through unchanged.
Keep the existing "provider is `None` → run the bare model on the configured provider"
fallback, since a configured `default` may still be a bare/unknown model name.

`_resolve_model_alias_result()` in `src/sase/llm_provider/model_alias_resolution.py`
already does the right thing for both cases: it consults configured aliases, then
implicit targets, then `implicit_model_alias_fallback_reference(...)`, and only falls
through to `resolve_default_alias_target()` (the provider tier default) when `default`
declares no shipped fallback. No new resolution logic is needed — only the removal of
the gate.

Make `resolve_default_alias_provider_model()` (the non-effort variant) delegate to the
`_with_effort` version and drop its effort component, rather than carrying a second copy
of the same branch logic that can drift again.

Preserve the existing precedence exactly. All three of these must still beat the shipped
fallback, in this order:

1. an active temporary `default` override (`get_active_alias_override("default")`),
2. a launch/`%model_alias` override for `default` (`active_launch_alias_overrides(...)`,
   handled in `resolve_effective_default_provider_model_with_effort`),
3. a user-configured `llm_provider.model_aliases.builtin.default`.

Verify there is exactly **one** consuming resolution on this path — the launch-override
branch in `resolve_effective_default_provider_model_with_effort` returns early, so it
must not also fall through into the repaired branch.

### 3. Decide and implement `model_tier` semantics

`resolve_default_alias_provider_model[_with_effort]` accepts `model_tier`
(`"large"`/`"small"`), but `resolve_default_alias_target()` hardcodes
`resolve_model_name("large")`. Once `default` always routes through the alias chain, the
tier only matters when the chain terminates at the provider tier default.

Preferred: thread `model_tier` into the terminal `resolve_default_alias_target()`
resolution so tier semantics survive. Note that `model_alias_resolution.py` looks this
function up through `config.__dict__.get("_resolve_default_alias_target", ...)`, a patch
point several tests rely on — keep that indirection intact and keep the patched
signature working.

Acceptable alternative if threading proves invasive: leave the tier alone and document
explicitly (docstring plus `docs/llms.md`) that a configured or shipped `default`
supersedes `model_tier`. A grep found no `model_tier="small"` / `model_size="little"`
call sites in `src/`, so this is an API-contract question rather than a live regression
— but decide it deliberately instead of letting it change by accident.

### 4. Remove the duplicate gate in `_invoke.py`

In `src/sase/llm_provider/_invoke.py`, the
`elif default_model_alias_name() in get_model_aliases():` branch must resolve the launch
default whenever no explicit `%model` directive and no explicit `provider_name` were
supplied — not only when `default` is configured. Keep `consume=True` here: this is an
authoritative launch lane.

Re-check the surrounding precedence in that function: an explicit `%model` directive and
an explicit `provider_name` argument must still win, and an active primary temporary
override must still take the new-launch-default slot ahead of the alias chain.

### 5. Record alias provenance for default-lane launches

`src/sase/axe/run_agent_directives.py` currently sets
`agent_model_alias = directives.model_alias if directives.model else None`, so a
no-`%model` launch writes `model_alias: null` into `agent_meta.json`. After step 2 those
launches resolve through the implicit `default` alias and into a pool, but nothing
records that — leaving no way to audit whether the default lane is distributing evenly,
because the artifact index cannot distinguish an implicit-default launch from an
explicit `%model codex/gpt-5.6-sol`.

Record the entry-point alias (`default`) for launches that resolved through the default
lane. This matches the existing convention: `@medium_worker` records `medium_worker`,
the alias the caller entered, not `smart`, the pool that owns the cursor.

**Decide the display consequence deliberately.** `model_alias` feeds the `← @<alias>`
provenance chip built in `src/sase/llm_provider/model_label.py`, so recording it would
add `← @default` to nearly every agent row in the TUI. Either:

- accept the chip (it is accurate, and it makes the routing visible), updating the ACE
  PNG snapshot suite accordingly — see the two-speed verification note below; or
- record the provenance in a separate metadata field that the chip does not read, if the
  ubiquitous chip is judged too noisy.

Do not preserve the reuse path's behavior by accident: `run_agent_directives.py`
deliberately reuses preserved `agent_meta` `model`/`llm_provider` on a runner re-exec so
a pool is not advanced twice. Whatever field is written must participate in that same
preservation, or a re-exec will re-resolve and double-advance the cursor.

### 6. Confirm the one-advance-per-launch invariant end to end

The core fairness invariant is that a single launch advances a pool cursor exactly once.
With two-member pools, a double advance pins every launch to one model —
indistinguishable at a glance from the bug this plan fixes.

Add a test that drives a launch end to end (not just `select_model_alias_pool_member` in
isolation) and asserts the cursor moved by exactly one, then asserts that a runner
re-exec reusing preserved metadata moves it by zero.

While here, check one known metadata-accuracy wrinkle: `resolve_model_meta()` in
`src/sase/axe/run_agent_exec_plan_accept_models.py` resolves with `consume=False` and
that result is written as the follow-up's `model`/`llm_provider` in `agent_meta.json`
(`_write_followup_model_meta` in `src/sase/axe/run_agent_exec_plan_accept.py`), while
the model that actually runs comes from the `consume=True` resolution inside
`invoke_agent` when the follow-up prompt's `%model:@<alias>` prefix is resolved.
Normally both read the same cursor position and agree; with concurrent agents advancing
the shared cursor in between they can diverge. A similar display-only `consume=False`
resolution lives at `src/sase/xprompt/workflow_executor_steps_prompt.py` (explicitly
labeled "for TUI display"). If the divergence is real but fixing it properly means
restructuring how the authoritative selection is threaded, do **not** expand this tale —
file a task bead with `/sase_new_task` and note it here.

### 7. Regression tests

The existing suites (`tests/llm_provider/test_load_balanced_aliases.py`,
`tests/llm_provider/test_load_balanced_alias_state.py`) cover the selector and the state
file well, and all passed while this bug was live — they test
`select_model_alias_pool_member` and `resolve_model_alias("@pool")` directly, never the
launch lane. Add tests at the level that would actually have caught it:

- With no user-configured `default` and a shipped `default -> @smarter` fallback,
  `resolve_effective_default_provider_model_with_effort(consume=True)` resolves into the
  pool and alternates across members on successive calls, carrying alias-borne effort.
- The same assertion through `_invoke.py`'s launch-default branch, so the second gate
  stays fixed.
- A user-configured `builtin.default`, an active temporary `default` override, and a
  launch alias override for `default` each still win over the shipped fallback.
- The tier decision from step 3, asserted explicitly.
- The cursor-advance-count assertions from step 6.

Prefer the existing fixtures — `tests/_model_alias_defaults_fixture.py` for a controlled
shipped-defaults graph, and whatever isolates `SASE_HOME` so tests never touch the real
`~/.sase/llm_lb.json`. Confirm that isolation before running anything with
`consume=True`.

### 8. Documentation

Update `docs/llms.md` where it describes the default lane so the text matches the
repaired behavior, including whatever step 3 decides about `model_tier`. The existing
`@default` row already claims delegation to `@smarter`, so most of this is confirming
rather than rewriting. Add a short "verifying pool fairness" note describing how to
check the distribution from recorded launches (see below).

Check whether `src/sase/doctor/checks_config_model_aliases.py` says anything that
assumes the configured-only behavior, and correct it if so.

## Verification

Follow the repo's two-speed verification rule. Run `just install` first — workspaces are
ephemeral and dependencies may have moved.

```bash
just install
just check
```

Because step 5 may touch ACE PNG snapshots, run the visual suite when the provenance
chip changes:

```bash
just test-visual
```

Run `just check-full` through `/sase_monitor` before landing — never inline — with a
`--next` action so the follow-up agent acts on the result.

Then verify the actual behavior change:

```bash
python -c "
from sase.llm_provider.registry import resolve_model_provider_with_effort
from sase.llm_provider.temporary_override import resolve_effective_default_provider_model_with_effort
print('default lane ->', resolve_effective_default_provider_model_with_effort(consume=False))
print('@default     ->', resolve_model_provider_with_effort('@default', consume=False))
"
```

Both should now agree, alias-borne effort included.

Finally, confirm equal distribution from real launches after the fix has been in use for
a while: query the agent artifact index for agents recorded with the default-lane
provenance from step 5 and count `provider/model` pairs. The counts across a pool's
members should be within one of each other. Before the fix the same query over launches
with no recorded alias showed roughly a 3:1 skew.

## Risks

- **Behavior change on every no-`%model` launch.** After this fix, half the default-lane
  launches on a two-member pool move to the other provider. That is the point of the
  change, but it is a visible shift in which CLI runs, and it takes effect the moment
  the code lands.
- **The provenance chip in step 5** will appear on nearly every agent row unless the
  separate-field option is chosen. Decide before implementing, not after seeing snapshot
  diffs.
- **Double-advance is the main way to get this wrong.** A second consuming resolution
  anywhere on the launch path silently pins two-member pools to a single model while
  looking like normal operation. Step 6's assertions are the guard.
