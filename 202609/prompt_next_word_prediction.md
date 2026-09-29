---
tier: epic
title: Next-word prediction chains in the prompt input
goal: 'In the prompt input, pressing Ctrl+T repeatedly first completes the current
  word, then previews and accepts confident guesses for the next words. The guesses
  come from the user''s own typed prompt history (weighted toward the same project,
  with the cross-machine prompt archive as a low-weight source) and appear as dim
  inline ghost text before anything is inserted. Predictions come from the Rust core
  in well under a millisecond and never block typing.

  '
phases:
- id: word-menu-ctrl-t
  title: Ctrl+T accepts the highlighted word-menu row
  depends_on: []
  size: small
  description: 'word-menu-ctrl-t: a second Ctrl+T on an open prompt-word or history-word
    menu accepts the highlighted row instead of re-dispatching and resetting the highlight;
    loading placeholders still re-dispatch; hints, docs, tests, and goldens are updated.'
- id: prompt-origin
  title: Record typed vs generated origin on prompt history rows
  depends_on: []
  size: medium
  description: 'prompt-origin: add an optional origin field (typed or generated) to
    PromptEntry, round-trip it through shard I/O with typed-wins merge rules, and
    thread it from every launch write site so the prediction corpus can exclude machine-generated
    prompts.'
- id: core-engine
  title: Rust prompt_prediction engine in sase-core
  depends_on: []
  size: medium
  description: 'core-engine: new sase_core::prompt_prediction module with the prose
    tokenizer and privacy filters, the legacy origin heuristic, the compiled n-gram
    corpus (recency-weighted mass, distinct support, project partitions), and multi-source
    stupid-backoff prediction with confidence presets, greedy continuation, and rank_prefix.'
- id: core-binding
  title: PyO3 handles, Python facade, and pin for prompt prediction
  depends_on:
  - core-engine
  size: small
  description: 'core-binding: expose frozen PromptPredictionCorpus and PromptPredictionModel
    pyclasses (compile releases the GIL), add the typed Python facade and wire mirror,
    the schema-version validator entry, a rust_backend.md section, and move the sase-core-revision
    pin.'
- id: prediction-cache
  title: Off-thread prediction corpus warm cache for the TUI
  depends_on:
  - prompt-origin
  - core-binding
  size: medium
  description: 'prediction-cache: build history rows (origin, project, cancelled),
    compile the history and session corpora off-thread next to the history-word warm
    job, swap them in atomically, compose the model, and give widgets a non-blocking
    predict accessor that degrades to silence.'
- id: ghost-chain
  title: Ghost-text next-word chain on Ctrl+T
  depends_on:
  - word-menu-ctrl-t
  - prediction-cache
  size: medium
  description: 'ghost-chain: arm the chain after every word commit, show gated predictions
    as inline ghost text with a border hint, make Ctrl+T/Alt+F take one word and Ctrl+F/Right/Ctrl+L
    take all, clear the ghost on every exit, add the next_word config keys, CSS, docs,
    tests, and PNG goldens.'
- id: next-word-menu
  title: Explicit next_word menu and word-end fallback
  depends_on:
  - ghost-chain
  size: medium
  description: 'next-word-menu: when a chain is armed but no ghost can be shown, Ctrl+T
    opens a next_word menu (context title, confidence meter, continuation preview)
    whose accept continues the chain; add the word-end fallback where current-word
    completion finds nothing, plus Ctrl+D forget.'
- id: context-ranking
  title: Context-aware current-word ranking
  depends_on:
  - prediction-cache
  - next-word-menu
  size: medium
  description: 'context-ranking: promote prompt-word and history-word candidates that
    the n-gram model predicts for the preceding words, and render the new sequence
    signal (violet meter share, dashed-arrow context chip, legend entry).'
- id: replay-harness
  title: Prequential replay harness and preset calibration
  depends_on:
  - core-binding
  - prediction-cache
  size: medium
  description: 'replay-harness: a Rust prequential replay evaluator plus a tools/prompt_prediction_replay
    script that prints aggregate-only accuracy, coverage, precision, keystroke-savings,
    run-length, latency, and memory tables by cohort, then calibrate the confidence
    presets.'
- id: archive-source
  title: Cross-machine prompt archive as a low-weight source
  depends_on:
  - prediction-cache
  - replay-harness
  size: medium
  description: 'archive-source: extract human-typed prose from the enabled projects''
    canonical prompt archives, dedup it against local history, compile a pruned low-weight
    archive corpus off-thread, add the next_word_sources config, and let the replay
    harness decide the default.'
- id: auto-mode
  title: Opt-in automatic ghost at word boundaries
  depends_on:
  - next-word-menu
  size: small
  description: 'auto-mode: add next_word auto, which shows the gated ghost right after
    a typed space that ends a prose word at end of line, reusing the ghost-chain acceptance
    and clearing contract.'
proposed_by: bbugyi200.apollo.2x
create_time: 2026-09-29 07:14:23
status: wip
bead_id: sase-1cj
---

- **BEAD:** [sase-1cj](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1cj/README.md)

# Plan: Next-word prediction chains in the prompt input

## 1. Summary

The feature is fish-style history autosuggestion for prompt prose, with word-at-a-time
accept on `<ctrl+t>`. It is built on a small, local, confidence-gated n-gram model
trained on the user's own typed prompts. The user's requested rhythm is kept exactly:
`<ctrl+t>` opens the word menu, `<ctrl+t>` accepts the word, then each further
`<ctrl+t>` takes the next word. One change makes that rhythm safe: **the guessed words
are shown as dim inline ghost text before `<ctrl+t>` takes them.** The ghost appears
automatically the moment a word is committed, so the chain costs no extra keystrokes
compared with blind insertion. The user can also see a wrong guess and ignore it.

Background research is in
`research:202609/prompt_next_word_prediction/prompt_next_word_prediction.md`. Read it
with `sase artifact read`. It consolidates five reports and a replay experiment over the
real prompt history. The design decisions it established:

- **The premise is false today.** A second `<ctrl+t>` on an open word menu re-dispatches
  completion and resets the highlight to row 1. It does not accept. Phase
  `word-menu-ctrl-t` fixes that.
- **Blind insertion loses keystrokes on novel text** (−7.5% keystroke savings).
  Preview-then-accept turns that loss into a gain.
