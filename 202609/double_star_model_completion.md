---
tier: epic
title: Double-star explicit model completion
goal:
  Choose concrete models with ** in the prompt widget and external editors, with
  consistent filtering, precise directive edits, responsive interaction, and a polished
  menu that complements the existing * alias shortcut.
phases:
  - id: core_lsp
    title: Shared model shortcut contract and LSP support
    depends_on: []
    size: medium
    description:
      "core_lsp: extend the Rust shortcut contract and Python bindings with explicit
      model selection, implement the complete LSP experience, and verify star-mode
      transitions, filtering, protected contexts, and edit ranges."
  - id: prompt_integration
    title: Prompt integration, visual polish, and editor parity
    depends_on:
      - core_lsp
    size: medium
    description:
      "prompt_integration: pin the landed core, integrate the explicit-model menu into
      ACE's completion lifecycle, verify installed-binary and Neovim parity, review
      visual snapshots, and document both shortcuts."
proposed_by: bbugyi200.athena.0hg
create_time: 2026-09-09 10:57:28
status: wip
---

# Double-star explicit model completion

## Outcome and tier

Typing `**` at the start of a logical prompt line or immediately after an ASCII space
opens an explicit-model completion menu. Typing a model prefix narrows it; accepting a
row replaces the entire `**query` token with `%m:<canonical-model>`. For example, with
`gpt-5.6-sol` in the catalog, accepting it from `Review this **gpt` produces
`Review this %m:gpt-5.6-sol ` without submitting the prompt.

The two shortcuts form a small, consistent vocabulary:

| Input          | Menu                            | Example accepted expansion |
| -------------- | ------------------------------- | -------------------------- |
| `*` / `*la`    | Model aliases                   | `%m:@large `               |
| `**` / `**gpt` | Explicit models                 | `%m:gpt-5.6-sol `          |
| `**codex/gpt`  | Explicit models scoped to Codex | `%m:codex/gpt-5.6-sol `    |

Model names in examples are illustrative catalog entries, not a new hard-coded
inventory. The installed provider/plugin metadata remains authoritative.

Use an **epic** with two sequential medium phases. SASE CI builds a pinned core
revision, and the new Python callers need actual landed bindings. The first phase
delivers the pure shared contract and a complete, tested LSP frontend; the second pins
that revision and completes ACE, documentation, and cross-surface verification. There is
no useful parallel phase dependency. Authoring this epic is xlarge work under the size
guidance; each implementation phase is bounded direct work.

This extends the approved `sase-yf` design and its subsequent LSP implementation. The
original plan was read through
`sase artifact read plan:202609/star_model_alias_completion.md`; the existing
installed-binary parity follow-up is present in commit `1852f091a`. Preserve the
single-star contract while intentionally changing the double-star interaction from
dismissal to model selection.

## Interaction contract

### Trigger, transitions, and literal writing

The shared Rust detector owns classification. The UI supplies editor state such as
Insert mode, focus, empty selection, and whether the pane is editable.

- Recognize exactly two leading stars at offset zero, after a logical newline, or
  immediately after literal ASCII space. Indentation ending in a space qualifies. A
  preceding tab or non-ASCII space does not; a soft visual wrap creates no new trigger
  boundary. CRLF logical lines work.
- The caret must be after both trigger stars. A caret between the two stars does not
  qualify for either shortcut. The query is the text between the second star and the
  caret; the replacement token also includes its suffix after the caret.
- Typing the first `*` opens aliases as today. Typing the second `*` immediately
  switches to explicit models in the same panel, with updated rows, title, preview, and
  action hint. This also works when the alias catalog is empty or loading.
- Backspacing an explicit-model query to bare `**` shows all model rows. Backspacing
  bare `**` to `*` switches to aliases. Reset selection to the first row when the
  shortcut kind changes; retain the selected canonical value while filtering within the
  same kind if it still matches. Otherwise select the first matching row. Do not retain
  a row index from the other kind.
- A third star, or any further star anywhere in the same whitespace-delimited token,
  makes that token literal. Thus `***`, `**bold**`, `**gpt**`, and a caret inside an
  already closed `**gpt**` token do not offer shortcut expansion. Never reinterpret the
  trailing two stars of `***` as a new trigger.
