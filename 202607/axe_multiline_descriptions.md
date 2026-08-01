---
tier: epic
title: Multiline AXE descriptions with a collapsible description panel
goal: 'Every AXE lumberjack and chop description authored by the sase-9t epic becomes
  a rich multi-line document with a one-line summary and an explanatory body, the
  shared Rust config authority owns and enforces that grammar, and the ACE AXE tab
  renders the full description in a beautiful accent-gutter panel that a new `d` keymap
  collapses to its summary line and expands again.

  '
phases:
- id: core_description_grammar
  title: Rust core owns the description grammar
  depends_on: []
  size: medium
  description: 'core_description_grammar: teach sase_core''s AXE config authority
    the summary/body description grammar — add a `split_axe_description` binding that
    normalizes and splits one description, add flag-gated shape diagnostics for a
    blank summary, an over-long summary, a missing blank separator line, and an over-long
    description, and release the crate.

    '
- id: sase_description_parts
  title: Plumb summary and body through sase and turn on shape enforcement
  depends_on:
  - core_description_grammar
  size: medium
  description: 'sase_description_parts: bump the sase-core-rs window, wrap the new
    split in the chop facade, carry `description_summary` / `description_body` on
    the AXE config dataclasses and TUI snapshots, add `maxLength` to both description
    schemas so editors switch to a multi-line text area, and flip `require_description_shape`
    on in the sase compose and mutation requests.

    '
- id: axe_tab_panel
  title: Collapsible AXE description panel and the `d` keymap
  depends_on:
  - sase_description_parts
  size: medium
  description: 'axe_tab_panel: rebuild the AXE description banner as a two-state accent-gutter
    panel that word-wraps the body, renders bullet blocks, and caps its height honestly;
    add the `d` toggle keymap with its config default, tab gating, footer, help, and
    palette registration; and cover it with unit and PNG snapshot tests.

    '
- id: cli_surfaces
  title: Summary-first AXE CLI listings
  depends_on:
  - sase_description_parts
  size: small
  description: 'cli_surfaces: keep `sase axe chop list` and `sase axe lumberjack list`
    on one summary line per entry so tables stay scannable, and add a `-v/--verbose`
    full-description rendering to both.

    '
- id: builtin_descriptions
  title: Rewrite every builtin lumberjack and chop description
  depends_on:
  - sase_description_parts
  size: medium
  description: 'builtin_descriptions: rewrite all five builtin lumberjack descriptions
    and all sixteen builtin chop descriptions in default_config.yml as multi-line
    documents by reading each chop script, and update the in-repo YAML examples in
    docs/axe.md and docs/configuration.md to match.

    '
- id: external_descriptions
  title: Rewrite user-owned and plugin descriptions
  depends_on:
  - sase_description_parts
  size: small
  description: 'external_descriptions: rewrite the lumberjack and chop descriptions
    in the chezmoi repo''s sase.yml and sase_athena.yml, and the example descriptions
    in the bugyi-chops README, as multi-line documents that satisfy the new grammar.

    '
- id: docs_and_contract
  title: Document the description contract
  depends_on:
  - axe_tab_panel
  - cli_surfaces
  - builtin_descriptions
  - external_descriptions
  size: small
  description: 'docs_and_contract: document the summary/body grammar, the authoring
    style guide, the new diagnostics, the `d` keymap, and the new config key across
    docs/axe.md, docs/configuration.md, docs/ace.md, and the CHANGELOG.

    '
create_time: 2026-07-26 13:59:52
status: done
bead_id: sase-9w
---

