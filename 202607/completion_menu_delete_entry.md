---
tier: tale
title: Delete common completion entries with Ctrl+D in the prompt completion menu
goal:
  Ctrl+D in the ACE prompt completion menu durably deletes the highlighted history word or saved common placeholder,
  toasts the user, and updates the menu in place instead of closing it.
create_time: 2026-07-29 08:35:37
status: done
---

- **PROMPT:** [202607/prompts/completion_menu_delete_entry.md](prompts/completion_menu_delete_entry.md)

# Plan: Delete common completion entries with `Ctrl+D`

## Objective

Add a `Ctrl+D` keymap to the ACE prompt input widget's completion menu that deletes the currently selected entry from
the two durable "common" lists that back manual completion:

- **History words** — the recently used words derived from prompt history, capped by
  `ace.prompt_completion.history_word_count`.
- **Saved common placeholders** — the durable `<placeholder>` store capped by
  `ace.prompt_completion.common_placeholder_count`.

Every successful deletion shows a toast, and the completion menu is updated in place rather than dismissed.

## Scope

`Ctrl+D` is already bound in this menu for the `file_history` completion kind (recent files), where it removes the
highlighted path from `~/.sase/file_reference_history.json`. This plan extends the same key to the two "common" lists
above, reusing that interaction model.

In scope:

- `history_word` completion rows.
- `placeholder` completion rows whose source is `common` (the gold `◆` rows).
- A toast on every successful deletion, including the existing `file_history` deletion, so one key never behaves two
  different ways in one menu.
- Menu stays open and re-renders after a deletion.

Out of scope (do not implement):

- Deleting from any other completion kind: `prompt_word` (words that live in the prompt text itself and have no durable
  store), `file`, `xprompt`, `xprompt_arg_*`, `directive`, `directive_arg`, `jinja`, and the VCS project/ref/repo kinds.
- An undo/restore surface or a CLI to manage deleted entries.
- Bulk deletion or row marking.
- Migrating the placeholder or history-word stores into the Rust core (see "Rust core boundary" below).
- Any change to `history_word_count`, `common_placeholder_count`, `word_min_length`, ranking, or cache scheduling.

## Current behavior

Completion menu state lives on `PromptTextArea` through the mixin chain in `src/sase/ace/tui/widgets/`:
`_file_completion_context.py` → `_file_completion_base.py` → `_file_completion_accept.py` →
`_file_completion_refresh.py` → `_file_completion_open.py` → `_file_completion.py`.

- `src/sase/ace/tui/widgets/_prompt_text_area_key_handling.py` handles `ctrl+d` only when
  `self._completion_kind == "file_history"`, calling `_delete_selected_file_completion()`. For every other kind the key
  falls through to the app-level `scroll_detail_down` binding.
- `_delete_selected_file_completion()` in `src/sase/ace/tui/widgets/_file_completion_accept.py` calls
  `remove_file_reference()` synchronously, deletes the row from `_file_completion_candidates`, clamps
  `_file_completion_index`, and calls `_update_file_completion_panel("")`. It shows no toast.
- `src/sase/ace/tui/widgets/_prompt_input_bar_completion_panel.py` renders the panel. `file_history` menus already show
  the affordance `[^L] accept  [^D] delete` in `panel.border_subtitle`; the placeholder menu instead shows
  `_PLACEHOLDER_SOURCE_LEGEND` when both `prompt` and `common` rows are visible, and the history-word menu shows no
  subtitle.

### History words

- `src/sase/history/prompt_words.py` derives words on demand from the sharded prompt-history store; there is no per-word
  record to delete. `history_words_source_token()` returns `(max_words, min_length, <per-shard stats>)`.
- `src/sase/ace/tui/actions/_startup_history_words.py` owns the memory-only app cache, rebuilding off-thread only when
  the source token changes, and refreshes open history-word menus through `_refresh_visible_history_word_surfaces()`.
  Its disabled path assigns the literal token `(0, min_length, ())`.
