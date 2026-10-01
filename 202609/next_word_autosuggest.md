---
tier: epic
title: 'Next-word autosuggest: automatic ghosts, mid-word completion, and a mid-sentence
  peek'
goal: 'Confident next-word guesses appear automatically as you type in the prompt
  input, with no Ctrl+T needed to see them. They complete the word you are typing,
  and they appear at line ends, before closing punctuation, and as a calm bordered
  peek in the middle of a sentence. The prose text never jumps. Ctrl+T after a space
  asks for the next words, and the recent-files menu moves to Ctrl+G r. The plan and
  gate feedback note editor gets the same autosuggest.

  '
phases:
- id: core-word-completion
  title: Gated current-word completion in the sase-core prediction engine
  depends_on: []
  size: medium
  description: 'core-word-completion: add an opt-in complete_current_word request
    and a word_completion result to sase_core::prompt_prediction. It uses a prefix-restricted
    gate with a conservative denominator, per-preset min_prefix_chars, typed-case
    suffixes, and gated continuation after the completed word, with no wire schema
    bump.

    '
- id: word-completion-calibration
  title: Replay calibration, Python wire, bench, and core pin for word completion
  depends_on:
  - core-word-completion
  size: medium
  description: 'word-completion-calibration: add a mid-word replay mode and calibrate
    min_prefix_chars per preset. Mirror the new request/result fields in the Python
    wire and facade, extend the replay tool with --midword and draft-length --bench
    buckets to choose the synchronous-draft threshold, move the sase-core pin, and
    record the results in docs/rust_backend.md.

    '
- id: boundary-ctrl-t
  title: Ctrl+T at a word boundary requests next words; recent files move to Ctrl+G
    r
  depends_on: []
  size: small
  description: 'boundary-ctrl-t: at a prose boundary with no token, Ctrl+T now runs
    the explicit next-word request (ghost, menu, or hint) instead of opening file
    history. Recent files and artifacts move to a new ctrl-g-only `r` continuation.
    Update the hints, help modal, docs, and tests.

    '
- id: inline-ghost-placement
  title: Inline ghost placement before closing punctuation, calmer hints, and module
    split
  depends_on:
  - boundary-ctrl-t
  size: medium
  description: 'inline-ghost-placement: split the next-word mixin into a host-neutral
    ghost display layer and pure placement helpers. Add the before-closer inline placement
    with tail-aware width fitting, and make Ctrl+L accept-all everywhere with fish
    keys only at true end of line. Typing-triggered hints wait for a 350 ms reveal
    beat, boundary auto triggers work away from line end, and ghosts are allowed in
    the legacy feedback mode.

    '
- id: mid-sentence-peek
  title: Mid-sentence next-word peek in the prompt border
  depends_on:
  - inline-ghost-placement
  size: medium
  description: 'mid-sentence-peek: where an inline ghost would shift prose, show a
    styled violet peek in the prompt bar''s border subtitle. It trims words the text
    after the cursor already has, degrades by width, appears after the reveal beat
    when typing-triggered, and is taken with Ctrl+T (one word) or Ctrl+L (all) using
    the menu separator rules.

    '
- id: midword-autosuggest
  title: Mid-word autosuggest from the core word completion
  depends_on:
  - word-completion-calibration
  - mid-sentence-peek
  size: medium
  description: 'midword-autosuggest: in auto mode, each typed word character requests
    complete_current_word. The UI composes suffix plus continuation as an inline ghost
    or peek, keeps silent on a core that lacks the field, filters deleted words, and
    guards keystroke latency with a measured synchronous-draft threshold and an off-pump
    deferred path.

    '
- id: gate-note-autosuggest
  title: Autosuggest in the gate input panel note editor
  depends_on:
  - midword-autosuggest
  size: medium
  description: 'gate-note-autosuggest: host the ghost display layer in the GateInputPanel
    note editor (plan and epic feedback, gate notes), with a host-neutral prediction
    accessor, the note editor''s own hint surface, and Ctrl+T/Ctrl+L accept keys.
    There is no menu there; tests and a golden cover it.

    '
- id: autosuggest-default
  title: Make auto the default and finish docs, help, goldens, and live captures
  depends_on:
  - gate-note-autosuggest
  size: small
  description: 'autosuggest-default: flip ace.prompt_completion.next_word from chain
    to auto in every config location and the default-contract tests. Consolidate the
    docs/ace.md next-word narrative and the help modal, rerun the next-word visual
    goldens, and capture the live screenshot walk.'
proposed_by: bbugyi200.athena.0u0
create_time: 2026-09-30 16:38:17
status: done
bead_id: sase-1dq
---

- **PROMPT:** [prompts/202609/next_word_autosuggest.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/next_word_autosuggest.md)
- **BEAD:** [sase-1dq](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1dq/README.md)

# Plan: Next-word autosuggest

## 1. Summary

Epic `sase-1cj` built a local, confidence-gated next-word engine in `sase-core`, plus a
ghost-text chain in the prompt input. Today a guess is visible only in these cases:

- after an explicit `Ctrl+T` word commit (`next_word: chain`, the default), or after a
  typed space at end of line (opt-in `next_word: auto`);
- only when the rest of the cursor's line is blank and the ghost fits the cursor's
  wrapped row;
- only at a word boundary, never while a word is being typed.