- **PROMPT:** [prompts/202607/axe_multiline_descriptions.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/axe_multiline_descriptions.md)
- **BEAD:** [sase-9w](https://github.com/sase-org/sase--beads/blob/main/pages/sase-9w/README.md)

# Plan: Multiline AXE descriptions with a collapsible description panel

## Goal

The sase-9t epic made `description` required on every AXE lumberjack and chop, but the resulting descriptions are all
single terse lines and the AXE tab shows them in a one-line, width-truncated banner. This epic turns descriptions into
real documentation: a one-line summary plus an explanatory body, owned and enforced by the shared config authority,
rewritten everywhere sase-9t authored one, and rendered on the AXE tab in a panel that a new `d` keymap collapses and
expands.

## Current state

- `description` is a required non-blank string on both entities. Validation lives in `validate_description` in
  `crates/sase_core/src/axe_chop/config.rs` in the `sase-core` repo and is gated behind the `require_descriptions`
  request flag that `src/sase/axe/config_backend.py` sets to `true` in both `compose_axe_config` and
  `plan_axe_entry_edit`. Multi-line strings are already accepted — nothing rejects a newline today.
- `src/sase/config/sase.schema.json` marks `description` required with `minLength: 1` on the lumberjack object, the
  map-form chop (`definitions.axeChop`), and the object-list chop form. No `maxLength`.
- `LumberjackConfig` and `ChopConfig` (`src/sase/axe/_config_types.py`) each carry a plain `description: str`, populated
  by `parse_lumberjacks` in `src/sase/axe/_config_targets.py`. Generated `for_each` instances inherit the base chop's
  description verbatim.
- `ChopSnapshot` and `LumberjackSnapshot` (`src/sase/ace/tui/actions/axe_display/_data.py`) carry `description`, cached
  by the collector so the AXE navigation path stays disk-free.
- `AxeDescriptionBanner` (`src/sase/ace/tui/widgets/axe_description_banner.py`) renders a single `Text` with
  `overflow="ellipsis"`: an accent `▌ ` prefix, the description, and a dim `· <target_key>` chip for generated
  instances. `styles.tcss` pins it to `max-height: 2`, so anything longer than the pane width is silently truncated.
  `AxeDashboard` calls `show_lumberjack` / `show_chop` from `update_lumberjack_overview` / `update_chop_run_display` and
  `hide()` from every other update path.
- `sase axe chop list` prints a `Description` column with `overflow="fold"`; `sase axe lumberjack list` prints
  `description: <text>` under each lumberjack name. Neither has a description-aware verbose mode (`sase axe chop list`
  already accepts `--verbose` for the policy column; `sase axe lumberjack list` takes no flags).
- The AXE entry editor (`src/sase/ace/tui/modals/axe_entry_editor_types.py`, `schema_object_form.py`) puts `description`
  first in the basics group. `_schema_editor_kind` picks the single-line `"string"` editor for it, and only upgrades to
  the multi-line `"text"` editor when the value already contains a newline or the schema declares `maxLength >= 240`.
- On the AXE tab `d` is **not** free in practice: `DEFAULT_BINDINGS` binds `d` to `show_diff` at
  `src/sase/ace/tui/bindings.py:18`, and `AceApp.check_action` never disables it outside the PRs sub-tab. Textual's
  `Screen.active_bindings` keeps the **first enabled** binding for a key, so a later `d` binding can never fire while
  `show_diff` stays enabled. Pressing `d` on the AXE tab today opens a diff for whatever ChangeSpec index happens to be
  selected — a latent bug this epic must fix before it can claim the key.
- Descriptions authored by sase-9t live in four places: `src/sase/default_config.yml` (5 lumberjacks, 16 chops), the
  chezmoi repo's `home/dot_config/sase/sase.yml` (1 lumberjack, 2 chops) and `home/dot_config/sase/sase_athena.yml` (3
  lumberjacks, 4 chops), and the `bugyi-chops` README's two YAML examples (2 lumberjacks, 3 chops). The longest is 91
  characters; none is multi-line.

## Design decisions

These are settled. Implementing agents should not re-litigate them.

### 1. The description grammar is the git commit message shape

A description is one string with this grammar:

```
<summary>
<blank line>
<body…>
```

- Line 1 is the **summary**: non-blank, at most 100 characters, no leading or trailing whitespace.
- If anything follows, line 2 **must be blank**. This is the single rule that makes the split unambiguous.
- Everything from line 3 on is the **body**: free-form prose. Blank lines separate blocks. A block whose lines begin
  with `-`, `*`, or `•` is a bullet list.
- A single-line description is still completely valid and has an empty body. Nothing authored by sase-9t breaks.

This shape was chosen because every engineer already knows it from `git commit`, it degrades to today's behavior for
free, and it needs no new keys, no nested mapping, and no wire-format change — `description` stays one string
everywhere.

### 2. The Rust core owns the grammar

Per the repo's Rust core backend boundary, the summary/body split is shared domain behavior: the AXE tab, both CLI
listings, `sase doctor`, and any future frontend must all agree on where the summary ends. Putting the splitter next to
the validator in `crates/sase_core/src/axe_chop/config.rs` guarantees the two can never disagree about the grammar.

The core exposes one pure binding, `split_axe_description`, following the existing `parse_chop_duration` precedent. It
is called **once per entity at config-parse time**, never on a render or keystroke path, and the result is cached on the
config dataclasses and TUI snapshots. Paragraph reflow and wrapping stay in Python: those are presentation, not domain.

### 3. Shape enforcement is flag-gated on the wire and flipped on in sase

`pyproject.toml` pins `sase-core-rs>=0.10.0,<0.11.0`, so a core release inside a compatible window is picked up by an
unchanged sase install. New unconditional errors would therefore break third-party configs the moment the crate is
published. The shape checks live behind a new `require_description_shape: bool` request field that defaults to `false`
on the wire, exactly mirroring how sase-9t introduced `require_descriptions`. `src/sase/axe/config_backend.py` turns it
on.

