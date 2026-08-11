---
tier: tale
title: Add external-issue context and actions to the Beads pane
goal:
  The Beads pane presents cached external issue relationships, reconciliation state,
  filters, details, and capability-gated actions without blocking navigation, while the
  existing Bugs pane remains available.
size: medium
proposed_by: bbugyi200.athena.sase-jd.6
bead: sase-jd.6
create_time: 2026-08-11 06:12:29
status: wip
---

- **PARENT:**
  [202608/external_artifact_ingestion.md](https://github.com/sase-org/sase--plans/blob/main/202608/external_artifact_ingestion.md)
- **BEAD:**
  [sase-jd.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-jd/sase-jd.6.md)

# Plan: Add external-issue context and actions to the Beads pane

## Objective

Complete epic phase `sase-jd.6` (`bead_bug_ui`) additively while the Bugs sub-tab still
exists. The Beads pane will show project-qualified external issue relationships,
reconciliation drift, remote metadata, filters, and tracker actions without performing
provider I/O on the UI thread or during row rendering. The existing bead actions and the
Bugs pane remain available; this phase does not retire or rename any Artifacts sub-tab.

## Existing seams and invariants

- `Issue.external_ref`, `normalize_external_ref`, and `find_external_ref_links` have
  already landed. `external_ref` is the mirrored identity; a `bug:` entry in `refs`
  without a matching `external_ref` is a reference.
- The provider boundary now exposes independent listing, read, and mutation capability
  probes. Every affordance must use the narrow capability it needs.
- `ArtifactsBeadsPane` already loads bead stores on a worker, keeps a source-keyed
  filter index, and debounces detail painting. Provider calls must join that worker
  path, never a row renderer, keypress handler, or Textual pump callback.
- The pane can show all enabled projects. Cache and errors must therefore be isolated
  per stable project key, while user-visible copy uses each configured display name.
- `#FF5F5F` is the external accent: plain bold text means externally anchored; a filled
  badge means reconciliation is required. Keep the Bugs accent mapping alive until the
  later `tabs` phase removes that pane.
- Preserve the existing bead primaries: `e` edits the bead and `s` changes bead status.
  Issue edits and state changes use the `b` prefix (`be`, `bs`).

## Implementation

### 1. Build an immutable, bounded external-issue cache beside the bead snapshot

- Refactor the tracker helper seam in `src/sase/ace/tui/artifacts_bugs.py` only as much
  as needed to expose a resolved project scope, independent capability flags, bounded
  issue listing, individual issue reads, issue mutations, and URL resolution. Preserve
  the Bugs pane's existing behavior and public helpers.
- Add immutable external-issue project/link records to the Beads data model. A project
  cache record contains stable/display project identity, workspace/ProjectSpec paths,
  listing/read/mutation capabilities, one bounded `state=all` issue tuple, refresh time,
  completeness/truncation state, and a provider error if one occurred.
- Extend the Beads worker load to refresh each resolved project at most once per cache
  TTL (60 seconds, following `_BUG_CACHE_TTL_S`) and exactly once when forced. Reuse
  fresh prior project records across local bead reloads and project-scope switches. Keep
  local bead errors and tracker errors independent. Include external cache identity in
  the filter source key so filters rebuild when remote state changes.
- Normalize `external_ref` and every `bug:` ref, deduplicate them, and precompute the
  issue relationships for every bead. Mark the matching `external_ref` as `mirrored` and
  refs-only matches as `referenced`. Determine open/closed, drift (remote and bead
  closure disagree), and stale (a successful complete tracker refresh cannot find the
  target). Never infer stale from an unavailable or truncated cache.
- Precompute remote-only/unmirrored counts and local reverse links without creating
  synthetic bead rows. Keep provider calls and bead/Patch discovery outside render and
  navigation paths.
- Make pane activation and explicit refresh revalidate cache age without blocking first
  paint. Continue coalescing concurrent loads and apply results only when scope and load
  generation still match.

### 2. Render the external relationship on rows, status, detail, and previews

- Define a named `EXTERNAL_ACCENT` in the Artifacts type/style seam and point the still
  live Bugs accent at it.
- Add a pure issue-chip builder used by task, epic, and phase rows immediately after the
  bead title and before existing `+1`/reopen metadata. Render `○#N` or `●#N` in bold
  external text, `?#N` as the stale form, and use the filled dark-on-external badge for
  drift/stale. Bound multiple links as `<primary chip> +N`; do not add another prefix
  glyph.
- Extend the Beads status line with the external cache age or a precise per-project
  unavailable/truncated summary, plus the bounded unmirrored count when nonzero.
- Add an `External issue` detail section containing state, project display name and
  number, author, comments, updated age, labels, URL, `mirrored by this bead` versus
  `referenced by this bead`, and reverse-linked beads/epics/Patches from
  `find_external_ref_links`. For multiple links, show all bounded relationships and use
  a picker before single-issue actions.
- Add a remote-body preview that uses the already cached `IssueWire`; opening detail or
  moving selection must never fetch.

### 3. Add external issue filter tokens to the existing in-memory index

- Extend the bead filter grammar, canonical token rendering, completion kinds, and
  filter-bar help with repeatable/negatable `bug:` and `label:` keys.
- Implement `bug:none`, `bug:open`, `bug:closed`, `bug:drift`, and `bug:<number>` from
  the precomputed link classification. Provider `label:<value>` matches cached tracker
  labels case-insensitively.
- Redefine `has:bug` as any `external_ref` or normalized `bug:` ref; do not use the
  legacy epic `patch_bug_id` as the criterion. Fold external refs, issue numbers,
  titles, labels, and URLs into the cached search haystack without provider work.
- Cover positive, negated, repeated, missing, unavailable, stale, drift, label, and
  numeric cases, including filter-index invalidation when only remote state changes.

### 4. Migrate capability-gated issue actions into Beads

- Add selected-link helpers that resolve one cached issue directly or show an issue
  picker for multiple links. Action availability and the conditional footer derive from
  the selected bead, cache completeness, and the narrow provider capabilities.
- Keep direct conditional bindings `o` for browser open and `y` for copying the
  canonical `bug:<display-project>#<number>` reference. Add a context-gated Beads issue
  prefix on `b`, backed by the existing configurable mode machinery, with sub-actions
  for viewing the cached body, editing (`be`), closing/reopening (`bs`), copying the
  URL, attaching an existing issue, and creating an issue for an unlinked bead. `R`
  refreshes both bead and external caches.
- Run browser resolution, reads, mutations, and local bead writes as tracked background
  tasks. Reuse the issue edit and confirmation modals, and add only the small
  issue-selection/attachment UI needed for multiple or entered issue numbers.
- Attaching canonicalizes the selected number, sets `external_ref` only when the bead
  has no mirrored identity, and appends the canonical `bug:` ref without duplication.
  Creating seeds the issue editor from the bead title/description, creates remotely,
  then records both identities locally. Surface a clear partial-failure message if the
  remote issue succeeds but the bead-store link fails.
- Apply completed edits/state changes to the in-memory cache immediately, invalidate its
  TTL, and schedule a coalesced refresh while preserving the selected bead.
- Register every new action in bindings, keymap metadata, command-palette metadata and
  availability, action allowlists, `AppKeymaps`, and `src/sase/default_config.yml`.
  Update the 57-column help modal and conditional footer labels. Do not remove Bugs
  actions, copy targets, aliases, or pane code in this phase.

### 5. Verify behavior and visual vocabulary

- Add focused data tests for one list call per project, TTL reuse, force refresh,
  per-project capability/error isolation, no stale classification on degraded/truncated
  data, drift classification, multiple-link ordering, and no remote-only rows.
- Add rendering/detail tests for open/closed/drift/stale chips, `+N`, relationship text,
  reverse links, external status age, and cached body previews.
- Add action tests for capability gating, multi-link selection, direct open/copy,
  `be`/`bs`, attach/create identity writes, deduplication, partial failures, tracked
  task keys, selection preservation, command-palette availability, default keymaps, and
  help content.
- Add parser/filter tests for `has:bug`, every `bug:` value, `label:`, negation, and
  dynamic completions.
- Run `just install` before repository checks. Run focused Beads/Bugs/provider/keymap
  tests, then `just test-visual`; inspect generated PNG diffs before accepting any
  intentional Beads golden changes and rerun the visual suite. Finish with `just check`.
  If the scoped selector escalates or reports unusual selection, run `just check-full`.

## Non-goals

- Do not remove the Bugs pane, rename `prs`, reorder Artifacts tabs, or change the
  Commits/Stitches label; those belong to the later `tabs` phase.
- Do not create mirror beads for remote-only issues or close beads automatically when an
  upstream issue closes.
- Do not add provider-specific GitHub behavior, polling daemons, mirror cursors, or SASE
  memory/glossary edits.