This epic turns that into a fish-style **autosuggest**. Guesses appear on their own
while you type forward. They complete the word you are in the middle of. They work at
line ends and before closing punctuation, and they have a new surface, the **peek**, for
the middle of a sentence. `Ctrl+T` after a space becomes "what comes next?". The rarely
used recent-files menu moves to `Ctrl+G r`. The gate/plan feedback note editor gets the
same autosuggest. The final phase makes `auto` the default.

### 1.1 Can ghost text always be shown? No.

Textual's `TextArea.suggestion` is spliced into the cursor's _logical_ line at the
cursor column (`_render_line`, `Text.assemble(line[:col], suggestion, line[col:])`). It
is then divided with wrap offsets computed _without_ the suggestion. So:

- **Mid-sentence on a single visual row**, the rest of the sentence slides right by the
  ghost's width on every keystroke, and it can run past the right edge and be clipped.
- **In a soft-wrapped paragraph**, every later wrapped row is shifted by the ghost's
  length. Words split at stale wrap points and the paragraph's tail is clipped.
- **Only two positions render cleanly**: at the true end of a line, and before a short
  tail of closing punctuation (for example `)`, `."`, `**`). Both also need the cursor
  to be on the logical line's _final_ wrapped row with room left.

The design therefore uses an inline ghost only where it renders cleanly. The middle of a
sentence gets the **peek**: a styled `⇢ word…` chip in the prompt's bottom border. It
never moves your text (§4).

## 2. Design principles

1. **Never insert a guess you have not seen.** (kept) Every inserted word was first
   visible as ghost text, peek, or a highlighted menu row.
2. **Silence when unsure.** (kept) The Rust core owns the gate. It never shows a guess
   from unigram-only evidence or a `<s>`-only context, or inside structural syntax.
3. **The text never jumps.** A guess never shifts prose. The inline ghost appears only
   where it cannot reflow text, and everything else uses the peek.
4. **Calm while typing, instant when asked.** Typing-triggered hints and peeks wait for
   a short pause (the _reveal beat_, 350 ms). Explicit actions (`Ctrl+T`, accepts)
   respond in the same keystroke. An inline ghost is shown immediately, because you type
   through it.
5. **Typing forward suggests; deleting or moving hides.** Backspace, delete, cursor
   motion, undo/redo, paste, mode changes, blur, and menus clear the guess. It comes
   back only on the next typed character or explicit request.
6. **Visible state decides behavior, and the hint names the keys.** `Ctrl+T` always
   takes one word and `Ctrl+L` always takes everything. `→`, `Ctrl+F`, and `Alt+F` take
   the ghost only at the true end of a line (fish parity). Everywhere else they stay
   cursor motion.
7. **Never block typing.** Predictions stay synchronous only below a measured
   draft-length threshold. Above it they are deferred off the Textual pump.
8. **The boundary rule holds.** Prediction, gating, prefix completion, and casing live
   in `sase-core`. Python owns placement, display, timers, and keys.

## 3. Modes, triggers, and contexts

`ace.prompt_completion.next_word` keeps its three values, and all three are permanent
user choices (config, not a feature flag):

| Mode                     | Typing-triggered guesses                                                                                                          | Explicit `Ctrl+T` / accepts                                |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| `off`                    | none                                                                                                                              | none; `Ctrl+T` after whitespace keeps opening recent files |
| `chain`                  | none                                                                                                                              | full ladder (§5), all placements                           |
| `auto` (default at land) | after each typed character: a word character asks for completion of the current word, any other character asks for the next words | full ladder (§5), all placements                           |

- **Triggers (auto).** The trigger runs in the post-key path of `_on_key` after a typed
  printable character or space, next to today's `_maybe_auto_next_word_ghost`.
  - If the keystroke was consumed by a visible ghost (Textual trims the suggestion and
    the ghost stays valid), keep the ghost and make no request.
  - If the character after the cursor is a word character, stay silent: you are editing
    inside a word.
  - Otherwise predict. A typed word character sends `complete_current_word: true` to the
    core. Any other character sends a plain next-word request.
- **Never triggered by** Backspace/Delete, cursor motion, undo/redo, paste
  (`events.Paste` does not pass through `_on_key`), programmatic edits (auto-pair edits,
  snippet expansion, completions), mode changes, or focus changes.
- **Global blocks (unchanged, plus one).** Guesses are blocked in these states: NORMAL
  or VISUAL mode, an open completion menu, an active snippet session, the Jinja panel
  showing, and a non-empty selection (new). The core's structural blocks still apply.
- **Contexts.**
  - Every prompt stack pane: agent, mini-xprompt, and snippet-target panes. Each pane
    keeps isolated state.
  - The coder-prompt `approve_prompt` mode.
  - The legacy `feedback` bar mode, now unblocked for next-word only; soft completion
    stays blocked there.
  - The GateInputPanel note editor, where plan and epic feedback and gate notes are
    typed (phase `gate-note-autosuggest`).

## 4. Placement and visual design

### 4.1 Placement classification

The rule is pure and host-neutral, and lives in the new
`src/sase/ace/tui/widgets/next_word_placement.py`. It classifies the cursor as follows:

| Placement     | Condition                                                                                                                                                                                           | Surface      |
| ------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ |
| `none`        | The character after the cursor is a word character (`[A-Za-z0-9_'’-]`), or a global block holds                                                                                                     | nothing      |
| `inline_eol`  | The rest of the logical line after the cursor is blank                                                                                                                                              | inline ghost |
| `inline_tail` | The rest of the line holds only a **closing tail**: at most 8 non-space characters from `) ] } " ' ” ’ . , ; : ! ? … * _` plus spaces. Backticks are excluded, because code spans block in the core | inline ghost |
| `peek`        | Anything else (prose follows the cursor), or an inline placement where not even the first ghost word fits                                                                                           | border peek  |

Rules for the two inline placements:

- The cursor's wrapped row must be the **last wrapped section** of its logical line.
- The ghost plus the shifted tail must fit the remaining cells of that row. Width
  fitting drops whole trailing words and caps at `next_word_max_words`, generalizing
  today's `fit_next_word_ghost` and `_next_word_available_width` to count the tail's
  cells.
- If the first word cannot fit, the placement becomes `peek`.

### 4.2 Inline ghost

The look is unchanged: Textual's `.text-area--suggestion` in `$text-muted`, with the
cursor block on its first cell. Two changes:

- It may now sit before a closing tail. The tail slides right by the ghost's width,
  which is only a few cells on the final row, so nothing reflows.
- The border hint is uniform across every placement: `[^T] word  [^L] all`.
  `NEXT_WORD_GHOST_HINT` changes from `[^T] word  [^F] all`.

```text
╭──────────────────────────────────────────────────────────────────────╮
│ Can you help me implement▌it now                                     │    (" it now" in $text-muted)
╰────────────────────────────────── [^T] word  [^L] all · Ln 1, Col 26 ─╯

│ Please check the logs (see the▌parser output)                        │    (" parser output" ghost before ")")

│ Can you help me imple▌ment it now                                    │    (mid-word: "ment it now")
```

### 4.3 Peek (mid-sentence)

The peek is the gated guess rendered in the prompt bar's **border subtitle**. That is
where next-word hints already live, so the text area never changes. The peek is a
`rich.text.Text` built by a pure helper:

- `⇢` (the `SEQUENCE_GLYPH` from `_ranking_signal_rows.py`) in bold `SEQUENCE_COLOR`
  (`#AF87FF`), the same violet as the next-word menu and context chips.
- The **first word**, which is what `Ctrl+T` inserts, in bold using the theme's `$text`.
  Mid-word, this is the whole completed word (`implement`).
- The remaining preview words in `$text-muted`.
- Then two spaces and `[^T] word  [^L] all` in the existing subtitle hint style.

```text
╭──────────────────────────────────────────────────────────────────────────────╮
│ Can you help me ▌the parser in src/foo.rs and make sure the tests pass       │
│ afterwards.                                                                  │
╰──────────────────── ⇢ review it now  [^T] word  [^L] all · Ln 1, Col 17 ─╯
                        └violet┘└bold┘└muted┘
```

- **Width degradation.** First drop trailing preview words, then `[^L] all`, then
  `[^T] word`. A word is never cut. If `⇢ <first>` does not fit next to the cursor
  readout, no peek is shown, and the readout keeps priority, as in the existing
  `_render_subtitle` contract. `_render_subtitle` and `show_next_word_hint` accept
  `str | Text`.
- **Redundancy trim.** Compare the peek words, casefolded, with the words that follow
  the cursor (skipping whitespace and punctuation). Cut the peek before the first peek
  word that equals the first following word. If that is the first peek word, show no
  peek. A guess the text already contains is never offered. For example, with
  `help me ▌the plan`, a guess of `review the plan` peeks as `⇢ review`.
- **Timing.** An explicit request or accept shows the peek immediately. A
  typing-triggered peek appears only after the reveal beat, and only if text and cursor
  are unchanged. A peek that has not been revealed is not visible, so `Ctrl+T` treats it
  as ladder row 3: an explicit request that reveals it immediately. It never inserts it.
- **Validity.** The peek is valid only while the text and cursor equal its snapshot. Any
  context change clears it and restores the mode subtitle. While a peek is visible,
  `_soft_completion_blocked()` returns True, as it does for the ghost.

### 4.4 Other hints

- `no next-word guess` and `warming next words…` stay as transient hints for explicit
  requests.
- When `Ctrl+T` at a whitespace boundary finds nothing, the hint teaches the moved
  feature: `no next-word guess  [^G r] recent files`.
- The next-word menu keeps its current look (`next word ⇢ "<context>"`, violet meter,
  `⇢ continuation` previews).

## 5. Keys

With a guess visible and no menu open, in INSERT mode:

| Key            | Inline ghost at end of line                              | Inline ghost before a closing tail    | Peek                  |
| -------------- | -------------------------------------------------------- | ------------------------------------- | --------------------- |
| `Ctrl+T`       | take one word\*                                          | take one word\*                       | take one word\*       |
| `Ctrl+L`       | take all                                                 | take all                              | take all              |
| `→` / `Ctrl+F` | take all (fish parity)                                   | unchanged motion (e.g. step past `)`) | unchanged motion      |
| `Alt+F`        | take one word (fish parity)                              | unchanged word motion                 | unchanged word motion |
| typing         | matching characters consume it; others re-predict (auto) | same                                  | re-predict (auto)     |

\* Mid-word, the first "word" is the rest of the current word (`ment`). Each accept is
one undo step and predicts again in the same handler, so there is no flicker. Peek
accepts use the next-word menu's separator rules (`_accept_next_word_completion`): a
leading space unless one is already there, and a trailing space when an identifier
character follows, with the cursor landing after the word.

