---
tier: epic
title: Placeholder completion in the prompt input and editors (LSP)
goal: 'Typing inside <> brackets in the ACE prompt input or in any LSP-attached editor
  (e.g. Neovim via sase-nvim) offers completion of placeholder texts already used
  elsewhere in the current prompt, including immediately after snippet expansions
  like cbi that park the cursor inside brackets. Placeholders are also highlighted
  so reused terms read consistently and beautifully.

  '
phases:
- id: core
  title: Placeholder engine and LSP completion source in sase-core
  depends_on: []
- id: tui
  title: ACE prompt input placeholder completion, snippet triggering, and highlighting
  depends_on:
  - core
- id: nvim
  title: sase-nvim placeholder completion smoke coverage and docs
  depends_on:
  - core
create_time: 2026-07-16 08:49:16
status: done
bead_id: sase-6b
---

# Plan: Placeholder completion for `<placeholder text>` in prompts

## Product context

Prompt authors use **placeholders** — strings of the form `<placeholder text>` — to name a concept once and reference it
repeatedly ("update `<the plan file>` ... then validate `<the plan file>`"). Reusing the _exact same_ placeholder text
matters: consistent placeholders read unambiguously to the model. Today nothing helps with that reuse — the author must
retype the text and hope it matches.

The user's `cbi` snippet (`ace.snippets` entry `cbi: "`<$1>`"`, mirrored as a LuaSnip snippet in their Neovim config)
expands to `` `<>` `` with the cursor parked between the brackets — the exact moment a completion menu of previously
used placeholders should appear.

This epic adds **placeholder completion** to both prompt-editing surfaces:

1. The ACE TUI prompt input widget (`PromptTextArea` / `PromptInputBar`).
2. Any editor attached to the sase xprompt LSP (`sase lsp` → the Rust `sase-xprompt-lsp` server), which sase-nvim users
   already get for `sase_ace_prompt_*.md` / `sase_prompt_*.md` buffers, xprompt files, gitcommit, and
   `sase`/`sase_prompt` filetypes.

## Feature semantics (single source of truth — one Rust implementation)

These rules are the contract; both surfaces must agree exactly, so they are implemented once in `sase_core` (see
Architecture).

**Placeholder span.** A placeholder is `<inner>` where `inner`:

- is non-empty, single-line, and contains no `<` or `>`;
- starts and ends with a non-whitespace character (so prose like `a < b and c > d` is _not_ a placeholder — there is a
  space right after `<`);
- is at most a sane length cap (~100 chars) to avoid pathological line-wide matches.

Extraction scans the **entire document/prompt text**, _including_ inline code and fenced code blocks — the primary `cbi`
use case wraps placeholders in backticks, and command templates in fenced blocks legitimately contain placeholders. No
literal-zone skipping.

**Trigger context.** The cursor is "inside brackets" when the nearest `<` before the cursor on the same line has no
intervening `>` or `<` between it and the cursor. The typed **prefix** is the text between that `<` and the cursor. A
closing `>` may or may not exist yet after the cursor; both states are valid trigger contexts.

**Candidates.** All distinct placeholder inner texts extracted from the document, _excluding the span the cursor is
currently inside_, ordered by first appearance in the document, deduplicated exactly, filtered by case-insensitive
prefix match against the typed prefix. **If no candidates remain, completion does not trigger / returns empty** — this
is the "only when placeholders have already been used elsewhere" rule, and it keeps the `<` trigger silent in prompts
that never use placeholders.

**Accept.** Accepting a candidate replaces the inner span (from after `<` up to the closing `>` when present on the same
line, else up to the cursor) with the candidate text. When no closing `>` exists yet, the accepted text appends one. The
cursor lands just after the closing `>` — for `cbi` that leaves only the trailing backtick / remaining tabstops, so
snippet tabstop navigation must keep working after acceptance.

**Explicit non-goals / decisions.**

- No `<` auto-pairing in the TUI (typing `5 < 6` in prose must stay frictionless); snippets and accept-time `>`
  insertion cover bracket closing.
- Placeholders are single-line only.
- In the TUI's multi-pane prompt stack, extraction scans the current pane's full text.
- No helper-bridge / catalog involvement: this completion source is purely document-content-based, like directive and
  file-path completion (unlike the xprompt/snippet/VCS catalogs).
- `sase lsp` launcher (`src/sase/integrations/xprompt_lsp.py`) needs no changes — the trigger character and completion
  source are entirely server-side.

## Architecture

Per the Rust core backend boundary: an editor integration needs this behavior to match the TUI, so the engine lives in
`sase_core` (sase-core repo), the LSP server consumes it directly, and the Python TUI calls it through the
`sase_core_rs` binding — the same "single source of truth" pattern the frontmatter panel already uses (`sase_core_py`
`py_frontmatter_*` ↔ `src/sase/xprompt/frontmatter_schema.py`).

Phase agents working on sase-core or sase-nvim must open those repos with the `/sase_repo` skill and work in the printed
checkout path.

