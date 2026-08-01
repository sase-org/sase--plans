---
tier: tale
title: Required description field for configured agent tribes
goal:
  Every configured agent tribe carries a required one-line description that ACE renders in the Agents-tab metadata panel
  when that tribe's panel is selected.
create_time: 2026-07-31 07:34:33
status: done
---

- **PROMPT:** [prompts/202607/tribe_description.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/tribe_description.md)

# Plan: Required `description` For Configured Agent Tribes

## Goal

Add a `description` field to `ace.tribes.<name>` config entries, make it **required whenever a tribe has any
configuration at all**, ship excellent descriptions for every built-in tribe, and render the description in the ACE
Agents-tab metadata panel whenever a tribe panel is selected.

## Current State

Per-tribe display config already supports `icon`, `color`, and `initially_expanded`:

- Schema: `src/sase/config/sase.schema.json:471-501` (`ace.tribes`, `propertyNames` pattern `^[A-Za-z0-9_.-]+$`,
  `additionalProperties` = a closed object with the three fields).
- Bundled defaults: `src/sase/default_config.yml:97-120` — six entries: `default`, `epic`, `research`, `chop`, `pinned`,
  `review`.
- Resolution: `src/sase/ace/tui/models/tribe_display.py` — `_TribeDisplay` dataclass, `_sanitize_icon`,
  `_sanitize_color`, and `_tribe_displays_for_token()` (an `lru_cache(maxsize=1)` keyed on `current_config_token()`).
  `tribe_display_for(panel_key)` maps the `None` panel key to the config key `default` and returns
  `DEFAULT_TRIBE_DISPLAY` for tribes with no config. There is **no** inheritance from the `default` entry to other
  tribes; each key resolves independently.
- Snapshot: `src/sase/ace/tui/models/agent_tribe_summary.py` — `AgentTribeSummarySnapshot` is the pure, renderer-facing
  document model; `_panel_label()` (line 162) is the existing precedent for reading tribe display config at
  snapshot-build time.
- Renderer: `src/sase/ace/tui/widgets/prompt_panel/_agent_display_tribe.py:155-194` — `_append_header()` emits the
  `TRIBE` heading plus `Name:`, `Status:`, `Composition:`, `Runtime:` and the fold line. `build_tribe_detail_text()`
  returns immediately after the header when `cheap=True`.
- Docs: `docs/configuration.md:464-480` (example) and `docs/configuration.md:615-640` (`ace.tribes` field table);
  `docs/ace.md:1030-1045` (Agents-tab panel prose).

### Verified findings that shape this plan

1. **The house precedent for a required `description` already exists.** `llm_provider.model_aliases.custom.<name>` in
   `src/sase/config/sase.schema.json` uses exactly the shape we want: `"required": ["model", "description"]` with
   `description` typed `{"type": "string", "minLength": 1, "maxLength": 160}`. It is enforced by a doctor check
   (`src/sase/doctor/checks_config_model_aliases.py:265-280`) and rendered by the Models panel
   (`src/sase/ace/tui/modals/models_panel_rendering.py:287-305`) with `_DESCRIPTION_STYLE = "italic #B0B0B0"` and a
   `_DESCRIPTION_MISSING_STYLE = "italic #D7AF87"` fallback reading
   `no description - set llm_provider.model_aliases.custom.<name>.description`. **This plan mirrors that precedent field
   for field, style token for style token.** Do not invent a new idiom.

2. **Schema `required` is real enforcement, not decoration.** The Rust config validator supports `required`
   (`sase-core/crates/sase_core/src/config/validate.rs:144-155`, emitting code `required_missing` at the _child_ path
   `ace.tribes.<name>.description`). It runs inside `config_inventory` and `config_plan_edit`, and the ACE Config Center
   refuses to write while any error-severity diagnostic is present (`src/sase/ace/tui/modals/config_transaction.py:318`,
   `ConfigTransactionPreview.is_valid`).

3. **No Rust change is required.** `sase-core` holds no copy of `sase.schema.json`; the schema is passed in as data from
   this repo (`crates/sase_core/src/config/schema.rs:3`). Nothing tribe-specific lives in the Rust core.

4. **Deep merge protects existing configs.** `merge_config_sources` (`src/sase/config/loading.py:149-181`) deep-merges
   layers, so a user entry like `epic: {icon: "X"}` inherits the bundled `description`. Only a _brand-new_ user tribe
   (or an explicit `description: ""`) can violate the requirement.

