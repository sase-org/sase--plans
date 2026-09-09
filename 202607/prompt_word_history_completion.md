---
tier: tale
title: History-derived common words in prompt word completion
goal: "The prompt input's Ctrl+T word completion menu offers commonly used words
  recorded from previous prompts beneath the prompt-local matches, with a clear visual
  distinction between the two sources and a configurable history size.

  "
create_time: 2026-09-09 19:53:20
status: wip
---

# Plan: History-derived common words in prompt word completion

## Product context

The ACE prompt input recently gained prompt-local word completion
(`feat(ace): add prompt-local word completion`): as the final `Ctrl+T` fallback for a
plain prose token, the menu completes the word prefix left of the cursor from words
already present in the active prompt. This plan extends that menu with a second
candidate source: the user's _commonly used words_, derived from historical prompts.

Requirements:

- Common words are not static. They are the last `<N>` unique words of five or more
  characters that appeared in previously submitted prompts, with `<N>` defaulting to
  1000 and overridable through sase configuration.
- History-sourced words render **beneath** any prompt-local words in the menu.
- The menu must visually indicate which rows came from the current prompt text and which
  came from the common-words history.
- The result should be intuitive, reliable, and beautiful.

## Design overview

Three cooperating pieces, all following established patterns in this repo:

1. **A rolling word-history store** in `src/sase/history/prompt_words.py`, modeled
   directly on `src/sase/history/file_references.py` (the store that already feeds the
   `Ctrl+T` "recent files" menu). Recording happens at the same prompt submit/cancel
   call sites that already record file references.
2. **A configuration field** `ace.prompt_completion.word_history_size` (default 1000,
   `0` disables the feature) threaded through the existing `PromptCompletionSettings`
   plumbing.
3. **A merged completion result**: `build_prompt_word_completion_result()` in
   `src/sase/ace/tui/widgets/prompt_word_completion.py` accepts the history snapshot,
   appends history candidates beneath prompt-local ones, and tags every candidate with
   its source so the panel can render the distinction.

The prompt-history package (`src/sase/history/`) is the right home for the store: prompt
history is Python-side today, and the file-reference history is the direct precedent. No
Rust-core (`sase_core_rs`) changes are needed; a future migration of the history package
would carry this store along.

## 1. Word-history store (`src/sase/history/prompt_words.py`)

**File format.** `sase_home() / "prompt_word_history.json"` containing
`{"words": ["most_recent", ...]}` — recency-ordered, most recently used word first,
original spelling preserved. Atomic writes (tmp file + `os.replace`), silent no-op on
`OSError`, empty result on missing/corrupt files — exactly the `file_references.py`
behavior.

**Word semantics.** A word is a maximal run of Unicode alphanumerics or underscores —
the same definition the prompt-word completion widget already uses. To keep exactly one
canonical tokenizer, move the character predicate and word-range scanner into this
history module (e.g. `iter_prompt_words()` / `is_prompt_word_character()`) and have
`src/sase/ace/tui/widgets/prompt_word_completion.py` import them from here. The
dependency direction (TUI → history) matches the rest of the codebase; history must not
import TUI modules.

**Recordable words.** From submitted prompt text, keep words that are at least five
characters long and are not purely digits (timestamps and issue numbers are noise). The
five-character floor is fixed by requirements, not configurable.

**API.**

- `extract_recordable_prompt_words(text) -> list[str]` — recordable words in prompt
  order.
- `record_prompt_words(text, *, max_words) -> None` — extract, prepend so the most
  recently used word lands at index 0, dedup **case-insensitively** with the newest
  spelling winning, truncate to `max_words`, atomically save. `max_words <= 0` is a
  no-op (feature disabled).
- `load_prompt_words() -> list[str]` — recency-ordered history. Cache the parsed list in
  a module-level cache keyed by the file's `(mtime_ns, size)` so repeated loads do not
  re-read disk (per the tui_perf rule: cache disk reads keyed by mtime). Test seams
  mirror `file_references.py`'s module-level `_HISTORY_FILE` override.

