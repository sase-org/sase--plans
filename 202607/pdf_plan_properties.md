---
tier: tale
title: Render plan frontmatter as a Properties card in Markdown PDFs
goal: 'Plan PDFs attached to Telegram approval messages (and every other Markdown->PDF
  attachment) render their YAML frontmatter as a beautiful, self-contained "Properties"
  card at the top of the document, using one shared ordering/label source so the card
  stays consistent with the ACE Plans detail pane and the Telegram properties card.

  '
create_time: 2026-07-23 12:40:15
status: done
---

- **PROMPT:** [202607/prompts/pdf_plan_properties.md](prompts/pdf_plan_properties.md)

# Render plan frontmatter as a Properties card in Markdown PDFs

## Problem

When a plan is proposed, SASE attaches a PDF rendering of the plan `.md` to the Telegram plan-approval message. That PDF
shows the plan body but **none of the YAML frontmatter properties** (`tier`, `title`, `goal`, `model`, and, for epics,
`phases`, plus system fields such as `create_time`, `status`, `bead`, `parent`, `bead_id`). The reviewer therefore
cannot see the plan's metadata in the PDF at all — the document jumps straight into the body.

## Root cause

The Markdown->PDF conversion is `render_markdown_pdf()` in `src/sase/attachments/markdown_pdf.py`. It hands the raw
`.md` file directly to `pandoc` (see `_pandoc_cmd()` and the `subprocess.run(cmd, ...)` call inside
`render_markdown_pdf`). Pandoc's Markdown reader natively treats a leading `---` ... `---` YAML block as **document
metadata**, not body content, so the frontmatter is consumed and never rendered. Two consequences:

1. The frontmatter properties disappear from the PDF body.
2. For the `wkhtmltopdf` engine the command adds `--metadata title=<source.stem>`. Because the plan gate attaches a file
   literally named `plan.md`, the PDF's title block renders as the word "plan" instead of the real plan title.

This is confirmed to be the only conversion path for the approval PDF: the `sase-telegram` plugin does **not** run its
own pandoc; its `pdf_convert.py` calls straight into `sase.attachments.markdown_pdf.render_markdown_pdf(...)`. So a fix
in this repo covers the Telegram approval PDF and every other Markdown->PDF attachment at once.

## Design overview

Preprocess the Markdown inside `render_markdown_pdf()` so that, when the source has YAML frontmatter, a rendered
**Properties card** is injected at the top of the body before pandoc runs. The card lists every top-level frontmatter
property as a label -> value row, in a stable, human-friendly order, styled to look like a compact metadata card
(echoing the existing Telegram "Properties" card and Obsidian's Properties panel).

To keep this **reliable** and avoid a third divergent copy of "how plan properties are ordered and labeled", extract the
ordering/label/value-rendering logic that is currently **duplicated** in two places today —

- ACE Plans detail pane: `src/sase/ace/tui/widgets/artifacts/plans_detail.py` (`_KNOWN_FRONTMATTER_KEYS`,
  `_ordered_frontmatter_items`, `_property_label`), and
- the `sase-telegram` plugin's `formatting.py` (`_PLAN_PROPERTY_ORDER`, `_humanize_plan_property_label`,
  `_render_plan_value_lines`), which carries a hand-maintained comment "Keep plan properties aligned with the ordering
  in ACE's Plans detail surface" —

into a single shared module `src/sase/sdd/plan_properties.py`. The PDF renderer and the ACE pane both consume it.
Because the shared module reproduces the exact ordering the Telegram card already uses, the new PDF card is
automatically consistent with the Telegram message the reviewer sees next to it, with no change required in the
`sase-telegram` repo. (Migrating the Telegram card to import this module too is a small follow-up — see "Out of scope".)

### Placement note (Rust core boundary)

The repo's `rust_core_backend_boundary` rule sends shared cross-frontend behavior to `sase-core`. This change
intentionally keeps the shared logic in Python because: (a) the identical logic already lives in Python presentation
layers today (ACE `plans_detail.py` and the Telegram plugin), i.e. the codebase already treats plan-property _display
ordering/labels_ as presentation, not core validation; (b) `sase-core` already owns the plan _schema/validation_
(`plan_frontmatter_schema`, `plan_validate`) but has no values-display projection, so this would be net-new Rust with no
existing home; and (c) every consumer is Python and depends on this package. The new module sits alongside the existing
shared plan-presentation module `src/sase/sdd/plan_display.py`. If the reviewer prefers this projection to live in
`sase-core` instead, that is a clean redirection at approval time.

## Detailed design

### 1. New shared module: `src/sase/sdd/plan_properties.py`

Create a small, dependency-light module that is the single source of truth for plan-frontmatter _display_:

- `PLAN_PROPERTY_ORDER: tuple[str, ...]` — the canonical leading order, matching today's duplicated constants exactly:
  `("title", "tier", "kind", "status", "create_time", "created", "created_at", "goal")`.
