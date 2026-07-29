---
tier: tale
title: Keep epic bead page URLs uninterrupted
goal:
  Epic clan summaries preserve each hosted bead page URL as one exact, flush-left logical line so terminal URL matchers
  can identify and open the complete target.
create_time: 2026-07-29 08:49:19
status: done
---

- **PROMPT:** [202607/prompts/unwrapped_epic_bead_page_url.md](prompts/unwrapped_epic_bead_page_url.md)

# Plan

## Why

Commit `f36f37d3` added the epic bead's hosted GitHub page to both forms of the ACE agent-clan summary. The new row
deliberately hard-wraps the URL before its final path segment and indents the continuation:

```text
   Page: https://github.com/sase-org/sase--beads/blob/main/pages/sase-ao/
         README.md
```

That is attractive as a path display, but it is the wrong behavior for a URL. Terminal URL matchers need one
uninterrupted address; the newline plus continuation spaces turn the page into separate tokens. In Kitty this defeats
the user's normal `ctrl+shift+e` workflow.

There are two layers of wrapping to distinguish:

1. `sase_clan_summary_epic` currently authors hard line breaks into the persisted Rich-markup summary at a fixed 76-cell
   budget. These breaks and their indentation become durable data and are the regression to remove.
2. Textual may visually reflow a long logical line when the live detail pane is narrower than the URL. The normal
   120-column ACE layout leaves only about 52 content cells in that pane. No renderer can show a 73-cell literal URL
   there on one physical row without either widening the pane, clipping the target, or allowing viewport reflow.

The reliable policy is therefore to persist the URL as one complete, flush-left logical line and never insert
application-authored whitespace inside it. On a wide or zoomed metadata panel it appears on one physical line. On a
narrow panel, Textual may continue it at the viewport edge, but the continuation begins immediately at column zero;
there is no label or indentation embedded in the URL. This preserves the literal address for terminal matching while
keeping all non-URL summary content responsive.

## User experience

The plan-backed summary places the label directly below `Bead:` and the complete URL on the next logical line:

```text
◆ EPIC sase-ao
  Title: Model aliases in the %model completion menu
   Goal: Typing `%m:` / `%model:` shows model aliases as unmistakable, …
   Path: plans:202607/model_alias_completion.md
   Bead: sase-ao
   Page:
https://github.com/sase-org/sase--beads/blob/main/pages/sase-ao/README.md
```

The bead-backed fallback uses the same treatment in its trailing reference region:

```text
Plan: 202607/model_alias_completion.md
      /home/bryan/…/plans/202607/model_alias_completion.md
Page:
https://github.com/sase-org/sase--beads/blob/main/pages/sase-ao/README.md
```

`Page:` remains a muted structural label. The URL retains the existing blue address styling, including the bold final
filename, but the raw plain text from `https://` through `README.md` is one exact line with no leading or internal
whitespace.

At a narrow viewport, the only acceptable visual continuation is:

```text
https://github.com/sase-org/sase--beads/blob/main/pa
ges/sase-ao/README.md
```

Joining the adjacent fragments produces the original URL byte-for-byte. In particular, no continuation indentation is
allowed.

## Design decisions

1. **Move the label above the URL.** Keeping `Page: ` in front of the address spends six or nine scarce cells and makes
   even common SASE GitHub URLs exceed the script's 76-cell budget. A separate label line lets the raw address use the
   full logical width and makes the rule obvious in both rendering paths.

2. **One URL means one logical `Text` line.** Do not call `_render_field_lines`, `_wrap_bead_page_url`, `Text.wrap`, or
   any preferred-break helper for the page value. Append the styled URL directly as a complete line. Its `plain` value
   must equal the resolver's URL exactly.

3. **Do not clip or ellipsize.** `no_wrap` plus `crop` would leave Kitty only a URL prefix, which is worse than the
   current output. A global `text-wrap: nowrap` rule would also damage every title, goal, phase, and provenance row in
   the metadata panel. Normal content continues to wrap as it does today.