- `a**b`, `path/**`, and escaped `\**` are literal. Typing a space after bare `**`
  closes the shortcut without replacing anything. Existing `* item`, `a * b`, and
  single-star emphasis behavior stays intact.
- An unfinished bold word can temporarily match a model, just as unfinished single-star
  emphasis can match an alias today. Only explicit acceptance rewrites it. Do not
  attempt a new general Markdown parser or silently turn bold prose into a directive at
  submission time.
- Reuse the existing core literal/ownership exclusions: inline and fenced code, unclosed
  fences, disabled xprompt regions, YAML frontmatter, Jinja, placeholders, and
  directive/xprompt-owned arguments. Respect path and reference ownership. Structured or
  read-only frontmatter panes do not open this menu. Reuse and test the shared exclusion
  helpers rather than copying them into a second detector.

### Catalog and search

- Show only canonical catalog entries with `kind == "model"`. Exclude `implicit_alias`,
  `user_alias`, and `provider` rows. A provider row is a drill-down action in the broad
  `%m:` menu, so it is not selectable here.
- Use the same cached/materialized catalog as `%m:` and `*`, including enabled plugin
  models, existing hidden-provider policy, canonical spelling, ordering, and
  deduplication. Do not invent a second inventory, probe provider APIs, or change
  routing or model availability policy.
- Reuse case-insensitive prefix matching on canonical values and the catalog's existing
  short-name hints. A short-name hint can find a concrete model; it is never inserted in
  place of the canonical value. `@` aliases remain exclusive to the alias menu. Do not
  add fuzzy search, MRU sorting, or a new effort picker.
- Support the existing provider-qualified query form, including `**codex/` and
  `**codex/gpt`. **Pass the complete catalog into the shared model filter before
  retaining only model results.** Its provider-scope detection needs the provider rows
  even though this menu never displays them. Preserve existing first-slash semantics for
  identifiers such as `opencode/anthropic/<model>`.
- An unscoped result inserts the catalog's canonical value. A scoped result inserts the
  canonical provider-qualified value produced by that same filter. Preserve the rest of
  the model identifier, including dots, hyphens, underscores, and nested slashes. Do not
  expand aliases through a resolver while completing.
- Bare `**` selects the first model immediately. A single result still requires
  acceptance. No matches closes the ACE panel and leaves the token as literal prose;
  subsequent qualifying typing or manual invocation can produce matches. The LSP returns
  an empty incomplete list for a recognized no-match context.

### Acceptance, caret, and dismissal

In ACE, `Enter` and `Ctrl+L` accept; Down/Up and Ctrl+N/Ctrl+P keep their existing
navigation behavior. Consume the acceptance key so it neither submits nor inserts a
newline. Loading, unavailable, and stale-result acceptance also consumes the key without
sending the prompt. External editors keep their configured completion accept/cancel
keys; the LSP supplies a standard text edit.

Accept exactly one complete `**query` token, including any same-token suffix after a
mid-token caret. Keep preceding prose, indentation, other directives, other panes, and
subsequent text intact. Use the established spacer rules:

| Text after the token          | Edit and resulting caret                                                                           |
| ----------------------------- | -------------------------------------------------------------------------------------------------- |
| Prompt end or LF/CRLF newline | Append one ASCII space; caret remains on this line after it.                                       |
| One or more ASCII spaces      | Consume and reinsert the first space in the edit; preserve the run; caret follows the first space. |
| Tab                           | Append no space; preserve the tab; caret follows the directive.                                    |

The shared edit's final character position must be its returned caret, so applying the
LSP `textEdit` alone produces the same result as ACE. Positions on the wire are UTF-16
editor columns; internal Rust scans use UTF-8 byte offsets; Textual uses Python
character columns. Preserve the existing conversion adapters, including non-BMP
characters and CRLF coverage.

Apply expansion as one undoable edit with the existing history boundary. Undo restores
`**query` without deleting earlier prose or reopening completion; redo restores the
accepted text/caret. Acceptance closes the menu without opening a follow-on `%m:` or `@`
menu. Unaccepted stars have no launch-time semantics. Existing directive precedence and
effort selection apply to the resulting `%m:` normally.

