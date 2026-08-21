---
tier: tale
title: A useful statistics footer for flag list
goal:
  Make every non-empty human-readable `sase flag list` end with a compact, trustworthy,
  and visually coherent summary of the flags that were rendered.
size: medium
proposed_by: bbugyi200.athena.09n
create_time: 2026-08-21 11:15:58
status: wip
---

# Plan: A useful statistics footer for `sase flag list`

## Outcome

Add a blank-line-separated footer to non-empty human `sase flag list` output. It should
answer four questions at a glance without competing with the detailed rows:

1. How many flags exist, and which lifecycle kinds do they use?
2. How many resolve on versus off in this process?
3. How many decisions came from a layer above the registry default?
4. Is any loaded removal bead approaching or past its deadline?

A representative plain-text footer is:

```text
3 flags · 2 beta  1 sunset · 2 on  1 off · 1 overridden · ⧗ 1 soon  ⧗ 1 due
```

When a dimension has only one non-zero value, fold it into the grammatical head just as
`sase bead list` does. For example, three default-valued beta flags that are all off
render as:

```text
3 off beta flags
```

The compact form must retain words as well as color and glyphs, remain understandable
with `NO_COLOR`, and use natural singular/plural wording (`1 on beta flag`).

## Existing behavior and boundaries

- `src/sase/feature_flags/cli_list.py` obtains one sorted tuple of `FlagView` objects,
  renders one Rich `Text` row for each, then prints resolver diagnostics. There is no
  footer today.
- `FlagView` already carries every trustworthy input needed here: the definition kind,
  the effective `FeatureFlagDecision`, the optional loaded bead, and the removal
  `due_state` calculated from the same pinned `today` and `release` used by the rows.
- `src/sase/bead_summary_presentation.py` establishes the desired visual grammar: a
  count head, folded homogeneous dimensions, middle-dot-separated groups, two spaces
  within count groups, non-zero buckets only, shared presentation metadata, and a blank
  line between rows and footer.
- This is presentation-only behavior. Do not add a Rust binding or duplicate flag
  resolution/removal-deadline logic.
- `sase flag list --json` is a versioned machine contract and does not print lines. Keep
  its schema version and payload unchanged; this feature is explicitly the human footer.
  A future request can add structured aggregates with an intentional schema revision.
- A missing `FlagView.bead` is not necessarily a broken flag: `flag list` is supported
  outside the SASE checkout, where the flag-bead store may be unavailable. Do not emit a
  misleading “missing bead” statistic.

## Design

### 1. Build one pure summary from the rendered views

Add a focused presentation module under `src/sase/feature_flags/` (for example,
`cli_summary.py`) with an immutable `FlagListSummary`, a pure
`summarize_flag_views(views)` function, and a pure Rich-text footer builder.

The summary should record:

- total rendered flags;
- fixed-order counts for `beta` and `sunset` kinds;
- enabled and disabled counts from `view.decision.enabled`;
- the number of decisions whose existing `view.decision.overridden` bit is true; and
- fixed-order `live`, `soon`, and `due` counts for views whose `due_state` is present.

Count the supplied `FlagView` sequence exactly once and render from that summary. Do not
re-read the registry, snapshot, bead store, clock, release, environment, or config. In
particular, “overridden” means the resolver supplied a non-default layer, even when that
layer selected the same boolean as the registry default; do not infer it by comparing
booleans.

### 2. Use compact folding and stable group order

Render groups in this order:

1. grammatical count head;
2. kind breakdown, only when kinds are heterogeneous;
3. effective on/off breakdown, only when values are heterogeneous;
4. `N overridden`, only when non-zero;
5. removal urgency entries for `soon` and `due`, each only when non-zero.

Fold a homogeneous effective value and/or kind into the head in adjective order:
`<count> <on|off> <beta|sunset> flag(s)`. Never print zero buckets. Do not print the
`live` removal count: healthy countdowns are already visible in rows, while the footer
should preserve its scarce attention for approaching and breached thresholds. Retain the
`live` count in the summary model so totals and boundary tests can prove that every
available due state was classified.

Use `" · "` between semantic groups and two spaces between peer counts. The pure builder
may sensibly return `0 flags`, but the CLI empty state remains more useful and must not
render a footer.

### 3. Reuse the command's visual language

Return Rich `Text` rather than hand-authored ANSI strings so the injected/test console,
TTY capability, and `NO_COLOR` behavior remain authoritative.

- Promote the existing kind and effective-value treatments in
  `sase.feature_flags.cli_render` into small shared Rich-text helpers (or shared style
  metadata), and have both `_list_row` and the footer call them. This prevents the new
  footer from merely copying presentation literals that can drift later.
- Through those shared helpers, render `beta` and `sunset` in the current italic
  treatment and render `on` as bold green and `off` as bold.
- Give the neutral `overridden` clause restrained bold emphasis; it is noteworthy but is
  not itself an error.
- Render each urgency entry with `FLAG_DUE_GLYPH` and the existing
  `FLAG_DUE_STYLES["soon"|"due"].rich` style from `sase.bead_flag_presentation`. Do not
  invent another urgency palette or recompute due-ness.
- Keep all semantic labels in the plain text so styling is never the only carrier of
  meaning.

Allow Rich to wrap naturally on narrow terminals rather than truncating or disabling
wrapping; the footer remains one logical line and degrades legibly when physical width
is constrained.

### 4. Make the footer the actual end of human output

In `_render_list`, preserve the row order, render any existing resolver diagnostics,
then print exactly one blank line and the footer. This makes the statistics line the
last logical output even when warnings exist:

```text
<flag rows>

warning: <diagnostic>

<statistics footer>
```

With no diagnostics the result is simply rows, one blank line, then the footer. Keep the
current empty-registry message/scaffold hint (and any diagnostics) unchanged and
footer-free. Keep JSON footer-free and byte-valid JSON.

### 5. Document the contract where users discover the command

Update the `sase flag list` parser description and the feature-flag CLI documentation in
`docs/configuration.md` to describe the footer's inventory, effective-value, override,
and actionable urgency summaries. Include one plain example so the folding and
separators are discoverable. Keep high-level command tables concise unless their
existing wording would become inaccurate.

## Verification

Add focused unit coverage for the summary presenter and integration coverage in
`tests/feature_flags/test_cli.py`:

- mixed kinds and effective values produce fixed-order, non-zero counts and the exact
  plain footer shown above;
- single and plural homogeneous inventories fold to `1 on beta flag`,
  `2 off beta flags`, and the corresponding partially folded mixed cases;
- `decision.overridden` is honored even when effective value equals the default;
- `live`, `soon`, `due`, and absent due states are counted without recomputation, while
  only non-zero `soon`/`due` entries render;
- Rich spans reuse the row value styles and shared urgency styles, while a colorless
  console emits the complete semantic plain text with no escape sequences;
- existing list rows retain their current kind and effective-value text/style after
  those treatments move behind shared presentation helpers;
- a non-empty human listing ends with exactly one blank-line-separated footer;
- diagnostics remain visible before the footer, and the footer is still the final
  logical line;
- the empty-registry scaffold remains unchanged and has no redundant `0 flags` footer;
- `--json` retains schema version 1 and its existing keys, contains no human footer, and
  remains parseable.

After implementation, run `just install` first as required for an ephemeral workspace,
then run `just check`. The change does not touch the broadening set and should not need
`just check-full` unless scoped verification escalates or reports unusual selection.