4. **Do not redesign the Agents split.** Widening the detail pane enough for this one value would shrink the fixed-width
   agent list and still fail on smaller terminals or longer repository names. The existing `Z` metadata zoom already
   gives a wide view when desired; this fix should not make navigation or layout jump as selection changes.

5. **Keep the literal URL rather than substituting a short label or directory URL.** The displayed target must remain
   exactly the authoritative value returned by `sase bead pages url` or recorded in plan provenance. This preserves
   copy/paste behavior and does not introduce redirect or GitHub-URL-shape assumptions.

6. **Do not depend on OSC-8 for the fix.** Kitty's default `ctrl+shift+e` is raw-URL hinting; its explicit-hyperlink
   hints are a separate mode. An OSC-8 target can be a later enhancement, but it does not replace keeping the visible
   literal address uninterrupted.

7. **Make the width exception narrow and testable.** Existing summary renderers cap ordinary rows at
   `_SUMMARY_WIDTH = 76`. The page URL is the sole intentional exception: its logical line may exceed 76 cells rather
   than being corrupted. All other rows retain their current width assertions and wrapping behavior.

8. **Preserve all lookup and omission semantics.** Stored plan provenance still wins over best-effort live lookup;
   unresolved pages still produce no `Page:` block; renderers remain filesystem-free; and a page block dropped by the 30
   KiB summary budget is still reported as `"bead page link"`.

## Rust core boundary

No `sase-core` change is needed. Hosted URL resolution, plan provenance targets, and launch-time summary persistence are
unchanged. This is pure Python/Rich presentation shaping in the existing summary adapters.

## Changes

### 1. Render the plan-backed page as a label plus one complete URL line

Update `src/sase/sdd/_plan_display_rendering.py`:

- Change `BEAD_PAGE_ROW_LABEL` from the field-shaped `"   Page: "` to the label-only `"   Page:"`.
- Keep `bead_page_url_text(url)` as the single styling authority. Preserve the existing directory-prefix and bold
  basename spans, but mark the returned standalone value as non-folding/overflow-preserving where Rich supports that
  without changing its plain text. This property protects direct uses of the line; correctness must not rely on the
  property surviving Rich-markup serialization.
- Remove `bead_page_wrap_hint`. It encodes the obsolete policy of inserting a preferred break before the filename.
- In both insertion branches of `render_plan_document`—after a `BEAD` provenance row and after the final provenance row
  when no `BEAD` row exists—append exactly two intro lines: the muted label and `bead_page_url_text(url)`. Do not pass
  the URL through `_render_field_lines` or `_wrap`.
- Leave the opt-in `bead_page_url` parameter and all ordinary field/phase wrapping unchanged.
- Remove the obsolete helper from `__all__`.

Update `src/sase/sdd/plan_display.py` to stop importing and re-exporting `bead_page_wrap_hint`.

### 2. Apply the same logical-line contract to the bead-backed fallback

Update `src/sase/scripts/sase_clan_summary_epic.py`:

- Stop importing `bead_page_wrap_hint`.
- Rewrite `_bead_page_block_lines` to emit the optional leading separator, one `Page:` label line, and one complete
  `bead_page_url_text(page_url)` line. The URL begins at column zero.
- Delete `_wrap_bead_page_url` and the local `_fold_text`/`Console` machinery that exists only to split this URL.
- Keep the page lines together as one `_DocumentBlock(omission_kind="page")`, so truncation remains atomic and the
  existing omission accounting is unchanged.
- Do not modify URL resolution, plan-first behavior, failure degradation, summary byte budgeting, or any other epic
  content.

### 3. Document the terminal-safe presentation guarantee

Update the bead-pages section of `docs/beads.md` to say that epic clan summaries put the literal hosted page URL on an
uninterrupted logical line so terminal URL matchers and copy/paste can consume the complete target.

Do not promise that every terminal width can display an arbitrarily long URL on one physical row; the guarantee is that
SASE itself inserts no line break or whitespace inside the URL.

## Tests

### Shared plan-display tests

Update `tests/test_plan_display.py`:

- Replace the preferred-basename-wrap assertions with an exact logical-line contract: the line following
  `BEAD_PAGE_ROW_LABEL` has `plain == page_url`, starts with `https://`, has no leading whitespace, and is the only line
  containing any portion of that URL.
- Cover a URL longer than the requested render width and assert it remains one complete line even though its `cell_len`
  exceeds `width`.
- Keep the no-URL opt-in regression test and the placement-after-final-provenance test.
- Continue asserting the URL's prefix and basename styles, including trailing-slash, bare-value, and multibyte filename
  shapes, but remove all `bead_page_wrap_hint` tests/imports.
- Assert every non-page-value line remains within the requested width. This prevents the narrow URL exception from
  weakening the rest of the renderer's layout contract.

### Plan-backed epic-summary tests

Update `tests/test_bead/test_clan_summary_epic_plan_script.py`:

- Assert the lines after `Bead:` are exactly `"   Page:"` and the complete URL, with the URL flush-left and present
  once.
- Use a URL longer than `_SUMMARY_WIDTH` in at least one case to prove `_render_plan_summary` and Rich-markup
  serialization do not add a newline inside it.
- Preserve style assertions for the URL prefix and bold basename.
- Keep the older-plan live-resolution, no-`BEAD`, resolver-failure, byte-budget, and no-bead-store-access tests.
- Replace blanket `<= 76` assertions with assertions that exclude only the exact page URL line.

### Bead-backed epic-summary tests

Update `tests/test_bead/test_clan_summary_epic_bead_script.py`:

- Assert both the with-plan and without-plan forms render `Page:` followed by the entire flush-left URL.
- Add a narrow Rich reflow regression: wrap the standalone styled URL at a width smaller than the address, assert every
  continuation begins without whitespace, and assert concatenating the visual fragments reconstructs the original URL
  exactly.
- Keep the absent-page and summary-truncation tests.
- Retain the 76-cell invariant for every line other than the exact URL value.

### ACE visual coverage

Update `tests/ace/tui/visual/test_ace_png_snapshots_agents_clan_panel.py` only if its structural assertions need to
describe the new two-line block; keep the fixed real-looking URL and no-I/O fixture.

Regenerate the three affected epic clan-panel PNG goldens:

- `agents_clan_panel_epic_120x40.png`
- `agents_clan_panel_epic_level_2_120x40.png`
- `agents_clan_panel_epic_level_3_120x40.png`

Inspect all three images. At the normal narrow detail width, the URL may visually continue at the panel edge, but the
next fragment must start at the value column with no spaces and the bold `README.md` must remain legible. Also inspect
the metadata zoom view or an equivalent wide render to confirm the full URL occupies one physical line when enough space
exists.

## Verification

Run:

```bash
just install
pytest -q \
  tests/test_plan_display.py \
  tests/test_bead/test_clan_summary_epic_plan_script.py \
  tests/test_bead/test_clan_summary_epic_bead_script.py
just test-visual -- --sase-update-visual-snapshots
just test-visual
just check
```

Then verify against a real epic bead:

1. `sase bead pages url <epic-id>` returns the authoritative hosted URL.
2. The generated plan-backed and bead-backed summaries each contain that exact URL as one flush-left logical line.
3. In `sase ace`, select the epic clan and inspect both the normal panel and `Z` metadata zoom.
4. In Kitty, invoke `ctrl+shift+e`; the complete hosted bead page is offered as one target and opens the same URL from
   step 1.

If the aggregate `just check` reports unrelated sidecar/provider drift, record that separately, but still run and report
every repository-local formatting, lint, type, structural, unit, and visual validation stage.

## Non-goals

- Changing hosted-link resolution, GitHub URL shapes, or the `<project>--beads` page layout.
- Adding page links to tale/ordinary plan clans, the ACE PLAN lane, or per-phase beads.
- Changing the Agents-tab column ratio, keymap, metadata search, fold behavior, or global wrapping CSS.
- Adding OSC-8-only labels, a new URL-opening action, or terminal-specific escape handling.
- Refactoring unrelated plan fields to use the page URL's intentional over-width policy.