- **Context is everything.** Orders 3–4 carry the value. A confidence gate gives 70–80%
  precision when a guess is shown. On formulaic or re-typed prompts, chains run 4–7
  words.
- **More text buys little.** A 50× larger corpus adds about 2 points. Generic English
  "common sense" adds 0.7–2 points and lowers gated precision.
- **About 70% of local history is machine-generated.** It must be excluded, which is why
  phase `prompt-origin` exists.
- **The engine belongs in `sase-core`.** This follows the Rust core boundary rule. A
  CPython model costs +182 MB at archive scale. Queries in Rust take microseconds.

## 2. Design principles

1. **Ctrl+T never inserts a predicted word you have not seen.** Every inserted guess was
   first visible, either as ghost text or as the highlighted menu row. Existing
   current-word completion is unchanged, including the unique-match commit on the first
   press, because there the user typed the prefix.
2. **Ctrl+T always moves you forward.** Pressing it again opens the menu, then accepts
   the row, then takes the next ghost word, then offers next words. Every accept arms
   the chain again.
3. **Silence when unsure.** A ghost appears only when the gate passes, and never from a
   unigram. A low-confidence explicit request opens a small menu. It never
   blind-inserts.
4. **Visible state decides behavior.** `<ctrl+t>` takes the ghost only when a ghost is
   showing. Otherwise it behaves as today, except in the explicitly armed chain. Nothing
   is hidden.
5. **Never block, never crash typing.** Queries are synchronous but microsecond-scale.
   Builds run off-thread with the GIL released. A cold or missing model means no ghost.
   Any prediction error disables the feature for the session and is logged once.
6. **Structured syntax is sacred.** The following are never predicted into: `#`, `%`,
   `@`, `+`, `=`, paths, Jinja, placeholders, code spans, fences, frontmatter, and VCS
   tags. Their completion paths are untouched.
7. **One ghost language.** The ghost uses the same muted style as the Command Line's
   fish-style ghost. Hints use the existing `[^X] verb` subtitle idiom. Menu rows reuse
   the existing score meter and reason chip.

## 3. Interaction contract: the Ctrl+T ladder

This contract applies in INSERT mode in a prompt input. The rows are evaluated in order,
and the first match wins:

| #   | State when `<ctrl+t>` is pressed                                                                                          | Behavior                                                                                                                                                                                                                                                                    | Phase                                                  |
| --- | ------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| 1   | A `prompt_word`, `history_word` or `next_word` menu is open and the highlighted row is a real candidate                   | Accept the highlighted row, then arm the chain                                                                                                                                                                                                                              | `word-menu-ctrl-t` (+ `next_word` in `next-word-menu`) |
| 1b  | The same menu is open, but the highlighted row is the "loading history words…" placeholder                                | Unchanged: re-dispatch, which refreshes once the cache is warm                                                                                                                                                                                                              | `word-menu-ctrl-t`                                     |
| 2   | A next-word ghost is visible                                                                                              | Insert the ghost's first word, with its leading separator, as one undo step. Then predict again and redraw the ghost in the same handler, so it never flickers                                                                                                              | `ghost-chain`                                          |
| 3   | The chain is armed (cursor and document unchanged since the last word commit) and no ghost is visible                     | Explicit next-word request: if the gate passes and the ghost fits, show it. Otherwise open the `next_word` menu of top candidates. If there is nothing to offer, show a dim `no next-word guess` hint. If the model is cold, show `warming next words…` and schedule a warm | `ghost-chain` (hint only), `next-word-menu` (menu)     |
| 4   | Anything else                                                                                                             | Unchanged dispatcher (`_try_file_completion_tab`)                                                                                                                                                                                                                           | —                                                      |
| 4b  | Inside row 4: the cursor is at the end of a prose word and both prompt-word and history-word completion find no candidate | Explicit next-word request, as in row 3. Today this press is a no-op                                                                                                                                                                                                        | `next-word-menu`                                       |

Other keys, all in INSERT mode:

| Key                                                                                                                                                             | With a ghost visible and no menu open                                                        | Otherwise                                                                |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| `ctrl+f`, `right`, `ctrl+l`                                                                                                                                     | Accept the whole ghost as one undo step, then arm the chain again (predict again)            | Unchanged. On an open word menu, accepting a row now also arms the chain |
| `alt+f`                                                                                                                                                         | Accept one ghost word (fish parity)                                                          | Unchanged word-right                                                     |
| Typing                                                                                                                                                          | Characters that match the ghost consume it (Textual built-in). Any other character clears it | —                                                                        |
| Backspace/delete, cursor motion, undo/redo, `esc`/NORMAL, blur, pane switch, submit, cancel, a menu opening, snippet/tabstop activity, an xprompt argument hint | Clear the ghost and disarm the chain                                                         | —                                                                        |
| Whitespace `<ctrl+t>` with no ghost                                                                                                                             | Unchanged: recent-file and artifact history                                                  | —                                                                        |

A ghost is shown only when all of the following hold:

- INSERT mode
- no completion menu is open
- no snippet session is active
- the Jinja panel is not showing
- the text after the cursor on the current line is blank
- at least the first ghost word fits in the visible width that remains on the cursor's
  wrapped row

Ghosts are truncated to whole words that fit, and capped at `next_word_max_words`. Ghost
display wins over the existing soft completion: while a ghost is visible,
`_soft_completion_blocked()` returns True.

Separators: the leading separator is a single space. It is omitted when the character
before the cursor is whitespace, the start of the text, or an opening bracket or quote.
Accepting from the menu in the middle of a line also adds a trailing space when an
identifier-like character follows the cursor. This mirrors `has_word_suffix` in
`_commit_word_completion`.

## 4. Visual design

The ghost is muted inline text. The cursor block sits on its first cell. While it is
visible, the prompt bar's border subtitle shows the accept hint:

```text
╭──────────────────────────────────────────────────────────────────────╮
│ #gh:sase Can you help me implement▌it now                            │   (" it now" in $text-muted)
╰──────────────────────────────────────────────── [^T] word  [^F] all ─╯
```

The `next_word` menu opens on an explicit, lower-confidence request. Its title names the
context it conditioned on. Each row shows the word, a 5-cell confidence meter in the new
**sequence** color, and a dim preview of where that choice leads:

```text
╭─ next word ⇢ “help me” ─────────────────────────╮
│ ▸ implement  ▰▰▰▰▱  ⇢ it now                     │
│   review     ▰▰▱▱▱  ⇢ the plan                   │
│   fix        ▰▱▱▱▱                               │
╰────────────────────────── [^T] accept  [^D] delete ─╯
```

In phase `context-ranking`, the current-word menus mark promoted rows with the same
glyph and color:

```text
╭─ history words ──────────────────────────────────────────────╮
│ ▸ implement    ▰▰▰▰▰  ⇢ help me                               │
│   important    ▰▰▰▱▱  ◷ 2d                                    │
╰── ⇄ relation ◷ recency ✦ frequency ⇢ context   [^T] accept  [^D] delete ─╯
```

Visual tokens:

- **Ghost:** Textual's `.text-area--suggestion` component class. Declare it explicitly
  for the prompt input in `src/sase/ace/tui/styles.tcss` as `color: $text-muted`, which
  is identical to the Command Line ghost.
- **Sequence signal:** color `#AF87FF` (soft violet, distinct from relation `#5FD7D7`,
  recency `#87D787` and frequency `#D7AF5F`) and glyph `⇢` (U+21E2). Define both next to
  the existing constants in `src/sase/ace/tui/widgets/_ranking_signal_rows.py`. Verify
  the glyph in the PNG goldens. If the bundled font stack renders it as tofu, switch to
  `→` everywhere.
- **Hints:** use dim text in the existing subtitle style:
  - `[^T] word  [^F] all` while a ghost is visible
  - `no next-word guess` and `warming next words…` as transient hints, cleared on the
    next context change
  - `[^T] accept` in the prompt-word menu subtitle
  - `[^T] accept  [^D] delete` in the history-word and next-word menu subtitles

## 5. Model design (`sase_core::prompt_prediction`)

### 5.1 Sources and weights

| Source role     | Content                                                                          | Default weight |
| --------------- | -------------------------------------------------------------------------------- | -------------- |
| `history`       | Local prompt history. Typed rows only; cancelled drafts are included             | 1.0            |
| `project` boost | Extra mass from same-project `history` observations, via per-project partitions  | +1.0           |
| `session`       | Prompts submitted in this TUI session that are not yet in the history corpus     | 1.0            |
| `draft`         | The text before the cursor in the current prompt, counted per request            | 1.0            |
| `archive`       | The enabled projects' canonical prompt archives, pruned (phase `archive-source`) | 0.25           |

Each observation is decayed by recency when the corpus is built:
`max(0.05, 0.5^(age_days / 14))`.

### 5.2 Tokenizer

Compile and query share one tokenizer, so context extraction always matches training.

- **Excluded regions.** Each of these is a hard boundary:
  - leading YAML frontmatter
  - fenced code (reuse `fenced_code::fenced_block_ranges`)
  - inline code spans (reuse `prompt_literals::inline_code_ranges`)
  - Jinja `{{…}}` / `{%…%}` / `{#…#}`
  - `---` segment-separator lines
  - pasted blocks: any line with more than 400 tokens, or one where fewer than half the
    tokens are words (logs, tracebacks, tables)
- **Other boundaries.**
  - blank lines
  - line-start list, heading and quote markers (`-`, `*`, `+`, `>`, `#{1,6} `,
    `\d+[.)]`)
  - sentence-final punctuation after a word (`.`, `?`, `!`, `…`)
  - `:` or `;` after a word
  - structural or non-word tokens

  A single newline inside a paragraph is plain whitespace, so hard-wrapped text keeps
  its sequence.

- **Structural tokens.** These are boundaries and are dropped:
  - raw whitespace tokens starting with `#`, `%`, `@`, `+`, `=`, `$`, `<`, `{`
  - tokens containing `/`, `\`, `://`, `=`, `|`, `](`, a backtick, or an interior `.`
- **Words.** Strip leading `(["'“‘*_` and trailing `)]"'”’*_,`, plus boundary
  punctuation. The core must:
  - match letters and digits joined by `'`, `’`, `-` or `_`
  - contain a letter
  - be at most 32 characters
  - not be digit-only, hash-like (7 or more hex characters including a digit), or
    secret-like (20 or more characters mixing letters and digits, or a known secret
    prefix such as `sk-`, `ghp_`, `xox`, `AKIA`)

  Anything else is a boundary.

- **Keys and surfaces.**
  - Key: Unicode lowercase, with `’` mapped to `'`.
  - Surface: chosen per key. Use the recency-weighted most common casing among
    non-sequence-initial occurrences. If every occurrence is sequence-initial, lowercase
    the first letter, unless the word is all caps (2 or more letters) or has an interior
    capital.
- **Markers.** A sequence starts with a `<s>` context marker at prompt start, after a
  blank line, list, heading or quote marker, and after sentence-final punctuation. A
  sequence that begins after a mid-sentence structural token, code span, `:` or `;`
  starts with an empty context. `<s>` is never predicted.

### 5.3 Corpus compile

- **Row filtering.**
  - Rows with `origin == "generated"` are dropped.
  - Rows with no origin go through the legacy heuristic in `origin.rs`. It marks a row
    generated if it:
    - contains a `#bd/work_phase_bead…` token, or any `#bd/…` token together with `%id(`
    - starts with `%wait(`
    - matches a swarm or lead template marker

    Inventory the markers from the sase repo's `src/sase/xprompts/` templates. Give each
    one a named const and a test.

  - Rows with exactly the same text are counted once, using the newest epoch.

- **Counting.** For each word position and each context length L from 0 to 4 (`<s>`
  counts as a context token), record `(context, word)` at most once per row:
  `mass += recency weight` and `distinct += 1`.
  - Context totals (mass and distinct) are accumulated before truncation.
  - Keep the top `max_successors_per_context` (default 32) successors by mass, with ties
    broken by key.
  - When `prune_singleton_contexts` is set (archive), drop contexts of length 2 or more
    whose total distinct count is 1.
  - Build per-project partitions when rows carry `project`.
  - Words in the compile option `excluded_words` (the history-word deletions store) are
    never predicted.
- **Storage.** Interned `u32` word ids and packed context keys keep memory lean. A local
  corpus must stay at or under 5 MB. An archive corpus must stay at or under 60 MB.

### 5.4 Prediction, gate, continuation