**The `Ctrl+T` ladder** (first match wins):

1. A `prompt_word` / `history_word` / `next_word` menu is open on a real row: accept it
   (unchanged).
2. An inline ghost or a revealed peek is visible: take one word.
3. **Explicit next-word request**, in any of these states:
   - the chain is armed;
   - the cursor is at a prose boundary with no token under it (after whitespace, at line
     start, or after an opening bracket/quote; this was the file-history slot);
   - the cursor is at the end of a prose word with no current-word candidate (old row
     4b).

   A confident guess shows immediately as an inline ghost or peek. Otherwise the
   `next_word` menu opens. With nothing to offer, a hint appears. A cold model shows
   `warming next words…`.

4. Otherwise, the unchanged dispatcher runs (structured tokens, paths, prompt words,
   history words).

**Recent files and artifacts** (the `file_history` menu) move to **`Ctrl+G r`**. It is a
`ctrl_g_only` continuation, so bare NORMAL-mode `gr` is not claimed, and it works from
INSERT and NORMAL. With `next_word: off`, `Ctrl+T` after whitespace still opens recent
files, because there is nothing else to request. `Ctrl+R` seeding from a highlighted
recent-file row keeps working.

## 6. Core: gated current-word completion

This work lives in `sase_core::prompt_prediction` in the linked `sase-core` repo. The
wire changes are additive, with no schema bump. sase's Python reader requires
`schema_version == 1` exactly and ignores unknown keys. No struct uses
`deny_unknown_fields`, so an old core simply ignores the new request field.

- **Request.** `PromptPredictionRequestWire.complete_current_word: bool`
  (`#[serde(default)]`).
- **Result.**
  `PromptPredictionResultWire.word_completion: Option<PromptPredictionWordCompletionWire>`
  (`#[serde(default, skip_serializing_if = "Option::is_none")]`). The new struct is
  `PromptPredictionWordCompletionWire { prefix, word, suffix }`:
  - `prefix` is exactly as typed;
  - `word` is the completed word with the typed casing kept;
  - `suffix` is the characters to insert. It may be empty when the typed word is already
    the predicted word.
- **Partial-word detection.** This applies only when `complete_current_word` is set and
  `text_before_cursor` ends in a word character under the tokenizer's word rules, after
  its leading-affix stripping.
  - The trailing token is the prefix. The context is the text before it, built the same
    way as `rank_prefix`. A blocked context yields no completion.
  - Text ending in whitespace or boundary punctuation is an ordinary request with
    `word_completion: None`. The result is identical to today's.
  - A prefix that cannot be a word (structural, hash-like, secret-like, over 32
    characters) blocks as today.
- **Minimum prefix.** Each preset gains `min_prefix_chars`. Seed values are cautious 3,
  balanced 2, eager 1, and phase `word-completion-calibration` sets the final values. A
  shorter prefix returns a non-confident result with no completion.
- **Restricted gate.** At each order, restrict the successor distribution to keys that
  start with the casefolded prefix, including the exact prefix key.
  - Compute `p` and margin over the restricted distribution with a **conservative
    denominator**: restricted mass plus the context's truncated (dropped) successor
    mass. Truncation must never inflate `p`.
  - Support is the restricted distinct support.
  - These rules are unchanged: evidence-order selection (needs a real word, never `<s>`
    alone, never unigram-only), `min_support`, and the `reject_conflicts` veto.
  - The draft source counts the sequences with the partial word removed. That matches
    the boundary request and the replay's `keys[..pos]`. Draft candidates are
    prefix-filtered.
  - `excluded_words` are never completed.
- **Casing.** Keep the typed prefix exactly and take the suffix from the canonical
  surface. If the typed prefix is all caps with 2 or more letters, uppercase the suffix.
  If the surface's casefold does not extend the typed casefold character for character
  (`’` maps to `'`), return no completion.
- **Continuation.** On a gate pass, `ghost` holds the gated continuation after the
  completed word, up to `max_words − 1` words, because `max_words` counts the completed
  word. `confident` is true. `continuation()` needs no change, since it only reads
  `pass.word`.
- **Menu.** `candidates` are the prefix-restricted ranked words, honoring `limit` with
  continuation previews. The TUI sends `limit: 0` from its autosuggest paths.
- **Performance.** The prefix restriction costs at most 32 successors × 4 orders per
  source. Extend the ignored release perf test with prefix requests and report their
  p95. Boundary requests must not regress.

## 7. Architecture and module layout (sase)

- `src/sase/ace/tui/widgets/next_word_placement.py` (pure, new):
  - placement classification (§4.1), with the closing-tail set as a named constant;
  - tail-aware fitting;
  - redundancy trim;
  - `build_next_word_peek_text(...)`, the styled, width-degrading peek;
  - composition of the inline ghost from `word_completion` plus `ghost`.

  Nothing here needs a mounted widget.

- `src/sase/ace/tui/widgets/_next_word_ghost_display.py` (host-neutral mixin, new, split
  out of `_prompt_next_word.py`, which is already 560 lines):
  - ghost and peek state and their validation on context change;
  - the reveal-beat timer: a thin synchronous `set_timer` callback, generation-checked
    and cancelled on clear;
  - the auto trigger and the synchronous/deferred predict dispatch;
  - accept one and accept all;
  - the hint surface hook `_next_word_hint_surface()`. It returns an object with
    `show_next_word_hint(str | Text)` and `hide_next_word_hint()`. PromptTextArea
    returns its prompt bar.
