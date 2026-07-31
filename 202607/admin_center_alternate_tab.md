---
tier: tale
title: Admin Center alternate-section jump on the opener key
goal:
  Inside a SASE Admin Center working tab, the opener key (`#` by default) jumps to the previously used section and
  toggles between exactly those two sections, advertised by a color-coded panel footer.
create_time: 2026-07-31 07:34:11
status: done
---

- **PROMPT:** [202607/prompts/admin_center_alternate_tab.md](prompts/admin_center_alternate_tab.md)

# Plan: Admin Center alternate-section jump (`#` inside a tab)

## 1. What the user asked for

Today the Admin Center opener key (`#`, configurable as `ace.keymaps.app.open_config_center`) has exactly one in-modal
meaning: **while the landing page is visible**, it resumes the last section that was active. Inside a working tab it is
deliberately inert and falls through to the focused widget.

Give it a second, tab-only meaning:

- Pressing the opener **while a working tab is selected** switches to the section the user was in _before_ the current
  one.
- Only two sections are ever remembered (the landing page already consumes the first one), so repeated presses ping-pong
  between the same two sections.
- The keymap must have an excellent, visually distinct footer description inside the Admin Center panel.

Vim's `Ctrl-^` / tmux's `prefix + l` are the mental model: a two-slot alternate, not an unbounded history stack.

## 2. Design decisions (and why)

These are settled decisions. Implement them as written; if you find a decision is impossible, say so explicitly in your
final report rather than silently substituting a different design.

### 2.1 A two-slot history value object, not a stack

Model the memory as an immutable pair, not a list:

```python
@dataclass(frozen=True)
class AdminCenterTabHistory:
    current: CenterTab | None = None
    alternate: CenterTab | None = None

    def remember(self, tab: CenterTab) -> AdminCenterTabHistory:
        """Return the history after ``tab`` becomes the active section."""
        if tab == self.current:
            return self
        return AdminCenterTabHistory(current=tab, alternate=self.current)
```

