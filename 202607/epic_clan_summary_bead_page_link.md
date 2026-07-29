---
tier: tale
title: Link the epic clan summary to its hosted bead page
goal:
  Every epic agent-clan summary shows the `<project>--beads` GitHub URL for its epic bead, styled and wrapped like the
  plan `Path:` row, and silently omits the row when no hosted page can be resolved.
create_time: 2026-07-29 08:13:07
status: done
---

- **PROMPT:** [202607/prompts/epic_clan_summary_bead_page_link.md](prompts/epic_clan_summary_bead_page_link.md)

# Plan

## Why

The ACE Agents-tab clan panel renders a frozen, launch-time epic summary that already names the epic bead
(`Bead: sase-ao`) but gives no way to reach the rendered bead page in the `<project>--beads` sidecar on GitHub. Today
the only way there is to leave the TUI and run `sase bead pages url <id>`. The bead page is the best "what is this epic,
really" artifact SASE publishes, so the clan summary — the panel a user stares at while an epic clan runs — should carry
its address.

The URL is cheap to obtain: `sase.sdd.hosted_links.HostedLinkResolver.bead_url` already derives it,
`sase.bead_pages.links.resolve_bead_page_target` already guards it against dead links, and epic plan files already
**store** the resolved URL in their `BEAD` provenance bullet (written by `refresh_bead_plan_section`). The Python
display adapter simply throws that target away today.

## Design

### What the user sees

Epic clan summaries render through two paths in `src/sase/scripts/sase_clan_summary_epic.py`. Both gain one new `Page:`
row.

**Plan-backed path** (`_render_plan_summary`, used whenever `SASE_EPIC_PLAN_REF` resolves — this is the common case and
the one in the reference screenshot). The row goes directly beneath the `Bead:` provenance row, in the same 9-cell label
column:

```
◆ EPIC sase-ao
  Title: Model aliases in the %model completion menu
   Goal: Typing `%m:` / `%model:` shows model aliases as unmistakable, …
   Path: plans:202607/model_alias_completion.md
   Bead: sase-ao
   Page: https://github.com/sase-org/sase--beads/blob/main/pages/sase-ao/
         README.md
```

**Bead-backed path** (`_render_epic_summary`, the fallback that reads the bead store). The row joins the trailing
reference region beside the existing `Plan:` row, in that region's 6-cell label column:

```
Plan: 202607/model_alias_completion.md
      /home/bryan/…/plans/202607/model_alias_completion.md
Page: https://github.com/sase-org/sase--beads/blob/main/pages/sase-ao/
      README.md
```

### Design decisions and their rationale

1. **Show the literal URL, not a shortened label.** The summary is persisted Rich markup rendered into a Textual panel;
   there is no click target and no action plumbing. A complete `https://…` string is selectable, copy-pasteable, and
   auto-detected as a link by every terminal that does URL detection. A prettified
   `sase-org/sase--beads › pages/sase-ao/README.md` label would fit on one line but is not a URL, so it fails the one
   job the row has.

2. **Do not emit an OSC-8 hyperlink.** It would have to survive `Text.markup` serialization at launch, a 30 KiB budget,
   `Text.from_markup` reparsing in the TUI, and Textual 8's segment pipeline. That is three round-trips of risk for a
   decoration that buys nothing once the visible text is already the URL. Listed as a follow-up, not as scope.

3. **Style the URL exactly like the plan `Path:` row.** Everything through the final `/` uses `COLOR_PLAN_PATH`
   (`#87AFFF`); the page filename (`README.md`, `sase-ao.1.md`) uses `COLOR_PLAN_PATH_BASENAME` (bold `#87AFFF`). The
   boilerplate host/org/repo/branch prefix recedes and the eye lands on `sase--beads` and the page name. The `Path:` and
   `Page:` rows then read as one family of "addresses", which is the whole point of putting them adjacent.

