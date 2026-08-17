---
tier: tale
title:
  Honor absolute provider reset timestamps instead of falling back to a flat 24h disable
goal:
  When a provider names the wall-clock instant its usage limit resets, the automatic
  provider disable ends at that instant — not 24 hours later — for every provider that
  reports one.
size: medium
proposed_by: bbugyi200.athena.04f
create_time: 2026-08-17 06:40:12
status: wip
---

# Plan: Honor absolute provider reset timestamps instead of falling back to a flat 24h disable

## Background: what actually happened

On 2026-08-17 a Codex agent (`sase-o8.2`) failed with:

```
[error] You've hit your usage limit. Visit https://chatgpt.com/codex/settings/usage to
purchase more credits or try again at Aug 20th, 2026 6:38 AM.
[turn.failed] You've hit your usage limit. Visit https://chatgpt.com/codex/settings/usage
to purchase more credits or try again at Aug 20th, 2026 6:38 AM.
```

Detection worked: `codex`'s `you've hit your usage limit` pattern matched, a usage-limit
disable was written, and the ACE header pill showed `CODEX off 23h44m`.

The disable duration is wrong. Codex named its reset instant — `Aug 20th, 2026 6:38 AM`,
roughly **three days out** — and SASE disabled for the flat 24h fallback instead. The
practical consequence is the inverse of the usual complaint: `codex` re-enters the
pooled aliases on Aug 18, every launch routed to it burns a full agent startup and dies
on the same limit, and each of those failures re-arms another 24h window. The provider
handed us the exact answer and we discarded it.

## Root cause

`parse_reset_hint()` (`src/sase/llm_provider/usage_limit_config.py:335`) recognizes
exactly three shapes, all defined at
`src/sase/llm_provider/usage_limit_config.py:281-308`:

| Constant                   | Requires                                    |
| -------------------------- | ------------------------------------------- |
| `_RESET_TIME_WITH_ZONE_RE` | `resets <h>[:<mm>]<am\|pm> (<Area>/<City>)` |
| `_RESET_TIME_RE`           | `resets <h>[:<mm>]<am\|pm>`                 |
| `_RESET_DURATION_RE`       | `resets in <dur>` / `try again in <dur>`    |

Every one of them requires a **digit immediately after the keyword**, and the only "try
again" spelling accepted is `try again **in**`. Codex says
`try again **at** Aug 20th, 2026 6:38 AM` — an absolute calendar date. No regex matches,
`parse_reset_hint` returns `(None, None)`, and `detect_usage_limit()`
(`src/sase/llm_provider/usage_limit_config.py:436-447`) leaves `disable_seconds` at the
`UsageLimitSettings.disable_seconds` default of `86400`
(`src/sase/llm_provider/usage_limit_config.py:36`).

This is a parser gap, not a Codex gap. Measured against the currently installed
providers, only one of the eight reset-hint shapes they actually emit parses today:

```
(None, None)                                <- Try again at Aug 20th, 2026 6:38 AM.
(None, None)                                <- Try again at 6:38 AM.
(None, None)                                <- ...weekly limit · resets Aug 20, 6:38 am (America/New_York)
(None, None)                                <- ...weekly limit · resets Aug 20, 6 am (America/New_York)
(1755383940.0, '6:38pm (America/New_York)') <- ...usage limit · resets 6:38pm (America/New_York)
(None, None)                                <- spend limit reached (monthly; resets 2026-08-20 06:38 UTC)
(None, None)                                <- resets at 8pm
(None, None)                                <- Your limit will reset at 3am (America/New_York)
```

The second-to-last line matters on its own: `resets at 8pm` is the example
`docs/configuration.md:1550` gives for what `honor_reset_hint` parses, and it does not
parse. The documented behavior and the implemented behavior have never agreed.

## Evidence: what each installed provider actually emits

Captured from the shipped binaries on this machine, not inferred. This is the same
standard the existing `codex`/`claude` patterns were built to (epic sase-n4 research).

**Codex** —
`.../@openai/codex/node_modules/@openai/codex-linux-x64/vendor/x86_64-unknown-linux-musl/bin/codex`
carries the sentence templates ` Try again at <X>.` and ` or try again at <X>.`, the
`chrono` format string `%b %-d<ord>, %Y %-I:%M %p`, a bare `%-I:%M %p`, and the ordinal
suffix table `stndrdth`. So Codex emits two shapes:

- `Try again at Aug 20th, 2026 6:38 AM.` — ordinal day, year always present, no comma
  before the time, uppercase meridiem
- `Try again at 6:38 AM.` — time only

Both are rendered from a local `DateTime` and carry **no timezone marker**.

**Claude** — `/home/bryan/.local/share/claude/versions/2.1.233` builds the suffix as
`` ` · resets ${swe(resetsAt, true)}` `` and appends it to
`` `You've hit your ${label}${suffix}` ``. `swe(epoch, withZone)` branches on distance:

- **more than 24h out** —
  `toLocaleString("en-US", {month:"short", day:"numeric", hour:"numeric", minute: mins===0 ? undefined : "2-digit", hour12:true})`,
  plus `year:"numeric"` only when the year differs, then
  `.replace(/ ([AP]M)/i, lowercase)` and a ` (${rBs()})` suffix where `rBs()` is
  `Intl.DateTimeFormat().resolvedOptions().timeZone`. Yields
  `resets Aug 20, 6:38 am (America/New_York)`, `resets Aug 20, 6 am (America/New_York)`
  when minutes are zero, and `resets Aug 20, 2027, 6:38 am (America/New_York)` across a
  year boundary.
- **within 24h** — `toLocaleTimeString(...)` yielding
  `resets 6:38pm (America/New_York)`.

Only the second branch parses today. The first branch is the one that fires for the
`seven_day` limits — the label map is
`{five_hour:"session limit", seven_day:"weekly limit", seven_day_opus:"Opus limit", seven_day_sonnet:"Sonnet limit", seven_day_overage_included:"Fable 5 limit", overage:"usage credit limit"}`,
which lines up with the patterns already in `src/sase/llm_provider/claude.py:163-172`.
**So Claude's weekly limit has exactly the same defect as Codex**: SASE detects it, then
disables for 24h instead of up to 7 days.

The same binary also emits a billing-error body containing
`` `spend limit reached (${period}; resets ${resetsAt.toISOString().slice(0,16).replace("T"," ")} UTC)` ``
— i.e. `resets 2026-08-20 06:38 UTC`.

**Antigravity (`agy`)** — `Resets in 4h14m50s`. Already parsed correctly by
`_RESET_DURATION_RE`; nothing to do.

**Grok** — `/home/bryan/.local/bin/grok`:
`You've reached your free Grok Build usage limit for now. Get SuperGrok for much higher limits, or try again later: <url>`
and `You hit your weekly limit.` Neither carries a reset instant.

**OpenCode** —
`Usage limit reached. To continue using this model now, enable usage from your available balance`.
No reset instant.

**Qwen** — no reset-bearing usage-limit prose.

So the reachable win is: Codex (both shapes), Claude (the >24h shape and the billing
shape), and the documented-but-broken `reset at` spelling. `agy` already works; `grok` /
`opencode` / `qwen` / `muse` have nothing to parse and correctly keep the flat fallback.

## Design decisions

**Fix the shared parser, not each provider.** A per-provider parsing hook is the wrong
shape here: the shapes are message _formats_, not provider identities, and Codex and
Claude already share the "month-name, day, clock time" format. One parser means every
provider that adopts a known format later is covered for free, and it keeps
`llm_default_usage_limit_config()` a pure pattern declaration.

**Zone-less absolute timestamps resolve in `sase.core.time.get_timezone()`.** Codex
renders local wall time with no marker. `get_timezone()` prefers the sase `timezone`
config key and otherwise resolves the host zone, which is what the CLI itself used. This
matches what `_RESET_TIME_RE` already does at
`src/sase/llm_provider/usage_limit_config.py:366-370`, so the assumption is not new.
Note in a comment that a sase `timezone:` deliberately set to something other than the
host zone would skew these parses.

**Leave `max_disable_seconds` at `604800`.** A genuine 7-day Claude reset produces
`604800 + _RESET_GRACE_SECONDS`, so the clamp trims the 60s grace and SASE re-enables
one minute early. The next launch fails and re-disables; the behavior is self-healing
and not worth widening a cap whose entire job is bounding untrusted provider input.

## Scope

In scope: reset-hint parsing, the multi-day rendering that becomes reachable because of
it, and the docs/schema text that describes the supported forms.

Out of scope — record as `PROPOSED FOLLOW-UP:` notes rather than doing them here:

- **Windowing the hint search around the matched pattern.** `parse_reset_hint` scans the
  whole error text. Broadening the anchors raises the odds that an unrelated
  `try again at …` elsewhere in a long stderr capture wins. The blast radius is bounded
  (detection only runs on text that already matched a usage-limit pattern, and
  `min`/`max_disable_seconds` clamp the result), but scoping the search to a window
  around the matched pattern is a real improvement and a separate change.
- **Two uncovered Claude limit labels.** The label map above contains
  `seven_day_overage_included: "Fable 5 limit"` and `overage: "usage credit limit"`;
  `src/sase/llm_provider/claude.py:163-172` covers neither. Do not guess at wording —
  note it with the captured map as evidence.
- **No failover for the discovering launch.** Unchanged from the `agy` plan: the agent
  that discovers the limit still dies. Only subsequent launches benefit.

