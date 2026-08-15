---
tier: tale
title: Give every Artifacts pane one shared shell and visual grammar
goal:
  Every configured Artifacts pane, including Patch and degraded third-party panes,
  renders through one contract-driven chrome with deterministic loading, empty, results,
  stale, and degraded states; provider accents are stable, perceptually spaced,
  theme-legible, and propagated through provider-specific chrome.
size: medium
proposed_by: bbugyi200.athena.sase-m6.5
bead: sase-m6.5
create_time: 2026-08-14 21:24:49
status: done
---

- **PROMPT:**
  [prompts/202608/artifacts_shared_shell.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/artifacts_shared_shell.md)
- **PARENT:** [202608/artifacts_pane_contract.md](artifacts_pane_contract.md)
- **BEAD:**
  [sase-m6.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m6/sase-m6.5.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-m6.5](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-m6.5.md)
- **COMMITS:**
  - [d19d086](https://github.com/sase-org/sase/commit/d19d08641246a2b0f9276fded07d93004815d640)
    — feat(tui): give every Artifacts pane a shared shell and visual grammar

# Give every Artifacts pane one shared shell and visual grammar

This is phase `shell` / bead `sase-m6.5` of the Artifacts-pane-contract epic. The parent
design is `@plan:202608/artifacts_pane_contract.md`. Its non-negotiable rules remain in
force: providers declare data rather than UI code, provider work never runs on render or
navigation paths, failures remain visible and isolated, and navigation p95 stays below
16 ms.

## Verified baseline

- Phase `contract` has landed. Every resolved sub-tab has an `ArtifactsPaneContract`
  carrying its id, label, icon, accent, provider kind, empty-state copy, detail-scroll
  id, capabilities, and declared-fact diagnostics.
- `ArtifactsSnapshotPane` already performs collection in worker threads and coalesces
  refresh requests. Beads, Files, and document panes retain their last snapshot while a
  new load runs, but each independently renders loading/error/count state. Stitches has
  a separate worker lifecycle, and Patch state is app-owned.
- Pane chrome currently drifts: Stitches, Patch, Beads, Files, documents, and degraded
  providers compose different headers, status lanes, empty states, detail regions, and
  footers. The host knows the active contract but does not render the pane through a
  shared visual-state model.
- A generic document pane still uses `ARTIFACTS_ACCENTS["plans"]` throughout its scope
  row, filter bar, list separators, detail metadata, and hints. Therefore a Research
  pane renders Plans-purple even though its descriptor and tab use the provider's
  accent.
- `_provider_accent_for_kind` is stable and excludes pinned built-in colors, but its
  six-color source was only a foundation mechanism. After built-in colors are removed,
  the effective palette is too small and was not selected or tested for perceptual
  spacing and legibility across ACE's dark and light themes.
- Existing visual coverage pins populated and empty Stitches, Beads, Files, and Plans
  panes, but it does not cover the canonical loading/stale/degraded grammar or a
  non-Plan provider using its own accent.

## Visual grammar and state contract

Add a host-owned shell vocabulary under `src/sase/ace/tui/widgets/artifacts/` and
document the same rules in `docs/artifacts_pane_visual_grammar.md`.

The vertical order is invariant for every pane:

1. the pane's inline filter slot (collapsed when that pane's current query UX does not
   expose one),
2. one identity/scope header built from the active contract,
3. one compact state/count lane,
4. the content region, using the shared split classes and a stable detail-scroll region
   when the contract advertises one, and
5. one footer-hint lane.

The header identifies the pane with its contract icon, singular label, and accent, then
shows project scope and active filter chips. Bespoke information such as Patch fold
levels, Stitch repository presence, Bead triage counts, and File origin counts belongs
in the state/count lane or the pane's rows, never in a second identity header. The
footer uses the existing configured keymap text, but every pane mounts it in the same
position and with the same separators, height, truncation, and accent treatment.

Represent visible state as a closed `ArtifactsPaneState` enum plus an immutable, purely
presentational shell-state record. Resolve state with this precedence:

- `degraded`: contract/discovery failure, or an initial load failure with no usable
  content;
- `loading`: first load for the current scope, with no cached content;
- `stale`: a refresh is running while usable content from the current scope remains;
- `empty`: the current scope loaded successfully but has no rows; distinguish an empty
  inventory from zero matches under an active query;
- `results`: usable rows are present and no refresh is in flight.

Loading renders a compact progress affordance in the shell instead of substituting a
blank list. Stale keeps the current selection, list, and detail visible and overlays a
`Refreshing` badge; it never rebuilds from disk or clears content on the event loop. An
empty-inventory card uses `contract.empty_state`, while a no-match card names the active
filter and includes the configured key that edits or clears it. Degraded keeps the tab
named and navigable, then shows provider kind, configuration source, stable diagnostic
code when available, the validation problem, and a direct recovery hint. Runtime errors
over existing content preserve that content and use the stale/error badge rather than
replacing the pane.

Patch is contract-in like every other pane, but its query migration remains in the later
`query` phase. This phase moves Patch identity/status/empty/footer presentation into the
shell without changing its established actions, grouping, detail renderer, or modal
query semantics.

## Implementation

### 1. Add the contract-driven shell primitives

- Create `src/sase/ace/tui/widgets/artifacts/shell.py` with the closed state enum,
  immutable shell-state input, a pure state resolver, and Rich/Textual renderers for the
  common identity header, state badge/count lane, empty/no-match card, degraded card,
  and footer grammar.
- Make the shell consume an `ArtifactsPaneContract`; no renderer may look up a pane id
  in `ARTIFACTS_ACCENTS`, invoke provider code, resolve providers, touch the filesystem,
  or perform data-scaled work.
- Add shared semantic classes in `styles.tcss` for shell header, state lane, panels,
  empty/degraded card, detail scroll, and footer. Preserve existing ids that actions and
  tests query, but make pane-specific selectors only supply content-specific layout
  where the shell cannot.
- Keep split-mode widths owned by `ArtifactsView`; normalize the list/detail shell
  classes so all panes inherit the existing narrow/even/wide behavior without adding a
  new refresh path.

### 2. Drive snapshot panes through the five states

- Extend `ArtifactsSnapshotPane` with a small state-notification seam that derives
  first-load versus cached-refresh state from `_snapshot`, project generation,
  `_loading`, and `_load_error`. Invoke it on request start and every terminal worker
  outcome; do not add async timer callbacks or synchronous collection.
- Convert Beads, Files, and `ArtifactsDocumentsPane` to compose the common header, state
  lane, main split, empty/degraded surface, and footer while retaining their specialized
  option/detail renderers and existing selector ids.
- Remove their duplicate loading/error/empty prose once the shell owns it. Preserve
  loaded rows and the selected stable target during refresh. Route provider label, icon,
  accent, empty copy, and source through the contract.
- Parameterize the document filter bar and document render helpers by `contract.accent`
  so `ref:research` and a synthetic unknown provider no longer render Plans-purple. The
  built-in Plan adapter keeps its pinned purple through the same path rather than a
  special case.

### 3. Adapt Stitches, Patch, and degraded descriptors

- Give Stitches the same shell-state input using its existing collection worker and last
  `VcsLogResult`: no result plus worker is loading, a result plus worker is stale, zero
  commits is empty/no-match, a successful non-empty result is results, and a first-load
  worker failure is degraded. Keep collection and filtering off-thread and leave
  commit/diff rendering specialized.
- Wrap `ArtifactsPatchesPane` in the shared chrome. Derive Patch loading and refresh
  flags from the app-owned Patch lifecycle without moving data ownership into the
  widget. Reuse the existing quickstart content inside the canonical empty surface and
  preserve list/detail/action behavior.
- Replace the one-off `ArtifactsDegradedPane` hero/card with the common degraded
  renderer, including descriptor `provider_kind`, `error_source`, `error_code`, and
  `error`. It must remain focusable through the tab strip and safe when no provider spec
  was parsed.