4. **Wrap before the page filename, never mid-token.** Summary lines are hard-capped at `_SUMMARY_WIDTH = 76` cells
   (asserted by existing tests), and a real URL is ~73 cells before the label column, so it _will_ wrap.
   `_render_field_lines` already accepts `preferred_break_before` / `preferred_segment_width` — the same mechanism
   `_path_basename_wrap_hint` uses for the `Path:` row. Reuse it so the break always lands on a `/` boundary and
   continuation lines align under the value column.

5. **Prefer the URL the plan already recorded; resolve live only as a fallback.** The plan-backed path exists precisely
   so a valid authored plan bypasses the bead store (there is a test asserting this). Reading `PlanProvenanceSection`'s
   `BEAD` target costs nothing and provably agrees with the `Bead:` row printed right above it. Live resolution runs
   only when a `BEAD` label exists without a target (older plans, or a plan written while the sidecar had no hosted
   remote), and in the bead-backed path where the store is open anyway. A plan with **no** bead never triggers a lookup
   and never gets an invented link.

6. **Absent URL means no row.** No `unavailable` placeholder, matching `hosted_links`' documented stance: "returns
   `None` instead of guessing, so a store without an authoritative hosted remote degrades to an unlinked label instead
   of a broken URL." Every resolution path is wrapped so it can only ever yield a URL or `None` — a decorative row must
   never fail an agent launch (the whole script already runs under a 20 s subprocess budget in
   `clan_summary_script.py`).

7. **Renderers stay pure; `main()` does the resolving.** `_render_plan_summary` and `_render_epic_summary` take
   `page_url` as a keyword argument rather than resolving internally. That keeps them filesystem-free for unit tests
   and, critically, lets the ACE PNG visual fixture — which calls `_render_plan_summary` directly — pass a fixed URL
   instead of touching git.

8. **`render_plan_document` gains an opt-in parameter, not automatic behavior.** Deriving the row from
   `summary.provenance` inside the renderer would silently add it to the ACE agent PLAN lane and
   `sase clan-summary-plan` too. Opt-in keeps this change scoped to the epic clan summary the request named, and makes
   "output is byte-identical when the parameter is omitted" a testable guarantee.

### Rust core boundary

No `sase-core` change. The `BEAD` target is already parsed and returned by the Rust-owned `sdd_plan_header_block_parse`
binding; `_plan_provenance_sections` in Python just discards it today. Everything added here is presentation shaping
plus a call to the existing Python `sase.bead_pages.links` seam, so it belongs in this repo.

## Changes

### 1. `src/sase/sdd/_plan_display_models.py`

Add a `targets: tuple[str | None, ...] = ()` field to `PlanProvenanceSection`, positionally parallel to `entries`.
Document that a `None` element means "this entry had no hosted destination" and that `targets` is either empty (no
target information) or the same length as `entries`.

### 2. `src/sase/sdd/_plan_display_loading.py`

In `_plan_provenance_sections`, populate `targets` alongside `entries`:

- list-shaped sections (`AGENTS`, `COMMITS`) → `tuple(entry.target for entry in section.entries)`
- link-shaped sections (`PLAN`, `PROMPT`, `PARENT`, `BEAD`) → `(section.target,)`

Behavior is otherwise unchanged; a malformed header block still yields no sections.

### 3. `src/sase/sdd/_plan_display_rendering.py`

- Add `BEAD_PAGE_ROW_LABEL = "   Page: "` (9 cells, aligning with `  Bead:` and ` Title:`).
- Add `bead_page_url_text(url: str) -> Text`: split on the last `/`; render the prefix (including the slash) with
  `COLOR_PLAN_PATH` and the remainder with `COLOR_PLAN_PATH_BASENAME`. A URL with no `/` renders entirely in the
  basename style.
- Add `bead_page_wrap_hint(url: str) -> tuple[int, int]`: return the character offset of the final segment and its
  `cell_len`, shaped for `_render_field_lines`' `preferred_break_before` / `preferred_segment_width` arguments.
