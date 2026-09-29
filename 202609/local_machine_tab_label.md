---
tier: tale
title: Name the local machine tab after this machine with a home indicator
goal:
  In machine-tab mode the Agents sub-tab strip labels this machine's tab with its
  configured machine name and a teal ⌂ home indicator instead of `⌨ local`,
  consistently on every surface that names that tab.
size: medium
proposed_by: bbugyi200.apollo.39
create_time: 2026-09-29 18:56:38
status: wip
---

# Plan: Name the local machine tab after this machine, marked with a home glyph

## Goal

In machine-tab mode, the Agents tab's sub-tab strip currently labels the tab for this
machine's own agents `⌨ local`. Remote machines get `⌨ <alias>`. Replace `local` with
this machine's configured name (`id.machine_name`, e.g. `athena`). Also give the tab a
distinct, attractive "this machine" indicator, so at any width you can tell the machine
you are sitting at from the remote ones.

## Design

### The look

```
before:  ▐ ⌨ local 179+ ▌ │ ⌨ apollo 79+ • ┊ sase 12
after:   ▐ ⌂ athena 179+ ▌ │ ⌨ apollo 79+ • ┊ sase 12
```

- **Indicator = the home glyph `⌂` (U+2302) in place of `⌨`.** Remote machines keep
  `⌨`. The tab for the machine you are at is "home".
- **Home accent = teal `#00D7AF`.** The Admin Center Machines pane already uses this
  color for its local row ("local" state/health), so "teal = this machine" becomes one
  color language across the TUI. Remote machine tabs stay cyan `#5FD7FF`. Tabs with
  project accents use the muted, luminance-matched project palette. So the home tab is
  easy to tell apart without being loud.
  - Inactive chip: `⌂` in bold teal, name in teal, count in the neutral `#AFAFAF`, and
    the usual `S`/`F`/`U` tokens and arrival dot.
  - Active chip: a teal pill (`bold #1C1C1C on #00D7AF`) instead of today's gray `main`
    pill.
- **Every width tier keeps the indicator.**
  - full: `⌂ athena 179+`
  - compact: `⌂ athena`
  - micro: `⌂`

  Today, in the micro tier, local and remote tabs both shrink to the same `⌨`. After
  this change, `⌂ │ ⌨ │ ⌨` stays readable.

- **Tooltip** (hover): `athena · this machine · 179 agents · S2 …`.
- **Tab picker rows**: `⌂ athena  179  this machine`. The trailing part is dim.
  Searching `athena` finds the tab.
- **Unchanged outside machine mode.** When machine tabs are off, the default tab stays
  the gray, glyph-less `main`. That tab means "untabbed agents", not "this machine".
- **Fallback.** If no machine name is configured, the tab is `⌂ local`, and the glyph
  and color still mark it as this machine.

Why `⌂`:

- It is the most widely understood "mine / home" mark.
- It is one cell wide and present in both bundled screenshot fonts (Fira Code and DejaVu
  Sans), so goldens and terminals agree.
- In sase, `⌂` already means "your default/home X": the `@default` tribe and the
  command-line project chip. This tab is literally the `Default` tab key, so the
  vocabulary stays consistent.

Rejected alternatives:

- `◉`/`◎`: read as "selected radio", which clashes with the active pill, and `◉` is the
  review tribe icon.
- `•`/`●`: collide with the arrival dot.
- `*`: reads as "active tab" (the tmux convention).
- `🏠`: a double-width color emoji that clashes with the monochrome strip.

The glyph and the color are each a single constant in Rust and in Python, so swapping
them later is a one-line change in each place.

### The behavior rules (single-sourced in sase-core)

The catalog labels ("⌨ alias", "main", the `local·remote` disambiguation) are already
built in `sase_core` by `build_agent_tab_catalog`. The new label therefore goes there
too, so every frontend agrees:

