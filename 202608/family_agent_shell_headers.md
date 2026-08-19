---
tier: tale
title: Family and agent-shell metadata kind headers
goal: "Selecting a family or an agent shell in the Agents tab metadata panel shows the
  same kind of underlined identity heading that clan and tribe selections already have,
  so every common agent-tree selection names its kind before Name and the rest of the
  document.

  "
size: medium
proposed_by: bbugyi200.athena.08e
create_time: 2026-08-19 18:51:09
status: wip
---

# Plan: Family and agent-shell metadata kind headers

## Why this exists

The Agents tab metadata panel already opens with an underlined kind label when the
selection is a grouping container:

- Tribe selection: gold underlined `TRIBE`, then `Name:`
- Clan selection: orchid underlined `CLAN`, then `Name:`

Family containers and agent shells skip that first line and start at `Name:`. The panel
still has plenty of later kind language (`FAMILY MEMBERS`, `AGENT PROMPT`, `MONITOR`),
but the top of the document is the place that answers "what did I just select?" The
missing headings make family and shell selections feel like an unlabeled default, while
clan and tribe feel named.

This tale completes that chrome for the two remaining everyday node kinds. It does not
invent a new panel, fold scale, or roster.

## Design

### Visual language

Keep the clan/tribe recipe. The kind label is document chrome, not a navigable section:

- One all-caps label on its own line
- Style `bold <identity-color> underline` as a **single span** (the space in
  `AGENT SHELL` is part of that span, so the underline is continuous)
- Immediately followed by the existing `Name:` field — no blank line
- Identity color matches the name that already belongs to that kind

| Selection        | Label         | Color     | Why this color                                                     |
| ---------------- | ------------- | --------- | ------------------------------------------------------------------ |
| Tribe            | `TRIBE`       | `#FFD75F` | Existing tribe identity (unchanged)                                |
| Clan             | `CLAN`        | `#D75FFF` | Existing clan identity (unchanged)                                 |
| Family container | `FAMILY`      | `#00AFFF` | Same cyan as family `Name:` and the `FAMILY MEMBERS` roster accent |
| Agent shell      | `AGENT SHELL` | `#FFD700` | Same gold as the agent `Name:` value and list-row name annotation  |

Grouping containers stay one word (`TRIBE`, `CLAN`, `FAMILY`). The leaf run uses the
glossary term `AGENT SHELL` rather than `SHELL` (ambiguous with proc shells and terminal
shells) or `AGENT` (already used as a roster kind and a list display type).

Do not restyle tribe gold or agent gold in this tale. They are already 5 hex apart
(`#FFD75F` vs `#FFD700`). The documents are otherwise distinct (tribe composition and
`@name` versus shell `Project` / `Workspace` / timestamps), so the proximity is an
accepted existing constraint, not a reason to invent a third gold.

### Classification

Decide the heading from predicates the Agents tab already trusts. Do not invent a
parallel kind enum.

In `build_header_text` (after the clan-container early return, before
`append_agent_metadata_fields`):

1. If `agent.is_family_container_row` → `FAMILY`
2. Else if `agent.is_agent_entry` → `AGENT SHELL`
3. Else → no kind heading

`is_agent_entry` is the existing "this row is an LLM/provider run" test: it is true for
`RUNNING` rows and for workflows that appear as agents (including `step_type == "agent"`
children), and it is already false for clan containers and monitors. That gives the
right exclusions without a second classifier:

- Family **member** (`--plan`, `--code`, …) → `AGENT SHELL`. The sibling
  `FAMILY MEMBERS · <family>` roster still names the family underneath.
- Standalone one-shell sase agent → `AGENT SHELL`. A family does not exist until there
  are two shells; the unlabeled `Name:` today is this case.
- Family root that is not yet a container (synthetic planner only) → `AGENT SHELL`.
- Monitor member → no kind heading. It is a proc shell. The document already has a
  `MONITOR` section; do not mislabel it `AGENT SHELL`.
- Workflow step children (`bash` / `python` / `parallel`) → no kind heading. They
  already get a `Step:` field and a distinct body renderer.
- Clan / tribe → unchanged (`CLAN` / `TRIBE` from their own builders).