The flip can land in phase 2 rather than last, because a survey of every AXE description in this repo, chezmoi, and
bugyi-chops found no description longer than 91 characters and none with a newline. Enforcement is a no-op on the
existing corpus and only constrains the rewrites that follow.

### 4. Multi-line descriptions are authored as YAML literal block scalars; the renderer reflows

Author descriptions with `|-` (literal, strip trailing newline), hand-wrapping source lines to keep the YAML file inside
the repo's 120-column prose width:

```yaml
description: |-
  Complete finished hooks and start stale ones, with zombie detection

  Scans every ChangeSpec matching the axe query, completes hooks whose runner exited, and starts the next
  stale hook when a runner slot is free.

  - Honors max_hook_runners; a full slot table defers work to the next tick rather than queueing.
  - Hooks still running past zombie_timeout_seconds are marked ZOMBIE and stop holding a slot.
```

Literal blocks were chosen over folded (`>-`) blocks because folding is indentation-sensitive in ways that silently
change meaning around bullet lists, and because the stored string then matches what an author sees in the file, which is
what `sase axe chop list --verbose` and the entry editor's text area both display.

Because the author's hard wraps are stored verbatim, the **renderer owns reflow**: consecutive non-blank, non-bullet
lines in a block are joined with single spaces into one paragraph and re-wrapped to the available width. A description
authored at 110 columns therefore still fills a 200-column pane and still reads correctly at 60.

### 5. The AXE banner becomes a two-state accent-gutter panel

`AxeDescriptionBanner` keeps its position between `AxeStatusSection` and the output `VerticalScroll`, keeps its `▌`
accent idiom, and gains a second state:

Collapsed — visually identical to today plus a disclosure hint:

```
▌ Complete finished hooks and start stale ones, with zombie detection            ▸ d
```

Expanded — the accent gutter extends down the whole block, binding it visually and separating it unmistakably from the
output pane below:

```
▌ Complete finished hooks and start stale ones, with zombie detection            ▾ d
▌
▌ Scans every ChangeSpec matching the axe query, completes hooks whose runner exited, and
▌ starts the next stale hook when a runner slot is free.
▌
▌ • Honors max_hook_runners; a full slot table defers work to the next tick rather than
▌   queueing.
▌ • Hooks still running past zombie_timeout_seconds are marked ZOMBIE and stop holding a slot.
```

The gutter reads as a blockquote, costs no extra rows (unlike a border), scales to any width, and reuses glyphs already
proven to render in the pinned visual-snapshot font (`▌`, `▸`, `▾`, `•`, `…`, `·`).

Rejected alternatives: a bordered panel (costs two rows and two columns of the output pane for no added clarity given
the gutter); a scrollable banner (a scrollbar with no keyboard affordance is a broken promise); a modal description
viewer (the whole point is that the description is visible while you read chop output).

### 6. `d` toggles for the session; the startup default comes from config

Toggling must not touch disk — the TUI performance rules forbid synchronous I/O on a keystroke path, and a durable
per-toggle write would need the whole coalesced-save-plus-teardown-flush apparatus the Admin Center uses. Instead:

- `ace.axe_description_expanded` (bool, default `true`) sets the state each `sase ace` session starts in.
- `d` flips an in-memory app reactive for the rest of the session and repaints the banner from cached snapshot state —
  no reload, no disk read, no config write.

This is intuitive (set your standing preference in config, deviate per session with one key), and it keeps the keystroke
path to a boolean flip plus one widget refresh.

### 7. Height is capped from the pane height, and overflow is stated honestly

An expanded panel must never crowd out the chop output it exists to explain. `AxeDashboard` computes a line budget from
its own height at update time — `max(3, min(16, floor(height * 0.45)))`, falling back to 10 when the height is not yet
known — and passes it to the banner. If the rendered block exceeds the budget, the last row is replaced with a dim
`… +N more · e` row. Nothing is ever silently dropped, and `e` (the AXE entry editor, whose first basics field is the
description in a multi-line text area) is the full-text escape hatch.

### 8. `show_diff` is scoped to the PRs sub-tab so `d` can be claimed

`check_action` gains a guard disabling `show_diff` outside `ARTIFACTS_TAB`. This is required for the new binding to ever
fire, and it independently fixes the existing bug where `d` on the AXE and Agents tabs opens a diff for an unrelated
ChangeSpec. `show_diff` is already `CL_ONLY` in the command palette metadata, so this only aligns the keymap with the
palette's existing contract.

### 9. The multi-line editor comes from the schema, not from new widget code

Adding `"maxLength": 2000` to all three description schema sites makes `_schema_editor_kind` and Config Center's
`editor_kind_for` both select the existing `"text"` (multi-line `TextArea`) editor, because both already switch at
`maxLength >= 240`. The same 2000-character bound is enforced in the Rust validator so the schema and the config
authority agree. No new editor widget is written.

### 10. CLI listings are summary-first

