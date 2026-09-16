---
tier: tale
size: medium
title: Project-scoped prompt history from Ctrl+K
goal:
  Replace literal VCS workflow text in Ctrl+K history searches with a visible, editable
  project filter that matches project identity reliably.
proposed_by: bbugyi200.apollo.06
create_time: 2026-09-16 14:06:54
status: wip
---

# Project-scoped prompt history

## Outcome and scope

Opening history from `#gh:sase fix parser` should show the query
`project:sase fix parser`. The project constraint groups equivalent workspace
references; the remaining text searches as it does today. Removing the project prefix
immediately searches across all loaded projects.

Implement this as one medium tale: the Rust domain contract, Python adapter, TUI
integration, documentation, and focused verification form one bounded change for one
coding agent. No implementation changes have been made during planning.

Keep the existing history store, recency order, cancelled toggle, page size, and
explicit load/unload controls. This task does not introduce archive-wide search, new CLI
options, fuzzy ranking, autocomplete, or a general query-language migration. Other
history entry points remain unscoped by default, but accept the new filter when the user
types it. No keymap change or new configuration is needed.

## Existing behavior and integration points

- `src/sase/ace/tui/widgets/_prompt_text_area_actions.py` sends the entire single line
  as `HistoryRequested.initial_filter`. That includes its workspace tag; there is no
  separate project-filter parser today.
- `src/sase/ace/tui/actions/agent_workflow/_prompt_bar_requests.py` passes that string
  into `PromptHistoryModal`. Its callback also owns submit/edit/load and preserving the
  originating prompt pane. Search preparation must not change those selection actions or
  the independent `vcs_prefix` replacement behavior.
- `src/sase/ace/tui/modals/prompt_history_modal.py` filters loaded records with a
  case-insensitive substring test against canonical and humanized prompt text. Its first
  page loads in a worker; `Ctrl+J` loads older records and `Ctrl+K` unloads the last
  page. `_prompt_history_models.py` carries row data.
- `src/sase/history/prompt_catalog.py` and `prompt_store.py` provide paged records.
  Current records contain prompt text, not authoritative project metadata. The legacy
  `workspace` and `branch_or_workspace` fields are not identity sources.
- `src/sase/project_display_names.py` has an immutable `ProjectRefDisplaySnapshot` for
  names, canonical keys, and aliases. `src/sase/xprompt/_parsing_vcs_tags.py` provides
  syntax-aware VCS spans, including `find_vcs_workflow_tag_span`; literal-zone handling
  is already available. Project records and Patch ownership provide better evidence than
  splitting a displayed basename or guessing ownership from a name prefix.
- The modal already has an attractive aligned table and preview, with styles in
  `src/sase/ace/tui/styles.tcss` and a PNG fixture in
  `tests/ace/tui/visual/test_ace_png_snapshots_prompt_history.py`.

## Interaction contract

### Initial query

| Draft in the originating pane                               | Initial history query                          |
| ----------------------------------------------------------- | ---------------------------------------------- |
| `#gh:sase fix parser`                                       | `project:sase fix parser`                      |
| `#git:sase fix parser`                                      | `project:sase fix parser`                      |
| `#gh:gh_sase-org__sase fix parser` with display name `sase` | `project:sase fix parser`                      |
| `#gh:sase-org/sase fix parser` resolving to that project    | `project:sase fix parser`                      |
| `#gh:sase`                                                  | `project:sase `, with the caret ready for text |
| `%m:opus #gh:sase fix parser`                               | `project:sase %m:opus fix parser`              |
| `fix parser`                                                | `fix parser`                                   |
| Empty input                                                 | Empty query                                    |

Use the first active workspace reference in the originating single-line pane, consistent
with the existing VCS span helper. Recognize supported colon, parenthesized, modifier,
and legacy underscore forms through existing syntax machinery, not a new TUI regex.
Ignore inline/fenced code and disabled xprompt regions. Remove only the recognized
workspace span and its redundant boundary whitespace; preserve remaining directives,
xprompts, punctuation, and text order. Never mutate the draft itself. Do not derive a
default scope from another pane, the current UI tab, the process working directory, or
the synthetic home context.

Prefer the configured project name in the generated token when it resolves
unambiguously; otherwise use the canonical project key. A Patch ref scopes to its owning
project when ownership is known from local records. An unresolved or ambiguous
reference, including an unavailable agent ref, must not trigger a network/provider
resolution or guess at a project. Remove its recognized VCS span, retain the text
search, and show the non-error hint
`Project scope unavailable; searching all loaded prompts`. No VCS tag means the existing
unscoped behavior, with no warning.

Keep the current single-logical-line, prompt-mode, and auxiliary-pane guards. Keep
completion cleanup, stack origin capture, focus restoration on Escape, and Enter /
Ctrl+G / Ctrl+I / Ctrl+Y behavior.

### Filter syntax and matching

