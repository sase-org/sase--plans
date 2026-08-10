---
tier: epic
title: Snippet target mode for a single prompt input pane
goal: 'Authoring and editing an ACE snippet is a first-class prompt-bar loop: `gt`
  / `Ctrl+G t` asks for the trigger name with live collision evidence, opens one visibly
  distinct snippet pane at the bottom of the prompt input stack (pre-filled with the
  existing definition when the trigger already exists), and `<enter>` saves it to
  the user''s configured snippet config file — showing a real diff before writing
  and offering exactly the follow-up actions that file needs — after which the pane
  disappears and the cursor returns to precisely where it was.

  '
phases:
- id: target
  title: Snippet destination resolution, the new config field, and the collision index
  depends_on: []
  size: medium
  description: 'target: add the `ace.snippet_config_path` config field with its schema
    and default, lift snippet-location discovery out of the modals package into a
    UI-free module, add the chezmoi-aware destination resolver and the pure trigger-collision
    index, and make the existing unified save panel honor the configured destination.

    '
- id: model
  title: Snippet panes in the prompt stack model
  depends_on: []
  size: medium
  description: 'model: teach PromptStackState about at most one pinned bottom snippet
    pane that never participates in launch, stash, or save-as payloads, add the agent-pane
    accessors and structural invariants, and convert every pane-count call site through
    an audited table pinned by tests.

    '
- id: name
  title: Trigger-name panel with live collision evidence
  depends_on:
  - target
  size: medium
  description: 'name: build the trigger-name panel that validates as you type, shows
    where a colliding trigger already lives and what saving would shadow, lets the
    destination be overridden for the session, and returns the existing definition
    body when the user commits to an existing trigger.

    '
- id: pane
  title: The gt keymap, pane lifecycle, and exact cursor restoration
  depends_on:
  - model
  - name
  size: medium
  description: 'pane: add the `gt` / `Ctrl+G t` keymap and its hint, open the named
    snippet pane at the bottom of the stack, keep cursor and vim mode per pane so
    closing the snippet pane returns to the exact prior position, and enforce the
    discard-confirmation and last-agent-pane guard rails.

    '
- id: save
  title: Save confirmation with a real diff, the write, and follow-up actions
  depends_on:
  - pane
  size: medium
  description: 'save: route `<enter>` in the snippet pane to a confirmation panel
    with draft/existing/diff views and an external-change guard, reuse the existing
    snippet write, session publish, and post-write action chain, and close the pane
    only after the write succeeds.

    '
- id: visual
  title: Visual language for the snippet pane
  depends_on:
  - pane
  size: medium
  description: 'visual: give the snippet pane its own theme-safe look — a trigger-labeled
    separator rule, a distinct pane accent, new/overwrite and dirty states, and its
    own subtitle — and pin it with PNG snapshot goldens.

    '
- id: docs
  title: Help modal and documentation
  depends_on:
  - name
  - save
  - visual
  size: small
  description: 'docs: add the new keymap to the `?` help popup''s Prompt Input section
    and document the snippet authoring loop and the new config field in `docs/ace.md`
    and `docs/configuration.md`.

    '
- id: verify
  title: End-to-end verification of the snippet loop
  depends_on:
  - docs
  size: small
  description: 'verify: run the full verification gate and drive the real TUI through
    creating, editing, shadow-warning, discarding, and chezmoi-backed snippet saves,
    confirming the cursor returns exactly and no launch path regressed.'
proposed_by: bbugyi200.athena.xl
create_time: 2026-08-10 14:49:25
status: done
bead_id: sase-j3
---

- **PROMPT:** [prompts/202608/snippet_target_mode.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/snippet_target_mode.md)
- **BEAD:** [sase-j3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-j3/README.md)

# Plan: Snippet target mode for a single prompt input pane

## Context: what already exists

Most of the write side of this feature is already built and must be **reused, not
rebuilt**. Read this before writing code.

- **Snippet storage.** `sase.xprompt.snippet_config_yaml.insert_snippet_into_config`
  performs a minimal-edit insert/replace of one `ace.snippets` entry, preserving
  comments and blank-line scaffolding, and creating the `ace:` / `snippets:` mappings
  when missing. `generate_snippet_yaml` renders the entry block.
- **Destination discovery.** `load_snippet_config_locations()` (today in
  `src/sase/ace/tui/modals/snippet_config_location_modal.py`, despite the name it
  contains no Textual code) enumerates the user config file, `sase_*.yml` overlays, and
  the project `sase/sase.yml`, each with a writability reason.
- **Name indexes and previews.**
  `sase.xprompt.save_index.names_for_location( "snippet_config", path)` gives
  mtime-cached trigger names per file, and
  `load_definition("snippet_config", path, name)` gives the rendered YAML block for a
  colliding entry. `sase.xprompt.naming.validate_snippet_trigger` validates a trigger,
  and `resolution_after_save()` computes which file wins after a save.
- **The save panel this feature is modeled on.** `UnifiedXPromptSaveModal` (`gx`,
  `Ctrl+G x`) already has live validation, a collision verdict line, shadow reporting, a
  draft/existing/diff preview with `difflib.unified_diff` rendered through
  `rich.syntax.Syntax`, and a re-armed `<enter>` before an overwrite. Its snippet mode
  is the "Save pane as snippet" panel the user asked us to take inspiration from.
