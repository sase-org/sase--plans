---
status: done
tier: epic
title: Artifacts sub-tab descriptions
goal: "Every Artifacts sub-tab — the five built-ins, the Plan pane, and every pane a
  sidecar ref creates — carries a resolved, never-empty description that the TUI renders
  as an accent-anchored pane brief under the sub-tab strip and as a hover tooltip on the
  strip itself, with the copy configurable per pane from a sidecar's ref spec or the
  user's sase.yml.

  "
phases:
  - id: resolve
    title: Pane description resolution layer
    depends_on: []
    size: medium
    description: "resolve: add a total, auditable description resolution ladder (user
      config → provider ref.pane → built-in copy → generated fallback), author the
      built-in copy, carry summary/body/source on the compiled pane contract, fix the
      descriptor cache token, and expose it all through sase artifact pane show and the
      config schema.

      "
  - id: brief
    title: The pane brief
    depends_on:
      - resolve
    size: medium
    description: "brief: render the resolved description as a host-owned accent-gutter
      brief under the Artifacts sub-tab strip, with an off/summary/full mode cycled by a
      new key, a click, or the command palette and seeded from config.

      "
  - id: hover
    title: Sub-tab hover tooltips
    depends_on:
      - brief
    size: small
    description: "hover: teach the shared PanelTabStrip to carry a per-tab description
      and show it as a hover tooltip using the same cell-accurate hit test its click
      handler uses, then feed it the Artifacts descriptors.

      "
  - id: goldens
    title: Visual goldens and end-to-end verification
    depends_on:
      - hover
    size: small
    description:
      "goldens: add new PNG goldens for the brief's three modes and the unconfigured
      provider hint, rebaseline every Artifacts golden the new row shifts, and run the
      full verification lane."
proposed_by: bbugyi200.athena.0e2
bead_id: sase-u6
create_time: 2026-09-09 19:50:00
---

- **PROMPT:**
  [prompts/202608/artifacts_subtab_descriptions.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/artifacts_subtab_descriptions.md)
- **BEAD:**
  [sase-u6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-u6/README.md)

# Plan: Artifacts sub-tab descriptions

## Problem

The Artifacts tab hosts a variable number of sub-tabs: five built-ins (Agent, Stitch,
Patch, Bead, File), the Plan pane, and one pane per document-provider `ref_kind`
discovered from any enabled project's sidecar repos — including sidecars the user wrote
themselves. A pane is currently identified only by an icon, a label, an accent colour,
and a digit shortcut. Nothing anywhere in the TUI says what a pane is _for_.

The plumbing is half-built and unused:

- `ArtifactsPaneContract.description` (`src/sase/ace/tui/_artifact_tab_model.py`) exists
  and is always `""` for built-ins.
- `PanePresentation.description` is compiled from a provider's `ref.pane.description` by
  `src/sase/ace/tui/_artifact_tab_presentation.py` and documented in
  `docs/artifacts_pane_contract.md`, but no renderer ever reads it.
- `ArtifactsTabDescriptor.description` is copied from the contract by `attach_contract`
  and then dropped on the floor.

This epic turns that dead field into a real, configurable, beautifully rendered feature.

## Design

### One resolved description per pane, always non-empty

A description has two parts, matching the vocabulary the AXE description banner already
uses (`description_summary` / `description_body`):

- **summary** — one line, ≤ 240 characters. What this pane holds, in a sentence.
- **body** — optional detail, ≤ 600 characters. What the rows are and what you can do
  with them.

Both are resolved by the same four-rung ladder, **independently per field**, so a user
who overrides only the summary keeps the good built-in body:

| Rung | Source                                               | `source` value |
| ---- | ---------------------------------------------------- | -------------- |
| 1    | `ace.artifacts.panes.<pane_id>.description[_body]`   | `config`       |
| 2    | `ref.pane.description` / `ref.pane.description_body` | `provider`     |
| 3    | The host-owned built-in copy table                   | `builtin`      |
| 4    | Generated from the pane label                        | `fallback`     |

