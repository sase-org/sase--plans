---
tier: tale
title: Parse Claude "<date> at <time>" usage-window reset expressions
goal:
  The ACE usage indicator shows a real reset countdown for every Claude window on hosts
  whose Claude CLI renders resets as "Sep 12 at 10am", and a future reset-format drift
  names the offending text in its own diagnostic.
size: small
proposed_by: bbugyi200.kellys_mbp.09.f0
create_time: 2026-09-12 09:21:49
status: wip
---

# Plan

## 1. Problem

On this Mac every Claude usage window renders its reset countdown as `?`, while Codex
and Grok windows render real countdowns. The same SASE commit and the same Claude CLI
version (`2.1.269`) behave correctly on the `athena` host.

Live evidence from `sase usage list` before the fix:

```
provider provider=claude status=ok ... reason=parse_error diagnostic="some Claude usage rows could not be parsed"
window provider=claude key=session               ... used_percent=18  reset=unknown
window provider=claude key=weekly                ... used_percent=94  reset=unknown
window provider=claude key=weekly:claude-fable-5 ... used_percent=100 reset=unknown
window provider=codex  key=codex:primary         ... reset="in 6d"
```

### 1.1 Root cause

The two hosts' Claude CLIs render the reset expression differently:

| Host   | Rendered reset row                                                     |
| ------ | ---------------------------------------------------------------------- |
| mac    | `Current session: 18% used · resets Sep 12 at 10am (America/New_York)` |
| athena | `Current session: N% used · resets Sep 12, 10am (America/New_York)`    |

`src/sase/llm_provider/usage/_claude_support_windows.py` captures the reset text
correctly in both cases, so the percentages, window keys, and scopes are all parsed
fine. The failure is one layer down, in `parse_claude_reset_timestamp`
(`src/sase/llm_provider/usage/_claude_support_reset_time.py`).

`_MONTH_RESET_RE` allows only an optional comma between the date and the time:

```
r"^([A-Za-z]{3,9})\s+(\d{1,2})(?:st|nd|rd|th)?,?\s+"
r"(?:(\d{4}),?\s+)?(\d{1,2})(?::(\d{2}))?\s*(am|pm)"
```

Against `Sep 12 at 10am (America/New_York)` the `(\d{1,2})` hour group has to match the
literal `at`, so the match fails and the parser returns `None`. The collector then
stores `resets_at: null`, marks the observation `completeness: partial` with
`reason_code: parse_error`, and the Rust projection classifies the window's
`reset_state` as `unknown`. `format_usage_countdown`
(`src/sase/ace/tui/widgets/_usage_indicator_format.py`) deliberately renders that
unknown state as `?`.

So the `?` is correct behavior over bad input; the defect is purely the parser's
rejection of the `at` connector.

### 1.2 Non-goals

- **`duration_seconds` is not involved.** Claude windows always store
  `duration_seconds: None`; the trailing indicator field is the countdown until reset,
  derived from `resets_at`. Do not change `duration_seconds`.
- **Do not change `format_usage_countdown`.** Rendering `?` for an unknown reset is the
  intended contract and must keep working for genuinely unparseable text.
- **Do not move this logic into `sase-core`.** Grok's normalization lives in Rust
  (`crates/sase_core/src/provider_usage/grok.rs`), but Claude has no Rust counterpart
  and all Claude `/usage` prose parsing is Python today. Migrating it is separate work;
  see §5.

## 2. Step 1 — Accept the `at` connector in the reset parser

File: `src/sase/llm_provider/usage/_claude_support_reset_time.py`

Rather than threading an optional `at` group through all three patterns, normalize the
connector away once, immediately after `normalize_text` in
`parse_claude_reset_timestamp`. This keeps `_ISO_RESET_RE`, `_MONTH_RESET_RE`, and
`_TIME_RESET_RE` unchanged and fixes every form in one place.

Add two module-level patterns and a small private helper:

```python
_AT_BETWEEN_RE = re.compile(r"(?<=\d)\s+at\s+(?=\d)", re.IGNORECASE)
_LEADING_AT_RE = re.compile(r"^at\s+", re.IGNORECASE)


def _strip_at_connector(text: str) -> str:
    """Drop the optional ``at`` word Claude places before a reset time."""
    return _LEADING_AT_RE.sub("", _AT_BETWEEN_RE.sub(" ", text), count=1)
```

Then apply it in `parse_claude_reset_timestamp` after the existing empty check and
before the first `_ISO_RESET_RE.match(...)` call:

```python
normalized = _strip_at_connector(normalized)
```

The digit lookbehind/lookahead keeps the substitution anchored between a date and a
time, so no month name, zone name, or unrelated prose is touched. This behavior was
prototyped against every supported form and produces:

| Input                                    | After stripping                  | Matched pattern |
| ---------------------------------------- | -------------------------------- | --------------- |
| `Sep 12 at 10am (America/New_York)`      | `Sep 12 10am (America/New_York)` | `_MONTH`        |
| `Sep 12, 10am (America/New_York)`        | unchanged                        | `_MONTH`        |
| `Sep 12, 2026 at 8pm (America/New_York)` | `Sep 12, 2026 8pm (America/...)` | `_MONTH`        |
| `Sep 12, 2026, 8pm (America/New_York)`   | unchanged                        | `_MONTH`        |
| `Sep 12 at 10:30am`                      | `Sep 12 10:30am`                 | `_MONTH`        |
| `at 10am`                                | `10am`                           | `_TIME`         |
| `8pm`                                    | unchanged                        | `_TIME`         |
| `2026-09-08 at 18:30 UTC`                | `2026-09-08 18:30 UTC`           | `_ISO`          |
| `someday soon`                           | unchanged                        | none (`None`)   |

The last row matters: unparseable prose must still return `None`.

## 3. Step 2 — Name the offending expression in the parse-error diagnostic

This bug needed a two-host comparison to localize precisely because the stored
diagnostic was the generic `some Claude usage rows could not be parsed`. Make the next
format drift self-diagnosing through `sase usage list -v`.

### 3.1 `src/sase/llm_provider/usage/_claude_support_windows.py`

Change `parse_usage_windows` to return `tuple[list[dict[str, Any]], str | None]` instead
of `tuple[list[dict[str, Any]], bool]`. The second element is `None` when every row
parsed, and otherwise a bounded, single-line diagnostic:

- While looping, collect the raw reset expressions that `parse_claude_reset_timestamp`
  rejected into a local list; keep tracking non-reset row failures (bad percent values,
  negative percentages, duplicate keys, rows that fail `_USAGE_ROW_RE`) with the
  existing boolean.
- After the loop:
  - If any reset expression was rejected, build
    `f'unparsed Claude usage reset expression: "{first}"'`, appending
    `f" (+{len(rest)} more)"` when more than one was rejected.
  - Else if only non-reset failures occurred, keep the existing
    `"some Claude usage rows could not be parsed"` text.
  - Else return `None`.