- **Context.** Tokenize `text_before_cursor`. The result is **blocked**, with a reason,
  when:
  - the cursor is in an excluded region: an unclosed fence, backtick span, Jinja tag or
    frontmatter
  - the last token is not a word
  - the current sequence has no word token

  The draft source counts the text's own sequences as an extra source, but leaves out
  the final position.

- **Scoring.** Use stupid backoff with α = 0.4. For each L from the longest matched
  context down to 0, compute the combined mass
  `Σ source weight · mass (+ project_boost · same-project partition mass)` divided by
  the combined total. Each word keeps its best score, multiplied by α^(Lmax − L).
  Candidates are sorted by score descending, then key ascending, so results are fully
  deterministic.
- **Gate.** Each of the three presets has its own thresholds (see the table below).
  1. Choose the evidence order L\*: the largest L whose context contains at least one
     real word (`<s>` alone does not count) and whose combined distinct total is at
     least `min_support`.
  2. The gate passes only if all of these hold:
     - the top-1 is also the top successor at L\*
     - its share `p` at L\* is at least `min_p`
     - `p` minus the runner-up's share at L\* is at least `min_margin`
     - its combined distinct support at L\* is at least `min_support`
     - with `reject_conflicts` (default on): no higher order that has observations ranks
       a different word first

  | Preset               | `min_p` | `min_margin` | `min_support` |
  | -------------------- | ------- | ------------ | ------------- |
  | `cautious`           | 0.75    | 0.35         | 4             |
  | `balanced` (default) | 0.6     | 0.2          | 3             |
  | `eager`              | 0.45    | 0.1          | 2             |

  These seed values come from the research. Phase `replay-harness` calibrates them.

- **Continuation.** Extend greedily. While the gate passes and the word count is below
  `max_words`, append the top-1 as a context token and predict again. The draft source
  stays frozen to the original text. The ghost is the gated continuation of the top-1.
  Every menu candidate carries its own gated continuation preview of up to 3 words.
- **Menu candidates.** Up to `limit` words that have evidence at order 1 or higher. A
  word with unigram evidence only is never offered.
- **`rank_prefix`.** Take the context from the text before the current word. Among
  successors at L ≥ 1 whose key starts with the casefolded prefix (the prefix itself
  excluded), return up to `limit` with score, order, support and `context_words`.
- **Performance budget.** Predict, including continuation and a draft of up to 20k
  characters:
  - p95 ≤ 0.5 ms local-only, ≤ 1 ms with the archive
  - local compile ≤ 50 ms per 1k prompts
  - archive compile ≤ 2 s per 700k tokens, with the GIL released

### 5.5 Wire contract

Every output struct carries `schema_version`, following the style of
`prompt_history_filter::wire`. All structs are serde `*Wire` structs.

- `PromptPredictionRowWire { text, epoch_seconds, project?, origin?, cancelled }`
- `PromptPredictionCorpusOptionsWire { now_epoch, recency_half_life_days=14, max_context_words=4, max_successors_per_context=32, prune_singleton_contexts=false, excluded_words=[] }`
- `PromptPredictionModelConfigWire { backoff_alpha=0.4, project_boost=1.0, draft_weight=1.0, reject_conflicts=true }`
- `PromptPredictionRequestWire { text_before_cursor, project?, limit=5, max_words=4, confidence="balanced", include_draft=true }`
- `PromptPredictionResultWire`, which contains:
  - `blocked_reason?`
  - `context_words` (the surfaces of the evidence context, oldest first, `<s>` omitted)
  - `confident`
  - `ghost: [word]`
  - `candidates: [{ word, key, score, probability, support, order, source_shares{history,project,session,draft,archive}, continuation: [word] }]`
- `PromptPrefixRankRequestWire { text_before_word, prefix, project?, limit }` and
  `PromptPrefixRankResultWire { context_words, matches: [{ word, key, score, order, support }] }`
- `PromptPredictionCorpusStatsWire { rows_used, rows_generated_skipped, rows_duplicate_skipped, tokens, contexts, successor_entries, approx_bytes }`

## 6. Architecture and boundaries

- **`sase-core` (Rust).** It holds all shared behavior:
  - tokenizer, origin heuristic, corpus compile
  - prediction, gate presets, continuation, `rank_prefix`
  - the replay evaluator

  The TUI, `sase-nvim`/the xprompt LSP, and any web UI need identical behavior, so the
  boundary rule applies. `prompt_history_filter` sets the precedent: the host resolves
  project facts and row text off the event loop, and core operations are pure and
  synchronous.

- **Bindings.** Frozen `#[pyclass]` handles modeled on `CommandLineGrammar`: build once
  off the event loop, then answer per keystroke. A `PromptPredictionCorpus` wraps an
  `Arc`, so a `PromptPredictionModel` is a cheap composition that Python rebuilds
  whenever any corpus swaps.
- **Python (sase).** Python owns:
  - shard and archive reads
  - project-key resolution, through the Ctrl+K `PromptHistoryProjectCatalog` loaded
    off-thread
  - deletions
  - warm and swap scheduling
  - all Textual state, keys, and rendering
- **No feature flag.** Every phase lands a coherent increment:
  - after `word-menu-ctrl-t`: a better menu
  - after `ghost-chain`: a working chain with a hint fallback
  - after `next-word-menu`: the full ladder

  `next_word: off` is the permanent, user-chosen opt-out, so it is a config field, not a
  flag. See the flag rules in `sase/memory/sase_flags.md`.

## 7. Configuration (`ace.prompt_completion`)

| Key                    | Values                                 | Default                                    | Phase                                               |
| ---------------------- | -------------------------------------- | ------------------------------------------ | --------------------------------------------------- |
| `next_word`            | `off`, `chain`, `auto`                 | `chain`                                    | `ghost-chain` (`off`/`chain`), `auto-mode` (`auto`) |
| `next_word_max_words`  | 1–8                                    | 4                                          | `ghost-chain`                                       |
| `next_word_confidence` | `cautious`, `balanced`, `eager`        | `balanced`                                 | `ghost-chain`                                       |
| `next_word_sources`    | a list drawn from `history`, `archive` | set by the `archive-source` harness result | `archive-source`                                    |

Every key change must update all of these together:

- `src/sase/default_config.yml`
- `src/sase/config/sase.schema.json`, whose `ace.prompt_completion` block has
  `additionalProperties: false`
- `parse_prompt_completion_settings` in `src/sase/ace/tui/widgets/prompt_completion.py`
- the default-contract tests in `tests/test_config_schema_ace.py`
- the `ace.prompt_completion` section of `docs/configuration.md`