Do **not** invent usage-limit patterns for `muse` / `opencode` / `qwen` here. Their
patterns stay as they are; this plan only changes how a reset hint is read once a
pattern has already matched.

## Work

### 1. Teach `parse_reset_hint` the absolute-timestamp forms

In `src/sase/llm_provider/usage_limit_config.py`, add two new forms and broaden the
anchor on the two existing clock-time forms. Resolution order, highest priority first:

1. **ISO-ish absolute** — `<YYYY>-<MM>-<DD>` followed by `T` or a space, `<HH>:<MM>`,
   optional `:<SS>`, optional `Z` / `UTC` / `±HH:MM`. Absent a zone marker, resolve in
   `get_timezone()`.
2. **Month-name absolute** — `<Mon> <D>[st|nd|rd|th][,] [<YYYY>[,]] <h>[:<mm>] <am|pm>`,
   with an optional trailing ` (<zone>)`. This single form must cover Codex's
   `Aug 20th, 2026 6:38 AM` and Claude's `Aug 20, 6:38 am (America/New_York)`,
   `Aug 20, 6 am (America/New_York)`, and `Aug 20, 2027, 6:38 am (America/New_York)`.
3. **Zoned clock time** — existing `_RESET_TIME_WITH_ZONE_RE`.
4. **Bare clock time** — existing `_RESET_TIME_RE`.
5. **Relative duration** — existing `_RESET_DURATION_RE`.

Constraints:

- **Preserve the "commit to the first matched anchor" invariant** documented in the
  `parse_reset_hint` docstring. If a form's regex matches but its payload does not
  resolve (bad month, day 32, unknown zone), return `(None, None)` — do not fall through
  to a lower-priority form. The existing
  `test_unknown_zone_does_not_fall_back_to_local_time` pins this and must stay green.
- **Broaden the anchor on forms 2, 3 and 4** to accept `at` / `on` after the keyword and
  to accept `try again` as a keyword alongside `reset`/`resets`. Something shaped like
  `(?:resets?|try\s+again)\s+(?:at\s+|on\s+)?` is sufficient. This is what makes
  `resets at 8pm`, `Your limit will reset at 3am (America/New_York)` and Codex's
  `Try again at 6:38 AM` work. Keep requiring the numeric/month payload immediately
  after the anchor so incidental prose (`connection reset by peer`, `try again later:`)
  cannot match.
- **Zone suffix parsing must accept `UTC`.** `_RESET_TIME_WITH_ZONE_RE`'s zone group
  requires at least one `/`, which is right for `America/New_York` but rejects the bare
  `UTC` that Claude's billing body emits.
- **Year inference for year-less dates.** Claude omits the year in the common case.
  Choose the year from `{now.year - 1, now.year, now.year + 1}` that minimizes
  `abs(candidate - now)`, skipping candidates that are not valid dates (Feb 29). This
  gets `Aug 20` right on Aug 17, gets `Jan 2` right on Dec 30, and — importantly — keeps
  a reset that just slipped a few minutes into the past as a few-minutes-ago instant
  (clamped up to `min_disable_seconds`) instead of rolling it a full year forward and
  clamping to the 7-day maximum.
- **An explicit year is used as given.** A past explicit year yields a negative
  duration, which `detect_usage_limit` already clamps to `min_disable_seconds`.
- `_RESET_GRACE_SECONDS` applies to the new forms exactly as it does to the existing
  ones.
- The returned display hint should echo the matched substring
  (`Aug 20th, 2026 6:38 AM`), since it is surfaced verbatim in the notification via
  `src/sase/notifications/senders.py:352`.
- Normalization already collapses newlines, so a message the provider wrapped across
  lines parses the same as a single-line one. Do not add separate multi-line handling.

### 2. Record the captured formats next to the providers they belong to

Comment-only, but required — this is what stops the next agent re-deriving it from
binaries.

- `src/sase/llm_provider/codex.py:311` — note that Codex emits
  `Try again at %b %-d<ord>, %Y %-I:%M %p` and a bare `%-I:%M %p`, both in local time
  with no zone marker, and that no pattern change is needed because
  `you've hit your usage limit` already matches the carrier sentence.
- `src/sase/llm_provider/claude.py:154` — note the ≤24h / >24h branch in the shipped
  formatter, that the >24h branch is what `seven_day` limits produce, and the
  `resets <ISO> UTC` billing shape.
- `src/sase/llm_provider/agy.py:392` — the existing comment already cites
  `Resets in 4h14m50s`; leave it alone beyond confirming it still reads true.

No `llm_default_usage_limit_config()` return value changes in this plan.

### 3. Render multi-day disable windows with a day unit