- Pass the result through `bounded_probe_diagnostic` (already exported from
  `sase.llm_provider.usage.types`; extend the module's existing import from that module)
  so it stays single-line, redacted, and within the observation length bound.

### 3.2 `src/sase/llm_provider/usage/claude.py`

Update the single call site (around the
`windows, had_parse_error = parse_usage_windows(` block) to unpack the diagnostic and
drive all three fields from it:

```python
windows, parse_diagnostic = parse_usage_windows(
    result_text,
    observed_at=context.request_started_at,
)
...
"reason_code": "parse_error" if parse_diagnostic else None,
"diagnostic": parse_diagnostic,
"completeness": "partial" if parse_diagnostic else "complete",
```

`parse_usage_windows` has exactly one caller, so no other call site needs updating. It
is re-exported through `src/sase/llm_provider/usage/_claude_support.py`; the name and
`__all__` entry are unchanged, so that file needs no edit.

## 4. Step 3 — Regression tests

All test work goes in `tests/llm_provider/test_claude_usage.py`, whose module-level
`OBSERVED_AT` is `2026-09-07 18:00 America/New_York` and whose default timezone is
`America/New_York`.

1. **Parser unit coverage.** Add a test next to
   `test_claude_reset_parser_handles_time_date_year_rollover_and_iso` asserting
   `parse_claude_reset_timestamp` handles the `at` connector:
   - `"Sep 12 at 10am (America/New_York)"` at `OBSERVED_AT` → `2026-09-12 10:00` NY.
   - `"Sep 12, 2026 at 8pm (America/New_York)"` at `OBSERVED_AT` → `2026-09-12 20:00`
     NY.
   - `"Sep 12 at 10:30am"` at `OBSERVED_AT` → `2026-09-12 10:30` NY (no zone, local tz).
   - `"at 10am"` at `OBSERVED_AT` → `2026-09-08 10:00` NY (the time-only resolver rolls
     forward a day because 10am already passed on Sep 7).
   - `"someday soon"` still returns `None`.

2. **End-to-end fixture from the real Mac output.** Add a probe test using the exact
   text captured from this host, including the `·` (U+00B7) separator:

   ```
   You are currently using your subscription to power your Claude Code usage

   Current session: 18% used · resets Sep 12 at 10am (America/New_York)
   Current week (all models): 94% used · resets Sep 12 at 8pm (America/New_York)
   Current week (Fable): 100% used · resets Sep 12 at 8pm (America/New_York)
   ```

   Build it with the existing `_runner` / `collect_claude_usage` helpers and assert:
   - `observation["completeness"] == "complete"` and
     `observation["reason_code"] is None` (this is the assertion that fails today).
   - `sorted(windows) == ["session", "weekly", "weekly:claude-fable-5"]`.
   - `windows["session"]["resets_at"] == datetime(2026, 9, 12, 10, 0, tzinfo=ny).timestamp()`.
   - both weekly windows'
     `resets_at == datetime(2026, 9, 12, 20, 0, tzinfo=ny).timestamp()`.

   Assert window keys and `resets_at`, not display labels — the Fable window's label is
   cosmetically awkward today (see §5) and should not be pinned by this test.

3. **Keep the comma form covered.**
   `test_claude_usage_probe_parses_all_windows_and_one_cent_cap_argv` already pins the
   `Sep 7, 6:30pm` / `Sep 12, 2026, 8pm` forms that `athena` emits. Leave it unchanged;
   it is the guard against the `at` handling regressing the comma handling.

4. **Diagnostic assertion.** Extend
   `test_claude_usage_probe_returns_partial_for_unparseable_reset` to assert the new
   diagnostic names the offending text, e.g.
   `assert "someday soon" in observation["diagnostic"]`, alongside the existing
   `completeness`/`reason_code`/`resets_at` assertions.

## 5. Proposed follow-ups (do NOT implement here)

- **Cosmetic label duplication.** `_usage_window_identity` renders the
  `Current week (Fable)` row as `Claude weekly week (Fable)`. Unrelated to the reset
  bug; worth its own task bead.
- **Rust core boundary.** Claude usage normalization is entirely Python while Grok's
  equivalent lives in `sase-core` at `crates/sase_core/src/provider_usage/grok.rs`
  ("Probe transport stays in Python"). Aligning Claude with that precedent is a separate
  migration, not part of this fix.

## 6. Verification

1. `just install` if this workspace's virtualenv is stale, then `just check`. Hand it to
   `/sase_monitor` if it runs long.
2. End-to-end on this Mac, which is the host that reproduces the bug:

   ```bash
   sase usage refresh
   sase usage list
   ```

   Expect every `provider=claude` window line to show `reset="in Xh Ym"` (or `in Nd Xh`)
   instead of `reset=unknown`, and the `provider provider=claude` line to carry neither
   `reason=parse_error` nor the `some Claude usage rows could not be parsed` diagnostic.
   The refresh runs a real, budget-capped (`--max-budget-usd 0.01`) Claude probe.

3. Confirm in ACE that the usage indicator's trailing countdown field shows a duration
   for Claude rather than `?`.