A multi-line cell would destroy the `sase axe chop list` table. Both listings print the **summary only** by default and
render the full text under `-v/--verbose`. `sase axe chop list` already has that flag; `sase axe lumberjack list`, which
takes no flags today, gains it with the short alias the repo's CLI rules require.

### 11. Cross-cutting rule

From phase 2 onward, **every AXE description authored anywhere — configs, fixtures, docs examples, test constructions —
must satisfy the grammar in decision 1.** Phase 2 turns enforcement on; a fixture that violates it fails the suite.

## Authoring style guide

Use this for every description rewritten or created in this epic.

**Summary (line 1)**

- One line, at most 100 characters, target 80. Sentence case, no trailing period.
- Present tense, active voice, describing what the entity _does_, not what it is.
- Must stand alone: the collapsed banner, both CLI tables, and the entry editor's one-line preview show only this.
- Keep the existing sase-9t summary text when it is already good; this epic adds bodies, it does not churn good prose.

**Body**

- One to three short paragraphs and/or one bullet list. Aim for six to ten rendered lines; never exceed what a reader
  will actually read.
- Answer, in this order, only what is true and non-obvious: what it actually does, when it fires, what state it reads or
  mutates, and the one thing an operator most needs to know (a failure mode, a safety property, a cost, a limit).
- Name the config knobs that matter for this entity when they are set (`interval`, `chop_timeout`, `run_every`,
  `trigger`, `inhibit_if`, `for_each`, `env`).
- Do not restate the summary, do not narrate the implementation line by line, and do not document SASE concepts that
  belong in `docs/`.

**Lumberjack bodies additionally**

- State the cadence in words and why that cadence is right for this lane.
- State what belongs in the lane and what deliberately does not, so a reader knows where to add a new chop.

**Mechanics**

- YAML: `description: |-`, source lines hand-wrapped so no line exceeds 120 columns including indentation.
- Bullets start with `- ` at the block's base indentation; continuation lines indent two further spaces.
- No trailing whitespace, no tabs, no blank line at the end of the block.

## Phase 1 — Rust core owns the description grammar

Repo: `sase-core`. Open it with the `/sase_repo` skill and use only the path it prints.

### Changes

1. `crates/sase_core/src/axe_chop/config.rs`
   - Add two constants: `AXE_DESCRIPTION_SUMMARY_MAX: usize = 100` and `AXE_DESCRIPTION_MAX_CHARS: usize = 2000`
     (character counts, not bytes, so multi-byte prose is not penalized).
   - Add a pure `split_axe_description(text: &str) -> (String, String)` helper:
     - Normalize `\r\n` and lone `\r` to `\n`.
     - Trim trailing whitespace from every line.
     - `summary` = line 1, trimmed.
     - `body` = lines 3 onward, with leading and trailing blank lines removed, rejoined with `\n`. A one- or two-line
       input yields an empty body.
     - This helper is total: it never fails, so callers that skip validation still get a sane split.
   - Extend `validate_description` with shape diagnostics emitted only when the request's new
     `require_description_shape` flag is true. All four use `severity: "error"` and the field's existing config path: |
     code | condition | message | | --- | --- | --- | | `description_summary_blank` | line 1 is empty or whitespace-only
     | `description must start with a non-blank summary line` | | `description_summary_too_long` | summary length > 100
     chars | `description summary line must be at most 100 characters (found <n>)` | |
     `description_body_separator_required` | a line 2 exists and is not blank |
     `description must leave line 2 blank to separate the summary from the body` | | `description_too_long` | total
     length > 2000 chars | `description must be at most 2000 characters (found <n>)` |
     - Emit at most the first applicable code per description so a single typo does not produce four diagnostics; check
       in the table's order.
     - The existing blank-description and `required_missing` behavior is unchanged.

2. `crates/sase_core/src/axe_chop/wire.rs`
   - Add `require_description_shape: bool` with `#[serde(default)]` to `AxeConfigValidationRequestWire`, and thread it
     through `validate_axe_config` into `validate_lumberjacks` and `validate_chop_config` alongside
     `require_descriptions`.

3. `crates/sase_core/src/config/axe.rs`
   - Add `require_description_shape: bool` with `#[serde(default)]` to `AxeConfigComposeRequestWire` and
     `AxeEntryMutationRequestWire`.
   - Mirror the existing `require_descriptions` threading exactly: `compose_values` passes `false` for the **per-layer**
     validation requests and the request-supplied value for the **merged** request; `plan_axe_entry_mutation` propagates
     the flag into both of its internal `compose_axe_config` calls.

4. `crates/sase_core_py/src/lib.rs`
   - Expose `split_axe_description(text: &str) -> (String, String)` as a module-level binding, following the
     `parse_chop_duration` pattern (plain in, plain out, no request dict).
   - No signature change is needed for the validation entry points; serde fills the new flag's default.

