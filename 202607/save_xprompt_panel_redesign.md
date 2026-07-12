---
create_time: 2026-07-11 16:29:13
status: done
prompt: 202607/prompts/save_xprompt_panel_redesign.md
tier: tale
---
# Redesign the "Save draft as xprompt" Panel

## Goal

Rebuild the prompt bar's save-as-xprompt experience (`gx` / `Ctrl+G x`, and the `gw` fallback when the draft is unbound)
around one keyboard-first screen that answers, at a glance:

1. **What** am I saving? (truthful preview of the exact document that will be written)
2. **As what name?** (validated live, collision- and shadow-aware)
3. **Where?** (grouped, badged destination list navigable with plain arrow keys)
4. **What happens when I press Enter?** (a single always-visible verdict line: create / overwrite / shadowed / invalid —
   with zero surprises)

The redesign must be intuitive (discoverable keys, visible consequences), reliable (no silent overwrites, no
silently-renamed xprompts), and beautiful (theme-driven styling, grouped lists with badges, syntax-highlighted previews,
PNG snapshot coverage).

## Current State

The flow lives in:

- `src/sase/ace/tui/modals/unified_xprompt_save_modal.py` — the one-screen modal (name input, flat location
  `OptionList`, status + preview statics).
- `src/sase/ace/tui/actions/agent_workflow/_prompt_bar_save_xprompt.py` (+ `_targets.py`, `_snippets.py`, `_git.py`,
  `_tasks.py` siblings) — event handler, write helpers, snippet flow, post-save git/chezmoi commit offer.
- `src/sase/xprompt/save.py` — persistence (`save_markdown_xprompt`, `save_config_xprompt`,
  `load_config_xprompt_markdown`, atomic writes).
- `src/sase/ace/tui/modals/xprompt_location_modal.py` — `get_all_xprompt_locations()` (location discovery; also used by
  the xprompt browser's create flow, which must keep working).
- Styling: `UnifiedXPromptSaveModal` rules in `src/sase/ace/tui/styles.tcss`.

### Problems observed (each maps to a design decision below)

**Intuitiveness**

- P1. Destination selection is hidden behind `ctrl+n`/`ctrl+p`; arrow keys do nothing while the name input has focus.
  The list is flat raw text (`label\npath`) with no group headers, icons, counts, or badges — unlike the polished
  grouped pickers elsewhere (`XPromptLocationModal`, `XPromptSaveTargetModal`).
- P2. You cannot see what you are about to save. The preview pane shows only the colliding existing definition, or a dim
  path; the draft itself (assembled frontmatter + body) is never shown.
- P3. Existing names at a location are invisible, so collisions are discovered only after typing an exact match.
- P4. The snippet path (`ctrl+t`) is invisible in the UI and inconsistent: it abandons the unified screen for a
  two-modal detour (`SnippetConfigLocationModal` → `SnippetNameModal`).
- P5. No description affordance, even though a good xprompt has a `description:`.

**Reliability**

- P6. **Silent overwrite**: `Enter` on a colliding name writes immediately. The `kind="write"` branch in
  `_prompt_bar_save_xprompt.py` never routes through `_confirm_overwrite_xprompt` (only the legacy `kind="overwrite"`
  path does). One keypress can destroy an existing definition.
- P7. **The typed name doesn't always win.** For markdown targets the file is written as `<typed>.md`, but if the
  draft's frontmatter carries a different `name:`, the loader uses the frontmatter name (`loader_sources.py`), so the
  xprompt is callable under a name other than what the toast reports. `frontmatter_for_new_target` only strips `name:`
  when it happens to equal the typed name.
- P8. **Namespaced names silently mutate.** `name.replace("/", "_")` maps `ns/foo` to `ns_foo.md`; the loader derives
  the name from the stem, so the user's `ns/foo` becomes callable only as `#ns_foo`. (`__` → `/` normalization applies
  to _references_, not filenames.)
- P9. **No shadowing awareness.** Discovery is first-wins across locations; saving into a lower-precedence location can
  produce an xprompt that `#name` never resolves to. Nothing warns.
- P10. No name validation beyond non-empty (spaces, `#`, path separators beyond `/`, trailing `.md`, etc. all pass
  through).
- P11. Built-in package locations (installed `sase` package dirs, `default_config.yml`, plugin configs) are offered as
  ordinary save targets. Useful when developing sase itself, a footgun otherwise; they get no visual distinction.

**Performance / beauty**

- P12. `load_unified_save_locations` eagerly reads the **full text of every xprompt in every location** at modal open.
  Off-thread, but contrary to the established lazy/mtime-cached patterns (see `memory/tui_perf.md` rules 1 and 7).
