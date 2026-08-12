---
tier: tale
title: Dynamic artifact document tabs and logical Files versions
goal:
  ACE derives artifact document tabs from enabled-project providers and presents one
  version-aware row per logical file.
size: medium
proposed_by: bbugyi200.athena.sase-js.7
bead: sase-js.7
create_time: 2026-08-12 07:46:21
status: wip
---

- **PARENT:**
  [202608/artifact_ref_contract.md](https://github.com/sase-org/sase--plans/blob/main/202608/artifact_ref_contract.md)
- **BEAD:**
  [sase-js.7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-js/sase-js.7.md)

# Dynamic artifact document tabs and logical Files versions

## Goal

Replace ACE's fixed nested `Files -> Plans | Chats | Other` structure with the
provider-driven top-level Artifacts layout described by `sase-js.7`:

`Stitches | Patches | Beads | <configured document providers> | Files`.

Document-provider tabs must be derived from enabled-project configuration, use stable
`ref:<kind>` identities, preserve the existing Plans experience through the generic
documents pane, and stay lazy/off-thread. Files must become one row per logical file,
merge the legacy artifact index with the published ref-file version index, expose
version navigation on `(`/`)`, and visibly distinguish `ref`, `created`, and `capture`
origins.

## Constraints and decisions

- Keep `stitches`, `patches`, `beads`, and `files` as stable fixed identities. Provider
  identities are `ref:<kind>`; labels come from the effective provider spec with a
  title-cased kind fallback. The visual order puts providers before Files, while fixed
  digit bindings retain their existing meanings and provider digits are allocated only
  after the fixed range.
- Resolve the union from enabled project records and each project's effective sidecar
  ref policies, not merely the installed provider registry. Cache descriptors by the
  current configuration/project-state inputs and expose an explicit reset seam for tests
  and config/project lifecycle refreshes.
- Normalize legacy persisted `prs` and `bugs` values as today. Any unknown or no-longer
  configured provider selection falls back to `DEFAULT_ARTIFACTS_SUBTAB` without
  raising.
- Keep every data-scaled read off Textual's message pump. Tabs and panes remain mounted
  under a `ContentSwitcher`, but each provider inventory and Files index is loaded only
  on first activation, coalesces refreshes, rejects stale worker results, and caches by
  provider-spec/project/repository revision inputs.
- Reuse the current MIME-aware file preview/viewer/materialization paths. The UI model
  may adapt a selected logical version to `ArtifactFile` for those paths, but it must
  not create a parallel renderer.
- Treat old artifact-index rows as migration input. Map `explicit=True` to `created` and
  automatic rows to `capture`, combine them with `query_ref_file_versions()`, and
  deduplicate versions by logical identity plus full SHA-256 while unioning provenance.
- Do not modify `sase-core`: the released ref-file fold already provides logical rows,
  ordered versions, origins, object-relative paths, agents, projects, MIME, size, and
  first-seen timestamps needed by this presentation phase.

## Implementation

1. **Introduce the runtime Artifacts tab registry.**
   - Replace the `Literal`/nested-Files tab model in `artifact_tabs.py` with
     string-based stable ids and immutable tab descriptors containing id, label, accent,
     pane id, provider kind/spec digest/spec, and digit shortcut.
   - Implement `resolve_artifacts_subtabs()` over enabled project records, effective
     project config, document sidecar roles, and `effective_sidecar_ref_policies()`.
     Deduplicate providers by stable kind, order deterministically, retain fixed digit
     assignments, and add cache reset/normalization helpers.
   - Update `PanelTabStrip` only as needed to render an explicit shortcut rather than
     assuming the strip position is the shortcut.

2. **Flatten Artifacts composition and state.**
   - Remove `FilesSubTab`, `current_files_subtab`, `ArtifactsFilesView`, its watcher,
     nested cycling, and the old Plans/Chats/Other pane-key translation.
   - Compose fixed panes, one `ArtifactsDocumentsPane` per resolved provider, and the
     new Files pane directly in `ArtifactsView`; route lifecycle, selection, detail,
     project scope, refresh, marks, jump history, click handling, commands, clipboard,
     action availability, testing-page state, onboarding, and help through the stable
     top-level pane id.
   - Generate top-level digit bindings/actions from the resolved descriptor set at app
     construction while keeping 1-4 stable for Stitches/Patches/Beads/Files. Promote
     `[`/`]` as the always-valid dynamic-tab navigation in help/onboarding.
   - Remove the obsolete Chats pane/actions/filter surface and lazy exports; retain chat
     transcript access through agent detail/artifact flows outside this retired pane.

3. **Generalize Plans into a provider-driven documents pane.**
   - Refactor the `plans_*` data/list/filter/detail/navigation modules behind provider-
     neutral document models. An inventory row carries project, provider kind, stable
     repo-relative identity, repository/root/revision, parsed frontmatter properties,
     body, timestamps, and optional plan-specific proposal/bead linkage.
   - Load only the selected provider's configured roots for the current project scope,
     enumerate repo-relative candidates off-thread, apply the shared Rust artifact-ref
     path filter, parse configured frontmatter properties, and cache snapshots by
     provider digest, project config, and repository HEAD/source signatures.
   - Drive filter facets and detail labels/order from the provider spec's `properties`
     and `detail` sections. Keep common free-text/project/date filtering and preserve
     Plans proposal/active/archive grouping, approval/rejection, linked-bead jumps,
     preview, filtering, and navigation when the provider kind is `plan`; other
     providers render ordinary document inventories without new pane code.

4. **Rebuild Files around logical identities and versions.**
   - Replace `FilesSnapshot.rows: ArtifactFile` with immutable logical-file and version
     models. The loader reads both `query_ref_file_versions()` and
     `query_artifact_files()` off-thread, resolves object-relative paths through known
     agent-sidecar checkouts, groups one row per portable logical identity, deduplicates
     unchanged digests, unions agent/project provenance, and sorts deterministically by
     most-recent first-seen time.
   - Widen the old explicit/default presentation to `ref | created | capture`; add an
     origin badge column, origin facet/completions/counts, and origin-aware detail text.
   - Store the selected version index per logical row. Add previous/next-version actions
     on `(`/`)`, clamp/reset selection when snapshots change, keep the logical row
     selected, and render `version i/n`, full digest, capture time, agents, projects,
     origin, MIME, size, and object location in the detail header.
   - Adapt open/preview/external/copy/agent actions to the selected version and reuse
     the existing MIME classifier, detail loader, materializer, and rich viewer. Stable
     marks/jumps target the logical identity, not a physical path or version digest.

5. **Update keymaps, styling, exports, and tests.**
   - Replace nested-Files cycle keymap fields/metadata/commands with Files version
     actions bound to the freed parenthesis keys in `default_config.yml`; update the
     keymap dataclass, command catalog/availability, help modal, footer hints,
     onboarding, test helpers, and public widget exports. Remove CSS and modules made
     dead by the collapsed panes.
   - Add focused unit tests for dynamic enabled-project union, disabled projects,
     provider removal and stale selection fallback, deterministic labels/order/cache
     reset, fixed and provider digit stability, and bracket cycling.
   - Add documents-pane parity tests for Plans filtering/detail/navigation/actions plus
     a second synthetic provider proving the pane is spec-driven and lazy.
   - Add Files tests covering legacy/ref-index merge, one logical row for repeat
     captures, unchanged-digest provenance union, all three origin badges/facets,
     version wraparound/navigation, selected-version actions/detail, and stale worker
     suppression.
   - Refresh the dedicated ACE PNG snapshots for the new tab strip, provider documents
     pane, empty Files pane, populated logical Files pane, and alternate version detail.

## Verification

1. Run focused non-visual suites for artifact tab resolution, the Artifacts scaffold,
   documents-pane behavior, Files data/filter/detail/actions/navigation, command/help/
   keymap behavior, and configuration/project enable-disable invalidation.
2. Run the artifact navigation benchmark (or the nearest focused `SASE_TUI_PERF` test)
   to ensure loaded-row navigation performs no filesystem/config work and remains within
   the existing p95 target.
3. Run `just test-visual --sase-update-visual-snapshots`, inspect the actual refreshed
   frames, then rerun `just test-visual` without update mode.
4. Run `just check`; if its selector escalates or reports unusual coverage, run
   `just check-full` as required by repository policy.