## 8. What "common sense" means here, and what is deferred

The user asked whether to add "common sense". The design's answer is to encode it as
**linguistic structure, not a generic corpus**:

- sentence and clause boundaries, and never chaining across them
- never predicting inside or after structural syntax or code
- casing canonicalization
- the **draft cache**, so what you already said in this prompt informs the next guess
- the **session source**, so what you just submitted is predictable immediately
- silence whenever the evidence is thin

Measured broader-English sources add under 2 points while lowering gated precision. An
LLM cannot meet the keystroke budget, costs money per keystroke, and sends drafts
off-machine. None of these is in this epic. Each is a later, harness-gated experiment:

- project phrasing from xprompt descriptions and docs, as a low-weight source
- an explicit, asynchronous "suggest phrase" LLM action
- Ctrl+T accept for non-word menus
- local acceptance counters
- serving the index to `sase-nvim` over the xprompt LSP

## 9. Phases

Conventions for every phase:

- Keep new Python modules under the repo's 500-line `toobig` limit, and split into
  sibling mixins or helper modules instead of growing large files.
- Follow `AGENTS.md` in `sase-core`:
  - free functions over serde `*Wire` structs, `thiserror` errors, no `macro_rules!`
  - facade-only `mod.rs`, with tests in `tests.rs` or `tests/`
  - files of at most 1,500 lines
  - import by module path, with no new root `pub use` and no `core_*` prelude aliases
  - run `sase tool run check` there
- In this repo, run `sase tool run check`. When rendered TUI output changes, run
  `just fix-tui-screenshots -- <selectors>` (hand long runs to `/sase_monitor`) and
  inspect the report.
- For visible behavior, also capture a live `sase screenshot` of the flow and inspect
  the PNG.
- If Symvision flags a public symbol that a later phase consumes, whitelist it with
  `--epic-symbol <epic_bead_id>(<symbol>)` in the Justfile, per `tools/AGENTS.md`. The
  consuming phase removes that whitelist entry.

### 9.1 Phase `word-menu-ctrl-t`

- **Key handling.** In the active-menu block of
  `src/sase/ace/tui/widgets/_prompt_text_area_key_handling.py`, handle `ctrl+t` when
  `_completion_kind` is `PROMPT_WORD_COMPLETION_KIND` or `HISTORY_WORD_COMPLETION_KIND`:
  - If the highlighted candidate is a real row, call `_accept_file_completion()`.
  - If the row's metadata is `HistoryWordCompletionPlaceholder`, fall through to the
    existing `ctrl+t` dispatch.
  - Keep the kind set in one named constant, which `next-word-menu` extends.
  - All other kinds keep their existing re-dispatch.
- **What does not change.** The unique-candidate commit on the first press, the
  shared-extension insert, and `ctrl+f`/`ctrl+l` accept.
- **Subtitles.** In `_prompt_input_bar_completion_panel_labels.py`:
  - the prompt-word menu subtitle shows `[^T] accept`
  - the history-word plain hint becomes `[^T] accept  [^D] delete`
  - `history_word_completion_subtitle` keeps its legend ladder
  - `file_history` keeps `[^L] accept  [^D] delete`
- **Tests.** In the `tests/ace/tui/widgets/` word-completion suites, cover:
  - ctrl+t accepting row 1
  - `ctrl+n` then ctrl+t accepting row 2
  - a placeholder row re-dispatching
  - unique and shared-extension behavior unchanged
  - ctrl+f/ctrl+l unchanged
  - no effect on file, xprompt, or directive menus

  Update the prompt-word and history-word PNG goldens.

- **Docs.** In `docs/ace.md`, update the Ctrl+T row under `### INSERT Mode (Default)`
  and the key table under `### Completion`.

### 9.2 Phase `prompt-origin`

- **Entry field.** In `src/sase/history/prompt_store.py`, add
  `PromptOrigin = Literal["typed", "generated"]` and
  `PromptEntry.origin: PromptOrigin | None = None`.
  - `prompt_entry_from_json` reads a valid `origin`, and ignores anything else.
  - `_prompt_to_json` writes it only when it is set.
- **Merge rule: typed > generated > None, never downgrade.** It applies in three places:
  - the exact-text update in `_apply_prompt_mutations`
  - `dedup_prompt_entries_newest_first`
  - `rewrite_prompt_text_exact`, which preserves origin
- **Mutation API.** In `src/sase/history/prompt_store_mutations.py`,
  `add_or_update_prompt(..., origin=None)` and
  `record_failed_launch_prompt(..., origin=None)` pass the origin on. `---` segment
  saves inherit the origin.
- **Launch helpers.** Thread an `origin` keyword (default `None`) through
  `launch_agents_from_cwd` and its single, fanout, and variant helpers.
- **Origin at each call site.** Verify every site when implementing. The inventory is at
  the end of this phase.
  - `typed`:
    - `main/query_handler/_launch.py` (TUI submit and `sase run`)
    - `prompt/cli_run.py`
    - `integrations/_mobile_agent_launch.py`
    - TUI cancel and editor-abort saves (`_prompt_bar_mount.py`,
      `_prompt_bar_submit.py`)
    - failed-launch, quit-flush and stash saves (`_launch.py`, `launch_cwd_guards.py`,
      `_pending_launch.py`, `_prompt_bar_stash_store.py`)
  - `generated`:
    - `agent/launch_request_response.py` (LaunchApproval / `sase_run`)
    - `agent/launch_admission_runtime.py`
    - `main/plan_direct_approval_run.py`
    - `agent/_restart_execute.py`
    - the axe chop launchers (`axe/chop_runner.py`, `chop_proposal_launch.py`,
      `chop_typed_admission.py`, `chop_runner_script_result_launch.py`)
    - bead work (`bead/cli_work_launch.py` → `launch_cwd_bead_work.py`)
  - Robustness rule: when the launching process runs inside a SASE agent (the agent
    identity environment is present), record `generated` whatever the entry point.
- **Tests.** Cover JSON round-trip, merge precedence, and dedup. Assert representative
  typed and generated call sites through their seams.
- **Docs.** Update the storage notes in `docs/prompt.md`.
- **Out of scope.** No backfill rewrite of existing shards. The core heuristic
  classifies rows that have no origin.