The resolver is a **total function**: every configured pane, degraded ones included,
gets a non-empty summary. That is what lets the brief be a fixed-height layout slot that
never appears or disappears as you switch panes — the same invariant the query bar
already holds (`docs/artifacts_pane_visual_grammar.md`).

Rung 4 is deliberately useful rather than apologetic: an unconfigured provider pane
tells its author how to describe it (see "The unconfigured hint" below). That is the
discoverability path for custom user sidecars.

### Where it renders

Three surfaces, in descending prominence:

1. **The pane brief** — a host-owned row directly under the Artifacts sub-tab strip,
   shared by every pane (built-in, provider, and degraded), so uniformity is structural
   rather than per-pane discipline. Three modes: `off` (0 rows), `summary` (1 row),
   `full` (summary + body, capped at 6 rows).
2. **Hover tooltips** on the sub-tab strip — learn about a pane you are _not_ on, at
   zero row cost, even in `off` mode.
3. **`sase artifact pane show <pane_id>`** — the resolved summary, body, and the rung
   each came from, so "why does my pane say that?" is answerable.

### Why a dedicated row rather than the identity header

The per-pane identity/scope header (`shell.build_shell_scope`) already carries the label
chip, the project scope, and a change hint. Appending prose there crowds an 80-column
terminal and buries the description at the wrong altitude. A dedicated row directly
beneath the strip is spatially correct — it explains the thing immediately above it —
and living in `ArtifactsView` rather than in each pane means a new document provider
gets it for free.

The cost is honest and stated: one row of vertical chrome by default, and a rebaseline
of every Artifacts PNG golden (phase `goldens`).

## Phase `resolve`: Pane description resolution layer

### New module: `src/sase/ace/tui/_artifact_tab_descriptions.py`

Widget-free, no Textual imports, sibling to `_artifact_tab_descriptors.py`. It owns:

- `BUILTIN_PANE_DESCRIPTIONS: dict[str, tuple[str, str]]` — the authored copy below,
  keyed by pane id (`agents`, `stitches`, `patches`, `beads`, `files`, `ref:plan`).
- `resolve_pane_description(pane_id, *, label, provider_summary, provider_body) -> PaneDescription`
  — the ladder.
- `sanitize_description(raw, *, max_len)` — collapse all whitespace runs to single
  spaces, strip Unicode `Cc` control characters, trim, and ellipsize past `max_len`.
  Model it on `_sanitize_description` in `src/sase/ace/tui/models/tribe_display.py`; the
  body variant preserves paragraph breaks (a blank line) but collapses runs within a
  paragraph.
- A merged-config read cached on `current_config_token()`, exactly like
  `_tribe_displays_for_token` in `models/tribe_display.py`.

`PaneDescription` is a new frozen slots dataclass in `_artifact_tab_model.py`:

```python
@dataclass(frozen=True, slots=True)
class PaneDescription:
    summary: str = ""
    body: str = ""
    summary_source: str = "fallback"   # config | provider | builtin | fallback
    body_source: str = "fallback"

    def to_payload(self) -> dict[str, Any]: ...
```

### The authored built-in copy

Written against the canonical SASE glossary (`glossary:stitch`, `glossary:patch`,
`glossary:artifact`, `glossary:agent`) and the bead/artifact reference notes. No Rich
markup, no backticks, no em-dash-heavy prose — these render as plain `Text`.

**`agents`** (Agent, `⬡`, `#0062FF`)

- summary:
  `Every SASE agent that has run, with the prompt it was given and the work it left behind.`
- body:
  `Rows are agent runs, families and their shells, scoped to the selected project. Selecting one shows its identity, lifecycle, provenance, and prompt preview, and the relation panel links it to the beads it worked and the stitches it landed. Live agents belong to the Agents tab; this pane is the durable record you can query.`

**`stitches`** (Stitch, `◉`, `#FFD700`)

- summary:
  `Commits landed through SASE, each one a stitch inside the Patch that carries it.`
