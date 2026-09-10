---
tier: tale
title: Show xprompt properties in the ACE preview reader
goal:
  Pressing K on an xprompt shows its declared properties - inputs, defaults, tags,
  skill/snippet/memory flags, local xprompts, and steps - in a compact always-on band
  above the source, with a full properties view on p, sharing one projection with `sase
  xprompt show`.
size: medium
proposed_by: bbugyi200.athena.yc
create_time: 2026-09-09 20:00:36
status: wip
---

# Show XPrompt Properties in the Preview Reader

## Problem

Pressing `K` on an xprompt reference in the prompt input opens the preview reader
(`PreviewPanelModal`). Today that reader shows the xprompt's **body and nothing else**.

Concrete example: `#bd/review_tasks` is defined in a user config `sase.yml` as

```yaml
bd/review_tasks:
  input:
    project: { type: line, default: sase }
  content: |
    Can you review all of the current task sase beads that are opened for
    the "{{ project }}" project? ...
```

`K` on that reference renders the body with a `{{ project }}` placeholder visible and
gives the reader no way to learn that `project` is a declared input, that it is a
`line`, or that it defaults to `sase`. The reader is the one surface where a user asks
"what is this xprompt and how do I call it?", and it currently answers only half the
question.

The information already exists in two places:

- `sase xprompt show <name>` renders a complete `PROPERTIES` / `INPUTS` /
  `LOCAL XPROMPTS` / `WORKFLOW STEPS` view from `XPromptShowRecord`
  (`src/sase/xprompt/cli_show_model.py`, rendered by
  `src/sase/xprompt/cli_show_render.py`).
- The prompt input's own argument-hint panel renders inputs inline while you type
  (`src/sase/ace/tui/widgets/_xprompt_arg_assist_inputs.py`).

The preview reader is the third surface and it is the only one that shows nothing. This
plan closes that gap **without inventing a fourth description of what an xprompt's
properties are**.

### Why the existing fallback does not already cover this

`src/sase/ace/tui/widgets/_prompt_preview_target.py` has `_xprompt_fallback_preview()`,
which emits a Markdown `## Inputs` section. It does not help here for two reasons:

1. It is gated behind
   `if not xprompt.description and not has_input_descriptions: return xprompt.content` —
   an xprompt with inputs but no description (the common case, including
   `#bd/review_tasks`) skips the section entirely.
2. It only runs when the definition's source file is **not** readable. For every
   `.md`-file xprompt the reader shows raw file text instead, so properties are visible
   only as hand-parsed YAML frontmatter.

Both paths are replaced by the explicit, always-on properties surface below.

## Design

### Shape of the feature

Two new surfaces, one new key:

1. **The properties band** — a compact, always-on, fixed block between the reader's
   title and its scrolling source pane. Answers "what does this take?" with zero
   keystrokes. Bounded height; never scrolls; never steals focus.
2. **The properties view** — a third full-pane view mode alongside the existing `source`
   and `rendered` modes, reached with `p`. Shows the complete property record,
   scrollable, for xprompts too rich for the band.

Non-xprompt previews (files, beads, plans, chats, commits, artifact files) are
completely unaffected: they carry no properties, so no band renders, the footer is
unchanged, and `p` only emits a warning toast.

### Why a fixed band rather than a section inside the scroll pane

Putting the band inside `#preview-scroll` would be simpler to compose but would break
search. `PreviewPanelModal._jump_to_current_match()` maps a matched **source line
number** to a scroll offset via `row_offsets[line_number - 1]`, where the offsets are
computed by `build_search_result()` over the source text alone
(`src/sase/ace/tui/modals/preview_search.py`). Any renderable prepended inside the same
scroll container silently shifts every match jump by the band's rendered height, and
that height varies per xprompt. Keeping the band **outside** the scroll container leaves
all existing search, scroll, and `g`/`G` math untouched — this is the single most
important reliability property of the design and must not be traded away.

### Band layout

```
 # XPROMPT SOURCE  #bd/review_tasks
 /home/bryan/.config/sase/sase.yml
 ────────────────────────────────────────────────────────────────
  Review open task beads for a project and triage them down to seven.

  project      line    default: sase    the project whose beads to review
  dry_run      bool    default: false   report only; make no changes

  config · 2 inputs · tags: bd
 ────────────────────────────────────────────────────────────────
 ┌──────────────────────────────────────────────────────────────┐
 │  1  Can you review all of the current task sase beads ...    │
```

Three stacked parts, each omitted when it has no content:

1. **Description** — the xprompt's `description`, folded, capped at 2 rendered lines.
2. **Inputs table** — a `rich.table.Table.grid` with four columns, matching the column
   shape of `cli_show_render._inputs()` so the reader and `sase xprompt show` read
   identically:
   - **name** — styled with the `xprompt.invocation_arg` role from `highlight_theme()`
     (`src/sase/xprompt/highlight_theme.py`), the same accent the prompt input uses for
     arguments. A repeatable input gets a trailing `…`.
   - **type** — the `InputType` value, `xprompt.directive` role.
   - **marker** — exactly one of `required` (warning-styled, so a required input is the
     one thing that visually pops), `default: <value>`, or `optional`. For an `enum`
     input the marker column instead reads `one of: a, b, c` (elided past ~3 choices),
     since the choice set is the input's real contract.
   - **description** — dim, folded.
3. **Chips row** — a dim `·`-separated summary of everything else the xprompt declares.
   Chips, in order, each emitted only when it applies: source bucket · `N inputs` ·
   `tags: a, b` · `skill` or `skill: claude, codex` · `snippet` or `snippet: <trigger>`
   · `memory · long` · `N local xprompts` · `N steps` · `swarm · N segments` ·
   `project: <name>`.

The chips row is what makes the band honest about the word "any properties": a user
never has to wonder whether the band is hiding a declared field, because every
frontmatter field the schema supports maps to either a table row or a chip.

### Bounded height and overflow

The band renders at most `_BAND_MAX_INPUT_ROWS = 6` input rows. When there are more, it
renders the first 5 and a final dim row:

```
  … +4 more  ·  p for all properties
```

Truncation is deterministic, always discloses the omitted count, and always names the
escape hatch. Description folding and chip-row wrapping are handled by Rich; only the
input table is explicitly capped.

### The `p` properties view

`p` switches `#preview-scroll` to a `properties` view mode. In that mode:

- The band is hidden (no duplication).
- The scroll pane renders the complete record: description, **all** inputs, local
  xprompts (name, input signature, line count, description), workflow steps (index,
  name, type, label), tags, skill/snippet/memory/log-skill-use, and provenance (source
  bucket, definition path). This is the TUI twin of `sase xprompt show`'s non-body
  sections.
- The title's mode chip reads `PROPERTIES` instead of `SOURCE` / `RENDERED`.
- `p` again returns to whichever mode was active before.
- Opening search with `/` forces the view back to `source` first, exactly as it already
  does for `rendered` mode in `action_open_search()`.
- On a payload with no properties, `p` emits
  `notify("This preview has no xprompt properties", severity="warning")` and changes
  nothing — the same shape as the existing non-Markdown `R` guard.

### Data flow, and the anti-drift rule

The properties are derived from the **object the reader already resolved**, not from a
second catalog lookup.

`_resolve_xprompt_preview()` in `src/sase/ace/tui/widgets/_prompt_preview_target.py`
already calls `get_xprompt_or_workflow(lookup, project=project)` and holds the resulting
`XPrompt` or `Workflow`. Building properties from that same object guarantees the band
can never describe a different definition than the body shown beside it. Calling
`resolve_show_record()` a second time was considered and rejected: it re-resolves the
catalog with different xprompt-vs-workflow shadowing precedence (`resolve_show_record`
prefers the workflow; `get_xprompt_or_workflow` prefers the xprompt), so under a
shadowed name the band and the body could disagree.

To keep the reader and `sase xprompt show` from drifting anyway, the **projection
functions become shared** rather than duplicated. `_show_inputs()`,
`_show_local_xprompts()`, and `_show_steps()` currently live as private helpers in
`src/sase/xprompt/cli_show_resolve.py`; they move into a new shared module that both
`resolve_show_record()` and the preview path import. There is exactly one definition of
"what an xprompt's inputs look like when displayed", with two consumers.

### Placement note: Rust core boundary

The repo's core-backend rule sends shared domain behavior to `../sase-core`. This change
deliberately keeps the projection in `src/sase/xprompt/`, because that is where the
existing, versioned display contract (`XPromptShowRecord`, `SHOW_SCHEMA_VERSION = 2`,
`ShowInput`) already lives, and because the models it projects from (`XPrompt`,
`InputArg`, `Workflow`) are Python dataclasses. This change **reduces** duplication
within the existing boundary rather than adding a new frontend-specific copy. Migrating
the whole show/display contract into `sase-core` is a real question but a separate,
larger one; do not attempt it here.

### Performance

Per `sase/memory/tui_perf.md`:

- `xprompt_properties()` is pure over an already-loaded model — no disk reads, no
  catalog reload — and is called inside the existing `asyncio.to_thread(...)` in
  `PromptPreviewMixin._resolve_preview_async()`, so no work is added to the UI thread.
