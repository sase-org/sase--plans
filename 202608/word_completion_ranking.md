---
tier: epic
title: Rank Ctrl+T history words by relation, recency, and frequency
goal: "The Ctrl+T history-word menu ranks candidates by how strongly they relate to the
  words already in the prompt, how recently they were used, and how often they were
  used, and every row shows a compact, colored signal explaining why it ranks where it
  does.

  "
phases:
  - id: index
    title: Prompt-word corpus index
    depends_on: []
    size: medium
    description: "index: build the immutable prompt-word corpus index (word stats,
      prompt co-occurrence postings, case-folded prefix lookup) behind a per-shard
      tokenization cache, and re-express the existing MRU word list on top of it without
      changing its output.

      "
  - id: ranking
    title: Relation, recency, and frequency scoring
    depends_on:
      - index
    size: medium
    description: "ranking: add the pure scoring engine that turns the corpus index plus
      the current prompt text into ranked words with per-signal contributions, a
      dominant reason, and the evidence each row displays.

      "
  - id: wiring
    title: Warm cache, menu, and settings wiring
    depends_on:
      - ranking
    size: medium
    description: "wiring: hold the index in the app-global warm cache, apply deletions
      at query time, feed ranked candidates into the history-word menu, order
      prompt-local words nearest-first, and add the two ranking settings.

      "
  - id: signals
    title: Ranking signals in the completion panel
    depends_on:
      - wiring
    size: medium
    description:
      "signals: render the stacked score meter, dominant-reason chip, and colored panel
      legend for history-word rows, degrade cleanly on narrow panels, and refresh the
      docs and PNG goldens."
proposed_by: bbugyi200.athena.03s.w0
status: done
bead_id: sase-na
create_time: 2026-09-09 19:52:05
---

- **PROMPT:**
  [prompts/202608/word_completion_ranking.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/word_completion_ranking.md)