- body:
  `Rows are VCS commits across your enabled projects, narrowed by the query bar and the project scope. Selecting one shows its message, diff, and repository context, and the relation panel walks its parents, its children, and the Patch it belongs to.`

**`patches`** (Patch, `⎇`, `#00D7AF`)

- summary:
  `SASE's unit of change: one Patch per change, with or without a PR behind it yet.`
- body:
  `A Patch carries a change's name, description, stitches, hooks, review comments, and mentors. Its status runs WIP, Draft, Ready, Mailed, Submitted, and a terminal Patch moves to the project archive. Selecting one shows the full spec beside its commits.`

**`beads`** (Bead, `◈`, `#D787FF`)

- summary:
  `The work SASE tracks: plan and epic beads, the phases beneath them, and standalone task beads.`
- body:
  `Rows come from the current project's bead store, grouped by hierarchy: a plan bead owns its phase beads, and typed task beads capture follow-up work agents discovered along the way. Selecting one shows its status, size, dependencies, append-only notes, and the artifacts it references.`

**`files`** (File, `▤`, `#FFAF5F`)

- summary:
  `Indexed artifact files: the snapshots agents registered, plus automatic captures from their runs.`
- body:
  `This is the artifact index, not every file in your repos. Explicit snapshots an agent registered are immutable and permanent, while automatic captures may be reclaimed as verified VCS locators or pruned by retention. Selecting one shows its metadata, its versions, and the agent that produced it.`

**`ref:plan`** (Plan, provider icon, `#AF87FF`)

- summary:
  `Plan documents from each project's plans sidecar, from proposed through approved and archived.`
- body:
  `Selecting a plan shows its body and its approval state. Approving one launches the epic that implements it, and the relation panel links a plan to the bead that tracks that work.`

### The generated fallback (rung 4)

For any pane with no configured, provider-declared, or built-in copy — that is, a
document-provider pane from a sidecar nobody has described yet:

- summary: `{label} documents contributed by this project's sidecar repos.`
- body: `""`, with `body_source == "fallback"`.

The brief renders an extra **unconfigured hint** line in `full` mode when
`summary_source == "fallback"`, styled `dim italic`, naming both config paths:

```
Describe this pane with ref.pane.description in its sidecar ref config,
or ace.artifacts.panes."<pane_id>".description in sase.yml.
```

This is the only place the brief renders anything developer-facing, and it is exactly
the affordance the "custom user sidecars must be configurable" requirement needs.

### Contract wiring

In `src/sase/ace/tui/_artifact_tab_model.py`:

- `PanePresentation` gains `description_body: str = ""`; include it in `to_payload()`.
- `ArtifactsPaneContract` keeps `description: str` (now the resolved **summary**, never
  empty) and gains `description_body: str = ""`, `description_source: str = "builtin"`,
  `description_body_source: str = "builtin"`. Keeping `description` a `str` avoids
  churning `attach_contract` and the existing `explanation_payload` key.
- `ArtifactsTabDescriptor` gains `description_body: str = ""` beside its existing
  `description`; `attach_contract` in `_artifact_tab_contract.py` copies both so
  envelope fields cannot drift from the contract.

In `src/sase/ace/tui/_artifact_tab_presentation.py`,
`compile_provider_pane_ presentation` accepts `description_body` inside `ref.pane`
(`_PANE_KEYS` gains it), validated by the existing `_optional_text` with `max_len=600`.
A body that is not a string, or is empty after normalization, degrades the pane the same
way every other `ref.pane` violation does.

In `src/sase/ace/tui/_artifact_tab_contract.py`:

- `compile_builtin_contract` and `compile_provider_contract` both call
  `resolve_pane_description(...)` and pass all four values into `_assemble_contract`.
- `_presentation_digest_for` and `_presentation_digest` payloads gain
  `description_body`, `description_source`, and `description_body_source`, so changing
  copy changes the digest.
  `tests/ace/tui/artifacts_contract/test_contract_compiler.py::test_presentation_ digest_is_deterministic_and_sensitive`
  asserts relational properties only, so no frozen digest constants need updating.