- **Write, publish, and follow-ups.** `PromptBarSaveSnippetMixin._write_snippet_target`
  writes off-thread, remembers the last-used location, publishes the new template into
  the live session catalog (`_publish_saved_snippet`) so every open prompt input sees it
  immediately, then calls `_offer_post_write_actions(...)`, which builds
  `PostWriteActionOffer`s via
  `sase.xprompt.write_targets.build_post_write_action_offers` and runs them as ordered
  tracked tasks — commit & push, then a **scoped** `chezmoi apply`. That chain is
  exactly the "there might be multiple confirmations required (i.e. if committing to a
  chezmoi repo)" behavior this feature needs, and it must be reused verbatim rather than
  reimplemented.
- **Chezmoi-aware write paths.**
  `sase.xprompt.write_targets.resolve_xprompt_write_target` redirects a home path to its
  chezmoi source (guarded on `use_chezmoi`, `$HOME` containment, and source existence)
  and reports the home `apply_target`.
- **XPrompt target mode (epic `sase-hp`).** `XPromptBinding` + `PromptStackState.bind()`
  put the **whole** bar into a targeting state: a `double $secondary` border, an
  `✎ <reference>` chip in the border title with clean/dirty/read-only/stale markers, a
  `[^G w] save …` subtitle hint, and a target-aware `<enter>` chooser
  (`PromptSubmitChoiceModal`). This feature borrows that vocabulary but is deliberately
  **pane-scoped**, not bar-scoped.

## Naming and vocabulary

- The user-facing noun for the pane is **snippet pane**; the state is **snippet target
  mode**. Prefer those words in strings, docstrings, and docs.
- A snippet is invoked by typing its **trigger** and pressing `Tab` — never `#trigger`.
  So the snippet chip is `⇥ <trigger>`, deliberately distinct from xprompt targeting's
  `✎ <reference>` chip. Every place this feature shows a snippet identity uses the `⇥`
  chip; no surface may render a snippet as `#name`.
- The word **destination** means the config file that will receive the `ace.snippets`
  entry, matching the unified save panel's existing vocabulary.

## Cross-cutting constraints

- **Rust core boundary.** Per `sase/memory/rust_core_backend_boundary.md`, shared domain
  behavior belongs in the sibling Rust core. The Rust core today owns snippet catalog
  _composition_ only (`sase.core.snippet_catalog_facade.compose_snippet_catalog`);
  snippet _discovery_, _destination policy_, and _writing_ are already plain Python
  under `src/sase/xprompt/` (`snippet_config_yaml.py`, `save_index.py`,
  `write_targets.py`). This epic follows that precedent: all new non-UI logic goes in
  UI-free modules under `src/sase/xprompt/` that import without Textual. If a phase
  agent finds the corresponding contract already modeled in `../sase-core`, extend it
  there and adapt here instead of duplicating it.
- **Never edit memory files.** No phase may add, edit, or remove anything in
  `sase/memory/*.md`, `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, or
  `QWEN.md`, including the glossary. A phase that concludes a memory note should change
  records a `PROPOSED FOLLOW-UP:` note on its own bead instead.
- **Do not hand-edit `CHANGELOG.md`** — release-please generates it from commit
  messages.
- **No blocking IO on the Textual event loop.** Every YAML read, `stat`, write, `git`,
  and `chezmoi` call goes through `asyncio.to_thread` or the tracked task queue, exactly
  as `_prompt_bar_save_xprompt.py` does today. Live-as-you-type collision checks must be
  debounced (`sase.ace.tui.util.debounce.DetailPanelDebouncer`) and served from the
  mtime-keyed caches in `save_index.py`. Read `sase/memory/tui_perf.md` through
  `/sase_memory_read` before touching anything on a refresh path.
- **Presentation boundary.** `PromptInputBar` is presentation-only: it posts a message
  and the app layer (`sase.ace.tui.actions.agent_workflow`) performs IO and pushes
  modals, matching `SaveAsXpromptRequested` / `WriteXpromptRequested`. Callbacks back
  into the bar must be staleness-guarded the way `_resolve_pane_target` is — a stack
  rebuild remounts panes under a fresh generation-scoped id.
- **TUI conventions.** `src/sase/ace/CLAUDE.md` requires the `?` help popup to stay in
  sync with any changed `sase ace` behavior and help-modal boxes to keep their fixed
  width. The keybinding footer is for _conditional_ app-level keymaps; prompt-bar `g`
  keys live in the hint panel and the help popup, not the footer, so this feature adds
  nothing to `keybinding_footer.py`.
- **Verification.** Workspaces are ephemeral: run `just install` first, then
  `just check` while iterating. The `verify` phase runs `just check-full` plus
  `just test-visual`.

## The loop this epic delivers

1. In any prompt bar, `gt` (NORMAL) or `Ctrl+G t` (INSERT) opens the **trigger-name
   panel**, which validates the trigger and reports collisions live as you type, and
   shows the destination file the snippet will be written to.
2. `<enter>` there opens one **snippet pane** at the bottom of the prompt input stack —
   empty for a new trigger, pre-filled with the current definition when the trigger
   already exists.
3. The pane looks unmistakably unlike a prompt pane: its own `⇥ <trigger>` separator
   rule naming the destination, its own accent, and its own subtitle.
4. `<enter>` in the pane opens the **save confirmation**, showing a real unified diff
   against the existing entry (or the draft YAML for a new one). Confirming writes the
   file and then runs the existing post-write chooser — commit & push, scoped
   `chezmoi apply` — which is where the additional confirmations for a chezmoi-managed
   destination come from.
5. On a successful write the pane disappears and focus returns to the pane that was
   selected before `gt`, at its exact cursor position and vim mode.

---

## Phase `target`: Snippet destination resolution, the new config field, and the collision index

### Gap

There is no way to say "put my snippets here". The unified save panel makes the user
pick a destination row every single time, and `load_snippet_config_locations` — the only
code that knows where snippets can live — sits in `src/sase/ace/tui/modals/`, so non-TUI
code cannot use it. Nothing resolves a snippet destination through the chezmoi-aware
write path, so a snippet written to a chezmoi-managed config would land on the applied
copy and be clobbered by the next `chezmoi apply`.

### Work

**1. New config field.** Add `ace.snippet_config_path` (a string, default `""`):

- `src/sase/default_config.yml`, under `ace:` next to `snippets: {}`, with a short
  comment stating the empty-string default behavior.
- `src/sase/config/sase.schema.json`, in the `ace` object: `"type": "string"`,
  `"default": ""`, and a description that states the resolution rules below. The `ace`
  object is `additionalProperties: false`, so the schema entry is mandatory, not
  optional.
- Read it in `src/sase/ace/tui/actions/_state_init_late.py` alongside `repro_output_dir`
  into `self._snippet_config_path: str`, following that field's defensive
  `isinstance(ace_cfg, dict)` shape.

**2. Lift snippet location discovery into a UI-free module.** Create
`src/sase/xprompt/snippet_targets.py` and move `SnippetConfigLocation` and
`load_snippet_config_locations` into it unchanged. Then delete
`src/sase/ace/tui/modals/snippet_config_location_modal.py` and update every importer:
`unified_xprompt_save_support.py`, the lazy export map and `__all__` in
`src/sase/ace/tui/modals/__init__.py`, the re-export in
`src/sase/ace/tui/modals/__init__.pyi`, and any test that imports the old path. Do not
leave a compatibility alias behind — a dangling alias is precisely what broke
`write_targets.py` three times during epic `sase-hp` (see that epic's bead notes).

**3. Destination resolver.** In the same module:

```python
@dataclass(frozen=True)
class SnippetSaveTarget:
    read_path: Path            # the config file the user names/sees
    write_path: Path           # where the entry is actually written
    apply_target: Path | None  # home path to `chezmoi apply` after a commit
    via_chezmoi: bool
    display_path: str          # ~-collapsed, for every UI surface
    source: Literal["configured", "default"]
    fallback_reason: str | None  # why a configured value was rejected