- Have `ArtifactsView` pass the resolved contract and descriptor diagnostics into every
  shell. Keep pane composition host-owned and do not allow sidecar-provided markup,
  widgets, command strings, or colors.

### 4. Replace the provider palette and prove its properties

- Replace `_PROVIDER_ACCENTS` with a curated palette large enough to distinguish eight
  or more simultaneous providers. Select colors in a perceptual color space, then pin
  their hex values so runtime assignment remains dependency-free and deterministic.
- Preserve the existing SHA-256 `ref_kind` assignment and the pinned Plan accent.
  Filter/collision-resolve against all built-in accents and assert palette membership,
  pairwise perceptual spacing, and readable contrast against the supported dark and
  light shell surfaces. Installing or removing an unrelated kind must not repaint any
  existing pane, and provider discovery must not mutate `ARTIFACTS_ACCENTS`.
- Audit the tab strip, split badge, filter bar, header, list separators, detail accents,
  empty/degraded cards, and footer so every provider-specific surface receives the
  descriptor/contract accent rather than a module-global Plans fallback.

### 5. Document and test the grammar

- Add focused unit tests for the state-precedence table, first-load versus stale
  refresh, no-inventory versus no-match copy, runtime error with and without cached
  content, degraded diagnostics, contract accent propagation, and palette stability,
  spacing, reserved-color exclusion, and theme contrast.
- Extend `tests/ace/tui/artifacts_contract/harness.py` so every configured descriptor,
  including the synthetic third-party and degraded fixtures, proves it can render the
  shared shell without provider code and with the descriptor's contract identity.
- Add or update TUI integration tests proving refresh leaves cached rows/selection
  visible and a coalesced follow-up request transitions `stale -> results` without
  blocking the message pump.
- Add deterministic PNG cases for the five canonical states, including a non-Plan
  provider with its own accent and the designed degraded surface. Update existing
  Artifacts goldens only after inspecting actual/expected/diff artifacts; do not accept
  unrelated renderer movement.
- In the visual-grammar document, record layout order, state precedence, exact required
  information, accent rules, accessibility constraints, provider-data boundary,
  extension checklist, and Patch's contract-in/spec-out asymmetry.

## Verification

1. Run `just install` before all repository checks.
2. Run focused shell, artifact-contract, pane lifecycle, filter-bar, split-mode, and
   affected Artifacts integration tests while iterating.
3. Run the affected Artifacts PNG nodes, inspect `.pytest_cache/sase-visual/`
   actual/expected/diff/source artifacts, update only intentional goldens, and rerun
   those nodes without the update flag.
4. Exercise the synthetic provider with at least eight deterministic provider kinds on
   both `textual-dark` and `textual-light`; assert each surface uses its contract accent
   and the degraded case stays named/navigable.
5. Run the navigation benchmark or equivalent `SASE_TUI_PERF=1` harness for Patch,
   Stitches, Beads, Files, Plan, and the synthetic provider. Require p95 below 16 ms and
   confirm no render/keypress path performs provider resolution, file I/O, subprocess
   work, or filesystem metadata reads.
6. Run `just check`. Because this phase touches Artifacts broadly and changes PNG
   goldens, run `just check-full` only through `/sase_monitor` with a `--next` action,
   then run the full `just test-visual` suite and require it to pass.
7. Inspect `git diff --check`, the final diff, and `git status --short`. Re-show
   `sase-m6.5`, record any out-of-scope discovery only as a `PROPOSED FOLLOW-UP:` note,
   and close only this phase with a note listing the focused tests, visual review,
   performance result, `just check`, and monitored `just check-full` result.

## Out of scope

- Do not implement the unified query engine, move Patch to the inline filter editor, add
  relation/grouping behavior, change the Artifacts keymap, or add `ref.pane`; those
  belong to later epic phases.
- Do not change the Rust provider-spec wire or run provider code in the TUI.
- Do not edit SASE memory files or generated instruction shims.
- Do not close parent epic `sase-m6` and do not create task beads from this phase.