```
sase-core repo                              sase repo (Python)
──────────────                              ──────────────────
crates/sase_core/src/editor/placeholder.rs  src/sase/xprompt/placeholder_completion.py
  extract spans / detect context /            (thin facade via require_rust_binding)
  build candidates                                    │
        │                                             ▼
        ├── crates/sase_xprompt_lsp (LSP)    ACE PromptTextArea completion kind
        │     completion dispatch, `<`        "placeholder" + highlight mixin
        │     trigger char, retrigger cmd
        └── crates/sase_core_py (pyo3)      sase-nvim repo (Lua)
              placeholder bindings           no functional change; smoke tests + docs
```

---

## Phase `core` — Placeholder engine and LSP completion source (sase-core repo)

Everything in this phase happens in the sase-core repo (open via `/sase_repo`).

**Engine (`crates/sase_core/src/editor/`, new `placeholder.rs` module):**

- Span extraction, trigger-context detection, and candidate building implementing the semantics above. Reuse
  `DocumentSnapshot` (`editor/token.rs`) for offset/Position mapping. Note `<`/`>` are token delimiters in
  `is_token_delimiter_at` (`token.rs:354`), so this needs its own line scanner modeled on
  `detect_directive_context_at_position` (`editor/completion.rs:1339`) rather than the generic token extractor.
- The candidate-building result must carry: prefix, inner replacement range, whether a closing `>` must be appended, and
  the ordered candidate texts — enough for both the LSP `TextEdit` and the TUI accept path to behave identically.
- Re-export with the `editor_` alias convention in `crates/sase_core/src/lib.rs` (~lines 175–215).

**Wire + classification:**

- Add `CompletionContextKind::Placeholder` (`editor/wire.rs:33`).
- Add the placeholder detector to `classify_completion_context_with_workflows` (`editor/completion.rs:74`). It should
  run **early** in the detector chain: an unclosed `<` immediately before the cursor is the most explicit context a user
  can create (including inside xprompt args or directive args, where a typed `<` still means "placeholder"), and the
  strict span rules keep it from hijacking other contexts. Add classification-table cases covering interplay with
  directive/xprompt-arg/VCS contexts.

**LSP server (`crates/sase_xprompt_lsp/`):**

- Match arm in `completion_list_for_context` (`src/server.rs:611`) calling the builder against the stored full document
  text (`OpenDocument.text`).
- Item conversion in `src/lsp_convert.rs`: `CompletionItemKind::VARIABLE` (distinct, variable-like icon in editors),
  `filter_text` = inner prefix, `sort_text` preserving document order, `TextEdit` per the accept semantics (append `>`
  when missing).
- Advertise `<` as an additional completion trigger character (`src/server.rs:797-810`) so editor auto-trigger fires on
  typing `<`.
- **Snippet retrigger:** when converting SASE snippet completion items (`sase_snippet_completion_item`), if the
  snippet's first tabstop (`$1`/`$0`) sits immediately inside `<>` in the expanded text, attach
  `command: editor.action.triggerSuggest` — the exact mechanism VCS namespace completion already uses
  (`lsp_convert.rs:309`). This makes accepting the `cbi` snippet in an editor auto-pop the placeholder menu at the
  tabstop.

**Python bindings (`crates/sase_core_py/src/lib.rs`):**

- Expose two functions following the `py_frontmatter_*` pattern (pyfunction → serde JSON → `json_value_to_py`,
  registered in the `sase_core_rs` pymodule):
  - `placeholder_completion(text, line, character)` → context + candidates payload (or null when not in a placeholder
    context / no candidates);
  - `placeholder_spans(text)` → all placeholder spans, for TUI highlighting.
- Mark them, like the frontmatter comment does, as the single source of truth shared by the TUI and the xprompt LSP.

**Tests (three existing layers):**

- Engine/classifier unit tests in `editor/completion.rs` `mod tests` (classification table + span-rule edge cases: prose
  `a < b`, empty `<>`, missing `>`, length cap, fenced/inline code inclusion, dedup and ordering, exclusion of the span
  under the cursor, case-insensitive prefix filter).
- Server tests in `src/server.rs` `mod tests` via `completion_for_text` (no bridge data needed): candidates offered
  inside `<`, empty result when no other placeholder exists, `<` trigger-char advertisement (copy the
  `advertises_plus_...` pattern at `server.rs:3021`), snippet retrigger command presence.
- End-to-end JSON-RPC coverage in `tests/jsonrpc_stdio.rs` (didOpen → completion inside `<>` returns the other
  placeholders).

**Release:** bump the workspace version per the repo's release conventions (release-plz) so a new `sase-core-rs` wheel
and `sase-xprompt-lsp` binary carrying the feature can be published; the `tui` phase depends on that wheel version.

## Phase `tui` — ACE prompt input completion, snippet triggering, highlighting (sase repo)

**Facade:** `src/sase/xprompt/placeholder_completion.py` wrapping the two new `sase_core_rs` bindings via
`require_rust_binding` (`src/sase/core/rust.py`), modeled on `src/sase/xprompt/frontmatter_schema.py`. Bump the
`sase-core-rs` minimum in `pyproject.toml` to the release from the `core` phase (the loader is strict — a stale wheel
raises `AttributeError`, so the pin bump is required, keeping `<0.5.0` unless the core phase shipped a major bump).

