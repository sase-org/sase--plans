---
tier: tale
title: External-issue presentation and actions in the Beads pane
goal:
  Beads present and operate on cached external tracker issues without blocking
  navigation or depending on the retiring Bugs pane.
size: medium
proposed_by: bbugyi200.athena.sase-jd.6
bead: sase-jd.6
create_time: 2026-08-10 21:24:37
status: wip
---

- **PARENT:**
  [202608/external_artifact_ingestion.md](https://github.com/sase-org/sase--plans/blob/main/202608/external_artifact_ingestion.md)
- **BEAD:**
  [sase-jd.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-jd/sase-jd.6.md)

# External-issue presentation and actions in the Beads pane

## Goal

Complete phase `bead_bug_ui` from `@plan:202608/external_artifact_ingestion.md` while
the Bugs sub-tab still exists. The Beads pane must remain bead-only, but every bead that
mirrors or references tracker issues must expose cached remote state, reconciliation
drift, relationship detail, reverse links, issue-aware filters, and capability-gated
tracker actions without performing provider I/O from a render or navigation path.

This phase is additive. Do not remove or rename the Bugs pane, do not change the
Artifacts sub-tab order, and do not perform the later `tabs` phase. Do not make Beads
depend on modules that the `tabs` phase is instructed to delete.

## Current seams to preserve

- `Issue.external_ref`, project-qualified `normalize_external_ref`, and
  `find_external_ref_links` already exist. `external_ref` is the mirror identity; `bug:`
  entries in `Issue.refs` are ordinary references. Preserve that distinction.
- `supports_issue_listing`, `supports_issue_reads`, and `supports_issue_mutations` are
  structural probes. Do not collapse them back into the all-or-nothing `supports_issues`
  check.
- `ArtifactsBeadsPane` loads local bead state off-thread and renders from immutable
  snapshots. Keep remote provider work off Textual's event loop and serial message pump,
  retain cached rows during refresh, and revalidate project/selection before applying
  results.
- The Bugs pane remains operational during this phase. Shared provider-neutral tracker
  helpers may be extracted from Bugs code, but Beads-specific behavior must not import
  `widgets/artifacts/bugs.py`, `bugs_rendering.py`, `actions/artifact_bugs.py`, or the
  Bugs-pane compatibility facade that the later phase will delete.
- Use `#FF5F5F` as the external accent. Introduce a shared `EXTERNAL_ACCENT` name while
  retaining the Bugs accent mapping/alias until the later removal phase.

## Implementation

### 1. Build a provider-neutral, bounded external-issue cache

Extract or add a neutral tracker adapter below `src/sase/ace/tui/` that resolves a
project to its display name, ProjectSpec, workspace, provider, and the three issue
capabilities. It should expose pure/frozen result records plus off-thread functions for
listing, reading, creating, updating, and resolving URLs. Update the still-live Bugs
backend to use or re-export these helpers so behavior is not duplicated.

Extend the Beads pane data model with per-project external-issue snapshots containing:

- issues indexed by project-qualified normalized ref and number;
- listing/read/mutation capabilities;
- successful refresh time, provider error/unavailable reason, and whether the bounded
  listing reached its configured cap;
- a stable cache/source token suitable for invalidating the filter index.

Refresh at most once per represented project per cache TTL, using one bounded
`state="all"` listing call per project. A forced Beads refresh bypasses the TTL. Keep
the last successful snapshot visible while a refresh is in flight or fails, and expose
the failure separately rather than replacing cached issues with an empty inventory. Load
local bead rows first or continue displaying the previous local snapshot, then run the
remote batch through a thread worker/pump-free task with coalescing and
last-request-wins guards. Never call a provider, stat/glob the store, or parse external
data from row rendering, filtering, detail navigation, or footer availability.

For an all-project Beads scope, refresh each enabled project independently so one
unsupported or failing provider does not hide other projects. A scoped project refresh
must not fan out to unrelated projects. Treat a missing remote record as stale only
after a successful, non-truncated inventory; an unavailable or capped inventory is
unknown, not stale.

### 2. Centralize bead-to-issue relationship classification

Add a pure presentation model/helper used by rendering, detail, filtering, and actions.
For each bead, canonicalize and deduplicate `external_ref` plus every `bug:` ref using
the bead's stable project key. Preserve order with the mirrored `external_ref` first.
For every link record:

- classify it as `mirrored` when it matches `Issue.external_ref`, otherwise
  `referenced`;
- attach the cached `IssueWire` when available;
- classify drift when upstream open/closed disagrees with whether the bead itself is
  closed;
- classify stale only under the complete-inventory rule above;
- compute reverse links with `find_external_ref_links` against cached in-project beads
  and Patches already held by ACE, with no I/O.

Keep remote-only issues out of the option list. Compute and surface their unmirrored
count in status data so the mirror gap is visible without manufacturing pseudo-bead
rows.

### 3. Render chips, status, and detail from cached relationships

Define `EXTERNAL_ACCENT = "#FF5F5F"` in a shared Artifacts presentation location and
have the live Bugs renderer retain a compatibility alias to it.

Update task, epic, and phase row builders so the external issue chip appears after the
title and before the existing +1/reopen/post-close badge cluster:

- `○#<number>` for an upstream-open issue and `●#<number>` for upstream-closed, in bold
  external-accent text;
- `?#<number>` for a conclusively stale link;
- the same chip as `bold #1a1a1a on #FF5F5F` when stale or drifting;
- ` +N` for additional deduplicated links, without silently discarding them.

Use the same classifier for all row kinds rather than inheriting a parent epic's legacy
`patch_bug_id`. Ensure row output remains single-line and ellipsized.

Extend the Beads status line with an aggregate external-cache summary. A scoped healthy
case should read like `external · refreshed 4m ago`; unavailable, stale-cache, capped,
and all-project mixed states must remain explicit. Include the count of remote issues
that have no covering bead.

Add an `External issue` section to the selected bead detail and preview. For each linked
issue show state, display-project ref, author, comment count, relative update time,
labels, URL, relationship (`mirrored by this bead` versus `referenced by this bead`),
and reverse-linked beads/epics/Patches. Include the cached remote body in the preview so
the Bugs-pane body remains viewable from Beads. When a bead has multiple issues, render
a bounded summary of all of them and use a picker for actions that require one target.

### 4. Add issue-aware Beads filters

Extend `sase.bead.filter_query` and the Beads filter bar with repeatable, negatable
`bug:` and `label:` keys and their completion/help metadata. `bug:` accepts `none`,
`open`, `closed`, `drift`, or a numeric issue number. `label:` matches cached provider
labels case-insensitively.

Update the prefolded Beads filter index so:

- `has:bug` means the bead has `external_ref` or a `bug:` ref, not `patch_bug_id`;
- `bug:none/open/closed/drift/<number>` uses the centralized relationship classifier;
- `label:<value>` uses cached labels;
- cached issue title, body, labels, author, URL, and normalized refs participate in
  free-text search;
- the index source key changes when external cache contents change, but navigation does
  not rebuild or fetch data per keypress.

Add static and dynamic completions for the new keys (known states/numbers and observed
labels) while preserving query round-tripping, negation, the default `-status:closed`,
and existing filters.

### 5. Migrate capability-gated issue actions into Beads

Add a dedicated Beads external-issue action mixin and reuse the neutral tracker adapter
instead of importing the Bugs action module. Keep every slow mutation in the tracked
task queue and every browser/clipboard/read operation pump-free or off-thread.

Provide these actions for the selected bead:

- open an issue in the browser (`o`) and copy its canonical `bug:` ref (`y`) only when
  the bead has an external link; choose from a picker when there are multiple links;
- view cached remote body/metadata through the existing bead detail/preview;
- edit an issue and close/reopen it via `be` and `bs`, preserving `e` and `s` as bead
  edit/status actions;
- attach an existing issue to an unlinked bead and create an issue prefilled from an
  unlinked bead, then persist the canonical project-qualified `external_ref` and
  resolvable `bug:` reference through the bead-store mutation/commit path;
- refresh the selected scope's external cache through the existing Beads refresh action.

Implement a Beads-scoped `b` prefix mode (or an equivalent scoped key-sequence state) so
it does not steal `b` outside the Beads pane. Include edit/state plus attach/create
sub-actions in that mode and make the mode/footer conditional on the selected row and
the relevant capability. Do not introduce a global custom-mode prefix that activates on
unrelated tabs.

Gate operations separately:

- listing controls batch refresh, cached presentation, and the attach picker;
- reads control exact issue retrieval/URL fallback when the cache is insufficient;
- mutations control create/edit/close/reopen;
- cached URL/body data remains usable even if mutation capability is absent.

After a successful provider mutation, update the cached immutable issue record
optimistically/on completion and repaint rows, filters, detail, status, and footer
without waiting for another network refresh. Roll back optimistic state on failure.
After an attach/create mutation, force a local Beads reload while retaining the current
stable selection. Report uniqueness/conflict failures without replacing an existing
mirror identity.

Thread all new actions through the action registry, configurable app keymaps,
`src/sase/default_config.yml`, bindings, action availability, command palette metadata,
and the help modal. Preserve the 57-character help box and the conditional-footer rule:
`o`, `y`, and issue-mutation mode hints appear only when their row/capability conditions
can vary.

### 6. Tests and verification

Add focused unit/integration coverage for:

- canonical link deduplication, mirrored-versus-referenced relationships, multiple
  links, open/closed agreement, both drift directions, stale versus unavailable/capped,
  and reverse links;
- row chip text/style/order for open, closed, drift, stale, and `+N` cases across bead
  row kinds;
- detail/preview content, display project names, cached body, and no remote-only rows;
- one bounded listing call per project, TTL reuse, force refresh, independent project
  failures, cached fallback, coalescing/last-request-wins, and no provider call during
  option rebuild, filtering, highlight movement, or detail painting;
- `has:bug`, every `bug:` value, `label:`, negation, completions, free text, and filter
  index invalidation on external cache changes;
- split capability combinations and action/footer/command availability, including
  multi-link picking, browser/copy, tracked edit/state/create, attach persistence, cache
  repaint, and selection preservation;
- help/keymap/default-config consistency and the existing Bugs pane continuing to work
  through the extracted neutral adapter.

Update the Beads text snapshots and the dedicated Artifacts Beads PNG goldens for the
new chip/status/detail states. Use only already-audited glyphs (`○`, `●`, `?`) and add a
glyph-audit test only if the existing audit does not already cover the rendered set. Run
`just install` before verification, then targeted Beads/Bugs/filter/keymap tests,
`just test-visual` with visual diff inspection and intentional golden updates, and
finally `just check`. If scoped selection escalates or reports an unusual selection, run
`just check-full` as required by the repository guidance.

Do not close the parent epic. After all implementation and verification pass, close only
`sase-jd.6` with a note naming the targeted tests, visual verification, and final check
result. Record any genuinely separate discovered work only as a `PROPOSED FOLLOW-UP:`
note on `sase-jd.6`.