- `src/sase/ace/tui/widgets/_prompt_next_word.py` (prompt-only): the chain, the `Ctrl+T`
  ladder, and the `next_word` menu.
- `src/sase/ace/tui/widgets/next_word_completion.py` keeps its pure helpers and hint
  constants.
- Keep new modules at or under about 500 lines. Split into siblings rather than growing
  files.

## 8. Phases

Conventions for every phase:

- Before changing TUI code, read `sase/memory/tui.md` and its `tui_perf` child with
  `/sase_memory_read`. Before finishing, read `lint_and_test.md`.
- Run `sase tool run check` in each repo you touch.
- When rendered TUI output changes, run `just fix-tui-screenshots -- <selectors>`. Hand
  long runs to `/sase_monitor`. Inspect the report and every golden change; generation
  is not approval.
- For visible behavior, also capture a live `sase screenshot` of the flow and inspect
  the PNG.
- In `sase-core`, open the repo with `sase repo open sase-core` and follow its
  `AGENTS.md`:
  - use `just`, never bare `cargo`;
  - free functions over serde `*Wire` structs, with `thiserror` errors;
  - facade-only `mod.rs`, files of at most 1,500 lines, and imports by module path;
  - finish with `sase tool run check`.
- After a binding-visible core change, move `sase-core-revision.txt` past it, per the CI
  pin section of `docs/rust_backend.md`. Never touch the `pyproject.toml` version
  window.
- If Symvision flags a public symbol that a later phase consumes, whitelist it with
  `--epic-symbol <this epic's bead id>(<symbol>)` per `tools/AGENTS.md` and
  `sase/memory/symvision.md`. The consuming phase removes that entry.
- Keymap and config changes update the help modal (`PROMPT_INPUT_SECTION` in
  `src/sase/ace/tui/modals/help_modal/binding_common.py`) and `docs/ace.md`. Check
  `src/sase/default_config.yml` for any keymap entry; the `Ctrl+G` prefix table itself
  is code, not config.

### 8.1 Phase `core-word-completion` (sase-core)

Implement §6 in `crates/sase_core/src/prompt_prediction/`:

- **Wire.** Add the new request and result fields and
  `PromptPredictionWordCompletionWire` in `wire.rs`. Update the request struct literals
  in `model.rs`, `replay.rs`, and `tests/`.
- **Detection.** Add partial-word detection that shares the tokenizer's word rules. Put
  it in `tokenize.rs` or a small sibling module.
- **Restricted gate.** Implement the prefix-restricted pass.
  - The simplest integration overrides the per-order `masses`, `distincts`,
    `draft_masses`, and `project_masses` in `PassCtx` with restricted sums. Then
    `combined_at_order`, `top_two_from_table`, and `gate_from_table` work unchanged.
  - Use the conservative denominator from §6.
  - `DraftCounts.pairs` has no per-context index. Either scan the pairs or add a small
    index, and keep every existing request result-identical.
- **Presets.** Add `min_prefix_chars` with the seed values in `predict.rs`.
- **Completion.** Build the casing-preserving suffix, then run continuation from the
  completed word.
- **Tests.** Cover:
  - partial word vs trailing space vs trailing punctuation;
  - an exact-word top giving an empty suffix plus continuation;
  - casing: Title, ALLCAPS, and `’`;
  - `min_prefix_chars` per preset;
  - restricted gate boundaries (p, margin, support, conflicts) and the conservative
    denominator under truncation;
  - the draft with the partial word removed;
  - excluded words;
  - structural and secret-like prefixes blocking;
  - `complete_current_word: false` staying result-identical: every existing test passes
    unchanged;
  - deterministic ties;
  - `max_words` counting the completed word.
- **Perf.** Extend the ignored perf test with prefix requests and report their p95.

### 8.2 Phase `word-completion-calibration` (sase-core, then sase)

- **Replay (core).** Add a mid-word mode to `replay.rs`.
  - For each scored position whose target word is longer than `k` characters, with `k`
    in 1..=4, evaluate the restricted gate and completion.
  - Report per preset and per `k`, as optional report fields: coverage, precision
    (completed word equals target), keystroke savings (`target_chars − k`, plus correct
    continuation words), and the novel/mid/near-duplicate cohorts.
  - Add a parity test modeled on `replay_matches_production_ranking_and_gate`.
- **Calibrate `min_prefix_chars`.** For each preset, choose the smallest `k` whose
  precision meets that preset's word-boundary target on all prompts (cautious ≥ 85%,
  balanced ≥ 75%, eager ≥ 60%) and is at most 10 points lower on novel prompts. Commit
  the constants.
- **Python wire.** In `src/sase/core/prompt_prediction_wire.py`:
  - add `PromptPredictionRequest.complete_current_word: bool = False`, including in
    `to_dict`;
  - add a frozen `PromptPredictionWordCompletion` and
    `PromptPredictionResult.word_completion: ... | None`, parsed with `.get` so an old
    core yields `None`.
  - Also update the facade in `src/sase/core/prompt_prediction_facade.py` if needed.
