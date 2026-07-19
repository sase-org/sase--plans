---
tier: tale
title: Make adjacent inline code recognition robust
goal: 'Inline code is recognized consistently in the ACE prompt editor even when spans
  are adjacent to punctuation or prose, while launch-time literal semantics, xprompt
  argument parsing, Unicode offsets, and prompt responsiveness remain in lockstep.

  '
create_time: 2026-07-19 08:37:24
status: wip
prompt: 202607/prompts/inline_code_adjacency.md
---

# Plan: Make adjacent inline code recognition robust

## Context and root cause

The screenshot demonstrates a lexical-boundary bug, not a styling bug. In the prompt input, the first span in
`` `foo`/`bar` `` receives the existing inline-code chip treatment, but the second does not. `CodeBlockHighlightMixin`
already paints every range returned by `inline_literal_ranges`; the missing range originates in
`src/sase/xprompt/_inline_code.py`, where `_can_open` accepts a backtick run only at line start or after whitespace and
the small `([{"'` context shared by xprompt markers. A slash is outside that set, so the second opener is skipped.

That scanner is also part of launch-time literal-zone protection. Fixing only `_codeblock_syntax_highlight.py` would
make the editor visually promise that code is inert while the launch path could still expand an xprompt, directive,
alternative, or Jinja expression inside it. The change therefore needs one authoritative lexical contract shared by the
prompt editor, syntax inspectors, prompt preprocessing, and the Rust launch backend.

This is a **tale**: it is a focused grammar correction with coordinated consumers and tests that one coding agent can
implement and validate as a unit. It does not need independently landable product phases or multiple parallel agents.

## Recognition contract

Adopt context-free, CommonMark-style placement for the existing deliberately single-line inline-code subset:

- Any unmasked backtick run may open a span, regardless of the preceding character. It closes at the nearest unmasked
  run of exactly the same length on the same line. This recognizes punctuation-adjacent and word-adjacent forms such as
  `` `foo`/`bar` ``, `` `foo`,`bar` ``, and ``prefix`value`suffix`` instead of maintaining an arbitrary punctuation
  allowlist.
- Preserve the current exact-run behavior for nested shorter runs, the single-line restriction, and the rule that an
  unmatched run produces no literal span. Do not claim full CommonMark whitespace normalization or multiline code-span
  support as part of this fix.
- Preserve parser precedence by masking fenced blocks, disabled regions, and the complete spans of recognized xprompt
  references and directives before inline scanning. Thus `` #name:`arg with spaces` `` and `` %model:`custom model` ``
  remain active colon-argument syntax, and backticks inside parenthesized arguments remain argument content rather than
  new inline literals.
- Once recognized, an inline span remains launch-inert and suppresses xprompt, directive, alternative, skill, and Jinja
  overlays throughout the whole range. The highlighted result and submitted behavior must continue to describe the same
  semantics.
- Define and test the offset contract explicitly for non-ASCII prompt text. Rust naturally scans UTF-8 byte offsets,
  while existing Python/Textual consumers use absolute character offsets; conversion must be linear, exact, and
  centralized at the binding/adapter boundary.

## Authoritative scanner and launch parity

Move the low-level inline-code delimiter scan into the linked `sase-core` repository, following the repository rule that
shared backend/domain behavior belongs in Rust. Open that repository through `/sase_repo` before implementation. Make
the scanner a reusable core primitive rather than another private copy inside the fan-out planner:

- Accept the source text plus precomputed masked ranges, merge masks once, and return ordered, non-overlapping ranges.
  Removing the opener-context gate should simplify the hot loop; retain bounded linear scanning and avoid regex
  backtracking or a full Markdown parser on each keystroke.
- Add a small PyO3 binding for the range primitive and a thin Python adapter. The adapter should be the only place that
  translates Rust byte offsets and Python character offsets. Retire the independent Python delimiter algorithm (or
  reduce its module to that adapter) so future delimiter fixes cannot drift between frontends.
- Compose the new ranges into every directive-sensitive Rust launch-planning path (model, repeat/name, alternatives, and
  wait), reusing the same literal-zone contract wherever those paths currently reason about fenced or disabled regions.
  Add focused parity tests, but do not broaden this work into an unrelated rewrite of fenced-block parsing.