5. Tests
   - `crates/sase_core/src/axe_chop/tests.rs`: split cases for a single-line description, a summary plus one paragraph,
     a summary plus a bullet block, CRLF input, trailing-whitespace input, and an input with leading and trailing blank
     body lines. Shape cases for each of the four diagnostics, plus proof that all four are silent when
     `require_description_shape` is false and that a valid single-line description passes when it is true.
   - `crates/sase_core/tests/config_parity.rs`: a compose case proving a sparse overlay that only sets `interval`
     produces no shape diagnostic, and an entry-mutation case proving the flag reaches `plan_axe_entry_mutation`'s
     diagnostics.
   - A binding-level test asserting `split_axe_description` round-trips through Python.

6. Bump the workspace version in `Cargo.toml` and release the crate so sase can depend on it.

### Acceptance

- `cargo test` passes in `sase-core`.
- A config whose descriptions are all single-line validates identically with the flag off and on.
- A description with a non-blank line 2, a 120-character summary, or a leading blank line produces exactly one precise
  diagnostic each when the flag is on, and none when it is off.

## Phase 2 — Plumb summary and body through sase and turn on shape enforcement

Repo: `sase`. Run `just install` before `just check` — ephemeral workspaces drift.

### Changes

1. `pyproject.toml`: bump the `sase-core-rs` window to require the phase 1 release.

2. `src/sase/core/axe_chop_facade.py`
   - Add `split_axe_description(text: str) -> tuple[str, str]` wrapping the new binding.
   - Add a `require_description_shape: bool = False` keyword to `validate_axe_config` and include it in the payload.

3. `src/sase/axe/_config_types.py`: add `description_summary: str = ""` and `description_body: str = ""` to both
   `LumberjackConfig` and `ChopConfig`, immediately after `description`. They are derived, never authored.

4. `src/sase/axe/_config_targets.py`: call `split_axe_description` once per entity in `parse_lumberjacks` and in the
   generated-instance construction path (`_config_targets.py:325` and `:475`) and populate the two new fields. Leave
   `_validate_target_config` on the non-requiring, non-shape-checking scope — it validates a synthesized partial config.

5. `src/sase/axe/config_backend.py`: add `"require_description_shape": True` next to the existing
   `"require_descriptions": True` in both `compose_axe_config` and `plan_axe_entry_edit` request dicts.

6. `src/sase/config/sase.schema.json`: add `"maxLength": 2000` to all three authored description sites —
   `definitions.axeChop.properties.description`,
   `properties.axe.properties.lumberjacks.additionalProperties.properties.description`, and the inline object-list form
   at `…lumberjacks.additionalProperties.properties.chops.items.oneOf[1].properties.description` — and extend each
   site's own schema `description` text to state the summary/body grammar so the Config Center and entry editor explain
   it inline.

7. `src/sase/ace/tui/actions/axe_display/_data.py`: add `description_summary: str = ""` and `description_body: str = ""`
   to `ChopSnapshot` and `LumberjackSnapshot`, populated in the collector from the parsed config. This keeps the AXE
   navigation path cache-only — no split is ever computed on a render or keystroke path.

8. `src/sase/ace/tui/actions/axe_display/_render.py`: include the two new fields in the cache-miss fallback snapshot
   construction.

9. `src/sase/axe/cli.py`: the `_oneshot` fallback builds
   `ChopConfig(name=chop_name, description=f"Manual one-shot run of {chop_name}")`. Populate `description_summary` from
   the same text and leave the body empty.

10. Tests: round-trip coverage that `parse_lumberjacks` splits a multi-line lumberjack and chop description into summary
    and body; that a generated `for_each` instance inherits both parts; that the collector populates both snapshot
    fields; and that `compose_axe_config` rejects a description with a non-blank second line with the
    `description_body_separator_required` diagnostic surfaced through `AxeConfigError`.

### Acceptance

- `just check` passes.
- A multi-line description in any config layer composes cleanly and its parts are visible on the parsed config objects.
- A malformed multi-line description fails `sase axe chop list` and `sase doctor` with the precise diagnostic and its
  config path.

## Phase 3 — Collapsible AXE description panel and the `d` keymap

Repo: `sase`. Read `sase/memory/tui_perf.md` with the `/sase_memory_read` skill before starting, and follow
`src/sase/ace/CLAUDE.md` for the footer and help-popup rules.

### Changes

