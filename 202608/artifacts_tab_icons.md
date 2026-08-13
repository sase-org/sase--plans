---
tier: epic
title: Icons for every Artifacts sub-tab, required per sidecar ref config
goal: 'Every Artifacts sub-tab renders a colored icon beside its name. The four fixed
  panes carry built-in marks, and every document-provider pane carries a mark declared
  as a required `ref.icon` field in its sidecar ref provider spec, so a sidecar repo
  chooses its own tab icon and the strip stays readable at any terminal width.

  '
phases:
- id: wire
  title: Required `ref.icon` in the artifact ref provider spec wire
  depends_on: []
  size: small
  description: 'wire: add a required `icon` string to the Rust `ArtifactRefProviderRefSpecWire`,
    validate it as a bounded one-or-two-cell glyph, and cover accept/reject and digest
    cases in the Rust unit tests.

    '
- id: strip
  title: Icons, cell-accurate click ranges, and reflow-to-fit in PanelTabStrip
  depends_on: []
  size: medium
  description: 'strip: give `PanelTab` an icon, render it between the number and the
    label, measure click ranges in terminal cells instead of characters, and add an
    opt-in reflow ladder that keeps every tab visible and clickable when the strip
    cannot fit its full render.

    '
- id: tabs
  title: Icons on Artifacts tab descriptors and in sidecar ref config
  depends_on:
  - wire
  size: medium
  description: 'tabs: add a built-in icon table for the four fixed panes, resolve
    each provider pane''s icon from its ref spec, accept `ref.icon` as a sidecar config
    override, give the built-in plan provider its mark, and keep an outdated provider
    plugin working behind a warning diagnostic.

    '
- id: research
  title: Research sidecar ref provider icon
  depends_on:
  - wire
  size: xsmall
  description: 'research: declare the research ref provider''s icon in the sase-research
    plugin so the Research pane matches the research tribe mark already configured
    there.

    '
- id: render
  title: Render icons in the Artifacts strip, then document and re-golden
  depends_on:
  - strip
  - tabs
  size: medium
  description: 'render: pass descriptor icons and the reflow opt-in into the Artifacts
    tab strip, update the ACE and configuration docs, extend the mechanical glyph
    audit to the new marks, and regenerate every affected PNG golden.'
proposed_by: bbugyi200.athena.z6.f2
create_time: 2026-08-13 09:16:19
status: wip
bead_id: sase-kv
---

