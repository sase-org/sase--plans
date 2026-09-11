---
tier: tale
title: Restore the usage freshness marker
goal:
  Restore a visible stale-reading cue without confusing collector health or regressing
  zero emphasis and usage grouping.
size: small
proposed_by: bbugyi200.athena.0j8.f0.f2.f0
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0j8.f0.f2.f0](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0j8.f0.f2.f0.md)
  - [bbugyi200.athena.sase-zp.1](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-zp.1/README.md)
- **COMMITS:**
  - [e48aa7d](https://github.com/sase-org/sase/commit/e48aa7db0fdcb39c35895b0bff82171c56e452c1)
    — feat(queue): rename runners to weighted capacity in Python adapters and TUI

# Restore the usage freshness marker

## Outcome and rationale

Restore the compact `~` suffix after a usage percentage when its retained observation is
stale or its freshness is unknown. A reading such as `62%~ 3d4h` should visibly say that
the percentage may be out of date, without requiring users to distinguish a neutral
color from a capacity color or open a tooltip.

The removed marker described observation freshness, not collector health. Before commit
`afbfc0883`, `_entry_fragment` appended it when projected `freshness` was `stale` or
`unknown`, except after a passed reset. A refresh problem can cause old readings, but a
late refresh is not proof of collector failure; conversely a collector can fail while an
observation is still fresh. The triangle was the separate collector failure cue and
stays removed under the user's earlier design decision.

Rust already computes these states independently. In
`sase-core/crates/sase_core/src/provider_usage/mod.rs`, `freshness_value` classifies
observations as fresh through two collection cadences, stale through four, and unknown
beyond four. Thus `unknown` is not limited to a missing timestamp. The indicator
projector recomputes freshness from the observation and injected clock, and exposes
collector health separately. Consume its existing fields; do not reproduce these
thresholds or collector rules in Python.

This is a small tale because the root cause is a known presentation removal, with one
important interaction: appending `~` must not break the renderer's exact-zero style
check. One coder can restore the marker, clarify documentation, and verify the affected
rendering without backend, wire, or configuration changes.

## Relationship to the other usage plans

This is a focused correction to the no-tilde decision in
`plan:202609/zero_usage_emphasis.md`. It also overrides the no-visible-freshness-marker
assumption carried forward by `plan:202609/usage_group_refinement.md`.

The grouping plan was pending in the supplied conversation; the explored checkout still
uses the earlier grouping and percentage-only zero highlight. This plan can be
implemented on either baseline. Preserve the layout and zero-highlight extent present at
implementation time. Do not implement, revert, or wait for the separate grouping plan as
part of this correction. Its outer whitespace, neutral intra-provider pipes,
visible-count parentheses, full-value zero highlight, and always-visible Fable default
remain the intended independent improvements. A later grouping implementation must
retain this marker contract instead of reinstating its obsolete no-tilde wording.

After both changes, a representative block body is:

```text
🎭 (10%~ 1d10h | fable 0%~ 1d10h)  🤖 81% 5d2h
```

The grouping plan owns this example's parentheses, gaps, margins, and full-value red
background; this plan owns the two percentage suffixes and their meaning.

## Display and styling contract

Apply the marker to each selected window independently, including default windows and
named extras. It is one ASCII tilde immediately after the percentage, with no added
space. It belongs to the complete window fragment for width measurement and overflow.

| Projected state                        | Window value text | Meaning                                                                         |
| -------------------------------------- | ----------------- | ------------------------------------------------------------------------------- |
| Fresh numeric observation              | `62% 3d4h`        | Current enough under the existing freshness policy                              |
| Stale numeric observation              | `62%~ 3d4h`       | Retained percentage may be out of date                                          |
| Unknown freshness, numeric observation | `62%~ 3d4h`       | Retained percentage is not known to be current, including very old observations |
| Stale positive fractional capacity     | `<1%~ 3d4h`       | Low but nonzero retained reading                                                |
| Stale or unknown freshness at zero     | `0%~ 3d4h`        | Last observed capacity was exhausted; freshness is uncertain                    |
| Stale with no reset timestamp          | `62%~ ?`          | Uncertain freshness and unknown reset time are distinct                         |
| Passed reset, with any freshness       | `?% 0h0m↻`        | Awaiting an observation for the new window; omit `~`                            |

Keep the existing fallback for an absent freshness field (`unknown`) and the existing
numeric normalization. Do not synthesize windows or invent a percentage for a
collector-only state. A fresh window with `collector_problem` alone receives no tilde.
When a stale window is also vendor-rejected, retain the separate rejection marker, for
example `fable ! 0%~ 1d10h`.

Nonzero stale and unknown readings keep their existing bold neutral name/value style.
The tilde uses the same effective style as its percentage; it is part of the displayed
value, not structural punctuation. Preserve the theme-derived contrast-safe zero style
and decide whether to use it from the undecorated formatted percentage. Thus `0%~` is
still exhausted, `<1%~` is not, clamped zero is still exhausted, and passed reset `?%`
is never exhausted based on an old retained zero.

On the current percentage-only baseline, the entire `0%~` token is red-backed and the
space/countdown keep their existing surface. Once the grouping refinement's full-value
highlight is present, the entire `0%~ <countdown>` run, including its tilde and internal
space, uses that red background. This avoids an unhighlighted hole in the value. The
same rule covers `0%~ ?`. Names, rejection markers, icons, punctuation, and exterior
spaces retain their own styles, with no red bleed. No additional colors or style
configuration are required.

## Implementation

1. In `src/sase/ace/tui/widgets/_provider_usage_indicator.py::_entry_fragment`, retain
   an undecorated percentage token for state/style selection and derive the visible
   token by adding `~` only for stale/unknown freshness when the reset has not passed.
   Keep the existing formatted-zero eligibility test independent of that decoration.
   Append the decorated token with the existing percentage/value style and preserve the
   renderer's active highlight extent. Do not change the shared numeric formatter; its
   callers outside the top bar have their own notation.
2. Restore marker-aware prose in `_entry_tooltip_lines`, such as
   `~ marks a retained reading that may be out of date; it does not by itself mean refresh failed.`
   Keep the explicit `freshness: stale` / `freshness: unknown` line, reset explanation,
   and existing collector-failure disclosure. Do not add a tilde explanation to the
   passed-reset entry as if that entry displayed it.
3. Update the notation legend in
   `src/sase/ace/tui/widgets/provider_disables_indicator.py` and the Providers · Usage
   section of `docs/ace.md`. Explain observation freshness separately from collector
   health, including that very old observations can be classified unknown. Keep
   collector diagnostics available in the tooltip and Providers · Usage, and describe
   zero as an exhausted last observation when it carries `~`. Remove the current
   assertion that freshness has no visible marker. Preserve any newer grouping prose.
4. Let `Text.cell_len` and the existing complete-window packing account for the extra
   cell. Do not add a second width formula or change provider/window selection,
   ordering, collection, routing, eligibility, refresh cadence, or polling. Keep this
   memory-only Python TUI work; the Rust core and shared numeric formatting need no
   change. No new flag, config key, CLI option, or keybinding is introduced. Check
   whether existing help text describes this notation and update it if needed under
   ACE's help-maintenance rule.

## Regression coverage

Update existing tests that assert the absence of `~`, rather than preserving the
obsolete expectation or reverting the zero-emphasis change wholesale.

- In `tests/test_provider_usage_indicator_presentation.py`, cover fresh, stale, unknown,
  and absent freshness; fresh/stale combinations with collector problems; stale without
  a collector problem; default/named windows; mixed-freshness windows within one
  provider; `<1%~`, `100%~`, zero/clamped-zero suffixes, rejected stale zero, unknown
  reset time, and passed reset for both stale and unknown retained zeros. Assert exact
  value text and the tooltip's distinct freshness/collector explanations.
- Extend the existing real `provider_usage_project_indicator` integration coverage with
  a fixed numeric observation and injected clock/cadence. Prove that aging it changes
  the marker without changing collector status, and replacing it with a fresh
  observation clears the marker. Use the actual projector rather than hand-writing a
  freshness algorithm in the test or renderer. Existing Rust threshold tests remain
  authoritative; this is a test of projection-to-renderer composition.
- In both themes, check effective Rich styles by character offsets: the tilde matches
  its percentage, `0%~` keeps the zero style, and neighboring names, `!`, structural
  punctuation, and gaps do not inherit red. Assert countdown styling according to the
  active baseline described above and retain the existing minimum contrast checks.
- Extend width/overflow coverage around an exact-fit boundary where adding one tilde
  forces a complete window into overflow. Check budgets on both sides, correct hidden
  totals, and shrink/grow restoration. Exercise a mixed group with a stale named extra;
  if parentheses are present, they remain balanced when the extra no longer fits. A
  narrow bar must never leave behind a detached suffix or split a value.
- In `tests/test_provider_disables_indicator_usage.py`, cover stale nonzero and stale
  zero beside routing/priority pills, preserving their styles and the current usage
  margins. Confirm the combined widget tooltip explains the marker and clicks still open
  Providers · Usage.

Update affected stale scenes in
`tests/ace/tui/visual/test_ace_png_snapshots_provider_usage_indicator_states.py`,
reusing `_provider_usage_indicator_fixtures.py`. Include a dark/light mixed-freshness
group with a stale exhausted named window, and retain routing-pill and narrow-bar
coverage. Keep a fresh-but-failing collector scene free of `~` to make the distinction
visible. Do not regenerate unrelated palette or layout goldens merely because the suite
contains them.

## Verification and completion

Run the focused presentation and composition tests. Refresh the affected usage PNG
goldens with `just test-visual` restricted to the usage snapshot files and
`--sase-update-visual-snapshots`; inspect the rendered images/diffs, then rerun those
files without update mode. Confirm tilde legibility, preserved zero contrast, clean
style boundaries, and complete-window overflow in both themes. Include the other usage
snapshot file if its existing scenes are affected by the added cell.

Read the current `lint_and_test.md` reference memory and run `just check` before
completion. Use `sase_monitor` for verification that becomes long-running, following the
skill's handoff workflow. Report current verification results accurately; do not assume
a failure from the inherited transcript still exists. Finish with `git diff --check` and
review the diff for only the intended renderer, documentation, tests, and goldens.

Completion means a visible freshness cue is restored, its meaning is unambiguous, zero
emphasis survives the suffix, existing layout work remains intact, and the focused
tests, inspected stable snapshots, and required repo check are accounted for.
