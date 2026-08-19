---
tier: tale
title: Auto-disable Claude on weekly-limit errors and honor the reset date
goal: 'When Claude Code reports a usage-limit failure such as "You''ve hit your weekly
  limit · resets Aug 22, 8pm (America/New_York)", SASE matches it, writes a temporary
  claude disable that expires at the parsed reset instant, and does not leave Claude in
  the alias pool until a human disables it by hand.

  '
size: medium
proposed_by: bbugyi200.athena.084
create_time: 2026-08-19 16:16:50
status: wip
---

# Plan: Auto-disable Claude on weekly-limit errors and honor the reset date

## Background: what actually happened

On 2026-08-19 agent `083` (`#gh:gh_bobs-org__bob-cli`, `%model` defaulting to
CLAUDE(opus)) failed after 9m52s with:

```
WorkflowExecutionError: Step 'main' failed: Error running LLM provider command (exit code 1)
stderr: [result] You've hit your weekly limit · resets Aug 22, 8pm (America/New_York)
output: I'll start by exploring the codebase ...
You've hit your weekly limit · resets Aug 22, 8pm (America/New_York)
```

Bytes of the trigger line (from
`~/.sase/projects/gh_bobs-org__bob-cli/artifacts/ace-run/202608/19/20260819153402/error_report.md`):
ASCII apostrophe in `You've`, U+00B7 MIDDLE DOT as the separator, compact `8pm` with
**no space** before the meridiem, month-day with no year, zone `(America/New_York)`. The
traceback unwinds through `src/sase/llm_provider/_invoke.py` (the `CalledProcessError`
handler that already calls `handle_possible_usage_limit`) and
`src/sase/llm_provider/claude.py` `_invoke_loop`. `agent_meta.json` records
`"exec_llm_provider": "claude"`.

The auto-disable did not fire:

- No `llm.usage_limit` notification was written for `claude`. The store
  (`~/.sase/notifications/notifications.jsonl`) has Codex usage-limit notifications from
  the same day, so the notifier itself works.
- An ACE screenshot ~2 minutes after the failure still showed Claude as available (no
  `CLAUDE off` pill).
- At 15:49 EDT the operator wrote a **manual** disable
  (`~/.sase/llm_provider_disables.json`, `source: "ace"`, `expires_at` exactly
  2026-08-22 20:00:00 America/New_York — **no** 60-second grace). A usage-limit write
  would have been `source: "usage_limit"` and `expires_at` 60 seconds later
  (`_RESET_GRACE_SECONDS`).

Claude stayed in the pooled aliases until that manual click. That is the failure this
plan closes.

## Root cause

Three stacked gaps. Matching the 083 string in isolation is not enough; the follow-up
agent must close all three.

### 1. Claude's pattern list is a stale label enumeration

`ClaudeCodeProvider.llm_default_usage_limit_config()`
(`src/sase/llm_provider/claude.py`) ships exact phrases:

- `you've hit your usage limit`
- `you've hit your weekly limit`
- `you've hit your session limit`
- `you've hit your opus limit`
- `you've hit your sonnet limit`
- `usage limit reached` / `claude usage limit reached`

The shipped Claude Code 2.1.235 binary (`~/.local/share/claude/versions/2.1.235`) builds
the hard-limit line as:

```js
function YZe(e, t, r, n) {
  let o = n?.progressSavedSuffix ? " · progress saved" : "";
  return `You've hit your ${e}${t}${o}`;
}
```

with label map

```
five_hour: "session limit"
seven_day: "weekly limit"
seven_day_opus: "Opus limit"
seven_day_sonnet: "Sonnet limit"
seven_day_overage_included: "Fable 5 limit"
overage: "usage credit limit"
```

plus additional prefixes collected in `GEn`:

- `You've hit your`
- `You've reached your`
- `You're out of usage credits`
- `Your org is out of usage`
- seat-type / admin-disabled / `$0` allocation variants
- `You've hit your monthly spend limit`
- `You've hit your monthly limit`

Two of those labels (`Fable 5 limit`, `usage credit limit`) were already called out as
uncovered in the `usage_limit_reset_timestamp` plan and are still uncovered. A
prefix-level match on the binary's own classifier list is the evidence-based way to stop
enumerating.

The 083 line **does** contain `you've hit your weekly limit`, so a pure substring match
of the current weekly pattern against the captured text succeeds. That is necessary but
not sufficient: the regression corpus never used this verbatim string (see gap 2), and
the only disable writer can fail open (see gap 3).

