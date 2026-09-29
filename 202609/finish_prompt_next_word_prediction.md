---
tier: epic
title: Finish next-word prediction correctness, budgets, and calibration
goal: 'The next-word prediction feature from epic sase-1cj meets its own contract.
  The core blocks every structural tail, including `%{...}` alternation. Support counts
  and confidence presets are honest and calibrated. Predict meets the p95 ≤ 0.5 ms
  local budget on real history. The TUI ghost never goes stale or inserts in the wrong
  place. The missing goldens, the archive default decision, and the epic-symbol cleanup
  are done, so sase-1cj can close.

  '
phases:
- id: core-correctness
  title: Core tokenizer, support, and origin correctness in sase-core
  depends_on: []
  size: medium
  description: 'core-correctness: block structural and alternation tails at query
    time, treat alternation spans as excluded regions, count support once per row
    instead of adding project partitions, replace the invented origin markers with
    a real inventory, make casing canonicalization deterministic, and add the missing
    gate, blocked-context, and boundary tests.'
- id: tui-fixes
  title: TUI ghost, ranking gate, warm-cache fixes, and epic-symbol cleanup
  depends_on: []
  size: medium
  description: 'tui-fixes: clear the stale Textual suggestion whenever the ghost is
    invalidated, gate context promotion on word_ranking smart, stop the archive and
    empty-history rebuild loops, recompile on deletions-only changes, repair the three
    red next-word tests with a sturdier fixture corpus, and retire all twelve sase-1cj
    epic-symbol whitelist entries.'
- id: core-perf
  title: Meet the prompt prediction latency, compile, and memory budgets
  depends_on:
  - core-correctness
  size: medium
  description: 'core-perf: make the perf test representative, measure production predict
    on real history, and optimize the corpus layout and predict path until p95, compile
    time, and corpus bytes meet their budgets, with identical results.'
- id: recalibrate
  title: Recalibrate presets and settle the archive default
  depends_on:
  - core-correctness
  - core-perf
  - tui-fixes
  size: medium
  description: 'recalibrate: re-run the prequential replay under the corrected support
    semantics, recalibrate balanced and eager, make cautious measurably stricter than
    balanced, finish the history-plus-archive comparison and RSS measurement, and
    record all numbers.'
- id: visual-verify
  title: Goldens, live screenshots, and docs
  depends_on:
  - tui-fixes
  - recalibrate
  size: small
  description: 'visual-verify: add the context-ranking and auto-mode goldens, re-verify
    the next-word goldens after recalibration, capture the live Ctrl+T chain flow,
    promote the docs to a real subsection, and run the final check.'
proposed_by: bbugyi200.athena.sase-1cj.land
parent_bead: sase-1cj
create_time: 2026-09-29 18:29:10
status: wip
bead_id: sase-1cj.12
---