def resolve_snippet_save_target(configured: str | None) -> SnippetSaveTarget: ...
```

Resolution rules, in order — each is separately testable and each guard matters:

1. `configured` empty or whitespace → default (rule 5).
2. Expand `~` and environment variables; a still-relative path resolves against
   `CONFIG_DIR`, so `sase_snippets.yml` means `~/.config/sase/sase_snippets.yml`.
3. The suffix must be `.yml` or `.yaml`; otherwise fall back to the default with
   `fallback_reason` set. The file itself need not exist — `insert_snippet_into_config`
   creates it — but its parent directory must be writable.
4. Reject a configured path whose `_writability_reason` (the existing helper moved with
   the discovery code) is non-`None`: read-only, invalid YAML, not a mapping, or a
   project `sase/sase.yml` that still needs its legacy migration. Fall back to the
   default and set `fallback_reason` to that reason.
5. Default: `CONFIG_DIR / "sase.yml"`, or
   `CHEZMOI_HOME / "dot_config" / "sase" / "sase.yml"` when `get_use_chezmoi()` is true.
   This intentionally matches `load_snippet_config_locations`' existing chezmoi handling
   so the two surfaces never disagree about where "the user's sase.yml" is.
6. Whatever path survives is then passed through
   `write_targets.resolve_xprompt_write_target()` to pick up the chezmoi source redirect
   and `apply_target`. That resolver already refuses to remap a path that is outside
   `$HOME`, already inside `CHEZMOI_HOME`, or whose source file does not exist, so a
   default computed by rule 5 passes through untouched while an arbitrary configured
   home path gets correct chezmoi handling.

`fallback_reason` is not decoration: the `name` phase renders it so a user with a typo
in their config learns why their snippets are going somewhere else instead of silently
losing the preference.

Unit-test the resolver directly with a fake home root and fake chezmoi root: unset;
absolute configured; relative configured; wrong suffix; unwritable parent; invalid YAML;
chezmoi off; chezmoi on with the source present; chezmoi on with the source missing; a
configured path already inside the chezmoi source tree.

**4. Collision index.** A pure function in the same module:

```python
@dataclass(frozen=True)
class SnippetTriggerMatch:
    trigger: str
    location_path: str
    display_path: str
    is_destination: bool

@dataclass(frozen=True)
class SnippetCollision:
    matches: tuple[SnippetTriggerMatch, ...]   # config files defining this trigger
    derived_from: str | None                   # xprompt reference, when derived
    winner_path: str | None                    # file that wins after saving to dest
    shadowed_by: str | None
    shadows: str | None