5. **The user currently has no user-defined tribes.** Verified three ways: `grep -rn tribes` over the chezmoi checkout
   (`home/dot_config/sase/sase.yml`, `sase_athena.yml`, `sase_kellys_mbp.yml`) finds nothing; the live
   `~/.config/sase/sase.yml` has no `ace.tribes` key; and `load_merged_config()["ace"]["tribes"]` returns exactly the
   six bundled entries. Every non-built-in tribe in `~/.sase/agent_tribes.json` is an ad-hoc epic-bead id (`sase-5l`,
   `sase-63`, …) with no configuration, so none of them require a description. **See step 8** — re-verify before
   concluding no chezmoi edit is needed.

## Design

### 1. One declarative rule, three surfaces

The requirement lives in exactly one place — the JSON Schema — and shows up wherever config is already checked:

| Surface                         | Behavior                                                                     |
| ------------------------------- | ---------------------------------------------------------------------------- |
| Config Center (raw-YAML editor) | `required_missing` error blocks the write until a description is added.      |
| `sase doctor -C config.tribes`  | New WARN check names every offending tribe and the exact key to set.         |
| Agents-tab metadata panel       | Renders an inline `no description - set ace.tribes.<name>.description` line. |

Rationale for all three: Bryan edits config as text in chezmoi, so the schema gate alone would never be seen until an
unrelated Config Center edit was blocked. The doctor check makes it proactive; the panel line makes it honest at the
point of use.

### 2. Where the description renders

Directly beneath the `Name:` line in the `TRIBE` header block, indented two spaces, in `italic #B0B0B0`:

```
TRIBE
Name: ▲ @epic
  Epic phase-worker clans from sase bead work, one member per phase of an approved plan.
Status: RUNNING [R2 D1]
Composition: 1 clan · 2 lanes · 2 nested
Runtime: 12m
Fold: 1/4
```

Design decisions, with reasons:

- **Subtitle, not a `Description:` label row.** The other header rows are short label/value pairs; a human sentence in
  that column would push a long value past the label gutter and wrap badly. An indented italic line reads as a subtitle
  of the name it follows and matches the Models-panel description strip.
- **Unconditional across fold levels, and included in the `cheap` header.** It is identity, not volume — one line at
  every level. Rendering it in the cheap path keeps fast and full paints the same height, so the panel never jumps.
- **Plain text, never markup.** Append the string directly (`text.append(...)`), never `Text.from_markup`, so a bracket
  in a description cannot corrupt the document (unlike `clan_summary`, which is deliberately markup-aware).
- **Unconfigured tribes render nothing.** Ad-hoc tribes (`@sase-63`) have no config, therefore no requirement and no
  line — the panel stays clean. The missing-description line appears **only** for a tribe that has config but no usable
  description, which is precisely the schema-invalid state.

### 3. Sanitization

Mirror `_sanitize_icon` / `_sanitize_color`: non-`str` → `""`; strip; drop control characters (Unicode category `Cc`);
collapse every internal whitespace run (including newlines) to a single space; truncate to
`MAX_TRIBE_DESCRIPTION_CHARS = 160` with a trailing `…`. The schema's `maxLength: 160` is the authoring contract; the
sanitizer is the defensive floor because the TUI reads merged config that was never schema-validated at load time.

### 4. Non-goals

- No `description` in `@tribe` prompt completions, the tribe modal (`N`), panel border titles, or
  `sase agent tribe list --json`. Metadata panel only for this change.
- No general "surface every `config_inventory` schema diagnostic" doctor check (worth doing, but it is its own feature
  and would change doctor output for unrelated keys).
- No auto-derived descriptions for ad-hoc epic-bead tribes.
- No inheritance of `default`'s description by other tribes.

## Implementation

### Step 1 — Schema (`src/sase/config/sase.schema.json`)

In the `ace.tribes` `additionalProperties` object (around line 477):

- Add to `properties`:
  ```json
  "description": {
    "type": "string",
    "minLength": 1,
    "maxLength": 160,
    "description": "One-line explanation of what this tribe is for, shown in the ACE Agents-tab metadata panel when the tribe panel is selected. Required for every configured tribe."
  }
  ```
- Add `"required": ["description"]` as a sibling of `"additionalProperties": false` in that same object.
- Update the `ace.tribes` container `description` string to mention that every configured tribe must carry a
  description.

### Step 2 — Bundled defaults (`src/sase/default_config.yml:97-120`)

Add a `description` to all six entries. Use these strings verbatim (each ≤ 160 chars, each fits one line at typical
panel width, none contain backticks or Rich markup):

