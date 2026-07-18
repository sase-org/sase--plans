---
tier: tale
title: History-word fallback completion for the prompt input
goal: 'When manual Ctrl+T word completion finds no matches inside the current prompt,
  the prompt input opens a second completion menu of recently used words derived from
  prompt history, with the retained word count and minimum word length configurable
  through new ace.prompt_completion config fields.

  '
create_time: 2026-07-18 14:38:05
status: wip
prompt: 202607/prompts/history_word_completion.md
---

# Plan: History-word fallback completion menu

## Product context

The ACE prompt input recently gained prompt-local word completion: the manual `Ctrl+T` completion chain in
`FileCompletionOpenMixin._try_file_completion_tab` (`src/sase/ace/tui/widgets/_file_completion_open.py`) ends at
`_try_prompt_word_completion_tab`, which completes a plain word prefix from words already present in the active pane
(`src/sase/ace/tui/widgets/prompt_word_completion.py`). When no other pane word matches the prefix,
`build_prompt_word_completion_result` returns `None`, the menu clears, and the keypress does nothing.

This plan fills that dead end: when prompt-local word completion fails, `Ctrl+T` opens a **history words** menu whose
candidates are the last `N` unique words (of at least `M` characters) the user typed in previously recorded prompts.
Defaults: `N = 1000`, `M = 5`, both user-configurable.

## Behavior specification

- **Trigger.** Unchanged entry point: `Ctrl+T` on a plain prose token (not xprompt, directive, path, Jinja, placeholder,
  or VCS trigger). Prompt-local word completion keeps absolute precedence; the history menu opens only when the local
  result is `None` and the cursor has a non-empty word prefix to its left. When the prefix matches nothing in history
  either, `Ctrl+T` behaves exactly as today (no menu).
- **Matching.** Identical word semantics to prompt-local completion (maximal runs of Unicode alphanumerics/underscore)
  and identical case-insensitive (`casefold`) prefix matching. Candidates equal to the complete word already around the
  cursor are excluded so accepting always changes the text.
- **Ordering.** Most-recently-used first. Unlike the alphabetical prompt-local menu, this list is _defined_ by recency,
  and with up to 1000 words a prefix can match many candidates; the words the user typed most recently are the likeliest
  picks.
- **Menu presentation.** The shared completion panel (`PromptInputBarCompletionMixin.show_file_completions`) with a new
  completion kind `history_word` and border title `history words`. Rows render exactly like prompt-local word rows
  (plain word, bold when selected, no icon) via the existing `append_prompt_word_completion_row` helper — the two word
  menus stay visually consistent, and the border title tells them apart.
- **Accept semantics.** Identical to prompt-local words: accepting replaces the whole word around the cursor; a single
  candidate auto-accepts immediately; a shared case-insensitive extension first extends the typed prefix and re-filters
  (reusing the same shared-extension guard that handles casefold length changes such as `ß` → `ss`).
- **Live refresh.** While the history menu is open, edits and cursor moves refresh it in
  `FileCompletionRefreshMixin._refresh_file_completion_from_cursor`
  (`src/sase/ace/tui/widgets/_file_completion_refresh.py`):
  - `_structured_completion_claims_cursor()` dismisses it, same as prompt-local words.
  - If prompt-local completion now yields a result (e.g. backspacing widened the prefix to match a pane word), the menu
    _switches_ to the prompt-local kind — the "local words always outrank history words" invariant holds at all times.
  - Otherwise re-filter history words against the new prefix; zero matches dismisses.
- **Cold cache.** If `Ctrl+T` reaches the fallback before the word list has been loaded, show a single non-selectable
  dim placeholder row (`loading history words…`) and schedule the off-thread load; when it completes, re-filter into
  real rows (or dismiss if nothing matches), mirroring the VCS repo menu's loading-placeholder mechanics.
- **Disablement.** `history_word_count: 0` disables the feature entirely; `Ctrl+T` behaves exactly as before this
  change.

## Design decisions

1. **Derive words from the existing prompt-history store; do not persist a second word store.** The word list is a pure
   function of the sharded prompt-history store (`src/sase/history/prompt_store.py`), computed as "walk entries
   newest-first, take unique words until `N`". Rationale:
   - No dual-write consistency problems, no migration, no doctor/prune support for a new file.
   - Deleting or pruning prompts (a privacy action) automatically removes their words.
   - Changing `N`/`M` in config takes effect on the next rebuild — no stale store built with old parameters.
   - Existing history is "backfilled" for free.
