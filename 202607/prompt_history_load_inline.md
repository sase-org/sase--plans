---
create_time: 2026-07-10 16:19:26
status: wip
prompt: 202607/prompts/prompt_history_load_inline.md
tier: tale
---
# Fix Prompt-History `<ctrl+i>` Load to Preserve the Existing Prompt Stack

## Problem

When the user triggers prompt history (`<ctrl+k>`) from one pane of a multi-pane prompt input bar, selects a prompt, and
presses `<ctrl+i>` (or `Tab`) to load it, the **entire prompt stack is replaced**: every other prompt input widget
disappears (their drafts are lost), and any xprompt properties (frontmatter) the user had staged are silently
overwritten or cleared. Only the newly loaded prompt survives.

### Root cause

The flow today:

1. `PromptTextArea.action_open_prompt_history()` (`src/sase/ace/tui/widgets/_prompt_text_area_actions.py`) posts
   `PromptInputBar.HistoryRequested(initial_filter=<pane text>, preserve_prompt_bar=True)`.
2. `on_prompt_input_bar_history_requested()` (`src/sase/ace/tui/actions/agent_workflow/_prompt_bar_requests.py`) opens
   `PromptHistoryModal`; `<ctrl+i>` dismisses it with `PromptHistoryAction.LOAD`.
3. The `LOAD` branch calls `bar.load_stack_from_text(_build_prompt(result.prompt_text))`
   (`src/sase/ace/tui/widgets/_prompt_input_bar_stack_rendering.py`), which does
   `self._stack = self._state_from_text(text)` — a **whole-stack replacement** that discards every other pane's text
   _and_ the stack's current frontmatter, then rebuilds all pane widgets from scratch.

## Desired behavior (spec)

Let "the current pane" be the prompt input widget from which the user opened the history modal.

1. **Single-agent history entry** (no real top-level `---` separators outside fenced blocks / frontmatter, per the
   canonical `split_prompt_text()` splitter — note an xprompt-swarm _invocation_ like `#foo` is a single segment): load
   the entry's body inline into the current pane only, replacing that pane's text (which held the single-line history
   filter). All other panes are untouched.
2. **Multi-agent history entry** (splits into N > 1 segments): load segment 1 into the current pane, then insert one new
   pane per subsequent segment **directly below the current pane, in order** (i.e. below the current pane or the last
   pane added by this load). Panes that were already above/below the current pane keep their text and relative order.
   Example: stack `[A, B*, C]` (B active) + entry `x --- y --- z` → `[A, x*, y, z, C]`.
