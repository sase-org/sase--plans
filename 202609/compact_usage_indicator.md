---
tier: tale
title: Compact, lossless provider usage attention
goal:
  Make ACE's usage indicator concise and consistent across providers while preserving
  scope, remaining-capacity meaning, and access to every detail.
size: medium
proposed_by: bbugyi200.athena.0hb
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0hb](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0hb.md)
- **COMMITS:**
  - [72b76ba](https://github.com/sase-org/sase/commit/72b76bae6673a1f50d9d5e0e2b3d1ec48f17e57d)
    — feat(ace): compact provider usage indicator

# Compact provider usage attention

## Outcome and sizing

Replace the verbose usage sentence in the top-right indicator with one compact visual
grammar. In the user's screenshot, the usage segment reads:

```text
! GROK 14% left · Grok included weekly allowance +2
```

The intended replacement is:

```text
! GROK 14% left · wk/all +2
```

Measured with this checkout's Rich terminal-cell measurement, these strings occupy 51
and 27 cells respectively, excluding the widget's outer padding: approximately 47% less
space. Preserve the adjacent provider-priority pill as a separate visual segment.

Apply the same grammar to Claude, Codex, Grok, and other providers that supply usage
observations. Provider identity, percentage **left**, affected allowance, attention, and
additional-provider count remain readable. Full labels and all underlying usage details
remain available through the tooltip and the existing keyboard-accessible Providers ·
Usage view. On constrained terminals, explicit disclosure replaces whole groups of
information rather than clipping their meaning.

This is a `tale` of size `medium`: one coding agent can implement the bounded Textual
presentation and its tests. The subscription model, collectors, cache, attention
selection, and Usage modal already exist. No separate backend phase is required.
Authoring this plan is `large` work under the canonical sizing guidance.

## Grounding and prerequisites

Read the following artifacts with `sase artifact read` when implementing:

- `plan:202609/subscription_capacity.md`: the original sase-y5 contract, especially
  remaining-capacity precision, scope, attention, and responsiveness.
- `plan:202609/usage_context_recovery.md`: the recovered indicator and interaction
  wiring. The planning checkout already contains that implementation. Work against the
  current code; do not reapply its historical recovery patch or restore removed flags.

The user supplied `c8c42e6517f5-file-ref.png` in the originating prompt; its exact usage
text is reproduced above so implementation does not depend on that temporary path.
`sase bead show sase-y5` and `sase bead show sase-y5.12` supply the related epic
context. Those epics were still in progress during planning, with the recovery phases
closed; reconcile any subsequent changes to the same indicator before implementation.

Relevant current seams:

- `src/sase/ace/tui/widgets/provider_disables_indicator.py`: merges routing and usage,
  builds the tooltip, owns the cached attention items and selected provider, and loads
  usage snapshots in a worker. `_append_usage_content` currently repeats `item.label`
  verbatim at wide widths and switches to `usage +N` below 120 terminal columns.
- `src/sase/llm_provider/usage/hints.py`: `indicator_usage_items` returns one attention
  hint per eligible provider, ordered by severity and then provider ID. A hint carries
  the selected `window_key`; its `scope` is display text, not structured applicability.
- `src/sase/llm_provider/usage/peek.py`: supplies the memory-only provider snapshots
  from which the selected window can be resolved without new I/O.
- `src/sase/ace/tui/widgets/_override_pill.py` and `styles.tcss`: established pill
  palette, spacing, and one-line right-aligned layout.
- `src/sase/ace/tui/actions/base.py`: `action_open_provider_usage` already selects
  `usage_open_provider` when called without an explicit provider argument.
- `src/sase/ace/tui/actions/_event_widgets.py`, `_app_layout.py`, and
  `widgets/tab_bar.py`: resize handling and top-bar layout integration if needed.

Read `tui_perf.md` and `lint_and_test.md` through the memory-read skill before coding.
Read the applicable `AGENTS.md`, including `src/sase/ace/AGENTS.md`.

## Visual and semantic contract

### One normal form

```text
<attention> <PROVIDER> <remaining> left · <window>/<scope> [+N]
```

Representative fixtures, with deliberately stated metadata:

| Observation                                                                                 | Compact segment                |
| ------------------------------------------------------------------------------------------- | ------------------------------ |
| Grok weekly included allowance; account scope; 14% left; two other providers need attention | `! GROK 14% left · wk/all +2`  |
| Grok monthly included allowance; account scope; 14% left                                    | `! GROK 14% left · mo/all`     |
| Claude weekly allowance explicitly scoped to model ID `opus`; 6% left                       | `! CLAUDE 6% left · wk/opus`   |
| Codex window with supplied 18,000-second duration; account scope; 12% left                  | `! CODEX 12% left · 5h/all`    |
| Codex five-hour window with unknown applicability; quantitative low attention               | `! CODEX 12% left · 5h/scope?` |
| Grok weekly account allowance with a positive remainder below 1%                            | `! GROK <1% left · wk/all`     |
| Codex collection problem with no quantitative conclusion                                    | `? CODEX usage`                |

These are presentation examples, not new provider facts or new applicability rules. For
a named nondefault bucket, retain its distinct name in the allowance token, for example
`Fast/5h/scope?`; fall back to disclosure if the entire token will not fit. Do not claim
every already-short label must shrink: retain necessary distinguishing information. The
screenshot case and verbose known-provider labels must get smaller.

Keep `left` in every quantitative form. A bare percentage could mean either consumed or
remaining capacity; saving those five cells would make the interface ambiguous. The
slash separates the window from its scope, not a reset countdown or a rate. `wk` means
weekly, `mo` monthly, `all` account-wide, and `scope?` uncertain applicability. Use
`session` when the observation establishes only a session, without inventing 5h.

Preserve the full provider name. Use the existing colorless attention marker from the
hint (`!` for capacity attention, `?` for unknown/collection problems). Keep the
existing yellow usage background and readable dark foregrounds; emphasize the marker,
provider, and numeric value with the primary style, and recess `left`, the separator,
scope, and count with the existing secondary style. No new icons, animations, progress
meters, color-only meanings, or extra bar rows. Maintain one cell of outer padding
without introducing a double gap beside a routing pill.

### Safe compact labels

Implement a private, pure TUI presentation helper rather than shortening collectors'
stored labels or changing the shared `CapacityHint.label` used by other surfaces. Build
an immutable presentation record from a hint and its exact matching window in the same
cached provider snapshot. Preserve the original hint and full window label for
disclosure. Resolve by provider ID and `window_key`, never by position or lowest
percentage. Do not independently rerank hints or combine windows.

Use structured `duration_seconds` and `applicability` when supplied. Render simple
durations exactly (`5h`, `7d`/`wk`); do not round an arbitrary duration to a familiar
window or turn variable calendar months into a fixed number of days. Where duration is
absent, conservative, exact aliases for existing normalized display labels may express
the period already stated in that label: weekly to `wk`, monthly to `mo`, session to
`session`. Removing a repeated provider prefix and the known boilerplate
`included ... allowance` is typography, not a domain inference. Keep such aliases in one
small helper, with fixtures; unknown labels take the generic path.

Render account applicability as `all`. Preserve explicit model IDs, product names, and
family distinctions instead of guessing from a vendor label. Multiple model IDs remain
an indivisible scope token; if too long, disclose the whole quantitative group. Never
replace unknown applicability with `all`, even when the full label says "all models." In
particular, the current Claude collector can emit product windows without an explicit
model mapping, which existing hints mark `scope unknown`; preserve that uncertainty as
`scope?`. Period abbreviation does not grant a stronger claim about applicability. If
existing hint metadata indicates uncertain scope, keep that qualification even when a
friendly product name is available.

Unknown period and unknown scope are independent: abbreviating a clearly stated week
must not erase `scope?`, and known account scope must not invent a week. Keep an
unrecognized window's distinguishing label, even if that makes disclosure necessary.

Preserve unfamiliar bucket names, provider names, and case-sensitive identifiers as
plain text. Do not introduce provider-specific behavior branches in the widget, use
substring guesses to classify scope, or chop labels at arbitrary characters. Use Rich
`Text` composition, not markup interpolation. If the selected window is missing or a
safe compact representation is unavailable, retain the original full hint as a candidate
and then use the disclosure forms below; never fabricate a period or scope.

Reuse the existing core-backed percentage formatting and its exact text, including
`<1% left`, genuine `0% left`, and non-exact `100%` handling. Do not round
`remaining_percent` independently or add Python numeric/freshness policy. Preserve any
stale, unknown, or reset-passed qualification supplied by the current view; a historical
reading cannot become an unqualified current percentage during abbreviation. Do not
replace missing values with zero or turn zero remaining into a vendor rejection.

### Counts and constrained space

The normal form displays the existing highest-attention provider. `+N` always means **N
additional providers needing attention**, excluding the displayed provider; it never
counts allowance windows, agents, or all configured providers. Omit `+0`. Keep the
existing severity order, provider-ID tie-breaking, eligibility, and quiet healthy state.
Multiple windows remain available in the Usage view.

Choose the richest complete candidate that fits the available usage-segment budget:

1. Normal form with provider, remaining capacity, complete window/scope, and `+N`.
2. Provider disclosure: `! GROK +2` (or `? CODEX`), removing the percentage and its
   entire scope together.
3. Total-count disclosure: `! usage 3` when it is shorter than the provider form and
   fits; here `3` is the total, not an additional count.
4. For the smallest residual space, `!3` or `?3`, still carrying the total. Explain the
   compact count in the tooltip and documentation. Keep it clickable.

Choose candidates by actual cell width, skipping ones that are not progressively
shorter. A long custom provider must not force overflow. Never render `14% left` on its
own after hiding its scope, truncate a percentage or `+N`, cycle providers, or use an
ellipsis that conceals whether additional providers exist.

Replace the fixed terminal-wide 120-column switch with a local width budget for the
existing indicator. Account for the routing segment, other visible top-bar widgets,
spacing, and the tab bar; measure terminal cells, not Python string length. Keep the
current widget order and the existing routing content. Re-evaluate from in-memory
layout/content on mount, terminal resize, and sibling geometry changes, including
expansion when space returns. Changing only the terminal width is insufficient when an
override or notification grows at the same width.

Use a coalesced layout hook and update only when the selected content changes. Derive
the budget from the container and siblings, excluding this indicator's previously
selected width, so compact and expanded forms cannot cause oscillation. Do not add
polling, repeated full-screen rebuilds, or synchronous work in resize/render handlers.
Handle pre-mount geometry conservatively. Existing extremely crowded combinations whose
non-usage widgets already exceed the terminal are outside this local redesign; the usage
portion must choose its minimum form and never make that overflow worse than the current
minimum usage display. Verify normal and crowded supported layouts.

### Disclosure and interaction

Build the segment, tooltip, and `usage_open_provider` from one captured snapshot. The
tooltip remains invariant when the visible form changes. Keep the existing routing
section intact. Its usage section lists every attention provider in the same order, full
original hint/label, and explicit attention kind. Explain the window/scope notation and
additional-versus-total count in one concise legend. Include the selected window's
available freshness/reset qualifications as needed to preserve its meaning; the existing
Usage modal remains the home for all windows, precise values, reset times, observation
ages, collection outcomes, and source details.

The compact indicator is an included-subscription allowance surface. State that in the
tooltip and documentation so removing "included allowance" from Grok's repeated label
does not blur it with token usage, spending, or a paid-credit balance.

Clicking any usage-bearing form opens Providers · Usage with the displayed/leading
provider selected, using the existing action. The command-palette Usage command and
Launch Control's `u` remain the keyboard paths to the same complete information. Opening
or hovering must not trigger a provider probe. Returning from the modal must not change
routing, selection order, or the stored usage data. Without usage attention, retain the
current routing-only click behavior.

## Implementation sequence

1. Add the private TUI presentation helper and fixtures for compact labels, retained
   full details, and ordered whole-token candidates. Consume existing hints and selected
   window metadata without changing their selection or shared labels.
2. Integrate that presentation into `ProviderDisablesIndicator`; extract usage-specific
   rendering from the already sizable widget rather than growing a monolithic file.
   Preserve its worker, token, and in-flight guards. Add only the necessary layout
   integration for a stable available-width budget and immediate resize updates.
3. Preserve and verify tooltip/click/keyboard parity for expanded and disclosed forms.
   Update `docs/ace.md` beside Providers · Usage with the compact example, legend, count
   semantics, and routes to details. No new global binding or config option is needed;
   the default keymap remains applicable.
4. Add focused semantic and interaction coverage, inspect actual rendered PNGs, and
   update only the intentional usage-indicator golden changes. Finish verification.

This work is Textual presentation over the existing domain result. Keep collectors,
shared attention/freshness/rounding/routing rules, storage, refresh admission, and CLI
output unchanged. If implementation uncovers a required shared-domain change, use the
`sase_repo` skill to open `gh:sase-org/sase-core` and the Rust boundary rules; do not
hide a backend reimplementation inside the TUI helper. Do not edit memory or introduce a
feature flag for this reversible presentation refinement.

## Acceptance and verification

Extend existing tests instead of creating broad duplicate suites:

- `tests/test_provider_disables_indicator.py` plus a focused private-helper test module
  if warranted: Grok weekly/monthly, Claude model-specific/session/unknown product
  scope, Codex primary/secondary durations and unknown buckets, a custom provider,
  literal markup/wide characters, exact zero and positive sub-percent values, missing
  windows/values, unknown periods, stale/reset qualifiers, all attention kinds, and
  one/two/three providers. Ensure no mutation of input snapshots or shared hint labels.
  Use actual collector-shaped fixtures, not only invented `Shared 5h` labels; primary
  and secondary windows must not be confused.
- Budget boundary tests: each candidate fits at its exact cell width and moves to the
  next complete form one cell below. Percent and scope disappear together; long unknown
  labels retain full tooltip text; `+N` and total `N` have distinct, verified meanings.
- `tests/ace/tui/test_top_bar_order.py` and focused `AcePage` interaction coverage:
  unchanged widget order/routing display; 60, 80, 120, and 160-column layouts; resize
  wide-to-narrow-to-wide; growing/shrinking siblings without changing terminal width;
  stable layout after settling; final visible regions remain in bounds in the supported
  fixtures. Confirm mount and keyboard interactions do not wait for a slow usage worker.
  Render/resize/hover/click cause no new store reads or probes.
- Verify opening the real Usage modal with the leading provider selected from both the
  chip and the existing keyboard command, including after rank changes. Ensure all
  attention providers and per-window details are reachable without a mouse.
- Preserve existing `tests/llm_provider/test_usage_hints.py` and
  `tests/llm_provider/test_usage_peek.py` coverage for selection and cache behavior.
- Extend `tests/ace/tui/visual/test_ace_png_snapshots_provider_usage_indicator.py`: a
  wide screenshot-equivalent priority plus Grok case, Claude and Codex examples, and a
  crowded narrow multiple-provider case. Inspect real PNG output for spacing, emphasis,
  contrast, and clipping in supported dark/light themes. Reuse the existing golden
  infrastructure; do not accept snapshots solely because assertions pass.

Run the relevant focused pytest files, then the dedicated visual subset with
`just test-visual` (it is excluded from ordinary tests). Inspect actual/expected/diff
artifacts before using `--sase-update-visual-snapshots`, then rerun the affected visual
subset. Run `just check` after tracked implementation changes; use `just install` if the
environment needs synchronization. Follow `lint_and_test.md` if selection escalates or
landing requires `just check-full`; long verification uses the monitor skill. Planning
itself changes only this scratch plan and requires plan validation.

Completion means the screenshot case occupies at most 27 content cells in its normal
form, other providers follow the same semantics, compacting never creates an unscoped
percentage or false certainty, complete information is accessible by both mouse and
keyboard, and the focused visual/behavior checks plus required repository checks pass.

<!-- sase:referenced-by:start -->

## Referenced By

| Relation | Artifact | Why | Uses |
| --- | --- | --- | ---: |
| cited-by | [agent:bbugyi200.athena.0hb--code][1] | prompt reference @plan:202609/compact_usage_indicator.md | 1 |

[1]: https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0hb.md

<!-- sase:referenced-by:end -->