- `plan_property_label(key: str) -> str` — humanize a key (`key.replace("_", " ").strip().capitalize()`), falling back
  to the raw key when empty. Matches ACE/Telegram behavior.
- `ordered_plan_property_items(frontmatter: Mapping[str, Any]) -> list[tuple[str, Any]]` — return all top-level items
  ordered by `PLAN_PROPERTY_ORDER` first (case- folded), then remaining keys alphabetically (case-folded). Mirrors
  `_ordered_frontmatter_items` / `_ordered_plan_properties`.
- `render_plan_value_lines(value: Any) -> list[str]` — deterministic, recursive rendering of a parsed YAML value into
  display lines, mirroring the Telegram `_render_plan_value_lines` semantics: `None` -> `—`; `bool` -> lowercase; `date`
  -> ISO; strings kept (empty -> `—`); sequences -> `• item` lines with nested indentation; mappings -> `key: value`
  lines with nested indentation; empty list/dict -> `[]` / `{}`; and a recursion guard for self-referential structures.
- Convenience: `plan_property_rows(frontmatter) -> list[tuple[str, list[str]]]` returning `(label, value_lines)` for
  each ordered property, so the PDF card (and, later, the Telegram card) can build its layout from one call.

Add focused unit tests in `tests/sdd/test_plan_properties.py` covering ordering (known-before-unknown, alphabetical
tail), label humanization, and value rendering for scalars, lists, nested mappings (epic `phases`), empties, and
`None`/`bool`/`date`.

### 2. PDF renderer: inject the Properties card

In `src/sase/attachments/markdown_pdf.py`, add a preprocessing step to `render_markdown_pdf()`:

- Add a parameter `include_properties: bool = True`.
- After the existing source validation (supported extension, destination is a PDF, source exists) and before building
  the pandoc command, when `include_properties` is true:
  1. Read the source text and call `sase.sdd.frontmatter.parse_frontmatter(content)` to get
     `(frontmatter, body, had_frontmatter)`.
  2. If `had_frontmatter` and the frontmatter is non-empty, build the card markup (see section 3) via
     `plan_property_rows(frontmatter)` and write a preprocessed temporary `.md` file in the destination directory whose
     content is `<card>\n\n<body>` — i.e. the frontmatter is removed (so pandoc does not re-consume it) and the rendered
     card takes its place at the top of the body. Hand this temp file to pandoc as the source for the engine loop.
  3. Otherwise (no/empty frontmatter, or `include_properties` false), behave exactly as today (pass the original
     source).
- **Title metadata:** capture the _original_ source before any swap. Thread an explicit `title` into `_pandoc_cmd`: use
  the frontmatter `title` value when present, else the original `source.stem`. This replaces the current
  `--metadata title=<source.stem>` so (a) the temp filename never leaks into the title and (b) plan PDFs show the real
  plan title instead of "plan". Keep this wired only into the existing `wkhtmltopdf` title branch; non-HTML engines are
  unchanged.
- **Cleanup & safety:** the preprocessed temp file is removed in the existing `finally` cleanup alongside the temporary
  PDF. Any exception during preprocessing (read error, YAML edge case, etc.) must be caught and fall back to rendering
  the original source unchanged — the card is strictly additive and must never break an otherwise-working render.
- `render_launch_preview_pdf()` calls `render_markdown_pdf(...)` for launch prompt previews, which are prompt bodies
  (and may carry unrelated xprompt prompt frontmatter). Pass `include_properties=False` there so launch previews keep
  their current output.

### 3. Card markup and styling (self-contained + beautiful)

The Telegram approval PDF is styled by the _telegram plugin's_ own `pdf_style.css` (it is passed to
`render_markdown_pdf` as `css_path`), **not** this repo's `markdown_pdf.css`. To make the card beautiful on that surface
without editing the telegram repo, the card must carry **self-contained styling** rather than rely on the caller's
stylesheet.

Design the card as a raw HTML block (pandoc passes raw HTML through, and the primary/expected engine `wkhtmltopdf`
renders it) with inline styles that match the existing PDF palette (background `#f6f8fa`, border `#d8dee4`, heading
color `#111827`, muted label color `#57606a`, radius ~4px):

- A titled container ("Properties"), then one row per property: a bold/muted label and its value. Multi-line values
  (lists, nested `phases`) render as indented lines / nested bullets.
- HTML-escape every label and value so property values containing `<`, `>`, `&`, or Markdown never break the layout or
  inject markup.