### 2. The >24h reset formatter does not emit the strings the tests used

Claude's `fW(epoch, withZone)` (the function the previous plan called `swe`) branches on
distance from now. For a `seven_day` limit the reset is more than 24h out, so it takes
the month-name branch:

```js
o
  .toLocaleString("en-US", {
    month: "short",
    day: "numeric",
    hour: "numeric",
    minute: mins === 0 ? undefined : "2-digit",
    hour12: true,
    // year only across a year boundary
  })
  .replace(/ ([AP]M)/i, (full, mer) => mer.toLowerCase()) +
  (withZone ? ` (${timeZone})` : "");
```

The replacement callback returns **only** the lowercased `am`/`pm`, so it **strips the
space** that `toLocaleString` inserted. Minutes of zero omit `:00`. The live 083 shape —
`resets Aug 22, 8pm (America/New_York)` — is exactly that branch.

The current tests and the comment on `claude.py` document a **different** string, with a
space: `resets Aug 20, 6:38 am (America/New_York)` and `resets Aug 20, 6 am (...)`.
Those strings are not what 2.1.235 emits. `_RESET_MONTH_DATE_RE` uses `\s*` before
`(am|pm)`, so compact `8pm` / `6:38am` already parse when the rest of the detector runs.
They are not in the captured corpus, so a later tightening of `\s*` to `\s+` would
silently drop this weekly-limit form back onto the 24h fallback — or, combined with gap
3, onto no disable at all.

`parse_reset_hint` also has no fallback when the keyword-anchored forms miss a date that
is sitting on the same line (`Aug 22, 8pm (America/New_York)`). The user-facing
requirement is to try hard to parse a date once a usage-limit pattern has already
matched.

### 3. The workflow-error path is not a disable writer, and Claude quota errors are not retryable

`handle_possible_usage_limit` (`src/sase/llm_provider/usage_limit_disable.py`) is the
only writer of `source="usage_limit"`. It is called from `_invoke.py`'s
`CalledProcessError` / `LLMInvocationError` handlers, and **swallows every exception**
so a detector bug cannot mask the provider error.

`handle_workflow_error` (`src/sase/axe/run_agent_exec_retry.py`) refreshes
`exec_llm_provider` and can _classify_ a usage-limit so it does not sleep through
retries — but:

- it never calls `handle_possible_usage_limit`;
- if no retry config matches, it `return "raise"` **before** the usage-limit
  classification block.

Claude's built-in retry patterns are `Prompt is too long`,
`socket connection was closed unexpectedly`, and `API Error`. A weekly-limit result
matches none of them. Codex usage-limit text often also matches Codex's `"rate limit"` /
`"429"` retry patterns, so Codex failures keep walking through `handle_workflow_error`.
Claude weekly-limit failures do not. If `_invoke`'s handler returns `None` (no match) or
swallows a write error, **nothing else writes the disable**. That matches 083: the
invoke handler ran (the traceback is the `raise LLMInvocationError` immediately after
it), no notification, no `usage_limit` record.

The follow-up agent must make the 083 string match **and** make a second writer attempt
the disable on the workflow-error path, first-writer so a successful `_invoke` write is
not extended or double-notified.

## Design decisions

**Broaden Claude patterns from the binary, do not keep enumerating labels.** The durable
signal is the `YZe` / `GEn` family, not today's seven-day wording. Positive patterns
should include the prefixes Claude itself uses to classify hard limits. Exclusions stay
the near-miss advisory/cooldown family, and must also cover the binary's fast-mode line
`You've hit your fast limit` (today's exclude `fast limit reached` does not suppress
that spelling).

**Do not guess at org/seat-type sentences beyond what `GEn` actually contains.** Add the
`GEn` prefixes and the `YZe` template prefix `you've hit your`. Leave long org/admin
sentences that are not in `GEn` as a `PROPOSED FOLLOW-UP:` if they are only visible as
fragments.

**Keep `parse_reset_hint`'s "commit to the first matched keyword form" invariant.**
Compact `8pm` after a month-day is a payload of form 2, not a new form. Add an
**unanchored** month-name / ISO fallback that runs only when **no** keyword form matched
at all, and only after `detect_usage_limit` already matched a usage-limit pattern (so
"Aug 22" in unrelated stderr cannot disable a provider by itself). Do not fall through
from a matched-but-unresolvable keyword form (unknown zone, day 32).

**Workflow-error backstop is a second call to the existing writer**, not a new store.
`try_disable_provider` / `try_disable_provider_until` already no-op when a window is
active.

