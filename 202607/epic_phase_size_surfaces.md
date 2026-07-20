---
tier: epic
title: Beautiful epic phase sizes across SASE surfaces
goal: 'Epic phase sizes are visible at a glance and attached to the correct phase
  wherever SASE presents epic plan metadata, with authoritative validation, safe legacy
  fallbacks, consistent accessible visuals, and no responsiveness or notification
  regression.

  '
phases:
- id: ace-agent-context
  title: Normalized size-aware ACE agent context
  depends_on: []
  size: medium
  description: '''Normalized size-aware ACE agent context'' section: carry authoritative
    phase sizes through cached plan enrichment and render consistent, responsive size
    chips for epic authors, landers, and the exact phase context shown to workers.

    '
- id: ace-plans-surface
  title: Size-aware Artifacts Plans surface
  depends_on:
  - ace-agent-context
  size: medium
  description: '''Size-aware Artifacts Plans surface'' section: reuse the ACE size
    vocabulary in phase list rows and bead detail properties while preserving complete
    authored properties and the worker-only data-loading contract.

    '
- id: telegram-epic-reviews
  title: Glanceable Telegram epic phase sizes
  depends_on:
  - ace-agent-context
  size: medium
  description: '''Glanceable Telegram epic phase sizes'' section: add a compact validated
    size breakdown to epic review messages without duplicating phase metadata, breaking
    MarkdownV2, exceeding Telegram''s budget, or weakening delivery fallbacks.

    '
- id: cross-surface-verification
  title: Cross-surface contract and visual verification
  depends_on:
  - ace-agent-context
  - ace-plans-surface
  - telegram-epic-reviews
  size: small
  description: '''Cross-surface contract and visual verification'' section: audit
    every epic-plan presentation path, exercise modern and legacy plans end to end,
    inspect visual output, update user documentation, and run both repositories''
    required checks.

    '
create_time: 2026-07-20 14:07:37
status: wip
---

# Plan: Beautiful epic phase sizes across SASE surfaces

## Outcome and verified surface inventory

Phase size is already a real execution property, not a new display-only label. The Rust-backed plan validator returns
normalized `phases[].size` values and enforces the current authoring enum (`small`, `medium`, or `large`). Its
launch-consumption mode is the compatibility contract: a missing pre-feature size becomes `small` with a warning, while
an explicit invalid value remains an error. Epic creation copies the authored size onto the phase bead, and phase launch
routing uses the persisted bead value.

The presentation rollout is incomplete and uneven:

- The ACE Agents-tab PLAN roadmap already shows each validated phase's title, ID, dependencies, optional model, and
  description for epic authors and landers, but its immutable display projection drops `size`. It also validates with
  strict authoring mode, so an otherwise valid pre-size epic can now lose its whole roadmap as `phases unavailable`. A
  selected phase worker receives a deliberately phase-local BEAD lane, but that exact phase context also omits its size.
- The Artifacts Plans pane has typed phase beads available in its worker snapshot, yet expanded phase rows and phase
  detail properties omit `Issue.size`. Proposal, archived-plan, and linked-plan property/source views already include
  the authored `phases` value (and therefore authored `size`) as complete plan data; those paths need an audit and
  regression coverage, not a second competing roadmap.
- Telegram's Properties card recursively renders the full `phases` mapping, so modern sizes are technically present.
  They are incidental, can be buried inside an expandable or truncated value, and provide no glanceable scope summary
  beside the existing phase count.
- Epic clan summaries already provide the strongest visual precedent: labeled, right-aligned blue/gold/rose chips and
  `small` fallback for legacy sizeless beads. `sase bead show`, the epic work preview, strict plan validation output,
  raw plan review/source views, PDFs, and mobile plan attachments already expose phase size. They should be locked into
  the audit matrix but should not gain redundant metadata.

This definition of “all surfaces” deliberately distinguishes purpose-built summaries from lossless source views. Every
compact roadmap, phase row, and review summary should make size obvious; full-source and generic property surfaces
should continue showing the authored field exactly once.

## Product and data contract

### One semantic truth, two legitimate sources

Authored-plan surfaces (agent PLAN/BEAD context and Telegram review) must consume the normalized Rust validator result.
Use launch-consumption mode for display so historical plans with a missing size remain readable as `small`; never
downgrade a malformed or explicitly invalid size. Validation failure keeps the established honest fallback: the Agents
panel shows one quiet unavailable state, while Telegram omits only the synthetic size summary and still delivers the raw
Properties card, body, attachment, and controls.

Execution/bead surfaces (Artifacts Plans phase rows and bead details) must use the persisted `Issue.size`, because that
is the value current work routing will honor and it may intentionally have been edited after plan creation. A missing
legacy bead size uses the same stable `small` fallback already used by work routing and clan summaries. Do not silently
compare the authored and persisted values or overwrite either source.