**Recording call sites.** Record words at the same three places that record file
references today, keeping submit and cancel behavior symmetrical:

- `record_prompt_file_references()` in
  `src/sase/ace/tui/actions/agent_workflow/_launch_history.py`, called from
  `_launch_body_impl.py` and `_launch_body_single.py` — generalize this helper (or add a
  sibling called alongside it at both call sites) so a submitted prompt records both
  file references and words in one call.
- `_save_text_as_cancelled()` in
  `src/sase/ace/tui/actions/agent_workflow/_prompt_bar_mount.py` — the cancel safety
  net; cancelled text was still typed by the user and its vocabulary is worth keeping.

The recording helpers read `word_history_size` from the app's parsed prompt completion
settings (`get_prompt_completion_settings()`); when it is `0`, recording is skipped
entirely. The write is a small bounded JSON file (at most `<N>` short strings) on the
same submit/cancel paths that already perform history writes, so no new threading is
required.

## 2. Configuration field

- `src/sase/default_config.yml`: add `word_history_size: 1000` under
  `ace.prompt_completion` with a short comment (`0` disables history-word completion and
  recording).
- `PromptCompletionSettings` in `src/sase/ace/tui/widgets/prompt_completion.py`: add
  `word_history_size: int = 1000`; parse it in `parse_prompt_completion_settings()` with
  the existing `_parse_non_negative_int` helper (invalid values fall back to 1000, `0`
  is respected as "off").
- No new plumbing is needed beyond the field: settings are already parsed in
  `src/sase/ace/tui/actions/_state_init_late.py` and exposed through
  `get_prompt_completion_settings()`
  (`src/sase/ace/tui/actions/_startup_prompt_catalog.py`), which both the prompt text
  area and the recording call sites can reach.

## 3. Merged completion result

Extend `build_prompt_word_completion_result(text, cursor_offset)` with a
`history_words: Sequence[str]` parameter (default empty, preserving all current behavior
when nothing is passed):

- **Prompt-local candidates** are built exactly as today: case-insensitive prefix match,
  original spellings, sorted casefolded-alphabetically.
- **History candidates** follow beneath: prefix-matched case-insensitively against the
  same cursor prefix, preserving the store's recency order (most recent first — the
  words you used last surface first). Exclude the current word under the cursor and any
  word that casefolds equal to a prompt-local candidate (the prompt-local spelling
  wins), so no word ever appears twice.
- The result is non-`None` when **either** group has matches. This deliberately widens
  today's gate: with a blank surrounding prompt, `Ctrl+T` on a prefix now opens a
  history-only menu instead of doing nothing.
- `shared_extension` is computed across the merged candidate list so the existing Tab
  longest-common-extension and single-candidate auto-accept behaviors keep working
  unchanged.
- **Source tagging:** add a small frozen metadata dataclass (e.g.
  `PromptWordCompletionMetadata(source: Literal["prompt", "history"])`) attached via the
  existing `CompletionCandidate.metadata` field — the same mechanism every other
  richly-rendered provider uses.

### Widget integration (open + refresh)

- `_try_prompt_word_completion_tab()` in
  `src/sase/ace/tui/widgets/_file_completion_open.py`: when `word_history_size > 0`,
  call `load_prompt_words()` (menu opening is an explicit user action, and the load is
  mtime-cached), stash the snapshot on the text area, and pass it to the builder.
- `_refresh_prompt_word_completion()` in `_file_completion_refresh.py`: pass the stashed
  snapshot instead of touching the store, so per-keystroke refreshes of the active menu
  never stat or read disk (tui_perf rules 8/11: keystroke paths are read-only, render
  paths never stat). Clear the snapshot wherever the completion state is cleared.
- Structured-provider precedence is untouched: `_structured_completion_claims_cursor()`
  still dismisses the word menu whenever a structured provider claims the cursor.

## 4. Visual design

Goals: the two groups must read at a glance, the common (prompt-local-only) case must
stay exactly as clean as it is today, and the treatment must follow the panel's existing
visual language (quiet dim badges, selection bold).

