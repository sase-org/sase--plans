---
tier: tale
title: Prompt-input K word definitions and spellcheck panels (dict + aspell)
goal: NORMAL-mode K on a plain word in the prompt input opens a definition panel for
  correctly spelled words and a single-keypress spellcheck-fix panel for misspelled
  ones, powered by the optional dict and aspell CLI tools, with gated installation,
  sase doctor --deep reporting, and documentation.
create_time: 2026-07-26 12:16:11
status: done
---

- **PROMPT:** [202607/prompts/prompt_word_definitions_spellcheck.md](prompts/prompt_word_definitions_spellcheck.md)

# Plan: `K` Word Definitions & Spellcheck in the Prompt Input (dict + aspell)

## Goal

Extend the prompt input's NORMAL-mode `K` (preview) keymap so it does something delightful for _plain English words_ —
the tokens that today only produce the "Move the cursor onto an xprompt, skill, or file path" warning:

- `K` on a **correctly spelled** word opens a beautiful, scrollable **Word Definition panel** showing one or more
  dictionary definitions for that word.
- `K` on a **misspelled** word opens a compact **Spellcheck panel** listing suggested corrections; a single key press
  (`1`–`9`, or `j`/`k` + `Enter`) replaces the word in the prompt with the chosen suggestion.
- Existing `K` behavior (xprompt / slash-skill / workflow / file preview) is completely unchanged and always wins; the
  word path is strictly the fallback when no other preview target matches.

Two optional CLI tools power this, chosen from the research note `202607/dict_and_spell_cli_tools.md` in the
`sase--research` sidecar repo (open it with `/sase_repo` if you need the full analysis):

- **`dict`** (DICT-protocol client) for definitions — the research's top overall pick: packaged on both Debian and
  Homebrew, stable CLI, multiple dictionaries per query, useful exit statuses for scripting.
- **GNU `aspell`** (+ `aspell-en`) for spell checking — the research's top pick for arbitrary-word validation: its
  Ispell pipe protocol answers "is this a word?" _and_ returns ranked suggestions in a single fast, offline call.

Both are optional. When they are missing, `K` on a plain word degrades to a clear, actionable toast, and
`sase doctor --deep` reports exactly what to install.

## Current Understanding

### The `K` preview chain

- `src/sase/ace/tui/widgets/_vim_normal.py` (line ~62): NORMAL-mode `K` calls `_preview_token_under_cursor()` when no
  pending operator/count is buffering.
- `src/sase/ace/tui/widgets/_prompt_preview.py` — `PromptPreviewMixin._preview_token_under_cursor()`:
  1. Computes the absolute cursor offset and warm slash-skill catalog (`_known_slash_skills_for_action`; may defer with
     a "catalog is still loading" toast and return `None` — that early return must stay untouched).
  2. Calls `detect_preview_target_at_cursor()` (`_prompt_preview_target.py`) which matches, in priority order: slash
     skills, `#xprompt` references, file paths.
  3. On a match, bumps `self._prompt_preview_request_id` (staleness guard) and resolves off-thread via
     `run_worker(... asyncio.to_thread(resolve_preview_target, ...))`, then pushes `PreviewPanelModal`.
  4. On no match, shows the warning toast. **This is the seam where the word fallback goes.**
- `PreviewPanelModal` (`src/sase/ace/tui/modals/preview_panel_modal.py`) is the presentational pattern to mirror: title
  Static + `VerticalScroll` body + footer Static, `j`/`k`/`ctrl+d`/`ctrl+u`/`g`/`G` scroll bindings, `esc`/`q` close.
  Its styles live in `src/sase/ace/tui/styles.tcss` (`PreviewPanelModal #preview-title` block, ~line 4248).

### Adjacent machinery to reuse

- **Word motions**: `_vim_motions.py` has `_char_class`-based `find_inner_word`, but its vim word-class semantics
  (underscores/digits are word chars) are wrong for natural-language lookup; we need a small dedicated extractor.
- **Applying text**: Textual's `TextArea.replace(insert, start, end)` performs an undoable ranged edit — the right
  primitive for applying a spelling fix.