- **Write-site inventory.** Search for `add_or_update_prompt(` and
  `record_failed_launch_prompt(` across `src/sase`. Every hit must pass an explicit
  origin or be a documented `None`.

### 9.3 Phase `core-engine` (in the linked `sase-core` repo)

Open the repo with `sase repo open sase-core`. It names that repo's `AGENTS.md`; read it
first.

- **Module layout.** Add `crates/sase_core/src/prompt_prediction/` and register it in
  `lib.rs` with `pub mod prompt_prediction;` only.
  - `mod.rs`: facade only
  - `wire.rs`: the §5.5 structs and `PROMPT_PREDICTION_WIRE_SCHEMA_VERSION = 1`
  - `tokenize.rs`: §5.2
  - `origin.rs`: the legacy heuristic, §5.3
  - `corpus.rs`: compile, §5.3
  - `model.rs` and `predict.rs`: source composition, backoff, gate presets,
    continuation, `rank_prefix`, §5.4
  - `tests/`
- **Public API** (free functions, no I/O):
  - `compile_prompt_prediction_corpus(&[PromptPredictionRowWire], &PromptPredictionCorpusOptionsWire) -> CompiledPromptPredictionCorpus`
  - `CompiledPromptPredictionCorpus::stats()`
  - `PromptPredictionModel::new(Vec<(Arc<CompiledPromptPredictionCorpus>, SourceRole, f32)>, PromptPredictionModelConfigWire)`
  - `model.predict(&PromptPredictionRequestWire) -> PromptPredictionResultWire`
  - `model.rank_prefix(&PromptPrefixRankRequestWire) -> PromptPrefixRankResultWire`
- **Query-code shape.** Write the successor lookup behind a small trait. The replay
  phase can then query a mutable builder without recompiling.
- **Tests.**
  - Tokenizer: every boundary, excluded-region, structural and privacy rule, and casing
    canonicalization.
  - Origin: each heuristic marker.
  - Compile:
    - per-row distinct versus mass
    - recency decay
    - exact-duplicate rows
    - successor truncation, with totals taken before truncation
    - project partitions
    - pruning
    - excluded words
  - Predict:
    - backoff order
    - each preset's gate boundaries (p, margin, support, conflicts)
    - `<s>`-only contexts never ghost
    - project boost changes the ranking
    - draft cache
    - blocked reasons (fence, backtick, Jinja, frontmatter, structural tail)
    - deterministic ties
  - Continuation: stops at the gate and at `max_words`.
  - `rank_prefix`.
  - An `#[ignore]` performance test on a synthetic 10k-prompt corpus that reports
    compile time, predict p95 and approximate bytes against the §5.4 budgets.

### 9.4 Phase `core-binding`

- **PyO3 module.** In `sase-core`, add
  `crates/sase_core_py/src/prompt_prediction/mod.rs` with a `register_prompt_prediction`
  function wired into the module init.
  - `#[pyclass(frozen)] PromptPredictionCorpus` wraps an `Arc`, with `__len__` and
    `stats()`.
  - `compile_prompt_prediction_corpus(rows: list[dict], options: dict)` converts through
    `json_bridge` with the GIL held, then compiles inside `py.allow_threads`.
  - `#[pyclass(frozen)] PromptPredictionModel` has
    `#[new](sources: list[dict{corpus, role, weight}], config: dict | None)`, the
    methods `predict(request)` and `rank_prefix(request)` returning `serialize_to_py`,
    and a `#[classattr] SCHEMA_VERSION`.
  - Invalid dicts raise `PyValueError`.
  - Add binding tests in the style of `editor_content/tests.rs`.
- **Python facade and wire.** In this repo, add:
  - `src/sase/core/prompt_prediction_wire.py`: frozen dataclasses plus the
    schema-version mirror.
  - `src/sase/core/prompt_prediction_facade.py`: `compile_prompt_prediction_corpus(...)`
    and a `PromptPredictionModel` wrapper holding `_handle` (as in
    `src/sase/completion/command_line_grammar.py`), with typed `predict`/`rank_prefix`
    results. It uses `require_rust_binding` and checks `SCHEMA_VERSION` on load.
  - `tests/core/test_prompt_prediction_facade.py`: round trip over a small corpus.
- **Validator, docs, and pin.**
  - Add the schema version to `tools/validate_sase_core_rs`.
  - Add a "Prompt prediction handles" section to `docs/rust_backend.md`, next to
    "Command Line grammar handle".
  - Move `sase-core-revision.txt` past the `sase-core` commit, per the CI pin section of
    `docs/rust_backend.md`.
  - Do not touch the `pyproject.toml` version window (the release job owns it).

### 9.5 Phase `prediction-cache`

- **Row loading.** `src/sase/history/prompt_prediction_rows.py`:
  - Load shards the way `build_prompt_word_index` does (`shard_limit` 24,
    `prompt_limit` 20000) and dedup newest first.
  - Emit rows with:
    - `epoch_seconds` from `last_used`, falling back to `timestamp`
    - `cancelled`
    - `origin`
    - `project`, resolved through the resolver below
  - Provide `prompt_prediction_source_token(...)`: shard stats plus the deletions token.
- **Project resolver.** It lives in the same module or a sibling.
  - Build it off-thread from `PromptHistoryProjectCatalog.load`, mapping aliases, owner
    and repo spellings, and project keys to one canonical project key.
  - `resolve_prompt_project(text)` is pure and keystroke-safe. It parses the draft's VCS
    or `+project` tag with the existing pure parsers and does an in-memory lookup.
  - An unresolved tag falls back to its lowercase text. With no tag, it uses the prompt
    bar's known project if that is cheaply available in memory, and otherwise returns
    `None`.
- **Warm mixin.** `src/sase/ace/tui/actions/_startup_prompt_prediction.py`:
  - Trigger it from the same points as `warm_history_prompt_words`, using the same
    coalescing flags and the `run_worker` + `asyncio.to_thread` pattern.
  - Rebuild only when the token changes.
  - Swap on the UI thread, compose the model, then call a
    `_refresh_visible_prompt_prediction_surfaces()` hook. It is a no-op here;
    `ghost-chain` fills it.
  - Pass `excluded_words` from the deletions store.
- **Session source.** Add `note_submitted_prompt_texts(texts)` and call it from the
  prompt-bar submit path.
  - Keep a bounded list (50) with submit times.
  - Rebuild the tiny session corpus off-thread, coalescing requests.
  - When a history rebuild lands, drop session texts whose normalized text now appears
    in history.