def snippet_collision(
    trigger: str,
    destination: SnippetSaveTarget,
    *,
    locations: Sequence[SnippetConfigLocation],
    derived: Mapping[str, str],
) -> SnippetCollision: ...
```

Build `matches` from `names_for_location("snippet_config", path)` (mtime-cached, so
repeated keystroke checks are cheap), and compute `winner_path` / `shadowed_by` /
`shadows` with the existing `sase.xprompt.naming.resolution_after_save`, exactly as
`UnifiedXPromptSaveModal._resolution` does. `derived` is passed in by the caller (the
app owns the live prompt catalog); the function stays pure and Textual-free so it is
unit-testable without an app. Cover: no collision; collision in the destination;
collision only in another file, both higher and lower precedence; xprompt-derived
trigger; a trigger that is both derived and explicitly defined.

**5. Make the existing save panel honor the preference.** `UnifiedXPromptSaveModal`
gains a `preferred_snippet_path: str | None` constructor argument;
`_default_location_path()` returns it in snippet mode when it names a selectable row,
ahead of the last-used location. The app passes
`resolve_snippet_save_target(self._snippet_config_path).write_path`. Xprompt mode is
untouched. This keeps `gx`'s snippet mode and the new `gt` flow pointed at the same file
by default — two surfaces disagreeing about the default destination would be a bug
report waiting to happen.

### Done when

`ace.snippet_config_path` exists in the default config and the schema and is read by
ACE; snippet discovery lives in a Textual-free module with no compatibility alias left
behind; the resolver and collision index are unit-tested against all the listed cases;
the unified save panel defaults to the configured destination in snippet mode; and
`just check` passes.

---

## Phase `model`: Snippet panes in the prompt stack model

### Gap

`PromptStackState` models exactly one kind of item: an agent prompt. Everything that
launches, stashes, saves, joins, or counts panes iterates `self._stack.items` or tests
`len(self._stack) > 1`. Dropping a snippet pane into that stack without a partition
would silently launch the snippet body as an agent prompt, stash it as a prompt, fold it
into a saved xprompt, and count it in the `· N agents` title. This phase makes the
partition explicit and pins every consumer, before any UI exists.

### Work

**1. The pane target record.** In `src/sase/ace/tui/widgets/prompt_stack.py`, next to
`XPromptBinding`:

```python
@dataclass(frozen=True)
class SnippetPaneTarget:
    trigger: str
    read_path: str
    write_path: str
    display_path: str
    apply_target: str | None
    via_chezmoi: bool
    exists: bool                      # entry existed in the destination at open time
    loaded_body: str | None           # body pre-filled into the pane, else None
    loaded_fingerprint: SourceFingerprint | None
    derived_from: str | None = None   # xprompt reference when the body came from one
```

`PromptStackItem` gains `snippet_target: SnippetPaneTarget | None = None`. Keep
`SnippetPaneTarget` a plain record: it carries no behavior and no IO.

**2. Stack API.** `PromptStackState` gains:

- `snippet_item` / `snippet_index` / `has_snippet_pane`
- `agent_items` / `agent_texts` / `agent_count`
- `append_snippet_pane(text, target) -> PromptStackItem`
- `remove_snippet_pane() -> PromptStackItem | None`
- `retarget_snippet_pane(target) -> None` (rename / re-destination, body untouched)
- `snippet_is_dirty` — the snippet pane's stripped text differs from `loaded_body or ""`

**3. Structural invariants**, each with its own unit test:

- At most one snippet item ever exists; `append_snippet_pane` on a stack that already
  has one raises rather than creating a second.
- The snippet item is always last. `append_bottom()` and `insert_below()` insert _above_
  it.
- At least one agent item always exists. `remove_selected()` returns `False` without
  mutating when the selection is the only agent item.
- `move_selected()` refuses to move the snippet item and refuses to move an agent item
  past it; `split_selected()` and `load_segments_at()` are no-ops on the snippet item.
- `texts`, `join()`, `editor_markdown()`, and `is_effectively_empty` cover **agent items
  only**, so the launch payload, the dirty hash, and the whole-stack editor never see
  the snippet body.

**4. Call-site audit.** `__len__` keeps meaning "mounted panes" (rendering and height
math legitimately need that). Every other consumer is converted deliberately. Write one
test module that walks this table and asserts the behavior with a snippet pane present:

| Call site                                                          | Uses          | Expected with a snippet pane open                                            |
| ------------------------------------------------------------------ | ------------- | ---------------------------------------------------------------------------- |
| `prompt_input_bar._refresh_title` (`· N agents`)                   | `agent_count` | counts prompts only; never `2 agents` for one prompt + one snippet           |
| `insert_mode_subtitle` / `normal_mode_subtitle`                    | `agent_count` | snippet-pane subtitle handled in `visual`                                    |
| `is_multi_pane()` (pane nav, `K`/`J` swallow)                      | `__len__`     | unchanged — the snippet pane is navigable                                    |
| `is_stacked()` (editor surface choice)                             | `agent_count` | `^G g` opens the single prompt, not a two-segment buffer                     |
| `_prepare_text_submission` / `_handle_whole_stack_submission`      | `agent_count` | snippet body never launched; see `pane` guard rails                          |
| `capture_stashable_panes` (`gs`, `gS`, `gx`)                       | `agent_items` | snippet body never stashed or saved as an xprompt                            |
| `action_cancel` per-pane branch                                    | `__len__`     | snippet pane cancel is its own path (`pane` phase)                           |
| `_g_prefix_available_pane_nav`                                     | `__len__`     | nav stays available                                                          |
| `_g_prefix_available_stash_all` / `_save_xprompt` / `_update_pin`  | `agent_items` | availability ignores snippet text                                            |
| `_g_prefix_available_convert_local_xprompt` (`gX`)                 | selected item | unavailable while the snippet pane is selected                               |
| `restore_stashed_entries` empty-lead drop                          | `agent_items` | restores above the snippet pane, never drops it                              |
| `_build_pane_widgets` / `_apply_active_classes` / `_update_height` | `__len__`     | unchanged rendering math                                                     |
| separator numbering (`agent N`)                                    | agent index   | numbering skips the snippet pane                                             |
| `load_stack_from_xprompt_markdown` (whole-stack replace)           | —             | drops the snippet pane; the `pane` phase routes it through the discard guard |

**5. Cursor and mode persistence.** `PromptStackItem.cursor` and `.mode` exist and are
documented as preserved across reorder/insert, but nothing writes them:
`_sync_state_from_widgets()` copies text only, and `_after_rebuild()` always calls
`_cursor_to_end()`. Extend `_sync_state_from_widgets()` to also record each mounted
pane's `cursor_location` and `_vim_mode` into its item. Do **not** change any existing
rebuild's focus behavior in this phase — restoring is opt-in and lands in `pane`. Add a
test that a reorder round trip preserves the recorded cursor on the model.

### Done when

The stack models a pinned, non-launchable snippet pane with all listed invariants under
test, the audit table is a passing test module, cursor and mode are recorded on sync,
existing prompt-stack behavior with no snippet pane is byte-for-byte unchanged, and
`just check` passes.

---

## Phase `name`: Trigger-name panel with live collision evidence

### Gap

Nothing asks for a snippet trigger today. The unified panel's snippet mode gets a name
field for free because it is a save panel; this feature needs the name _before_ there is
anything to save, and the user asked that an already-used name be obvious live, as they
type, with `<enter>` on a collision loading that snippet's current definition.

### Work

**1. The modal.** `src/sase/ace/tui/modals/snippet_name_modal.py`:

```python
@dataclass(frozen=True)
class SnippetNameResult:
    trigger: str
    target: SnippetSaveTarget
    exists: bool
    existing_body: str | None
    derived_from: str | None

