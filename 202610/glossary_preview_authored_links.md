---
tier: tale
title: Glossary preview honors authored [[target]] links
goal:
  The glossary preview card renders authored [[target]] links as term links and lists
  their glossary targets under SEE ALSO.
size: medium
decisions:
  see_also_order:
    ask:
      Should authored [[links]] take the first SEE ALSO chips, ahead of phrase mentions?
    choices:
      links_first:
        Link targets fill chips first and never fall past the 9-chip cap; mentions
        follow
      position:
        One first-occurrence order across links and mentions, like the Memory panel
        strand card
    default: links_first
    why:
      Deliberate links should not be crowded out of the 1-9 keys by incidental phrase
      matches
    answer: links_first
  foreign_links:
    ask:
      Should the card also list links that leave the glossary (e.g. [[decisions/x]]) or
      do not resolve?
    choices:
      skip:
        The card lists glossary terms only; those links stay as authored text in the
        definition
      list:
        Adds an unnumbered display-only LINKS row naming each non-glossary or unresolved
        target
    default: skip
    why: SEE ALSO numbers are follow keys, and the card can only follow glossary terms
    answer: skip
proposed_by: bbugyi200.athena.16n
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-31 11:45:55
status: wip
---

# Glossary preview: respect authored `[[target]]` links in the definition and SEE ALSO

## Problem

`K` on a highlighted glossary phrase opens `GlossaryPreviewModal`
(`src/sase/ace/tui/modals/glossary_preview_modal.py`). Its SEE ALSO chips and its
definition hyperlinks come only from implicit phrase matches
(`scan_glossary_spans(catalog.compiled, definition)`). Authored memory links such as
`[[project-note]]`, `[[glossary:tool-run]]`, `[[glossary/artifact-reference]]` and
`![[...]]` are ignored. In practice:

- bob-cli's `Task Successor` strand links `[[project-note]]`, `[[project-task]]`,
  `[[task-link]]` and `[[successor-offer]]`, all of which are real glossary strands
  there, but SEE ALSO shows only `1 Pomodoro` and the links render as literal `[[...]]`
  text.
- In this repo, `glossary:receipt` authors four `[[glossary:...]]` links and its card
  shows **no** SEE ALSO chips.
- Phrase matches inside link brackets are hyperlinked, which corrupts the text:
  `[[glossary:proc]]` renders as `[[glossary:[proc](glossary:43)]]`.

The Memory panel strand card and `sase memory read glossary:<term>` already fold
same-web authored links into the same closure (see `strand_link_spans` /
`build_strand_mention_catalog` in `src/sase/memory/web/closure.py`, and
`selector_web.py`). `docs/reference/tui/prompt-completion-panels.md` promises that the
preview card and the CLI "can never disagree about which terms a definition references".
Right now they do disagree.

## Approach

Resolve each glossary definition's authored links once, off the event loop, when the
editor glossary catalog loads. The TUI already warms it with `asyncio.to_thread`
(`_startup_prompt_catalog_semantics._run_prompt_glossary_warm`). Store the results on
`EditorGlossaryCatalog`. The preview's pure render helpers then merge these links into
the span list they already feed to the definition Markdown and to
`resolve_glossary_closure`. Link scanning and resolution reuse the existing Python
memory-link code (`sase.memory.links.scan_memory_links` plus closure.py's
`_local_strand_target`), which already backs `sase memory read` and the Memory panel. No
Rust change is needed, and porting link scanning to sase-core is out of scope.

Nothing new runs on the render path: the modal's `_set_entry` already scans mentions for
one definition, and the change adds only a dict lookup plus a small merge.

## Changes

### 1. `src/sase/memory/links.py`: allow every occurrence

`scan_memory_links` de-duplicates by `(target, inline)`, so a second `[[task-link]]` in
the same body is never returned. The preview needs every occurrence to render each one
and to suppress mention matches inside each one. Add a keyword-only
`unique: bool = True` parameter. When `False`, skip the `seen` filter. Existing callers
keep the default, so their behavior is unchanged.

### 2. `src/sase/memory/web/closure.py`: per-definition link records

Add a public frozen dataclass and a builder next to `strand_link_spans`, so the builder
can use the private `_local_strand_target`:

```python
@dataclass(frozen=True, slots=True)
class GlossaryDefinitionLink:
    """One authored [[target]] / ![[target]] link inside a glossary definition."""

    raw: str            # e.g. "[[glossary:tool-run]]" or "![[x]]"
    target: str         # the stripped inner target text
    byte_start: int     # UTF-8 byte offsets into entry.definition.strip()
    byte_end: int
    target_index: int | None  # same-web catalog entry index; None if foreign/unresolved


def glossary_definition_links(
    web: MemoryWeb,
    strands: tuple[MemoryStrand, ...],
    entries: tuple[GlossaryEntry, ...],
) -> dict[int, tuple[GlossaryDefinitionLink, ...]]:
```

Rules:

- `strands` and `entries` are parallel; use `zip(..., strict=True)`. The caller passes
  `ordered_web_strands(web)`, which is the order `memory_web_glossary_entries` used to
  build the catalog. Build `slug -> entry.index` from that zip.
- Skip strands whose effective `link_reference == "none"` (same rule as
  `strand_link_spans`).
- Scan `entry.definition.strip()` with `scan_memory_links(..., unique=False)`, because
  the preview's mention spans are byte offsets into that same stripped text. Convert
  each `link.span` from char offsets to UTF-8 byte offsets
  (`len(text[:i].encode("utf-8"))`).
- Resolve `link.target` with `_local_strand_target(replace(web, strands=strands), ...)`.
  That covers bare slug/keyword/alias/unique-prefix and the `glossary:x` / `glossary/x`
  forms, and returns `None` for other webs. One exception: a bare target ending in `.md`
  (no `:` or `/`) is always a flat-note link and never a glossary strand. This matches
  `resolve_memory_link_target`, which `sase memory read` uses. Map the resolved slug to
  its entry index, or leave `target_index=None`.
- Keep self-links: `target_index` equals the entry's own index, and consumers decide
  what to do with them.
- Omit entries with no links from the returned dict. Add the new names to `__all__`.

### 3. `src/sase/macro/glossary_catalog.py`: carry links on the catalog

- Add
  `definition_links: Mapping[int, tuple[GlossaryDefinitionLink, ...]] = field(default_factory=dict)`
  as the **last** field of `EditorGlossaryCatalog`, so the many existing test
  constructors keep working.
- In `_web_editor_glossary_catalog`, after `build_glossary_catalog`, compute
  `glossary_definition_links(web, ordered_web_strands(web), catalog.entries)`. Wrap it
  so any exception degrades to `{}`: a link problem must never cost the user the
  glossary catalog. Pass the result into the constructor.
- Leave `to_wire()` unchanged, so the LSP catalog payload stays byte-identical.

### 4. `src/sase/ace/tui/modals/glossary_preview_render.py`: merge links into spans

All helpers keep accepting the duck-typed `catalog: Any` that tests build with
`SimpleNamespace`. Read links with
`(getattr(catalog, "definition_links", None) or {}).get(entry.index, ())`.

- `glossary_reference_spans(catalog, entry)` returns mention spans plus link spans,
  sorted by `(byte_start, byte_end)`:
  - Drop every mention span that overlaps **any** authored link range, whether resolved,
    foreign or unresolved. This fixes the `[[glossary:[proc](...)]]` corruption and
    stops `[[decisions/goal-ledger]]` from mention-matching "goal".
  - For each link whose `target_index` is in the catalog, build a synthetic
    `GlossarySpan(term=<target term>, entry_index=target_index, alias_index=-1, alias="", matched_text=link.raw, byte_start, byte_end, range={}, segments=(), kind="link")`.
    This mirrors `build_strand_mention_catalog`. Include self-links here; both the
    closure and the Markdown already special-case self.
  - Keep the existing "degrade to no spans on failure" behavior for the mention scan.
- `glossary_definition_markdown`: for a `kind == "link"` span, use the **target entry's
  term** as the label instead of the matched text, so `[[project-note]]` becomes
  `[Project Note](glossary:<index>)` and `[[glossary:failure-signature]]s` becomes
  `[Failure Signature](glossary:<index>)s`. Escape the label with
  `_escape_markdown_link_text`. A self-targeting link span renders as the plain term
  text (no link). Self _mention_ spans keep today's behavior (left as authored text).
  Foreign and unresolved links have no span, so they stay verbatim.
- `glossary_cross_references`: the closure call is unchanged. It already walks
  precomputed spans, link spans included, drops self and de-duplicates.

> [!decision] see_also_order = links_first After the closure, stable-partition the
> related entries: first those targeted by at least one `kind == "link"` span in this
> definition (in first-link order), then the mention-only ones (in first-mention order).
> Then apply the existing cap of 9.