Keep size separate from status, progress, dependencies, model, and tier. It is the author's scope estimate and routing
input; colors and placement must not suggest that `large` is unhealthy or that `small` is complete. Fixed summary
ordering is always `small`, `medium`, `large`, omitting zero-count buckets where compactness matters.

### Accessible visual vocabulary

Reuse the established clan-summary palette and make the word itself the primary signal: blue `small`, gold `medium`, and
rose `large`. A user must never need color vision to distinguish the values. Consolidate the pure Rich chip/style
construction used by ACE surfaces (and, where practical without changing output, the clan summary) so labels, padding,
foreground contrast, and legacy normalization cannot drift.

In the author/lander PLAN roadmap, place a compact size chip on every phase title row, right-aligned when space permits.
The flexible title column folds completely while the fixed size label remains visible; narrow panels and metadata zoom
retain the same authored order and never ellipsize phase data. The unwrapped logical-text form must include the size
label too, preserving search, copy, style inspection, and header splicing behavior.

In a phase worker's BEAD lane, add only that phase's `Size` value. Preserve the current information boundary: phase
workers still do not receive the other roadmap phases, goal, dependencies, or models merely because size is now visible.

In Artifacts Plans, place the labeled chip before the phase title in each single-line expanded row so the identity,
readiness state, and size survive before title ellipsis. Add a `Size` property to a selected phase's detail and a
compact `Phase sizes` breakdown to an epic bead's detail, derived from its already-loaded direct phase children. Keep
proposal/archive/linked-plan property rows complete and avoid duplicating their authored `Phases` property with another
full roadmap.

Telegram cannot reproduce terminal colors reliably, so use a calm textual line below the review heading, for example
`Phase sizes: 2 small · 1 medium · 1 large`. This line is a derived review summary, not authored frontmatter, and stays
visually separate from the lossless Properties card. The detailed phase mapping remains unchanged and continues
associating each `size` with its phase; the summary stays glanceable even when that mapping becomes expandable or
value-truncated.

## Normalized size-aware ACE agent context

Extend the immutable associated-plan phase projection with a typed size and carry it from `ValidatedPlanPhase` through
the existing file-signature cache. Validate epic display data in launch-consumption mode inside the deferred enrichment
worker: modern plans preserve exact sizes, missing legacy sizes normalize to `small`, and all other schema errors keep
the phase list all-or-nothing. Do not add YAML work, validation, stats, bead queries, or subprocesses to Rich rendering
or the immediate selection path.

Extend the phase-local summary produced from the same validated plan entry so a selected phase worker can render its
exact size. Missing, unreadable, out-of-range, or invalid parent-plan data should show an honest unavailable value
without exposing other phases. Preserve the current plan-path cache keys, mtime/size invalidation, negative TTL, role
detection, author/phase/land separation, artifact deduplication, and family propagation.

Introduce or extract one small Rich phase-size chip helper with the established palette and accessible labels. Use it in
the responsive PLAN phase title table and the BEAD definition rows; keep logical and width-aware rendering
text-equivalent. Refactor the clan summary onto the same constants/helper only if its standalone script constraints
remain intact and its serialized markup and visual output stay unchanged.

Coverage must include all three modern sizes, a legacy missing-size plan normalized to small, explicit invalid sizes
remaining unavailable, phase-worker isolation, author and lander role resolution, cache reuse/invalidation, exact
styles, narrow and wide Unicode folding, metadata zoom, logical text/search visibility, and file-hint numbering. Update
the focused epic-roadmap and phase-context PNG fixtures to show all three sizes, inspect their pixel diffs, document the
size semantics in `docs/ace.md`, and confirm the Agents j/k path remains memory-only and within its established
performance target.

## Size-aware Artifacts Plans surface

Reuse the ACE phase-size helper for the Plans pane's typed bead views. Expanded phase rows remain one line and retain
the existing order of status, ID, readiness, size, and title; the title alone yields to ellipsis. The detail property
grid adds `Size` only for phase issues and uses an explicit `small` fallback for legacy beads. Epic bead details add a
concise fixed-order count breakdown from the phase children already in the immutable snapshot, omitting the row when no
phase context exists rather than inventing plan data.

Keep all snapshot reads, issue loading, and linked-plan parsing in the existing worker path. The change should need no
new filesystem reads, no render-time validation, and no Rust/backend mutation: `Issue.size` and phase children already
cross the shared backend boundary. Preserve readiness glyphs, progress counts, stable row targets, filtering, project
scope, expand/collapse behavior, detail debouncing, and list focus.