`current` is what the landing page resumes to (today's `resume_tab`). `alternate` is the new jump target. Two invariants
fall out for free and must hold everywhere: `alternate != current`, and re-activating the section you are already on
changes nothing. The ping-pong property is then automatic: from `(logs, config)`, jumping to `config` yields
`(config, logs)`, and jumping again yields `(logs, config)`.

**Name it `alternate`, never `previous`.** `ConfigCenterModal` already has `action_prev_center_tab` (Shift+Tab, "the
previous tab in catalog order"). Reusing "previous" for this concept would make the code ambiguous. Internal identifiers
use `alternate`; user-facing copy says "back to <Label>".

### 2.2 The pair is persisted, seeded from the same machine-local file

The alternate must survive closing and reopening the modal — the Admin Center is opened and dismissed constantly, and a
modal-lifetime-only alternate would be empty almost every time you want it. So extend the existing machine-local resume
state rather than adding modal-only memory.

`~/.sase/ace_admin_center_last_tab.txt` currently holds exactly one newline-terminated tab. Extend the _same file_ to
hold one **or two** newline-terminated lines: line 1 is `current`, optional line 2 is `alternate`.

- Keep the filename and the 64-byte cap (`statistics` is the longest id at 10 chars, so two lines fit easily).
- A legacy one-line file still loads, as `(current=<tab>, alternate=None)`. No migration step.
- Reject and return an empty history for: zero lines, more than two lines, a line that is not a catalog tab, a missing
  trailing newline, non-UTF-8 bytes, or oversized content — matching today's strictness.
- Reject line 2 when it equals line 1 (degenerate/corrupt), falling back to `alternate=None` while keeping `current`.
  The `alternate != current` invariant must never be violated by on-disk data.
- Writing `current=None` is a programming error; keep the existing `ValueError` behaviour for an invalid tab.

A pre-existing sase build reading a two-line file rejects it and shows "resumes after your first section visit" until
the next activation rewrites it. That downgrade path is self-healing and non-fatal; a second sidecar file would trade
that for a torn-state hazard across two non-atomic writes, so the single-file form wins.

### 2.3 It stays Python-side, not in the Rust core

Per this repo's Rust core backend boundary rule, ask whether another frontend would need this to match the TUI. It would
not: this is Textual panel navigation state for one modal, it already lives in
`src/sase/ace/tui/modals/config_center_state.py`, and no CLI, web, or editor surface reads it. Keep the whole feature in
this repo alongside the existing resume-tab code.

### 2.4 Key dispatch: a second, non-priority binding on the same key

Bind the opener key twice on `ConfigCenterModal` and let `check_action` select which one runs. Textual's
`App._check_bindings` iterates _every_ binding registered for a key in a namespace and calls `run_action` on each, which
consults `check_action` first — so a binding whose `check_action` returns `False` falls through to the next binding for
the same key. (Verified against the installed textual 8.0.1: `binding.py` stores
`key_to_bindings: dict[str, list[Binding]]`, and `app.py::_check_bindings` loops that list.)

Two properties matter, and both are load-bearing:

- **The existing `resume_last_tab` binding stays `priority=True`.** Do not touch it.
- **The new `alternate_center_tab` binding must be `priority=False`.** This is not a style choice. Priority bindings run
  _before_ the focused widget sees the key; a priority binding here would make `#` untypeable in every pane text input
  (Config's `:` path jump, the plugin/xprompt filters, and so on).
  `tests/ace/tui/test_config_center_resume.py::test_literal_opener_remains_typeable_and_cannot_nest_modal` exists
  precisely to pin that behaviour and **must keep passing unchanged**. A non-priority binding is only consulted after
  the key has bubbled unhandled past the focused widget, which is exactly the semantics we want and is the same tier the
  numbered `1`-`7` tab bindings already use.

**Ordering inside the `BindingsMap`:** append the new binding _after_ `*self.BINDINGS`, unlike the opener binding which
is deliberately first. Rationale: a user may configure the opener to a key that collides with a modal key (the existing
code comments call out `q` and `Tab`). With `resume_last_tab` first, "resume" wins on home; with `alternate_center_tab`
last, `q` still closes and `Tab` still cycles inside a working tab instead of being hijacked. Add a comment saying so.

### 2.5 Enablement rule

```python
def check_action(self, action, parameters):
    if action == "resume_last_tab":
        return self._active_tab is None and not self._initial_navigation_pending
    if action == "alternate_center_tab":
        return (
            self._active_tab is not None
            and not self._initial_navigation_pending
            and self._history.alternate is not None
        )
    return super().check_action(action, parameters)
```

When there is no alternate the action reports `False`, so the key falls through and stays typeable — consistent with
today's home-page behaviour. **Do not** emit a toast or notification on the inert press: the footer already explains the
state, and a toast on a no-op keystroke is noise.

### 2.6 Rejected alternatives (do not implement these)

- **`#` returns to the landing page when no alternate exists.** Rejected: it gives one key three different meanings and
  makes the outcome unpredictable from the user's point of view.
- **An unbounded MRU stack with repeated presses walking backwards.** Rejected: the user explicitly asked for a
  two-section toggle, and a stack makes repeated presses non-idempotent and hard to reason about.
- **Putting the affordance in each of the seven panes' existing hint rows.** Rejected: seven copies drift, and the hint
  rows describe pane-local actions while this is panel-global navigation.
- **Marking the alternate inside `PanelTabStrip`** (vim's `:ls` `#` marker). Attractive, but it changes a shared
  reusable widget for one caller and the user asked for a footer. Out of scope.

## 3. The footer

### 3.1 Placement

Add one modal-owned `Static` as the last child of `#config-center-container`, after the `ContentSwitcher`. Because the
switcher is `height: 1fr`, the footer naturally pins to the bottom of the panel.

**It is `display: none` while the landing page is showing** and visible only when a working tab is active. That keeps
the landing page pixel-identical to today (no home snapshot churn) and is honest: cross-section navigation only exists
once you are in a section. Its height is constant (1 row) for the entire time a tab is active, including when there is
no alternate yet — a footer that appears and disappears mid-session would reflow the pane under the user's cursor, which
is worse than a dim row.

Style it as a sibling of the existing hint rows:

```css
ConfigCenterModal #config-center-footer {
  width: 100%;
  height: 1;
  margin: 0;
  text-align: center;
  text-wrap: nowrap;
  text-overflow: clip;
}
```

### 3.2 Content and colors

This is the "visually distinct" part, and the distinctness comes from color-coding the _destination_, not from adding
chrome. The panel already has three chip idioms — the landing's reversed numeric key chips, the tab strip's accented
labels, and the accented `› description` caption. The footer reuses them so it reads as part of the same system, while
the reversed accent chip makes it pop against the uniformly muted pane hint row directly above it.

Enabled (alternate = Logs, whose catalog accent is `#FFD700`):

```
 #  ↔  Logs  ·  press again to return here
```

- `" # "` — `bold reverse <alternate accent>`, using `key_display_name(self._opener_binding)` so a custom opener renders
  correctly (the same helper the landing hint uses).
- `"  ↔  "` — `#666666`. Use `↔` (U+2194); `→` from the same Unicode block is already used in committed PNG goldens, so
  the pinned Fira Code visual fixture renders it fine.
- `"Logs"` — `bold <alternate accent>`, the catalog label.
- `"  ·  press again to return here"` — `#777777`. This sentence is deliberate: it teaches the toggle on first sight,
  which is what makes the feature discoverable rather than merely present.

Disabled (no alternate yet) — same shape, fully desaturated so the row reads as inert:

```
 #  ↔  no earlier section yet
```

`" # "` in `bold reverse #585858`, `"  ↔  "` in `#444444`, the copy in `#666666`.

Width handling: build the roomy string; if its plain length exceeds the widget width, fall back to the compact form
(chip + arrow + label, dropping the trailing sentence). Do this inside `render()` reading `self.size.width`, exactly
like `ConfigCenterHeaderDivider.render()` already does — no `Resize` handler needed.

### 3.3 Clickable

Give the footer an `on_click` that triggers the same switch when an alternate exists, mirroring
`AdminCenterLandingRow.on_click`: take a `Callable[[CenterTab], None]` in the constructor and pass
`self._schedule_switch`. Every other navigation surface in this panel (landing rows, tab strip) is clickable; this one
should be too.

## 4. Files to change

New:

| File                                                | Contents                                                               |
| --------------------------------------------------- | ---------------------------------------------------------------------- |
| `src/sase/ace/tui/modals/config_center_history.py`  | `AdminCenterTabHistory` frozen dataclass + `remember()`. Pure, no I/O. |
| `src/sase/ace/tui/modals/config_center_footer.py`   | `AdminCenterFooter(Static)` widget + its text builder.                 |
| `tests/ace/tui/test_config_center_history.py`       | Pure unit coverage of the value object.                                |
| `tests/ace/tui/test_config_center_alternate_tab.py` | Modal-level behaviour + footer coverage.                               |

Modified:

| File                                                                      | Change                                                                     |
| ------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| `src/sase/ace/tui/modals/config_center_state.py`                          | Load/save the pair; replace the single-tab functions.                      |
| `src/sase/ace/tui/modals/config_center_modal.py`                          | `alternate_tab` ctor arg, `_history`, new binding + action, footer wiring. |
| `src/sase/ace/tui/actions/_admin_center_persistence.py`                   | Coalesce and persist a history pair instead of one tab.                    |
| `src/sase/ace/tui/actions/_state_init.py`                                 | Seed the history pair at startup.                                          |
| `src/sase/ace/tui/actions/base.py`                                        | Pass `alternate_tab` into `ConfigCenterModal`.                             |
| `src/sase/ace/tui/styles.tcss`                                            | `#config-center-footer` rule.                                              |
| `src/sase/ace/tui/modals/help_modal/{agents,axe,changespecs}_bindings.py` | Help description (see §6).                                                 |
| `tests/test_keymaps_display_help.py`                                      | Follows the help string change.                                            |
| `tests/ace/tui/test_config_center_state.py`                               | Follows the persistence API change (see §5.2).                             |
| `tests/ace/tui/test_config_center_resume.py`                              | Follows the persistence API change.                                        |
| `docs/configuration.md`                                                   | Admin Center prose + keymap table row.                                     |

Explicitly **not** changed: `src/sase/default_config.yml` and `src/sase/ace/tui/keymaps/*`. This feature adds **no new
configurable keymap** — it is a second, context-dependent meaning of the existing `ace.keymaps.app.open_config_center`
binding. Do not add a keymap entry, and do not hand-edit `CHANGELOG.md` (release-please generates it).

## 5. Implementation steps

### 5.1 `config_center_history.py`

Add the frozen dataclass from §2.1. Also give it a validating constructor helper used by the loader, so the
`alternate != current` invariant is enforced in one place rather than at every call site.

### 5.2 `config_center_state.py`

Replace `load_admin_center_last_tab` / `save_admin_center_last_tab` with
`load_admin_center_tab_history() -> AdminCenterTabHistory` and
`save_admin_center_tab_history(history: AdminCenterTabHistory) -> None`. Keep the existing atomic-write shape verbatim
(mkstemp in the target directory, `fsync`, `os.replace`, temp cleanup on failure) — only the serialized payload changes.
Update `__all__`.

**Delete the two old functions rather than keeping wrappers.** Symvision scans `src/` and test references never keep a
public symbol alive, so leaving unused wrappers would fail `just lint`.

Test updates this forces in `tests/ace/tui/test_config_center_state.py`:

- `b"tasks\nlogs\n"` currently sits in the _malformed_ parametrize list. Move it to a valid round-trip case.
- Add malformed cases for the new rules: `b"tasks\nlogs\nconfig\n"` (three lines), `b"tasks\ntasks\n"` (degenerate
  duplicate — must yield `current="tasks", alternate=None`, not an empty history), and `b"tasks\nmissing\n"`.
- Keep the existing atomicity, replace-failure, and invalid-tab tests, retargeted at the new function.

### 5.3 `_admin_center_persistence.py` and `_state_init.py`

The mixin's coalescing writer currently dedupes on a single tab. **That dedupe becomes incorrect under the pair model
and must be re-keyed on the whole `AdminCenterTabHistory`.** Concretely: from `(logs, config)` a user can reach
`(logs, tasks)` via `logs → tasks → logs`. The durable value would still be `logs`, so the existing
`if tab == self._admin_center_tab_durable: return` early-out would skip the save and leave a stale `alternate` on disk.
Change `_admin_center_tab_durable`, `_admin_center_tab_queued`, and the `_admin_center_tab_save_pending` tuple to carry
the history object, and compare histories.

Everything else about the writer is correct and must be preserved exactly: the generation counter, the single-writer
restart loop, `spawn_pump_free_task` (per the TUI perf rules, this write must never run on the message pump), the
retry-after-failure behaviour, and the bounded `_flush_admin_center_tab_state` timeout.

- `_remember_admin_center_tab(value)` keeps its name and defensive `validated_center_tab` guard, but now applies
  `self._admin_center_history = self._admin_center_history.remember(tab)` and enqueues the resulting pair. A no-change
  `remember()` (same `current`) must still enqueue nothing.
- Keep `_last_admin_center_tab` working as the `current` accessor — several existing tests and `actions/base.py` read
  it. A read-only property delegating to the history is the cleanest way to avoid a two-source-of-truth bug; make sure
  the visual test at `tests/ace/tui/visual/test_ace_png_snapshots_config_center_home.py:46`, which _assigns_
  `page.app._last_admin_center_tab = resume_tab`, still works — either keep it a plain attribute kept in sync, or update
  that assignment to seed the history. Pick one and be consistent.
- `_state_init.py` seeds `self._admin_center_history` from `load_admin_center_tab_history()`. This is one bounded read
  before the event loop starts, as today — do not add more startup work.

### 5.4 `config_center_modal.py`

1. New ctor kwarg `alternate_tab: CenterTab | None = None`, validated with `validated_center_tab`. Seed
   `self._history = AdminCenterTabHistory(current=self._resume_tab, alternate=<validated alternate>)`, dropping the
   alternate if it equals the resume tab.
2. Register the second binding (§2.4) and extend `check_action` (§2.5).
3. `action_alternate_center_tab()` — guarded by the same condition, calls `self._schedule_switch(alternate)`.
4. In `_switch_to`, capture `previous_history = self._history` alongside the existing `previous_tab` / `previous_pane`,
   apply `self._history = self._history.remember(tab)` immediately before the header sync, and **restore
   `self._history = previous_history` on the rollback path**. The existing failure path already restores `_active_tab`
   and the switcher; the history must roll back with them or a failed switch corrupts the alternate.
5. Rename `_sync_header` to `_sync_chrome` and have it update the tab strip, the description caption, **and** the
   footer, so the three can never drift. Update all three call sites in the modal and the monkeypatch in
   `tests/ace/tui/test_config_center_navigation.py:112-119`.
6. Yield the footer as the last child in `compose()`, constructed hidden.

### 5.5 `actions/base.py`

`_open_config_center` passes `alternate_tab=` from the app's history next to the existing `resume_tab=`. The
`on_tab_activated` callback is unchanged: the modal and the app both start from the same seed and both apply the same
`remember()` transition, so they stay in sync without new plumbing.

## 6. Help modal

`src/sase/ace/tui/CLAUDE.md` requires the `?` popup to track ace behaviour changes. All three copies of the
`open_config_center` row carry the identical description `"Admin Center: 1-7 jumps; Statistics [/] t/T/c/g/p/P/r/?"` (55
chars).

Note that `help_modal/modal.py:329` truncates descriptions at `CONTENT_WIDTH - key_width - 2` = **32 characters**, so
today that string already renders as `Admin Center: 1-7 jumps; Stat...` — the Statistics tail is invisible and is
separately documented by the dedicated Statistics `?` modal.

Replace it in all three files with a description that fits fully:

```
"Admin Center: 1-7 jump, # back"
```

(30 chars.) Update the assertion at `tests/test_keymaps_display_help.py:104` to match.

## 7. Tests

Write these as real behavioural tests using the existing helpers in `tests/ace/tui/_config_center_tabs_helpers.py`
(`_patch_stub_panes`, `_StubPane`, `_InputPane`, `_HostApp`) and `sase.ace.testing.AcePage`. Follow the existing file's
style: `await page.press(...)` plus `await page.wait_for(...)` rather than fixed sleeps.

**`test_config_center_history.py`** (pure):

- `remember()` on an empty history yields `(tab, None)`.
- `remember()` with a different tab shifts current into alternate.
- `remember()` with the same tab returns the identical object (no churn).
- Three-way sequence `a → b → c` yields `(c, b)`; the toggle `a → b → a → b` alternates `(b, a)` / `(a, b)`.
- `alternate` can never equal `current` for any reachable sequence.

**`test_config_center_alternate_tab.py`**:

- Entering a second section then pressing the opener switches back to the first, and pressing it repeatedly ping-pongs
  between exactly those two — assert the full `calls` list so a third section is never constructed.
- With only one section ever visited, the opener is inert: `check_action("alternate_center_tab", ())` is `False`,
  `_active_tab` is unchanged, and no extra pane is created.
- The seeded pair survives a close/reopen: activate two sections, `escape`, reopen, resume from home with the opener,
  then press it again and land on the seeded alternate.
- A custom opener (`f2`, monkeypatching `sase.config.load_merged_config` as
  `test_custom_opener_opens_home_resumes_and_is_displayed` does) drives the jump and is what the footer renders.
- **Regression guard:** with `_InputPane` focused and an alternate available, pressing the opener types the literal
  character into the `Input` and does _not_ switch sections. This is the single most important test in the feature; the
  existing `test_literal_opener_remains_typeable_and_cannot_nest_modal` must also still pass untouched.
- A failed switch (`_create_pane` raising, as `test_failed_resume_retains_prior_target_and_remains_retryable` does)
  leaves the history unchanged and the jump retryable.
- Footer: hidden on home; visible with the alternate's catalog label and accent color once a tab is active; disabled
  copy when no alternate exists; updates on every switch; clicking it navigates.
- Persistence: after two sections, the on-disk file contains both lines in the right order.

Run `just test` for the fast lane. Also run `just test-visual`.

## 8. Visual snapshots

Adding a permanently visible footer row to every working tab changes the rendered height of all seven panes, so the
committed PNG goldens under `tests/ace/tui/visual/snapshots/png/` for working tabs will fail. This is an intentional
visual change, not a regression.

- The four `config_center_home_*` goldens **must not change** — if they do, the footer is not correctly hidden on the
  landing page. Treat any home-snapshot diff as a bug to fix, not a golden to accept.
- Regenerate the rest with `just update-visual-snapshots` (which runs
  `just test-visual -- --sase-update-visual-snapshots`), then re-run `just test-visual` clean.
- Before accepting, actually _look_ at two or three regenerated PNGs (a colorful one such as
  `config_center_tasks_tab_120x40.png` and a narrow one such as `config_center_statistics_narrow_90x30.png`) and confirm
  the footer reads well, the glyph is not tofu, and nothing important was squeezed out. Report what you saw.

Add a short `sase bead create -T task` follow-up only if you discover a _pre-existing_ flaky or failing check you did
not cause.

## 9. Docs

`docs/configuration.md`:

- In the "SASE Admin Center (interactive editor)" section (~line 133), document the new in-section meaning of the opener
  and the two-section toggle.
- While editing that paragraph, fix an adjacent stale sentence: it currently claims the resume target "is memory-only
  and is cleared by starting a new ACE process." That has not been true since resume state moved to
  `~/.sase/ace_admin_center_last_tab.txt` (`_state_init.py` loads it at startup). Correct it to say the target is
  persisted machine-locally across ACE processes, and state that the alternate is persisted the same way. Keep this
  correction tight — it is one sentence, not a rewrite of the section.
- Update the keymap table row (~line 308) from
  `From home, resume the last section used in this ACE process; otherwise do not open a nested Admin Center` to cover
  both meanings: resume from home; from inside a section, jump back to the previously used section and press again to
  toggle back. Keep the table's existing column alignment.

Do **not** modify any file under `sase/memory/`, `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, or `QWEN.md`. No
user permission for memory edits has been given for this work.

## 10. Verification

```bash
just install     # ephemeral workspace: dependencies may have drifted
just check       # ruff + mypy + symvision + fast tests
just test-visual # after regenerating goldens
```

Symvision specifics for this change: every new public symbol (`AdminCenterTabHistory`, `AdminCenterFooter`, the two
history load/save functions, the footer text builder) must have a real non-test consumer in `src/`. Test-only references
do not count. Delete the superseded `load_admin_center_last_tab` / `save_admin_center_last_tab` rather than leaving them
unreferenced, and do not reach for a pragma or an epic whitelist here — every symbol in this plan has a genuine in-repo
consumer.

## 11. Done when

1. Inside any Admin Center working tab, the opener key jumps to the previously used section, and pressing it repeatedly
   toggles between exactly those two sections.
2. On the landing page the opener still means "resume last section" and is unchanged.
3. The alternate survives closing and reopening the Admin Center and restarting ACE.
4. The opener character is still typeable in every pane text input.
5. A color-coded footer names the jump destination whenever a section is active, dims honestly when there is no
   alternate yet, is absent on the landing page, and is clickable.
6. `just check` and `just test-visual` pass; the home PNG goldens are byte-identical to before.