- **App accessors.** `get_prompt_prediction_model()` and
  `get_prompt_prediction_project(text)`.
- **Widget mixin.** `src/sase/ace/tui/widgets/_file_completion_prediction.py`:
  - `_prompt_prediction_model()`, `_prompt_prediction_is_cold()` and
    `_schedule_prompt_prediction_load()`.
  - `_predict_next_words(text_before_cursor, *, limit, max_words, confidence)`. It
    post-filters words deleted in memory for immediacy, truncating the ghost before a
    deleted word. It catches any exception, then disables prediction for the session and
    logs once.
- **Failure mode.** A missing binding (`AttributeError`/`ImportError`) leaves the model
  `None` for the session, with no toast.
- **Tests.**
  - rows and tokens
  - staleness and partial rebuilds
  - atomic swap
  - session pruning
  - project resolution
  - the missing-binding degrade
  - no disk I/O on the accessor path

### 9.6 Phase `ghost-chain`

- **Config.** Add `next_word` (`off`/`chain`), `next_word_max_words` and
  `next_word_confidence` everywhere §7 lists. `off` skips the warm build and every
  ladder row added by this epic.
- **Mixin.** `src/sase/ace/tui/widgets/_prompt_next_word.py` defines
  `PromptNextWordMixin`, added to the `PromptTextArea` bases in `prompt_text_area.py`.
  Pure helpers (ghost splitting, separator choice, width fitting, hint text) live in
  `src/sase/ace/tui/widgets/next_word_completion.py`.
- **State.**
  - `NextWordChain(anchor_offset, edit_generation)`. The chain is armed when both still
    match. The edit generation is bumped on every document edit.
  - `NextWordGhost(anchor_offset, edit_generation, full_text)`. It is valid while:
    - the text between the anchor and the cursor equals the consumed prefix of
      `full_text`
    - `self.suggestion` is its remaining suffix
    - the cursor stays on the anchor row, with the rest of the line blank

    Check this in `_on_prompt_completion_context_changed`, and clear the ghost
    otherwise.

- **Arming.** Call `_arm_next_word_chain()` after the menu is cleared, at every word
  commit:
  - the unique commits in `_try_prompt_word_completion_tab` and
    `_try_history_word_completion_tab`
  - the prompt-word and history-word accept branches of `_accept_file_completion`
  - ghost accepts

  Arming predicts synchronously and sets the ghost if the gate passes and the ghost
  fits.

- **Keys.** In the `ctrl+t` branch, check ladder rows 2 and 3 before
  `_try_file_completion_tab()`. In this phase, row 3 shows the hint only.
  - Accept-all: override `action_cursor_right` in `_prompt_text_area_edit_actions.py`.
    When our ghost is visible, insert it through our own accept and arm again; otherwise
    keep today's behavior. INSERT `ctrl+f` already maps to `cursor_right`. Add `ctrl+l`
    and `alt+f` handling in INSERT mode.
- **Clearing.** Clear the ghost and disarm the chain in:
  - `_enter_normal_mode`
  - `on_blur`
  - `_clear_active_completion_state`
  - submit, cancel, history, editor, and finder actions
  - `ctrl+c`
  - `_update_file_completion_panel` when a menu opens
  - snippet expansion and tabstop moves
  - xprompt argument hints

  `_soft_completion_blocked()` also returns True while a ghost is visible.

- **Prompt bar.** Add `show_next_word_hint(text)` and `hide_next_word_hint()` for the
  border subtitle. Mirror `show_soft_completion`/`hide_soft_completion`, so the mode
  subtitle is restored.
- **Cold model.** An explicit press schedules the warm and shows `warming next words…`.
  `_refresh_visible_prompt_prediction_surfaces()` then shows the ghost if the chain is
  still armed.
- **CSS.** Add the `.text-area--suggestion` rule for the prompt input (§4).
- **Tests.**
  - pure helpers
  - pilot tests for:
    - each ladder row
    - type-through
    - backspace, cursor move and undo clearing the ghost
    - one undo step per accept
    - accept-all arming again
    - `esc` followed by `right` inserting nothing
    - pane switch
    - multi-pane isolation
    - mid-line and near-wrap-edge suppression
    - the soft-completion block
    - cold and missing model
    - `next_word: off`

  Use deterministic tiny corpora compiled through the facade.

- **PNG goldens.** Chain ghost with hint; no-guess hint.
- **Docs.** Add a "Next-word prediction" subsection under `### Completion` in
  `docs/ace.md`, with the ladder, the keys, and one line on the principles. Update the
  INSERT table.

### 9.7 Phase `next-word-menu`

- **Completion kind.** Add `NEXT_WORD_COMPLETION_KIND = "next_word"`. Its candidates
  carry
  `NextWordCompletionMetadata(score, probability, support, order, continuation, context_words)`.
- **Panel wiring.**
  - A classify flag in `_prompt_input_bar_completion_panel_kinds.py`.
  - A title branch in `completion_panel_title`: `next word ⇢ “<last ≤3 context words>”`.
  - `_next_word_rows.py`, reusing `_ranking_signal_rows` with the sequence color and
    glyph from §4: the word, a meter from `probability`, and a dim `⇢ continuation`.
    Degrade by width without clipping the word, as `_history_word_rows.py` does.
  - Subtitle `[^T] accept  [^D] delete`.
- **Behavior.**
  - Ladder row 3 now opens this menu (limit 5) when the gate fails, or when a confident
    ghost cannot be shown (mid-line or no width). If there are no candidates, it shows
    the hint.
  - Ladder row 4b: in `_try_history_word_completion_tab`, and in the prompt-word-only
    path when history words are disabled, run the explicit next-word request instead of
    clearing when the cursor is at the end of a prose word and no current-word candidate
    exists.
  - Accept inserts the separator (§3) and arms the chain. Extend the ctrl+t accept kind
    set.
  - `ctrl+d` forgets the word through the history-word deletions store: it is hidden
    immediately and persisted. `_completion_supports_delete` covers `next_word`.
- **Tests.** Cover rows 3 and 4b, accept and continue, ctrl+n/p then ctrl+t, ctrl+d
  persistence and exclusion, and structural contexts never opening the menu.
- **Goldens and docs.** Menu goldens at 120x40 and a narrow width. Update the docs.