- **PROMPT:** [prompts/202609/finish_prompt_next_word_prediction.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/finish_prompt_next_word_prediction.md)
- **PARENT:** [202609/prompt_next_word_prediction.md](https://github.com/sase-org/sase--plans/blob/main/202609/prompt_next_word_prediction.md)
- **BEAD:** [sase-1cj.12](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1cj/sase-1cj.12.md)

# Plan: Finish next-word prediction correctness, budgets, and calibration

## 1. Context

Epic `sase-1cj` ("Next-word prediction chains in the prompt input") closed all eleven of
its phases. Its landing audit then found epic-caused gaps that must be fixed before the
epic can close. This epic is that remaining work. The original design lives in the
parent epic's plan, `plan:202609/prompt_next_word_prediction.md`; read it with
`sase bead read sase-1cj`. Section numbers like "parent §5.4" below refer to that plan.

Everything below was reproduced on sase `4fce27e507` with sase-core pinned at `1e51ff3`,
using the installed release `sase_core_rs` wheel.

**Repos.** Core work happens in the linked `sase-core` repo. Open it with
`sase repo open sase-core`, read its `AGENTS.md`, and run `sase tool run check` there.
Also run `sase tool run check` in this repo. The commit finalizer's
`repos.linked[].revision_pin` (commit `c257a3f220`) commits the sase-core sibling first
and moves `sase-core-revision.txt` itself. Only hand-edit the pin if the finalizer
evidence shows it did not. After any core change, run `just install` in sase so the
local `sase_core_rs` wheel is rebuilt before running Python tests.

**Accepted deviations.** These are deliberate, so leave them unchanged:

- The binding takes JSON strings instead of `list[dict]`.
- `NextWordChain` stores an anchor-text snapshot instead of an edit generation.
- The ranking signal is named `context`, not `sequence`.
- Promoted prompt-word rows show a full `⇢ <ctx>` chip.
- Inline `#[cfg(test)]` modules are used; this is the repo-wide convention.
- Human `%if`/`%proc` launches record `origin=generated`, because the parent plan lists
  `launch_admission_runtime.py` under generated.
- Archive rows use the document mtime as their epoch.
- `replay.rs` keeps its own scorer, guarded by
  `replay_matches_production_ranking_and_gate`.

## 2. Phase `core-correctness` (sase-core)

All files are under `crates/sase_core/src/prompt_prediction/`.

1. **Structural tails must block (parent §5.4).** In `tokenize.rs`,
   `tokenize_with_excluded` (around lines 378-386) pops trailing empty sequences where
   `started == false`. A mid-sentence structural token, a closed code span or Jinja tag,
   or a `:`/`;` each create one of these sequences. `tokenize_cursor_text` then falls
   back to the words before them.
   - **Reproduced through the binding.** Each of these returns a confident ghost
     `the parser and fix` from context `please look at`:
     - `please look at src/foo.rs`
     - `please look at #gh:sase`
     - ``please look at `x` ``
     - `please look at {{ x }}`
     - `please look at %{a,b}`

     `please look at this:` returns context `please look at this` instead of blocking.

   - **Fix.** When the last non-whitespace token before the cursor is structural or
     non-word, or is a `:`/`;` clause boundary, or ends a closed excluded region, return
     `Blocked`:
     - `structural_tail` when the tail is structural
     - `no_word_context` when the fresh sequence has no word yet
   - **Keep:**
     - a word followed by trailing whitespace still yields that word context (auto mode
       relies on this)
     - sentence-final punctuation still never ghosts
   - Apply the same rule to `rank_prefix`: no matches after a structural tail.
   - Training-side boundaries are unchanged.

2. **Alternation is sacred syntax.** This integrates with sase-1co, which added
   `crate::editor::alternation::scan_alternations` in sase-core `1160ea4` and `1e51ff3`.
   - Treat every `%{...}`, `%(...)` and `%alt(...)` span it reports as an excluded
     region (a hard boundary), at both compile and query time. The spans are byte
     offsets, like the tokenizer's.
   - An unclosed opener before the cursor (`close == None`) blocks with a new
     `BLOCKED_UNCLOSED_ALTERNATION = "unclosed_alternation"`.
   - Test these cases:
     - mid-word `imple%{ment,mentation} it`
     - spaced `%{fix the bug | add a test}`
     - unclosed `%{fix the `
     - an alternation inside a code span, which must stay inert
3. **Support is counted once per row.** `predict.rs` `combined` (around lines 270-300)
   adds `project_word_stats_for(..)` distinct onto the global distinct. The project
   partition is a subset of the same rows, so 2 same-project rows report support 4 and
   pass `min_support` 4.
   - Distinct support (successor support and the context totals used to choose L\*) must
     count each observation once across history, session, archive and draft.
   - The project boost changes only mass, total, and source shares.
   - Make the identical change in `replay.rs` `combined_at` (around lines 490-520).
     `replay_matches_production_ranking_and_gate` must pass.
   - Add a test: 2 same-project rows do not pass `min_support` 4.
4. **Real origin inventory (parent §5.3).** In `origin.rs`:
   - `MARKER_SWARM = "%swarm("` and `MARKER_LEAD = "%lead("` match nothing sase
     produces; `grep -rn "%swarm\|%lead" src/sase` in sase returns zero hits.
   - `MARKER_WORK_TASK` and `MARKER_LAND_EPIC` are declared but never used by
     `looks_generated`.
   - Inventory strings that appear only in machine-assembled prompts. Look in sase's
     `src/sase/xprompts/` templates (for example `tribe.md` and `t.md`), in the
     bead-work prompt builders (`src/sase/agent/launch_cwd_bead_work.py`), and in swarm
     expansion (`src/sase/agent/xprompt_swarm.py`).
   - Give each real marker a named const and a test. Delete the invented and unused
     ones.
   - Record `rows_generated_skipped` on the real history before and after in the phase
     note, as aggregate numbers only.
5. **Determinism.** `tokenize.rs` `canonical_surface` (around line 742) tallies in a
   randomly seeded `std::collections::HashMap`, so casing ties can flip between
   compiles. Tie-break deterministically, for example with `BTreeMap` or an explicit
   surface-order tie-break.
6. **Closing punctuation.** `this.)` and `"really?"` currently lose their sentence
   boundary. Strip the trailing closers before checking for sentence-final punctuation.
7. **Missing tests (parent §9.3):**
   - for each preset, gate boundaries just below and at `min_p`, `min_margin` and
     `min_support`
   - `reject_conflicts` on and off
   - blocked for a backtick span at the cursor and for frontmatter at the cursor
   - a pasted-block line and a `---` separator line as boundaries
   - every structural-tail and alternation case above, for both `predict` and
     `rank_prefix`
8. **Verify.**
   - In sase-core, run `sase tool run check`.
   - In sase, run `just install`, then the prediction suites:
     - `tests/core/test_prompt_prediction_facade.py`
     - `tests/ace/tui/widgets/test_prompt_next_word.py`
     - `test_next_word_menu.py`
     - `test_prompt_context_ranking.py`
     - `test_file_completion_prediction.py`
     - `tests/ace/tui/test_prompt_prediction_cache.py`

   If the support fix changes a fixture corpus's gating, add rows to the fixture. Never
   weaken assertions. If a new public Python symbol is needed and a later phase will
   consume it, whitelist it keyed to this epic's phase ID.

## 3. Phase `tui-fixes` (sase)

1. **The stale ghost is still drawn and inserted (HIGH).** `_validate_next_word_ghost`
   in `src/sase/ace/tui/widgets/_prompt_next_word.py` (around lines 269-289) sets
   `_next_word_ghost = None` but leaves Textual's `self.suggestion` set. The
   `if not self.suggestion: pass` block does nothing. Textual keeps drawing the
   suggestion, and `action_cursor_right` inserts it. Reproduced with `NextWordTestApp`
   after arming on `Can you help me`:
   - `left` then `right` turns the text into `Can you help m implemente`
   - `home` then `ctrl+f` turns it into ` implementCan you help me`
   - `backspace` leaves ` implement` drawn
   - undo leaves the ghost over an empty prompt
   - soft completion is unblocked while the stale ghost is drawn

   Fix: whenever our ghost becomes invalid, clear `self.suggestion`. Only the next-word
   mixin sets `suggestion` in the prompt text area. Extend the pilot tests to assert:
   - `ta.suggestion == ""` after backspace, cursor motion, `home`, undo and redo
   - `left` then `right` is a plain cursor move
   - `home` then `ctrl+f` inserts nothing

2. **Context promotion must respect `word_ranking`.** Parent §9.8 promotes only under
   `word_ranking: smart`. `_rank_prefix_context` in
   `src/sase/ace/tui/widgets/_file_completion_prediction.py` (around line 108) and its
   callers ignore the setting. The callers are `_file_completion_base_panel.py` (around
   line 205, prompt word) and `_file_completion_history.py` (around line 130, history
   word).
   - Return no ranking unless `_prompt_completion_settings().word_ranking == "smart"`.
   - Test that `recent` leaves candidate order, meter and legend unchanged.
3. **Warm-cache loops** in `_load_prompt_prediction_caches`
   (`src/sase/ace/tui/actions/_startup_prompt_prediction.py`):
   - **Archive loop (around lines 391-451).** When an archive rebuild yields no rows or
     fails, `archive_corpus` stays `None` and `archive_built_at` is never set. So
     `want_archive_rebuild` is true on every warm, and every prompt-bar open or submit
     re-reads 24 history shards plus the inventory, contrary to the comment near
     line 447. Set `archive_built_at = now` on every attempted rebuild so the token gate
     and 10-minute throttle apply. Test that three consecutive warms with an empty
     archive build it once.
   - **Empty-history loop (around lines 122-127).** An empty history reruns the full
     rebuild, including the catalog load, on every cold warm. Treat an empty build as
     built for an unchanged token.
   - **Deletions-only changes.** A change to the deletions token alone does not
     recompile the history corpus, so an undeleted word stays excluded. Recompile
     history when `deletions_stale` (parent §9.5 rebuilds on any token change).
   - **Log spam.** Build failures call `log.exception` on every warm (around line 265).
     Log once per session.
   - **Inventory bounds.** In `src/sase/history/prompt_prediction_archive.py` (around
     line 210), read the inventory bounded to the 6 recent month directories if the
     facade accepts `month=`. That avoids loading every month's bodies before the
     filter.
4. **Three red tests on HEAD** in `tests/ace/tui/widgets/test_prompt_next_word.py`:
   `test_arm_after_word_commit_shows_ghost_with_hint`,
   `test_ctrl_t_takes_one_word_then_advances_ghost` and
   `test_type_through_consumes_ghost`.
   - The cause: the replay-calibrated balanced preset (`min_p` 0.75, `min_support` 4,
     sase-core `1ad57ea`) shortens the 5-row fixture's ghost from ` implement it` to
     ` implement`.
   - Rebuild the fixture corpus with enough distinct rows that the two-word chain passes
     balanced with margin. It must keep passing after `core-correctness` counts support
     once per row.
   - Keep every assertion. Run every prediction and next-word suite.
5. **Retire the twelve `--epic-symbol 'sase-1cj(...)'` Justfile entries** in
   `_lint-symvision`. Follow the Symvision hierarchy and the wire-module precedent of
   private `_x_from_dict` helpers and private nested classes (for example
   `src/sase/core/agent_alias_history_wire.py`).
   - **Privatize in-file-only helpers** in `src/sase/core/prompt_prediction_wire.py`
     (add a `_` prefix and drop them from `__all__`):
     - `prompt_prediction_source_shares_from_dict`
     - `prompt_prediction_candidate_from_dict`
     - `prompt_prefix_rank_match_from_dict`
     - `prompt_prediction_replay_gate_metrics_from_dict`
     - `prompt_prediction_replay_cohort_from_dict`
     - `prompt_prediction_replay_sweep_point_from_dict`
   - **Privatize in-file-only classes** the same way, and update the test imports:
     - `PromptPredictionSourceShares`
     - `PromptPredictionReplaySweepPoint`
   - **Add pragmas for the replay tool's imports.** `tools/prompt_prediction_replay`
     imports three symbols, and Symvision does not scan that extensionless script. Put a
     `# symvision: tools/prompt_prediction_replay` pragma directly above each:
     - `PromptPredictionReplayCohort` and `PromptPredictionReplayGateMetrics` (wire)
     - `evaluate_prompt_prediction_replay` (`src/sase/core/prompt_prediction_facade.py`)

     The precedent is `src/sase/core/tool_run.py`.

   - **Privatize the test-seam type.** `ArchivePredictionTarget` in
     `src/sase/history/prompt_prediction_archive.py` becomes `_ArchivePredictionTarget`.
     Update `__all__` and `tests/history/test_prompt_prediction_archive.py`.
   - Delete all twelve entries and add a line to the consumption comment above
     `_lint-symvision`.
   - `just _lint-symvision` must pass, and `sase bead epic-symbols sase-1cj` must list
     nothing.

6. **Docstring nit.** `history_word_completion_subtitle` in
   `src/sase/ace/tui/widgets/_prompt_input_bar_completion_panel_labels.py` (around
   line 484) still names the `[^L] accept  [^D] delete` fallback. The fallback is now
   `[^T] accept  [^D] delete`.
7. **Verify** with `sase tool run check`.

## 4. Phase `core-perf` (sase-core, plus the sase replay tool and docs)

**Measured** on 2026-09-29 with the pinned release wheel. Real local history has 11,537
rows; 3,803 are used, with 283k tokens, 225k contexts and 320k successor entries.

| Metric                                                 | Measured                               | Budget                                   |
| ------------------------------------------------------ | -------------------------------------- | ---------------------------------------- |
| Compile                                                | 0.67-0.76 s (~190 ms per 1k used rows) | 50 ms per 1k                             |
| `approx_bytes`                                         | 27.8 MB                                | ≤ 5 MB local                             |
| Predict p95, defaults (limit 5, max_words 4, draft on) | ~2.5 ms (max ~5 ms)                    | ≤ 0.5 ms local-only, ≤ 1 ms with archive |
| Predict p95, no draft                                  | ~1.5 ms                                | —                                        |
| Predict p95, limit 1 / max_words 1 / no draft          | 0.46 ms                                | —                                        |

Predict was timed over 300 sampled prefixes, 221 of them blocked.

- `docs/rust_backend.md` records p50 ~1.4 ms / p95 ~3.2 ms and ~54 MB. Those numbers
  come from `replay.rs`'s own scorer, not from production predict.
- The `#[ignore]` perf test (`prompt_prediction/tests/mod.rs`, around line 151) passes
  only because it uses 64 formulaic templates and no draft.

**Work:**

1. **Make the `#[ignore]` perf test representative** and have it assert every budget.
   Use about 4k rows of about 75 tokens drawn Zipf-style from a few thousand words, with
   varied lengths and project tags. Requests must include a multi-KB draft.
2. **Measure production predict on real history.** Add a `--bench` mode to
   `tools/prompt_prediction_replay`. It compiles real history through
   `prompt_prediction_rows` and times production `PromptPredictionModel.predict` and
   `rank_prefix` over sampled non-blocked prefixes. It prints aggregates only: compile
   ms, `approx_bytes`, and p50/p95/max.
3. **Profile, then optimize** while keeping results identical. All core tests and
   `replay_matches_production_ranking_and_gate` must pass unchanged, and ordering must
   stay deterministic. Likely levers:
   - Implement parent §5.3's packed context keys. Contexts are currently `Vec<u32>`. Use
     flat successor arrays and no per-entry allocations.
   - Count the draft once per request; it stays frozen during continuation.
   - Reuse per-order context lookups across continuation steps.
   - Compute continuation previews only for the returned candidates.
   - Short-circuit gate failures.
   - Pre-size compile maps.
4. **Budgets.** Predict p95 must be ≤ 0.5 ms local-only and ≤ 1 ms with the archive, for
   the default request with text up to 20k characters. Compile must be ≤ 50 ms per 1k
   prompts. The local corpus must be ≤ 5 MB and the archive ≤ 60 MB.
   - The latency budgets are mandatory.
   - If the 5 MB corpus budget cannot be met without dropping evidence, stop at the best
     lossless layout. Record the measured bytes and the reason in
     `docs/rust_backend.md`, and add a `PROPOSED FOLLOW-UP:` note on this phase bead.
     Never prune silently.
5. **Update the cost numbers** in `docs/rust_backend.md` from `--bench`.

## 5. Phase `recalibrate` (sase-core presets, plus sase tool, config and docs)

1. **Re-run the replay** with sweep (`tools/prompt_prediction_replay --sweep --json`)
   now that support counts once per row. Hand long runs to `/sase_monitor`. Widen the
   sweep grid if needed (`min_p` up to 0.95, `min_support` up to 8).
2. **Presets** in `predict.rs`:
   - **balanced:** the max-coverage grid point with overall precision ≥ 75% and novel
     precision ≥ 65% (parent §9.9).
   - **eager:** the max-coverage point with overall precision ≥ 60%.
   - **cautious:** today `0.75/0.35/4` behaves exactly like balanced `0.75/0.20/4`. With
     `min_p` 0.75 the runner-up share is at most 0.25, so the margin is always met, and
     the docs table shows identical rows.
     - Pick the max-coverage point with overall ≥ 85% and novel precision ≥ balanced's
       novel precision + 5 points, strictly tighter than balanced on `min_p` or
       `min_support`.
     - If no grid point reaches +5 novel, pick the point with the highest novel
       precision among points whose coverage is at least half of balanced's.
     - Cautious must measurably differ from balanced.

   Put the measured numbers in each preset's doc comment.

3. **Archive default (parent §9.10).** The `--sources history,archive` replay did not
   finish within 40 minutes.
   - Re-run it under `/sase_monitor` with a budget of at least 3 hours.
   - If it is still too slow, add a deterministic `--score-every K` option that scores
     only every K-th post-warm row. Compare history and history+archive on that same
     selection.
   - Include `archive` in the `next_word_sources` default only on +1 point or more novel
     top-3 at equal coverage, with near-duplicate top-1 dropping at most 1 point.
   - If the default changes, update every parent §7 location:
     - `default_config.yml`
     - `sase.schema.json`
     - `parse_prompt_completion_settings`
     - `tests/test_config_schema_ace.py`
     - `docs/configuration.md`
4. **RSS.** Measure the process RSS delta of building the history and archive corpora
   (VmRSS before and after, aggregates only). It must stay ≤ 60 MB.
5. **Docs.** Rewrite the `docs/rust_backend.md` calibration and archive subsections
   with:
   - the three-preset table
   - the method
   - the archive verdict with its numbers
   - the RSS
6. **TUI suites.** Re-run the TUI prediction suites. Fix any fixture corpus the new
   presets break without weakening assertions.
7. **Verify** with `sase tool run check` in both repos.

## 6. Phase `visual-verify` (sase)

1. **Goldens** (parent §9.6-§9.8 and §9.11):
   - Add a history-word menu golden that shows context-ranking promotion: the violet `⇢`
     meter share, the `⇢ <ctx>` chip, and the `⇢ context` legend.
   - Add an auto-mode golden: the ghost shown right after a typed space.
   - Re-verify these goldens after recalibration:
     - `next_word_ghost_120x40.png`
     - `next_word_no_guess_120x40.png`
     - `next_word_menu_120x40.png`
     - `next_word_menu_narrow_70x24.png`
     - the prompt-word and history-word completion goldens

   Use `just fix-tui-screenshots -- <selectors>`. The renderer environment must match
   the pinned visual env, so run `just install-visual` first; the default venv has
   drifted. Hand long runs to `/sase_monitor` and inspect the report.

2. **Live flow.** Read the `tui_screenshot` memory, then capture a live
   `sase screenshot` of the parent §10 flow:
   1. type `Can you help me imple`
   2. press `ctrl+t` twice
   3. the ghost is visible
   4. `ctrl+t` takes a word and the ghost advances
   5. `ctrl+f` takes the rest

   Also check that `left` then `right` inserts no stale ghost. Inspect the PNGs.

3. **Docs.** In `docs/ace.md`, turn "Next-word prediction" into a real subsection under
   `### Completion` (it is a bold bullet today). Give it the Ctrl+T ladder, the keys,
   and one line on the principles. Confirm the INSERT-mode table row.
4. **Final check.** Run `sase tool run check`.

## 7. Landing handoff

This epic's parent is `sase-1cj`. Do not close `sase-1cj`, run its Symvision pass, or
mark its plan file done inside any phase. After this epic lands, its land agent resumes
the `sase-1cj` landing through the parent link.