### Cache-token fix (correctness, not polish)

`resolve_artifacts_subtabs()` caches on `provider_source_token()`
(`src/sase/ace/tui/_artifact_tab_discovery.py`), which folds in each project's
`project_file` and per-project config stat but **not** the user's merged config. With
descriptions now readable from `ace.artifacts.panes.*` in `~/.config/sase/sase.yml`,
editing that file must invalidate the descriptor cache. Append
`sase.config.core.current_config_token()` to the tuple `provider_source_token()`
returns, keeping the existing `None`-is-uncacheable contract intact.

### Config surface

`src/sase/default_config.yml`, inside the existing `ace.artifacts` block:

```yaml
ace:
  artifacts:
    # off | summary | full. Press D on the Artifacts tab to cycle this for the
    # current session.
    description_mode: "summary"
    # Per-pane description overrides, keyed by Artifacts pane id: agents,
    # stitches, patches, beads, files, or a provider pane such as "ref:plan".
    # Both fields are optional and resolve independently; an omitted field keeps
    # the sidecar-declared or built-in text.
    panes: {}
```

`src/sase/config/sase.schema.json`:

- `ace.artifacts.description_mode`: `enum: ["off", "summary", "full"]`, default
  `"summary"`.
- `ace.artifacts.panes`: object, `propertyNames.pattern`
  `"^[a-z][a-z0-9_-]*(?::[a-z][a-z0-9_-]*)?$"` (accepts `stitches` and `ref:plan`),
  `additionalProperties` an object with `additionalProperties: false` and optional
  `description` (`minLength: 1`, `maxLength: 240`) and `description_body`
  (`minLength: 1`, `maxLength: 600`).
- The `sidecarRef` definition sets `additionalProperties: false` but never listed
  `pane`. Add a `sidecarRefPane` definition covering the whole block already documented
  in `docs/artifacts_pane_contract.md` (`label`, `description`, `description_body`,
  `order`, `row`, `default_sort`, `facets`, `group_by`, `empty_state`) and reference it
  from `sidecarRef.properties.pane`.
- `capabilities`, `relations`, `grouping`, and `xprompt` are in `KNOWN_REF_CONFIG_KEYS`
  (`src/sase/_sidecar_ref_constants.py`) but likewise absent from `sidecarRef`. That is
  a pre-existing gap outside this epic's scope: file it through `/sase_new_task` rather
  than widening this phase. Note that config layers are not schema-validated at runtime
  (`src/sase/config/file_hooks.py:41`), so this is an editor-completion and
  Config-Center fidelity fix, not a functional unblock.

### CLI

`src/sase/artifact_cli/pane.py`:

- Bump `PANE_SHOW_SCHEMA_VERSION` to `3`.
- `ArtifactsPaneContract.explanation_payload()` keeps `"description"` as the summary
  string and adds a `"description_detail"` object:
  `{summary, body, summary_source, body_source}`.
- `_print_text` gains a `Description` section under the header grid: the summary, the
  body when present, and a dim `source: <rung>` line per field.

### Docs

- `docs/artifacts_pane_contract.md`: document `ref.pane.description_body` beside the
  existing `description`, and state that both feed the pane brief.
- `docs/configuration.md`: add `description_mode` and `panes` rows to the
  `ace.artifacts` table.

### Tests

New `tests/ace/tui/test_artifacts_pane_descriptions.py`:

- Each rung wins over the one below it, per field independently (config summary +
  builtin body is a real, asserted combination).
- Sanitization: whitespace runs collapse, `Cc` control characters are stripped,
  over-length text ellipsizes at the cap, a non-string value falls through to the next
  rung rather than raising.
- Totality: every descriptor from `resolve_artifacts_subtabs()` has a non-empty
  `description`, including a degraded provider descriptor.
- A config override keyed `"ref:plan"` (colon in the YAML key) resolves.
- Editing merged config invalidates `resolve_artifacts_subtabs()` — assert through
  `provider_source_token()` changing, not by sleeping.

