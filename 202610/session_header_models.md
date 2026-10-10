---
tier: tale
title: Always-visible models in the session header
size: medium
goal:
  Keep distinct recorded agent-turn models visible in the session sticky header,
  including its collapsed state, with graceful overflow and a complete expanded view.
proposed_by: bbugyi200.athena.0zh
create_time: 2026-10-10 13:14:35
status: wip
---

# Always-visible models in the session header

## Outcome and scope

The SESSION identity header above the Agents tab's deck panels should answer “which
models worked on this session?” without expanding anything. Show each distinct recorded
model once in a clearly labeled, provider-colored model strip. Give the strip one row
normally and a second row when needed before omitting models. Expanding the header
reveals the complete list, regardless of the turn-detail display limit.

This is a medium tale: a single implementer can deliver the bounded Rust aggregation,
thin Python adapter, responsive TUI treatment, and tests together. There is no phased
rollout or independent phase graph. The design choices below are intentional defaults;
there are no unresolved reviewer decisions.

Apply this to session-container identity headers. Individual agent-turn headers retain
their existing own-model display, including effort and alias provenance. The header
continues to belong to the selected identity and span all deck panels; switching decks,
splitting, or zooming must not duplicate it. Clan and tribe summaries, named procs,
standalone monitors/gates, model routing, provider execution, and persistence formats
are outside this change.

## Current implementation and constraints

- `src/sase/ace/tui/widgets/prompt_panel/_identity_header_compact.py` builds two
  no-wrap, ellipsized chip rows. `build_agent_compact_lines()` puts a turn-count chip in
  the session branch instead of the single-turn model label. Simply appending a model
  chip would leave it vulnerable to truncation behind a long name.
- `_agent_display_header.py` publishes an `IdentityHeader` containing compact and
  expanded content. `_identity_header.py` also supports raw-prompt attachment and inline
  document rendering. Model facts must survive those transformations.
- `_agent_display_header_metadata_sections.py` builds expanded `Turns:` lanes using
  `_agent_turn_section.py`, but applies `TURN_LANE_LIMIT = 12`. That limited list is
  neither a valid aggregation input nor a sufficient full-list overflow destination.
- `models/agent_session_members.py::concrete_agent_session_turn_rows()` already projects
  the loaded session in causal order, deduplicates overlapping child links, handles
  planner proxies and renamed roots, excludes parallel sessions, and guards cycles.
  Reuse it; do not walk the tree independently or parse the rendered roster.
- `widgets/agent_header_panel.py` knows the actual content width, repaint digest, and
  detail-column height. It currently assumes two compact rows.
  `widgets/agent_header_preview.py::preview_row_budget()` similarly assumes fixed
  border/chip/tab overhead. Both assumptions need to account for model rows.
- `widgets/agent_detail.py` reapplies deck pins after header-height changes. The
  stylesheet gives the header auto height with a 50% maximum. Model fitting must work
  with those bounds, without adding a collapsed-header scrollbar.
- `sase.llm_provider.model_label` supplies the existing provider palette. Its current
  formatter can resolve model aliases; do not accidentally re-resolve historical model
  identities using today's alias/default configuration.

Before implementation, read the applicable AGENTS instructions and the `tui.md`,
`tui_perf.md`, `tui_screenshot.md`, and `lint_and_test.md` reference memories using
`/sase_memory_read`. Open the linked backend with
`sase repo open sase-core -r "Implement recorded session model summaries"` and use only
its printed checkout path. The plan's `sase-core:` paths below are relative to that
repository, not the primary checkout.

## Visual and interaction design

Preserve the two existing identity/context rows. Insert the model strip immediately
below them and above the optional RAW PROMPT card. Use `Models:` even for one model so
the label stays stable as a session grows. Use the existing metadata-label style,
provider colors for model tokens, and dim `·` separators. Do not add another border,
card background, animation, or per-model badge box. Names and versions should be the
visual focus; effort levels and launch aliases remain in the per-turn details.

Illustrative collapsed layout, using fixture model names rather than a hardcoded list:

```text
╭ SESSION ─────────────────────────────────────────╮
│ refactor · 6 turns                               │
│ [existing context/status chips]                  │
│ Models: opus · gpt-5 · sonnet                     │
│ ▎ RAW PROMPT                                     │
│ ▎ Simplify the retry path…                       │
╰───────────────────────────────────── ▾ d more ───╯
```