- `src/sase/ace/tui/widgets/history_word_completion.py` builds rows from that cache; a cold cache renders one
  non-selectable `HistoryWordCompletionPlaceholder` row ("loading history words…").

### Common placeholders

- `src/sase/history/prompt_placeholders.py` owns the authoritative store at `~/.sase/prompt_placeholders.json`, guarded
  by a dedicated lock file, written atomically in display order, with LRU retention.
  `seed_common_placeholders_from_history()` runs only when the store file is absent.
- `src/sase/ace/tui/actions/_startup_common_placeholders.py` owns the memory-only app cache and a
  `_common_placeholders_generation` counter; `_publish_common_placeholders()` bumps that counter and refreshes open
  placeholder menus.
- `src/sase/ace/tui/widgets/_placeholder_highlight.py` memoizes `_placeholder_completion_at_cursor()` on
  `(cursor_offset, len(text), common_placeholders_generation)`, so the generation bump is what invalidates an open
  menu's memo.
- Rows carry `PlaceholderCompletionMetadata(source=...)`, where `source` is `"prompt"` or `"common"`.

## Design decisions

1. **History-word deletion needs a durable suppression store.** History words are re-derived from prompt history on
   every cold rebuild, so removing a row from the cache alone would resurrect the word on the next ACE start. Add a
   small durable store of deleted words that the derivation filters out. The alternative — rewriting prompt-history
   shard text — is destructive and rejected.

2. **Placeholder deletion is a plain store removal.** `prompt_placeholders.json` is authoritative, so removing the entry
   is sufficient. Writing that tag in a later prompt legitimately re-records it; that is the intended "learned from what
   you write" behavior and is not a bug. Because the store file still exists after the removal (even if it becomes
   empty), the one-time history seed never re-runs and cannot undo the deletion.

3. **Keep the keystroke path non-blocking.** Per `sase/memory/tui_perf.md` (rules 1 and 11), key handling must not do
   synchronous disk I/O or take a shared-store lock. Deletion therefore follows the established optimistic pattern:
   mutate the in-memory app cache and repaint the menu immediately, then persist off-thread through the helper in
   `src/sase/ace/tui/util/io_async.py`, which already toasts and rolls back on failure. The existing synchronous
   `file_history` delete is left as-is: it is a pre-existing path, changing it is not required by this feature, and its
   store has no lock.

4. **Rust core boundary.** `crates/sase_core/src/editor/placeholder.rs` in the sibling `sase-core` repo owns placeholder
   matching, ordering, and dedup, and takes the saved list as a caller-supplied `common: &[String]` slice. The durable
   stores themselves are Python-owned today. Deletion is a store mutation, not matching logic, so it belongs beside the
   existing Python store functions; no Rust change is needed.

5. **Only `common` placeholder rows are deletable.** A `prompt`-source row comes from the text the user is editing, and
   the Rust merge dedups a common entry away when the identical text also appears in the prompt. `Ctrl+D` on a
   `prompt`-source row is consumed and shows an explanatory toast rather than silently doing nothing, because the menu
   advertises the key. A saved tag that is currently shadowed by an identical prompt tag cannot be deleted from that
   view; this is an accepted limitation.

6. **Deleting the last row closes the menu.** When no candidates remain there is nothing to render, matching today's
   `file_history` behavior. "Do not close the menu" applies to every deletion that leaves at least one row.

7. **Show the affordance.** The panel subtitle advertises `[^D] delete` for the newly deletable kinds, matching the
   existing `file_history` subtitle. This changes three PNG snapshot goldens.

## Implementation

### 1. New durable store for deleted history words

Add `src/sase/history/prompt_word_deletions.py`, modeled on `src/sase/history/file_references.py` (atomic `tempfile` +
`os.replace`, no lock file; a concurrent second ACE instance may lose one deletion, which matches the existing
file-reference store and is acceptable for a user-initiated single-row action):

- Module constant `_STORE_VERSION = 1`; store path `sase_home() / "prompt_word_deletions.json"`; payload
  `{"version": 1, "words": [...]}` in oldest-to-newest order.
