---
tier: tale
title:
  Detect Antigravity's real quota message and stop mislabelling pooled-alias launches
goal:
  An Antigravity usage-limit failure auto-disables `agy` and drops it out of the pooled
  aliases, and every surface that names an agent's provider names the one that actually
  ran.
size: medium
proposed_by: bbugyi200.athena.049
create_time: 2026-08-16 16:07:43
status: wip
---

# Plan: Detect Antigravity's real quota message and stop mislabelling pooled-alias launches

## Background: what actually happened

A phase agent (`sase-nb.4`, `%model:@small`) failed after ~46 minutes with:

```
Step 'main' failed: LLMInvocationError: Error running LLM provider command (exit code 1)
stderr: Error: Individual quota reached. Please upgrade your subscription to increase
your limits. Resets in 4h14m50s.
```

Every SASE surface reported that agent as **codex/gpt-5.5** — the ACE row carried the 🤖
codex badge and `sase agent show sase-nb.4` printed
`Model: CODEX(gpt-5.5) @ high ← @small`. That attribution is false. The run log
(`/home/bryan/.sase/workflows/202608/gh_sase-org__sase_ace-run-260816_123639.txt`)
records the literal argv
`['agy', '--print-timeout', '24h', '--model', 'gemini-3.7-flash-high', ...]`, the
traceback unwinds through `src/sase/llm_provider/agy.py`, and that run's
`agent_meta.json` carries `"exec_llm_provider": "agy"` alongside the stale
`"llm_provider": "codex"`. The failure was Antigravity's Gemini quota; the codex quota
was never touched.

Two independent defects chained together to produce that outcome, and both are fixed
here.

### Defect A — `agy`'s usage-limit patterns do not match Antigravity's real message

`src/sase/llm_provider/agy.py` `llm_default_usage_limit_config()` ships four guessed
Google-API-shaped patterns: `resource_exhausted`, `quota exceeded`,
`insufficient_quota`, `you exceeded your current quota`. The comment above them says
they were inferred rather than captured. Antigravity's actual wording is
`Individual quota reached. Please upgrade your subscription to increase your limits. Resets in <duration>.`
— "quota **reached**", which matches none of them.

Consequences, all observed:

- `detect_usage_limit()` returns `None`, so `handle_possible_usage_limit()`
  (`src/sase/llm_provider/usage_limit_disable.py`) writes no disable and sends no
  notification. The machine-wide disable store `~/.sase/llm_provider_disables.json` does
  not exist — the auto-disable has never fired for this provider.
- Because no disable exists, `resolved_target_is_available()` keeps reporting `agy`
  available, so the `@small` / `@xsmall` round-robin pools kept handing it out for the
  rest of the day.
- Two runs burned this way on the same day: `...ace-run-260816_103731.txt` (reset in
  1h25m) and `...ace-run-260816_123639.txt` (reset in 4h14m), costing roughly two hours
  of wall clock and stalling an epic phase.

The reset-hint parser already handles this message correctly _once the pattern matches_:
`_RESET_DURATION_RE` in `src/sase/llm_provider/usage_limit_config.py` extracts `4h14m`
from `Resets in 4h14m50s` (the trailing seconds are dropped, which the +60s
`_RESET_GRACE_SECONDS` buffer covers), yielding a ~4h15m disable. So the detection gap
is the whole of Defect A.

### Defect B — the metadata reconciliation is gated on a check a display rename silently disables

Provider/model selection for a pooled alias happens twice by design:

1. `src/sase/axe/run_agent_directives.py:303` resolves a **non-consuming preview** at
   reservation time and writes it to `agent_meta.json`. Its own comment states the
   preview's "metadata is reconciled with the authoritative selection afterward".
2. `src/sase/xprompt/workflow_executor_steps_prompt.py:285` performs the single
   **consuming** resolution immediately before the provider call, advancing the
   machine-global pool cursor.

The reconciliation that makes (1) agree with (2) lives at
`src/sase/xprompt/workflow_executor_steps_prompt.py:314`, guarded by
`if self.workflow.is_anonymous():`. `Workflow.is_anonymous()`
(`src/sase/xprompt/workflow_models.py:249`) is implemented as
`self.name.startswith("tmp_")`.

But `src/sase/xprompt/workflow_runner.py:281` mutates that name in place —
`workflow.name = wf_name` — turning `tmp_<timestamp>` into `gh` and then returning
`None` (not flattened). That line is a **display-only** rename added in `ef6686efb`
(2026-02-23, "Fix resume workflow showing tmp_ name instead of 'resume' in TUI"), long
before the reconciliation block landed in `9df15dbe2` (2026-08-14). Its own neighbouring
comment says the rename exists "so workflow_state.json shows the real workflow name".