- Extend `render_plan_document(...)` with a keyword-only `bead_page_url: str | None = None`. When it is not `None`, emit
  the `Page:` row through
  `_render_field_lines(BEAD_PAGE_ROW_LABEL, bead_page_url_text(url), width=width, preferred_break_before=…, preferred_segment_width=…)`
  immediately after the `BEAD` provenance row, or after the last provenance row when the plan has no `BEAD` row. When it
  is `None`, output must be byte-identical to today.
- Export the two new helpers plus `COLOR_PLAN_PATH` and `COLOR_PLAN_PATH_BASENAME` from `__all__`.

Leave `render_plan_lines`, `plan_logical_text`, and `plan_provenance_rows` signatures alone.

### 4. `src/sase/sdd/plan_display.py`

Re-export `COLOR_PLAN_PATH`, `COLOR_PLAN_PATH_BASENAME`, `BEAD_PAGE_ROW_LABEL`, `bead_page_url_text`, and
`bead_page_wrap_hint`, keeping the import block and `__all__` sorted as they are today.

### 5. `src/sase/bead_pages/links.py`

Add a public, never-raising convenience wrapper next to the existing helpers:

```python
def resolve_bead_page_url_from_cwd(
    bead_id: str,
    *,
    cwd: str | os.PathLike[str] | None = None,
) -> str | None:
    """Return *bead_id*'s hosted page URL for the checkout at *cwd*, or ``None``."""
```

It resolves the store through the existing private `_resolve_store`, delegates to `resolve_bead_page_target` (so the
"page can actually exist" guard still applies), and converts every exception into `None`. Add it to `__all__`.

Note in its docstring how it differs from `sase.bead.cli_detail.resolve_bead_page_url`, which deliberately skips the
existence check for `sase bead show`; do **not** refactor that function here.

### 6. `src/sase/scripts/sase_clan_summary_epic.py`

- `_render_plan_summary(epic_id, summary, *, page_url: str | None = None)` → forward `page_url` to
  `render_plan_document(summary, width=_SUMMARY_WIDTH, bead_page_url=page_url)`. Header-line replacement is unchanged.
- `_render_epic_summary(epic, phases, child_epics=(), *, page_url: str | None = None)` → when `page_url` is set, append
  a `_DocumentBlock` with `omission_kind="page"` after the existing plan block. Its lines are a `Page:` row (label
  `"Page:"` styled `bold dim`, matching the neighbouring `Plan:` row) followed by `bead_page_url_text(page_url)`,
  wrapped at `_SUMMARY_WIDTH` with the same final-segment break hint and a 6-space continuation indent. Emit a leading
  blank `Text()` only when no plan block precedes it, so the two rows sit in one uninterrupted region.
- `_omission_line` → when `remaining["page"]` is non-zero, add the label `"bead page link"` so a truncated summary still
  says what was dropped.
- Add `_plan_bead_page_url(summary: PlanDisplay) -> str | None`: locate the `BEAD` provenance section; return its first
  non-empty `targets` entry; otherwise, when it has a label, delegate to `_resolve_bead_page_url(label)`; otherwise
  `None`.
- Add `_resolve_bead_page_url(bead_id: str) -> str | None`: a `try`/`except Exception` wrapper around
  `resolve_bead_page_url_from_cwd`, imported lazily inside the function so the fast plan path never pays for the
  `sdd.store` import chain.
- `main()` → plan path passes `page_url=_plan_bead_page_url(plan)`; bead path passes
  `page_url=_resolve_bead_page_url(snapshot.epic.id)`. Neither may raise, and the existing fallback and stderr
  diagnostics stay exactly as they are.

### 7. `docs/beads.md`

In the "Bead pages" section, add one sentence noting that an epic agent clan's summary panel shows the hosted page URL
for its epic bead when one resolves, and that `sase plan links refresh` is what repairs a plan whose `BEAD` bullet
predates hosted links.