- **Tool.** `tools/prompt_prediction_replay` gains `--midword`, which prints the
  mid-word tables.
  - `--bench` also times prefix requests and buckets both request kinds by draft length
    (≤1k, ≤4k, ≤10k, ≤20k characters).
  - Choose `NEXT_WORD_SYNC_MAX_DRAFT_CHARS`: the largest bucket whose p95 for the
    typing-path request (`limit: 0`, `complete_current_word` on) is ≤ 1 ms on real
    history. If no bucket qualifies, use the ≤1k bucket.
- **Docs.** Add a "Current-word completion" subsection to `docs/rust_backend.md`'s
  prompt-prediction section. Record the wire, the gate, casing, the calibration table
  and method, the bench numbers, and the chosen threshold, which phase
  `midword-autosuggest` consumes.
- **Pin.** Move `sase-core-revision.txt` past the core commits.
- **Tests.** In `tests/core/test_prompt_prediction_facade.py`, cover the word-completion
  round trip and a result dict without `word_completion` parsing to `None`.

### 8.3 Phase `boundary-ctrl-t`

- **Dispatch.** In `_try_file_completion_tab` (`_file_completion_tab.py`), the
  `token_info is None` branch no longer calls `_try_file_history_completion()`. When
  `next_word` is not `off`, it calls a new `_try_next_word_boundary_request()` in
  `_prompt_next_word.py`, which works like `_try_next_word_word_end_fallback`: arm the
  chain at the cursor, show a visible ghost, or run `_explicit_next_word_request()`.
  - When the request ends with no guess, the transient hint is
    `no next-word guess  [^G r] recent files`, a new constant in
    `next_word_completion.py`.
  - With `next_word: off`, the branch keeps opening file history.
- **`Ctrl+G r`.** Append
  `_PromptGPrefixBinding("r", "open_recent_file_history", "_g_prefix_label_recent_files", "_g_prefix_available_recent_files", ctrl_g_only=True)`
  to `_PROMPT_G_PREFIX_BINDINGS`.
  - The action delegates to `self.active_text_area()._try_file_history_completion()`, as
    `edit_definition_under_cursor` does.
  - The label reads "recent files".
  - It is available whenever a prompt pane is active.
  - Append it last. The hint panel shows 12 rows and the no-guess hint teaches the key.
- **Tests.** Cover:
  - whitespace `Ctrl+T` with a confident corpus giving a ghost, a weak corpus giving the
    `next_word` menu, and none giving the new hint;
  - `off` opening file history;
  - `Ctrl+G r` opening `file_history` from INSERT and NORMAL;
  - bare NORMAL `gr` not being claimed;
  - `Ctrl+R` seeding from a highlighted recent-file row;
  - placeholder, VCS, directive, xprompt, and `@` dispatch at whitespace being
    unchanged, since they run before this branch;
  - the ordered-list assertions in
    `tests/ace/tui/widgets/test_prompt_g_prefix_hint_entries.py`.
- **Docs and goldens.**
  - `docs/ace.md`: the INSERT table (`Ctrl+T` row text and a new `Ctrl+G r` row), the
    Completion key table, the "File-history completion" bullet (now `Ctrl+G r`), and the
    `Ctrl+R` paragraph.
  - The help modal.
  - Add a golden for the whitespace no-guess hint.

### 8.4 Phase `inline-ghost-placement`

- **Split.** Create `next_word_placement.py` and `_next_word_ghost_display.py` (§7).
  Move ghost state, validation, fitting, accepts, and hint calls behind the hint-surface
  hook, and keep behavior identical except for the changes below.
- **Placement.** Implement §4.1. `inline_tail` is new.
  - Ghosts need an empty selection.
  - The `peek` placement shows nothing in this phase. An explicit request whose
    confident ghost cannot be placed still opens the menu, as today.
- **Keys.**
  - `Ctrl+L` takes all at every placement.
  - `→` and `Ctrl+F` (the `action_cursor_right` override in
    `_prompt_text_area_edit_actions.py`) and `Alt+F` take the ghost only at
    `inline_eol`. Otherwise they are plain motion, so `→` before `)` steps past it.
  - `NEXT_WORD_GHOST_HINT` becomes `[^T] word  [^L] all`.
- **Reveal beat.** Add `NEXT_WORD_REVEAL_DELAY_MS = 350`.
  - A typing-triggered ghost shows its hint only after the beat, and only if text and
    cursor are unchanged.
  - Explicit actions show the hint immediately.
  - The timer is a thin synchronous callback, cancelled whenever the guess clears
    (`tui_perf` rule 2).
- **Boundary auto trigger.** Replace `next_word_auto_space_eligible`'s end-of-line
  requirement with the placement classification. In auto mode, a typed non-word
  character following a word predicts next words. Inline placements render. A `peek`
  placement renders nothing until the next phase.
- **Feedback mode.** Drop the `feedback`-mode block from `_next_word_ghost_allowed`.
  Soft completion's block stays.
- **Tests.**
  - Pure: every placement and tail variant, a tail that is too long, a backtick, a word
    character after the cursor, a non-last wrapped section, and tail-aware fitting.
  - Pilot:
    - a ghost before an auto-paired `)` after typing inside parentheses;
    - `→` before `)` moving across it while `Ctrl+L` accepts;
    - `→`, `Ctrl+F`, and `Alt+F` accepting at end of line;
    - the reveal beat: no hint right after an auto ghost, a hint after `pilot.pause`;
    - an explicit `Ctrl+T` hint appearing immediately;
    - a selection suppressing ghosts;
    - the feedback-mode ghost;
    - a highlighted span after the cursor keeping its style next to a tail ghost;
    - multi-pane isolation;
    - chain and `off` behavior unchanged.