- New optional catalog option `local_alias` (this machine's name).
- In machine mode, the default tab's label is `⌂ <local_alias>`, or `⌂ local` when the
  alias is missing or blank. Outside machine mode it stays `main`.
- A remote alias that equals the effective local name (case-insensitive) gets
  `⌨ <alias>·remote`. This extends today's rule, and a remote literally named `local`
  keeps rendering `⌨ local·remote`. Both resolved and unresolved machine keys follow
  the rule.
- The `Default` tab key does not change. Selection, folds, sticky panels, and the
  persisted active tab keep working across the relabel and across machine renames.

The Python side:

- Resolves the name once per config token.
- Passes the name to the core.
- Renders the glyph and color.
- Updates every text surface that names the local tab.

With that, bulk confirmations, "tab is gone" toasts, launch hints, and Machines-pane
toasts all say `⌂ athena` consistently.

## Implementation

### A. sase-core (linked repo)

Open it with `sase repo open sase-core -r "<why>"`, read its `AGENTS.md`, and work only
in the printed path.

1. `crates/sase_core/src/agent_tab.rs`
   - Add `local_alias: Option<String>` to `AgentTabCatalogOptionsWire` with
     `#[serde(default, skip_serializing_if = "Option::is_none")]`. The struct does not
     deny unknown fields, so older callers keep working.
   - Add named consts for the glyphs (e.g. `MACHINE_TAB_GLYPH = "⌨"`,
     `LOCAL_MACHINE_TAB_GLYPH = "⌂"`) and use them in the label helpers.
   - Change `default_tab_label(machine_mode, local_name)`:
     - machine mode → `format!("{LOCAL_MACHINE_TAB_GLYPH} {local_name}")`, where
       `local_name` is the trimmed, non-empty `local_alias` or `"local"`;
     - otherwise → `main`.
   - Change `machine_tab_label(alias, local_name)` → `⌨ {alias}·remote` when
     `alias == "local"` or the lowercased `alias` equals the lowercased `local_name`.
     Otherwise it returns `⌨ {alias}`. Resolve `local_name` once per
     `build_agent_tab_catalog` call and use it for the configured, seen, and unresolved
     machine labels.
   - Update the doc comments that mention `⌨ local`: the `AgentTabKeyWire` comment and
     the `build_agent_tab_catalog` comment.
   - Tests:
     - update existing expectations (`"⌨ local"` → `"⌂ local"`);
     - `local_alias: Some("athena")` → default label `⌂ athena`;
     - blank or whitespace alias → `⌂ local`;
     - outside machine mode, `local_alias` is ignored → `main`;
     - a remote alias `Athena` (resolved and unresolved) → `⌨ Athena·remote`;
     - a remote alias `local` with `local_alias` set to athena still →
       `⌨ local·remote`;
     - options JSON without `local_alias` still deserializes.
2. `crates/sase_core_py/src/agent_tab/tests.rs`: add a binding test that passes
   `"local_alias": "athena"` in the options dict and asserts the default entry's label.
3. Run the sase-core check recipe that its `AGENTS.md` names. Leave the CHANGELOG alone;
   release tooling generates it. The host commits sase-core first and moves
   `sase-core-revision.txt` automatically. Do not bump the pin by hand.

### B. sase

After step A, rebuild the local extension (`just rust-install`) so the Python tests use
the new core.

1. **Name source.** In `src/sase/config/_owner.py`, add `get_local_machine_name()`. It
   returns the complete owner's `machine_name`, else the valid machine-name selector
   (`snapshot.selector`), else `None`. Both are read from the token-cached owner
   snapshot. Export it from `sase/config/__init__.py`. Refactor `local_machine_label()`
   in `src/sase/ace/tui/modals/machines_pane_rendering.py` to
   `get_local_machine_name() or platform.node() or "local"`, so the Machines pane
   behaves as before and both surfaces share one resolver.
2. **Core wrapper.** `build_agent_tab_catalog` in `src/sase/core/agent_tab.py` gains the
   keyword `local_alias: str | None = None`. Send it in options only when it is
   non-blank.
3. **View config.** In `src/sase/ace/tui/agent_tabs_settings.py`:
   - Add `local_machine_name: str = ""` to `AgentTabsViewConfig`.
   - Resolve it in `_agent_tabs_view_config_for_token` with `get_local_machine_name()`.
     Failures fall back to `""`.
   - Include it in `token` so the tab-index memo rebuilds when the name changes.
     `current_config_token()` already stats the `~/.sase/machine_name` selector and the
     overlays, so no new I/O reaches render paths.
   - Add a tiny `local_machine_tab_name(view)` helper that returns
     `view.local_machine_name or "local"`.
4. **Index.** `build_agent_tab_index` in `src/sase/ace/tui/models/agent_tab_index.py`
   passes `local_alias=view_config.local_machine_name` to the catalog.
5. **Strip model.** In `src/sase/ace/tui/widgets/_agent_tab_strip_model.py`:
   - Add `LOCAL_MACHINE_GLYPH = "⌂"` and `LOCAL_MACHINE_STYLE = "#00D7AF"`.
   - Add `AgentTabDescriptor.is_local_machine: bool = False`.
   - In `agent_tab_label_style`, the local tab returns `LOCAL_MACHINE_STYLE`. The
     non-machine default keeps `MAIN_LABEL_STYLE`.
   - Add `agent_tab_glyph_style(descriptor)`. The local tab gets
     `bold {LOCAL_MACHINE_STYLE}`; every other glyph keeps `MACHINE_GLYPH_STYLE`, as
     today.
   - Add `local_machine_tab_label(name)`, which returns `"⌂ <name or 'local'>"`.
   - Re-export the new public names through `widgets/agent_tab_strip.py`.
6. **Descriptors.** In `src/sase/ace/tui/models/agent_tab_descriptors.py`:
   - `_split_machine_label` also splits a leading `⌂ `.
   - The default key in machine mode gets glyph `LOCAL_MACHINE_GLYPH`, accent
     `LOCAL_MACHINE_STYLE`, `is_local_machine=True`, and `machine_alias=label`.
   - Add `is_local_machine` to `descriptor_signature`.
   - In `machine_off_tab_extras`, the default tab's word becomes the bare local label
     (`+3 athena agents on other tabs: …`) instead of `local`. Strip either glyph prefix
     through one helper instead of the repeated `removeprefix("⌨ ")`.
7. **Strip widget.** In `src/sase/ace/tui/widgets/_agent_tab_strip_strip.py`:
   - Glyph fragments use `agent_tab_glyph_style`.
   - The active pill accent is `LOCAL_MACHINE_STYLE` for the local tab. It stays
     `MAIN_LABEL_STYLE` only for the non-machine default.
   - `_agent_tab_tooltip` inserts `this machine` right after the label for the local
     tab.
   - The kind-divider logic already treats `is_default` as a machine chip, so keep it.
8. **Catalog helpers.** In `src/sase/ace/tui/actions/agents/_agent_tabs_catalog.py`:
   - `_fallback_label` for the default key returns
     `local_machine_tab_label(local_machine_tab_name(view))` in machine mode and `main`
     otherwise.
   - `agent_tab_health_for_owner` skips the default key entirely. The local tab has no
     remote feed, so a host issue or diagnostic whose alias matches the local name can
     never paint it amber or red. Today it matches on the bare label.
9. **Gone-tab toast.** In `src/sase/ace/tui/actions/agents/_agent_tabs_lifecycle.py`
   (around the `"⌨ local·remote"` fallback), the reconstructed remote label follows the
   core rule (alias equal to `local` or to the local name → `·remote`). The
   default-label fallback uses `local_machine_tab_label(...)` in machine mode.
10. **Launch view.** `active_machine_tab_alias` in
    `src/sase/ace/tui/agent_tabs_launch_view.py` resolves the alias from
    `agent_tabs_view_config().machine_order` by the key's installation id first. Only as
    a fallback does it parse the catalog label, and it then strips a trailing `·remote`.
    This way `gD launch on …` never shows a disambiguated label. Update the docstring.
11. **Prompt launch hints.** In
    `src/sase/ace/tui/widgets/_prompt_input_bar_dispatch.py`:
    - `runs on ⌨ local` becomes `runs on ⌂ <local name>`.
    - The named-tab hint also fires when the named tab equals the local machine name
      (case-insensitive): `named tab wins over ⌂ athena`. The existing alias case keeps
      `named tab wins over ⌨ machine`.
12. **Machines pane.** In `src/sase/ace/tui/modals/machines_pane.py`, the "here" row's
    toast becomes `Showing ⌂ <local name> tab`, taken from the tab label helper. Update
    the `_machine_tab_key` docstring.
13. **Tab picker.** In `src/sase/ace/tui/modals/agent_tab_picker_modal.py`,
    `_picker_row_text` must render the glyph exactly once. When a descriptor exists, use
    `descriptor.label` after the glyph; raw catalog labels already carry `⌨ `/`⌂ `. Add
    a dim `  this machine` suffix for the local tab. Check the machine-mode rows for a
    doubled glyph today and fix that if you find it.
14. **Docs.** Update the Agent Tabs section in `docs/ace.md` (currently "labeled `main`,
    or `⌨ local` in machine mode …"):
    - the `⌂ <machine name>` label and its `id.machine_name` source;
    - the `⌂ local` fallback;
    - the teal home styling;
    - the `·remote` collision rule.

### C. Tests (sase)

- `tests/test_agent_tab_adapter.py`: update the `⌨ local` expectations and cover the
  `local_alias` passthrough, including that a blank value is omitted.
- `tests/ace/tui/test_agent_tabs_settings.py`: the view config resolves
  `local_machine_name` from a stubbed `get_local_machine_name`, the token changes with
  the name, and a resolver failure → `""`.
- `tests/ace/tui/test_agent_tab_strip_machine.py` / `test_agent_tab_strip_render.py` /
  `test_agent_tab_strip_catalog.py`:
  - the local descriptor has `⌂`, teal accent, and `is_local_machine`;
  - full, compact, and micro chip text (micro shows `⌂`, remote shows `⌨`);
  - active pill style;
  - the tooltip contains `this machine`;
  - off-tab extras word;
  - a remote host issue with the local name's alias leaves the local tab `ok`;
  - non-machine mode is unchanged (`main`, gray, no glyph).
- `tests/ace/tui/test_agent_tabs_launch_view.py` and
  `tests/ace/tui/widgets/test_prompt_launch_tab_context.py`:
  - `runs on ⌂ athena`;
  - the alias resolves from `machine_order` and never yields `·remote`;
  - the named-tab-equals-local-name hint.
- Picker: a machine-mode `_picker_row_text` test (single glyph, `this machine` suffix).
- Visual goldens (`tests/ace/tui/visual/test_ace_png_snapshots_agent_tab_strip.py`):
  - give `_install_tab_view` a `local_machine_name` parameter;
  - pin `"athena"` in the machine-mode goldens (`machine_mode`, `by_machine_*`,
    `named_vs_machine_alias`, `stale_host`, `feed_unavailable`);
  - update the `wait_for_svg_contains` tokens (`⌂`, `athena`);
  - regenerate with `just fix-tui-screenshots -- <those selectors>` and inspect the
    report and PNGs.

  Keep every test hermetic: stub the name resolver, and never read the developer's real
  owner identity.

## Verification

- sase-core: its `AGENTS.md` check recipe passes.
- sase: `just check` passes, and the targeted `just fix-tui-screenshots` run shows only
  the intended strip and picker diffs.
- Live look: take `sase screenshot -o /tmp/agents_local_tab.png` on a machine with
  dispatch machines configured. Confirm that `⌂ <machine name>` renders as a teal pill
  when active and as teal text when inactive, and that `]` / `[` cycling, the tooltip,
  and the picker behave as designed.

## Follow-ups

- The glossary's `machine-tab` strand still says the default tab renders as `⌨ local`.
  This plan does not edit memory. After landing, file a `memory` task bead through
  `/sase_new_task` for `sase/memory/glossary/machine-tab.md`. It should say the default
  tab renders as `⌂ <machine name>` (the viewer's `id.machine_name`, `⌂ local` when
  unset), and that a colliding remote alias renders as `⌨ <alias>·remote`.
- Out of scope, unchanged:
  - the `group: by machine` banner key `local`: a grouping and fold-persistence key, not
    a tab label;
  - the `machine:local` query token;
  - the dispatch target picker's `local launch` choice.

  If the user wants those renamed too, capture that as a separate task.