1. `src/sase/ace/tui/widgets/axe_description_banner.py` — rebuild around a Rich renderable so Textual's auto-height
   measurement works.
   - Add a private `_DescriptionBlock` renderable implementing `__rich_console__(self, console, options)`. It receives
     `summary`, `body`, `accent_style`, `target_key`, `expanded`, `max_lines`, and wraps at `options.max_width` so
     `Static.get_content_height` measures the same rows it later paints.
   - Rendering algorithm, with `GUTTER = "▌ "` (2 columns) and `content_width = max(1, options.max_width - 2)`:
     1. Summary row(s). The generated-instance chip `  · <target_key>` is appended to the summary text before layout.
        Collapsed: exactly one row, ellipsis-truncated. Expanded: word-wrapped over as many rows as needed.
     2. Disclosure hint. When the body is non-empty and `content_width` leaves at least four spare columns, right-align
        `▸ d` (collapsed) or `▾ d` (expanded) on the summary's first row. Drop it silently when the width is too tight.
     3. Stop here when collapsed or when the body is empty.
     4. Body blocks, split on blank lines, separated by one blank gutter row:
        - A block whose first line starts with `-`, `*`, or `•` is a **bullet block**: each marker line starts a bullet,
          non-marker lines join the current bullet, and each bullet renders as `• ` plus text wrapped with a two-column
          hanging indent.
        - Any other block is a **paragraph**: join its lines with single spaces and word-wrap to `content_width`.
     5. Cap. If the row count exceeds `max_lines`, keep `max_lines - 1` rows and append a dim `… +N more · e` row where
        `N` is the number of rows dropped.
   - Every row is prefixed with the gutter: the summary row uses the entity accent at full strength, body rows use the
     same hue dimmed.
   - Styles: keep `_LUMBERJACK_ACCENT = "bold #FFD700"`, `_CHOP_ACCENT = "#D7AF87"`, `_TARGET_STYLE = "dim #B87333"`,
     and the summary's `italic #D7D7AF`. Add a non-italic `#AFAF87` body style so long prose stays legible, a dim accent
     style for bullet markers and the hint, and `dim italic` for the overflow row. Refine the exact values against the
     PNG snapshots; keep the collapsed state pixel-identical to today apart from the hint.
   - Replace the `description` parameters of `show_lumberjack` / `show_chop` with `summary` and `body`, cache the last
     shown state on the widget, and add `set_expanded(expanded: bool)` plus `set_max_lines(max_lines: int)` that
     re-render from that cache with no data reload.

2. `src/sase/ace/tui/widgets/axe_dashboard.py`
   - Add `_description_max_lines()` returning `max(3, min(16, int(self.size.height * 0.45)))`, falling back to `10` when
     the height is `0` or unknown.
   - Pass the app's current expanded state and that budget into the banner from `update_lumberjack_overview` and
     `update_chop_run_display`, using the snapshots' `description_summary` / `description_body`.
   - Add `refresh_description_banner(expanded: bool)` that forwards to `set_expanded` for the `d` action.

3. `src/sase/ace/tui/styles.tcss`: change `#axe-description-banner` to `height: auto; max-height: 50%;` (keeping
   `padding: 0 1` and the surface background). The Python line cap is authoritative; the CSS clamp is a guard that must
   never bind.

4. Keymap registration — `d` as `toggle_axe_description`:
   - `src/sase/default_config.yml`: `toggle_axe_description: "d"` under `ace.keymaps.app`, placed with the other AXE
     keys, with a comment noting it is AXE-tab only.
   - `src/sase/ace/tui/keymaps/app_keymaps.py`: add the field to `AppKeymaps`. The loader derives its default from
     `default_config.yml` and fails closed when a field has no default there, so both edits are required.
   - `src/sase/ace/tui/keymaps/metadata.py`: add `("toggle_axe_description", "Toggle AXE Description", False)` to
     `_BINDING_META`.
   - `src/sase/ace/tui/bindings.py`: add the fallback `Binding("d", "toggle_axe_description", ...)`.
   - No schema change is needed for the key itself: `ace.keymaps.app` is `additionalProperties: {"type": "string"}`.

5. `src/sase/ace/tui/app.py` — `check_action`:
   - `if action == "toggle_axe_description": return self.current_tab == "axe"`.
   - `if action == "show_diff" and self.current_tab != ARTIFACTS_TAB: return False`. Textual keeps the **first enabled**
     binding per key, so without this guard the new action can never fire. Add a regression test asserting the AXE tab
     resolves `d` to `toggle_axe_description` and the PRs sub-tab still resolves it to `show_diff`.

6. Session state and the action:
   - `src/sase/default_config.yml`: add `axe_description_expanded: true` under `ace:`, with a comment that `d` toggles
     it for the session.
   - `src/sase/config/sase.schema.json`: add the boolean with a default of `true`.
   - Load it into the config model wherever the other `ace.*` scalars are read, and initialize an
     `axe_description_expanded` app reactive from it in `src/sase/ace/tui/actions/_state_init.py`.
   - `src/sase/ace/tui/actions/axe.py`: add `action_toggle_axe_description()` that flips the reactive, calls the
     dashboard's `refresh_description_banner`, and refreshes the footer. It must do no I/O and no data reload.