- **Goldens.** Update `next_word_ghost_120x40` and `next_word_auto_space_120x40` for the
  new hint text. Add `next_word_ghost_before_closer_120x40`.

### 8.5 Phase `mid-sentence-peek`

- **State.** Add
  `NextWordPeek(anchor_offset, anchor_text, words, word_completion | None, revealed)` in
  the host-neutral layer, with the §4.3 validity rules and clearing everywhere the ghost
  clears. A visible peek sets `_soft_completion_blocked()`.
- **Rendering.** Build the peek with `build_next_word_peek_text` (§4.3):
  - `SEQUENCE_GLYPH` and `SEQUENCE_COLOR` from `_ranking_signal_rows.py`;
  - `$text` and `$text-muted` resolved from `self.app.theme_variables`, as the search
    pill does;
  - `_render_subtitle` and `show_next_word_hint` accepting `str | Text`;
  - width degradation against the real usable width and the cursor readout.
- **Behavior.**
  - `peek` placements from explicit requests and accepts show immediately.
  - Auto peeks reveal after the beat.
  - Apply the redundancy trim.
  - `Ctrl+T` takes one word and `Ctrl+L` takes all, with the menu separator rules and
    one undo step each. Both predict again and show the next guess immediately.
  - An explicit confident guess that cannot be inline now becomes a peek. The menu
    remains for non-confident explicit requests.
  - Chain mode shows peeks only from explicit requests and accepts.
- **Tests.**
  - Pure: the redundancy trim, and peek text at 120, 70, and 40 columns with and without
    a search pill.
  - Pilot:
    - an auto peek after a mid-sentence space appearing only after the pause;
    - `Ctrl+T` before the reveal revealing without inserting;
    - `Ctrl+T` after the reveal inserting and predicting again;
    - `Ctrl+L` taking all in one undo step;
    - the trailing-space and cursor rule before a following word;
    - suppression when the next word is already present;
    - a wrapped mid-paragraph end of line that cannot fit becoming a peek;
    - Backspace, cursor motion, and blur clearing the peek;
    - pane switch and multi-pane isolation;
    - `off` and `chain` modes.
- **Goldens.** Add `next_word_peek_120x40` and `next_word_peek_narrow_70x24`.

### 8.6 Phase `midword-autosuggest`

- **Plumbing.** `_predict_next_words(..., complete_current_word=False)` passes the flag
  into `PromptPredictionRequest`. `_post_filter_deleted_words` also drops a completion
  whose word is deleted in memory.
- **Trigger.** In auto mode, a typed word character with no word character after the
  cursor requests `complete_current_word=True` with `limit=0`. A still-valid consumed
  ghost is kept.
- **Old-core safety.** If the request asked for completion, the text ends in a word
  character, and `word_completion` is `None`, show nothing. An old wheel ignores the
  flag and would otherwise predict after a half-typed word.
- **Composition.**
  - The inline ghost is `suffix` plus, when continuation exists, a space and the
    continuation words. An empty suffix means the ghost starts with that space
    (`implement▌ it now`).
  - The peek's first word is the completed `word`.
  - `Ctrl+T` first finishes the word, then takes one word per press;
    `split_next_word_one` already handles both shapes.
- **Latency guard.** Typing-triggered requests run synchronously while
  `len(text_before_cursor) ≤ NEXT_WORD_SYNC_MAX_DRAFT_CHARS` (from §8.2).
  - Above that, defer. Use a thin `set_timer(debounce_ms)`, then `spawn_pump_free_task`
    with `asyncio.to_thread(predict)`, and apply only when the generation, text, and
    cursor are unchanged. Mirror `_prompt_soft_completion.py`.
  - Confirm that the frozen `PromptPredictionModel` handle is safe to call off-thread.
    If it is not, keep the deferred call on the UI thread inside the thin timer.
- **Chain mode.** No mid-word guesses; `Ctrl+T` mid-word keeps the current-word menus.
- **Tests.**
  - `imple` after a known context shows `ment it now`;
  - matching keystrokes consume the ghost, and a divergent key re-predicts with no blank
    frame;
  - an exact word shows ` it now`;
  - `Ctrl+T` finishes the word, then continues;
  - Title and ALLCAPS casing;
  - a deleted word is never completed;
  - an old-core fake result means silence;
  - the deferred path applies only when current;
  - chain mode shows no mid-word ghost;
  - a word character after the cursor keeps silence.
- **Goldens.** Add `next_word_midword_ghost_120x40` and `next_word_midword_peek_120x40`.

### 8.7 Phase `gate-note-autosuggest`

- **Host.** `_NoteInput(VimModeRoutingMixin, VimTextArea)` in
  `src/sase/ace/tui/modals/gate_input_panel.py` mixes in the host-neutral ghost display
  layer.
  - Extract the prediction accessors (`_predict_next_words`,
    `_prompt_prediction_is_cold`, `_schedule_prompt_prediction_load`, and the session
    disable) from `FileCompletionPredictionMixin` into a host-neutral
    `_next_word_prediction_access.py`. They only need `self.app`.
  - `FileCompletionPredictionMixin` keeps using that module.
