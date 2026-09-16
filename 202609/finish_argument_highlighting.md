---
tier: epic
title: Finish structured argument highlighting and prove frontend parity
goal: Argument highlighting remains structured while typing, preserves names and multiline
  or nested values across ACE and LSP, respects directive colors, and meets the existing
  responsiveness and acceptance contracts.
parent_bead: sase-11i
phases:
- id: core-editing
  title: Complete incremental argument spans and remove suffix reparsing
  size: medium
  depends_on: []
  description: 'core-editing: preserve structural spans in unfinished calls and make
    directive extraction use bounded shared parsing.'
- id: lsp-output
  title: Emit complete LSP names and argument coverage
  size: medium
  depends_on:
  - core-editing
  description: 'lsp-output: emit invocation and directive names and preserve multiline
    and partially overlapped argument spans.'
- id: tui-palette
  title: Preserve directive argument colors and responsive editing
  size: medium
  depends_on:
  - core-editing
  description: 'tui-palette: consume span source in theme selection and integrate
    structured highlighting with keyword completion and warm catalogs.'
- id: real-parity
  title: Verify actual frontends, snapshots, and input latency
  size: medium
  depends_on:
  - lsp-output
  - tui-palette
  description: 'real-parity: replace the synthetic LSP mapping check with real-server
    parity and complete visual and performance acceptance.'
proposed_by: bbugyi200.athena.sase-11i.land
create_time: 2026-09-16 00:36:10
status: wip
bead_id: sase-11i.6
---