- The band and properties-view renderables are built once per payload and cached on the
  modal instance. Render paths do not stat, glob, or re-project per keypress.
- No new timers, workers, or pump callbacks.

## Implementation

### 1. Shared properties projection — `src/sase/xprompt/properties.py` (new)

Create the module with a module docstring explaining that it is the single projection of
an xprompt/workflow definition into display-ready properties, shared by
`sase xprompt show` and the ACE preview reader.

Move these three helpers out of `src/sase/xprompt/cli_show_resolve.py` and make them
public here, preserving their current behavior byte-for-byte:

- `_show_inputs` → `show_inputs(inputs: list[InputArg]) -> list[ShowInput]`
- `_show_local_xprompts` → `show_local_xprompts(...) -> list[ShowLocalXPrompt]`
- `_show_steps` → `show_steps(workflow: Workflow) -> list[ShowStep]`

Update `cli_show_resolve.py` to import and call them; delete the private copies. Keep
`ShowInput` / `ShowLocalXPrompt` / `ShowStep` in `cli_show_model.py` (they are part of
the versioned JSON schema) and import them here.

Add the preview-facing aggregate:

```python
@dataclass(frozen=True, slots=True)
class XPromptProperties:
    """Display-ready properties of one resolved xprompt or workflow."""

    reference: str
    kind: str
    description: str | None
    input_signature: str | None
    inputs: list[ShowInput]
    local_xprompts: list[ShowLocalXPrompt]
    steps: list[ShowStep]
    tags: list[str]
    skill: bool | list[str] | None
    skill_name: str | None
    snippet: str | bool | None
    log_skill_use: bool | None
    memory_type: MemoryType | None
    segment_count: int
    project: str | None
    source_bucket: str | None
    definition_path: str | None

    @property
    def is_empty(self) -> bool:
        """Whether there is nothing worth rendering for this definition."""
```

and the projection entry point:

```python
def xprompt_properties(
    obj: XPrompt | Workflow,
    *,
    reference: str,
    kind: str,
    project: str | None = None,
    source_bucket: str | None = None,
    definition_path: str | None = None,
) -> XPromptProperties:
```

Behavior requirements:

- Accept either an `XPrompt` or a `Workflow`; normalize a `Workflow` through the same
  shape `resolve_show_record()` uses (`xprompt_to_workflow` is already used in the
  reverse direction; read `_workflow_descriptor()` in `cli_show_resolve.py` and mirror
  its field mapping).
- Filter out `is_step_input` inputs — they are never user-facing.
- Reuse `format_inputs()` from `sase/xprompt/_catalog_format.py` for `input_signature`,
  so the signature matches the completion and CLI surfaces.
- Compute `segment_count` with `xprompt_segment_count()` from
  `sase/xprompt/segment_separators.py`; `segment_count > 1` means swarm.
- `is_empty` is `True` only when there is no description, no inputs, no local xprompts,
  no steps, no tags, and no skill/snippet/memory/swarm signal — i.e. a bare-body
  xprompt. The band is suppressed entirely in that case rather than rendering an empty
  frame.
- Pure: no filesystem access, no catalog loads, no logging.

Export everything through `__all__`, and follow `sase/memory/symvision.md` conventions
so the newly-public `show_*` helpers do not trip private-misuse or unused-symbol lints.

### 2. Carry properties on the payload — `_prompt_preview_target.py`

- Import `XPromptProperties` / `xprompt_properties`.
- Add `properties: XPromptProperties | None = None` as the last field of the frozen
  `PreviewPayload` dataclass, so every existing positional/keyword construction site
  keeps working unchanged.
- In `_resolve_xprompt_preview()`, after `obj` is resolved and `kind_label` /
  `source_path` are computed, build the properties from that same `obj` and pass them
  into the returned `PreviewPayload`. Derive `source_bucket` from the existing
  `classify_source()` helper used by `_source_display_path()`; if that raises, fall back
  to `None` rather than failing the preview.
- `_resolve_file_preview()` is untouched — file payloads keep `properties=None`.
- Wrap the projection in a narrow `try/except Exception` that degrades to
  `properties=None`. A malformed definition must never turn a working body preview into
  an error toast.
- Leave `_xprompt_fallback_preview()` and `_workflow_fallback_preview()` alone; they
  still supply body content. Their `## Inputs` sections now duplicate the band for the
  narrow case where they fire, which is acceptable and strictly better than removing
  content the user may already rely on.