> [!decision] see_also_order = position Keep the closure's first-occurrence order as is,
> then apply the cap of 9.

> [!decision] foreign_links = list Add
> `glossary_foreign_link_targets(catalog, entry) -> tuple[str, ...]`, which returns the
> de-duplicated `link.target` of every link with `target_index is None`, in order. Also
> add `build_link_chips(targets, *, accent)`, which draws unnumbered chips the same way
> `build_alias_chips` does. In `GlossaryPreviewModal._build_chip_rows`, append a
> `("LINKS", chips)` row after SEE ALSO when non-empty. Cache the targets in
> `_set_entry` next to `_references` so `_build_meta` does no extra work. The chips are
> display-only: add no key binding and no footer hint.

> [!decision] foreign_links = skip No modal change. `GlossaryPreviewModal` already gets
> everything through `glossary_reference_spans` / `glossary_cross_references` /
> `glossary_definition_markdown`, and the footer's `1-9 see also` hint stays correct.

### 5. Docs: `docs/reference/tui/prompt-completion-panels.md` (Glossary terms section)

This is a behavior change to an existing feature, so update the Reference page in place.
State that SEE ALSO chips cover glossary terms the definition mentions **or links with
`[[target]]` / `![[target]]`**. State that resolved links render as followable term
links in the definition, and describe the decided ordering. Correct the parity paragraph
so it says both the card and `sase memory read` walk mention spans plus same-web
authored links. Under `foreign_links = list`, add one sentence about the display-only
`LINKS` row. Keep the page within its budget: edit sentences, don't add a section.

## Tests

- `tests/memory/test_memory_links.py`: `unique=False` returns repeated occurrences with
  their own spans, and the default stays de-duplicated.
- New `tests/memory/test_memory_web_definition_links.py` (build a small web in
  `tmp_path`, following `tests/memory/test_memory_web.py` /
  `tests/macro/test_glossary_catalog.py` fixtures):
  - bare slug, `glossary:x`, `glossary/x`, keyword and alias targets resolve to the
    right entry index
  - `[[decisions/x]]`, a bare `[[foo.md]]` and an unknown target give
    `target_index=None`
  - a self-link keeps its own index
  - links inside backtick code are ignored
  - a strand with `link_reference: none` yields nothing
  - byte offsets stay correct when non-ASCII text (e.g. `⌖`) precedes a link, and after
    leading whitespace is stripped
  - a repeated link yields two records
- `tests/macro/test_glossary_catalog.py`: a strand-backed web whose strand authors
  `[[other-term]]` loads with `definition_links` populated. `to_wire()` gains no new
  key.
- `tests/ace/tui/modals/test_glossary_preview_render.py` (SimpleNamespace catalogs with
  a `definition_links` mapping):
  - SEE ALSO includes link targets that no phrase matches
  - ordering follows the decided `see_also_order`
  - the Markdown turns `[[project-note]]` into `[Project Note](glossary:N)`
  - a mention inside a resolved link and a mention inside a foreign link both stay
    unlinked
  - a self-link renders as plain term text
  - foreign/unresolved links stay verbatim
  - a catalog without `definition_links` behaves exactly as today (existing tests keep
    passing unchanged)
- `tests/ace/tui/modals/test_glossary_preview_modal.py`: pressing `1` follows a chip
  that exists only because of an authored link. Under `foreign_links = list`, the meta
  also renders a `LINKS` row and the footer is unchanged.
- PNG goldens: none of the existing glossary preview visual fixtures contain `[[`, so
  `glossary_preview_card_*` goldens must not change. No new golden is required.

## Verification

Run `just fix` (format + keep-sorted), then `sase tool run check`. Do not run
`just check-full`. Optionally confirm visually: open `sase tui`, type a prompt
containing a glossary term whose definition authors links (e.g. "receipt" in this repo),
press `K`, and check that SEE ALSO lists the linked terms and `1`-`9` follows them.

## Non-goals

- Making the definition's Markdown hyperlinks clickable with the mouse (no
  `Markdown.LinkClicked` handling exists today for mention links either).
- Following non-glossary links from the card (for example into the Memory panel), or a
  REFERENCED BY row on the preview card.
- Changing the Memory panel or `sase memory read` closure semantics, the LSP catalog
  wire, or moving memory-link scanning into sase-core.