- **Single-key choice modal pattern**: `approve_options_modal.py` — `on_key` acts as an event barrier (stops printable
  keys so they can't leak into app-level handlers), letter keys select rows, `Static` rows re-rendered with a `>` marker
  and `.selected` class on selection change.
- **Optional-tool CLI adapters precedent**: `src/sase/core/clipboard.py` and `src/sase/editor_resolver.py` are the
  established Python-side homes for local-tool glue. This feature is TUI presentation + local subprocess glue, so per
  the Rust-core boundary litmus test it stays in Python; **no `sase-core` change**.
- **Doctor**: `src/sase/doctor/checks_tools.py` has `_OPTIONAL_TOOLS: tuple[_ToolRequirement, ...]` feeding the
  `tools.optional` check, which is already registered with `deep=True` (runs under `sase doctor --deep` / `-D`).
- **Docs surfaces for `K`**: `docs/ace.md` (~line 2941 prose + ~line 3123 keymap table row), help modal
  `src/sase/ace/tui/modals/help_modal/binding_common.py` line ~32 (`("K", "Preview xprompt/skill/file")`, 32-char
  description budget), and the INSTALL.md "Recommended / optional" tool table.

### TUI perf rules that bind this design (from `tui_perf.md` long-term memory)

- Keystroke paths are read-only and prompt-free: the `K` handler itself may only do pure string work; every subprocess
  call happens off-thread inside a `run_worker` coroutine using `asyncio.to_thread` (exactly like
  `_resolve_preview_async` today).
- All subprocess calls get hard timeouts (`dict` talks to dict.org over the network and can hang on restrictive
  networks; `aspell` is local but still bounded).
- Reuse the existing `_prompt_preview_request_id` counter as the staleness guard so a slow lookup never pops a modal
  over newer state; drop results when `request_id` is stale or the widget is unmounted.

## Design

### Behavior decision tree (what `K` does on a plain word)

```
K pressed, no xprompt/skill/file token at cursor
└─ extract lookup word at cursor (letters + internal ' / - only)
   ├─ no word (whitespace, punctuation, digits, identifiers) → existing warning toast,
   │    reworded to include words: "Move the cursor onto an xprompt, skill, file path,
   │    or word to look it up"
   └─ word found → background worker:
      ├─ aspell available → spell-check the word (aspell pipe, one call, ~ms)
      │  ├─ correct → dictionary lookup
      │  │  ├─ dict available, definitions found → WordDefinitionModal
      │  │  ├─ dict available, no match          → info toast: no definition found
      │  │  ├─ dict missing                      → warning toast: install dict (doctor -D)
      │  │  └─ dict error/timeout               → error toast with short reason
      │  └─ misspelled
      │     ├─ suggestions returned → SpellcheckPanelModal (single-key apply)
      │     └─ no suggestions       → warning toast: "'X' is not in the dictionary
      │                                (no suggestions)"
      └─ aspell missing → skip spell gate, go straight to dictionary lookup
         (same sub-branches as above; the "no match" toast additionally hints that
         installing aspell enables spelling suggestions)
```

Sequencing rationale: aspell runs first because it is local and effectively instant, and its verdict picks the panel;
`dict` (potentially a network call) only runs for correctly spelled words, so misspellings never wait on the network.

### New module: `src/sase/core/word_lookup.py`

Pure, Textual-free adapters (unit-testable with monkeypatched subprocess runners). Contents:

- `WordSpan` frozen dataclass: `word: str`, `row: int`, `start_col: int`, `end_col: int` (end exclusive).
- `extract_lookup_word(line: str, row: int, col: int) -> WordSpan | None` — expands from `col` over Unicode letters,
  allowing `'` and `-` only _between_ letters (so `don't` and `state-of-the-art` extract whole, but trailing/leading
  punctuation is excluded). Returns `None` when the character run under the cursor contains digits or underscores
  (identifiers are not natural-language words), when the cursor sits on whitespace/punctuation, or when the word exceeds
  a sanity cap (64 chars). Cursor may be anywhere inside the word, including on an interior `'`/`-`.
- `check_spelling(word: str, *, runner=...) -> SpellCheckResult` — frozen dataclass with
  `status: Literal["correct", "misspelled", "unavailable", "error"]`, `suggestions: tuple[str, ...]`, `detail: str`.
  Implementation: `shutil.which("aspell")` gate → run `aspell -a --encoding=utf-8` (Ispell pipe mode; add `--lang=en_US`
  — hardcoded v1 default, a config knob is a non-goal), write the word to stdin, parse the first result line per the
  documented protocol: `*`/`+`/`-` → correct; `&`/`?` → misspelled with comma-separated suggestions after the colon; `#`
  → misspelled, no suggestions. Timeout ~3 s. A missing English dictionary or nonzero exit surfaces as `status="error"`
  with a one-line `detail` (first stderr line) so the TUI can toast something actionable.
- `look_up_definitions(word: str, *, runner=...) -> DefinitionResult` — frozen dataclass with
  `status: Literal["ok", "no_match", "unavailable", "error"]`, `sections: tuple[DefinitionSection, ...]`, `detail: str`;
  `DefinitionSection` = `source: str` (e.g. `WordNet (r) 3.1 (2024)`), `body: str` (dict's own indentation preserved).
  Implementation: `shutil.which("dict")` gate → run `dict -- <word>` with timeout ~5 s, parse stdout by splitting on
  `From <source> [<db-id>]:` header lines (tolerate any preamble before the first header). Distinguish `no_match` from
  transport errors via dict's documented exit statuses (verify the exact codes against `man dict` after install — the
  research notes distinct codes for "no match" vs connection failures; fall back to treating empty-output nonzero exits
  as `no_match` only for the documented code, everything else as `error`). Subprocesses run with stdout-not-a-tty so
  dict never pages.
- Both runners live behind small injectable callables so tests never spawn real processes.

### TUI wiring: `src/sase/ace/tui/widgets/_prompt_word_lookup.py`

New `PromptWordLookupMixin` (mirrors `_prompt_preview.py`'s shape), mixed into `PromptTextArea` alongside
`PromptPreviewMixin`:

- `_lookup_word_under_cursor() -> bool` — pure/fast: reads `self.cursor_location`, gets the current line from
  `self.document`, calls `extract_lookup_word`. On `None` returns `False` (caller falls through to the warning toast).
  Otherwise bumps `self._prompt_preview_request_id`, launches
  `run_worker(self._resolve_word_lookup_async(span, request_id=...), name=f"prompt-word-lookup:{request_id}")`, and
  returns `True`.
- `_resolve_word_lookup_async(span, *, request_id)` — `await asyncio.to_thread(check_spelling, span.word)`, then per the
  decision tree optionally `await asyncio.to_thread(look_up_definitions, span.word)`. After every await, drop the result
  if `request_id != self._prompt_preview_request_id` or `not self.is_mounted`. Terminal actions: push
  `WordDefinitionModal(payload)`, or push `SpellcheckPanelModal(...)` with a dismiss callback, or `self.notify(...)`
  with the toasts from the tree (severity: `information` for no-definition, `warning` for missing tools / no
  suggestions, `error` for tool failures).
- Suggestion apply (dismiss callback receives `str | None`): re-verify the document still contains exactly `span.word`
  at `(span.row, span.start_col..span.end_col)` (cheap guard against any text drift), then
  `self.replace(suggestion, (row, start_col), (row, end_col))` and move the cursor to the start of the replacement. If
  the guard fails, toast a warning instead of editing. The edit is a normal undoable TextArea edit.
- `_prompt_preview.py` change is tiny: in `_preview_token_under_cursor`, replace the `token is None` warning branch with
  `if self._lookup_word_under_cursor(): return` followed by the (reworded) warning toast.

### New modal: `src/sase/ace/tui/modals/word_definition_modal.py`

`WordDefinitionModal(ModalScreen[None])`, closely following `PreviewPanelModal`'s structure and bindings (`esc`/`q`
close; `j`/`k`, `ctrl+d`/`ctrl+u`, `g`/`G` scrolling):

- Title block: a single-cell gold glyph (use a plain char such as `≡` — avoid emoji to keep cell-width and PNG snapshots
  deterministic), kind label `WORD` in the same bold `#87D7FF` style the preview modal uses, then the word in bold
  white. Second title line (dim): the definition sources, e.g. `dict.org — WordNet (r) 3.1, GCIDE`.
- Body: a Rich `Text` assembled from `DefinitionResult.sections` — for each section a dim-cyan rule-style header line
  (`─── WordNet (r) 3.1 ───`), then the section body verbatim (dict output is already sensibly indented; no lexer, no
  line numbers). Blank line between sections.
- Footer: same hint style as the preview modal (`Ctrl+D/U scroll | j/k line | g/G top/bottom | Esc/q close`).
- Styles: new `WordDefinitionModal` block in `styles.tcss` cloned from the `PreviewPanelModal` sizing (centered, wide,
  bounded height, rounded border) with a distinct accent so it reads as "dictionary", not "file preview".

### New modal: `src/sase/ace/tui/modals/spellcheck_panel_modal.py`

`SpellcheckPanelModal(ModalScreen[str | None])`, following the `approve_options_modal.py` single-key pattern:

- Constructor takes the misspelled word and up to **9** suggestions (truncate aspell's longer list; nine keeps every
  choice a single digit press and the panel compact).
- Title: glyph + `SPELLING` label in a warm warning accent (e.g. `#FF8787`-family), then the misspelled word in bold
  with strikethrough or red underline styling.
- Rows: `1 accommodate`, `2 accommodated`, … rendered as `Static` rows with the `>` selection marker and `.selected`
  class exactly like the approve-options rows; digit press applies that suggestion _immediately_
  (`self.dismiss(suggestion)`), `j`/`k` (and `ctrl+n`/`ctrl+p`) move the selection, `Enter` applies the selected row,
  `esc`/`q` dismisses with `None`.
- `on_key` is an event barrier: stop all printable characters so stray keys never reach the app beneath (copy the
  approve-options pattern, including `event.prevent_default()`/`event.stop()`).
- Footer: `1-9 apply | j/k move | Enter apply | Esc cancel`, dim, matching existing footer hint styling.
- Styles: compact centered container (~56 cols) in `styles.tcss`, selected-row highlight consistent with
  approve-options.

### `sase doctor --deep` integration

In `src/sase/doctor/checks_tools.py`:

- Append two `_ToolRequirement` rows to `_OPTIONAL_TOOLS`:
  - `_ToolRequirement("dict", ("dict",), "prompt word definitions (K on a plain word)")`
  - `_ToolRequirement("aspell", ("aspell",), "prompt spell checking and fix suggestions (K on a plain word)")`
- Retitle the check from "Optional artifact tools" to "Optional feature tools" (id stays `tools.optional`; the list has
  outgrown "artifact"). Update any test that asserts the old title.
- Small deep-check polish: when the `aspell` binary is present, verify an English dictionary is actually installed
  (bounded `aspell dump dicts` call, ~2 s timeout, reusing/paralleling how `checks_deep_providers.py` shells out) and
  degrade the row to missing-with-hint ("aspell found but no English dictionary — install aspell-en") when it is not.
  Keep this inside the existing `tools.optional` runner so there is no new check id.

### Installing the tools on this machine (gated)

This machine is Debian. After the code changes compile and unit tests pass, and **before** any live end-to-end
verification, use the `/sase_gate` skill to propose the install command for user approval:

```bash
sudo apt update && sudo apt install -y dict aspell aspell-en
```

Explain in the gate body what each package enables (dict → definitions panel; aspell + aspell-en → spellcheck panel). Do
not run the install without the gate approval, and do not let any test _require_ the tools (see Testing). If the gate is
denied, the feature still ships — the degraded-toast paths and doctor rows are the contract.

### Documentation

- `INSTALL.md` "Recommended / optional" table: add `dict` and `aspell` rows (`aspell` row notes the `aspell-en`
  dictionary package on Debian; Homebrew's aspell bundles English).
- `docs/ace.md`: update the prose sentence (~line 2941) and the NORMAL-mode keymap table `K` row (~line 3123) to cover
  the plain-word fallback, and add a short "Word definitions & spellcheck" subsection near the preview docs describing
  both panels, the required optional tools, and the doctor/deep hint.
- Help modal `binding_common.py`: change `("K", "Preview xprompt/skill/file")` to
  `("K", "Preview xprompt/skill/file/word")` (31 chars — fits the 32-char budget).
- No keybinding-footer change (`K` is a prompt NORMAL-mode key, not a conditional footer binding) and no
  `default_config.yml` change (`K` is hard-coded in the vim dispatcher, not keymap-configured).

## Implementation Steps

1. **Core adapters** — add `src/sase/core/word_lookup.py` (dataclasses, `extract_lookup_word`, `check_spelling`,
   `look_up_definitions`, injectable runners, timeouts).
2. **Unit tests for the adapters** — `tests/test_word_lookup.py` (flat `tests/`, sibling of `test_clipboard_utils.py`):
   - Extraction: cursor mid-word / at first & last char / on interior apostrophe or hyphen; leading/trailing `'`/`-`
     excluded; digits/underscores → `None`; whitespace & punctuation → `None`; unicode letters accepted; 64-char cap.
   - `check_spelling` parsing: `*`, `+`, `&` (with suggestion list), `#`, missing binary → `unavailable`, timeout and
     nonzero exit → `error` (all via fake runners).
   - `look_up_definitions` parsing: multi-section output split on `From … […]:` headers with preamble tolerated,
     `no_match` exit code, timeout/connection failure → `error`, missing binary → `unavailable`.
3. **Modals + styles** — `word_definition_modal.py`, `spellcheck_panel_modal.py`, and their `styles.tcss` blocks.
4. **TUI mixin + wiring** — `_prompt_word_lookup.py`, mix into `PromptTextArea`, the small `_prompt_preview.py` fallback
   edit (including the reworded no-target toast), suggestion-apply dismiss callback with the span guard.
5. **Doctor** — `_OPTIONAL_TOOLS` rows, retitle, aspell-en dictionary sub-check, update `tests/doctor` assertions
   (extend the existing `checks_tools` doctor test module).
6. **TUI behavior tests** — new `tests/ace/tui/widgets/test_prompt_normal_mode_word_lookup.py` using `PromptPage`
   (mirror `test_prompt_normal_mode_preview.py`; monkeypatch the two adapter functions at the `_prompt_word_lookup`
   import site):
   - `K` on a correctly spelled word pushes `WordDefinitionModal`.
   - `K` on a misspelled word pushes `SpellcheckPanelModal`; pressing `3` replaces the word in the prompt text with the
     third suggestion, closes the modal, and leaves the cursor at the word start; `Esc` leaves text untouched.
   - `K` with aspell unavailable falls through to definitions; `K` with both tools unavailable toasts and pushes no
     modal; `K` on punctuation/identifier shows the reworded warning toast.
   - Regression: `K` on `#xprompt` / file tokens still opens `PreviewPanelModal` (existing tests keep passing).
7. **PNG visual snapshots** — `tests/ace/tui/visual/test_ace_png_snapshots_word_lookup.py` modeled on
   `test_ace_png_snapshots_preview_panel.py`: one snapshot of `WordDefinitionModal` with a fixed two-section payload,
   one of `SpellcheckPanelModal` with a fixed suggestion list; generate goldens with `--sase-update-visual-snapshots`.
8. **Docs & help** — INSTALL.md, `docs/ace.md`, `binding_common.py` (per the Documentation section).
9. **Install gate + live verification** — submit the `/sase_gate` install proposal; after approval, verify end-to-end in
   a real `sase ace` session (`K` on a real word → definitions; on a typo → apply a fix; `sase doctor -D` shows the new
   rows as available) and confirm `aspell dump dicts | grep en` behavior matches the doctor sub-check.
10. **Quality gates** — `just install`, then `just check` (lint + mypy + tests, including the visual suite) before
    finishing, per repo policy.

## Testing Notes

- No test may require `dict`/`aspell` on the machine: unit tests inject fake runners; TUI tests monkeypatch the adapter
  functions. If any optional live integration test is added, guard it with
  `pytest.mark.skipif(shutil.which(...) is None, ...)` — but the mocked coverage above is the contract.
- Keep every new subprocess call out of Textual's event loop and off the pump (workers + `asyncio.to_thread` only), and
  re-check staleness after each await, per the TUI perf memory.

## Non-Goals

- No spell checking of anything other than the single word under the cursor on explicit `K` (no as-you-type squiggles,
  no whole-prompt checking).
- No personal-dictionary management ("add word") in v1 — the panel only offers replacements.
- No configuration knobs (language is hardcoded `en_US`; tool commands are not configurable) — future work if wanted.
- No offline dictionary provisioning (`sdcv`/StarDict data, local `dictd`) — `dict` uses its default dict.org server;
  the timeout + error-toast path covers offline sessions, and the research note documents the offline upgrade path.
- No Rust-core (`sase-core`) changes — this is Python-side presentation and local-tool glue by the boundary litmus test.