**No feature flag.** This is a miss in an already-shipped auto-disable, not new
user-reaching behavior.

**No `sase-core` change.** Matching, parsing, and the backstop call stay in this repo.
The Rust disable store already accepts a 3-day `until` timestamp (Codex wrote one the
same morning).

## Scope

In scope: Claude usage-limit patterns, reset-hint parsing of the compact month-day form
(and a keyword-miss fallback), the workflow-error disable backstop, comments/docs that
currently advertise a spaced `6:38 am` string the binary does not emit, and regression
tests using the 083 capture.

Out of scope — record as `PROPOSED FOLLOW-UP:` notes rather than doing them here:

- Windowing `parse_reset_hint` around the matched pattern (already noted on the
  `usage_limit_reset_timestamp` plan).
- Org/admin `GEn` fragments that were not captured as a full live failure.
- Failover of the discovering launch onto another provider (unchanged: 083 still dies;
  subsequent launches benefit).
- Restarting or version-pinning a long-lived ACE process so its header pill matches a
  newer detector (operator concern, not this change).

## Work

### 1. Broaden Claude's built-in usage-limit patterns

In `src/sase/llm_provider/claude.py` `llm_default_usage_limit_config()`:

- Keep the existing exact phrases (additive, not a replacement) so current tests stay
  meaningful.
- Add evidence-based prefixes from 2.1.235: `you've hit your`, `you've reached your`,
  `you're out of usage credits`, `your org is out of usage`,
  `you've hit your monthly spend limit`, `you've hit your monthly limit`.