Extend `tests/ace/tui/artifacts_contract/test_contract_compiler.py` (digest sensitivity
to body and sources), `test_pane_declarations.py` (a provider declaring
`ref.pane.description_body`, and a malformed one degrading), and the conformance harness
in `tests/ace/tui/artifacts_contract/harness.py` with a non-empty-summary check. Extend
the `sase artifact pane show` CLI tests for the new payload and version.

## Phase `brief`: The pane brief

### Mode helpers: `src/sase/ace/tui/artifacts_description.py`

New widget-free module, a deliberate mirror of the existing
`src/sase/ace/tui/artifacts_split.py`:

```python
ArtifactsDescriptionMode = Literal["off", "summary", "full"]
ARTIFACTS_DESCRIPTION_MODE_ORDER = ("off", "summary", "full")
DEFAULT_ARTIFACTS_DESCRIPTION_MODE: ArtifactsDescriptionMode = "summary"
ARTIFACTS_BRIEF_MAX_LINES = 6

def normalize_artifacts_description_mode(value: object) -> ArtifactsDescriptionMode
def cycle_artifacts_description_mode(mode: object, direction: int) -> ...
```

Cycling is forward-only in the UI (one key, three modes); `direction` exists so the
helper matches its split-mode sibling and stays symmetric for tests.

### Renderer: `build_pane_brief` in `widgets/artifacts/shell.py`

Pure, widget-free, contract-in, `Text`-out — obeying the provider-data boundary in
`docs/artifacts_pane_visual_grammar.md`. Signature takes already-computed presentation
values:

```python
def build_pane_brief(
    *,
    icon: str,
    accent: str,
    summary: str,
    body: str,
    mode: ArtifactsDescriptionMode,
    width: int,
    disclosure_key: str | None,
    unconfigured_hint: str | None = None,
    max_lines: int = ARTIFACTS_BRIEF_MAX_LINES,
) -> Text
```

Layout, modelled on `AxeDescriptionBanner._DescriptionBlock` in
`src/sase/ace/tui/widgets/axe_description_banner.py`:

- Every row opens with a two-cell gutter `"▌ "`. The summary row's gutter is
  `bold {accent}`; body rows use `dim {accent}`. The gutter is the pane's colour
  signature and visually ties the brief to the accent-coloured active tab directly above
  it.
- The summary row is `gutter + icon + two spaces + summary`. The label is deliberately
  **not** repeated: the strip above already shows it uppercase in the same accent, and
  the gutter plus icon carry the identity without the echo.
- Styles are theme-safe: summary `"italic"`, body `"dim"`, unconfigured hint
  `"dim italic"`, disclosure hint `f"dim {accent}"`. Only the accent is a hex colour,
  matching how `shell.py` already treats secondary text. No renderer here looks up
  `ARTIFACTS_ACCENTS`.
- Right-aligned disclosure hint on the summary row: `"▸ {key}"` in `summary` mode,
  `"▾ {key}"` in `full` mode, using the resolved display name of
  `cycle_artifacts_description` — never a hardcoded `D`. Dropped entirely when the row
  lacks room for it, or when `body` and `unconfigured_hint` are both empty.
- `summary` mode truncates the summary with `overflow="ellipsis"`; `full` mode wraps it
  and appends a blank gutter row, the wrapped body paragraphs, and the unconfigured hint
  when present.
- The whole block is capped at `max_lines`; an overflow row reads `f"… +{dropped} more"`
  in `dim`.

### Widget: `src/sase/ace/tui/widgets/artifacts/pane_brief.py`

`ArtifactsPaneBrief(Static)` holding the current mode, contract-derived values, and
disclosure key; `set_state(...)` repaints and `render()` returns the cached renderable
(the `AxeDescriptionBanner` pattern, so it renders without an app mount and is
unit-testable). It posts a `Clicked` message on click.

### `ArtifactsView` integration

`src/sase/ace/tui/widgets/artifacts/view.py`:

- `compose()` yields `ArtifactsPaneBrief(id="artifacts-pane-brief")` between
  `#artifacts-header` and `#artifacts-content-switcher`.
- `apply_description_mode(mode)` mirrors the existing `apply_split_mode`: normalize,
  store, set `display` from `mode != "off"`, repaint.
- `switch_to()` and `on_mount()` repaint the brief from the active descriptor, beside
  the existing `_refresh_split_badge()` call.
- `set_keymap_registry()` forwards the registry so the disclosure hint tracks a remapped
  key.
- `@on(ArtifactsPaneBrief.Clicked)` cycles the app's mode, exactly as
  `_on_split_badge_clicked` already does for the split badge.

`src/sase/ace/tui/styles.tcss`, beside the existing `#artifacts-header` rules:

```
#artifacts-pane-brief {
    width: 100%;
    height: auto;
    max-height: 6;
    padding: 0 1;
    background: $surface;
}
```

### App state and the new binding

- `AceApp.artifacts_description_mode: reactive[str]` in `src/sase/ace/tui/app.py`,
  seeded in `src/sase/ace/tui/actions/_state_init_late.py` from
  `ace.artifacts.description_mode` — the same shape the adjacent
  `axe_description_expanded` and `relations_expanded` reads already use — with a watcher
  that calls `ArtifactsView.apply_description_mode`.
- `action_cycle_artifacts_description` in `src/sase/ace/tui/actions/artifacts.py`, and
  `"cycle_artifacts_description"` added to `NON_PRS_ARTIFACT_ACTIONS` there.

The new action must be wired through every place `cycle_artifacts_split` appears:

| File                                                     | Change                                                       |
| -------------------------------------------------------- | ------------------------------------------------------------ |
| `default_config.yml`                                     | `cycle_artifacts_description: "D"`                           |
| `config/sase.schema.json`                                | keymap property + description                                |
| `ace/tui/keymaps/app_keymaps.py`                         | dataclass field                                              |
| `ace/tui/keymaps/metadata.py`                            | `("cycle_artifacts_description", "Pane Description", False)` |
| `ace/tui/bindings.py`                                    | `Binding("D", ..., show=False)`                              |
| `ace/tui/_app_action_availability.py`                    | gate to `current_tab == ARTIFACTS_TAB`                       |
| `ace/tui/commands/_app_metadata.py`                      | palette entry, `Display` category                            |
| `ace/tui/commands/_availability_artifacts.py`            | add to the artifacts availability set                        |
| `ace/tui/modals/help_modal/patches_artifact_bindings.py` | help row                                                     |

**Key-collision handling.** `D` is already `toggle_attempt_view` (Agents tab). Its
action body early-returns off the Agents tab, and the Artifacts branch of
`_app_action_availability.py` already disables it on every non-Patch Artifacts pane —
but not on the Patch pane. Add an explicit `toggle_attempt_view` →
`current_tab == "agents"` gate there, mirroring the existing `toggle_axe_description`
gate two lines above, and add the pair to the shared-key table in
`docs/configuration.md` (which already documents `.` and `a` sharing keys across tabs
this way).

### Docs

- `docs/artifacts_pane_visual_grammar.md`: the layout order list gains a new item 0,
  **Pane brief**, owned by `ArtifactsView` rather than by any pane; a new section
  specifies the three modes, the gutter grammar, the accent and style rules, the line
  cap, and the never-empty invariant that keeps it a fixed slot.
- `docs/ace.md`: describe the brief and the `D` cycle in the Artifacts section.
- `docs/configuration.md`: the `cycle_artifacts_description` remap, beside the existing
  `cycle_artifacts_split` paragraph.

### Tests

New `tests/ace/tui/test_artifacts_pane_brief.py` (renderer, no app):

- All three modes at 120 and 80 columns.
- Summary ellipsizes rather than wrapping in `summary` mode.
- The disclosure hint uses the resolved key display name and is dropped when the row is
  too narrow or there is nothing to disclose.