Use a deliberately small grammar: one optional **leading** `project:<value>` qualifier
followed by an optional literal text substring. This preserves existing phrase searches
without introducing token-wise AND, boolean operators, or fuzzy matching. Leading
whitespace before the qualifier is allowed. A `project:` occurrence later in ordinary
text remains literal.

- The field name and project equality are case-insensitive, following existing Rust
  query normalization. A bare value ends at whitespace. Also accept a double-quoted
  value with escaped quote/backslash support. Empty values and unfinished quotes produce
  an inline diagnostic, never an unscoped search.
- Match the complete project identity: `project:sase` excludes `sase-core` and unrelated
  prompts merely mentioning `sase` in prose or code. Resolve canonical keys, configured
  names, and registered aliases to the same key. Ambiguous labels do not match several
  projects silently; report the ambiguity and ask for a canonical key in the inline
  hint.
- Use verified project facts for owner/repo and Patch refs. Never identify a project
  solely by the final path component or by a Patch-name prefix. Include disabled
  projects and `home` in the read-only identity inventory.
- Preserve searchability of deleted or unregistered historical direct project refs: an
  unknown filter value can match the same exact unresolved direct VCS ref. Keep these
  raw-ref identities distinct from registered keys; do not infer owner/repo aliases or
  Patch ownership without evidence.
- For stored multi-prompt history entries, collect the effective active project
  reference for each segment and match if any segment belongs to the requested project.
  Ignore literal-zone mentions. A legacy record with no active VCS reference has no
  inferred project and remains accessible unscoped.
- After the qualifier, trim only the separator whitespace and use the remaining string
  as one case-insensitive substring against either canonical or display text. Preserve
  its internal whitespace and punctuation. Queries without a qualifier retain the
  current substring behavior exactly.
- Provide `\project:sase` as an escape for a literal substring beginning with
  `project:`; remove that one escape before matching. Use the same encoding helper when
  an unscoped draft would be interpreted as query syntax, including an existing escape;
  its encode/parse round trip must recover the exact draft search text. Ctrl+K must not
  reinterpret prompt prose as a filter. Everything after a parsed qualifier is already
  literal, including another `project:` token. Document that this grammar supports one
  leading scope, not repeated field constraints.
- A valid project constraint combines with the text substring and cancelled visibility
  using AND. A malformed constraint shows no selectable results; submit, edit, load, and
  copy must not act on a stale selection. Fixing it immediately restores results.
  Unknown, well-formed values give ordinary empty results, not an exception.

### Presentation

The query text is the only scope control. Add one restrained helper line directly
beneath the input, using the existing bottom-margin space where practical:

```text
[ project:sase fix parser                                      ]
Project: sase · Remove the project filter to search all loaded prompts

History · 7 / 100 loaded · ^j +100 older            Preview
```

Use the existing theme accent for the project name and muted text for guidance. When
unscoped, the line teaches `project:<name> text`; on invalid input it shows the
actionable diagnostic using the existing error color. Render names as plain text, not
markup. Keep the line to one row and ellipsize on narrow terminals; the full editable
query must remain accessible. Preserve the table alignment, preview proportions,
cancelled styling, and footer keyboard hints.

Add a disabled empty-result row and clear the preview. While older pages remain, say
`No matches in loaded prompts` and point to Ctrl+J and removing the project filter;
after exhaustion, say `No matching prompts`. Counts continue to describe loaded records,
not an invented global match count. Keep the loading placeholder distinct from empty
results. Do not auto-drain all history to find a match.

## Implementation sequence

1. Open `sase-core` with `/sase_repo` and follow its instructions. Shared query parsing,
   identity matching, and seed construction belong in `crates/sase_core`, exposed
   through `crates/sase_core_py`; add a small focused prompt-history module and typed
   wire records. A dedicated prefix parser is intentional: the existing `query/flat.rs`
   grammar changes whitespace into conjunction and would alter legacy substring
   searches. Reuse suitable core normalization and literal helpers without importing its
   incompatible grammar.

   Define pure operations to compile the prefix query, construct a seed from recognized
   VCS span facts, project record/ref facts into stable identities, and batch-match
   prepared history rows. Return structured diagnostics and scope labels with matches.
   Use immutable inputs and no disk/network access in these operations. Keep any new
   project-equivalence rules in Rust.

2. Add a thin Python facade/wire adapter under `src/sase/core` and a history preparation
   adapter under `src/sase/history`. Reuse the existing xprompt parsers to supply
   recognized span/ref/segment facts, and existing read-only project/Patch readers to
   supply identity facts to Rust. Do not duplicate either the new matcher or
   alias-resolution policy in Python. Preserve disabled/unregistered history and
   canonical stored prompt text. No storage migration, prompt rewrites, side-effectful
   `resolve_ref`, or network access.

   Prepare one identity snapshot per modal opening, refreshed on reopening. Gather
   needed Patch facts in a batch, including archived ownership where available, instead
   of reading files per row. Prepare each loaded row's canonical/display text and
   project identities once, off the event loop. Humanized text and project identities
   should use the same snapshot. Release prepared rows when their page is unloaded; no
   process-wide unbounded cache.

