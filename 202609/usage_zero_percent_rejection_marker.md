---
tier: tale
title: Drop the redundant rejection marker from exact-0% usage badges
goal:
  An exhausted (exact 0%) usage window in the ACE top bar renders identically whether or
  not its vendor reports a rejected state, so the Codex 0% badge looks like the Grok 0%
  badge with no exclamation point.
size: small
proposed_by: bbugyi200.apollo.1j.w0
create_time: 2026-09-22 15:56:42
status: wip
---

# Plan: Drop the redundant `!` rejection marker from exact-`0%` ACE usage badges

## Problem

In the ACE top bar, the Codex usage badge for an exhausted window renders as
`🤖 ! 0% 1d21h`, while the equally exhausted Grok badge renders as `🚀 0% 2d17h`. The
user wants the Codex `0%` badge to look exactly like the Grok one, with no exclamation
point.

## Root cause

`_entry_fragment()` in `src/sase/ace/tui/widgets/_provider_usage_indicator.py` adds a
`!` marker before the percentage whenever the projection entry has
`vendor_state == "rejected"`. This happens in both the exact-`0%` branch (where the
marker is folded into the inverted red zero run) and the non-zero branch.

Only some collectors ever report `vendor_state="rejected"`:

- **Codex** (`src/sase/llm_provider/usage/codex_collector.py`, `_windows_from_bucket`)
  sets `vendor_state = "rejected"` for every window in a bucket once
  `rateLimitReachedType` is present. That flag is set whenever the bucket's limit is
  hit, so an exhausted Codex window is always marked `rejected`.
- **Claude** (`vendor_state_from_rate_limit_info` in
  `src/sase/llm_provider/usage/_claude_support_windows.py`) can also report `rejected`.
- **Grok** never reports `rejected`, so its exhausted window shows a bare `0%`.

So the `!` depends on which vendor's API happens to report a rejection flag, not on
anything the user can act on. At exactly `0%` it adds nothing: the whole value run is
already drawn in the inverted red "exhausted" style (`usage_zero_value_style`), which
already says "this window is exhausted." It only makes providers look inconsistent.

The marker still means something on a **non-zero** window. There it is the only visible
sign that the vendor is refusing requests even though this window still shows remaining
capacity. For example, a Codex 5h window at 60% gets `!` while its sibling weekly window
in the same bucket is exhausted. That case is not touched by this plan.

This is presentation-only Textual rendering, so the change stays in this repo. The Rust
core and the usage projection are unchanged, and `vendor_state` is still recorded in the
data, so tooltips, `sase usage list`, and Providers · Usage keep their `rejected` /
"exhausted" state reporting.

## Change

### 1. Widget: omit the marker in the exact-`0%` branch

In `src/sase/ace/tui/widgets/_provider_usage_indicator.py`, `_entry_fragment()`:

- Delete the `if rejected:` block inside the `if percent_text == "0%":` branch, so an
  exact-`0%` run is always `[name ]0% <countdown>` in `usage_zero_value_style`, whatever
  the `vendor_state`.
- Keep the non-zero branch's `!` marker (`usage_rejected_style`) and its spacing exactly
  as they are.
- Compute `rejected` only where it is still used (the non-zero branch), or leave the
  assignment where it is if that reads more naturally. Either way, avoid an unused
  variable warning.

Result: a rejected `0%` window renders byte-for-byte and style-for-style like a
non-rejected `0%` window for the same entry.

### 2. Tests

In `tests/test_provider_usage_indicator_presentation_style.py`:

- `test_named_and_rejected_zero_run_forms_one_inverted_block`: update the expected text
  to `"🚀 grok-preview 0% 3d4h"` and the asserted zero-style run to
  `"grok-preview 0% 3d4h"`. Rename the test to say the rejected zero run omits the
  marker, e.g. `test_named_rejected_zero_run_omits_marker_in_one_inverted_block`.