- The line cap emits the overflow row and never exceeds `max_lines`.
- The unconfigured hint appears only in `full` mode and only for a `fallback` summary.
- The accent appears in the gutter and the hint, and nowhere else.

New `tests/ace/tui/test_artifacts_description_modes.py`, modelled on the existing
`tests/ace/tui/test_artifacts_split_modes.py`: normalization of junk values, forward and
reverse cycling with wraparound, config seeding, the app watcher reaching the view,
click-to-cycle, and `display` toggling with `off`.

## Phase `hover`: Sub-tab hover tooltips

`src/sase/ace/tui/widgets/panel_tab_strip.py`:

- `PanelTab` gains `description: str = ""`.
- A new `on_mouse_move` reuses the exact centre-padding hit test `on_click` already
  performs against `self._tab_ranges` and `self._line_width` (cell offsets, not
  character counts — the existing docstring explains why that distinction matters when a
  two-cell icon is present), and sets `self.tooltip` to the hovered tab's description,
  or `None` when the pointer is between tabs.
- `on_leave` clears the tooltip.
- Both handlers no-op when no tab carries a description, so the six other
  `PanelTabStrip` call sites (Config Center, Statistics, Plugins browser, Projects, Help
  modal) are untouched until they opt in.

`ArtifactsView._panel_tabs()` passes `descriptor.description` as each tab's description.
This works in `off` mode, which is what makes turning the brief off a reasonable choice
rather than a loss of information.

New `tests/ace/tui/test_panel_tab_strip_tooltips.py`: hit-test correctness at the
`full`, `compact`, and `micro` tiers, with a two-cell icon present, with the strip
centre-padded in a wider container, and the no-description no-op.

## Phase `goldens`: Visual goldens and end-to-end verification

New PNG goldens (`tests/ace/tui/visual/snapshots/png/`):

- `artifacts_brief_summary_120x40` — a built-in pane's one-row brief.
- `artifacts_brief_full_120x40` — the expanded body.
- `artifacts_brief_narrow_80x24` — ellipsized summary, disclosure hint dropped.
- `artifacts_brief_provider_fallback_120x40` — an unconfigured provider pane showing the
  hint.

Then rebaseline. The brief adds one row above the content switcher, so **every**
existing `artifacts_*` golden shifts by one row: 43 files today, including the
`artifacts_split_*` and `copy_as_stitches_*` families. Run
`just test-visual --sase-update-visual-snapshots`, then inspect the
`.pytest_cache/sase-visual/` actual/expected/diff artifacts before accepting, per
`CLAUDE.md`. A golden that changes by anything other than the expected one-row shift
plus the new brief row is a bug in an earlier phase, not a golden to accept.

Finish with `just check-full` through `/sase_monitor` (it routinely outruns a turn),
using the `TESTING` / `TESTED` status pair.

## Risks and constraints

- **`toobig` lint** (`just _lint-toobig`, thresholds `1000 850 700`). This is why the
  resolution ladder, the mode helpers, and the brief widget are new modules rather than
  additions to `_artifact_tab_contract.py` (506 lines) or `view.py` (377).
- **Symvision** flags unused public symbols. Every new exported name must have a real
  caller or an explicit pragma; read `symvision.md` with `/sase_memory_read` before
  fixing a symvision failure.
- **Patch/stitch terminology lint** (`just _lint-patch-stitch-terminology`) gates
  user-facing prose. The authored copy uses "Patch", "stitch", "PR", and "commit"
  deliberately and consistently with the glossary; do not paraphrase it into
  "ChangeSpec" or "commit entry" vocabulary.
- **Prose formatting**: `just fmt-md-check` runs prettier at `printWidth: 88`,
  `proseWrap: always` over every Markdown file. Run `just fmt` after doc edits.
- **Config is not schema-validated at runtime**, so a schema mistake fails no test but
  degrades Config Center fidelity and editor completion. The bundled defaults _are_
  validated (`tests/test_config_schema_ace.py` and its siblings), so
  `default_config.yml` and the schema must move together.