```yaml
tribes:
  default:
    icon: "⌂"
    color: "#87D7FF"
    description: "Agents with no assigned tribe. Presentation-only: never written to the tribe store."
  epic:
    icon: "▲"
    color: "#AF87FF"
    description: "Epic phase-worker clans from sase bead work, one member per phase of an approved plan."
  research:
    icon: "∴"
    color: "#5FD7AF"
    description: "Research agents that answer open questions with durable reports instead of code changes."
  chop:
    icon: "†"
    color: "#FFAF5F"
    initially_expanded: false
    description: "Scheduled AXE chop automation; starts collapsed so routine background work stays quiet."
  pinned:
    icon: "◆"
    description: "Agents promoted by hand to keep them close; the value the tribe modal seeds by default."
  review:
    icon: "◉"
    description: "Mentor, CRS, and hook review runners that AXE groups here, plus reviews you tag yourself."
```

Also extend the explanatory comment block above `tribes:` (lines 97-102) with one sentence: a description is required
for every configured tribe and is shown in the Agents-tab metadata panel.

### Step 3 — Display resolution (`src/sase/ace/tui/models/tribe_display.py`)

- Add `MAX_TRIBE_DESCRIPTION_CHARS = 160`.
- Extend `_TribeDisplay` with `description: str = ""` and `configured: bool = False`. `DEFAULT_TRIBE_DISPLAY` keeps both
  defaults, so "has no config" stays distinguishable from "has config but no description".
- Add `_sanitize_description(raw: object) -> str` implementing the rules in Design §3.
- In `_tribe_displays_for_token`, populate `description=_sanitize_description(raw.get("description", ""))` and
  `configured=True`.
- Add `tribe_config_key(panel_key: PanelKey) -> str` returning `"default"` for `None` and the panel key otherwise, and
  use it in `tribe_display_for` and `tribe_identity_colors` in place of the two inline `config_key = ...` expressions
  (three call sites total, so symvision stays satisfied).
- Export `MAX_TRIBE_DESCRIPTION_CHARS` and `tribe_config_key` from `__all__`.

### Step 4 — Snapshot model (`src/sase/ace/tui/models/agent_tribe_summary.py`)

- Add two defaulted fields to `AgentTribeSummarySnapshot`: `description: str = ""` and
  `description_missing: bool = False`. They must come after `entry_target` (which already has a default).
- Add a `_panel_description(panel_key) -> tuple[str, bool]` helper next to `_panel_label` that calls `tribe_display_for`
  and returns `(display.description, display.configured and not display.description)`.
- Populate both fields in `build_agent_tribe_summary_snapshot` (line 494).

### Step 5 — Renderer (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_tribe.py`)

- Add module constants `_DESCRIPTION_STYLE = "italic #B0B0B0"` and `_DESCRIPTION_MISSING_STYLE = "italic #D7AF87"` (same
  values as `models_panel_rendering.py:41-42`).
- In `_append_header`, immediately after the `Name:` line:
  ```python
  if snapshot.description:
      text.append("  ")
      text.append(f"{snapshot.description}\n", style=_DESCRIPTION_STYLE)
  elif snapshot.description_missing:
      text.append("  ")
      text.append(
          f"no description - set ace.tribes.{tribe_config_key(snapshot.panel_key)}.description\n",
          style=_DESCRIPTION_MISSING_STYLE,
      )
  ```
  Import `tribe_config_key` alongside the existing `tribe_identity_style` import from `...models.tribe_display`.
- Nothing else in the file changes; the header is inside the `cheap` path by construction.

### Step 6 — Doctor check

- New `src/sase/doctor/checks_config_tribes.py` with `check_config_tribes() -> DiagnosticCheck`, modeled on
  `checks_config_model_aliases.py` (read it first for the exact `DiagnosticCheck` shape and `MAX_DETAIL_ROWS` use):
  - Read `load_merged_config()["ace"]["tribes"]`; tolerate a missing/non-dict value by returning `OK`.
  - Flag an entry when it is not a dict, or when `description` is absent, not a `str`, or blank after stripping.
  - `status = "WARN"` when problems exist, else `"OK"`; summary `"N configured tribe(s) missing a description"` /
    `"N configured tribe(s) documented"`; details capped at `MAX_DETAIL_ROWS`, each naming
    `ace.tribes.<name>.description`; `next_steps` pointing at `~/.config/sase/sase.yml`.
  - `data` carries `{"tribe_count": ..., "problem_count": ..., "problems": [...]}`.