### 3. Pure renderers — `src/sase/ace/tui/modals/preview_properties_render.py` (new)

Two pure functions, no Textual imports, unit-testable without an app:

```python
def build_properties_band(
    properties: XPromptProperties,
    *,
    max_input_rows: int = 6,
) -> RenderableType | None: ...


def build_properties_view(properties: XPromptProperties) -> RenderableType: ...
```

- `build_properties_band` returns `None` when `properties.is_empty`.
- Both pull colors from `highlight_theme()` (`src/sase/xprompt/highlight_theme.py`)
  rather than hardcoding hex values, matching `cli_show_render.py` and
  `glossary_preview_render.py`. Only use dim/bold/italic as literal styles.
- Reuse `cli_show_render._single_line_default()`'s behavior for multi-line defaults
  (first line plus ` …`); if that helper is useful verbatim, promote it into
  `properties.py` as a public helper rather than copying it.
- `build_properties_view` groups sections with a `rich.rule.Rule` and section titles in
  the same order as `render_show()`: properties chips, inputs, local xprompts, workflow
  steps. It renders **all** rows — no truncation.

### 4. Modal wiring — `src/sase/ace/tui/modals/preview_panel_modal.py`

- Widen `_ViewMode` to `Literal["source", "rendered", "properties"]`.
- Add instance state: `_properties_band` (built once in `__init__` or lazily on first
  compose), `_previous_view_mode: _ViewMode` for `p`-toggle restore.
- `compose()`: yield `Static(band, id="preview-properties")` between `#preview-title`
  and `#preview-scroll`, with `display = band is not None`. Inside `#preview-scroll`,
  yield a third `Static(id="preview-properties-view")` with `display = False`, beside
  the existing `#preview-content` and `#preview-rendered`.
- `_build_title()`: when `_view_mode == "properties"`, append `" PROPERTIES"` with the
  existing mode-chip style, regardless of `_is_markdown_payload()`.
- `_build_footer()`: append `"p properties"` (or `"p source"` when already in the
  properties view) only when the payload has non-empty properties. Keep `esc close`
  last. The footer is a single ellipsized line, so add nothing when properties are
  absent.
- `_refresh_preview_widgets()`: extend the three-way display toggle to cover
  `#preview-properties-view`, and hide the `#preview-properties` band whenever
  `_view_mode == "properties"`.
- New binding `("p", "toggle_properties", "Properties view")` and
  `action_toggle_properties()`, guarded exactly like `action_toggle_rendered()`'s
  non-Markdown case. Verify `p` is not already claimed by `CopyModeForwardingMixin`
  (`modals/base.py`) or `SourceFileActionsMixin` (`modals/_source_file_actions.py`) and,
  if it is, fall back to `P` and adjust every footer/help/doc string accordingly.
- `action_open_search()`: treat `properties` like `rendered` — force back to `source`
  before showing the search input.
- `action_copy_contents()` keeps copying the body, not the properties, in every mode.

### 5. Styling — `src/sase/ace/tui/styles.tcss`

Add, next to the existing `PreviewPanelModal` rules (~line 4808):

```
PreviewPanelModal #preview-properties {
    height: auto;
    max-height: 12;
    margin-bottom: 1;
    padding: 0 1;
    border-top: solid $secondary;
    border-bottom: solid $secondary;
}

PreviewPanelModal #preview-properties-view {
    width: 100%;
    height: auto;
}
```

The `border-top`/`border-bottom` pairing deliberately echoes the existing
`#preview-footer` `border-top: solid $secondary`, so the band reads as part of the
reader's chrome rather than as a floating box. Confirm the modal container's
`height: 85%` still leaves a usable source pane with a full-height band at an 80×24
terminal; if it does not, lower `max-height` and the input-row cap together.

### 6. Documentation

- `docs/ace.md` § **Preview Reader** (~line 401): add a `p` row to the key table and a
  short paragraph describing the band and the properties view.
- `src/sase/ace/tui/modals/help_modal/patches_artifact_bindings.py` — add
  `("p", "XPrompt properties view")` to the `"Preview Reader"` section. Per
  `src/sase/ace/CLAUDE.md`, keybinding descriptions are capped at 32 characters and the
  57-character box width must hold.
- `src/sase/ace/tui/modals/help_modal/binding_common.py` — the existing
  `("K", "Preview xprompt/skill/file/word")` entry stays as-is.
- Update the `PreviewPanelModal` class docstring and the `_prompt_preview_target.py`
  module docstring to mention properties.
- Do **not** touch `sase/memory/*.md`, `AGENTS.md`, or any generated provider
  instruction shim; no memory update was authorized for this work.