3. **xprompt properties (frontmatter) policy** — the only data from the pre-existing draft that may ever be overwritten:
   - Loaded entry has **no** frontmatter → the stack's current frontmatter (and the properties panel) is left untouched.
   - Loaded entry has frontmatter, current stack has **no properties set** → adopt the loaded frontmatter; the
     properties panel auto-shows (existing `refresh_frontmatter_panel_from_stack()` behavior).
   - Loaded entry has frontmatter **and** the current stack already has properties set → show a y/n confirmation
     ("Loading this prompt will replace your current xprompt properties.") **before** touching anything.
     - Yes → perform the full load (panes per rules 1–2, frontmatter overwritten with the loaded entry's).
     - No → abort the entire load; nothing changes; focus returns to the current pane.
     - Skip the confirmation when the incoming raw frontmatter string is byte-identical to the current one (overwrite
       would be a no-op).
4. Focus/mode after a successful load: the current pane stays the active pane, cursor at end, insert mode (matching
   `update_active_pane()`'s `enter_mode="insert"` convention).

Unchanged behaviors (out of scope): `Enter` (SUBMIT) and `<ctrl+g>` (EDIT*FIRST) history-modal actions; the `,.`
Agents-tab history entry point (`_entry_prompt_history.py`) whose LOAD mounts a _fresh* bar (nothing to lose); history
filtering/paging; VCS-tag substitution (`_build_prompt()` still runs on the full entry text before any
parsing/splitting).

## Design

Presentation-only Textual stack/pane manipulation — per the Rust core backend boundary rule this stays entirely in this
repo (no `sase-core` changes).

### Phase 1 — `PromptStackState.load_segments_at()` (pure model, `src/sase/ace/tui/widgets/prompt_stack.py`)

New method modeled on `split_selected()`'s in-place replacement:

```python
def load_segments_at(self, index: int, segments: list[str]) -> None
```

- Replaces `items[index].text` with `segments[0]` (empty list behaves as `[""]`).
- Inserts one new item (via `_new_item`) per remaining segment at `index+1, index+2, …`.
- Leaves every other item intact; `selected_index` stays on `index` (clamped).

Also expose the currently private frontmatter splitter so the app layer can inspect an incoming entry without loading
it: promote `_extract_frontmatter(text)` to a public `split_frontmatter(text) -> tuple[str, str]` (keep behavior
byte-identical; update `__all__` and in-module callers).

### Phase 2 — `PromptInputBar.load_prompt_into_pane()` (widget layer, `_prompt_input_bar_stack_rendering.py`)

```python
def load_prompt_into_pane(self, target_text_area: object, pane_id: str, text: str) -> bool
```

- Resolve the origin pane with the existing staleness guard `_resolve_snippet_target()` (`_prompt_input_bar_actions.py`)
  — generalize its name to `_resolve_pane_target()` since it now serves both the `#@` selector and history loads (keep
  the docstring's semantics; update both call sites). Return `False` when the pane/bar is stale so the caller can notify
  without touching any prompt.
- `_sync_state_from_widgets()` first (like `update_active_pane()`), so live edits in _other_ panes survive the rebuild.
- Parse `text` mirroring `_state_from_text()` semantics: lift leading frontmatter, split the body with
  `split_prompt_text()`; a single-segment body is kept verbatim (the `lift_frontmatter=True` single-pane path),
  multi-segment bodies use the splitter's stripped segments.
- Map the resolved pane widget back to its stack index (match `self._pane_id(item)` against the widget id), call
  `load_segments_at(index, segments)`.
- Frontmatter: only when the incoming frontmatter is non-empty, overwrite `self._stack.frontmatter` with it (the
  conflict confirmation happens in the app layer _before_ this method is called). Incoming-empty → leave current
  frontmatter alone.
- Finish with `_rebuild_stack(enter_mode="insert")` + `refresh_frontmatter_panel_from_stack()` (existing patterns; no
  new refresh paths, no event-loop I/O — this is pure in-memory text parsing on an explicit user action).

Add a small query for the confirmation decision:

```python
def has_frontmatter_properties(self) -> bool
```

True when `self._stack.frontmatter` is non-empty and parses to a non-`is_empty` `PromptFrontmatter` model;
conservatively `True` when the string is non-empty but unparseable (mid-edit YAML), so a confirmation is shown rather
than silently clobbering it.

### Phase 3 — Origin-pane targeting on the message (`_prompt_input_bar_messages.py`, `_prompt_text_area_actions.py`)

`HistoryRequested` today carries no origin, so the app handler re-queries the bar and acts on whatever pane is active
when the modal closes — the exact staleness bug class `SnippetRequested` was already extended to fix. Mirror that
pattern:

- Add `origin_bar`, `origin_text_area`, `origin_pane_id` keyword fields to `HistoryRequested` (defaults `None`/`""` keep
  existing constructors working).
- `action_open_prompt_history()` populates them (`self._find_prompt_bar()`, `self`, `self.id or ""`).

### Phase 4 — App-layer LOAD branch + confirmation (`_prompt_bar_requests.py`)

Rewrite the `PromptHistoryAction.LOAD` branch of `on_history_select()`:

1. `built = _build_prompt(result.prompt_text)` (unchanged VCS-tag substitution).
2. Resolve the bar: `event.origin_bar` when it is a live `PromptInputBar`, else the current `#prompt-input-bar` query
   (defensive fallback for programmatic callers without an origin; the load then targets the bar's active pane via the
   same fallback semantics `_resolve_pane_target()` already has for an empty `pane_id` — pass the bar's
   `active_text_area()` in that case).
3. `incoming_fm, _ = split_frontmatter(built)`.
4. Conflict test: `incoming_fm` non-empty **and** `bar.has_frontmatter_properties()` **and**
   `incoming_fm != bar._stack.frontmatter` (expose via a tiny accessor if reaching into `_stack` reads poorly, e.g.
   `bar.current_frontmatter()`).
   - Conflict → `push_screen(ConfirmActionModal(...))` (existing canonical y/n dialog,
     `src/sase/ace/tui/modals/confirm_action_modal.py`) with `kind=ConfirmKind.DANGER` (destructive overwrite ⇒ default
     focus lands on "No") and a message stating the current xprompt properties will be lost. Callback: `True` → apply
     the load; `False`/`None` → abort and refocus the origin pane (same refocus pattern the `result is None` branch
     uses).
   - No conflict → apply the load directly.
5. "Apply the load" = `bar.load_prompt_into_pane(event.origin_text_area, event.origin_pane_id, built)`; on a `False`
   return (stale pane), `notify("Prompt pane is no longer available - selection discarded", severity="warning")` — same
   UX as the `#@` selector's staleness path. Never unmount the bar and never clear `self._prompt_context` on this path
   (the bar stays mounted with its surviving panes).

### Phase 5 — Retire the now-dead whole-stack path + docs

- After Phase 4, `load_stack_from_text()` has **no production callers** (its only caller was this LOAD branch;
  editor/stash reloads use `load_stack_from_xprompt_markdown()`). Remove it and migrate its tests (see below) to the new
  method. Before deciding removal vs. whitelist, re-read `memory/pyvision.md` via `/sase_memory_read` (unused-symbol
  rules). If removal ripples too far, keeping it with a test-only justification is acceptable — but removal is the
  default.
- Update stale docstrings/comments that describe the old behavior: `PromptHistoryAction.LOAD` comment
  (`_prompt_history_models.py`), `_rebuild_stack()`'s caller list, and the `HistoryRequested` docstring.
- The history modal's hint line (`^i: load`) and `BINDINGS` stay as-is (the keybinding is unchanged). Check the `?` help
  modal (`src/sase/ace/tui/modals/help_modal/`) for any text describing history-load semantics and update if present
  (per the ace help-sync rule); `binding_common.py`'s `#@ Ctrl+I` entry is the xprompt selector, not this modal — leave
  it.

## Testing

- **Model** (`tests/ace/tui/widgets/test_prompt_stack.py`): `load_segments_at` — single segment replaces in place; multi
  segments insert contiguously below the index preserving surrounding items and order; selection stays on the loaded
  index; unique item ids; empty-list → single empty pane text.
- **Widget** (`tests/ace/tui/widgets/test_prompt_input_bar_stack.py`, existing pilot-app harness):
  - Load a single-segment entry into the middle pane of a 3-pane stack → other panes' (live-edited) text intact, target
    pane replaced, target still active, insert mode.
  - Load a multi-segment entry into a pane with panes below it → new panes appear between target and the old below-pane;
    `all_prompt_texts()` ordering matches the spec example.
  - Entry with frontmatter + stack without properties → frontmatter adopted, panel shown.
  - Entry without frontmatter + stack with properties → frontmatter and panel untouched.
  - Stale target (pane id from a previous generation) → returns `False`, stack unchanged.
  - Migrate the `load_stack_from_text` tests (rebuild/lift-frontmatter/hide-panel/swarm-stays-single-pane cases) onto
    the new entry points that now own those semantics.
- **App layer** (`tests/ace/tui/test_prompt_bar_history_requests.py`, existing harness pattern):
  - LOAD without frontmatter conflict → `load_prompt_into_pane` called once with the built text; no extra modal.
  - LOAD with conflict → `ConfirmActionModal` pushed; callback `True` applies the load; `False` aborts, refocuses the
    origin pane, leaves the bar mounted and `_prompt_context` intact.
  - Identical incoming/current frontmatter → no confirmation.
  - Stale pane → warning notification, no unmount.
- Run targeted pytest subsets for the touched areas plus `just check` static gates (note: the full `just test` phase may
  be environment-killed in sandboxes; prefer targeted subsets and report honestly).

## Edge cases & risks

- **Feedback / approve-prompt bars**: `action_open_prompt_history()` already guards `bar._mode != "prompt"`, so the new
  pane-insertion path never runs outside prompt mode.
- **Single-pane bar**: behavior is a strict subset (replace the lone pane; multi-segment entries grow the bar into a
  stack via the normal separator rendering) — the previous UX is preserved except that existing frontmatter now survives
  a frontmatter-less load, which is the spec'd fix.
- **Frontmatter-only history entry**: body splits to zero segments → current pane text becomes empty; frontmatter policy
  still applies. Acceptable and consistent.
- **Modal-open races**: the origin-pane id is generation-scoped; any stack rebuild while the modal is open makes the
  target stale and the load becomes a safe no-op with a warning (mirrors the `#@` selector fix).
- **Perf**: no disk I/O or subprocess work added to handlers; splitting reuses the Rust-backed parser already used on
  this path, and rendering goes through the existing `_rebuild_stack()`/`refresh_frontmatter_panel_from_stack()` fast
  paths (per `memory/tui_perf.md` rules — no new refresh code paths).