- P13. Hardcoded hex colors, no `Syntax` highlighting in the preview, terse hint line, no PNG snapshot coverage at all
  for this surface.

## Design

### Screen layout (target mock)

```
╭───────────────── Save draft as xprompt ── 3 panes · frontmatter ✓ ─────────────────╮
│ Name         #code_review_swarm                                                    │
│ Description  Fan out one reviewer per concern      (optional · saved as metadata)  │
│                                                                                    │
│ Destination                          │ Preview                 Draft · Existing ·  │
│ ── Project (sase) ──                 │                                  [Diff]     │
│ ▸ 📁 .xprompts/  (CWD)          12   │ ⚠ Overwrite ~/.xprompts/code_review_swarm.md│
│   📁 ~/.xprompts/               34 ● │ ──────────────────────────────────────────  │
│ ── Config files ──                   │ --- a/code_review_swarm.md                  │
│   📄 sase.yml  (user)            8   │ +++ b/code_review_swarm.md                  │
│   📄 sase_home.yml               2   │ -Old body …                                 │
│ ── Built-in (dev) ──                 │ +---                                        │
│   📁 sase package xprompts/     41   │ +description: Fan out one reviewer …        │
│                                      │ +…                                          │
│                                                                                    │
│ ⚠ #code_review_swarm exists here — press Enter again to overwrite                  │
│ ↑↓ destination · tab next field · ^o cycle preview · ^t snippet · ^d/^u scroll     │
╰────────────────────────────────────────────────────────────────────────────────────╯
```

All colors/styles come from theme variables (`$primary`, `$warning`, `$error`, `$success`, `$text-muted`), replacing the
hardcoded hex values.

### D1. Interaction model (fixes P1)

- The name input keeps focus for the whole interaction. **Up/Down arrows move the destination highlight** (a single-line
  `Input` has no use for them); `ctrl+n`/`ctrl+p` remain as aliases.
- `Tab`/`Shift+Tab` cycle Name → Description → destination list (list focus enables `j`/`k` and type-ahead later if
  wanted; not required for v1).
- `Enter` saves from any focus point. `Esc` cancels — nothing is ever written without an explicit Enter.
- A single **verdict line** above the hints is the source of truth for what Enter will do:
  - `✓ Create #name at <path>` (success style)
  - `⚠ #name exists here — press Enter again to overwrite` (warning/danger style, see D5)
  - `⚠ Saving here will be shadowed by <higher-precedence source>` (see D6)
  - `✗ Invalid name: <specific reason>` (error style; Enter disabled)

### D2. Name field intelligence (fixes P7, P8, P10)