Revalidate the live text, cursor, shortcut kind, and selected value against the current
filtered catalog before editing. Reject an alias passed to the model planner, a provider
row, a stale/no-longer-matching model, and unsafe/malformed catalog values that cannot
be represented as one inline model directive. Reuse the core's directive-value rules
where available. Do not permit a catalog value to inject extra directives or whitespace.

Escape leaves the text unchanged and preserves ACE's Insert-to-Normal transition. Ctrl+C
retains prompt cancellation. Focus loss, mode changes, pane switching, cursor movement
out of the token, text restoration, or deleting the trigger clears completion. Returning
the caret alone does not reopen a dismissed menu; a fresh qualifying edit or Ctrl+T can.
Preserve Tab/snippet handling.

Honor `ace.prompt_completion.auto_directive_menu` for automatic ACE opening. Ctrl+T
opens either shortcut manually even with that setting disabled, including a unique
result. Once manually opened, editing between star kinds keeps the menu in the same
manual session. The setting does not control LSP clients. No permanent keybinding or
configuration setting is added.

## Visual design

Use the existing panel above the prompt, titled **explicit models**, paired with the
existing **model aliases** title. Keep one flat, scrollable list in catalog order. An
illustrative wide rendering is:

```text
┌─ explicit models ─────────────────────────────────────────┐
│ ▸ gpt-5.6-sol                  CODEX         sol           │
│   claude-fable-5               CLAUDE        fable         │
└─ Enter → %m:gpt-5.6-sol · Codex (sol) ─────────────────────┘
  Review this **
  [Enter] accept model  [Esc] normal  [^C] cancel
```

Names and row ordering here illustrate layout, not prescribed provider defaults. Use
existing provider colors and the selection pointer, with canonical model identity as the
dominant text. Give model names available width before optional details. A dedicated
shortcut row needs no repeated `model` badge or alias pool/target columns. Display
provider identity in a compact secondary column and short-name hints in subdued text.
Preserve existing meaningful routing/advisory information from the catalog without
resolving anything during rendering.

Highlight the actual matched prefix in the model name or short-name hint using the
existing match helper; for scoped searches highlight the corresponding provider/name
portions. Do not invent a name match when only a short hint matched. Carry advisory
fields through shared metadata if needed rather than parsing rendered descriptions to
recover them.

The selected-row subtitle begins with the exact expansion: `Enter → %m:gpt-5.6-sol`.
Append useful provider/description/advisory text if it fits. At narrow widths remove
optional detail columns and truncate descriptions before truncating the model name or
expansion. Preserve the selection pointer, visible model identity, and accept
affordance. Use cell-width-aware ellipsis; labels are literal Rich `Text`, never
interpreted markup. A long identifier must remain inspectable through the selected
preview as space permits.

The prompt action hint reads `[Enter] accept model`, with existing Escape and
cancellation hints, and restores the prior mode hint on every close path. Loading and
failure hints must not claim a model is selected. Use theme-aware readable
title/subtitle colors in both light and dark themes; do not rely on the existing
dark-blue border color as the text color. Keep the current height budget, row cap,
scrolling, and cursor readout, including stacked panes.

For LSP clients, supply the canonical model label, the exact expansion in
`labelDetails.detail`, model/provider context, and the existing model documentation with
advisory details. Put the expansion in a standard detail/documentation field as well so
it remains discoverable when a client omits label details. The server does not control
an editor's popup typography, theme, or per-character highlighting.

## Phase core_lsp: shared contract and complete LSP support

All paths in this section are relative to the **sase-core repository**. Open it with
`/sase_repo` and `sase repo open sase-core -r "<specific reason>"`, use only the
returned checkout, and follow its AGENTS.md. Do not locate a sibling checkout by
filesystem search. This phase owns the core and LSP implementation together.

1. Extend `crates/sase_core/src/editor/model_alias_shortcut.rs` with a focused shared
   shortcut implementation, extracting a sibling module if needed to keep file sizes
   reasonable. Factor common token scanning, literal exclusions, UTF-16 conversion, and
   spacer/edit construction. Add an explicit shortcut kind (`alias` versus `model`) to
   the new generic contract. Match the full leading star run so double/triple stars
   cannot fall back to the alias detector.