class SnippetNameModal(ModalScreen[SnippetNameResult | None]): ...
```

Constructed with the resolved destination, the discovered locations, the app's derived
snippet catalog, and an optional `initial_trigger` (the rename path from the `pane`
phase). It performs no IO on the event loop: every disk touch goes through
`asyncio.to_thread`, debounced with `DetailPanelDebouncer`, exactly like
`UnifiedXPromptSavePreviewMixin._schedule_existing`.

**2. Layout**, borrowing the unified panel's vocabulary so the two read as one family:

```
 New snippet
 Trigger  ┃todo_______________________________┃
   matches   todo      ~/.config/sase/sase.yml   "TODO($1): $0"
             todos     ~/.config/sase/sase.yml   "- [ ] $1"
 Destination  ~/.config/sase/sase.yml
 ⚠ todo already exists here — Enter loads its definition for editing
 tab complete · ↑↓ destination · enter open · esc cancel
```

- **Matches list** — up to six existing triggers with the typed text as a prefix, each
  with its source file and a single-line, ellipsized body preview. This is what makes
  "edit an existing snippet" discoverable instead of something you have to already know
  the name of. `Tab` completes the input to the highlighted match.
- **Destination line** — the resolved `display_path`. When `target.fallback_reason` is
  set, append it in the warning tone (e.g.
  `~/.config/sase/sase.yml · configured path unusable: read-only`) so a broken config
  value is visible rather than silent. `↑`/`↓` and `^P`/`^N` cycle among the selectable
  discovered locations for this invocation only — matching `UnifiedSaveInput`'s existing
  arrow-forwarding idiom — and never write the config back.
- **Verdict line** — one line, always present, in the unified panel's three tones:

| State                              | Verdict                                                                                                                              |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| empty / invalid trigger            | `✗ Invalid trigger: <validate_snippet_trigger message>`                                                                              |
| new, no collision                  | `✓ Create ⇥ <trigger> in <display path>`                                                                                             |
| exists in the destination          | `⚠ ⇥ <trigger> exists here — Enter loads its definition for editing`                                                                 |
| exists only in another config file | `⚠ ⇥ <trigger> is defined in <other> — saving here will shadow it` (or `will be shadowed by <winner>`, from `resolution_after_save`) |
| xprompt-derived only               | `⚠ ⇥ <trigger> comes from <#reference> — this entry will override it`                                                                |
| destination not selectable         | `✗ <disabled reason>`                                                                                                                |

**3. `<enter>` semantics.** Invalid → refuse and keep the panel open. Valid → dismiss
with a `SnippetNameResult`, loading the starting body off-thread first:

- Collision in the destination → the destination's own body (a plain YAML read of
  `ace.snippets[trigger]`; add `load_snippet_template(path, trigger) -> str` next to the
  resolver rather than reusing `load_definition`, which returns a rendered YAML block,
  not the template).
- Collision only elsewhere → that file's body, with `derived_from=None`; the pane's
  destination stays the resolved one, and the shadow warning carries into the save
  confirmation.
- Derived only → the composed template from the passed-in catalog, with `derived_from`
  set to the xprompt reference.
- No collision → `existing_body=None`, and the pane opens empty.

There is deliberately **no** re-arm ("press Enter again") here: `<enter>` on a collision
is not destructive — it loads a definition for editing. The destructive confirmation
belongs to the `save` phase, where a diff can be shown.

**4. Cancel.** `esc` dismisses with `None`; no pane is created and the caller restores
focus to the pane the user came from.