3. Give `HistoryRequested` and the modal an explicit optional prompt-seed input,
   distinct from an already-authored `initial_filter`. Ctrl+K supplies the draft through
   that path; other callers keep their literal query contract. Carry it through
   `_prompt_bar_requests.py` without changing result callbacks. Callers supply either a
   seed or an authored query, never both; validate this invariant rather than silently
   prepending a second project constraint.

   Open and focus the modal immediately. Resolve the seed and prepare the first page in
   the existing worker path. Do not flash the raw VCS token in the input as a temporary
   query. Show the loading state until preparation finishes. Apply the seed once only if
   the user has not edited the filter meanwhile; track input revision so slow work
   cannot overwrite typing or a deliberate clear. Escape must work during preparation as
   well as after it. Place the cursor at the end without selecting the prefix. Discard
   late results after dismissal. Programmatic seeding must not count as a user edit or
   trigger another history request.

4. Replace the modal's substring list comprehension with the core-backed
   compiled-query/batch-match adapter, retaining the existing page lifecycle. Parse once
   per input change; match cached row facts, with no project loading, metadata
   extraction, filesystem work, or provider calls per keypress. Page arrivals must use
   the latest query and cancelled setting, not values captured before an await. Query
   errors, cancellation toggles, load/unload, and resize must keep preview and selection
   consistent.

5. Implement the helper line and empty states in the existing modal/style files; split
   presentation helpers if size limits warrant it. Update the prompt-input key table and
   Prompt History / Filtering documentation in `docs/ace.md`, including the leading-only
   grammar, escaping, examples, identity behavior, and loaded-page scope.

6. Verify the complete change and the actual Rust binding used by Python. Use the
   repository's supported local binding build/install workflow for linked development.
   Do not add a Python fallback or manually change Rust release versions. The additive
   binding must be released through the normal core release workflow before the Python
   consumer lands; raise the Python minimum dependency to the actual published version
   containing it when needed, and check against that supported version. Include both
   repositories in host-owned completion. The old automatic raw-VCS query seed is
   intentionally removed as requested; this complete tale requires no migration flag or
   opt-out.

## Acceptance and verification

Add focused behavioral coverage at the core/binding boundary and extend the existing
trigger, request-handler, modal, and visual suites:

- Core contract vectors cover the example table, supported VCS spellings, directives
  before a tag, protected text, EOF tags, and exact preservation of the residual text.
  Test aliases, canonical/display names, disabled projects, registered owner/repo
  references, Patch ownership, collisions, unknown refs, absent project metadata, and
  multi-segment records. Test the leading prefix, quoted/escaped values, incomplete
  input, literal escape, mixed case, Unicode text, and the unchanged plain substring
  semantics. Assert no false match for `sase-core`, prose-only references, or shared
  repository basenames.
- Binding tests round-trip the new wires and exercise real Rust parsing and matching;
  Python tests must not all mock the core behavior.
- Extend `tests/ace/tui/widgets/test_prompt_history_trigger.py` and
  `tests/ace/tui/test_prompt_bar_history_requests.py` for Ctrl+K seed routing, retained
  mode/line guards, Escape draft preservation, exact stack origin, and unchanged
  submit/edit/load behavior. Keep leader-open history unscoped.
- Extend `tests/ace/tui/modals/test_prompt_history_modal.py` for project-only and
  combined filters, alias-equivalent rows, cancelled toggles, malformed queries
  disabling actions, clearing scope, page-two matches, unload/reload, unknown scope,
  preview clearing, and loaded-versus-exhausted empty states. Use blocked-worker tests
  to prove focus/Escape remain responsive, typing wins over late seed preparation, and
  late pages use the latest query. Assert that filter edits reuse prepared facts without
  fresh disk reads.
- Extend the existing PNG suite with scoped results and empty/error states. Inspect
  actual images at the normal 120x40 size and a compact 80x24 size, including light/dark
  theme coverage for the new helper text. Ensure the query and table stay readable and
  no new helper line wraps over the panels. Update only intentional golden changes after
  viewing the diffs.
- Read `lint_and_test.md` through `/sase_memory_read` before finishing implementation.
  Run focused tests, the targeted visual suite, and `just check` in the SASE repo. Run
  the core repo's `just check` (or its documented check script), including PyO3 binding
  tests; `cargo test -p sase_core` alone is insufficient. Follow the required
  `just check-full` landing/monitor workflow if the combined change triggers it. Report
  any unavailable release or verification prerequisite explicitly rather than claiming
  it passed.

Success means Ctrl+K searches the intended project across equivalent VCS refs, the user
can see and remove the scope, history remains responsive and honest about pagination,
and selecting or cancelling history preserves existing prompt editing behavior.