2. Add a model-only filtering operation around
   `crates/sase_core/src/model_completion.rs` and a selection-to-edit operation that
   validates the canonical filtered value and constructs `%m:<value>`. Keep the catalog
   schema at v1: its existing `kind`, `provider`, `aliases`, `value`, and presentation
   fields already describe the needed data. Do not change `%model:` filtering globally
   to achieve shortcut-only behavior.
3. Export additive APIs through the editor exports, `crates/sase_core/src/lib.rs`, and
   `crates/sase_core_py`: `model_shortcut_context(text, position)`,
   `filter_explicit_model_shortcut_entries(entries, query)`, and
   `model_shortcut_edit(text, position, entries, selected_value)`. The context returns
   `schema_version: 1`, `kind` (`alias` or `model`), `query`, `token`, `caret`,
   `token_range`, and `replacement_range`. The edit returns `schema_version: 1`, `kind`,
   canonical `value`, `replacement`, `edit`, and `caret`; its `replacement` matches
   `edit.new_text`, including any spacer. Invalid contexts/selections return None;
   malformed wire input follows existing binding error conventions. Document the UTF-16
   position units. Preserve the existing `model_alias_shortcut_context`,
   `filter_model_alias_shortcut_entries`, and `model_alias_shortcut_edit` binding names,
   v1 shapes, and alias-only behavior as thin wrappers. Their double-star rejection
   stays valid.
4. Update `crates/sase_xprompt_lsp/src/server.rs` to dispatch a recognized model
   shortcut before generic catalog refresh/file/snippet completion, alongside the
   current alias fast path and after higher-priority structured ownership. Use the
   shared detector, filter, and edit planner. Keep `*` as the single LSP trigger
   character: the second typed star causes another request; do not advertise a
   two-character `**` trigger. Manual invocation and incomplete-list re-requests use the
   same classification.
5. Extend `crates/sase_xprompt_lsp/src/lsp_convert.rs` by sharing the existing model
   candidate projection and shortcut response scaffolding. Model shortcut items carry
   the full-token `textEdit`, `filterText = "**" + context.query` exactly as typed,
   stable zero-padded `sortText`, and first-row `preselect`. Use
   `CompletionList { isIncomplete: true }`, including zero results and missing/malformed
   catalogs. A recognized empty context owns the response; it must not leak unrelated
   completion rows. Preserve alias response fields. Use plain-text edit format without
   commit characters that could turn typing a space, slash, or third star into
   accidental acceptance.
6. Retain the local model-catalog snapshot and refresh semantics. The LSP may read its
   existing materialized local catalog; it must not invoke a provider, helper
   subprocess, network refresh, or unrelated xprompt-catalog load for this shortcut.
   ACE's live overlays and the LSP's launch-time snapshot need not display identical
   transient status; their core results for the same catalog must agree.

Add core and real PyO3 binding tests for the interaction matrix, malformed input,
alias-wrapper compatibility, and actual plain dict/list/None/error shapes. Test provider
scoping with provider rows present, short-name matches that do not prefix the canonical
name, hidden-provider catalog fixtures, nested slash IDs, and catalog-order
preservation. Validate selected scoped values against the filtered derived rows, not
only the unqualified source values.

Extend the LSP service tests and
`crates/sase_xprompt_lsp/tests/jsonrpc_stdio_model_alias_shortcut.rs`, or add a focused
companion stdio suite. Exercise a single document through `*`, `**`, `**query`,
backspace to `**`, and backspace to `*` with real didChange versions and
trigger/incomplete/manual requests. Verify full edits, exact filter text, preselection,
empty-response ownership, and Unicode/CRLF. Assert protected stars never produce a model
directive edit. Avoid oversized additions to `server.rs` or the binding crate's already
large root module by following local module patterns.

Run the core repository's `just check` / `./scripts/check.sh`, including the binding and
LSP crates. `cargo test -p sase_core` alone is insufficient. Use a Python >=3.12
interpreter for PyO3 as required there and `/sase_monitor` for long work. Leave release
versions to release-plz. Finish through host-owned finalization and record the landed
revision containing both the binding and LSP feature for the next phase. The public LSP
behavior is complete and tested in this phase; do not land a partially implemented
active handler.

## Phase prompt_integration: ACE, presentation, and parity

Unless a repository is named, paths below are relative to the **SASE repository**. Read
`lint_and_test.md` and `tui_perf.md` through `/sase_memory_read` before editing.