**5. Tests.** Validation messages; each verdict row in the table above; matches list
filtering, ordering, and `Tab` completion; destination cycling skipping non-selectable
rows and never mutating config; `fallback_reason` rendering; `<enter>` returning the
right body for each of the four collision cases; `esc` returning `None`; and an
assertion that no YAML read happens on the event loop (patch the loader and assert it is
called through `to_thread`, matching how the existing preview tests do it).

### Done when

The panel reports validity, collisions, shadowing, derived overrides, and the real
destination live as the user types with no event-loop IO; `<enter>` yields the trigger,
destination, and starting body; and `just check` passes.

---

## Phase `pane`: The `gt` keymap, pane lifecycle, and exact cursor restoration

### Gap

With the model and the name panel in place, nothing yet opens, closes, or protects the
snippet pane, and the existing rebuild path always drops the cursor at the end of the
document — so returning "to the exact same cursor position" is not yet possible.

### Work

**1. The keymap.** Add to `_PROMPT_G_PREFIX_BINDINGS` in
`_prompt_input_bar_g_prefix_actions.py`:

```python
_PromptGPrefixBinding(
    "t",
    "request_snippet_target_pane",
    "_g_prefix_label_snippet_target",
    "_g_prefix_available_snippet_target",
),
```

`t` is unused by the prompt `g` table and is not one of the vim `g` commands the
dispatcher falls through to (`gg`, `ge`/`gE`, `gu`/`gU`/`g~`) — confirm that again
before implementing. Availability: prompt mode only. Label: `new snippet…` normally,
`rename ⇥ <trigger>…` when the snippet pane is already open (see step 4). The new pane
always lands in INSERT regardless of which prefix invoked it, so the binding does not
take `uses_target_mode`.

**2. Request and open.** The bar posts a new
`PromptInputBar.SnippetTargetRequested(origin_bar, origin_pane_id, initial_trigger)`
message; the app handler in `sase.ace.tui.actions.agent_workflow` (a new
`_prompt_bar_snippet_pane.py` module alongside `_prompt_bar_save_xprompt.py`) loads the
destination, locations, and derived catalog off-thread, pushes `SnippetNameModal`, and
on a result calls back into the bar with a staleness guard. The bar then:

- captures the return focus record — `(item_id, cursor, vim_mode)` of the currently
  selected pane, taken after `_sync_state_from_widgets()`;
- builds a `SnippetPaneTarget` from the `SnippetNameResult`;
- calls `self._stack.append_snippet_pane(existing_body or "", target)` and
  `self._rebuild_stack(enter_mode="insert")`;
- leaves the cursor at the end of a pre-filled body (the user is editing it), and at
  offset 0 of an empty one.

**3. Close.** `close_snippet_target(reason: Literal["saved", "discarded", "replaced"])`
removes the snippet item, rebuilds, and restores focus. Add an opt-in
`restore_focus: PromptFocusRestore | None` argument to `_rebuild_stack()` /
`_after_rebuild()` that focuses the recorded `item_id` when it still exists (falling
back to the bottom agent pane when it does not), sets `cursor_location` to the recorded
position **clamped** to the current document, and re-enters the recorded vim mode —
instead of the default `_cursor_to_end()`. Every existing rebuild call site keeps the
current behavior by omitting the argument.

**4. `gt` while the snippet pane is open** re-opens the name panel prefilled with the
current trigger and dismisses into `retarget_snippet_pane(...)`: the trigger and
destination change, the body the user typed is never replaced. The verdict line will
report the new collision state, and the destructive confirmation still happens at save
time.

**5. Guard rails.** Three rules, each with a test:

- **R1 — the snippet pane is not a prompt.** Enforced by the `model` phase's filtered
  accessors; this phase adds the routing: `_handle_text_submission()` detects that the
  origin pane is the snippet pane and delegates to the `save` phase's entry point
  instead of launching.
- **R2 — never strand the bar.** `remove_selected()` already refuses to remove the only
  agent item; `action_cancel` on the snippet pane discards the snippet draft (subject to
  R3) and returns focus **without** posting `Cancelled`, because a snippet draft is not
  prompt history.
- **R3 — never lose a snippet draft silently.** Any path that would unmount the bar or
  replace the whole stack while `snippet_is_dirty` is true first pushes a
  `ConfirmActionModal` — `Discard unsaved snippet ⇥ <trigger>?`, default cancel — and
  proceeds only on confirmation. The paths are: whole-stack submit, `action_cancel_all`
  (`^G ^C`), `stash_all_panes`, `stash_all_and_load_xprompt_markdown`, and
  `load_stack_from_xprompt_markdown`. A clean (empty or unchanged) snippet pane is
  dropped silently — no confirmation for nothing.

**6. Tests.** Opening from NORMAL `gt` and INSERT `^G t`; the pane lands last, in
INSERT, with the pre-filled body; the return record survives a rebuild; closing restores
the exact `(item_id, cursor, mode)` including the clamped case where the pane's text
shrank while the snippet pane was open; `gt` twice re-targets without touching the body;
cancelling the name panel adds no pane and restores focus; each R3 path confirms when
dirty and skips the confirmation when clean; a launch with a snippet pane open sends
only the agent panes.

### Done when

`gt` / `^G t` opens a named snippet pane at the bottom of the stack, closing it returns
to the exact prior cursor and mode, no draft can be lost without an explicit
confirmation, no launch payload ever contains the snippet body, and `just check` passes.

---

## Phase `save`: Save confirmation with a real diff, the write, and follow-up actions

### Gap