## Tests

### `tests/test_bead/test_clan_summary_epic_plan_script.py`

- A `PlanDisplay` whose `BEAD` provenance section carries a target renders a `Page:` row directly after the `Bead:` row;
  assert the plain text, that every line is `<= 76` cells, and that the wrap falls immediately before the page filename
  (line one ends with `/`).
- A `PlanDisplay` with a `BEAD` label but no target makes `main()` fall back to `_resolve_bead_page_url` (monkeypatched)
  and renders the row.
- A plan with no `BEAD` section renders no `Page:` row **and** performs no resolution: monkeypatch
  `_resolve_bead_page_url` to raise `AssertionError`, mirroring the existing `_patch_unexpected_bead_load` guard.
- A resolver that raises still produces a summary with no `Page:` row and exit code 0.
- Assert the rendered `Page:` row carries both the prefix and the bold basename style.

### `tests/test_bead/test_clan_summary_epic_bead_script.py`

- `_render_epic_summary(epic, phases, page_url=…)` places `Page:` after the `Plan:` block, keeps all lines `<= 76`
  cells, and breaks before the page filename.
- With `page_url=None`, output is unchanged from today (extend `test_epic_summary_omits_absent_optional_sections`).
- An epic with no `design` but a `page_url` still gets a correctly separated `Page:` region.
- A summary truncated past the page block reports `"bead page link"` in its omission line.

### `tests/test_plan_display.py`

- `render_plan_document(summary, width=…)` with no `bead_page_url` is byte-identical to the current output — the
  regression guard for the ACE plan lane and `sase clan-summary-plan`.
- With `bead_page_url` set, the row lands after the `BEAD` row; with a plan that has provenance but no `BEAD` row, it
  lands after the final provenance row.
- `bead_page_url_text` and `bead_page_wrap_hint` handle a trailing-slash URL, a URL with no `/`, and a multi-byte page
  name without raising or miscounting cells.

### `tests/sdd/` plan-display loading

- `_plan_provenance_sections` carries link-shaped targets (`BEAD`) and list-shaped targets (`AGENTS`, `COMMITS`) through
  to `PlanProvenanceSection.targets`, keeps `targets` aligned with `entries`, and still returns `()` for an unparseable
  header block.

### `tests/ace/tui/visual/test_ace_png_snapshots_agents_clan_panel.py`

- Give the `_EPIC_CLAN_SUMMARY` fixture a `provenance` tuple containing a `BEAD` `PlanProvenanceSection` for `sase-6n`,
  and pass an explicit `page_url="https://github.com/sase-org/sase--beads/blob/main/pages/sase-6n/README.md"` to
  `_render_plan_summary` so the golden covers the wrapped row without any git or store access.
- Add `assert_page_svg_contains(page, "Page:")`.
- Regenerate the affected goldens with `just test-visual -- --sase-update-visual-snapshots`, then rerun
  `just test-visual` clean and eyeball the accepted PNGs in `tests/ace/tui/visual/snapshots/png/` to confirm the row
  reads well at every fold level.

## Verification

```bash
just install
just check
just test-visual
```

Then confirm end to end against a real epic: `sase bead pages url <epic-bead-id>` must print the same URL that the clan
panel shows for that epic in `sase ace`.

## Non-goals

- **Tale/plan clan summaries** (`src/sase/scripts/sase_clan_summary_plan.py`). The same row would fit there and the
  helpers added here are reusable, but the request is scoped to epic clans and that script carries its own visual
  goldens. Worth a follow-up for consistency.
- **The ACE agent PLAN lane** (`_agent_plan_section.py`) and `sase plan show`.
- **Per-phase bead page links.** Only the epic bead is linked.
- **Clickable/openable affordances** — no OSC-8 hyperlink, no keybinding, no `App.open_url` action.
- **Refactoring `sase.bead.cli_detail.resolve_bead_page_url`**, which intentionally has different (unchecked) semantics.