- Keep reference/directive-specific mask construction at the layer that owns those grammars. In Python,
  `_literal_zones.py` can continue composing fenced/disabled ranges with full recognized reference/directive spans
  before calling the core scanner. In Rust launch planning, derive equivalent masks from the core's canonical
  directive/reference inspection rather than reviving the old left-context shortcut.

Treat the added binding as a coordinated core/SASE contract. Follow `sase-core`'s release-plz rules (no manual crate
version edits), expose the function from both the pure Rust crate and `sase_core_rs`, and update the SASE dependency
floor/lockfile and required-binding checks when the new core release is consumed. A stale accepted wheel must fail a
published-core compatibility gate rather than first failing while a user types in ACE.

## Python and prompt-editor integration

Keep `inline_literal_ranges` as the single Python composition entry point so all current consumers inherit the new
recognition automatically:

- `CodeBlockHighlightMixin` should continue to consume those ranges and emit the existing `codeblock.inline` and
  `codeblock.delimiter` spans. No new CSS, palette, or custom line renderer is needed; the existing quiet-chip style is
  already visible and theme-aware.
- `protect_fenced_blocks`/`code_literal_ranges` and the processor, directive, Jinja, history, retry, and metadata paths
  should receive the broader literal ranges through their existing shared helper rather than through call-site edits.
- `xprompt_inspect`, `alt_inspect`, `jinja_inspect`, xprompt argument assistance, and read-only prompt previews should
  continue excluding the same ranges. Add direct regression assertions where necessary to prove that the newly
  recognized second span cannot receive an actionable overlay.
- Preserve the existing 80 KB / 1,200-line live-overlay guards and fail-open painting behavior. The new scanner and
  binding call must remain pure, synchronous, memory-only work; no disk access, subprocess, timer, or background task
  belongs in the keystroke path.

## Regression and compatibility coverage

Build a compact grammar matrix around the reported example rather than adding only a one-off slash special case:

1. In `sase-core`, cover adjacent spans separated by slash and other punctuation, word-adjacent spans, equal-length
   delimiter runs containing shorter runs, unmatched runs, CRLF/single-line boundaries, overlapping/adjacent masks, and
   Unicode before and inside spans. Exercise the PyO3 result shape and offset conversion as part of binding tests.
2. In `tests/test_xprompt_inline_code.py`, replace the old assertion that mid-word/colon contexts categorically cannot
   open with the new delimiter-placement contract. Retain and strengthen end-to-end regressions proving that recognized
   xprompt/directive colon arguments and parenthesized argument backticks are still parsed normally, while xprompts,
   directives, alternatives, and Jinja inside punctuation-adjacent inline spans survive verbatim.
3. In `tests/ace/tui/widgets/test_prompt_codeblock_highlight.py`, use the exact `` `foo`/`bar` `` shape and assert two
   complete inline ranges plus both pairs of delimiter overlays at the correct offsets. Also assert that semantic
   overlays remain absent inside the second chip and present immediately outside it.
4. Update the existing code-highlight visual fixture to include the adjacent-span example in realistic prose. Run its
   solo and stacked dark/light PNG cases, inspect actual/expected/diff artifacts, and accept only the intentional newly
   highlighted span in the four goldens.
5. Extend the existing code-heavy tokenizer benchmark with repeated punctuation-adjacent spans and keep its p95 below
   the established 16 ms target at the overlay guard scale. This guards the Python/Rust range conversion as well as the
   scanner itself.

## Validation and delivery

- In `sase-core`, run Rust formatting, Clippy with warnings denied, the focused scanner/binding/fan-out tests, and the
  full Cargo workspace test suite.
- In SASE, build/install the linked core binding, run the focused inline-literal, launch-processing, syntax-inspector,
  widget, visual, and benchmark tests, then run `just install` followed by the required `just check`.
- Run the published-minimum `sase-core-rs` binding smoke after updating the dependency range, and audit for any
  remaining duplicate delimiter scanner or stale documentation that still describes the old leading-context restriction.
- Do not edit SASE memory files or generated instruction shims; the user requested a code/plan change, not a memory
  update. No keymap or `default_config.yml` changes are involved.

The result is intentionally broader than adding `/` to an allowlist: every syntactically valid unmasked single-line code
span gets the same visible chip and literal behavior, while xprompt argument forms retain explicit precedence and the
live prompt path stays within its current performance envelope.