`<enter>` in the snippet pane currently means "launch". The write, session publish, and
post-write chain exist but are reachable only from `UnifiedXPromptSaveResult`, and
nothing shows a diff before overwriting a snippet the user may have spent real time on.

### Work

**1. Confirmation panel.** `src/sase/ace/tui/modals/snippet_save_confirm_modal.py`,
built from the same parts as `UnifiedXPromptSavePreviewMixin` so the two look and behave
alike:

- Title: `Save snippet ⇥ <trigger>`.
- Header: `Overwrite <display path>` / `Insert into <display path>`, plus the tab strip
  `[Draft] Existing · Diff` — the same rendering the unified panel uses, with `^O`
  cycling and `^D` / `^U` scrolling.
- Diff: `difflib.unified_diff` over the rendered `ace.snippets` entry blocks
  (`generate_snippet_yaml(trigger, body)`), shown through
  `Syntax(diff, "diff", theme="ansi_dark")`. For a new trigger only `Draft` exists, and
  the panel opens on it; for an overwrite the panel **opens on `Diff`**, because the
  diff is the reason the confirmation exists.
- When the pane came from a shadowing collision or a derived override, carry that
  warning into a line here too — the decision to write is being made on this screen.
- `<enter>` saves, `esc` returns to the pane with its cursor untouched.

Refusals and short-circuits, before anything is written:

- Empty body → `✗ Snippet body is empty` and no save.
- Overwrite whose rendered entry is byte-identical to the existing one → `✓ No changes`
  and `<enter>` simply closes the pane. Writing an identical file would produce a
  pointless commit offer.
- The destination changed on disk since the pane opened (compare
  `SourceFingerprint.stat_signature` against `loaded_fingerprint`, off-thread) → show
  `⚠ <display path> changed on disk` and offer `r` to reload the current definition into
  the pane (dismissing the panel) alongside `<enter>` to overwrite anyway. This mirrors
  `XPromptWriteConflictModal`'s overwrite/reload choice without inventing a second
  dialect for it.

**2. Reuse the write chain.** Extract the body of
`PromptBarSaveSnippetMixin._write_snippet_target` into a shared coroutine:

```python
async def save_snippet(
    self,
    *,
    write_path: str,
    read_path: str,
    location_path: str,
    display_path: str,
    trigger: str,
    body: str,
    exists: bool,
) -> bool: ...
```

It keeps the existing sequence exactly — `insert_snippet_into_config` off-thread,
`save_last_used_location("snippet", ...)`, `_publish_saved_snippet` so every open prompt
input sees the new template immediately, the created/saved toast, then
`_offer_post_write_actions(..., noun="snippet", commit_type="snippet", refresh_config_on_success=True)`
— and returns whether the write succeeded. `_write_snippet_target` becomes a thin caller
so the `gx` snippet path and the new `gt` path cannot drift. Build the post-write target
from the `SnippetPaneTarget`'s already resolved `write_path` / `apply_target` /
`via_chezmoi` rather than re-deriving it, so a chezmoi-managed destination reliably
offers both the commit and the **scoped** apply.

**3. Close only on success.** The pane closes through `close_snippet_target("saved")`
only after `save_snippet` returns `True`; a failed write leaves the pane, its body, and
its target intact so the user can retry. The pane closes as soon as the file is written
— the follow-up commit/apply tasks run in the tracked queue and must not hold the pane
open.

**4. Tests.** Confirmation rendering for create, overwrite, no-change, empty, and
changed-on-disk; the diff content for a real edit; `esc` leaving the pane and cursor
untouched; a successful save writing the file, publishing to the session catalog, and
closing the pane; a failed write keeping the pane and notifying; the post-write offers
for a chezmoi-managed destination including a scoped apply; and `gx`'s snippet mode
still working through the shared helper.

### Done when

`<enter>` in the snippet pane shows a truthful diff, writes exactly one `ace.snippets`
entry, publishes it live, offers the right follow-ups, closes the pane on success only,
and `just check` passes.

---

## Phase `visual`: Visual language for the snippet pane

### Gap

Without its own look the snippet pane is indistinguishable from an agent prompt, which
is dangerous: `<enter>` means something completely different in it. XPrompt targeting
solved the same problem at the _bar_ level; this state is pane-scoped, so it needs its
own vocabulary rather than a reuse of the bar border.

### Work

Read `sase/memory/tui_perf.md` through `/sase_memory_read` first: nothing here may add
work to a hot refresh path.

**1. The separator becomes the pane's title bar.** `_PromptStackSeparator` currently
renders a centered `agent N`. Generalize it to take a label and a kind so the snippet
pane's rule reads:

```
──────────── ▍ ⇥ todo · ~/.config/sase/sase.yml ● ────────────
```

- `⇥ <trigger>` is the chip; the destination follows in the dim tone, middle-elided with
  the existing `_middle_elide_cells` helper so a deep path never overruns the rule.
- The state marker mirrors xprompt targeting's grammar: dim `✓` when the pane matches
  its loaded definition, the warning `●` when dirty, and `new` when the trigger does not
  exist yet.
- Agent numbering keeps counting agent panes only, so a stack of two prompts plus a
  snippet pane reads `agent 1`, `agent 2`, `⇥ todo` — not `agent 3`.

**2. Pane accent.** New CSS in `styles.tcss` next to the existing `.prompt-pane` rules:
a distinct `border-left` accent and background tint for `.prompt-pane.snippet-target`,
plus its `.active` / `.inactive` variants and the separator's own color. Use theme
variables only — no hardcoded hex — and pick a variable distinct from both `$accent`
(plain active pane) and `$secondary` (xprompt targeting) so the three states are never
confusable. The bar border itself does **not** change: the state belongs to one pane.