Cheap/header-only paints must include the heading. `update_header_only` and
`build_header_text(..., cheap=True)` share this builder; a heading that appears only on
the debounced full path would flicker in after j/k, which is worse than no heading. The
append is one `Text.append` with no I/O.

### Not a section title

`Ctrl+J` / `Ctrl+K` cycle **rendered section titles**. Clan and tribe kind labels are
ordinary header text in the top waypoint, not titles. `FAMILY` and `AGENT SHELL` must be
the same:

- Call a dedicated kind-header helper (or the same raw
  `text.append(label + "\n", style=...)` clan already uses)
- Do **not** call `append_section_heading` — that attaches `SECTION_MARKER_META_KEY` and
  would steal the first jump from `FAMILY MEMBERS` / `SASE CONTEXT` / `AGENT XPROMPT`

The first navigable title on a family container remains `FAMILY MEMBERS` (or the next
rendered title if that roster is absent). The kind line lives in the existing "true top
of the document" waypoint with `Name:`, `Project:`, timestamps, and `Fold:`.

### Shared helper

Add one helper, for example `append_kind_header(text, label, color)`, that writes
`{label}\n` with `bold {color} underline` and does not mark a section.

Use it for `FAMILY` and `AGENT SHELL`. Switch the existing `CLAN` and `TRIBE` appends
onto the same helper if — and only if — clan/tribe tests stay output-identical
(`bold #D75FFF underline` / `bold #FFD75F underline`, label then `Name:`, no extra blank
line). Do not "improve" clan/tribe spacing, copy, or colors while doing that.

Reuse existing color constants. Do not mint a third copy of family cyan. Prefer:

- Family: `_agent_display_family.FAMILY_IDENTITY_COLOR` (already `#00AFFF`) or the
  list-styling `_FAMILY_IDENTITY_COLOR` that clan already imports
- Agent shell: `_AGENT_NAME_ANNOTATION_STYLE` (`#FFD700`) from `_agent_list_styling.py`,
  the constant whose comment says it must stay in lockstep with the metadata `Name:`
  value

### Layout that must not move

Family containers already render `Fold: N/2` **after** the identity fields and
**before** `FAMILY MEMBERS`. The new `FAMILY` line goes above `Name:` only. Do not slide
`Fold:` up next to the kind label just because clan/tribe put `Fold:` in their
kind-header block — those documents have no `Project` / `Model` / `Timestamps` stack.

Member and standalone shell documents keep their current three-level agent fold
behavior, including the absence of a `Fold: N/M` line on member panels.

## Implementation sketch

Presentation-only Textual/Rich work. Stay in this repo. No Rust core change, no feature
flag (this is completing existing chrome, not a beta or a sunset), no keymap or
`default_config.yml` change.

Primary seam: `src/sase/ace/tui/widgets/prompt_panel/_agent_display_header.py` in
`build_header_text`. Clan still returns early into `build_clan_detail_text`. Tribe still
uses `append_tribe_header`. Family containers still take `_update_family_display` after
the shared header is built — that path automatically picks up the new first line.

Keep the helper next to `append_section_heading` in `prompt_panel/_helpers.py` (or a
tiny sibling module) so the "chrome, not a section" rule sits beside the function people
would otherwise reuse by mistake.

## Tests

Lock the classification and the style, then repair first-line assertions the new line
will shift.

New coverage (prefer extending
`tests/ace/tui/widgets/test_agent_display_family_roster.py` /
`test_agent_display_family_member_roster.py` and adding a focused header test module
rather than a one-off dump):

- Family container `build_header_text` starts with `FAMILY\nName:`, style at `FAMILY` is
  `bold #00AFFF underline`, and that `FAMILY` span ends before `FAMILY MEMBERS`
- Family member starts with `AGENT SHELL\nName:`, gold underline, and still shows
  `FAMILY MEMBERS · <family>` rather than a `FAMILY` kind line
- Standalone `is_agent_entry` agent starts with `AGENT SHELL\n`
- Cheap/header-only path (`cheap=True` and `update_header_only`) includes the heading on
  the first paint