**New completion kind `"placeholder"`** threaded through the existing hard-popup stack (this mirrors how every other
kind is wired; no new UI machinery):

- Pure builder module (e.g. `src/sase/ace/tui/widgets/placeholder_completion.py`) mapping the facade payload to
  `CompletionCandidate`s, following the `jinja_completion.py` result shape (prefix + replacement range + candidates).
- Auto-open: branch in `_try_auto_prompt_reference_completion` (`_file_completion_open.py`) so typing `<`, or any edit
  that leaves the cursor in a placeholder context, opens the menu — gated by the existing auto-completion settings and
  by "≥ 1 candidate". Explicit trigger: branch in `_try_file_completion_tab` (Ctrl+T).
- Live narrowing: branch in `_refresh_file_completion_from_cursor` (`_file_completion_refresh.py`).
- Accept: branch in `_file_completion_accept.py` implementing the accept semantics (replace inner span, append `>` when
  absent, cursor after `>`), without clearing active snippet tabstop state so Tab still reaches later tabstops (e.g.
  `cbi`'s end position).
- Rendering: `append_placeholder_completion_row` in `_prompt_input_bar_completion_rows.py` plus the kind branch in
  `_prompt_input_bar_completion_panel.py` — rows show the candidate text with a subtle `<>` glyph badge, a "placeholder"
  border title, styling consistent with the other kinds (tcss additions only if needed).

**Snippet-driven triggering (the `cbi` flow):** after `_try_expand_snippet` and after `_try_advance_tabstop`
(`_snippets.py`), if the cursor now sits in a placeholder context with candidates, open the placeholder menu. This
covers any snippet whose tabstop lands inside `<>`, not just `cbi`.

**Highlighting (the "beautiful" part):** a small `_placeholder_highlight.py` mixin on `PromptTextArea` modeled on
`_alt_syntax_highlight.py`, drawing spans from the facade's `placeholder_spans` so highlighting agrees exactly with
completion extraction. Style the brackets as dim delimiters and the inner text in a distinct accent consistent with the
existing xprompt token styles; defer to code-block overlay precedence where they overlap.

**Performance (per the TUI perf rules):** the Rust scan is a linear in-memory pass, but still memoize spans/candidates
per document edit generation rather than rescanning on every highlight rebuild or keystroke; never block the event loop
(no I/O is involved).

**Tests:**

- Facade + builder unit tests.
- Widget tests (patterns: `test_auto_xprompt_completion.py`, `test_prompt_snippet_expansion.py`,
  `test_prompt_alt_syntax_highlight.py`): auto-open on `<` only when candidates exist; narrowing; accept with and
  without an existing closing `>`; no popup in placeholder-free prompts; a `cbi`-style snippet expansion auto-opens the
  menu and tabstops still work after acceptance; highlight spans.
- PNG visual snapshot of the popup (pattern: `test_ace_png_snapshots_vcs_project_completion.py`) plus one showing
  placeholder highlighting in the input.
- `just check` must pass.

## Phase `nvim` — smoke coverage and docs (sase-nvim repo)

The plugin is a thin client, so **no functional Lua change is expected** for completion itself: the `<C-t>` dispatcher
already asks the LSP first, and native `vim.lsp.completion` / nvim-cmp inherit the server's new `<` trigger character
and the snippet-item retrigger command automatically.

- Smoke test `tests/lsp_placeholder_smoke.lua` modeled on `tests/lsp_vcs_project_smoke.lua` /
  `tests/lsp_snippet_smoke.lua`: completion inside `<` offers the other placeholders with correct `textEdit`s; prefix
  filtering; empty result when the buffer has no other placeholder; the `cbi`-style SASE snippet item carries the
  retrigger command.
- README: add placeholder completion to the `<C-t>` dispatcher table and a short feature section (typing `<`, the
  snippet flow, manual trigger), with a manual smoke check recipe like the existing ones.
- Recommended polish: a `SasePlaceholder` highlight group applied to placeholder spans in the same buffers the LSP
  attaches to, following the `alt_highlight` module pattern (default link to something subtle like `Special`),
  configurable/disabled via `setup()` like `alt_highlight`. Keep it small; drop it if it grows beyond a thin module.

## Risks and mitigations

- **Over-triggering on prose containing `<`** — mitigated by the strict span rules (non-space-adjacent inner text), the
  only-with-candidates gate, and no auto-pairing. The menu literally cannot appear in a prompt that never used a
  placeholder.
- **Classifier precedence regressions** (placeholder detector stealing directive/VCS/ xprompt-arg contexts) — covered by
  classification-table tests in the core phase.
- **Version skew** between the sase repo and `sase_core_rs` — the strict binding loader fails loudly on stale wheels;
  the `tui` phase's pin bump plus the `core`-phase release ordering (enforced by phase dependencies) prevents shipping a
  mismatched pair.
- **TUI responsiveness** — linear scans memoized per edit; no disk or subprocess work in any handler.
