---
tier: epic
status: done
title: Star-triggered model alias completion
goal:
  Make choosing any configured model alias fast, clear, and reliable by expanding an
  accepted prompt-widget star completion into a canonical model directive.
phases:
  - id: core_alias_shortcut
    title: Define the shared model alias shortcut contract
    size: small
    depends_on: []
    description:
      "core_alias_shortcut: implement literal-aware trigger detection, replacement
      planning, public Rust exports, PyO3 bindings, and contract tests."
  - id: prompt_alias_menu
    title: Integrate and polish the prompt alias menu
    size: medium
    depends_on:
      - core_alias_shortcut
    description:
      "prompt_alias_menu: consume the landed core API, integrate cached model aliases
      with prompt completion, and complete documentation, behavioral tests, and visual
      review."
proposed_by: bbugyi200.athena.087
bead_id: sase-yf
create_time: 2026-09-09 19:52:47
---

- **PROMPT:**
  [prompts/202609/star_model_alias_completion.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/star_model_alias_completion.md)
- **BEAD:**
  [sase-yf](https://github.com/sase-org/sase--beads/blob/main/pages/sase-yf/README.md)

# Star-triggered model alias completion

## Outcome and scope

Typing `*` at the start of a logical prompt line or immediately after an ASCII space
opens an alias-only completion menu. Type an alias prefix and press Enter to replace the
entire `*query` token with `%m:@<selected_alias>`. The first result is already selected.
For example, `Explain this *la` becomes `Explain this %m:@large ` after Enter.
Acceptance edits the prompt; a subsequent Enter uses the normal submission flow.

This feature belongs to the prompt input widget, including editable prompt panes in a
stack. It reuses the existing model catalog and completion UI. It does not introduce
launch-time interpretation of unaccepted stars, alter model routing, or add an
editor/LSP frontend feature. The headless contract lives in Rust so another frontend can
reuse identical behavior later.

Use an epic because SASE's CI builds a pinned core revision. The core contract and
bindings must land first, allowing the second worker to pin a real revision that
contains the new symbols. This is a two-phase dependency chain, not parallel work.
Planning this epic is xlarge work under the sizing guidance; implementation phases are
small and medium because the design below bounds their scope.

## Interaction contract

### Trigger and filtering

- In Insert mode with an empty selection, automatically open for a freshly typed bare
  `*` or `*query` at prompt offset zero, after a logical newline, or immediately after a
  literal ASCII space. Indentation ending in a space works. Soft wrapping does not
  create a new logical line. A tab immediately before the star is not a trigger; this
  follows the requested space-or-line-start rule.
- Query with the alias name without typing `@`. Pass `@` plus the query through the
  existing Rust-backed alias-only model filter, retaining its case-insensitive prefix
  matching, canonical spelling, and stable catalog order. Do not add fuzzy matching, MRU
  ordering, or alias-specific routing behavior.
- Include all effective built-in, user/project, and enabled-plugin model aliases exposed
  by the canonical catalog, once each. Existing configuration precedence decides
  collisions. Provider shorthand matches and concrete model/provider rows are not `@`
  aliases and do not belong in this menu. Preserve existing handling of malformed
  entries and degraded target metadata.
- The first result is selected immediately, including a bare-star menu. A single match
  remains a suggestion until explicitly accepted. Typing filters in place; preserve a
  selected alias while it still matches, otherwise select the first result. Backspacing
  an active query to the bare star shows all aliases again.
- No matches dismisses the panel and leaves the literal text untouched. An unknown
  `*query` does not become a directive when the prompt is submitted.
- Honor `ace.prompt_completion.auto_directive_menu` for automatic opening. Manual Ctrl+T
  can open the shortcut even with that setting disabled. Keep the existing independent
  soft-completion setting and Tab/snippet behavior. Manual invocation opens the menu,
  including for one result, so Enter consistently performs the expansion.

### Acceptance and dismissal

- Enter or Ctrl+L accepts; Down/Ctrl+N and Up/Ctrl+P navigate using existing menu
  behavior. Enter on the initial bare-star selection accepts without an extra navigation
  keystroke. Consume the event so it cannot submit or insert a newline in the same
  keypress.
- Replace the complete live `*query` token, including a remaining alias suffix to the
  right of the caret when editing within a token. Preserve text before and after the
  token, indentation, newlines, other directives, and other panes. Keep replacement
  token-local; do not relocate the model directive or delete an earlier `%m` elsewhere
  in the prompt. Existing directive precedence applies.
- At logical end of line, append one ASCII space so the user can keep typing
  immediately. If an ASCII space already follows the token, reuse it and position the
  caret just after that first space without changing the whitespace run. Before a tab,
  place the caret just after the inserted directive and preserve the tab. Before a
  newline, append the convenience space on the current line and keep the caret there.
  Never consume the newline.
- Apply expansion and any convenience space as one undoable edit, with a history
  boundary that lets undo restore the original `*query` without removing earlier prompt
  prose. Redo restores the expansion and caret position. Acceptance closes the menu and
  does not automatically reopen `%m` or `@` completion.
- Revalidate the trigger and selected alias against the current prompt/cursor and
  current candidate snapshot at acceptance. If they no longer match, dismiss or refresh
  without editing; consume that acceptance key so a stale result cannot accidentally
  submit the prompt.
- Escape leaves the star text unchanged and follows the established Insert-to- Normal
  transition. Ctrl+C retains its existing prompt cancellation behavior. Moving out of
  the token, losing focus, switching panes, changing mode, loading prompt text, or
  deleting the star clears its completion state. Returning the caret alone does not
  reopen a dismissed menu; Ctrl+T or a fresh qualifying edit can reopen it. Undo must
  not reopen it as a side effect.

### Ordinary writing and protected contexts

`a*b`, `path/*`, and `\*` never trigger. A second star in `**bold**` dismisses any
transient bare-star menu. Typing the space in `* item` or `a * b` dismisses it without
replacing the star. A completed `*emphasis*` token is literal. An unfinished emphasis
word can temporarily match an alias; only explicit acceptance changes it.

Use the Rust core's existing prompt literal-zone semantics for fenced code, inline code,
and disabled prompt regions. Also exclude YAML frontmatter and read-only or structured
frontmatter panes. Test unclosed fences and cursor-at-end boundaries. Do not implement a
second Markdown parser. Respect existing structured completion ownership inside
directive/xprompt arguments, placeholders, Jinja, and path/reference contexts; a space
inside an owned argument does not turn its asterisk into a top-level model shortcut.

## Visual design

Use the shared completion panel in its established location above the prompt, with the
title `model aliases`. Reuse `ModelCompletionMetadata`, existing alias kind badges,
provider/model/effort styling, availability/override state, and the selection pointer.
Keep alias identity visually dominant and render available descriptions as secondary
detail.

The selected row's contextual subtitle should begin with the actual expansion:
`Enter → %m:@large · <alias description>`. At narrow widths, drop/truncate the
description first, then truncate an exceptionally long alias with an ellipsis. Do not
show configuration-edit instructions when a description is absent; the expansion preview
is sufficient. Labels are literal Rich Text, not interpreted markup. Add prefix
highlighting through the existing match-highlighting helper where it fits the model-row
renderer, without changing other model menus.

While this menu is open, the prompt bar's action hint must say `Enter accept alias`
rather than `Enter send`, with the existing Escape/Normal hint. Restore the previous
mode hint on every close path. Loading or unavailable states must not claim a selectable
result. Retain the current row cap, scrolling, height budgeting, and cursor readout. A
narrow terminal must preserve the selected alias and acceptance affordance before
optional target/status columns.

## Existing implementation to reuse

- `src/sase/xprompt/model_completion.py` owns the cached catalog bridge and calls
  `filter_model_completion_entries` in Rust. Prefix `@` already restricts results to
  alias kinds. The config merge in `src/sase/config/core.py` includes plugin defaults
  before user/project overrides. Alias target enrichment already uses cached views and
  read-only override/routing snapshots.
- `src/sase/ace/tui/widgets/_directive_completion_models.py` maps catalog entries into
  model row metadata. Share this conversion rather than duplicating fields.
- `_file_completion_open.py`, `_file_completion_context.py`,
  `_file_completion_refresh.py`, `_file_completion_tab.py`, and
  `_file_completion_accept.py` implement the common menu lifecycle.
  `_prompt_text_area_key_handling.py` owns Enter and automatic opening.
- `_file_completion_base.py::_warm_model_completion_catalog` already schedules a thread
  worker at prompt mount; `_prompt_input_bar_stack_rendering.py` also warms newly
  mounted panes. Warming is currently a one-shot and the catalog builder can rebuild
  synchronously on a cache miss: reuse and tighten this path so the shortcut cannot race
  cold startup into filesystem work on a keypress.
- `_prompt_input_bar_completion_panel_kinds.py`,
  `_prompt_input_bar_completion_panel_content.py`,
  `_prompt_input_bar_completion_panel_labels.py`,
  `_prompt_input_bar_completion_rows_directives.py`, and
  `_prompt_input_bar_completion_panel.py` handle model rows and mode subtitles.
- Rust's `crates/sase_core/src/model_completion.rs` supplies alias filtering;
  `prompt_literal_zone_ranges`, `editor` document/range utilities, and existing PyO3
  completion bindings supply the relevant headless contracts.

All Python paths above are relative to the SASE repository root. Rust paths are relative
to the `sase-core` repository root. Each worker must use `/sase_repo` and
`sase repo open sase-core -r "<specific reason>"` before accessing that checkout, then
use only the returned path and follow its AGENTS.md.

## Core alias shortcut

Implement a focused core editor module for shortcut context detection and edit planning.
Expose a typed context containing query, complete replacement range, and caret
information. Expose a selection-to-edit operation returning the range, canonical
`%m:@alias` replacement including the specified spacer behavior, and resulting caret.
Validate current context and selected canonical alias before returning an edit; reject
invalid ranges and non-alias selections. Reuse existing model filtering and literal-zone
helpers. Keep this pure and free of I/O or provider resolution.

Export the contract through `crates/sase_core/src/lib.rs` and thin PyO3 bindings in
`crates/sase_core_py`. Use established document position conversions; state the wire's
position units explicitly and test Unicode before and within the surrounding prompt.
Python/Textual character columns must never be mistaken for Rust byte offsets or editor
UTF-16 columns. Add typed wire conversion as needed, without refactoring unrelated
completion providers or changing LSP trigger lists.

Core tests cover all trigger boundaries and exclusions above, full-token replacement
with a mid-token caret, multiple lines, whitespace preservation, literal regions,
invalid/stale ranges, non-alias inputs, and Unicode/CRLF input. PyO3 tests must exercise
the actual exported functions and their plain dict/list shapes, including None/error
outcomes and cursor conversion.

Run the core repository's `just check` or `./scripts/check.sh`, which includes workspace
clippy and tests for the binding crate. Use `/sase_monitor` if long. Do not substitute
`cargo test -p sase_core`. Leave crate versioning to release-plz. Finish through
host-owned finalization so phase two can use the landed revision.

## Prompt alias menu

First open the core checkout and confirm the phase-one symbols exist. Update
`sase-core-revision.txt` using the established ratchet workflow only once that revision
is durably available, and verify the selected revision contains these symbols. Handle
the published `sase-core-rs` dependency floor/lock through the existing repository
tooling when its release is available; do not invent a SHA or version, silently fall
back to Python, or ship callers ahead of their binding. Build/install the opened core
through `just install` before Python verification.

Add a thin Python adapter and a distinct `model_alias` completion kind. This allows
whole-star-token acceptance while sharing model metadata/rendering with ordinary `%m:@`
completion. Wire it through automatic opening, manual Ctrl+T, cursor refresh, selection
preservation, acceptance, teardown, and structured completion precedence. Ensure
classification and row-renderer dispatch both recognize the new kind; setting the title
alone is insufficient.

Share the existing model-catalog warm path across prompt panes. Provide a cache-only
read for this keystroke path and schedule coalesced background work on a cold or
invalidated catalog. Never load config/plugin files, probe providers, advance alias
selectors, or acquire durable routing state while typing/rendering. Use current config
invalidation and cheap read-only override/routing peeks; rebuild static catalog metadata
off the event loop and serial message pump.

When cold, show a non-selectable `Loading model aliases…` row; Enter/Ctrl+L is consumed
without sending or inserting. Worker completion rechecks prompt text, cursor, selected
pane, mode, and request generation before publishing matches. Completion of a stale
worker must not revive a dismissed menu or write into a different pane. On load failure,
show a quiet non-selectable unavailable state, keep text editable, and permit an
explicit Ctrl+T retry. No repeated error toasts. Warm filtering and navigation stay in
memory and update immediately.

Implement the visual treatment above and update the prompt completion section of
`docs/ace.md`, the existing `auto_directive_menu` documentation in
`docs/configuration.md`, and its comment in `src/sase/default_config.yml`. Add `*alias`
help to `src/sase/ace/tui/modals/help_modal/binding_common.py`, respecting the help
renderer's width and description limits. Document examples, boundaries, acceptance
versus send, Escape behavior, and the automatic-menu setting. Correct the adjacent stale
claim that bare `%` stays quiet while editing that paragraph. No new keybinding or
configuration switch is needed.

### Behavioral and visual verification

Use the actual prompt widget and real Rust binding with deterministic catalog fixtures.
Add meaningful cases alongside the existing directive completion tests:

1. Bare star at start, after a space, and on a later line; first-row Enter acceptance;
   unique partial acceptance; exact canonical result and cursor.
2. Case-insensitive filtering, keyboard navigation, selection retention,
   backspace-to-star, unmatched text, and explicit Ctrl+T with auto disabled.
3. Full-token replacement mid-token, whitespace/newline preservation, atomic undo/redo,
   and confirmation that the acceptance key emitted no submission.
4. Markdown/escape/code/frontmatter/argument exclusions and preservation of other
   completion providers, Tab/snippets, list editing, and normal mode behavior.
5. A real merged-config fixture with built-in, plugin, and user aliases, a user override
   of a plugin alias, and degraded metadata. Assert exactly one effective alias per
   name, no concrete models/provider rows, and normal `%m:@` resolution after expansion.
   Clear config caches as required by existing fixtures.
6. Slow cold load while typing, Enter during loading, no matches, refresh after config
   invalidation, failure/retry, Escape/unmount/pane-switch races, and prompt
   restoration. Use a controllable worker barrier rather than timing sleeps to prove
   another key event is processed before catalog load completes. Assert repeated warm
   keystrokes do not call the static catalog builder or resolver.

Reuse `tests/ace/tui/widgets/test_model_completion_rows.py`,
`tests/ace/tui/test_model_completion_panel_titles.py`, and the existing PNG model
completion harness. Add deterministic snapshots for a full alias menu, a filtered
selection with expansion preview, a narrow terminal with long aliases/descriptions, and
an active stacked pane. Exercise light and dark themes where contrast differs. Inspect
actual rendered PNGs and diff artifacts before accepting intentional goldens;
metadata-only assertions cannot establish layout quality. Preserve the existing ordinary
model completion snapshots unless an intentional shared fix requires a reviewed update.

Run focused new/related tests, `just check`, and the relevant dedicated PNG visual suite
(`just test-visual`; goldens in `tests/ace/tui/visual/snapshots/png/`). Read
`lint_and_test.md` via `/sase_memory_read` before finishing. Before landing the combined
epic, run `just check-full` only through `/sase_monitor` with the TESTING/TESTED status
pair. Include the new core revision/binding compatibility checks in landing
verification. Host-owned finalizers own commits and landing.

## Completion criteria

The feature is complete when a user can type `*`, narrow an alias if desired, and press
Enter to obtain the exact canonical directive without submitting the prompt; all
effective alias sources participate with their existing precedence; literal writing,
undo, and other completion flows remain reliable; cold data cannot block typing or
publish stale results; the preview and action hints match the key's behavior at wide and
narrow sizes; and both repositories' required checks and reviewed visual snapshots pass
against the pinned compatible core.