### Dependency integration

Open `sase-core` through `/sase_repo`, confirm the first phase has landed, and ratchet
`sase-core-revision.txt` to an available revision containing its APIs and server
behavior using the established revision tooling. Check ancestry/symbols, not just
version strings. Follow the existing published-wheel floor/window and lockfile workflow;
do not invent an unreleased version or let a published install use a floor missing
required bindings. Follow current release reconciliation rules for when that published
window changes.

Build the binding with `just install` using the opened core, and explicitly run
`just rust-lsp-install` into the same workspace environment. The LSP executable is
separate from the Python wheel; rebuilding the wheel alone is insufficient. Exercise
`.venv/bin/sase-xprompt-lsp` through the existing parity harness and verify it is the
newly installed binary, not a stale copy elsewhere on PATH.

### Completion lifecycle and performance

Create a thin adapter for the new shared contract alongside
`src/sase/ace/tui/widgets/model_alias_completion.py`. A distinct `model_explicit`
completion kind carries token-wide acceptance while sharing model row metadata. Reuse
the catalog-to-candidate conversion in `_directive_completion_models.py` and the wire
helpers in `src/sase/xprompt/_model_completion_wire.py`; do not duplicate catalog fields
or implement a Python scanner/filter/editor planner.

Wire both star kinds through `_file_completion_context.py`, `_file_completion_open.py`,
`_file_completion_refresh.py`, `_file_completion_tab.py`, `_file_completion_accept.py`,
and the relevant prompt text-area key handlers. Explicitly handle a kind change while a
menu is already active: today's alias refresh clears the menu as soon as its detector
rejects `**`, so merely adding an auto-open branch is insufficient. Preserve manual
versus automatic session ownership, initial selection, within-kind selection retention,
and stale-key consumption. Keep generic `%m:` and other completion providers unchanged.

Share `_file_completion_base.py`'s cache-only model catalog state and
`_file_completion_workers.py`'s coalesced worker. Both currently contain alias-only
branches that must include explicit models. When cold, show non-selectable
`Loading models…`; on failure, show `Models unavailable` and allow Ctrl+T retry. Do not
conflate an empty valid catalog with load failure.

Record and validate the latest request's pane identity, shortcut kind, generation, text,
cursor, and mode before applying a worker result. Recheck focus and active pane. A
worker started for `*` may warm the shared cache for `**`, but may refresh only the
current qualifying request, using its current kind/query. It must never restore alias
rows after the second star, revive a dismissed menu, or write into a pane that lost
focus. Cover the reverse transition and an ABA-style sequence that returns to identical
text after cancellation.

Warm filtering, navigation, and rendering remain in memory. No synchronous config or
plugin reads, provider/routing resolution, blocking shared-store locks, or subprocess
work is introduced on a keystroke or render path. Rebuild invalidated catalogs off the
UI thread and serial message pump using the existing worker. Preserve config
invalidation, coalescing, and failure/retry behavior across panes.

### Rendering and discoverability

Extend `_prompt_input_bar_completion_panel_kinds.py`, panel content/labels,
`_prompt_input_bar_completion_rows_directives.py`, and panel dispatch to implement the
visual contract. A kind must select the right renderer, title, subtitle, and mode hint
even when its rows are placeholders. Extract focused helpers if needed to respect
file-size gates. Preserve ordinary `%m:` and alias menu rendering unless a necessary
shared fix is deliberately reviewed.

Update `docs/ace.md`, `docs/editor.md`, and `docs/xprompt.md` to explain both shortcuts,
provider-qualified and short-hint searches, literal-bold behavior, acceptance versus
send, and existing LSP snapshot/restart semantics. Replace the old blanket claim that
the second star only dismisses completion and clarify that only completed emphasis is
unconditionally excluded. Update the existing `auto_directive_menu` prose in
`docs/configuration.md`, its comment in `src/sase/default_config.yml`, and
`src/sase/ace/tui/modals/help_modal/binding_common.py` with `**model` help. Keep help
text within its rendering limits; no new keymap/config value is needed.

### Behavioral, installed-binary, and visual verification