Model names get the same provider colors as existing turn details. Color is additional
information, not the only means of distinguishing entries: if two distinct providers
have the same model name, qualify both labels as `provider/model`. Determine those
collisions from the full model list, before fitting, so labels do not change identity
when a neighboring entry is omitted. Unknown-provider names retain a neutral style. Do
not prettify away version identifiers or map different recorded spellings to a current
marketing name.

### Fitting and overflow

1. Work in terminal cells, using Rich's cell-aware measurement/cropping and preserving
   spans. Fit against the header content width after borders, padding, and scrollbar
   gutter; never assume terminal width or one deck panel's width.
2. Try all complete model tokens on one row. If necessary, use a second row with a
   hanging indent aligned after `Models:`. Keep complete tokens together where possible.
   Use the second row before hiding any model when height permits. There is no arbitrary
   “first three models” limit.
3. Only after the available one or two rows cannot hold everything, omit whole entries
   and reserve space for a visible `+N more` token. `N` counts omitted distinct known
   models, never turns, effort variants, or unknown metadata. Refit after reserving the
   token so Rich cannot silently clip the overflow indicator. Reserve the separate
   unknown-metadata marker too when it is present.
4. Retain the most recently used known model, then keep as many earliest-first entries
   as fit, presenting the retained entries in their original first-use order. This keeps
   the current/recent model visible without reordering the whole strip on each turn. If
   only one model fits, show the most recent known model plus overflow.
5. Do not shorten ordinary model names just to squeeze more tokens into a line. When one
   token itself cannot fit, middle-ellipsize that token in cells, preserving useful
   beginning/version or ending information. Keep any necessary provider disambiguation.
   Expanded mode always recovers the original full text. At pathological widths where
   even a label and one shortened model cannot fit, use a bounded count-only fallback;
   never wrap indefinitely, crash, or produce an incorrect count.
6. Keep the existing live keymap-aware expand hint (`d` by default). Overflow is
   resolved by that same control, without requiring hover, a new shortcut, or another
   modal.

### Height and expanded mode

Reserve at least one model row for session metadata independently of raw-prompt preview
settings. Allow a second model row when needed and when the collapsed height budget can
accommodate it. Count all borders, identity rows, actual model rows, and the RAW PROMPT
tab before calculating preview-body space. Prefer complete models over optional prompt
preview rows. If there is no space for both the tab and at least one preview-body row,
omit the preview card; do not leave an orphan tab.

Treat the configured share as the target total collapsed height with a floor for the
mandatory metadata footprint (two existing rows, one model row, and borders). Respect
the actual panel maximum and leave usable deck space. On short columns, reduce model
rows from two to one before exceeding that budget; on exceptionally small viewports,
prioritize name and the one model row over optional context chips. Avoid CSS clipping
that hides the model row while keeping less important content. Existing non-session
preview behavior remains as-is.

`collapsed_max_share: 0` still disables only the prompt preview: models remain visible,
using up to two rows within the panel's actual height limit. No new settings or default
keymaps are necessary. Update the existing setting descriptions to explain model-row
overhead and the session preview's zero-row case when metadata uses the available space.

Expanded mode contains an uncapped, wrapping `Models:` summary near the top of the
identity fields, before the possibly capped `Turns:` section. It shows every distinct
provider/model in full, plus explicit missing-metadata information when applicable. Use
the existing expanded-header scrolling behavior for long lists. Preserve the per-turn
effort/alias details and inline document rendering. Toggling back uses the current width
rather than a previously truncated copy.

## Model identity and provenance contract

The input is the full concrete turn projection already loaded for the selected session,
before the expanded 12-turn limit and independent of deck fold state, visible nav rows,
search filtering, or the selected Reply block. Do not derive models from the synthetic
container's inherited model when a concrete planner represents it.

Count concrete LLM agent turns, including failed/stopped/completed turns with recorded
models and the running turn. Ignore monitors, gates, non-agent workflow steps, parallel
helper sessions, and their `next_model` or fallback-route fields: those do not establish
that a model ran in this session. Use the recorded effective model facts already used
for concrete-turn metadata; this feature does not infer additional within-turn retry or
provider-internal model usage from transcripts or configuration.