- **Behavior.**
  - Respect `next_word`: `off` shows nothing, `chain` responds to explicit `Ctrl+T`
    only, and `auto` is full autosuggest.
  - Inline ghost and peek, with the keys from §5. There is no completion panel, so a
    non-confident explicit `Ctrl+T` shows `no next-word guess` instead of a menu.
  - Verify that no key conflicts with the GateInputPanel bindings (`ctrl+s`, `escape`,
    the gate keymaps) or with VimTextArea INSERT bindings. `ctrl+t` is unbound there
    today, and `ctrl+f` and `alt+f` are motions that the end-of-line rule overrides.
- **Hint surface.** Use the note editor's container border subtitle, or the panel's
  existing mode/indicator line. There must be no layout shift. The peek reuses the same
  Text builder.
- **Project.** Resolve the prediction project from the gate request's project when it is
  cheaply in memory; otherwise use `None`.
- **Tests, golden, and docs.** Add pilot tests for the ghost, peek, accepts, clears, and
  the `off` and `chain` modes. Add the golden `gate_note_next_word_ghost_120x40` and
  document it in the gate input panel docs.

### 8.8 Phase `autosuggest-default`

- **Default.** Flip `next_word` from `chain` to `auto` in all of these:
  - `src/sase/default_config.yml`;
  - the `src/sase/config/sase.schema.json` default and description;
  - the `PromptCompletionSettings` default and the fallback in `_parse_next_word_mode`
    (unknown values fall back to the default);
  - the `_next_word_settings` fallback in the ghost/prompt mixins;
  - the default-contract tests in `tests/test_config_schema_ace.py`;
  - `docs/configuration.md`.

  This needs no feature flag, because `chain` and `off` remain permanent choices.

- **Docs.** Rewrite the `docs/ace.md` "Next-word prediction" subsection as the final
  autosuggest narrative:
  - modes and triggers;
  - placements, with a sentence on why mid-sentence uses the peek;
  - keys and the `Ctrl+T` ladder;
  - the reveal beat;
  - contexts, including gate notes;
  - principles.

  Then do a final pass on the INSERT table, the Completion key table, and the help
  modal.

- **Goldens and screenshots.** Run the next-word and g-prefix visual goldens and inspect
  every change. Capture the live `sase screenshot` walk (§9).
- **Cleanup.** Remove any `--epic-symbol` whitelist entries left for this epic.

## 9. Verification (whole epic)

- `sase-core`:
  - `sase tool run check`;
  - the ignored perf test, run once per engine change;
  - the replay `--midword` tables recorded.
- sase:
  - `sase tool run check` per phase;
  - targeted `just fix-tui-screenshots` per visual change, with reports inspected.
- Live `sase screenshot` walk, in `auto` mode:
  1. Type `Can you help me imple`. The mid-word ghost `ment it now` shows.
  2. Press `Ctrl+T`. The word finishes and the ghost advances.
  3. Press `Ctrl+L`. The rest is taken.
  4. Move the cursor mid-sentence and type a space. After a pause, the violet peek
     appears in the border.
  5. Press `Ctrl+T`. The word is inserted with correct spacing.
  6. Type inside `()`. The ghost appears before `)`, and `→` steps past `)`.
  7. Press `Ctrl+T` after a space with no guess. The `[^G r] recent files` hint shows.
  8. Press `Ctrl+G r`. The recent files menu opens.
  9. In a gate note, type the same phrase. The ghost appears.
- Keystroke latency: the `--bench` p95 for the typing-path request at the chosen
  threshold is recorded in `docs/rust_backend.md`.

## 10. Risks and mitigations

- **Autosuggest noise.**
  - The core gate is unchanged, and the mid-word gate is calibrated by replay with
    `min_prefix_chars`.
  - Hints and peeks wait for the reveal beat. Deleting and moving stay silent.
  - `chain` and `off` remain.
- **Keystroke latency.** Draft re-counting is the known cost; `sase-1cj` land note #3
  covers the unmet §5.4 budgets, which still need an owner decision. Measure the
  typing-path request, keep it synchronous only below the measured threshold, and
  otherwise defer off the pump. This epic does not change the budgets.
- **Textual splice artifacts.**
  - Inline ghosts are limited to the final wrapped row, at end of line or before a
    closing tail of at most 8 characters.
  - Highlights come from the highlight map applied before the splice, so they keep their
    spans.
  - A strip post-processor that uses wrapped offsets (`_restore_todo_selection`) only
    runs with a selection, and ghosts are suppressed with a selection.
  - A golden covers a highlighted span after a tail ghost.
- **Old wheel.** An old core ignores `complete_current_word`. The UI requires
  `word_completion` before rendering a mid-word guess, so the result is silence, not a
  wrong guess.
- **Muscle memory.** `Ctrl+T` after a space no longer opens recent files. The no-guess
  hint names `Ctrl+G r`, and `next_word: off` keeps the old slot.

## 11. Deferred (not in this epic)

- Autosuggest in the bead note, create, and edit modals (`Ctrl+T` is already the
  audience toggle in the bead note modal), the single-line questions and sudo inputs,
  and `sase-nvim` over the xprompt LSP.
- A cursor-anchored floating chip for the peek, which would need an overlay layer and
  cursor-offset tracking. The border peek is the reliable first surface.
- Local acceptance counters feeding back into the gate, and any LLM phrase suggestion.
- The `sase-1cj` item (g) budget decision (draft-count redesign or revised budgets).
