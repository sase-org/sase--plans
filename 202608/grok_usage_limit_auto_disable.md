---
tier: tale
title:
  Auto-disable the grok provider when Grok Build reports its usage balance exhausted
goal:
  A Grok Build usage-limit failure writes a `usage_limit` provider disable
  automatically, sized for Grok's weekly usage pool, so later launches route around grok
  instead of burning agent startups on the same 402.
size: medium
proposed_by: bbugyi200.athena.05u
create_time: 2026-08-18 07:43:48
status: wip
---

# Plan: Auto-disable the grok provider when Grok Build reports its usage balance exhausted

## Background: what actually happened

On 2026-08-18 three grok agents failed back to back — 07:00:22, 07:21:03 and 07:24:45
EDT — each with the same stderr:

```
Error running LLM provider command (exit code 1)
stderr: Error: Internal error: {
  "message": "API error (status 402 Payment Required): Grok Build usage balance exhausted",
  "http_status": 402,
  "promptUsage": { ... }
}
```

Transcripts:
`~/.sase/chats/202608/gh_sase_org__sase-gh-workflow_gh_main_ERROR-260818_065135.md`,
`…_072101.md`, `…_072442.md`. Grok's own log records the same failure as
`shell.turn.inference_failed` with `status_code: 402`, `is_retryable: false`
(`~/.grok/logs/unified.jsonl`, `sid` `3e74946a…`).

Nothing classified it. No `usage_limit` disable was written, so every subsequent launch
routed to grok started an agent, spent a workspace, and died on the identical 402. The
only grok record in `~/.sase/llm_provider_disables.json` carries `source: "ace"` — the
operator disabled grok by hand from Launch Control after the third failure. The
subsystem that exists precisely to do this automatically never fired.

## Root cause

`GrokProvider.llm_default_usage_limit_config()`
(`src/sase/llm_provider/grok.py:169-180`) declares exactly three patterns, all derived
from one captured **free-tier** message:

```python
patterns=[
    "reached your free grok build usage limit",
    "usage limit for now",
    "get supergrok for much higher limits",
]
```

The failure above is the **paid credit/balance** family and shares no substring with any
of them. So `find_matching_pattern()` returns `None`, `detect_usage_limit()`
(`src/sase/llm_provider/usage_limit_config.py:578`) returns `None`, and
`handle_possible_usage_limit()` — which the invocation error path does call correctly
(`src/sase/llm_provider/_invoke.py:367,396`) — returns `None` without writing anything.

This is a coverage gap in one provider's pattern list, not a defect in the detection,
enforcement, precedence, or notification machinery. All of that already works; it was
simply never handed a matching string.

Worth noting because it explains why the failure was _entirely_ unclassified: grok's
retry patterns did not match either. `llm_default_retry_config()`
(`src/sase/llm_provider/grok.py:157-162`) looks for `"xAI API error"`,
`"xAI rate limit"`, `"xAI server error"`, `"xAI upstream request failed"`, and the CLI
emits `API error (status 402 …)` with no `xAI` prefix. That is a separate bug; see
Scope.

## Evidence: Grok Build never reports a reset instant

The prompt asks that the disable duration come from "the date and time provided by
grok's error output". It does not provide one, and this is worth establishing firmly
rather than assuming, because it decides the whole duration design.

A full literal scan of the shipped binary (`~/.grok/downloads/grok-1.0.5-linux-x86_64`,
the target of `~/.grok/bin/grok`) finds:

- **Zero** occurrences of `resets at`, `resets in`, or `Resets in`. Every limit message
  in the binary ends in `try again later` with no instant attached.
- The only `Retry-After` literal belongs to `xai-file-utils`' S3 storage client
  (`crates/codegen/xai-file-utils/src/storage_client.rs`), not the sampling path.
- The `expires_at: "2026-08-18T13:46:54…+00:00"` field in the `turn.terminal_failure`
  log record is the **OIDC token expiry** — the same record carries `auth_mode: "Oidc"`
  and `key_prefix` — not a quota reset. Do not treat it as one.