7. Footer, help, and palette:
   - `src/sase/ace/tui/widgets/_keybinding_bindings.py`: give `_compute_axe_bindings` a `description_expanded: bool`
     parameter and, when `config_row_selected` is true, append `(self._kd("toggle_axe_description"), "collapse desc")`
     when expanded or `"expand desc"` when collapsed. Thread the app state through the caller.
   - `src/sase/ace/tui/modals/help_modal/axe_bindings.py`: add the binding to the AXE tab's section list next to the
     other config-row actions.
   - `src/sase/ace/tui/commands/_app_metadata.py`: register the command AXE-tab-only, mirroring `add_axe_item`'s tab
     scope, with aliases such as `("description", "expand description", "collapse description")`.
   - `src/sase/ace/tui/commands/availability.py`: in `_axe_available`, return `_is_lumberjack(item) or _is_chop(item)`
     for `app.toggle_axe_description`, matching `app.edit_spec`, and extend the module docstring's AXE bullet.

8. Tests
   - `tests/ace/tui/test_axe_description_banner.py`: collapsed one-line rendering; expanded paragraph reflow (author
     hard wraps joined and re-wrapped to a different width); bullet block rendering with hanging indent; the blank
     separator row between blocks; the disclosure hint appearing only with a body and disappearing at narrow widths; the
     overflow row with the correct dropped-row count; empty-body descriptions rendering identically in both states; and
     the generated-instance target chip surviving both states.
   - `tests/ace/tui/test_axe_navigation.py`: `d` toggles both directions on a lumberjack row and a chop row, the state
     persists across selection changes within the session, the config default seeds the initial state, and the action is
     a no-op on other tabs.
   - Footer, help-modal, and command-palette coverage for the new entry.
   - PNG snapshots (`tests/ace/tui/visual/test_ace_png_snapshots_axe.py`): re-record `axe_lumberjack_description_120x40`
     and `axe_chop_description_120x40` with multi-line fixtures in the expanded state, and add
     `axe_chop_description_collapsed_120x40` and `axe_description_overflow_120x40`. Refresh any snapshot whose layout
     shifts because the banner grew. Use `--sase-update-visual-snapshots` only to accept these intentional changes.

### Acceptance

- `just check` and `just test-visual` pass.
- On the AXE tab, selecting a lumberjack or chop shows its full description; `d` collapses it to the summary line and
  `d` again restores it; the footer label tracks the state.
- `d` on the PRs sub-tab still opens a diff; `d` on the Agents tab no longer opens an unrelated ChangeSpec's diff.
- The expanded panel never consumes more than roughly 45% of the dashboard height, and any truncation is stated in the
  final row.

## Phase 4 — Summary-first AXE CLI listings

Repo: `sase`. Read `sase/memory/cli_rules.md` with the `/sase_memory_read` skill before touching the parsers.

### Changes

1. `src/sase/axe/chop_render.py`: render `chop.description_summary or "-"` in the `Description` column so the table
   stays one line per chop. When `verbose` is set, render the summary followed by a blank line and the body in a dim
   style within the same cell.

2. `src/sase/axe/cli.py` (`handle_axe_lumberjack_list`): print `description:` with the summary only. When verbose, print
   the body beneath it, indented to align under the summary and dimmed.

3. `src/sase/main/parser_ace.py`: add `-v/--verbose` to the `axe lumberjack list` sub-parser, which currently takes no
   flags. `axe chop list` already gets `-v/--verbose` from `_add_chop_diagnostic_flags`, so reuse that flag rather than
   adding another. Keep options alphabetically sorted and the help text scannable, per the CLI rules, and update the
   chop-list flag's help text to mention that it also expands descriptions.

4. Tests: `tests/test_axe_cli.py` and `tests/test_axe_chop_inventory.py` — a multi-line description shows only its
   summary in the default listing for both commands, shows summary and body under `-v`, and a single-line description
   renders identically in both modes.

### Acceptance

- `just check` passes.
- `sase axe chop list` stays one row per chop with multi-line descriptions configured.
- `sase axe lumberjack list -v` and `sase axe chop list -v` print the full text.

## Phase 5 — Rewrite every builtin lumberjack and chop description

Repo: `sase`.

### Changes

Rewrite all 21 descriptions in the `axe:` block of `src/sase/default_config.yml` as multi-line documents following the
authoring style guide. Keep each existing summary when it is already accurate; the work is adding real bodies.

Read the implementation before writing each body — do not paraphrase the summary. The chop scripts are
`src/sase/scripts/sase_chop_*.py`, registered as console scripts in `pyproject.toml`:

- `hooks` lane (`interval: 5`, `chop_timeout: 90s`): `hook_checks`, `mentor_checks`, `workflow_checks`,
  `pending_checks_poll`, `comment_zombie_checks`, `suffix_transforms`, `orphan_cleanup`, `stale_running_cleanup`.