- `load_deleted_prompt_words() -> set[str]` — a missing, unreadable, corrupt, or version-mismatched file reads as an
  empty set. Matching is exact-spelling: history words are stored with their original case and `"Foo"` must not delete
  `"foo"`.
- `delete_prompt_word(word: str) -> bool` — appends `word` if absent, dedups, truncates the oldest entries beyond a
  module-level cap (`_MAX_DELETED_WORDS = 10_000`, to keep the file bounded), writes atomically, and returns whether the
  file changed. Raises on write failure so the caller's off-thread error path can surface it.
- `prompt_word_deletions_source_token() -> tuple[str, int, int]` — `(path, st_mtime_ns, st_size)` with a
  `(path, -1, -1)` sentinel for an absent file, mirroring `common_placeholder_source_token()`.

### 2. Filter deleted words out of history-word derivation

In `src/sase/history/prompt_words.py`:

- Load the deleted set once at the top of `collect_recent_prompt_words()` and skip those words **before** they count
  toward `max_words`, so deletions free budget for other words instead of leaving holes.
- Extend `history_words_source_token()` to append `prompt_word_deletions_source_token()`, and widen the
  `HistoryWordsSourceToken` alias to match, so a deletion invalidates a warm cache in every ACE process.
- Update the disabled-feature token literal in `src/sase/ace/tui/actions/_startup_history_words.py`
  (`(0, min_length, ())`) to the new arity.

### 3. Placeholder store removal

In `src/sase/history/prompt_placeholders.py`, add `remove_common_placeholder(text: str) -> bool`: take
`_locked_placeholder_store()`, load the payload, drop entries whose `text` matches exactly, save when something changed,
and return whether it did. Unlike `record_prompt_placeholders()`, this one is not best-effort — it must raise on failure
so the caller can toast and roll back. Do not re-rank: `_save_payload()` receives the already-ordered remainder.

### 4. Promote the off-thread persistence helper

In `src/sase/ace/tui/util/io_async.py`, rename `_schedule_persist` to `schedule_persist` (it now has a real cross-module
consumer, so Symvision's private-import rule requires a public name), delete the `_SCHEDULE_PERSIST_COMPAT_REFERENCE`
anchor that only existed to keep the unused private symbol alive, and update the module docstring and `_AppLike`
docstring references. Behavior is unchanged.

### 5. App-cache invalidation hooks

- `src/sase/ace/tui/actions/_startup_history_words.py` — add `forget_history_prompt_word(self, word: str) -> None`:
  filter the word out of `_history_prompt_words_cache` when that cache is a list, set
  `_history_prompt_words_source_token = None` so any later warm rebuilds unconditionally from disk, then call
  `_refresh_visible_history_word_surfaces()` so every open history-word menu (including other prompt panes) repaints.
- `src/sase/ace/tui/actions/_startup_common_placeholders.py` — add `forget_common_placeholder(self, text: str) -> None`:
  filter the text out of `_common_placeholders_cache`, set `_common_placeholders_source_token = None`, then call
  `_publish_common_placeholders()` to bump the generation counter (which invalidates the widget memo) and repaint open
  placeholder menus. Clearing the token means the next warm takes the first-warm path and calls
  `seed_common_placeholders_from_history()`, which is a no-op while the store file exists — note this in the method
  docstring.

Both methods are the rollback point too: on a failed persist the caller re-runs `warm_history_prompt_words()` /
`warm_common_placeholders()`, and the cleared source token guarantees a real reload rather than a token-equal skip.

### 6. Widget deletion dispatch

In `src/sase/ace/tui/widgets/_file_completion_accept.py`, restructure `_delete_selected_file_completion()` into a
dispatcher that resolves the selected candidate once, then routes by `self._completion_kind`:

- `"file_history"` — the existing body, unchanged apart from now emitting the shared toast.
- `HISTORY_WORD_COMPLETION_KIND` — return `False` when the selected row's metadata is a
  `HistoryWordCompletionPlaceholder` (the "loading history words…" row is not deletable). Otherwise take
  `selected.insertion` as the word, call the app's `forget_history_prompt_word` (guarded with `getattr` + `callable`,
  matching the existing `_history_prompt_words()` / `warm_history_prompt_words` accessors so lightweight test harnesses
  still work), toast, and `schedule_persist(...)` `delete_prompt_word` with `error_label="Deleting history word"` and an
  `on_error` that calls the app's `warm_history_prompt_words`.