- The `rejected-zero` param of the parametrized zero-emphasis test: change the expected
  text from `"🚀 ! 0% 3d4h"` to `"🚀 0% 3d4h"`.
- Add a regression test, parametrized over `dark` in `(True, False)`, that builds two
  otherwise identical `0%` entries for the same provider (use `codex` to match the
  report): one with `vendor_state="rejected", display_attention="rejected"` and one with
  the default `vendor_state="allowed"`. Assert their
  `build_usage_indicator_segment(...)` results have equal `.plain` **and** equal
  `.spans`, and that `"!"` does not appear in the rejected one. This pins "looks exactly
  like the non-rejected `0%`," not just "has no `!` character."
- Leave `test_rejected_marker_renders_bold_on_badge_surface` (the 4% case) unchanged. It
  guards the non-zero marker that is intentionally kept.

In `tests/test_provider_usage_indicator_presentation.py`, leave
`test_vendor_rejected_window_shows_marker_before_percentage` (4%) and
`test_collector_problem_entry_has_no_warning_marker_but_keeps_tooltip_prose` (50%)
unchanged. They are non-zero and should still pass. Add a short sibling assertion or
test if it helps show the zero/non-zero contrast next to them. This is optional; don't
duplicate the style-file regression test.

Before finishing, grep `tests/` for any other expectation of `"! 0%"` or a rejected
zero-percent header badge and update it. None is known beyond the two above. The
`rejected` 0% entries in `tests/test_provider_usage_indicator_widget.py` and
`tests/_provider_disables_indicator_helpers.py` only assert tooltip or layout content
and should keep passing unchanged.

### 3. Docs

In `docs/ace.md`, in the top-bar usage indicator section:

- In the sentence saying an exact `0%` highlights "its name, rejection marker,
  percentage, and reset countdown together", drop "rejection marker". The zero run is
  now name, percentage, and reset countdown.
- Change "`!` marks a vendor-rejected window." to say that `!` marks a vendor-rejected
  window that still shows remaining capacity. An exact `0%` window never shows it,
  because the inverted red run already signals exhaustion, so every provider's exhausted
  window looks the same whether or not its vendor reports a rejection.
- Keep "a non-zero window's name and rejected marker" keeping normal surfaces. It is
  still accurate.

Do not edit `CHANGELOG.md`; release-please generates it.

## Out of scope

- Codex's bucket-level `rateLimitReachedType` is copied onto every window in the bucket,
  so a non-exhausted sibling window can show `! N%`. That behavior is unchanged. If it
  turns out to be noisy, it is a separate collector-level follow-up.
- Providers · Usage modal and `sase usage list` state labels (`exhausted` / `bold red`)
  are separate surfaces and are unchanged.

## Verification

1. Run the focused unit tests first:
   `tests/test_provider_usage_indicator_presentation_style.py`,
   `tests/test_provider_usage_indicator_presentation.py`,
   `tests/test_provider_usage_indicator_presentation_layout.py`,
   `tests/test_provider_usage_indicator_widget.py`,
   `tests/test_provider_disables_indicator*.py`, and
   `tests/ace/tui/test_usage_header.py`.
2. Run `just fix` (or at least `just fmt`), then `sase tool run check` (falling back to
   `just check`). Do not run `just check-full`.
3. PNG goldens: no existing ACE visual fixture renders a rejected exact-`0%` badge. The
   only rejected fixture is Grok at 4% in
   `tests/ace/tui/visual/test_ace_png_snapshots_provider_usage_indicator.py`, and its
   rendering is intentionally unchanged. So no golden is expected to change. Confirm
   this with a targeted check-only visual run over the provider usage indicator snapshot
   modules (`just test-visual` with a `-k provider_usage_indicator` style selector after
   `--`). If any golden differs, inspect the report before accepting it, because a diff
   would mean the change leaked beyond the exact-`0%` rejected case.