2. **MRU ordering, exact-spelling uniqueness.** Unique by exact spelling (consistent with prompt-local completion, which
   deliberately offers distinct spellings), matched case-insensitively. All-digit runs (PR numbers, timestamps) are
   excluded as noise. Cancelled history entries are _included_: they are still words the user typed, and failed-launch
   prompts are recorded as cancelled despite being real submissions.
3. **Shared word-scanning primitives.** Promote the private word-run helpers in `prompt_word_completion.py`
   (`_word_range_at_cursor`, `_word_ranges`, `_is_word_character`, `_shared_extension`) to public module functions
   consumed by both the prompt-local and history modules, so tokenization semantics can never drift apart (and Symvision
   sees no cross-module private use).
4. **This stays in this repo.** The prompt-history store is Python-local here; the feature is TUI presentation plus a
   derivation over that local store, with no cross-frontend behavior contract, so no `sase-core` (Rust) changes are
   needed.

## Technical design

### 1. Word derivation: `src/sase/history/prompt_words.py`

New module beside the other prompt-history readers:

- `extract_prompt_words(text, *, min_length)` — yield the word-character runs of one prompt in source order, keeping
  runs with `len >= min_length` that are not all digits. Uses the shared word-scanning helpers from decision 3.
- `collect_recent_prompt_words(*, max_words, min_length) -> list[str]` — iterate shards newest-first
  (`iter_shard_paths_newest_first` + `load_shard`, entries sorted by `last_used` descending, matching how
  `load_all_prompt_history` orders them), collect unique spellings in first-seen (most recent) order, and stop early
  once `max_words` is reached so huge histories stay bounded. Corrupt shards are already masked by `load_shard`.
- `history_words_source_token(*, max_words, min_length)` — a cheap staleness token: `(path, mtime_ns, size)` per shard
  file plus the two config values. Used to decide whether a cached list must be rebuilt (perf memory rule: cache disk
  reads keyed by mtime).

### 2. App-level cache and off-thread warming

A small app mixin (e.g. `src/sase/ace/tui/actions/_startup_history_words.py`, wired like `StartupPromptCatalogMixin`):

- State: warm `list[str] | None`, its source token, and an in-flight/coalescing guard (last-request-wins, mirroring the
  prompt-catalog rebuild guards).
- `history_prompt_words() -> list[str] | None` — return the warm list (never touches disk); `None` while cold.
- `warm_history_prompt_words()` — schedule an off-thread build (`run_worker`/ `asyncio.to_thread`): compute the source
  token, skip if unchanged, otherwise call `collect_recent_prompt_words` with the configured `N`/`M` and publish the
  result on the UI task.
- Warm trigger: the prompt text area's existing mount-time warm hook (`_warm_vcs_project_completion_catalog` is called
  from `prompt_input_bar.py` and `_prompt_input_bar_stack_rendering.py`) gains a sibling call for history words, gated
  the same way (`getattr(self.app, ...)` capability check so lightweight test harnesses skip it). Because warming
  re-checks the token on every prompt-bar mount, prompts recorded earlier in the session (which change shard mtimes) are
  picked up for the next prompt without any explicit invalidation hook. Startup first-paint is untouched (no O(history)
  work before the stopwatch ends).

### 3. Completion menu wiring

New module `src/sase/ace/tui/widgets/history_word_completion.py`:

- `HISTORY_WORD_COMPLETION_KIND = "history_word"`.
- `build_history_word_completion_result(text, cursor_offset, words)` — same result shape as the prompt-local builder
  (prefix, replacement range, MRU-ordered candidates, shared extension), excluding the current whole word.
- `build_loading_history_words_placeholder()` — the dim non-selectable loading row (modeled on
  `build_loading_placeholder` in `vcs_repo_completion.py`).

Wiring changes, all modeled on existing per-kind branches:

- `_file_completion_open.py`: `_try_prompt_word_completion_tab` falls through to a new
  `_try_history_word_completion_tab(cursor_offset)` when the local result is `None`. It is gated on
  `history_word_count > 0`, reads the warm list from the app, handles cold-cache placeholder + scheduling, and otherwise
  reuses the local menu's single-candidate / shared-extension / open logic.
- `_file_completion_base.py`: worker scheduling + result application for the cold-cache load, mirroring
  `_schedule_vcs_repo_completion_fetch` / `_apply_vcs_repo_completion_result` (apply only if the menu is still open on
  the history kind; re-filter at the current cursor).
- `_file_completion_refresh.py`: a `history_word` refresh branch implementing the switch-to-local rule, the
  structured-claims dismissal, and prefix re-filtering.
- `_file_completion_accept.py`: ignore the loading placeholder row (same pattern as the non-selectable VCS
  placeholders).