## Testing

### New: `tests/xprompt/test_xprompt_properties.py`

Projection unit tests (place the file to match the repo's existing xprompt test layout):

- Required input (no default) → `required=True`, `default_display is None`.
- Input with a default → `required=False`, `default_display` set; an explicit `null`
  default renders distinctly from `UNSET`.
- Multi-line default elides to first line plus `…`.
- `repeatable=True` survives the projection.
- `enum` input carries its choices through to the rendered marker.
- `is_step_input` inputs are filtered out.
- Tags, `skill` as `True` and as a provider list, `snippet` as bool and as a string
  trigger, `memory_type`, `log_skill_use`, local xprompts, and workflow steps all
  project.
- `segment_count > 1` for a body with top-level `---` separators.
- `is_empty` is `True` for a bare-body xprompt and `False` as soon as any property
  exists.
- **Drift guard:** for one representative `XPrompt` and one `Workflow`, assert
  `xprompt_properties(...).inputs == resolve_show_record(...).inputs` (and the same for
  `local_xprompts` and `steps`), so the shared projection provably keeps the reader and
  `sase xprompt show` in agreement.

### New: `tests/ace/tui/modals/test_preview_properties_render.py`

Pure renderer tests against `Text.plain` / console capture:

- Band returns `None` for empty properties.
- Band contains the input name, type, and `default: sase` for the `#bd/review_tasks`
  shape.
- Required inputs render the `required` marker; optional ones do not.
- Overflow: 9 inputs with `max_input_rows=6` renders 5 rows plus
  `… +4 more  ·  p for all properties`.
- Chips row includes tags / skill / steps / swarm chips only when they apply.
- `build_properties_view` renders every input, with no truncation marker.

### Extend: `tests/ace/tui/modals/test_preview_panel_modal.py`

- `#preview-properties` is displayed for an xprompt payload with properties and hidden
  for the existing file payload.
- `p` switches to the properties view: `#preview-properties-view` displayed,
  `#preview-content` hidden, band hidden, title chip reads `PROPERTIES`.
- `p` again restores the previous mode, including when the previous mode was `rendered`.
- `p` on a properties-free payload emits a warning and leaves the view mode unchanged.
- Footer contains `p properties` only when properties exist.
- `/` while in the properties view returns to `source` and opens the search input.
- Existing search/match tests still pass unchanged — this is the regression guard for
  the "band outside the scroll" decision.

### Extend: `tests/ace/tui/widgets/test_prompt_preview_target.py`

- An xprompt token with a declared input yields a payload whose `properties.inputs`
  names that input.
- A file token yields `properties is None`.
- A definition that raises during projection still yields a usable payload with
  `properties is None` (patch the projection to raise).

### Extend: `tests/ace/tui/visual/test_ace_png_snapshots_preview_panel.py`

- A band snapshot: xprompt payload with a description, two inputs (one required, one
  defaulted), and tags.
- A properties-view snapshot after pressing `p`.

Run `just test-visual` and accept intentional new goldens with
`--sase-update-visual-snapshots`; goldens land in `tests/ace/tui/visual/snapshots/png/`.
On failure inspect `.pytest_cache/sase-visual/`.

### Gates

Run `just install` first (workspaces are ephemeral and dependencies may be stale), then
`just check` during development. Run `just check-full` before landing: this change
touches `src/sase/xprompt/` (imported broadly) and moves symbols between modules, so the
scoped test lane is not a sufficient backstop.

## Acceptance criteria

1. `K` on `#bd/review_tasks` (an xprompt with one defaulted input and no description)
   shows a properties band naming `project`, its `line` type, and `default: sase`.
2. `K` on an `.md`-file xprompt shows the same band above its raw source; the source
   pane content is byte-identical to today's.
3. `K` on a file path renders exactly as it does today — no band, no footer change.
4. `p` opens a complete, scrollable properties view and toggles back; the band is hidden
   while that view is active.
5. Search (`/`, `n`, `N`) jumps to the correct source lines with the band visible — the
   existing search tests pass unmodified.
6. `sase xprompt show` output is unchanged by the helper extraction (its existing tests
   pass without edits).
7. `just check-full` passes, and the new visual goldens are committed.

## Out of scope

- Editing properties from the reader. It stays read-only; the frontmatter panel (`g=`)
  and `o` (open in `$EDITOR`) remain the editing paths.
- Previewing prompt-local `#_helper` xprompts, which `get_xprompt_or_workflow()` does
  not resolve today.
- Migrating the show/display contract into `sase-core`.
- Changing the `R` rendered-Markdown behavior or the fallback preview builders.