So for the single most common SASE launch shape — a prompt beginning with
`#gh:<project>`, whose referenced workflow has a `prompt_part` and therefore takes the
`has_prompt_part()` branch — the workflow is renamed to `gh`, `is_anonymous()` flips to
`False`, and the reconciliation never runs. `agent_meta.json` keeps the stale preview
forever. The failing run's log contains the exact `logger.warning` emitted on the line
immediately above that rename, which pins the code path.

Why the preview and the real selection diverged at all in this instance: the agent was
reserved at 12:36 and previewed pool member `codex`, then blocked ~2h18m on a `%w`
dependency; when the real consuming resolution finally ran at 14:55 the machine-global
cursor had advanced to member `agy`. Long `%w` waits make preview/consume divergence
routine, not exceptional.

**Blast radius.** Every provider-naming surface reads the `llm_provider` meta field and
none reads `exec_llm_provider`:

- ACE agent-row badge — `ordered_row_providers()`
  (`src/sase/ace/tui/widgets/_agent_list_helpers.py:49`) → `provider_emoji_badge()`
  (`src/sase/integrations/provider_badges.py`), which is why `agy`'s 🪐 rendered as
  codex's 🤖.
- `src/sase/integrations/_agent_list_entry_builder.py:131`.
- `sase agent show` (`src/sase/agents/cli_show.py:61`) and `sase agent list`
  (`src/sase/agents/cli_list.py:146`).
- `src/sase/agents_sync/rendering_agent_page.py:56` and the other agents_sync renderers.
- The Launch Control alias agent-history panel, via `AliasHistoryRun.llm_provider`
  (`src/sase/llm_provider/alias_history.py:94`) — the one panel whose entire purpose is
  answering "what did `@small` actually launch?".

Fixing the reconciliation at the root makes all of these truthful at once, with no
changes to any display code and no change to the Rust alias-history wire contract in
`../sase-core`. That is the reason this plan fixes the gate rather than teaching each
reader to prefer `exec_llm_provider`.

**Why the existing test suite is green.**
`tests/test_pooled_alias_single_consumption.py:168`
(`test_root_metadata_step_marker_and_chat_agree_with_invoked_model`) asserts exactly the
property that is broken in production, but it builds the workflow with
`create_anonymous_workflow()` and hands it straight to `WorkflowExecutor`, never routing
through `execute_workflow()` where the rename happens. The name stays `tmp_*`, so the
gate stays open and the test passes. Closing that gap is a required part of this work.

## Scope

In scope: Defect A and Defect B, plus regression coverage that would have caught each.

Out of scope (record as `PROPOSED FOLLOW-UP:` notes rather than doing them here):

- **No failover for the discovering launch.** A `|` round-robin pool selects one member
  in `select_model_alias_pool_member()` and `invoke_agent()` then raises. Even with
  Defect A fixed, the _first_ agent to discover a provider's limit still dies; the
  auto-disable only protects subsequent launches. Re-selecting once after a usage-limit
  disable is written is a real improvement but a separate design decision.
- **`agy`'s retry patterns have the same blind spot.** `llm_default_retry_config()` in
  `src/sase/llm_provider/agy.py` lists `RESOURCE_EXHAUSTED`, `UNAVAILABLE`,
  `DEADLINE_EXCEEDED`, `deadline exceeded`, `rate limit`, `print-timeout`,
  `connection reset` — none of which match this message either. Retrying a multi-hour
  quota exhaustion is the wrong response anyway, so leave retry alone; just note it.
- **Other providers carry unverified guessed patterns.** `qwen.py` ships the identical
  guessed set, and `muse.py` / `opencode.py` carry comments explicitly stating their
  patterns are "unverified". Do **not** invent replacements for them here. Guessing is
  precisely what produced Defect A; those entries should only change when someone has a
  captured failure message in hand.

## Work

### 1. Make `agy` detect the real Antigravity quota message

In `src/sase/llm_provider/agy.py`, extend `llm_default_usage_limit_config()` with the
observed wording. Add pattern(s) covering the captured message:

```
Individual quota reached. Please upgrade your subscription to increase your limits.
Resets in 4h14m50s.
```

Constraints on the pattern choice:

- Patterns are case/apostrophe/whitespace-normalized substring matches
  (`find_matching_pattern()` in `src/sase/llm_provider/usage_limit_config.py`), not
  regexes. Do not embed the variable duration.
- Keep the existing four patterns. They are unverified but harmless, and removing them
  is an unrelated risk.