**3. Subtitle and hints.** When the snippet pane is the active pane,
`insert_mode_subtitle()` / `normal_mode_subtitle()` return snippet-specific hints —
`[Enter] save ⇥ todo  [Esc] normal  [^C] discard  [^G t] rename` — elided sensibly on a
narrow terminal, and never the agent-stack hints. When another pane is active the
subtitle is the ordinary one. The `g` prefix hint panel shows the `gt` row with the
label from the `pane` phase.

**4. PNG snapshot goldens.** Add goldens under `tests/ace/tui/visual/snapshots/png/`
following the existing prompt-stack suite
(`tests/ace/tui/visual/test_ace_png_snapshots_prompt_stack.py` and
`_ace_prompt_png_snapshot_helpers.py`) for: a new empty snippet pane below one prompt
pane; a dirty overwrite pane with the `●` marker; the trigger-name panel showing a
collision verdict; and the save confirmation showing a diff. Run `just test-visual` and
use `--sase-update-visual-snapshots` only to accept the new goldens deliberately. These
goldens are what make "beautiful" checkable by the next agent instead of a matter of
opinion.

### Done when

The snippet pane is unmistakable at a glance in both light and dark themes, its
separator names the trigger and destination with a truthful state marker, the subtitle
matches what `<enter>` will do, the PNG goldens pass, and `just check` passes.

---

## Phase `docs`: Help modal and documentation

### Work

- **`?` help popup.** Add to the `Prompt Input` section in
  `src/sase/ace/tui/modals/help_modal/binding_common.py`, next to the existing `gx` /
  `gX` / `gw` rows: `("gt / Ctrl+G t", "New/edit snippet pane")`. Respect the
  32-character description cap and the fixed box width from `src/sase/ace/CLAUDE.md`.
- **`docs/ace.md`.** Add `gt` / `Ctrl+G t` to both prompt keymap tables (the INSERT
  `Ctrl+G` table and the NORMAL `g` table), and extend the existing snippets section
  with a short "Authoring a snippet from the prompt bar" subsection: the five-step loop,
  what each verdict in the name panel means (including shadowing and the xprompt-derived
  override), what the diff shows, which follow-up actions appear for a chezmoi-managed
  destination, and that `^C` discards with a confirmation when the draft is dirty. State
  plainly that the snippet pane is never launched with the stack.
- **`docs/configuration.md`.** Document `ace.snippet_config_path`: the default, the
  chezmoi behavior, relative-path resolution against `~/.config/sase/`, the accepted
  suffixes, and the fallback-with-reason behavior for an unusable value. Mention that it
  is also the default destination in the `gx` snippet save panel.
- **`docs/xprompt.md`.** Only if that file's snippet cross-references become stale —
  otherwise leave it alone.
- **Do not touch memory files or the glossary.** If a glossary entry for "snippet pane"
  seems warranted, record it as a `PROPOSED FOLLOW-UP:` note on the bead.

### Done when

The help popup and the docs describe the shipped behavior with no stale keys or invented
ones, the new config field is documented where users look for config, and `just check`
passes.

---

## Phase `verify`: End-to-end verification of the snippet loop

### Work

1. `just install`, then `just check-full` — every lint gate plus the full suite — and
   `just test-visual` for the new goldens.
2. Drive the real TUI (see the project's run conventions) through the whole loop,
   confirming the visual state at each step:
   - **New snippet.** `gt`, type a fresh trigger, confirm the `✓ Create` verdict and the
     destination line, `<enter>`, type a body with a `$1` tabstop, `<enter>`, confirm
     the draft view, save. Then verify the trigger expands with `Tab` in a prompt pane
     **in the same session** (the live publish path) and that the YAML landed in the
     configured file with surrounding entries and comments untouched.
   - **Edit an existing snippet.** `gt`, type the existing trigger, confirm the
     collision verdict, `<enter>`, confirm the body pre-filled, edit it, `<enter>`, and
     confirm the diff shows exactly the edit before saving.
   - **Shadowing.** Define the same trigger in a second config file and confirm the name
     panel names the winner and the shadow direction correctly.
   - **Derived override.** Use a trigger that comes from an xprompt-derived snippet and
     confirm the override warning and the pre-filled composed template.
   - **Discard.** Open a snippet pane, type, `^C`, confirm the discard confirmation, and
     confirm the cursor returns to the exact prior position in the prior pane.
   - **Launch is unregressed.** With a snippet pane open, launch the prompt stack and
     confirm only the prompts launch and the discard confirmation appears for a dirty
     draft; with no snippet pane, confirm `<enter>` behaves exactly as before.
3. Exercise the chezmoi destination: confirm the write lands on the chezmoi source and
   the follow-up offers a commit plus an apply **scoped** to the home target. If
   `use_chezmoi` is not enabled in the verification environment, cover it with an
   integration test against a fake home and fake chezmoi source tree instead — and say
   plainly on the bead which of the two was done.
4. Set `ace.snippet_config_path` to a valid non-default file and confirm both `gt` and
   `gx`'s snippet mode default to it; then set it to a nonsense value and confirm the
   fallback-with-reason is visible rather than silent.

### Done when

`just check-full` and `just test-visual` pass, every loop above is confirmed by
observation, and any gap found is either fixed or filed through `/sase_new_task`.
