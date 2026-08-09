---
tier: tale
title: Drop Matched and uppercase the field labels in the ACE glossary preview
goal:
  The `K` glossary preview panel renders `ALIASES:`, `PROJECT:`, and `SOURCE:` field
  labels and no longer renders a redundant `Matched:` field.
size: medium
proposed_by: bbugyi200.athena.wg
create_time: 2026-08-09 09:09:33
status: wip
---

# Drop `Matched:` and uppercase the field labels in the ACE glossary preview

## Problem

Pressing `K` on a glossary phrase in the ACE prompt input opens the Markdown preview
panel (`PreviewPanelModal`) with the term, its definition, and a block of metadata
fields at the bottom:

```
Aliases: task block link

Project: bob-cli

Source: /home/bryan/projects/github/bobs-org/bob-cli/sase/sase.yml:12:17

Matched: task link
```

Two problems with that block:

1. **`Matched:` is redundant.** The modal title block already renders the matched text
   as the panel subtitle, immediately left of the source path
   (`src/sase/ace/tui/modals/preview_panel_modal.py:210-216`, fed by
   `PreviewPayload.reference`). In the screenshot that is the
   `task link  →  /home/bryan/projects/github/bobs-org/bob-cli/sase/sase.yml` line under
   the `G GLOSSARY RENDERED  Task Link` header. The `Matched:` field repeats it at the
   bottom of the body.
2. **The labels read as prose, not as field markers.** SASE now renders glossary field
   labels in all caps, so these should match.

## Interpretation of "capitalize the keys"

The three surviving labels are already title-case (`Aliases:`, `Project:`, `Source:`),
so "capitalize" here means **all caps** — `ALIASES:`, `PROJECT:`, `SOURCE:`. That is the
convention already adopted for the generated glossary memory: `sase memory init` renders
`ALIASES:` via `GLOSSARY_ALIASES_LABEL`
(`src/sase/main/init_memory/glossary.py:30,151`), landed by the earlier plan
`202608/glossary_aliases_uppercase_label.md`, which explicitly deferred this TUI surface
as out of scope. This plan brings the TUI preview in line.

Do **not** bold, colorize, or otherwise restyle the labels — case only.

## Current behavior

All four labels come from one function,
`src/sase/ace/tui/widgets/_prompt_glossary.py:368-391`:

```python
def _glossary_preview_markdown(
    catalog: EditorGlossaryCatalog,
    span: GlossarySpan,
    entry: GlossaryEntry,
) -> str:
    lines = [
        f"# {entry.term}",
        "",
        entry.definition.strip(),
    ]
    aliases = tuple(alias for alias in entry.configured_aliases if alias)
    if aliases:
        lines.extend(("", f"Aliases: {', '.join(aliases)}"))
    project = getattr(catalog.project, "name", None) or getattr(
        catalog.project, "key", ""
    )
    if project:
        lines.extend(("", f"Project: {project}"))
    source = _glossary_source_display(catalog, entry)
    if source:
        lines.extend(("", f"Source: `{source}`"))
    if span.matched_text and span.matched_text != entry.term:
        lines.extend(("", f"Matched: {span.matched_text}"))
    return "\n".join(lines).rstrip() + "\n"
```

It has exactly one caller, `_preview_glossary_under_cursor()` at line 136, which builds
the `PreviewPayload`. Note line 138 in that same call: `reference=span.matched_text` —
that is the subtitle described above and it must stay.

The `Matched:` block only ever rendered when the cursor sat on an **alias** (matched
text differing from the canonical term), which is why the screenshot shows it for
`task link` under the term `Task Link`.

## Implementation steps

### 1. Rewrite the metadata block

In `src/sase/ace/tui/widgets/_prompt_glossary.py`:

- Uppercase the three surviving labels: `ALIASES: `, `PROJECT: `, `SOURCE: `.
- Delete the `if span.matched_text and span.matched_text != entry.term:` block (lines
  389-390) entirely.
- `span` is then unused inside the function, so **drop the parameter** and update the
  single call site at line 136 to `_glossary_preview_markdown(catalog, entry)`.

Keep everything else byte-identical: the `# {entry.term}` heading, the stripped
definition, the blank-line-before-each-field pattern, the empty-alias filter, the
`getattr(...name) or getattr(...key)` project fallback, the backticks around the source
path, and the trailing `.rstrip() + "\n"`.

Do **not** touch `reference=span.matched_text` on line 138 of the `PreviewPayload`
construction. Removing that would delete the matched text from the panel entirely, which
is not what was asked; the whole justification for dropping the body field is that the
subtitle already carries it.

The `GlossarySpan` import stays — it is still used by `_entry_for_span` (line 360) and
the `_glossary_match_under_cursor` return annotation (line 226). Do not remove it.

### 2. Update and extend the widget tests

`tests/ace/tui/widgets/test_prompt_glossary.py`, in
`test_k_on_glossary_term_pushes_markdown_preview` (lines 340-341):

```python
assert "Aliases: clan, agent clans" in payload.content
assert "Project: sase" in payload.content
```