- Additionally add matching `.sase-properties` rules to `src/sase/attachments/markdown_pdf.css` so callers that use the
  default stylesheet get the same look; the inline styles guarantee correctness even when a different `css_path` (e.g.
  Telegram's) is supplied.

Exact markup (inline-styled table vs. definition list vs. card grid) is finalized during implementation and confirmed by
visual QA (section 5). Requirement: the card is readable and on-brand under `wkhtmltopdf`, and — because raw HTML is
dropped by LaTeX engines — implementers should prefer a form that still degrades to visible property text under the
`xelatex`/`pdflatex` fallback (e.g. wrap a pandoc-native table in the styled container, or emit a plain-text fallback).
In practice `wkhtmltopdf` is the installed/first engine and the mobile profile is tuned for it.

### 4. ACE Plans detail pane: consume the shared module

Refactor `src/sase/ace/tui/widgets/artifacts/plans_detail.py` to import from `sase.sdd.plan_properties`:

- Replace the local `_KNOWN_FRONTMATTER_KEYS` with `PLAN_PROPERTY_ORDER`.
- Replace `_property_label` with `plan_property_label`.
- Have `_ordered_frontmatter_items` delegate to `ordered_plan_property_items`.

This is a pure refactor: the constants and label function are identical to today's, so the ACE Plans detail rendering
(and its PNG snapshots) must not change. Keep ACE's existing value stringification (its frontmatter arrives as
`dict[str, str]`); the richer `render_plan_value_lines` is only needed by the PDF path, which parses real YAML objects.
This step is what makes the new module a genuine shared source rather than a third copy.

## Reliability / edge cases

- No frontmatter, empty frontmatter, or unparseable YAML -> no card; original render behavior preserved (never raise).
- Preprocessing failure of any kind -> fall back to the original source.
- Values are HTML-escaped; nested/recursive structures are rendered safely with a recursion guard.
- pandoc or a PDF engine missing -> unchanged existing skip/return-None behavior.
- Non-plan Markdown with frontmatter still gets a correct card (unknown keys sort alphabetically after the known order);
  this is intended and consistent.

## Beauty

- Card sits at the very top of the document, so the reviewer sees metadata first, mirroring Obsidian's Properties panel
  and the Telegram "Properties" card.
- Palette and typography match the existing PDF CSS for a cohesive look.
- The PDF title block now shows the real plan title instead of "plan".

## Testing strategy

- `tests/sdd/test_plan_properties.py` — unit tests for ordering, labels, value rendering (deterministic; no external
  tools).
- `tests/attachments/` (mirror existing test layout) — unit tests for the card builder: given frontmatter, assert the
  injected Markdown/HTML contains the expected escaped labels/values in order, and that no-frontmatter input is a no-op.
  These need no pandoc.
- A pandoc-gated smoke test (skipped when pandoc/engine unavailable, matching the renderer's existing tool checks):
  render a sample tale and a sample epic plan and assert a PDF is produced. Optionally assert extracted text contains a
  property label.
- ACE: run `just test-visual`; the `plans_detail` refactor must leave PNG snapshots green (pure refactor). Update
  snapshots only if an intentional visual change is chosen, using `--sase-update-visual-snapshots`.
- Run `just check` (install first per repo convention) before completion.
- Manual visual QA: render a real tale and a real epic plan (the epic to confirm `phases` render acceptably) and eyeball
  the card for beauty; optionally attach to the approval message.

## Out of scope / follow-ups

- Migrating the `sase-telegram` plugin's `formatting.py` properties card to import `sase.sdd.plan_properties` (removing
  its duplicated constants and the "keep aligned" comment). Not required for this fix — the PDF already matches the
  Telegram card because the shared module reproduces the same ordering — but a worthwhile fast-follow for full dedup,
  and cross-repo so best done separately.
- Relocating the projection into `sase-core` (see "Placement note").
- Any PDF page-layout or profile changes beyond the new card.

## Files to change

- Add `src/sase/sdd/plan_properties.py` (+ `tests/sdd/test_plan_properties.py`).
- Edit `src/sase/attachments/markdown_pdf.py` (preprocessing, `include_properties` param, title threading in
  `_pandoc_cmd`, `render_launch_preview_pdf` opt-out).
- Edit `src/sase/attachments/markdown_pdf.css` (`.sase-properties` rules).
- Edit `src/sase/ace/tui/widgets/artifacts/plans_detail.py` (consume shared module).
- Add card-builder tests under `tests/attachments/`.

## Acceptance criteria

- A proposed plan's Telegram approval PDF shows all top-level frontmatter properties as a styled Properties card at the
  top, in `PLAN_PROPERTY_ORDER`, with humanized labels and correctly rendered scalar/list/nested values.
- The PDF title block shows the plan's real `title` (not "plan").
- Markdown PDFs without frontmatter are byte-for-behavior unchanged; launch preview PDFs are unchanged.
- ACE Plans detail pane behavior and PNG snapshots are unchanged.
- Rendering never crashes on malformed/edge-case frontmatter; failures fall back to the pre-existing render.
- `just check` and `just test-visual` pass.