Use deterministic catalogs and the real binding, extending the patterns in
`tests/ace/tui/widgets/test_model_alias_completion.py`,
`tests/test_xprompt_model_alias_shortcut_parity.py`, and the model completion row,
title, filtering, and payload tests. Add a focused explicit-model companion suite.

The acceptance matrix must cover:

1. Start/space/later-line boundaries, spaces versus tabs/non-ASCII whitespace,
   `*`/`**`/`***` transitions, bare and filtered first-row acceptance, one-result manual
   opening, selection retention/reset, and disabled automatic menus.
2. Model-only results with mixed model/alias/provider fixtures; case-insensitive
   canonical and short-hint queries; known and unknown provider scopes; nested slash
   identifiers; hidden-provider policy; unchanged `%m:` and `*` results.
3. Whole-token mid-caret replacement, all spacer cases, UTF-16/non-BMP and CRLF, atomic
   undo/redo, unchanged surrounding directives, and no submit on accept.
4. Completed emphasis, escapes, code, frontmatter, Jinja, structured arguments, paths,
   placeholders, nonempty selections, and other completion providers. Untouched
   `**unknown` is still ordinary submitted text.
5. Empty/loading/unavailable catalogs, config invalidation, retry, stale selected
   values, focus loss, restoration, cancellation, pane switches, and worker completion
   while changing shortcut kind. Use controlled worker barriers, not timing sleeps, to
   demonstrate typing continues during a cold load. Assert repeated warm typing never
   calls the static builder or resolver.
6. ACE/LSP parity from the exact same fixture: ordered canonical values, core
   classification, whole replacement ranges, final prompt text and caret,
   short-hint/scoped search, empty results, and the unchanged single-star path. Compare
   stable selection/edit data rather than transient live status overlays.

Open `sase-nvim` with `/sase_repo` before reading or editing it. Its existing
`lua/sase/lsp.lua` uses server trigger capabilities and native completion, and
`lua/sase/complete.lua` delegates Ctrl+T to the attached LSP; no new client-side star
parser is expected. Add `tests/lsp_model_shortcut_smoke.lua` following the existing
native-client smoke harness, with an explicit command pointing at the newly installed
LSP. Verify results survive the real Neovim completion conversion for bare `**`, a
canonical prefix, a short-hint prefix, and a provider scope; check ordering and applying
the chosen text edit. Exercise typing the second star and backspacing it in a
native-completion session, plus manual completion. This must catch client filtering
failures that raw JSON-RPC assertions cannot. Add a concise README smoke recipe.
Restrict any necessary Lua fix to evidence from this test; do not implement a duplicate
filtering/editing path in the client. Keep this linked repo in the host-owned final
declaration if changed.

Extend the PNG model completion harness in
`tests/ace/tui/visual/test_ace_png_snapshots_model_completion.py`. Capture full model
menus in dark and light themes, a filtered/short-hint match with the actual expansion
preview, a long/scoped model at 70x24, and a stacked pane. Include an advisory-bearing
model and assert loading/failure hints are truthful. Inspect rendered
actual/expected/diff PNGs before accepting intentional goldens; check title/subtitle
contrast, selected-name legibility, truncation, and panel height. Retain the existing
alias and ordinary-model visual coverage.

Run focused behavior/parity tests, the relevant Neovim smoke tests, `just check`, and
`just test-visual` (targeted iteration followed by the applicable visual suite). Follow
each changed linked repository's required checks. Revalidate actual pinned-core and
installed-binary compatibility rather than passing only against an unpinned local build.

## Landing and completion criteria

The final combined tree must pass the required core checks, SASE `just check`,
cross-surface tests, native-editor smoke coverage, and reviewed PNG goldens. Run
`just check-full` before landing through `/sase_monitor` with the `TESTING`/`TESTED`
pair. Report any verification failures from current evidence; do not treat historical
`sase-yf` failures as automatically still applicable. Host-owned finalizers own commits
and landing across changed repositories.

Done means a user can type `**`, find a concrete model by its canonical name, short
hint, or provider scope, and accept the previewed canonical directive in both surfaces.
The first and second stars switch cleanly, literal writing is preserved until
acceptance, cold data cannot stall typing or publish stale rows, undo is reliable, and
the menu remains readable in wide, narrow, light, dark, and stacked prompt layouts.
Existing aliases and general model completion retain their behavior.