- Prefer a pattern specific enough not to fire on advisory or near-miss text.
  `quota reached` plus the upgrade-prompt sentence is a reasonable anchor; if you choose
  a short pattern, justify in the code comment why a false positive is implausible for
  this provider, and consider whether an `exclude_patterns` guard is warranted (compare
  the `claude` and `codex` providers, which both guard against "usage limit
  approaching").
- Replace the now-falsified comment ("usage-limit failures surface as Google API quota
  errors rather than distinctive prose") with one that states the message was captured
  from a real failure and cites the reset-hint format.

### 2. Make anonymity an identity the display rename cannot clear

Give `Workflow` an explicit anonymity property established at construction in
`create_anonymous_workflow()` (`src/sase/xprompt/models.py:297`) and not derived from
the mutable `name`, then gate the root-metadata reconciliation at
`src/sase/xprompt/workflow_executor_steps_prompt.py:314` on that property.

Constraints:

- The mechanism must survive `workflow_runner.py:281`'s in-place rename. A field that
  defaults to `False` and is set `True` only by `create_anonymous_workflow()` satisfies
  this; whatever shape you pick, the rename must not be able to clear it.
- **Do not change the behaviour of the other two call sites in this change.**
  `workflow_runner.py:467` is evaluated before the rename, so it is unaffected either
  way. `workflow_runner.py:528` (`is_anonymous_single_step`, which decides whether a
  `WorkflowOutputHandler` is created) _is_ evaluated after the rename, so switching it
  would change run-log output for every `#gh:` launch. That divergence is real and was
  caused by the same rename, but it is a separate, user-visible behaviour change: record
  it as a `PROPOSED FOLLOW-UP:` note instead of bundling it here.
- The reconciliation block already runs before `invoke_agent()`, so once the gate opens
  it corrects `model`, `llm_provider`, `model_alias`, `model_alias_trail`,
  `model_alias_origin`, and `reasoning_effort` from the authoritative `LaunchSelection`.
  No change to the block's body should be needed — verify that before editing it.
- Leave `exec_llm_provider` as-is. It stays the record of the execution-provider
  override seam and is written by `src/sase/llm_provider/_invoke.py:285`; this plan does
  not make it a display source.

## Verification

Every item below is required.

1. **Unit: `agy` detects the captured message.** Assert
   `detect_usage_limit("agy", <verbatim captured message>)` returns a detection, and
   that it resolves a reset-hint duration of ~4h15m (`used_reset_hint` true,
   `disable_seconds` ≈ `4*3600 + 14*60 + 60`). Use the message verbatim, including the
   trailing `50s` that the duration regex drops, so the test pins real-world input
   rather than a tidied paraphrase. Add it beside the existing usage-limit tests
   (`tests/` already covers this module; follow the file naming used there).
2. **Unit: end-to-end enforcement.** Assert that an `agy` failure carrying this message
   causes `handle_possible_usage_limit()` to write an active provider disable, and that
   `resolved_target_is_available()` then reports the `agy` member of the `@small` pool
   unavailable — i.e. the pool actually stops handing it out. This is the property that
   was missing in production; a pattern-match assertion alone does not prove it.
3. **Regression: reconciliation through the rename path.** Add a test that drives the
   launch through `execute_workflow()` (not a directly constructed `WorkflowExecutor`)
   with a prompt shaped like `#gh:<project>` — the shape that triggers the
   `has_prompt_part()` rename — writes a non-consuming preview first via
   `extract_directives_and_write_meta()`, and then asserts that `agent_meta.json`'s
   `model` and `llm_provider` equal the provider/model in the `LaunchSelection` handed
   to `invoke_agent()`. Confirm this test **fails before** the Work item 2 change and
   passes after; a regression test for this defect that passes on unfixed code is
   worthless, since that is exactly how the existing test missed it.
   `tests/test_pooled_alias_single_consumption.py` is the natural home and its
   `_run_composed_launch()` helper is the model to adapt.
4. **Behaviour-preservation check.** Confirm the two untouched `is_anonymous()` call
   sites in `workflow_runner.py` still behave as before — in particular that run-log
   output handling for a `#gh:` launch is unchanged.
5. `just install` first (workspaces are ephemeral and dependencies may have moved), then
   `just check`. Because this touches the LLM provider and xprompt executor cores, run
   `just check-full` through `/sase_monitor` with a `--next` action rather than inline.

## Done means

- A live Antigravity quota failure auto-disables `agy` for the provider-reported reset
  window, notifies, and removes it from `@small` / `@xsmall` until the window expires.
- For a `#gh:` launch of a pooled alias, `sase agent show`, `sase agent list`, the ACE
  row badge, agents_sync pages, and the Launch Control alias-history panel all name the
  provider and model that actually answered.
- Both defects have a regression test that fails on the pre-fix code.