Define a distinct identity as `(recorded provider, recorded model)`, excluding effort,
service tier, and alias provenance. Normalize surrounding whitespace and a redundant
matching `provider/` prefix. Support the existing explicit provider-prefixed legacy
spelling, e.g. `claude/opus`, without resolving user aliases or default routing.
Preserve custom slash-containing model IDs when a provider is already recorded; remove
only a matching prefix. Unknown provider/name facts must not be guessed from mutable
defaults. If a stored value is only an unresolved `@alias`, treat it as unresolved
metadata rather than fabricating the model that alias resolves to today.

Return unique identities in first-use order and the identity of the latest known model
use, including repeats such as A, B, A. Missing model metadata produces a separate count
of affected agent turns. Show `unknown` after `Models:` when no known model is
available, and a dim `N unknown` token alongside known entries when some turns lack a
recorded model; the expanded summary explains `N turns have no recorded model`. This
count is separate from `+N more`. Empty or incompletely loaded sessions must not invent
`default` as a known model. Use the normal background reconciliation to populate later
facts; no archive scan or new synchronous loader may run to complete the header.

## Implementation work

1. **Add the small shared summary operation in Rust.** Add a focused domain module such
   as `sase-core:crates/sase_core/src/agent_models.rs` with serde request/result wires
   and a pure function over ordered recorded turn facts. Own identity normalization,
   eligibility, stable distinctness, missing-model counts, and latest-known identity
   there. Accept the existing projected turn order; do not migrate the whole session
   tree or agent-scan store for this feature. Expose/register the operation through
   `sase-core:crates/sase_core_py/src/agent_scan/`, following its neighboring
   aggregation binding and adding a round-trip registration test. Use a thin typed
   Python facade in `src/sase/core/` with `require_rust_binding`; there is no Python
   fallback.

2. **Carry immutable summary data with the identity.** Adapt
   `concrete_agent_session_turn_rows()` to the wire facts once during header assembly,
   using the complete projection rather than capped turn lanes. Extend `IdentityHeader`
   with optional summary data defaulting to absent for other node types. Thread it
   through cheap/full header builds, raw-prompt attachment, expanded and inline
   renderables. Add the expanded summary without changing turn navigation/numbering.
   Python owns only data adaptation and presentation, not duplicate semantic rules.

3. **Implement a pure compact fitter and integrate it in `AgentHeaderPanel`.** Keep
   structured model entries intact until render width is known. Return fitted Rich text,
   actual row count, and omitted count from a focused helper. Preserve the existing
   two-row compact content and insert the fitted strip in collapsed mode. Extract/reuse
   provider style selection without invoking alias resolution for these historical
   labels. Generalize row budgeting to accept actual metadata overhead with defaults
   preserving existing non-session callers.

4. **Make updates and resizing reliable.** Derive `rendered_row_count` from the actual
   fit; include width, summary facts, omission count, and relevant height inputs in
   repaint/change detection. Refit when width, height, or member metadata changes;
   update an already selected session when a later turn appears, even if the root's
   identity/model did not change. Check existing detail and hint render caches for
   suppressed updates; include all member facts affecting this summary in any key that
   would otherwise serve stale headers. Use existing refresh/debounce paths. Preserve
   bottom pins, focused deck/card, and expanded state as row counts change. Resize
   callbacks must converge without a paint/resize feedback loop.

5. **Document and verify the user-visible contract.** Update the relevant header section
   of `docs/ace.md`, `docs/configuration.md`, descriptions in
   `src/sase/default_config.yml` and `src/sase/config/sase.schema.json`, and relevant
   help text in `modals/help_modal/agents_main_sections.py` within its established box
   width. Explain deduplication, overflow, expanded recovery, and height interaction.
   Keep existing config values and bindings. Add the focused tests below and inspect
   actual pixels before accepting golden changes.

The summary is O(loaded turns), has no I/O, and requires only one small Rust call per
changed identity snapshot. Resizing repacks the saved summary and must not call the
backend or re-walk history. Avoid a global history cache or persistent new summary
field. Register any required binding validation entry in `tools/validate_sase_core_rs`
following neighboring bindings. Include both repositories in the host final declaration
so the host commits core first and updates `sase-core-revision.txt` before committing
the Python caller, as documented in `docs/rust_backend.md`; do not invent an uncommitted
SHA or manually create commits.