- Register in `src/sase/doctor/checks_config.py`: import it, add a
  `CheckSpec(id="config.tribes", group="config", title="Tribe descriptions", runner=check_config_tribes)` to
  `config_check_specs`, and add the `_check_config_tribes = check_config_tribes` module alias that every other check in
  that file has.

### Step 7 — Docs

- `docs/configuration.md:615-640`: add a `description` row to the `ace.tribes` field table (type `str`, **required**, no
  default) and a sentence explaining that any tribe entry must carry one, that ACE shows it in the Agents-tab metadata
  panel, that deep merge means overriding only `icon` keeps the bundled description, and that a missing description is
  an error-severity config diagnostic which blocks Config Center writes until fixed. Mention
  `sase doctor -C config.tribes`.
- `docs/configuration.md:464-480`: add `description` to the `ace.tribes` YAML example.
- `docs/ace.md:1030-1045`: extend the tribe-panel paragraph — the selected `TRIBE` header shows the configured
  description under `Name:`, and a configured tribe with no description shows an inline fix hint instead.

### Step 8 — Chezmoi user tribes

Re-verify before doing anything: open the repo with the `/sase_repo` skill (`sase repo open chezmoi -r "<reason>"`) and
`grep -rn "tribes:" home/dot_config/sase/`. As of planning, there are **no** user-defined tribes there, so no chezmoi
edit is expected and the built-in descriptions from step 2 cover every configured tribe. If entries have appeared since,
add an equally excellent one-line description to each, keeping the same voice as the built-ins (concrete, ≤ 160 chars,
no markup), and run `chezmoi apply` guidance past the user rather than applying it yourself.

## Testing

- `tests/test_config_schema_tribes.py`: add `description` to the accepted fixtures; add rejection cases for a tribe
  entry with no `description`, `description: ""`, a non-string description, and a 161-character description.
- `tests/ace/tui/models/test_tribe_display.py`: the existing `_TribeDisplay(...)` equality assertions need
  `configured=True` and the new `description`. Add coverage for whitespace/newline collapsing, control-character
  rejection, non-string input, 160-char truncation with `…`, `configured=False` for unknown tribes, and
  `tribe_config_key(None) == "default"`.
- `tests/ace/tui/models/test_agent_tribe_summary.py`: snapshot carries the configured description; a configured tribe
  with a blank description sets `description_missing=True`; an unconfigured tribe sets neither.
- `tests/ace/tui/widgets/test_agent_display_tribe.py`: description line renders under `Name:` with `italic #B0B0B0`; the
  missing-description hint renders with `italic #D7AF87` and the exact `ace.tribes.<name>.description` key (including
  the `default` mapping for the `None` panel key); the description is present in the `cheap` header (extend
  `test_cheap_tribe_paint_is_header_only`); a description containing `[bold]` renders literally.
- `tests/doctor/test_checks_config_tribes.py` (new): OK when every configured tribe has a description; WARN naming each
  offender for missing / blank / non-string / non-dict entries; detail rows capped.
- Visual PNG snapshots: the `TRIBE` header grows one line, so goldens change. Run `just test-visual`, confirm every
  failure is the expected extra description line, then regenerate with `--sase-update-visual-snapshots` and eyeball the
  resulting PNGs. Known affected files: `tests/ace/tui/visual/test_ace_png_snapshots_agents_tribe_panel.py`
  (`agents_tribe_panel_display_config_120x40`, `agents_tribe_panel_level_1..4_120x40`,
  `agents_tribe_panel_isolation_armed_120x40`, `agents_tribe_panel_selected_expanded_120x40`) and
  `tests/ace/tui/visual/test_ace_png_snapshots_agents_panels.py` (the collapsed-panel summary assertions at lines
  110-135 render the `@chop` `TRIBE` document). Add one plain-text assertion in the tribe-panel visual test that the
  `@epic` description appears in the metadata panel.
- Finish with `just install` then `just check` from the workspace directory.

## Risks

- **Config Center lockout is global.** Any error diagnostic anywhere blocks every Config Center write, so a tribe
  without a description blocks unrelated edits until fixed. This is the intended strictness of "required", it cannot
  trigger for the bundled tribes after step 2, and the doctor check plus the panel hint both name the exact key to set.
  Call it out explicitly in the docs (step 7).
- **Snapshot churn.** Adding a header line touches several goldens; regenerate deliberately and review the diffs rather
  than blanket-accepting.
- **Description length.** Descriptions longer than roughly 90 characters wrap in a narrow metadata panel, and the
  wrapped continuation is not indented. The 160-char cap bounds it; the bundled strings are all under 95.