- New pure helper module (proposed `src/sase/xprompt/naming.py`, no Textual imports):
  - `validate_xprompt_name(name) -> str | None` — returns a human-readable error or `None`. Rules: non-empty; allowed
    charset (letters, digits, `_`, `-`, `.`, and `/` as namespace separator); no leading/trailing separators; no
    whitespace; no leading `#`; reject `.md`/`.yml` suffixes with a targeted message ("the extension is added
    automatically").
  - `markdown_save_plan(name, frontmatter) -> (filename, frontmatter)` — the typed name is **authoritative**: derive the
    filename from the name; whenever the loader would derive a different callable name from the stem (namespaced names,
    frontmatter `name:` mismatch), stamp `name: <typed>` into the frontmatter explicitly; strip a redundant `name:` when
    the stem already yields it. This single choke point replaces the scattered `replace("/", "_")` +
    `frontmatter_for_new_target` logic and guarantees the saved xprompt is callable as exactly `#<typed name>`
    afterwards.
- Validation runs on every keystroke; the verdict line shows the specific error and Enter is inert while invalid.

### D3. Destination list (fixes P1, P3, P11)

- Grouped `OptionList` with disabled header rows, ordered: Project dir → CWD dirs → Home dirs → Config files → Plugin
  dirs → **Built-in (dev)** last. Reuse `get_all_xprompt_locations()` grouping and extend the row model with
  per-location metadata:
  - icon (📁 directory / 📄 config), shortened display path (existing `_shorten_display_path`),
  - xprompt **count** badge,
  - a **collision dot** (●) that live-updates as the name is typed — you can see every location where the name already
    exists before committing (P3),
  - `(will be created)` italic suffix for offered-but-missing standard directories: `.xprompts/` in CWD and
    `~/.xprompts/` are always offered even when absent (today missing dirs are simply not listed, leaving new users with
    only `sase.yml`),
  - read-only rows disabled with the reason inline (reusing the `_disabled_reason` writability checks from the legacy
    target modal),
  - Built-in rows carry a dim `dev` badge and caution styling — still selectable (this repo dogfoods itself) but
    visually distinct (P11).
- Default highlight: last-used location when still present (D8) → project directory → first home directory → first row.

### D4. Truthful preview panel (fixes P2, P13)

- Header line: action verb + resolved target path (`→ Create <path>` / `⚠ Overwrite <path>` /
  `→ Insert into <config>.yml under xprompts.<name>`).
- Body tabs, cycled with `ctrl+o` (tab strip rendered in the panel header):
  - **Draft** — the exact document that will be written, assembled through the same `_build_markdown_xprompt` used at
    write time (never a paraphrase), `Syntax`-highlighted as markdown (or YAML snippet for config targets).
  - **Existing** — the current definition at the target (only when colliding).
  - **Diff** — unified diff existing → draft (`difflib.unified_diff` + `Syntax(..., "diff")`), the default tab whenever
    a collision exists.
- `ctrl+d`/`ctrl+u` scroll the preview (consistent with the other pickers).
- Preview refresh is debounced while typing and all disk reads stay off the event loop (`memory/tui_perf.md` rules 1 and
  6).

### D5. Overwrite arming instead of silent write (fixes P6)

- First `Enter` on a colliding name does **not** write. It arms the overwrite: the verdict line flips to the danger
  style ("press Enter again to overwrite #name"), and the preview jumps to the Diff tab so the user sees exactly what is
  being replaced. The second `Enter` commits.
- Any state change (name/description keystroke, destination move, tab cycle) disarms. Key-repeat is filtered (an Enter
  arriving within a short interval of the arming Enter is ignored) so holding the key cannot confirm.
- Rationale vs. `ConfirmActionModal`: an extra modal hides the diff and context. The inline arm keeps the evidence on
  screen; the danger styling + disarm-on-change + repeat filtering keep it safe. The legacy `kind="overwrite"` confirm
  path is removed together with the legacy flow (D7).
- Non-colliding saves stay single-Enter.

### D6. Shadowing verdict (fixes P9)

- New pure helper (proposed alongside D2, e.g. `resolution_after_save(name, target, project)`): using the existing
  first-wins discovery order in `sase.xprompt.loader`, compute whether `#<name>` would resolve to the just-saved
  definition afterwards. If a higher-precedence source already defines the name, the verdict line warns "will be
  shadowed by <short source>" (still saveable — sometimes intentional). Symmetrically, when the save will itself shadow
  a lower-precedence definition, note it informationally.
- Rust-core boundary check: xprompt discovery/persistence currently lives in Python in this repo, so these helpers stay
  here; no `sase-core` change. They are deliberately non-Textual so a future move behind the core boundary stays cheap.

### D7. Snippet mode on the same screen (fixes P4)

- `ctrl+t` no longer launches a separate two-modal flow; it **toggles the same screen** into snippet mode and back:
  - Title/verdict restyle ("Save pane as snippet"); the name field becomes the trigger (validated against snippet
    trigger rules); the destination list swaps to `load_snippet_config_locations(project)` rows; the preview shows the
    active pane body plus the `ace.snippets` YAML entry that will be inserted.
  - Multi-pane drafts keep today's semantics — snippet source is the active pane only — but this is now stated in the
    header ("snippet saves the active pane · 3 panes in draft") instead of buried in a preview paragraph.
  - Collision handling, arming, verdicts, and the post-save toast/commit offer behave identically.
- After this, the save flow no longer pushes `SnippetConfigLocationModal`/`SnippetNameModal`; if the audit confirms this
  flow was their only consumer, delete them and their tests. The same audit applies to the legacy
  `XPromptSaveTargetModal` and the save flow's create-detour usage of `XPromptLocationModal`/`XPromptNameModal`
  (`_create_xprompt_flow`, `_ask_new_xprompt_name`, `kind="create"` dispatch). **Note:** `XPromptLocationModal` itself
  must survive — the xprompt browser (`xprompt_browser_actions.py`) uses it — only the save flow's dependency is
  removed.

### D8. Description field + small persistence (fixes P5)

- An optional single-line Description input under Name, prefilled from the draft frontmatter's `description:`; the value
  is written back into the frontmatter on save (typed value wins; empty leaves frontmatter untouched). Freely skippable
  with Tab/Enter.
- Remember the last-used save location per kind (xprompt vs snippet) in a small per-user JSON state file following the
  `~/.sase/agent_tags.json` precedent; used only for default highlight (D3).

### D9. Lazy, cached location index (fixes P12)

- Replace `load_unified_save_locations`'s read-everything approach with a **names-only index** per location (directory
  glob stems / config `xprompts:` keys), reusing `existing_names_for_location`-style logic, cached keyed by file/dir
  mtime. That is all the open-modal path and the collision dots need.