This corroborates the grok note already recorded in
`sase/repos/plans/202608/usage_limit_reset_timestamp.md`, which reached the same
conclusion from the 1.0.3 binary.

The limit-message family the binary does carry, verbatim:

| String                                                                                           | Where it lives                          |
| ------------------------------------------------------------------------------------------------ | --------------------------------------- |
| `Grok Build usage balance exhausted` (as `API error (status 402 Payment Required): …`)           | sampling error path — **captured live** |
| `usage balance exhausted`, `out of credits`, `spending limit`, `usage limit reached`             | shell billing-error classifier          |
| `out of credits or over your spending limit. Add credits and retry`                              | shell prose                             |
| `You've reached your free Grok Build usage limit for now. Get SuperGrok for much higher limits…` | shell prose — already matched           |
| `You hit your free usage limit.`, `You hit your weekly limit.`                                   | pager billing panel                     |
| `You've hit the credit limit for your plan.`, `Purchase credits to keep using Grok Build`        | pager billing panel                     |
| `You've hit the rate limit for your plan. Upgrade your account or try again later.`              | pager — **transient, not a quota**      |
| `You've hit your team's API rate limit. Ask a team admin to purchase more credits…`              | pager — **transient, not a quota**      |

Why the durations matter: per xAI's own FAQ (`docs.x.ai/grok/faq`, quoted with
provenance in `sase/repos/research/202608/supergrok_subscription_tiers/`), since June
2026 paid Grok meters every product — Build included — against **one shared weekly usage
pool**, and the reset schedule is visible only in Settings → Usage. The binary's
`You hit your weekly limit.` string is the same regime. So an exhausted balance is
typically a multi-day condition, and the global 24h fallback under-shoots it.

## Design decisions

**Key the new patterns to the API error, not the TUI prose.** SASE only ever sees what
reaches stderr in headless mode. `usage balance exhausted` and the
`status 402 payment required` transport form are both confirmed present in a real
captured failure; the pager strings are added on lower confidence and must be commented
as such, exactly the way `opencode.py` and `muse.py` already flag their unverified
baselines.

**Do not classify grok's rate-limit family as a usage limit.**
`You've hit the rate limit for your plan` and the team API rate-limit message are
throttling, not quota exhaustion — disabling the provider for hours on a per-minute RPM
bounce would be strictly worse than the status quo. They must keep flowing to
retry/fallback, and a negative test must pin that.

**No reset-hint work.** `honor_reset_hint` stays on and costs nothing:
`parse_reset_hint` simply finds nothing in grok's text and returns `(None, None)`. If
xAI ever adds an instant, the shared parser already handles every common shape and grok
inherits it for free. This plan adds no grok-specific parsing.

**Give grok a per-provider `disable_seconds` of 48h (172800).** With no instant to
parse, the duration is a judgement call between two costs:

- _Too short_ — the window expires while the weekly pool is still empty. Every agent
  launched into grok at that moment burns a workspace and dies. With this operator's
  parallel launch pattern that is several agents per re-probe, not one.
- _Too long_ — grok stays out of the pooled aliases after the pool has actually reset.

48h halves the number of re-probes versus the 24h global default while never locking
grok out more than two days past a real reset, and the operator can clear it instantly
from Launch Control (`,m`) the moment Settings → Usage shows headroom. This is the one
number in the plan worth arguing about at approval time; it is a single field, tunable
without a code change via `llm_provider.usage_limit.providers.grok.disable_seconds`.

Note that `min_disable_seconds`/`max_disable_seconds` clamp only reset-hint-derived
durations, so the administrator-chosen 172800 is used exactly as configured
(`src/sase/llm_provider/usage_limit_config.py:628`).

**grok becomes the first provider to set `disable_seconds`.** The config field, the
merge precedence, the docs and the schema all already support it; nothing new is
required in `usage_limit_config.py`.

## Scope

In scope: `grok`'s built-in usage-limit config, its regression corpus, one enforcement
test, and the documentation that describes per-provider duration overrides.