Audit proposal, archive, and linked-plan details explicitly. Their complete authored `Phases` property/source already
contains per-phase sizes, so retain it once and add tests using valid small/medium/large frontmatter to prevent a future
flattening or filter-index change from dropping the labels. Do not parse flattened archive strings back into a second
pseudo-schema. If the audit reveals a genuine structured-data gap needed by another frontend, extend the Rust
plan-search wire and Python mirror rather than adding TUI-only parsing; that is a contingency, not expected scope.

Add focused row/detail tests for every size, legacy fallback, phase-only `Size`, epic breakdown grammar/order, narrow
ellipsis placement, and no duplicate properties. Update the populated Plans visual fixture/golden if the size labels are
visible there, and amend the Artifacts Plans documentation with the distinction between authored plan size and current
persisted bead size.

## Glanceable Telegram epic phase sizes

In the linked `sase-telegram` repository, extend the existing single-read plan formatting pipeline. Feed the
already-read document content to SASE's normalized epic validator in launch-consumption mode and, only on a successful
epic result, compute a fixed-order count of normalized sizes. Retain the current best-effort raw-sequence phase count
contract independently so a later property-format fallback does not erase the heading count.

Render a nonempty `Phase sizes:` line immediately below the bold review heading and agent attribution, composing cleanly
with provider/model and runtime context. Omit zero buckets and the whole line for zero phases. A missing legacy size
counts as `small`; an invalid size, invalid epic structure, unavailable validator capability, or preview exception omits
the derived line without affecting message delivery. Do not duplicate validation rules in the plugin or trust arbitrary
`size` strings as known values. Align the plugin's SASE dependency floor/release ordering with the first SASE release
that exposes the required normalized size and launch-mode API; older capabilities degrade by omission, not by a guessed
summary.

Include the line in the existing shared 4,096-character budget before allocating notes, Properties-card, and body space.
Preserve every property label, the nested per-phase `size` values, expandable-blockquote behavior, value/body truncation
markers, original plan/PDF attachment, MarkdownV2 escaping, gate bundle loading, callback payloads, and keyboard layout.

Extend formatter and command-backed gate coverage with exact readable output for one size, mixed sizes, repeated sizes,
and legacy missing sizes; provider/model, agent, and runtime composition; metadata-heavy truncation; invalid or
nonmapping phases; missing/unreadable files; and property-rendering fallback. Assert the complete message budget,
attachment, and controls in every degradation case. Update the Telegram README and outbound documentation with the
summary line, its relationship to the detailed Properties card, and its best-effort omission semantics.

## Cross-surface contract and visual verification

Perform a final inventory against every epic-plan presentation path and record why each one changed or correctly
remained unchanged. The acceptance matrix must cover:

- ACE author, lander, and phase-worker selections;
- Artifacts Plans proposal, epic, phase, linked-plan, and archive selections;
- Telegram epic approval messages and their attached source/PDF;
- epic clan summaries, `sase bead show`, epic dry-run/launch previews, the TUI raw plan approval modal, CLI
  validation/schema output, and mobile attachment plumbing.

Use one representative modern epic with small/medium/large phases and one valid pre-feature epic with missing sizes.
Confirm every compact semantic surface displays the intended labels, every generic/source surface retains the authored
fields exactly once, modern invalid sizes never produce a confident badge or count, and phase-worker views remain
phase-local. Check that an authored-plan view and a mutated bead view each show their own authoritative source rather
than silently reconciling them.

Run `just install` before repository validation. In the main SASE checkout, run the focused model/render/Plans tests,
affected visual snapshots, `just test-visual`, the relevant j/k performance benchmark or trace, and the mandatory
`just check`. In the linked Telegram checkout, run focused formatter/gate tests and `just check`. Inspect all changed
PNGs and Telegram MarkdownV2 fixtures rather than accepting snapshots or escaped text mechanically. Finish with
clean-worktree/diff audits in both repositories and a documentation sweep that uses the same `small`, `medium`, and
`large` meanings.

## Boundaries and risks

- No new phase-size enum, routing rule, or Rust validation behavior is expected. The existing core validator and bead
  model remain authoritative. A discovered backend payload gap must be fixed in `sase-core`, its binding, and mirrors
  before frontend work continues; frontends must not fork the domain rule.
- The size chips consume horizontal space. Fixed chips must remain visible while only flexible titles fold or ellipsize;
  narrow and wide-Unicode tests plus visual review protect this hierarchy.
- Color can accidentally imply health or progress. Every chip contains its literal label, status glyphs keep their
  existing palette and column, and Telegram uses text rather than color-coded emoji.
- Telegram already contains detailed phase sizes. The derived summary must remain visibly separate from authored
  Properties and participate in the hard message budget so improved glanceability never costs delivery reliability.
- Launch-mode compatibility is deliberately narrow: only a missing historical size becomes `small`. Invalid values and
  unrelated schema damage remain unavailable so the UI never turns corrupt metadata into a confident estimate.