- **Row rendering** (`append_prompt_word_completion_row` in
  `_prompt_input_bar_completion_rows.py`, driven by the candidate metadata):
  - When the visible menu contains **only prompt-local rows**, rendering is unchanged —
    plain word, bold when selected. No churn for the existing experience or its PNG
    golden.
  - When **history rows are present**, every row gets a two-cell source gutter so the
    word column stays perfectly aligned: history rows show a dim-cyan `↺ ` recall glyph;
    prompt-local rows show a blank gutter (their words remain plain/bold). The glyph
    marks the _exception_ (recalled words), which keeps the panel quiet.
- **Legend:** while history rows are visible, set the panel `border_subtitle` to a dim
  `↺ history` so the glyph is self-explanatory the first time it appears. The panel
  title stays `prompt words`.
- The panel's existing scrolling (`MAX_VISIBLE`, `↓ N more…`) needs no changes; with up
  to 1000 stored words a short prefix may match many rows, and the scroll affordance
  already handles that.

The row helper will need the metadata-based branching plus the "any history row visible"
flag; thread that from `show_file_completions()` in
`_prompt_input_bar_completion_panel.py`, mirroring how other providers pass label
widths.

## 5. Documentation

- `docs/ace.md`: update the _Prompt-local word completion_ bullet in the Completion
  section (and the `Ctrl+T` key-table blurb) to describe the history source, its
  ordering, the `↺` glyph, and the `ace.prompt_completion.word_history_size` knob;
  mention `0` as the opt-out.
- `src/sase/default_config.yml` comment doubles as the config reference.

## 6. Testing

- **Store unit tests** (`tests/history/test_prompt_words.py`, mirroring
  `test_file_references.py`): extraction (five-char floor, underscore words, all-digit
  exclusion), recency ordering, case-insensitive dedup with newest-spelling-wins,
  `max_words` truncation, `max_words=0` no-op, missing and corrupt files, atomic
  overwrite, mtime-cache invalidation after a write.
- **Builder unit tests** (extend
  `tests/ace/tui/widgets/test_prompt_word_completion.py`): merged ordering (local block
  sorted, history block recency-ordered beneath), cross-source dedup, current-word
  exclusion, history-only results, shared extension across the merged list, source
  metadata on every candidate, empty `history_words` preserving today's behavior
  byte-for-byte.
- **Widget interaction tests**: `Ctrl+T` opens a history-backed menu with a blank
  surrounding prompt; refresh while typing narrows both groups without reloading the
  store (assert via the stashed-snapshot seam); accepting a history candidate replaces
  the whole word under the cursor; structured providers still take precedence;
  `word_history_size=0` restores the exact pre-feature behavior.
- **Settings tests**: `parse_prompt_completion_settings` coverage for the new field
  (default, override, `0`, invalid values).
- **PNG visual snapshot**: a new golden alongside
  `test_ace_png_snapshots_prompt_word_completion.py` showing a mixed menu — prompt-local
  rows on top, `↺`-badged history rows beneath, subtitle legend — plus keeping the
  existing local-only golden green to prove the unchanged base case.

## 7. Risks and edge cases

- **Keystroke-path performance**: the design keeps all disk access on the explicit
  menu-open action and caches by mtime; refresh uses the in-memory snapshot. This is the
  main regression risk and is covered by the snapshot-reuse test.
- **Menu-open behavior change**: `Ctrl+T` on a prefix with no prompt-local matches
  previously did nothing; it now opens a history menu. This is the intended product
  change; the `word_history_size=0` escape hatch restores the old behavior.
- **Concurrent writers**: multiple ACE instances may append concurrently;
  last-writer-wins on a recency list of vocabulary words is acceptable and matches the
  file-reference store's existing stance.
- **Privacy/noise**: the store lives under `sase_home()` next to the existing prompt
  history (which already stores full prompt text, so no new exposure); the
  five-character floor plus digit filtering keeps obvious noise out.