- `_prompt_input_bar_completion_panel.py`: recognize the new kind — border title `history words`, rows via
  `append_prompt_word_completion_row`, placeholder rows dim italic.

### 4. Configuration

Two new fields under `ace.prompt_completion` (flat, matching the sibling fields):

| Field                     | Type | Default | Meaning                                                                       |
| ------------------------- | ---- | ------- | ----------------------------------------------------------------------------- |
| `history_word_count`      | int  | `1000`  | Unique recent words retained from history; 0 disables the history-words menu. |
| `history_word_min_length` | int  | `5`     | Minimum word length (characters) to retain; clamped to >= 1.                  |

Plumbing (each mirrors the existing fields):

- `PromptCompletionSettings` + `parse_prompt_completion_settings` in `src/sase/ace/tui/widgets/prompt_completion.py`
  (conservative fallbacks on bad values, like the sibling parsers).
- `src/sase/default_config.yml` (`ace.prompt_completion`).
- `src/sase/config/sase.schema.json` (`prompt_completion` properties).
- `docs/configuration.md` — the `ace.prompt_completion` YAML block, field table, and a short prose note about the
  history-words fallback.

### 5. Documentation and help

- `docs/ace.md` — extend the manual-completion section (rewritten by the prompt-local word completion change) to
  document the full fallback chain: structured providers → prompt-local words → history words, including the config
  knobs and disablement.
- Help modal — audit the `?` popup content for completion descriptions; the current `Ctrl+T` line ("Manual completion /
  accept") likely needs no change, but keep the help popup in sync per the ace guidelines if any box enumerates the
  fallback chain.

## Testing

- **Derivation unit tests** (`tests/history/test_prompt_words.py`): extraction semantics (word runs, underscore,
  Unicode), `min_length` filtering, all-digit exclusion, exact-spelling uniqueness, MRU ordering across multiple shards,
  early stop at `max_words`, cancelled entries included, corrupt shard tolerance, and the source token changing on shard
  writes and config changes.
- **Widget tests** (`tests/ace/tui/widgets/test_history_word_completion.py`, modeled on
  `test_prompt_word_completion.py`): opens only when prompt-local completion fails; structured providers keep
  precedence; casefold prefix filtering; MRU candidate order; current-word exclusion; single-candidate auto-accept;
  shared-extension advance; refresh narrowing, switch-back-to-local, and dismissal; cold-cache placeholder followed by
  async application of the loaded list; placeholder row not acceptable; feature fully inert at `history_word_count: 0`.
- **Settings tests**: extend the existing `parse_prompt_completion_settings` coverage for the two new fields (defaults,
  clamping, garbage input).
- **PNG snapshot** (`tests/ace/tui/visual/`): a new `test_ace_png_snapshots_history_word_completion.py` plus golden
  (`history_word_completion_panel_120x40.png`), modeled on the prompt-word snapshot test (drive `show_file_completions`
  with the new kind and representative rows).
- Run `just check` (and `just test-visual` for the new golden) before finishing.

## Performance and reliability constraints

Per `sase/memory/tui_perf.md` (read during design):

- The `Ctrl+T` key handler only ever reads the warm in-memory list; all disk reads and JSON parsing run in background
  workers.
- The cache is keyed by a shard mtime/size token — no per-keypress stats, no rebuilds when nothing changed.
- Warming happens at prompt-bar mount (off the startup path and off the keystroke path), following the established
  `_warm_vcs_project_completion_catalog` precedent.
- The shard walk early-stops at `N` unique words, bounding work for very large histories.

## Risks and edge cases

- **Empty or missing history** (fresh installs): the fallback silently yields nothing; `Ctrl+T` behaves exactly as
  today.
- **Casefold length changes** (`ß` → `ss`): reuse the existing shared-extension guard from the prompt-local module.
- **Multi-pane prompt stacks**: the word list is app-global; per-pane menu state is already handled by the existing
  mixins, so no pane-specific logic is needed.
- **Concurrent history writers** (other agents recording prompts mid-session): the token check at the next prompt-bar
  mount picks up changes; a slightly stale list mid-session is acceptable and self-heals.
- **Symvision**: new public helpers must all have consumers (the promoted word-scanning helpers are consumed by both
  completion modules; the derivation API by the app mixin and tests).

## Out of scope

- Frequency-based ("most common") ranking — the store is recency-defined per the feature request; frequency weighting
  can layer on later without changing storage.
- Auto (soft) completion integration — history words remain manual-`Ctrl+T`-only, matching prompt-local words.
- A persisted word store or new CLI surfaces (`sase history words …`) — not needed for the TUI feature and easy to add
  later on top of `collect_recent_prompt_words`.