- **BEAD:**
  [sase-na](https://github.com/sase-org/sase--beads/blob/main/pages/sase-na/README.md)

# Plan: Rank Ctrl+T history words by relation, recency, and frequency

## Problem

`Ctrl+T` falls back to prompt-history words (`history_word` menu) once prompt-local
words have no match. Today `collect_recent_prompt_words()`
(`src/sase/history/prompt_words.py`) walks every shard newest-first and returns the
first `history_word_count` distinct spellings it sees, and
`build_history_word_completion_result()`
(`src/sase/ace/tui/widgets/history_word_completion.py`) preserves exactly that order
after a case-insensitive prefix filter. So the menu is ordered purely by "most recently
written", which has three practical failures:

1. A one-off token from the newest prompt (a typo, a pasted identifier, a hash-like
   word) outranks a word written hundreds of times.
2. Nothing in the ordering knows what the current prompt is about, even though the
   prompt is sitting right there in the widget.
3. The panel shows at most 8 content rows (7 once the `↓ N more…` line appears), so a
   bad first-7 is the whole user experience.

The rows also carry no explanation, so a user cannot tell whether a row is on top
because it is fresh, because it is common, or because it fits the sentence.

The prompt-local provider has a smaller version of the same problem:
`build_prompt_word_completion_result()` sorts its candidates alphabetically
(`ordered = sorted(spellings, key=...)`), which ignores the strongest local signal there
is — how close the word is to the cursor.

## Design

### Ranking model

Every history-word candidate gets three normalized signals in `[0, 1]` and a weighted
composite score:

```
score = 0.50 * relation + 0.30 * recency + 0.20 * frequency
```

The weights are the whole ranking policy, expressed once as module constants. Relation
is deliberately the largest single term (the user's explicit ask), while recency plus
frequency together can still outrank a weak relation, so an unrelated word that was
written 20 minutes ago and 300 times does not fall behind a word that co-occurred once,
a year ago.

**Frequency** saturates so a runaway word cannot flatten the scale:

```
frequency(w) = log1p(df(w)) / log1p(max_df)
```

where `df(w)` is the number of distinct prompts containing `w` and `max_df` is the
largest `df` in the index. On the author's real store (5,874 prompts, 9,957 distinct
words at `word_min_length: 5`) this maps `df=1 → 0.08`, `df=10 → 0.28`, `df=100 → 0.55`,
`df=4603 → 1.00`.

**Recency** is an exponential decay of the word's newest use:

```
recency(w) = 0.5 ** (age_seconds(w) / RECENCY_HALF_LIFE_SECONDS)   # 7 days
```

Age is computed against a `now` captured once per ranking call, not baked into the
index, so a long-lived ACE session does not drift into stale recency.

**Relation** answers "does this word show up in prompts like the one I am writing?" It
is a shrunk, capped lift (normalized PMI) rather than a raw co-occurrence count, because
raw counts just re-elect the globally common words that the frequency term already
covers:

1. Context words `C` are the distinct index words appearing anywhere in the prompt text,
   minus the identifier-like word the cursor sits in. Matching is case-insensitive
   against the index's folded spellings.
2. Drop context words with `df(c) / N > 0.20` (they carry no information) and keep the
   `24` rarest of the rest, so cost is bounded by the most informative words.
3. Weight each prompt that contains a context word by that word's inverse document
   frequency, then decay it by prompt age:
   `prompt_weight(p) = (Σ idf(c) for c in C ∩ p) * 0.5 ** (age(p) / 30 days)`.
4. Keep the `250` heaviest prompts; call their total weight `T`.
5. Accumulate `mass(w) = Σ prompt_weight(p)` and `hits(w)` over the words of those
   prompts — one pass over the related prompts, _not_ one pass per candidate.
6. Convert mass into association:

```
observed(w)   = mass(w) / T
background(w) = df(w) / N
lift(w)       = observed(w) / background(w)
relation(w)   = min(1, log2(lift) / log2(8)) * hits(w) / (hits(w) + 2)   # 0 if lift <= 1
```

The `log2(lift) / log2(8)` term means "8× more likely here than in an average prompt"
saturates the signal, and the `hits / (hits + 2)` shrinkage keeps a word that appeared
in exactly one related prompt from claiming a perfect score. Relation is forced to `0`
for every word when the store has fewer than `8` prompts, where the statistics are
meaningless.

Sorting is `(-score, folded_spelling, spelling)` so ties are deterministic and tests are
stable.

### Why this shape is fast enough

Two costs matter, both measured against the real store described above (5,874 prompts,
3.8 MB, 98,252 word/prompt postings):

- **Per menu open**: building the related-prompt set (2.2 ms) plus the single pass over
  the related prompts (2.1 ms). The context is memoized on the index and keyed by the
  set of context word ids, so typing more characters of the word under the cursor —
  which does not change the context — reuses it.
- **Per keystroke**: a `bisect` prefix range over a case-folded sorted word array
  (measured 0.003 ms per lookup versus 1.9 ms for the current full scan), then a score
  lookup and a sort over the matched slice. Under 1 ms.

Both stay far inside the 16 ms key-to-paint budget in `sase/memory/tui_perf.md`, and all
disk work stays in the existing off-thread warm path (rule 1 and rule 11: the keystroke
path stays read-only, allocation-light, and prompt-free).

The index build itself gets _cheaper_ than today's scan. The current derivation
tokenizes with the pure-Python `word_ranges()` character loop, which takes ~500 ms over
the real store. `re.compile(r"[\w\-]+")` is exactly equivalent to the
`is_word_character()` predicate — verified over the entire Unicode range, `0..0x10FFFF`,
zero disagreements — and tokenizes the same corpus in ~156 ms, so building the far
richer index costs ~240 ms cold and ~145 ms after a prompt submit (only the current
month's shard is re-tokenized; the rest come from the per-shard cache).

### Visual signal

Each ranked row renders three parts, aligned across the visible window the way the other
enriched providers already align their columns (`_row_layout()` in
`_prompt_input_bar_completion_panel_content.py`):

```
╭─ history words ────────────────────────────────────────────────────────────╮
│ ▸ reconcile        ▰▰▰▰▱  ⇄ monitor                                        │
│   reconciliation   ▰▰▰▱▱  ⇄ monitor                                        │
│   recording        ▰▰▱▱▱  ✦ 47×                                            │
│   recovered        ▰▱▱▱▱  ◷ 20m                                            │
╰─ ⇄ related · ◷ recent · ✦ frequent ───────────── [^L] accept  [^D] delete ─╯
```

- A **5-cell stacked meter** whose filled length is the composite score and whose cell
  colors are the per-signal contributions, largest-remainder distributed in a fixed
  relation → recency → frequency order so the colors never shuffle between repaints.
- A **dominant-reason chip**: `⇄ <context word>` for relation (the context word from the
  heaviest related prompt), `◷ <age>` for recency, `✦ <count>×` for frequency.
- A **colored legend** in the panel's border subtitle, so the meter's colors are
  self-explanatory without a help lookup.

Palette: relation `#5FD7D7` (cyan), recency `#87D787` (green), frequency `#D7AF5F`
(gold, already the saved-placeholder accent in
`_prompt_input_bar_completion_rows_simple.py`). All three signal glyphs are BMP symbols
that the existing PNG snapshot font (Fira Code) renders, and the visual suite's glyph
audit is the guard for that.

Everything the renderer needs is computed during scoring and carried on the candidate's
`metadata`, so rendering stays pure formatting over the ≤8 visible rows.

### Where the code lives

Scoring and indexing are pure, dependency-light functions in `src/sase/history/`, next
to the store they read, with no Textual imports — they are unit-testable without a TUI
and portable if prompt history ever moves.

`sase/memory/rust_core_backend_boundary` asks for shared backend behavior to live in
`../sase-core`. This plan deliberately keeps the work in Python: the entire
prompt-history subsystem (`src/sase/history/prompt_store.py` and friends) is
Python-only, `sase_core` has no prompt-history reader, wire, or binding today
(`crates/sase_core/src/editor/completion.rs` covers structured tokens — xprompts,
directives, `@` references, VCS refs — not prose words), and porting the store is a much
larger project than ranking its output. Keeping the ranking pure and TUI-independent
makes that future port mechanical rather than harder.

## Prompt-word corpus index

Add `src/sase/history/prompt_word_index.py`.

**`PromptWordIndex`** — a slots dataclass treated as immutable except for one private
memo slot (see the ranking phase). Prompt ids are assigned newest-first, so truncating a
posting list keeps the newest entries. Fields:

- `words: tuple[str, ...]` — word id → exact spelling (first spelling wins, newest
  first, matching today's behavior).
- `folded_order: tuple[int, ...]` and `folded_keys: tuple[str, ...]` — word ids sorted
  by `(casefold, spelling)` and their parallel folded keys, for `bisect` prefix ranges.
- `document_frequency: array("i")` — word id → number of prompts containing it.
- `last_used_epoch: array("d")` — word id → newest use as epoch seconds.
- `mru: tuple[int, ...]` — word ids by newest use descending, then insertion order.
- `prompt_words` / `word_prompts` — compressed sparse row pairs (`offsets: array("i")`,
  `values: array("i")`) for prompt → word ids and word id → prompt ids.
- `prompt_epoch: array("d")`, `prompt_count: int`, `max_document_frequency: int`,
  `source_token: PromptWordIndexToken`.

**Methods**: `word_ids_with_prefix(prefix)` (bisect over `folded_keys`, upper bound
`prefix + "￿"`, returns a `range` into `folded_order`), `word_ids_for_spelling`
(case-insensitive exact match, same bisect), `spelling(word_id)`, and
`prompt_word_ids(prompt_id)` / `word_prompt_ids(word_id)` returning memoryview-free
`array` slices.

**Builder**:
`build_prompt_word_index(*, min_length, shard_limit=24, prompt_limit=20000)`. It walks
`iter_shard_paths_newest_first()`, stops after `shard_limit` shards and `prompt_limit`
prompts (both bounds logged when they bite, per the "no silent caps" habit), sorts each
shard's entries by `last_used` descending exactly as today, and keeps the existing
candidate filters: `len(word) >= max(1, min_length)`, `is_prompt_word_candidate()`, and
`_has_useful_history_content()` — these move into this module and the old private copies
in `prompt_words.py` are deleted, not duplicated.

**Tokenizer**: `_PROMPT_WORD_RE = re.compile(r"[\w\-]+")` replaces the character loop
for bulk scans. `word_ranges()` and `is_word_character()` stay put in
`src/sase/ace/tui/widgets/prompt_word_completion.py` as the widget-side tokenizer; a
test pins their equivalence.

**Timestamps**: parse the `%y%m%d_%H%M%S` SASE timestamp by fixed-width integer slicing
plus `time.mktime`, not `datetime.strptime` (roughly ten times cheaper over thousands of
entries). An unparsable or empty timestamp yields epoch `0.0` — very old, never a crash.

**Per-shard cache**: a module-level dict keyed by
`(str(path), st_mtime_ns, st_size, min_length)` holding that shard's tokenized prompts
(`tuple[tuple[str, ...], float]` per prompt), bounded to 32 entries with
oldest-insertion eviction and cleared by an exported `clear_prompt_word_index_cache()`
that tests call. Only shards whose stat changed are re-tokenized, so the steady-state
rebuild after a prompt submit re-reads one shard.

**Token**: `prompt_word_index_source_token(*, min_length, shard_limit)` returns
`(min_length, shard_limit, tuple[(path, mtime_ns, size), ...])`, mirroring
`_shard_token()`.

Then re-express `collect_recent_prompt_words()` in `prompt_words.py` on top of the index
(`mru` order, deletions filtered, truncated to `max_words`) and keep
`history_words_source_token()` exported with its current signature and semantics. Its
existing tests must pass untouched; that is the contract for "no behavior change" in
this phase.

**Tests** (`tests/history/test_prompt_word_index.py`, plus the existing
`tests/history/test_prompt_words.py` staying green):

- The regex and `is_word_character()` agree over `range(0x110000)` and over a
  hyphen/underscore/Unicode-dash corpus, and `_PROMPT_WORD_RE` reproduces
  `word_ranges()` spans on a fixture prompt set.
- Prompt ids are newest-first; `word_prompts` postings are ascending.
- `document_frequency` counts prompts, not occurrences (a word repeated in one prompt
  counts once).
- `word_ids_with_prefix` finds case-insensitive matches, respects Unicode folding, and
  returns an empty range for a miss.
- The per-shard cache is reused when stats are unchanged and invalidated when the newest
  shard is rewritten (assert re-tokenization by counting calls).
- `shard_limit` and `prompt_limit` truncate deterministically.
- Corrupt, empty, and missing shards yield an empty index rather than raising.

## Relation, recency, and frequency scoring

Add `src/sase/history/prompt_word_ranking.py`, importing only the index module and the
standard library.

**Constants** (one block, documented, the tuning surface of the feature):
`RELATION_WEIGHT = 0.50`, `RECENCY_WEIGHT = 0.30`, `FREQUENCY_WEIGHT = 0.20`,
`RECENCY_HALF_LIFE_SECONDS = 7 * 86400`,
`PROMPT_RECENCY_HALF_LIFE_SECONDS = 30 * 86400`, `RELATION_LIFT_CAP = 8.0`,
`RELATION_SHRINKAGE = 2.0`, `RELATION_MIN_PROMPTS = 8`, `CONTEXT_STOPWORD_RATIO = 0.20`,
`CONTEXT_MAX_WORDS = 24`, `CONTEXT_MAX_POSTINGS_PER_WORD = 2000`,
`RELATED_PROMPT_LIMIT = 250`, `HISTORY_WORD_MAX_ROWS = 200`.

**`RankedWord`** (frozen slots dataclass): `word`, `score`, and the three _weighted_
contributions `relation`, `recency`, `frequency` (so the meter needs no re-derivation),
plus `reason: Literal["relation", "recency", "frequency"]`, `related_to: str`,
`use_count: int`, and `age_seconds: float`. `reason` is the argmax of the weighted
contributions with a fixed relation → recency → frequency tiebreak, and is `"recency"`
when every contribution is zero.

**`build_word_ranking_context(index, text, *, exclude_range, now)`** returns a
`WordRankingContext` holding the context key (sorted tuple of context word ids),
`relation_by_word_id: dict[int, float]`, and
`related_context_by_word_id: dict[int, int]`. It implements steps 1–6 of the ranking
model. The result is memoized in one private mutable slot on the `PromptWordIndex`
instance keyed by the context key; because a rebuild replaces the whole index object,
the memo can never serve results from a stale corpus (`sase/memory/tui_perf.md` rule 8).

**`rank_history_words(index, context, *, prefix, deleted, exclude_exact, now, limit=HISTORY_WORD_MAX_ROWS)`**
takes the bisect prefix range, skips deleted spellings and the `exclude_exact` no-op
spelling, scores what is left, sorts, and truncates to `limit`. It returns
`(ranked, shared_extension_source)` where the second element is the **full** matched
spelling list — the `Ctrl+T` shared-prefix extension must be computed over every match,
not the truncated top-N, or narrowing would jump past valid candidates.

**`rank_recent_history_words(...)`** is the `recent` mode path: the existing MRU filter,
returning `RankedWord`s with zeroed contributions and no reason chip, so downstream code
has exactly one row type.

**Tests** (`tests/history/test_prompt_word_ranking.py`) build small in-memory corpora
through the phase-1 builder and assert behavior, not float literals:

- A word co-occurring with the prompt's context outranks a more recent unrelated word;
  with the context words removed from the prompt, the order flips.
- A word in every prompt (`lift ≈ 1`) gets `relation == 0` and does not displace a
  genuinely associated word.
- Shrinkage: one related-prompt hit scores strictly below the same lift with many hits.
- Recency decay halves at the half-life; a future timestamp clamps to `1.0`; an
  unparsable timestamp ranks last rather than raising.
- Frequency saturates and never exceeds the recency+relation pair on its own.
- Corpora below `RELATION_MIN_PROMPTS` produce zero relation everywhere.
- Context extraction skips the word under the cursor, includes words after the cursor,
  and drops stop-word-frequency context words.
- Determinism: equal scores sort by folded spelling then spelling; the same inputs
  produce the same order across runs.
- The memo returns the identical object for the same context key and recomputes for a
  different one.
- `shared_extension_source` covers matches beyond `limit`.
- A bench-style assertion (marked `slow`) that ranking a 5,000-prompt synthetic corpus
  stays well under the per-keystroke budget.

## Warm cache, menu, and settings wiring

**Settings** — add to `PromptCompletionSettings`
(`src/sase/ace/tui/widgets/prompt_completion.py`) and
`parse_prompt_completion_settings()`:

- `word_ranking: Literal["smart", "recent"] = "smart"` — `smart` is the new ranking,
  `recent` restores today's exact MRU order and suppresses the signal column.
- `word_ranking_signals: bool = True` — render the meter/chip/legend.

Unknown or malformed values fall back to the default the same way the neighboring
parsers do. Mirror both keys in `src/sase/default_config.yml` under
`ace.prompt_completion` and in `src/sase/config/sase.schema.json` with descriptions and
defaults.

**Warm cache** (`src/sase/ace/tui/actions/_startup_history_words.py`) — the cached
payload becomes the index plus the deletion set instead of a word list:

- `history_prompt_word_index()` returns the warm `PromptWordIndex | None`.
- `history_prompt_words()` stays, returning the MRU spelling list derived from the warm
  index (computed once and cached alongside it), so existing callers, tests, and the
  `recent` path keep working.
- `history_prompt_word_deletions()` returns the warm `frozenset[str]`.
- The off-thread load returns `(index, deletions)`; the staleness token becomes
  `(index_token, deletions_token)`, and a deletions-only change re-reads just the small
  deletions file instead of rebuilding the index.
- `forget_history_prompt_word()` adds the word to the in-memory deletion set and
  refreshes visible surfaces without invalidating the index, so `Ctrl+D` no longer
  triggers a full corpus rebuild. The disk write stays on the existing
  `schedule_persist()` path in `_file_completion_accept.py`, unchanged.
- Keep the `history_word_count <= 0` short-circuit, the in-flight/pending coalescing
  guards, and the cold-cache `loading history words…` placeholder exactly as they are.

**Menu** (`src/sase/ace/tui/widgets/history_word_completion.py`) — add

```python
@dataclass(frozen=True, slots=True)
class HistoryWordCompletionMetadata:
    reason: str
    related_to: str
    use_count: int
    age_seconds: float
    score: float
    relation: float
    recency: float
    frequency: float
```

and rebuild `build_history_word_completion_result()` around the index: same
left-of-cursor prefix, same replacement range, same `has_word_suffix` handling, same
"exact spelling suppressed when accepting it is a no-op" rule, same
`WordCompletionResult` shape. The candidate order becomes the ranked order, each
candidate carries its metadata, and `shared_extension` is computed from the full match
set. Keep a list-of-spellings overload for `recent` mode and for the existing callers
and test harnesses that pass plain word lists.

Call sites to update, preserving their current control flow:
`_try_history_word_completion_tab()` (`_file_completion_tab.py`),
`_refresh_history_word_completion()` (`_file_completion_refresh.py`), and
`_apply_history_word_completion_result()` (`_file_completion_base.py`). The context for
ranking is the text area's full text with the cursor's word range excluded; pass `now`
explicitly so tests can pin it.

**Prompt-local ordering** (`prompt_word_completion.py`) — replace the alphabetical sort
in `build_prompt_word_completion_result()` with nearest-first: track each distinct
spelling's latest start offset before the active word and order by that offset
descending. The word you just wrote is the one you are most likely repeating, and this
matches how editors' own keyword completion behaves. Everything else about that function
is unchanged.

**Tests** — extend `tests/ace/tui/widgets/test_history_word_completion.py` and
`tests/ace/tui/widgets/test_prompt_word_completion.py`:

- Ranked ordering end to end from a seeded history store, including a case where the
  related word is _not_ the most recent one.
- `word_ranking: recent` reproduces today's ordering exactly and attaches no signal
  metadata.
- `history_word_count: 0` still disables the fallback; the cold cache still shows the
  placeholder; the placeholder is still non-selectable and still refuses `Ctrl+D`.
- `Ctrl+D` removes the row immediately and does not invalidate the index.
- Shared-prefix narrowing still works when the match set exceeds
  `HISTORY_WORD_MAX_ROWS`.
- Mid-word completion (`foo<cursor>baz`) keeps the behavior committed in `be0e35d81`:
  prefix-only replacement, preserved suffix, separating space.
- Prompt-local candidates come back nearest-first, with the alphabetical expectation in
  the existing tests updated.

## Ranking signals in the completion panel

Add `src/sase/ace/tui/widgets/_history_word_rows.py` holding the palette, glyphs, meter
builder, chip formatter, and the row appender; keep
`_prompt_input_bar_completion_rows_simple.py` as the plain-word renderer for the
`prompt_word` menu and for `recent` mode.

- `history_word_label_width(candidate)` measures the word column, capped at 28 cells,
  and joins the existing `_RowLayout` measurement in
  `_prompt_input_bar_completion_panel_content.py` so every visible row aligns.
- `build_score_meter(metadata)` returns the 5-cell stacked bar: filled cells are
  `max(1, round(score * 5))` when the score is positive, distributed across the three
  signals by largest remainder in relation → recency → frequency order, each cell styled
  with its signal color and the remainder rendered as dim `▱`.
- `format_reason_chip(metadata)` returns the dominant-reason chip: `⇄ <word>` (the
  related context word, truncated to 16 cells), `◷ <age>` with a compact `s/m/h/d/w`
  unit, or `✦ <count>×`.
- The renderer degrades by width, never by clipping: the chip is dropped first, then the
  meter, leaving a plain word row on very narrow panels. Signals are also skipped
  entirely when `word_ranking_signals` is off, when the row carries no metadata, and for
  the loading placeholder, which keeps its `dim italic` styling.
- `history_word_completion_subtitle(visible, inner_width)` in
  `_prompt_input_bar_completion_panel_labels.py` returns a `Text` legend —
  `⇄ related · ◷ recent · ✦ frequent` with each glyph in its signal color and dim
  separators — joined with the existing `[^L] accept  [^D] delete` hint, and falls back
  to the current plain hint when the panel is too narrow or no row carries metadata.
  Branch to it in `show_file_completions()` (`_prompt_input_bar_completion_panel.py`)
  ahead of the generic `completion_delete_subtitle()` call, mirroring the existing
  artifact/model subtitle branches; leave `completion_delete_subtitle()`'s other two
  providers untouched.

**Docs and help** — update the History-word and Prompt-local bullets in the
`docs/ace.md` Completion section: the ranking model and its three signals, the
meter/chip/legend, the two new settings, nearest-first local ordering, and the note that
`Ctrl+D` is now instant. The `?` help modal needs no change: it documents keys, and no
keybinding or key semantics change here.

**Tests**:

- Unit tests for the meter (filled length tracks score, color runs follow the
  contributions, sums to 5 cells, zero score renders empty), the chip formatter (each
  reason, age unit boundaries, long context-word truncation), and the legend (wide,
  narrow, no-metadata, signals-disabled).
- Widget tests that render the panel through `show_file_completions()` and assert the
  row text and border subtitle for a wide and a narrow panel, and that
  `word_ranking_signals: false` renders plain rows.
- Refresh the PNG goldens
  (`tests/ace/tui/visual/test_ace_png_snapshots_history_word_completion.py` →
  `history_word_completion_panel_120x40.png`, and
  `prompt_word_completion_panel_120x40.png` if local reordering changes its rows) by
  extending the fixture to attach ranking metadata and regenerating with
  `--sase-update-visual-snapshots`; the goldens are the acceptance evidence that the
  panel actually looks right. The visual glyph audit covers the three new symbols.

## Verification

Each phase leaves the tree green with `just check`. The final phase additionally runs
`just test-visual` for the refreshed goldens, and the combined tree is landed behind
`just check-full` through `/sase_monitor`.