- **PROMPT:** [prompts/202608/artifacts_tab_icons.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/artifacts_tab_icons.md)
- **BEAD:** [sase-kv](https://github.com/sase-org/sase--beads/blob/main/pages/sase-kv/README.md)

# Icons For Every Artifacts Sub-Tab, Required Per Sidecar Ref Config

## Problem

The ACE Artifacts tab strip identifies every pane by a number and a word:

```
 1 STITCHES │ 2 Patches │ 3 Beads │ 4 Plans │ 5 Researchs │ 6 Files
```

Nothing distinguishes a fixed pane from a sidecar-backed document pane, nothing tells a
reader at a glance which pane is which, and a sidecar repo has no way to give its pane a
visual identity. Every other tabbed surface in ACE already solved this: the notification
strip renders `⚑ ✖ ◈ ✉ ☾ ⊘` with per-tab colors, resolved through
`src/sase/ace/tui/widgets/notification_tab_style.py` and configurable under
`ace.notification_tabs`. The Artifacts strip is the odd one out.

There is a second, load-bearing problem that adding icons forces into the open. The
strip above already measures **78 cells** — two short of an 80-column terminal — and
`PanelTabStrip` handles overflow badly:

- `_build_content()` records `self._line_width = len(text.plain)` and every entry of
  `self._tab_ranges` in **characters**, while `on_click()` compares those ranges against
  `event.x`, which is in **terminal cells**. Any two-cell glyph (an emoji icon, for
  instance) silently shifts every click target to its right onto the wrong tab.
- The Artifacts strip passes neither `compact_below` nor `micro_below`
  (`src/sase/ace/tui/widgets/artifacts/view.py:66-72`), so `on_resize()` returns
  immediately and the strip never reflows. It simply clips, and a clipped tab is both
  invisible and — since `on_click` only knows the ranges built for the full render —
  effectively unclickable.

So this feature is not "add a glyph to a label". It is: define where a sidecar declares
its mark, resolve marks for every pane, and make the strip that renders them honest
about width.

## Goal

```
 1 ◉ STITCHES │ 2 ⎇ Patches │ 3 ◈ Beads │ 4 ✎ Plans │ 5 ∴ Researchs │ 6 ▤ Files
```

Each icon is drawn in its pane's existing accent color, bold on the active pane and
dimmed elsewhere, exactly as the notification strip already does. `ref.icon` becomes a
required field of the artifact ref provider spec, so a sidecar repo's own configuration
decides its pane's mark, and the strip degrades through a designed ladder instead of
clipping when the terminal is narrow.

## Design Decisions

These are the judgment calls behind the phases below. Review them first; each one is
reversible in isolation.

### D1. The icon lives at `ref.icon`, and it is required at the wire level

`icon` is added to `ArtifactRefProviderRefSpecWire` in the Rust core as a **required**
`String` with no serde default. That is what "required configuration field" means here:
any spec that reaches validation without one is rejected, in Rust, for every frontend —
which is exactly where the Rust/Python backend boundary in `CLAUDE.md` puts a schema
rule.

`ref.icon` (inside the `ref` mapping) rather than a top-level `icon` because that is
where every other authored field already lives, so it joins `_KNOWN_REF_CONFIG_KEYS` in
`src/sase/sidecar_ref_config.py` and becomes a legal `repos.sidecar.*.<role>.ref.icon`
override with no new merge path.

### D2. Do **not** bump `ARTIFACT_REF_PROVIDER_SPEC_WIRE_SCHEMA_VERSION`

Adding a required field is a breaking wire change, so bumping `1` -> `2` is the textbook
move. Do not do it here, for a concrete CI reason: `.github/workflows/ci.yml:397` and
`.github/workflows/publish.yml:247` install the **published floor**
(`sase-core-rs==<pyproject minimum>`, currently `0.26.6`) against this checkout's Python
and run it. A floor core would reject a `schema_version: 2` spec outright and fail that
job, which cannot be fixed without releasing core first — a deadlock.

Leaving the version at `1` gives the opposite, benign behavior: a floor core
deserializes the spec with serde's default unknown-field handling, ignores `icon`,
validates, and passes. Enforcement simply does not apply until the core carrying it is
installed. The regular CI lane builds `sase-org/sase-core` at HEAD
(`.github/workflows/ci.yml:41-68`), so it _does_ enforce. Note also that
`artifact_ref_provider_spec_wire_schema_version` is not in `REQUIRED_BINDINGS` or the
expected-version map in `tools/validate_sase_core_rs`, so nothing else probes it.

### D3. Three resolution rungs, so a missing icon never deletes a tab

Required must not mean fragile. A user whose Research pane vanishes because a plugin
release lagged has been served badly. The rungs, highest first:

1. **Declared.** `ref.icon` from the assembled spec: the sidecar's own choice, whether
   it came from the provider's registered spec or a local `ref.icon` override.
2. **Generic base.** `_default_document_spec()` in `src/sase/sidecar_ref_config.py`
   synthesizes the spec for a document role with no provider and no `ref` block; it
   carries `◆`, the generic document mark. An author writing an inline `ref:` spec
   inherits `◆` and overrides it with `ref.icon`.
3. **Plugin compatibility shim.** In `_validate_ref_providers()`
   (`src/sase/artifact_providers/registry.py`), a candidate ref-provider spec with no
   `ref.icon` gets `◆` injected before validation, plus a `missing_ref_provider_icon`
   **warning** diagnostic naming the plugin. Without this, an outdated third-party
   plugin fails validation, gets dropped from the registry, and a
   `ref: {use: <that provider>}` config then reports the misleading "provider is not
   installed". This shim is deliberately temporary — mark it with a comment saying it
   exists for plugins released before `ref.icon` and can be removed once none remain.

Rung 3 is the one place "required" is softened, and only for third-party code this repo
does not control. **If you would rather it fail hard, say so in review** — it is a
~10-line deletion and one test.

### D4. The icon set

Every glyph below is verified single-cell under `rich.cells.cell_len` and covered by the
fonts bundled at `tests/ace/tui/visual/fonts/`, so no PNG golden renders a `.notdef`
box. Phase `render` adds a mechanical audit that keeps this true.

| Pane                | Icon | Codepoint | Bundled coverage       | Why                                                            |
| ------------------- | ---- | --------- | ---------------------- | -------------------------------------------------------------- |
| Stitches            | `◉`  | U+25C9    | Fira Code, DejaVu Sans | a commit node, as drawn on a git graph                         |
| Patches             | `⎇`  | U+2387    | Fira Code, DejaVu Sans | the standard branch/alternative mark; a Patch is proposed work |
| Beads               | `◈`  | U+25C8    | DejaVu Sans            | identical to the `beads` notification tab icon                 |
| Files               | `▤`  | U+25A4    | Fira Code, DejaVu Sans | a page with ruled lines                                        |
| `plan` provider     | `✎`  | U+270E    | DejaVu Sans            | an authored document                                           |
| `research` provider | `∴`  | U+2234    | Fira Code, DejaVu Sans | identical to the research tribe icon `sase-research` ships     |
| generic document    | `◆`  | U+25C6    | Fira Code, DejaVu Sans | honestly generic; claims nothing specific                      |

Two of these are consistency wins worth protecting in review: `◈` is already
`ace.notification_tabs.beads.icon`, and `∴` is already `ace.tribes.research.icon` in
`sase-research`'s `default_config.yml`. DejaVu-only coverage is acceptable — five of the
six shipped notification icons are DejaVu-only today.

### D5. Fixed-pane icons are built-in constants, not configuration

Only sidecar-backed panes get a configurable icon, matching the request. Stitches,
Patches, Beads, and Files are not sidecars and are not provider-backed; their icons live
in an `ARTIFACTS_ICONS` table beside the existing `ARTIFACTS_ACCENTS`. An
`ace.artifact_tabs`-style user override for them is explicitly out of scope.

### D6. Scope boundary: the tab strip only

`ArtifactsTabDescriptor.icon` becomes available to every consumer, but this epic renders
it in exactly one place: the Artifacts tab strip. The command palette
(`src/sase/ace/tui/commands/catalog.py`) and the quickstart card
(`src/sase/ace/tui/widgets/tab_quickstart.py`) keep their current text. Adopting the
icon there is a reasonable follow-up; doing it here would widen the golden-image churn
for no additional user-visible answer to the request.

## Required `ref.icon` In The Artifact Ref Provider Spec Wire

Phase id: `wire`. This phase is in the **sase-core** repo. Open it with
`sase repo open sase-core -r "<reason>"` and use only the path that command prints.

Edit `crates/sase_core/src/artifact_ref/provider_spec.rs`:

1. Add `pub icon: String,` to `ArtifactRefProviderRefSpecWire`. Place it after `kind`
   and before `expansion_format`. Give it **no** `#[serde(default)]` — its absence must
   be a deserialization error, which surfaces through
   `artifact_ref_provider_spec_from_pydict` in `crates/sase_core_py/src/lib.rs` as
   `spec is not a valid ArtifactRefProviderSpecWire dict: missing field 'icon'`. That
   message is already actionable; no binding change is needed.
2. Add a `validate_tab_icon` free function and call it from
   `validate_artifact_ref_provider_spec` right after the `kind` checks. Rules, in this
   order, each with its own error message:
   - non-empty, and equal to its own `trim()` (no leading or trailing whitespace);
   - no control characters (`char::is_control`);
   - at most 8 `char`s and at most 32 UTF-8 bytes — a bound, not a shape test, so
     combining marks and ZWJ emoji sequences stay legal;
   - total display width of 1 or 2 cells via `unicode_width::UnicodeWidthStr::width`.
     `unicode-width = "0.2"` is already a dependency of the `sase_core` crate; no
     `Cargo.toml` change is needed.

   Two cells is the ceiling rather than one so emoji remain usable; phase `strip` makes
   the renderer cell-accurate, so a two-cell icon renders and clicks correctly. Say so
   in the function's doc comment.

3. Do not touch `ARTIFACT_REF_PROVIDER_SPEC_WIRE_SCHEMA_VERSION` (see **D2**). Add a
   short comment above the new field recording that decision and its CI reason, so a
   later reader does not "fix" it.

`icon` participates in the digest automatically, because
`artifact_ref_provider_spec_digest` serializes the whole struct. That is correct and
wanted: changing a sidecar's icon should change its spec digest.

Tests, in the existing `mod tests` of the same file:

- extend `valid_spec()` with an icon and confirm
  `valid_spec_passes_and_digest_is_stable` still passes;
- a new `rejects_missing_or_malformed_icon` covering empty, whitespace-padded, a control
  character, an over-long string, and a three-or-more-cell string;
- a case asserting a legal two-cell emoji icon is accepted;
- a digest case asserting two specs differing only in `icon` produce different digests.

Land this phase in `sase-core` **before** the `tabs` and `research` phases merge, since
both send `icon` through this validator.

## Icons, Cell-Accurate Click Ranges, And Reflow-To-Fit In PanelTabStrip

Phase id: `strip`. All edits are in `src/sase/ace/tui/widgets/panel_tab_strip.py`. This
phase touches no Artifacts code and can proceed in parallel with `wire` and `tabs`.

`PanelTabStrip` has six other callers (`statistics_pane.py`, `config_center_catalog.py`,
`config_center_modal.py`, `plugins_browser_layout.py`, `plugins_browser_pane.py`,
`projects_pane.py`, `help_modal/modal.py`). Every change here must be a no-op for a
strip that passes no icons and does not opt into reflow.

### Icons on `PanelTab`

Add `icon: str = ""` to the `PanelTab` dataclass. In `_build_content()`, when `tab.icon`
is non-empty, append it after the number (or after the leading pad when `show_numbers`
is false) and before the label, followed by one space. The icon is styled with the same
`number_style` the shortcut uses — `tab.accent_color` when active, `#666666` otherwise —
so the mark carries the pane's color exactly as the notification strip does. An empty
`icon` appends nothing, which is what keeps the six existing callers byte-identical.

### Measure in cells, not characters

`_build_content()` currently advances positions with `len(text.plain)`. Replace that
with a running column counter incremented by `rich.cells.cell_len(fragment)` for each
appended fragment, and set `self._line_width` from the same counter — the pattern
`_render_tabs()` in `src/sase/ace/tui/modals/notification_modal_tags.py:258-303` already
uses, and for the same reason documented there. `on_click()` needs no change once the
ranges it reads are in the same unit as `event.x`. This is a latent bug fix today and a
correctness requirement the moment a user configures a two-cell icon.

### Reflow to fit

Add `reflow_to_fit: bool = False` to `__init__`. When it is set:

- `_build_content()` renders the `full` tier, and if `cell_len` of the result exceeds a
  known positive widget width, retries at `compact`, then `micro`, taking the first tier
  that fits and keeping `micro` if none do. Track the chosen tier in `self._tier` so the
  existing `set_active_tab`/`set_tabs` refresh paths reuse it.
- `on_resize()` must stop returning early when `reflow_to_fit` is set and both
  thresholds are `None`; re-run the ladder and update only when the tier changes.
- In the `micro` tier, drop the label of every **inactive** tab and keep the active
  tab's label, **but only when every tab in the strip has a non-empty icon**. That
  condition is the correctness guard, not a style knob: an icon-less tab rendered
  without its label would be an unidentifiable blank. When any tab lacks an icon,
  `micro` keeps today's label-shortening behavior. No extra constructor flag is needed.

Explicit `compact_below`/`micro_below` thresholds keep working unchanged and take
precedence; `reflow_to_fit` only supplies a ladder for strips that passed neither.

Concrete widths for the six-pane strip, so the ladder is verifiable rather than assumed
(separator `" │ "` in full, `"│"` in compact and micro):

| Tier    | Per-tab shape                         | Total |
| ------- | ------------------------------------- | ----- |
| full    | `" {digit} {icon} {LABEL} "`          | 90    |
| compact | `"{digit} {icon} {label}"`            | 68    |
| micro   | `"{digit}{icon}"`, active keeps label | 32    |

Tests, in `tests/ace/tui/test_panel_tab_strip_compact.py`:

- an icon renders between the number and the label, and an empty icon changes nothing;
- with a deliberately two-cell icon, `_tab_ranges` and `_line_width` are in cells, and a
  `pilot.click` at the computed offset of the **last** tab selects that tab (this is the
  regression the character-count version fails);
- `reflow_to_fit` picks `full`, `compact`, and `micro` at three widths, and every tab id
  remains present in `_tab_ranges` at each tier;
- in `micro` with every tab iconed, inactive labels are gone and the active label
  remains; with one tab's icon blank, every label survives;
- an existing strip with `compact_below`/`micro_below` and no icons renders exactly as
  it does today (guard the six other callers).

## Icons On Artifacts Tab Descriptors And In Sidecar Ref Config

Phase id: `tabs`. Depends on `wire`, whose validator these specs must satisfy. Run
`just install` first so the local `sase_core_rs` is rebuilt from the sase-core checkout
carrying phase `wire`.

### `src/sase/ace/tui/artifact_tabs.py`

- Add `icon: str = ""` to `ArtifactsTabDescriptor`, after `pane_id` among the defaulted
  fields. It defaults to empty so a hand-built descriptor in a test is still
  constructible; every production path fills it.
- Add, beside `ARTIFACTS_ACCENTS`:

  ```python
  ARTIFACTS_ICONS: dict[str, str] = {
      "stitches": "◉",
      "patches": "⎇",
      "beads": "◈",
      "files": "▤",
  }

  DEFAULT_DOCUMENT_TAB_ICON = "◆"
  ```

  Do not add compatibility aliases; `ARTIFACTS_ACCENTS` carries `plans`/`other` only for
  older callers, and nothing needs an icon under those keys.

- Add `sanitize_tab_icon(raw: object) -> str`, returning a safe icon or `""`. Reuse the
  existing definitions rather than writing new ones: call
  `sase.notification_gates.model_validation.validate_icon` inside a `GateError` guard,
  then reject anything whose `rich.cells.cell_len` exceeds 2 — the same two-rung check
  as `notification_tab_style._sanitize_icon`, for the same reason. Keep this in
  `artifact_tabs.py`; it adds a `rich` import but no Textual dependency, so the module's
  widget-free contract holds.
- `_fixed_descriptor()`: pass `icon=ARTIFACTS_ICONS[subtab]`.
- `_provider_descriptors()`: read the icon from the assembled spec —
  `spec["ref"]["icon"]` when `spec["ref"]` is a `Mapping` — through `sanitize_tab_icon`,
  falling back to `DEFAULT_DOCUMENT_TAB_ICON` when it is missing or rejected. Add it to
  `__all__` alongside the new table. Do **not** add either to the `widgets/artifacts`
  re-export surface; symvision flags unused re-exports.

### `src/sase/sidecar_ref_config.py`

- Add `REF_ICON_CONFIG_KEY = "icon"`, include it in `_KNOWN_REF_CONFIG_KEYS`, and export
  it in `__all__`. This is what makes `repos.sidecar.*.<role>.ref.icon` a legal override
  instead of an "unknown sidecar ref field(s)" diagnostic. No `_ref_override` change is
  needed: `icon` is a scalar, and scalars already replace on deep-merge, so a local
  `ref.icon` correctly beats a `use:`-supplied provider icon.
- `_default_document_spec()`: add `"icon": DEFAULT_DOCUMENT_TAB_ICON` to the synthesized
  `ref` mapping (rung 2 of **D3**). Import the constant from
  `sase.ace.tui.artifact_tabs` only if that does not introduce an import cycle — check
  first, and if it does, define the single-character constant locally in this module and
  have `artifact_tabs` import it from here instead. One definition, either way.

### `src/sase/artifact_providers/_builtin.py`

Add `"icon": "✎"` to `builtin_plan_ref_provider_spec()`'s `ref` mapping. This is the
"built-in `plans` sidecar ref configuration" the request names.

### `src/sase/artifact_providers/registry.py`

Implement rung 3 of **D3** in `_validate_ref_providers()`: before
`validate_ref_provider_spec`, if the candidate's `ref` mapping has no non-empty string
`icon`, set it to the generic mark and append an
`ArtifactProviderDiagnostic(code="missing_ref_provider_icon", severity="warning", ...)`
naming the provider and its provenance label. Comment it as a removable compatibility
shim for plugins released before `ref.icon`.

### `src/sase/ace/testing/_startup.py`

`_fast_artifacts_subtabs()` hand-builds its `ref:plan` descriptor. Give it `icon="✎"` so
every AcePage-driven test and PNG golden exercises a real provider icon rather than the
fallback.

### `src/sase/config/sase.schema.json`

Add `icon` to the `sidecarRef` definition's `properties` as
`{"type": "string", "minLength": 1, "description": "..."}`. Do **not** add it to a
`required` list: a role using `ref: {use: <provider>}` inherits the provider's icon and
must not be forced to restate it, and the generic base supplies one otherwise. The
requirement is enforced on the _assembled_ spec in Rust, which is the only place that
sees the merged result. Note that `sidecarRef` sets `additionalProperties: false`, so
omitting this edit makes every `ref.icon` a schema error — it is not optional cleanup.

Tests:

- `tests/test_sidecar_ref_config.py`: a `ref.icon` override reaches
  `policy.spec["ref"]["icon"]`; `icon` is accepted rather than reported as an unknown
  field; a role with no `ref` block gets the generic mark; a local `ref.icon` beats a
  `use:`-supplied provider icon.
- `tests/test_artifact_provider_registry.py`: the builtin `plan` provider record carries
  `✎`; a plugin spec with no icon is admitted with the generic mark **and** a
  `missing_ref_provider_icon` warning diagnostic; a plugin spec with a malformed icon
  (three cells) is rejected by validation as before.
- New `tests/ace/tui/test_artifact_tab_icons.py`: each fixed pane's icon; a provider
  descriptor takes its icon from its spec; a provider spec with a missing or junk icon
  falls back to the generic mark; `sanitize_tab_icon` rejects multi-glyph, control, and
  over-wide input; every icon this repo can render without configuration is exactly one
  cell wide.

## Research Sidecar Ref Provider Icon

Phase id: `research`. This phase is in the **sase-research** repo. Open it with
`sase repo open sase-research -r "<reason>"` and use only the path that command prints.
It depends on `wire` only so the field name and validation rules are settled first; the
edit itself is inert against an older core, which ignores unknown fields.

In `src/sase_research/provider.py`, add `"icon": "∴"` to `RESEARCH_REF_PROVIDER_SPEC`'s
`ref` mapping, beside `kind`. Add or extend a test asserting the spec declares an icon
and that it matches the `ace.tribes.research.icon` value in
`src/sase_research/default_config.yml` — that shared mark is the point, and a test is
what stops the two drifting.

Do not touch `RESEARCH_HIGHLIGHTS_HOOK_SPEC`; file-hook specs have no ref block.

Landing note: `sase-research` is consumed as a released wheel, not from a checkout, so
until a release carries this change the installed plugin hits rung 3 of **D3** — the
Research pane renders `◆` and `sase doctor -C config.repos` shows the
`missing_ref_provider_icon` warning. That is the designed degradation, not a bug; it is
also the reason rung 3 exists.

## Render Icons In The Artifacts Strip, Then Document And Re-Golden

Phase id: `render`. Depends on `strip` and `tabs`.

### Wire the strip up

In `src/sase/ace/tui/widgets/artifacts/view.py`:

- `_panel_tabs()` passes `icon=descriptor.icon` into each `PanelTab`.
- `compose()` passes `reflow_to_fit=True` to the `PanelTabStrip` constructor, alongside
  the existing `show_numbers=True, uppercase_active=True`.

Nothing else in the Artifacts widget tree changes.

### Docs

- `docs/ace.md`: in **Document Provider Panes** (near line 367), state that each pane
  renders an icon in its accent color, that the four fixed marks are built in, and that
  a provider pane's mark comes from its sidecar `ref.icon`. In the number-keys paragraph
  (near line 97), add one sentence that the strip narrows by dropping inactive labels
  before it ever clips a tab. Leave the **Files Pane** section (near line 459) alone.
- `docs/configuration.md`: add a `repos.sidecar.*.<role>.ref.icon` row to the sidecar
  field table (near line 1700) — type string, default "role/provider dependent",
  described as the Artifacts tab mark — and add `icon` to the `research` example in the
  YAML block above it.
- `docs/artifact_references.md`: add `icon` to the inline `design` provider spec example
  in **Provider Specs** (near line 120) and one sentence naming it a required field of
  an assembled spec.
- `docs/plugins.md`: near line 605, note that a ref provider spec must declare
  `ref.icon`, and that a spec without one is admitted with a generic mark plus a warning
  during the compatibility window.

Do **not** touch `sase/memory/*.md`, `AGENTS.md`, or the generated provider instruction
shims (`CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`). None document this behavior,
and editing them needs separate explicit user permission.

### Glyph audit

`tests/ace/tui/visual/test_tab_icon_glyphs.py` already performs exactly the right check
for notification icons — bundled-font coverage plus a rasterizes-to-ink assertion —
using the shared helpers in `tests/ace/tui/visual/_glyph_audit.py`. Extend its
`_AUDITED_ICONS` to include `ARTIFACTS_ICONS.values()`, `DEFAULT_DOCUMENT_TAB_ICON`, and
the builtin plan provider's icon read from `builtin_plan_ref_provider_spec()`, and widen
the module docstring to say it now covers both strips. Reading the plan icon from the
spec rather than restating `"✎"` is what keeps the audit honest if someone changes it
later.

`sase-research`'s `∴` cannot be audited from this repo unless the plugin happens to be
installed; do not add a conditional import for it. It is covered by Fira Code and DejaVu
Sans (verified during design), and phase `research` pins it with its own test.

### PNG goldens

Every Artifacts-tab snapshot renders the strip, so all of them change. Expect the
Artifacts strip to gain six icons under the fast test strip (`◉ ⎇ ◈ ✎ ▤` across five
panes) and possibly to switch tiers at narrow snapshot widths.

1. Run `just test-visual` and expect broad `artifacts_*` failures.
2. Inspect at least one actual/expected/diff triple under `.pytest_cache/sase-visual/`
   and confirm the change is icons and, where the width demands it, the reflow ladder.
   **Any `.notdef` box means stop** — the glyph audit above should have caught it, so a
   box here means the audit is not covering what the renderer draws.
3. Re-run with `--sase-update-visual-snapshots` to accept, then `just test-visual`
   clean.

Be aware of a known local hazard recorded during design: this host currently shows a
large number of pre-existing `test-visual` failures unrelated to any change. Establish a
`master` baseline before attributing a failure to this work.

### Gates

- `just install` first — ephemeral workspace, and this epic's core dependency changed.
- `just check` after each phase's edits.
- `just test-visual` as above.
- `just check-full` before landing the combined tree, launched through `/sase_monitor`
  (`sase monitor start --command 'just check-full' …` with a `--next` action). Never
  inline.
- In `sase-core`, run that repo's own `just check`/`cargo test` for phase `wire`.

### Manual verification

Run `sase ace` in a project with both the `plan` and `research` providers configured and
open Artifacts. Confirm the strip reads
`1 ◉ STITCHES │ 2 ⎇ Patches │ 3 ◈ Beads │ 4 ✎ Plans │ 5 ∴ Researchs │ 6 ▤ Files`, that
each mark is drawn in its pane's accent color, and that the active pane's mark is bold
while the rest are dim. Then narrow the terminal below ~90 columns and confirm the strip
tightens and then drops inactive labels rather than clipping, and that clicking the
right-most tab still selects it at every width.

## Risks

- **A glyph that renders as tofu in Bryan's terminal.** The bundled-font audit proves
  the goldens are clean, not that a given terminal font carries the mark. `◉ ⎇ ▤ ∴ ◆`
  are in Fira Code itself; `◈` and `✎` are DejaVu-only, matching five of the six
  notification icons already shipped. If one looks wrong in practice, it is a one-line
  table change plus a golden refresh.
- **Cross-repo landing order.** `tabs` and `research` both send `icon` through the
  validator from `wire`. Land `wire` first. Because of **D2**, a stale core degrades to
  ignoring the field rather than failing, which bounds the damage of getting this wrong.
- **The compatibility shim outliving its purpose.** Rung 3 of **D3** silently accepts an
  icon-less plugin spec forever if nobody removes it. It emits a warning diagnostic and
  carries a comment saying so; that is the mitigation, and deleting it later is a small,
  well-marked change.
- **Reflow changes a widget six other panels use.** Every behavior added in `strip` is
  gated on either a non-empty icon or `reflow_to_fit=True`, and the phase includes an
  explicit regression test that an icon-less, threshold-driven strip renders exactly as
  it does today.
- **Snapshot churn volume.** Every Artifacts golden moves at once, which makes a real
  regression easy to wave through. The `.notdef` stop-condition in the golden step is
  there specifically to catch the one failure mode a reviewer cannot see by eye.