- Monitor member does **not** start with `AGENT SHELL` or `FAMILY`
- `bash` / `python` / `parallel` workflow step does **not** start with `AGENT SHELL`
- Clan still starts with `CLAN`; tribe still starts with `TRIBE`
- `Ctrl+J` / `Ctrl+K` section navigation does not treat `FAMILY` or `AGENT SHELL` as
  titles (first jump is still the first real section)

Repair existing first-line contracts that assume `Name:` is line 0, including at least:

- `tests/ace/tui/widgets/test_agent_display_header_only.py`
  (`startswith("Name: unassigned\n")`)
- `tests/ace/tui/widgets/test_agent_display_name_model_metadata.py`
  (`assert_metadata_prefix(..., "Name: ...")`)
- `tests/ace/tui/widgets/test_agent_display_bead_metadata.py`
  (`splitlines()[0] == "Name: ..."` and prefix tuples)
- `tests/ace/tui/widgets/test_agent_display_plan_section.py`
  (`splitlines()[1].startswith("Patch:")` — that index becomes 2)

Substring pitfall: do not assert SVG/plain `FAMILY` as proof of the kind heading.
`FAMILY MEMBERS` already contains that token. Assert `startswith("FAMILY\n")` or a span
whose end is before `FAMILY MEMBERS`. `AGENT SHELL` is unique and is safe as a literal.

## Docs

Update the user-facing metadata docs so they describe all four kind labels together:

- `docs/ace.md` — **Clan and Family Detail Panels** currently says a family root shows
  "normal agent metadata plus a `FAMILY MEMBERS` roster". Say it opens with underlined
  `FAMILY` (cyan, matching the name) like `CLAN` / `TRIBE`. Add that a selected agent
  shell — standalone or family member — opens with underlined `AGENT SHELL` (gold,
  matching the name).
- `docs/ace.md` — **Agents Tab Metadata Panel** list that already documents
  `CLAN / MEMBERS`. Add sibling bullets for `FAMILY` and `AGENT SHELL`, including that
  they are header chrome, not `Ctrl+J` titles.
- `docs/agent_families.md` — **Family detail folding** / **Family member detail
  folding**. The member section already calls a member row "an agent shell node"; say
  the panel names that with `AGENT SHELL`, and the container with `FAMILY`.

Do not edit `sase/memory/*.md` or generated instruction shims.

## Visual goldens

Any snapshot whose right-hand metadata document currently begins at `Name:` for a family
container or an agent shell will shift down one row and pick up the new underline. That
includes the family-panel suite
(`tests/ace/tui/visual/test_ace_png_snapshots_agents_family_panel.py`) and other
Agents-tab captures that show a selected shell (selected-row, metadata zoom,
auto-approve metadata, neighbors, and similar).

Update only goldens this heading actually changes. Use the targeted recipe:

```bash
just test-visual -- --sase-update-visual-snapshots tests/ace/tui/visual/<affected_module>.py
```

Inspect `.pytest_cache/sase-visual/` diffs before keeping a golden. `just check` does
not run the PNG suite; do not treat a green scoped run as proof the pixels are current.
Do not pass `--sase-update-visual-snapshots` to `just check`.

If a family-panel visual test already calls
`assert_page_svg_contains(page, "FAMILY MEMBERS")`, add a distinct `AGENT SHELL`
assertion on the member-roster snapshot, and rely on unit tests (not a bare `FAMILY` SVG
needle) for the container kind line.

## Non-goals

- No `MONITOR`, `PROC SHELL`, or `STEP` kind headers. Monitor and step documents already
  identify themselves (`MONITOR` section, `Step:` field). Adding them would expand the
  color/label design and the golden surface without fixing the gap this tale is for.
- No change to `FAMILY MEMBERS` / `CLAN MEMBERS` / `TRIBE MEMBERS` roster titles,
  numbering, or fold glyphs.
- No change to list-row glyphs, grouping banners, or tribe panel chrome.
- No feature flag, config field, or keymap.
- No Rust core / `sase_core_rs` work — this is metadata-panel presentation.

## Verification

- `just install` if this workspace venv is stale, then `just check`
- `just test-visual` (or the targeted modules above) after goldens are updated
- If `just check`'s scoped lane escalates or the change hits the broadening set, run
  `just check-full` through `/sase_monitor` rather than inline )