- `waits` lane (`interval: 10`): `bead_claim_checks`, `bead_store_refresh` (`run_every: 30s`, `timeout: 2m`),
  `wait_checks`.
- `checks` lane (`interval: 300`): `pr_submitted_checks`, `stale_running_cleanup`.
- `comments` lane (`interval: 60`): `comment_checks`.
- `housekeeping` lane (`interval: 3600`): `error_digest`, `managed_tmp_reap`.

For each lumberjack, the body must explain the cadence and what does and does not belong in the lane. For each chop, it
must explain what the chop actually reads and mutates, when it fires, and the one operational fact that matters most —
for example that `stale_running_cleanup` appears in two lanes with different roles, or that `bead_store_refresh` runs on
its own `run_every` because integrating the canonical bead store is too slow for a 10-second tick.

Also update the illustrative YAML in `docs/axe.md` (the "Lumberjack Configuration" example) and `docs/configuration.md`
(the `axe.lumberjacks` example) so every in-repo example demonstrates the multi-line form.

### Acceptance

- `just check` passes and no shape diagnostic is produced by `sase axe chop list` or `sase doctor`.
- Every summary is at most 100 characters and every YAML line is at most 120 columns.
- Selecting each builtin lumberjack and chop on the AXE tab shows a body that teaches an operator something the summary
  does not.

## Phase 6 — Rewrite user-owned and plugin descriptions

Repos: `chezmoi` and `gh:bbugyi200/bugyi-chops`. Open each with the `/sase_repo` skill and use only the printed paths.

### Changes

1. chezmoi `home/dot_config/sase/sase.yml`: the `telegram` lumberjack and its `tg_inbound` / `tg_outbound` chops.
2. chezmoi `home/dot_config/sase/sase_athena.yml`: the `run_every`, `code_quality`, and `refresh_docs` lumberjacks and
   the `toobig_split`, `recent_bug_audit`, `recent_improvement_audit`, and `refresh_docs` chops.
3. bugyi-chops `README.md`: the `maintenance` and `audits` example lumberjacks and the `toobig_split`,
   `recent_bug_audit`, and `recent_improvement_audit` example chops.

Follow the same style guide. These entities are configured with `trigger`, `inhibit_if`, `for_each`, and `run_every`, so
their bodies should say what the trigger threshold and checkpoint policy mean in practice and what the inhibit guard
prevents. The three chezmoi `sase_athena.yml` lumberjacks all currently read as near-identical "Minute lane that…"
summaries; differentiate them.

### Acceptance

- `sase ace` on the user's machine loads both chezmoi configs with no diagnostic.
- The bugyi-chops README examples remain copy-pasteable into a working config.

## Phase 7 — Document the description contract

Repo: `sase`.

### Changes

1. `docs/axe.md`: extend the description row of the chop field table and the lumberjack section with the grammar, and
   add a short "Description grammar" subsection covering the summary rule, the blank separator line, the body's block
   and bullet handling, the 100/2000-character limits, the four diagnostic codes, and the authoring style guide.
2. `docs/configuration.md`: document `ace.axe_description_expanded` and the `toggle_axe_description` keymap alongside
   the other `ace.*` settings, and update the `axe.lumberjacks` reference rows.
3. `docs/ace.md`: replace the three-line AXE banner paragraph (currently around line 1281) with the two-state panel
   behavior — the gutter layout, the `d` toggle, the config-seeded default, the height cap, and the overflow row.
4. `CHANGELOG.md`: add a `BREAKING CHANGES` entry under the unreleased heading stating that AXE descriptions must now
   place a blank line after the summary, that summaries are capped at 100 characters and descriptions at 2000, and
   naming the diagnostics that report violations.
5. Sweep for stale prose: any doc that describes the AXE description as a single line.

### Acceptance

- `just check` passes, including markdown formatting at `--print-width=120`.
- The docs describe exactly the shipped grammar, keymap, config key, and diagnostics.

## Risks and mitigations

- **A third-party config with a long or oddly shaped description breaks on upgrade.** Mitigated by decision 3: the shape
  checks are off by default on the wire, so only sase installs that opt in enforce them, and the CHANGELOG entry names
  the exact diagnostics and fixes.
- **The expanded panel crowds the output pane on short terminals.** Mitigated by decision 7's height budget, which
  floors at three rows and always leaves the majority of the pane to output, and by the config key for users who prefer
  to start collapsed.
- **PNG snapshot churn.** The banner is present in most AXE goldens, so several will shift. Re-record deliberately and
  inspect `.pytest_cache/sase-visual/` diffs before accepting.
- **Glyph availability in the pinned snapshot font.** Only glyphs already used elsewhere in the TUI (`▌`, `▸`, `▾`, `•`,
  `…`, `·`) are introduced, so no new tofu boxes appear in the goldens.