- `PLACEHOLDER_COMPLETION_KIND` — when the selected row's metadata is not `PlaceholderCompletionMetadata` with
  `source == "common"`, consume the key with an information toast explaining that the tag comes from the current prompt
  rather than the saved list, and return `True` without touching the store. Otherwise call the app's
  `forget_common_placeholder`, toast, and `schedule_persist(...)` `remove_common_placeholder` with
  `error_label="Deleting saved placeholder"` and an `on_error` that calls the app's `warm_common_placeholders`.
- Any other kind — return `False`.

Add a small `_completion_supports_delete()` predicate on the same mixin returning whether `_completion_kind` is one of
the three deletable kinds, so kind knowledge stays in one module.

Because both cache-invalidation hooks already repaint every open menu, the deletion methods do not repaint directly;
they must, however, still handle an app that does not expose the hook (fall back to removing the row locally, clamping
`_file_completion_index`, and calling `_update_file_completion_panel("")` exactly as the `file_history` path does
today).

Add a shared private toast helper on the mixin that calls
`self.app.notify(message, severity="information", markup=False)` inside a `try`/`except` so a notification failure can
never break a deletion. `markup=False` matters: placeholder text can contain `[`, which Rich would otherwise parse as
markup.

### 7. Key handling

In `src/sase/ace/tui/widgets/_prompt_text_area_key_handling.py`, change the `ctrl+d` guard inside the
`if self._file_completion_active:` block from the `"file_history"` literal to `self._completion_supports_delete()`.
Other completion kinds must keep falling through to the app-level `scroll_detail_down` binding.

### 8. Panel affordance

In `src/sase/ace/tui/widgets/_prompt_input_bar_completion_panel.py`, extend the `border_subtitle` selection:

- `file_history` — unchanged (`[^L] accept  [^D] delete`).
- history word — `[^L] accept  [^D] delete`, but only when the visible rows are real candidates; suppress it while the
  single `HistoryWordCompletionPlaceholder` row is showing.
- placeholder — append `[^D] delete` to the existing source legend when both sources are visible, and show `[^D] delete`
  alone otherwise.

Keep the composition in one small helper so the three branches stay readable.

### 9. Help popup and documentation

- `src/sase/ace/tui/modals/help_modal/binding_common.py` — add an entry to `PROMPT_INPUT_SECTION` next to the existing
  `("Ctrl+L in panel", "Keep placeholder literal")` row, for example
  `("Ctrl+D in panel", "Delete saved completion entry")`. Keep the description within the 32-character help-modal limit
  and verify the row renders inside the 57-character box.
- `docs/ace.md`, prompt-completion section:
  - Add `Ctrl+D` to the completion key table (currently `Ctrl+T`, `Ctrl+N`/`Down`, `Ctrl+P`/`Up`, `Enter`/`Ctrl+L`,
    `Escape`), described as deleting the highlighted entry from recent files, saved placeholders, or history words.
  - Extend the **Placeholder completion** bullet: `Ctrl+D` deletes the highlighted saved (`◆`) placeholder from the
    store; current-prompt (`<>`) rows are not deletable.
  - Extend the **History-word completion** bullet: `Ctrl+D` deletes the highlighted word and records it durably so it is
    filtered out of future derivations; name the store file so a user can reset it.

## Testing

1. `tests/history/test_prompt_word_deletions.py` (new): round-trip, idempotent re-deletion, exact-case matching,
   missing/corrupt/version-mismatched file reads as empty, cap eviction drops the oldest entries, and the source token
   changes after a write.