Out of scope — record each as a `PROPOSED FOLLOW-UP:` note rather than doing it here:

- **Grok's stale retry patterns.** `"xAI API error"` / `"xAI rate limit"` /
  `"xAI server error"` / `"xAI upstream request failed"`
  (`src/sase/llm_provider/grok.py:157-162`) do not match what Grok Build 1.0.5 actually
  emits (`API error (status <code> …)`,
  `Model is temporarily overloaded. Try again in a moment.`), so genuinely transient
  429/5xx grok failures are almost certainly not being retried today. That is a real bug
  with its own blast radius and deserves its own task bead — fixing retry patterns in
  the same change as usage-limit patterns muddies which classifier a test is proving.
- **Adaptive backoff for providers that report no reset instant.** The principled answer
  to "how long?" when the provider never says is to escalate: 6h, then 12h, 24h, 48h,
  capped at 7d, resetting after a clean run. That needs per-provider disable _history_,
  and the disable store is Rust-backed in `../sase-core`; per the repo's Rust core
  boundary rule that is cross-repo epic work, not a tale.
- **The discovering agent still dies.** Unchanged from the prior usage-limit plans. Only
  later launches benefit — except where `run_agent_exec_retry` has an enabled fallback
  provider to route to, which it already handles.
- **Other providers.** Do not touch `claude`, `codex`, `agy`, `qwen`, `opencode`, or
  `muse` patterns here.

## Work

### 1. Extend `grok`'s built-in usage-limit config

`src/sase/llm_provider/grok.py:169-180`. Keep all three existing free-tier patterns —
the merge is additive and that message is still real — and add the credit/balance
family. Patterns are case/apostrophe/whitespace-normalized substrings; write them
lowercase to match the file's existing style.

Add:

| Pattern                                      | Confidence                                            |
| -------------------------------------------- | ----------------------------------------------------- |
| `usage balance exhausted`                    | captured live 2026-08-18; also the classifier literal |
| `status 402 payment required`                | captured live; transport form of the same failure     |
| `out of credits or over your spending limit` | shell prose in the binary                             |
| `you've hit the credit limit for your plan`  | pager string, unverified on stderr                    |
| `you hit your weekly limit`                  | pager string, unverified on stderr                    |
| `usage limit reached`                        | grok's own billing-error classifier keyword           |

Set `disable_seconds=172800`.

Do **not** add `exclude_patterns`: none of the above is a substring of grok's rate-limit
messages, so no exclusion is needed to keep those on the retry path. Verification item 4
proves that rather than assuming it.

Replace the existing comment with one that records: the verbatim captured 402 message
and its date; that the last three patterns come from binary strings not yet observed on
stderr; that Grok Build emits **no** reset instant in any limit message (so
`honor_reset_hint` is a no-op here and the flat duration is load-bearing); and why the
duration is 48h rather than the 24h default. Cite the weekly-pool mechanic. This comment
is the artifact that stops the next agent re-deriving all of it from a 167MB binary.

### 2. Add the captured failure to the regression corpus

`tests/test_llm_provider_usage_limit_defaults.py`. Add a module-level constant next to
`_GROK_USAGE_LIMIT` (`:37-41`) holding the **verbatim** stderr from the Background
section, JSON braces and newlines intact — normalization collapses the whitespace, and
using the real multi-line shape is what proves that.

Add to `TestGrokBuiltInDefaults` (`:188`):

- the new message matches;
- `_GROK_USAGE_LIMIT` still matches (the additive merge did not regress the free tier);
- `config.disable_seconds == 172800`.

### 3. Prove the duration and the disable actually reach the store

`tests/test_llm_provider_usage_limit_disable.py`. Following the shape of
`test_codex_reset_at_date_failure_writes_until_disable_at_parsed_instant` (`:106`), add
a test that calls
`handle_possible_usage_limit(provider="grok", error_text=<verbatim 402>)` with a pinned
`now` and asserts:

- a detection is returned with `used_reset_hint is False` and `expires_at is None` —
  grok gives no instant, and this is the assertion that fails loudly if someone later
  mistakes the OIDC `expires_at` for a reset;
- the stored disable has `source == "usage_limit"` and expires at `now + 172800`,
  **not** `now + 86400`.

Note that this test must exercise the `try_disable_provider` branch
(`src/sase/llm_provider/usage_limit_disable.py:81`), not the `…_until` branch.

Also add `"grok"` to the `registered_providers` fixture if the new test needs it — it is
already listed at `:41`.

### 4. Pin the negative case

In `TestGrokBuiltInDefaults`, assert that
`You've hit the rate limit for your plan. Upgrade your account or try again later.` and
`You've hit your team's API rate limit. Ask a team admin to purchase more credits for higher limits, or try again later.`
both return `is_usage_limit_error(...) is False`. This is the guard that keeps transient
throttling on the retry path.

### 5. Documentation

- `docs/configuration.md:1592` — the `providers.<provider>.disable_seconds` row
  currently documents the field abstractly. Note that `grok` ships a non-null built-in
  default (48h) because Grok Build reports no reset instant and meters against a weekly
  pool.
- `docs/llms.md` (Usage-Limit Auto-Disable, around `:1926-1946`) — the bullet describing
  `providers.<provider>.disable_seconds` should say the same thing in one sentence.
- `src/sase/default_config.yml:1049` — the commented `disable_seconds: null` example
  says "null inherits the global value"; that stays true. Add a one-line note that a
  provider plugin may ship its own non-null default, which user config then overrides.

No default _values_ change in the config file or schema — grok's default lives in the
plugin, which is where every other built-in usage-limit default already lives.

## Verification

1. **The reported failure now disables grok.** Work items 2 and 3 must both fail on the
   pre-fix tree. Confirm that explicitly rather than assuming it — the pattern list is
   the entire fix, so a test that passes before the change is testing nothing.
2. **The free-tier message still matches.**
   `test_matches_captured_live_failure_with_curly_apostrophe`
   (`tests/test_llm_provider_usage_limit_defaults.py:197`) stays green untouched.
3. **No other provider moved.** The rest of
   `tests/test_llm_provider_usage_limit_defaults.py` and all of
   `tests/test_llm_provider_usage_limit_disable.py`, `tests/test_provider_disable.py`,
   `tests/llm_provider/test_provider_disable_smoke.py` and
   `tests/fakey/test_usage_limit_e2e.py` pass unchanged. Detection is provider-scoped,
   so a grok pattern must not be reachable from another provider's failure.
4. **Rate-limit text stays retryable.** Work item 4.
5. **Provider metadata still serializes.** `default_usage_limit_config` is dumped into
   provider metadata at `src/sase/llm_provider/_registry_metadata.py:92`; a non-null
   `disable_seconds` is a new value in that dict.
   `tests/llm_provider/test_grok_provider_core.py` and the agent-CLI metadata consumers
   (`src/sase/agent_clis/operations.py:48,121`) must stay green.
6. `just install` first — workspaces are ephemeral and dependencies may have moved —
   then `just check`. This touches the LLM provider core, so finish with
   `just check-full` through `/sase_monitor` with a `--next` action rather than inline.
   No ACE rendering changes here, so `just test-visual` is not required unless
   `just check-full` says otherwise.

## Done means

- A Grok Build
  `API error (status 402 Payment Required): Grok Build usage balance exhausted` failure
  writes a `source: "usage_limit"` disable automatically, and the ACE provider-disables
  indicator shows grok off for ~48h without anyone touching Launch Control.
- The three 2026-08-18 failures would have become one failure plus a disable, instead of
  three identical failures.
- Grok's free-tier usage-limit message and every other provider's detection behave
  exactly as before.
- The code comment records, verbatim and with provenance, that Grok Build reports no
  reset instant — so the next agent does not go looking for one.
- The stale grok retry patterns and the adaptive-backoff idea are captured as
  `PROPOSED FOLLOW-UP:` notes rather than silently dropped.