Update both to the uppercase labels. Add a negative assertion for the old casing (e.g.
`assert "Aliases:" not in payload.content`) so a regression to title case fails, and
assert the uppercase `SOURCE:` label as well — the existing source assertion at line 342
only checks that the path substring is present, not the label.

That existing test is **not** a regression guard for the `Matched:` removal: its fixture
puts the cursor on the canonical term, where matched text equalled `entry.term` and the
field never rendered. Add a new test that exercises the alias path:

- `_catalog_for_text()` (line 504) currently forces `entry.term == term == matched_text`
  — it uses the same string for the span wire and the `GlossaryEntry`. Add an optional
  keyword parameter (e.g. `entry_term: str | None = None`, defaulting to `term`) so a
  span can match the alias `clan` while the entry term stays `Agent Clan`. Keep the
  default behavior identical so the nine existing call sites are unaffected.
- The new test presses `K` with the cursor over the alias occurrence and asserts
  `"Matched:" not in payload.content` **and** `payload.reference == "clan"` — i.e. the
  matched text is gone from the body but still reaches the subtitle.

Model the new test on `test_k_on_glossary_term_pushes_markdown_preview` (same
`PromptPage` / `_install_warm_glossary` / `_wait_for(_top_is_preview)` shape).

### 3. Docs: no change required

`docs/ace.md:4342` describes this panel as opening "with the canonical term, definition,
configured aliases, project, and source path". It never documented a matched-text field,
so removing it makes the sentence more accurate, and it enumerates fields rather than
label strings, so the casing change does not touch it. Do not edit `docs/ace.md` for
this change.

`CHANGELOG.md` is generated by release-please from conventional commits — do not
hand-edit it.

## Explicitly out of scope

- **The Rust LSP glossary hover** — `sase-core`,
  `crates/sase_xprompt_lsp/src/server.rs:2726-2752` (`glossary_hover_markdown`). It is a
  separate surface with a deliberately different layout: a bold term, an
  ``Aliases: `clan` `` line with code-spanned aliases, and a trailing
  ``project `x` | source `y` `` meta line. It has no `Matched:`, `Project:`, or
  `Source:` labeled fields, so nothing there is required by this request. If the user
  later wants the editor hover to match the TUI, that is its own change in the
  `sase-core` repo (opened via `/sase_repo`), not a silent add-on here.
- `src/sase/main/init_memory/glossary.py` — already emits `ALIASES:`; unchanged.
- Project-alias surfaces (`src/sase/main/project_handler.py:142`,
  `src/sase/ace/tui/modals/project_management_rendering.py:237`,
  `src/sase/ace/tui/modals/project_alias_editor_modal.py:33`) — unrelated concept.
- `src/sase/main/plan_search_render.py:321` renders a `Matched` row for plan search;
  unrelated.
- The generic preview-panel chrome (title, subtitle, footer keys) in
  `preview_panel_modal.py` — unchanged.

## Rust core backend boundary

This stays in Python. `_glossary_preview_markdown()` is presentation-only Markdown
assembly for one Textual modal; it consumes already-computed core data (`GlossaryEntry`
/ `GlossarySpan` from `sase.core.glossary_facade`) and computes no domain behavior. The
Rust core has no shared "glossary preview body" renderer to update — its LSP hover
builds its own distinct markup (see out of scope above). No wire, binding, or
`sase-core` test changes are needed.

## Verification

```bash
just install    # ephemeral workspace: dependencies may be stale
just check
```

The diff-scoped lane should select `tests/ace/tui/widgets/test_prompt_glossary.py`; if
it does not, run it explicitly:

```bash
just test -- tests/ace/tui/widgets/test_prompt_glossary.py
```

The ACE PNG visual snapshots do **not** need regeneration. The only glossary goldens are
`prompt_glossary_highlight_{dark,light}_120x40.png`, produced by
`tests/ace/tui/visual/test_ace_png_snapshots_prompt_highlighting.py:315` — that test
renders prompt-input highlighting and never opens the preview modal. Do not pass
`--sase-update-visual-snapshots`.

Manual confirmation in a real `sase ace` session (optional but preferred): put the
cursor on a glossary **alias** in the prompt, press `K`, and confirm the body ends at
the `SOURCE:` line while the alias still shows in the subtitle next to the source path.

## Definition of done

- The `K` glossary preview body renders `ALIASES:`, `PROJECT:`, and `SOURCE:` and no
  `Matched:` field.
- `_glossary_preview_markdown()` no longer takes a `span` parameter, and its one call
  site is updated.
- `PreviewPayload.reference` still carries `span.matched_text`, so the panel subtitle is
  unchanged.
- `tests/ace/tui/widgets/test_prompt_glossary.py` asserts the uppercase labels, rejects
  the title-case labels, and has a new alias-cursor test asserting `Matched:` is absent
  from the body while `reference` still holds the matched text.
- `grep -n "Matched:" src/sase/ace/tui/widgets/_prompt_glossary.py` returns nothing.
- The Rust LSP hover, the generated glossary memory, and the project-alias surfaces are
  untouched.
- `just check` passes.