`format_remaining_until` (`src/sase/ace/tui/widgets/_override_pill.py:44`) tops out at
hours, so the Codex window from the background section renders as `CODEX off 76h22m` in
the ACE header pill (`src/sase/ace/tui/widgets/provider_disables_indicator.py:84`) and
`76h22m left` in its tooltip (`:109`). Multi-day windows were unreachable before this
change and become routine after it. Add a day unit (`3d4h`) in the same compact style.

Constraints:

- The helper is shared with temporary LLM override pills. Changing it changes those too;
  that is the intended, consistent outcome, but check
  `tests/test_provider_disables_indicator.py` and the override-pill tests for assertions
  on the hour-only format.
- `_format_duration` in `src/sase/notifications/senders.py:288` already emits `3d 4h`;
  match its unit choices, not its spacing (the pill format is unspaced).
- If any PNG snapshot in `tests/ace/tui/visual/snapshots/png/` renders a >24h duration
  the goldens will move. Check first; refresh with `--sase-update-visual-snapshots` only
  if the diff is exactly this change.

### 4. Make the documentation describe what the parser does

- `docs/configuration.md:1550` — replace the `honor_reset_hint` description with the
  forms actually supported after Work item 1. The current example (`resets at 8pm`) is
  the one that has never worked.
- `src/sase/default_config.yml:944` — same correction to the inline comment.
- `src/sase/config/sase.schema.json:2147` — same correction to the `honor_reset_hint`
  description.

No default values change in any of the three.

## Verification

Every item below is required.

1. **Unit: each captured shape parses.** In `TestParseResetHint`
   (`tests/test_llm_provider_usage_limit_config.py:388`), add a case per shape listed in
   the "Evidence" section, using the strings **verbatim** — ordinal suffix, uppercase
   `AM`, lowercase `am`, missing minutes, missing year, `(America/New_York)`, bare
   `UTC`. Pin a fixed `now` and a fixed timezone the way the existing cases do.
2. **Unit: the year-inference rule.** Cover, at minimum: `Aug 20` from Aug 17 same year;
   `Jan 2` from Dec 30 resolving to next year; and a hint a few minutes in the past
   resolving to a few minutes in the past, **not** ~1 year out. The third case is the
   one that fails loudly if year inference is written as a naive roll-forward.
3. **Unit: no fall-through on a matched-but-unresolvable payload.** e.g.
   `resets Aug 32nd, 2026 6:38 AM` returns `(None, None)`.
4. **Regression: the reported failure, end to end.** Assert that
   `detect_usage_limit("codex", <the verbatim message from the Background section>, now=<fixed epoch shortly before 2026-08-20 06:38 local>)`
   returns a detection with `used_reset_hint is True` and `disable_seconds` ≈ the real
   gap to that instant (~3 days), **not** `86400`. Add the message to the captured
   corpus in `tests/test_llm_provider_usage_limit_defaults.py:19-40` alongside the
   existing `_CODEX_UPGRADE_TO_PRO` entry. Confirm this test fails on the pre-fix code.
5. **Regression: Claude's weekly limit.** The same assertion for
   `You've hit your weekly limit · resets Aug 20, 6:38 am (America/New_York)` under
   `claude`. This is the "equivalent fix for other providers" claim; it must be proven,
   not asserted in a comment.
6. **Regression: `agy` is unchanged.**
   `test_agy_captured_failure_uses_reset_hint_duration`
   (`tests/test_llm_provider_usage_limit_defaults.py:179`) must still pass untouched —
   the new higher-priority forms must not steal the duration form's match.
7. **Enforcement: the disable window actually reaches the store.** Assert that
   `handle_possible_usage_limit()` on the Codex message writes a disable whose
   `expires_at` lands at the parsed instant (within the grace buffer), not
   `now + 86400`, and that it goes through the `try_disable_provider_until` branch at
   `src/sase/llm_provider/usage_limit_disable.py:74`.
   `tests/test_llm_provider_usage_limit_disable.py` is the home.
8. **Rendering.** A disable ~3 days out renders with a day unit in both the ACE pill and
   its tooltip.
9. `just install` first (workspaces are ephemeral and dependencies may have moved), then
   `just check`. Because this touches the LLM provider core and an ACE widget, finish
   with `just check-full` through `/sase_monitor` with a `--next` action rather than
   inline; add `just test-visual` if Work item 3 moved any golden.

## Done means

- A Codex failure naming `try again at <date> <time>` disables `codex` until that
  instant — for the reported failure, ~3 days rather than 24 hours — and the ACE pill
  shows the window with a day unit.
- A Claude `seven_day` failure naming `resets <Mon> <D>, <h>:<mm> <am|pm> (<zone>)`
  disables `claude` until that instant, up to the 7-day cap.
- `agy`'s duration hint, and every currently-passing usage-limit test, behave exactly as
  before.
- Every example the docs and schema give for `honor_reset_hint` actually parses.
- Both new-shape regressions fail on the pre-fix code.