- Full existing-definition text is read lazily, off-thread, only when the Existing/Diff preview for a specific collision
  is requested; result cached per (path, mtime).

## Implementation Phases

Each phase lands independently green (`just check`).

### Phase 1 — Pure foundations

- `sase/xprompt/naming.py` (or extend `save.py`): `validate_xprompt_name`, `markdown_save_plan`
  (typed-name-authoritative filename + frontmatter stamping), snippet-trigger validation reuse.
- Shadow/resolution helper over loader discovery order.
- Names-only location index with mtime caching + lazy definition reads.
- Unit tests for all of the above (validation matrix, namespaced-name round-trips proving `#ns/foo` stays callable as
  typed, shadow verdicts across location precedence, cache invalidation on mtime change).

### Phase 2 — Modal rebuild

- Rebuild `UnifiedXPromptSaveModal` on the new row model: grouped/badged destination list, arrow-key navigation,
  Description input, verdict line, tabbed truthful preview (Draft/Existing/Diff), inline overwrite arming with disarm +
  repeat filtering, debounced off-thread preview refresh.
- Wire `_prompt_bar_save_xprompt.py` to the new result type; **all** writes route through one path that honors the
  arming contract (eliminating the silent-overwrite branch); keep the post-save binding, toast, and git/chezmoi commit
  offer intact.
- Theme-variable styling in `styles.tcss`; remove hardcoded hex.
- Behavior tests: arrow navigation, collision arming/disarming, invalid-name Enter inertness, config-target save,
  typed-name-wins frontmatter stamping, verdict text for shadowed saves.

### Phase 3 — Snippet mode + legacy retirement

- In-screen snippet toggle (`ctrl+t`) with swapped locations/preview/validation; identical arming and post-save
  behavior.
- Audit and delete now-dead flows: the save path's `kind="create"` detour
  (`_create_xprompt_flow`/`_ask_new_xprompt_name`), `XPromptSaveTargetModal`, and — if the save flow was their only
  consumer — `SnippetConfigLocationModal`/`SnippetNameModal`, plus their tests and `modals/__init__.py` exports.
  Preserve `XPromptLocationModal` for the xprompt browser.
- Behavior tests for mode toggling and snippet writes (including the active-pane-only rule with multi-pane drafts).

### Phase 4 — Polish and coverage

- PNG visual snapshots (120x40) for the key states: create verdict, collision + armed Diff view, snippet mode, and the
  no-writable-locations state — added to the existing visual suite (`tests/ace/tui/visual/`, goldens under
  `tests/ace/tui/visual/snapshots/png/`).
- Last-used-location persistence (D8) with tests.
- Help modal sync: `gx / Ctrl+G x` entry in `help_modal/binding_common.py` still reads "Save draft as xprompt/snippet" —
  verify wording matches the merged screen and update the help popup wherever the panel's keys are documented (per
  `src/sase/ace/CLAUDE.md`, help must track behavior).
- Perf sanity: modal open on a large catalog does no full-content reads (assert via the index-only loader tests);
  preview updates stay debounced.

## Explicit non-goals

- No change to xprompt discovery precedence, the `.gp`/ChangeSpec world, or the launch pipeline.
- No editing of existing xprompts here (the browser owns that); this panel only saves the current draft.
- No new user-configurable keymap entries: the panel's keys remain modal-internal `BINDINGS`, consistent with sibling
  modals (so no `default_config.yml` changes expected).
- No `sase-core` (Rust) changes (boundary check in D6).

## Risks & mitigations

- **Double-Enter overwrite arming is novel in this codebase** (elsewhere: `ConfirmActionModal`). Mitigated by loud
  danger styling, disarm-on-any-change, key-repeat filtering, and behavior tests; if review prefers consistency, the
  arming step can be swapped for the standard confirm modal without touching the rest of the design.
- **Deleting legacy modals may break unnoticed consumers.** Phase 3 starts with a usage audit; anything with a surviving
  consumer is kept and only unhooked from this flow.
- **Snippet-mode merge touches a stable flow.** The write helpers (`write_snippet_sync`, `_refresh_snippet_caches`,
  commit offer) are reused unchanged; only the UI entry path moves.
- **Frontmatter stamping changes saved-file bytes** (a `name:` line may now appear where it didn't). This is the
  correctness fix for P7/P8; tests lock the exact rules (stamp only when stem-derived name would differ, strip when
  redundant).