## Verification and acceptance

Use the existing session fixtures and the header/prompt-preview tests under
`tests/ace/tui/widgets/`, especially `_agent_display_agent_session_helpers.py`,
`test_agent_header_panel_basic.py`, `test_agent_header_panel_preview.py`,
`test_agent_header_preview.py`, and `test_agent_display_agent_session_roster.py`.

- Rust semantics and binding tests: A/B/A yields A/B with A latest; different efforts
  and aliases do not duplicate one model; same model name under different providers
  stays distinct; explicit prefixed/unprefixed recorded identities deduplicate; missing
  values and unresolved aliases remain unknown; monitor/gate metadata is excluded.
  Exercise empty input, whitespace, and custom slash-containing IDs.
- Adapter tests: renamed root, concrete planner proxy, overlapping runtime/followup
  links, nested monitors/gates, cycles, failed turns, and parallel children. Include a
  new unique model after turn 12; it must appear in both summary states. Verify folding
  and nav filtering do not limit the summary to visible rows.
- Pure fit tests: one and many models, exact-fit boundaries, one versus two rows, long
  names, Unicode/wide cells, provider-name collisions, very narrow widths, and
  deterministic retention of the latest model. Assert cell bounds, correct `+N more` and
  unknown counts, unmodified source text/spans, and no omissions when all full tokens
  fit the available rows. Hidden-count reservation must handle multiple digits.
- Mounted tests: collapsed defaults; expand/full recovery/collapse; resize wide to
  narrow and back; preview disabled; short/tall columns; unknown-to-known metadata; new
  member arriving with unchanged root; cheap-to-full pending prompt paint; no stale
  models after selecting another identity or empty state. Verify actual visible lines
  and height, not just the unfitted source `Text`. Preserve split-deck and bottom-pin
  behavior, and ensure the model strip stays above all deck content.
- Performance checks: assert the fit/render path does not open/stat files, resolve
  current aliases, scan archives, spawn work, or reaggregate on resize. Confirm
  unchanged updates do not force full nav rebuilds. A deterministic many-turn fixture is
  enough; run broader profiling only if measurements or a regression justify it.
- Visuals: add deterministic session-header snapshots at 160x50, 120x40, and 80x24,
  covering one model, mixed providers, two-row fit, real overflow, expanded full list,
  and coexistence with RAW PROMPT. Include a short viewport and a deck split, and
  inspect readability with the existing supported theme fixtures. Use
  `just fix-tui-screenshots -- <affected test selectors>` and inspect the retained
  report, every creation/removal, and each update group. A `partial` result is not proof
  that all relevant goldens are current. Target all affected session/header snapshots,
  not just newly added tests; do not delete unrelated goldens in a targeted run.
  Complement fixtures with a local `sase screenshot` capture and inspect its PNG.
- Run appropriate targeted Rust/Python tests, formatting, then `sase tool run check`
  from each changed repo. Follow the linked core's no-bare-cargo build guidance and
  install the local binding only into the workspace test environment when needed. Use
  `/sase_monitor` for long verification and visual commands; its continuation must
  inspect mutating screenshot reports before finalization. `just check-full` is not
  requested and is not part of this plan.

## Risks and controls

The main layout risk is spending deck height on metadata. The two-row ceiling, actual
cell measurements, and preview-first reduction make that cost explicit. The main data
risk is showing the synthetic root's model or today's alias resolution as history; the
concrete-turn projection and recorded-identity contract avoid that. The main refresh
risk is a child-only change leaving the selected root's header stale; mounted tests must
exercise that case through the ordinary refresh path, not only rebuild the header
directly. Existing history-loading limits still apply: this feature summarizes the
loaded session and converges through the existing reconciliation mechanism rather than
claiming a new exhaustive archive query.

Done means ordinary sessions display all recorded distinct models while collapsed;
space-constrained sessions visibly account for omissions and preserve the latest model;
expansion always recovers the complete known list; metadata updates and resizing remain
correct; and the model strip is readable without crowding the deck or breaking scroll
position. No source implementation changes are authorized by merely authoring this plan:
submit it through `sase plan propose` for the requested review handoff first.