### 9.8 Phase `context-ranking`

- **Ranking.** Call `rank_prefix` synchronously when building prompt-word and
  history-word results (`_prompt_word_completion_result`, `_build_history_word_result`,
  and `build_indexed_history_word_completion_result`), with the text before the word
  range.
  - When `word_ranking: smart` is set and the model is warm, candidates the model
    matched (order ≥ 1) are promoted to the top, ordered by model score. The rest keep
    their current order.
  - The candidate sets, unique commit, and shared-extension behavior are unchanged.
- **Signals.** Add an optional `sequence` signal (default 0.0) and a `sequence` reason
  to `_RankingSignals` and `HistoryWordCompletionMetadata`.
  - `build_score_meter` adds the violet share in a fixed relation → recency → frequency
    → sequence order.
  - `format_reason_chip` renders `⇢ <context words>`, truncated to 16 cells.
  - `ranking_signal_legend` shows `⇢ context` only when a visible row has a sequence
    signal.
  - Promoted prompt-word rows (plain rows) get a dim trailing `⇢`.
- **Tests and goldens.** Promotion order, a cold model leaving the order unchanged,
  legend presence, meter distribution, and history-word menu goldens.

### 9.9 Phase `replay-harness`

- **Replay evaluator.** In `sase-core`, add
  `prompt_prediction/replay.rs::evaluate_prompt_prediction_replay(rows, options) -> PromptPredictionReplayReportWire`.
  It does a prequential replay:
  1. Warm on the oldest 40% of typed rows.
  2. Score every position of each later row, then add that row.
  3. Split results into cohorts: novel (under 30% 5-gram overlap with prior text), mid,
     and near-duplicate (70% or more).

  Record per-position stats once, then sweep preset and grid thresholds over those
  records without replaying again.

- **Report.** It contains aggregates only, never prompt text:
  - top-1 and top-3
  - gated coverage and precision
  - ghost keystroke-savings upper bound
  - chain run lengths
  - predict latency p50/p95
  - corpus bytes
- **Binding.** Bind it as `evaluate_prompt_prediction_replay` and move the pin again.
- **Tool.** Add `tools/prompt_prediction_replay`, an executable Python script that
  reuses `prompt_prediction_rows`. Flags: `--confidence`, `--sweep`,
  `--sources history[,archive]`, `--json`. It prints rich tables.
- **Calibration.** Adjust the preset constants in Rust:
  - `balanced`: precision ≥ 75% on all prompts and ≥ 65% on novel prompts, maximizing
    coverage
  - `cautious`: ≥ 85%
  - `eager`: ≥ 60%

  Record the aggregate table and method in the `docs/rust_backend.md` prompt-prediction
  section. Include tests for the evaluator on synthetic corpora.

### 9.10 Phase `archive-source`

- **Extraction.** `src/sase/history/prompt_prediction_archive.py`:
  - Enumerate the enabled projects' agents-sidecar prompt archives through the
    Rust-backed `prompt_archive_inventory` facade
    (`src/sase/core/prompt_archive_facade.py`). Bound it to the most recent 6 months and
    1.5M tokens.
  - Use `PromptArchiveDocument.body`, stripping the rendered header bullets and link
    tables.
  - Keep only top-level, user-launched prompts when the archive records that. Otherwise
    rely on the core origin heuristic.
  - Tag each row with its project key and an epoch.
  - Dedup paragraphs by normalized hash, both within the archive (swarm copies) and
    against local history rows.
- **Corpus.** Compile it with `prune_singleton_contexts` as role `archive`, weight 0.25.
  - Use a cheap source token: month-directory mtimes and counts.
  - Rebuild only after first paint and on token change, at most every 10 minutes, off
    the pump.
- **Config and tool.** Add `next_word_sources` (§7). The replay tool gains
  `--sources history,archive`.
- **Default decision.** Include `archive` in the default only if the harness shows **+1
  point or more top-3 on novel prompts at equal coverage**, and near-duplicate top-1
  does not drop by more than 1 point. Otherwise the default stays `[history]` and the
  docs describe the opt-in.
- **Memory.** Report the RSS delta and keep it at or under 60 MB.
- **Tests.** Cover extraction, dedup, and weighting.

### 9.11 Phase `auto-mode`

- **Config.** Add `auto` to `next_word` in every §7 location.
- **Behavior.** In the post-key path of `_on_key` (next to
  `_open_auto_reference_completion_after_change`), when `next_word: auto` and the
  inserted character is a space:
  - the space must follow a word token
  - the cursor must be at end of line
  - the ghost preconditions must hold

  Then predict. If the gate passes, show the ghost with no leading separator. Everything
  else is the ghost-chain contract.

- **Cost.** Prediction runs only on that space keystroke.
- **Tests, golden, and docs.** Include the behavior after `, ` (a ghost may appear)
  versus after `. ` (never, because the context is `<s>` only).

## 10. Verification (whole epic)

- Rust: `sase tool run check` in `sase-core`, plus the `#[ignore]` performance test run
  once per engine change.
- sase: `sase tool run check` per phase, targeted `just fix-tui-screenshots` for each
  visual change, and live `sase screenshot` captures of this flow:
  1. type `Can you help me imple`
  2. `ctrl+t` twice
  3. ghost visible
  4. `ctrl+t`: word taken, ghost advances
  5. `ctrl+f`: rest taken
- Replay harness numbers are recorded, and presets are calibrated before the epic lands.

## 11. Risks

- **Ghost at soft-wrap edges.** Textual splices the ghost without re-wrapping. This is
  handled by the end-of-line-only rule and width fitting to whole words, with the menu
  as the fallback.
- **Gate drift as the corpus grows.** Support counts distinct observations, and presets
  are re-tuned from the harness.
- **Heuristic origin misses** in legacy rows, such as `%dispatch`-assembled prompts.
  Origin recorded at write time is the durable fix, and the heuristic ages out.
- **Privacy.** Predictions can resurface text from history, the same exposure as Ctrl+K.
  Mitigations: the secret, hash and URL filters, deletions (`sase prompt delete`
  rebuilds through the shard token, and Ctrl+D hides immediately), and everything stays
  local.
- **Binding-window lag.** Released `sase` blocks on the `sase-core` release via the
  existing release lane. Source installs build `sase_core_rs` locally. A stale wheel
  degrades the feature to silence.