- Extend `exclude_patterns` so a broadened `you've hit your` cannot trip on fast-mode
  cooldown or the existing approaching/grace/close-to advisories. In particular add
  `you've hit your fast limit` (the binary's actual spelling); keep `fast limit reached`
  for the older captured cooldown string.
- Rewrite the comment above the hook. Document `YZe` / `XOt` / `GEn` / `fW`'s
  space-stripping replace, and that `seven_day` renders
  `resets <Mon> <D>, <h>[:mm]am|pm (<zone>)` with **no space** before the meridiem and
  no minutes when they are zero. Stop advertising `6:38 am` with a space as the live
  form.

Substring matching is case/apostrophe/whitespace-insensitive (`normalize_for_match` in
`src/sase/llm_provider/usage_limit_config.py`). Do not switch these to regex.

### 2. Honor compact month-day reset hints and fall back to a date scan

In `src/sase/llm_provider/usage_limit_config.py`:

- Keep `_RESET_MONTH_DATE_RE`'s `\s*` before `(am|pm)` — that is what makes `8pm` and
  `6:38am` parse. Add a comment citing `fW`'s replace callback so it is not "cleaned up"
  to `\s+`.
- After the existing keyword-anchored forms all miss (the function is about to
  `return None, None`), run one additional scan for a month-name datetime or ISO-ish
  timestamp **without** requiring `resets` / `try again`. Reuse
  `_resolve_month_name_datetime` / `_resolve_iso_datetime` and the same zone /
  year-inference / grace rules. This fallback is only reachable from
  `detect_usage_limit` after a pattern has already matched; do not export a looser
  public parse for unrelated callers if that would change
  `test_unparseable_text_returns_none`. Either gate the fallback on a keyword
  (`parse_reset_hint(..., allow_unanchored=False)` defaulting to today's behavior,
  `detect_usage_limit` passing `True`) or keep a private helper that
  `detect_usage_limit` calls when `parse_reset_hint` returns `(None, None)`.
- Do not violate the matched-but-unresolvable invariant: if form 2's regex matches and
  the zone is unknown, still return `(None, None)` and do **not** run the unanchored
  fallback.

No change to `min_disable_seconds` / `max_disable_seconds` / grace.

### 3. Attempt the disable again on the workflow-error path

In `src/sase/axe/run_agent_exec_retry.py` `handle_workflow_error`, after
`_refresh_execution_provider` and **before** the
`if not active_retry_cfg: return "raise"` bail-out, call `handle_possible_usage_limit`
with:

- `provider=tracker.execution_provider` when that is set (083 recorded `claude`);
- `error_text=error_str` (the wrapped `Step 'main' failed: ...` string, which still
  contains the weekly-limit line);
- `model=ctx.agent_model`;
- `artifacts_dir=state.current_artifacts_dir or ctx.artifacts_dir`.

When `execution_provider` is missing, skip this call rather than scanning every provider
— quoted another-provider prose must not change policy. The existing first-writer store
makes a duplicate of a successful `_invoke` write a silent no-op (no second
notification).

Do not change retry-vs-usage-limit precedence beyond this extra write attempt.

### 4. Docs and examples that still show a spaced `am`

Update the month-name examples so at least one advertised form is the live compact
spelling (`resets Aug 22, 8pm (America/New_York)`):

- `docs/llms.md` Reset-Hint Forms table (form 2 currently shows `9am`, which is compact;
  keep it and add a weekly-limit example with a month-day).
- `src/sase/default_config.yml` `honor_reset_hint` comment if it still implies a spaced
  meridiem.
- `src/sase/config/sase.schema.json` `honor_reset_hint` description, same correction.

No default-value changes.

## Verification

Every item below is required.

1. **Unit: the 083 capture matches and uses the reset hint.** Add the verbatim trigger
   line (and the full `_invoke` wrapper with the `[result]` prefix and the
   `Step 'main' failed: ...` wrapper) to the corpus in
   `tests/test_llm_provider_usage_limit_defaults.py`. Pin `now` to 2026-08-19 15:43:56
   America/New_York. Assert `detect_usage_limit("claude", <each shape>)` returns a
   detection with `matched_pattern` covering the weekly limit,
   `used_reset_hint is True`, `reset_hint` containing `Aug 22` and `8pm`, and
   `disable_seconds` ≈ the real gap to 2026-08-22 20:00 America/New_York (about 3.2
   days), **not** `86400`. Confirm this test would fail if `_RESET_MONTH_DATE_RE`
   required `\s+` before `am|pm`.
2. **Unit: compact `fW` spellings parse.** In
   `tests/test_llm_provider_usage_limit_reset_hint.py`, add cases for
   `resets Aug 22, 8pm (America/New_York)`, `resets Aug 20, 6:38am (America/New_York)`,
   and `resets Aug 20, 6am (America/New_York)` (no space, minutes omitted). Keep the
   existing spaced cases so both ICU shapes parse.
3. **Unit: unanchored fallback.** A usage-limit-matched message that names
   `Aug 22, 8pm (America/New_York)` **without** `resets`/`try again` still honors the
   instant when `detect_usage_limit` runs with `honor_reset_hint`. `parse_reset_hint` on
   that text with the public default still returns `(None, None)` so incidental dates do
   not parse. `resets Aug 32nd, 2026 6:38 AM` still returns `(None, None)` with no
   fallback.
4. **Negative: fast-mode and advisories still do not match** after the broadened
   `you've hit your` prefix — including the binary's
   `You've hit your fast limit · resets in 5m`, plus the existing approaching /
   grace-window / close-to cases.
5. **Positive: previously uncovered `YZe` labels.**
   `You've hit your Fable 5 limit · resets Aug 22, 8pm (America/New_York)` and
   `You've hit your usage credit limit · ...` match under `claude`.
6. **Enforcement: `handle_possible_usage_limit` writes `source="usage_limit"` until the
   parsed instant** for the 083 wrapper string, going through
   `try_disable_provider_until`. Home: `tests/test_llm_provider_usage_limit_disable.py`.
7. **Backstop: a Claude weekly-limit `WorkflowExecutionError` that never matched a retry
   pattern still writes the disable.** In
   `tests/test_axe_run_agent_exec_retry_usage_limits.py`, feed `handle_workflow_error`
   the wrapped 083 error with Claude's real retry config and
   `execution_provider="claude"`. Assert it returns `"raise"`, does not sleep, and that
   `handle_possible_usage_limit` ran (spy) or that the disable store contains `claude` /
   `usage_limit`.
8. `just install` first, then `just check`. Finish with `just check-full` through
   `/sase_monitor` (`TESTING` / `TESTED`) rather than inline; this touches LLM provider
   core and the agent retry loop.

## Done means

- Replaying 083's captured stderr against `claude` auto-disables Claude until 2026-08-22
  20:00 America/New_York plus the existing 60s grace, and emits one `llm.usage_limit`
  notification.
- The same disable is attempted from `handle_workflow_error` even though Claude's retry
  patterns do not match the error, so a swallowed `_invoke` handler cannot leave Claude
  enabled.
- Fast-mode cooldown and approaching/grace advisories still do not disable Claude.
  Codex, agy, and grok captured cases stay green.
- A later Claude label that still uses `You've hit your <label>` (Fable 5 limit, usage
  credit limit, monthly spend limit) matches without another pattern enumeration.