2. `tests/history/test_prompt_words.py`: deleted words are excluded from `collect_recent_prompt_words()`, an excluded
   word does not consume `max_words` budget, and `history_words_source_token()` changes when the deletion store changes.
3. `tests/history/test_prompt_placeholders.py`: `remove_common_placeholder()` removes exactly the matching entry,
   returns `False` for an absent text, preserves display order and counts for the rest, and leaves the store file
   present after removing the last entry so seeding cannot re-run.
4. Widget tests alongside `tests/ace/tui/widgets/test_history_word_completion.py` and
   `tests/ace/tui/widgets/test_placeholder_completion.py` (a new focused module is fine): `Ctrl+D` on a history-word row
   removes the row, keeps the menu open, and toasts; `Ctrl+D` on the "loading history words…" row is a no-op; `Ctrl+D`
   on a saved placeholder row removes it and keeps the menu open; `Ctrl+D` on a prompt-source placeholder row leaves the
   store untouched and toasts the explanation; deleting the last remaining row closes the menu; `Ctrl+D` on a
   non-deletable kind is not consumed.
5. `tests/ace/tui/test_history_words_cache.py` and `tests/ace/tui/test_common_placeholders_cache.py`: the new `forget_*`
   hooks prune the cache, clear the source token, bump the placeholder generation, and repaint open menus; a subsequent
   warm reloads from disk.
6. Extend the shared `tests/ace/tui/widgets/_completion_helpers.py` app with the new hooks (recording calls) so widget
   tests can assert both the "app exposes the hook" and "app does not" paths.
7. Visual: the panel subtitle change affects
   `tests/ace/tui/visual/snapshots/png/history_word_completion_panel_120x40.png`,
   `placeholder_completion_panel_120x40.png`, and `placeholder_common_completion_panel_120x40.png`. Run
   `just test-visual`, inspect the diffs in `.pytest_cache/sase-visual/`, confirm the only change is the new subtitle,
   then accept with `--sase-update-visual-snapshots`.

## Validation

Read `sase/memory/tui_perf.md` through the `sase memory read` workflow before touching the TUI paths, then:

```bash
just install
pytest -q \
  tests/history/test_prompt_words.py \
  tests/history/test_prompt_word_deletions.py \
  tests/history/test_prompt_placeholders.py \
  tests/ace/tui/widgets/test_history_word_completion.py \
  tests/ace/tui/widgets/test_placeholder_completion.py \
  tests/ace/tui/test_history_words_cache.py \
  tests/ace/tui/test_common_placeholders_cache.py
just test-visual
just check
```

`just check` covers ruff, mypy, `toobig`, and Symvision. Two Symvision points are load-bearing here: the promoted
`schedule_persist` must have a real non-test consumer (the widget deletion path), and every new public store function
must be reachable from non-test code.

## Acceptance criteria

- `Ctrl+D` in the prompt completion menu deletes the highlighted history word and the highlighted saved (`◆`)
  placeholder, and still deletes the highlighted recent-file entry.
- Every successful deletion shows a toast naming the deleted entry.
- After a deletion that leaves at least one candidate, the menu stays open with the row gone and a sane selection; the
  menu closes only when no candidates remain.
- A deleted history word does not reappear after the app cache rebuilds or after ACE restarts.
- A deleted saved placeholder does not reappear until the user writes that tag in a prompt again.
- `Ctrl+D` on the "loading history words…" row does nothing; on a current-prompt (`<>`) placeholder row it toasts an
  explanation and leaves the store untouched; on every non-deletable completion kind it still reaches the app-level
  scroll binding.
- No synchronous store write or store lock happens on the key-handling path for the two new lists; the durable write
  runs off the event loop, and a write failure toasts an error and restores the entry in the cache.
- The panel subtitle advertises `[^D] delete` for the deletable kinds, the `?` help popup lists the new binding, and
  `docs/ace.md` documents it.
- Focused tests, the visual snapshot suite, and `just check` pass.