- **PROMPT:** [prompts/202609/finish_argument_highlighting.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/finish_argument_highlighting.md)
- **PARENT:** [202609/xprompt_keyword_arg_highlighting.md](https://github.com/sase-org/sase--plans/blob/main/202609/xprompt_keyword_arg_highlighting.md)
- **BEAD:** [sase-11i.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11i/sase-11i.6.md)

# Finish structured argument highlighting

## Scope and audit evidence

This is remaining work from the interrupted landing of `sase-11i`, whose approved design
is `plan:202609/xprompt_keyword_arg_highlighting.md`. Read that artifact and the parent
bead's landing note. All original phases are closed, but the source does not yet satisfy
the approved editing, rendering, parity, and performance contracts. Keep the existing
shared Rust grammar, byte-offset wire representation, append-only LSP legend,
cold-catalog sentinel, and graceful rendering fallback.

The audited feature commits are `sase-core` `4c25db2` and `f07bf53`, `sase-nvim`
`502e716`, and `sase` `6847172922` and `649be3cb27`. The implementation is substantial
and should be repaired in place, not repeated. Open other repositories through
`sase repo open` and use its printed path; never assume a sibling checkout path.

Confirmed gaps:

1. `crates/sase_core/src/editor/xprompt_args.rs::parse_parenthesized` marks an
   unfinished call open and returns before parsing its body.
   `xprompt_argument_spans("#foo(key=42, other=true")` consequently returns only the
   opening `arg_delimiter`. Keys, assignments, typed values, and commas appear only
   after the closing parenthesis. This contradicts structural highlighting during
   editing and is especially visible with the keyword-completion work that landed in
   `sase` `a98b96e510` while the parent epic was running.
2. `argument_spans.rs::parse_directive_call_at` constructs a synthetic string for the
   entire remaining document and calls `parse_xprompt_calls`, even though it consumes
   only the first result. Repeating this for every directive makes the new synchronous
   highlighting path quadratic. Five-sample installed-binding measurements of repeated
   `%queue(capacity=2, priority=3) #foo(key=42)` lines gave medians of 7.18 ms for 100
   lines, 147.52 ms for 600 lines, and 375.67 ms for 1,000 lines. The largest input is
   43,999 bytes, within the 80,000-byte and 1,200-line guard. These are diagnostic
   timings, not widget p95 measurements.
3. `crates/sase_xprompt_lsp/src/semantic_tokens.rs` appends `function` and `macro` to
   its legend but never produces tokens of those types. It also drops every multiline
   argument value in `push_token`. Its overlap filter discards an entire argument value
   when a nested artifact token overlaps any part of it, instead of retaining the
   portions around the artifact.
4. In `src/sase/ace/tui/widgets/_xprompt_syntax_highlight.py`, all structured arguments
   use a palette derived from `app_theme.success`; `_text_area_style_name` ignores
   `HighlightSpan.source`. Directive names stay warning-colored while their keys,
   punctuation, and untyped values switch to the xprompt family. `highlight_theme.py`
   and CLI style lookup also currently have one argument palette. Both dark/light
   goldens visibly contain this mismatch.
5. `tests/xprompt/test_argument_surface_parity.py` defines its own LSP type and modifier
   dictionaries and verifies those dictionaries. It never requests semantic tokens from
   the production server, so it passes despite gaps 1–3.

The source review also found a second comma/quote scanner in `argument_spans.rs`. Use
the existing shared argument parser's lexical boundaries rather than growing another
independent grammar while repairing the confirmed defects. Preserve supported quote,
backtick-shorthand, block, nested-parenthesis, colon, double-colon, plus, and
trailing-text forms; do not expand launch syntax accidentally.

## Phase: core-editing — incremental structure and bounded parsing

Work in `sase-core`.

- Parse the available argument body of an open parenthesized call for structural spans.
  Preserve `is_open` so catalog-dependent errors stay suppressed until the call closes.
  Emit keys and `=` even when the value is still empty, typed spans for completed
  values, and all existing top-level commas. Never invent closing punctuation or emit an
  out-of-bounds byte offset.
- Make directive calls use a shared parser entry point that parses exactly the call at
  its offset. Avoid allocating and scanning the entire suffix for every directive. Reuse
  delimiter and clause boundaries for punctuation spans; remove the duplicate scanner if
  those boundaries make it unnecessary.
- Preserve existing diagnostic binding and repeatable-tail semantics. Gate directive
  invalidity for unfinished calls too once they begin returning args. Unresolved names
  and unresolvable values must not become false errors.
- Keep existing wire fields compatible. Expose any additional frontend-neutral name
  spans or parsing entry points needed by the LSP through Rust rather than adding
  frontend lexical copies. Test the PyO3 boundary as well as native Rust.
- Add meaningful regression coverage for `#foo(key=42, other=true`, `#foo(key=`,
  `%queue(capacity=2, priority=`, UTF-8 before and inside calls, supported quoted
  commas/parentheses and multiline `[[...]]` values, all existing shorthand and
  trailing-body forms, and literal-zone exclusion. Test that closed calls retain
  validity while open calls remain structural.
- Record before/after scaling for the repeated mixed-directive workload and
  representative short prompts. Check the hot path for other repeated document scans
  exposed by the measurement. Do not meet the target by dropping supported content or
  shrinking the existing size guards.

Run the repository's required `just check`, including binding tests. Read the repo's
`AGENTS.md` and use the monitor skill when builds or checks are long.

## Phase: lsp-output — complete semantic-token coverage

Work in `sase-core` and update the existing `sase-nvim` smoke coverage if needed.

- Emit `function` for xprompt invocation names and `macro` for directive names, using
  core-owned spans. Cover bare names and names with each supported argument form,
  aliases, and literal-zone exclusion. Do not reorder the first four token types or
  existing modifier indices.
- Split multiline argument spans into valid per-line semantic tokens with correct UTF-16
  coordinates, retaining the role and modifiers. Include CRLF and astral characters in
  tests.
- Resolve overlaps by preserving the non-overlapping pieces of a lower-priority value
  around higher-priority artifact spans. Keep kind/payload/fragment tokens and existing
  code/glossary behavior correct. A container must not erase nested syntax or disappear
  wholesale because a small nested token wins.
- Preserve cold/warm catalog behavior and diagnostics as the validity channel.
  Unknown-key `deprecated` modifiers must still agree with the shared validity result.
  Reuse the server's existing catalog cache.
- Add decoded-token assertions through the real JSON-RPC path for names, open calls,
  multiline strings, and a value with both surrounding text and an artifact reference.
  Assert exact ranges and non-overlap, not just token counts.
- Keep Neovim's glossary filter restricted to unmodified `type` tokens, and keep
  colorscheme-owned standard token styling. Extend the smoke test to assert the newly
  delivered name tokens and multiline coverage without adding redundant overlay groups.

Run `sase-core`'s required checks and the three existing Neovim scripts:
`tests/glossary_highlight.lua`, `tests/xprompt_semantic_highlight.lua`, and
`tests/lsp_argument_semantic_smoke.lua`. Point the smoke test at the binary built from
the checkout under review, not an arbitrary installed server.

## Phase: tui-palette — source-aware colors and completion integration

Work in `sase`.

- Use `HighlightSpan.source` to choose the warning-derived directive argument family and
  success-derived xprompt family for punctuation, keys, and untyped values. Retain the
  specified theme secondary/accent/primary literal refinements and foreground/background
  blend rules. Avoid new hard-coded colors.
- Make the actual TextArea registration and style selector agree, including invalid
  variants, theme changes, and both dark/light themes. Preserve readable validity
  underlining and the value foreground. Check Rich/Textual's supported underline
  attributes before claiming independent error-colored underlines; document any
  representational limit instead of silently making that claim.
- Carry the same source-aware styling through shared/CLI consumers where spans retain
  the source. Keep the existing `derive_argument_color` contract and pager palette
  consumers working; do not alter unrelated pager syntax.
- Exercise keyword completion from `a98b96e510` followed by highlighting before the
  closing parenthesis, with both cold and exact warm catalogs. Verify that accepted
  keys, value typing, and a later catalog refresh preserve the intended structure and
  validity without synchronous I/O.
- Measure catalog conversion costs in the actual widget. If warm entries are repeatedly
  serialized despite an unchanged catalog, cache only with correct identity/invalidation
  or pass the minimal relevant data through the existing adapter. Shared parsing stays
  in Rust, and render paths stay read-only.
- Add regression assertions against resolved TextArea styles, source-aware CLI output,
  nested Jinja/placeholder/artifact precedence, and the binding-failure flat-container
  fallback. Do not merely assert that a style key exists.

Read `tui_perf.md` and `lint_and_test.md` through audited memory reads. Run the focused
tests and `just check` after changes.

## Phase: real-parity — actual consumers and acceptance evidence

Work in `sase`, with source-matched `sase-core` and `sase-nvim` checkouts.

- Replace the test-local LSP lookup proof with a test that actually calls the shared
  core span binding, the production TUI highlighting/role assignment, and the built LSP
  over JSON-RPC for the same controlled text and catalog. Reuse the existing LSP harness
  pattern in `tests/_xprompt_directive_completion_parity_lsp.py` where practical.
- Cover all argument roles and validity states; open and closed calls; names;
  multiline/UTF-16 offsets; artifact nesting and retained surrounding value pieces;
  colon/double-colon/plus/trailing bodies; and cold/warm catalogs. Compare expected
  surface differences explicitly, including LSP diagnostics versus TUI underlines. Tests
  must fail if the actual server stops emitting a required token or changes its mapping;
  a copied expected mapping alone is not evidence.
- Integrate intervening static conditional work (`sase` `d4b409921e`, core `7e56423`),
  keyword completion (`a98b96e510`), and proc queue admission (`sase` `b6b11f2155`, core
  `20f1dce`). Use current directive metadata for `%if(should_run=false)` and
  `%queue(capacity=2, priority=3)`; preserve directive-owned fenced-code exclusion and
  existing completion contracts. Recheck later drift without taking over the active
  `%hold` epic `sase-11l`.
- Extend the current argument PNG scenarios only where needed to cover the fixes,
  regenerate deliberately, then run a clean compare. View both dark/light images: values
  remain readable, punctuation recedes, directive arguments stay in the warning family,
  and nested syntax and invalidity remain visible.
- Record actual `SASE_TUI_PERF=1` input/navigation p95 measurements against the existing
  16 ms target, plus the long mixed-directive case within the guards. Confirm no
  filesystem work, helper process, or unbounded catalog reload enters the synchronous
  highlight path. Address epic-caused stalls before claiming acceptance; microbenchmarks
  alone do not satisfy widget responsiveness.
- Rebuild/install first, then run focused Python/Rust/Neovim checks, CLI rendering
  coverage, `just check-full` through `/sase_monitor` with TESTING/TESTED, and
  `just test-visual`. Preserve concrete logs/results and classify every failure. Fix
  failures caused by this epic, including existing snapshots affected by the deliberate
  highlight change. Do not rebaseline unrelated UI changes wholesale.

The audit workspace's compiled extension was missing, and the ambient installed LSP did
not advertise semantic tokens. These were environment observations, not source
regressions. Install the checkout's extension and LSP in lockstep before claiming
runtime parity. The installed-binding reproductions above supplement the audited source;
final evidence must identify the source revisions actually tested.

## Existing follow-up disposition and handoff

Original phase `sase-11i.5` note #1 reports 600 unrelated full-visual mismatches while
its new argument goldens passed. The land audit followed `/sase_new_task`, searched
current and closed CI tasks, swept recent CI tasks and active epics, and forwarded that
observation as +1 to existing ready `sase-x5` (now +7). No new broad visual task was
created. `sase-10u` separately covers stale Agents-footer goldens. The count of 600 and
the renderer-drift hypothesis remain phase-reported, not independently verified
root-cause evidence. The acceptance phase must distinguish current epic-caused failures
from that backlog and leave precise proposed follow-up notes for its land agent if
further unrelated work is discovered. Subsequent review of `sase-10w.3` note #4 found
that it had already rebaselined 119 goldens on September 14 and observed a green visual
run before four distinct fixture-determinism failures recurred. Therefore neither the
old `sase-x5` count nor the `sase-10u` footer diagnosis proves the cause of the
September 16 report. Use fresh failure artifacts to determine which older records still
apply.

The parent landing audit has already reviewed every original phase note and found no
parent of `sase-11i`. `sase bead epic-symbols sase-11i` returned no exemptions. This
plan's `parent_bead` is the continuation link back to the interrupted landing. No child
phase is assigned the parent close, post-close Symvision run, or original plan status
update; those remain land-agent responsibilities after this remaining implementation and
acceptance work is complete.
