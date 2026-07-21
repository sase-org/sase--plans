---
tier: tale
title: Commit-first comprehensive update confirmation
goal: 'The Updates-tab comprehensive-update confirmation leads with a responsive,
  repository-grouped preview of the commits the captured update will introduce, while
  remaining truthful and usable when sources are large or unavailable.

  '
create_time: 2026-07-21 11:11:17
status: done
prompt: '[202607/prompts/comprehensive_update_commits.md](prompts/comprehensive_update_commits.md)'
---

# Plan: Commit-first comprehensive update confirmation

## Product outcome

Make the confirmation opened by `u` in the SASE Admin Center's Updates tab answer the user's first question—“what code
will this bring in?”—before presenting the lower-level component and command plan. The top of the modal should contain
an “Incoming commits” section whose repositories are easy to distinguish and whose commit subjects are easy to scan; the
existing comprehensive execution preview and `y`/`n` decision remain below it.

This is an informational preview, not a second source of update authority. The update must still execute the immutable
comprehensive plan that produced the confirmation, and failure to obtain commit metadata must never broaden, replace, or
silently alter that plan.

## Commit scope and data contract

- Derive a frozen list of repository commit-source specifications from the SASE leg of the captured
  `ComprehensiveUpdatePreview`, then pass a loader for that frozen list into `PluginActionConfirmModal`. Do not inspect
  mutable pane selection or inventory after the modal opens.
- For editable updates, include each actionable, deduplicated checkout root and compare the planned local ref with the
  exact planned upstream ref. Exclude dirty, diverged, current, blocked, and otherwise skipped roots because they will
  not be fast-forwarded by this update.
- For managed SASE/core/plugin updates, map only the packages represented by the plan to their loaded core/catalog
  entries and compare installed and target release refs. The broad managed `uv tool upgrade sase` fallback may use the
  loaded set of updatable SASE/core/plugin candidates because that command intentionally resolves the whole installed
  set. In a mixed editable/managed plan, union the exact editable roots with the exact managed candidates and dedupe by
  repository/range while preserving plan order.
- Reuse `CommitSourceSpec`, `RepoIncomingCommits`, the existing complete-cache seeding rules, and grouped fetch helpers.
  Keep repository labels stable and human-oriented (`sase`, `sase-core`, plugin name), with one subsection per unique
  repository range and newest commits first.
- Do not infer commit history for Agent-CLI provider commands. Their package managers/installers are opaque in the
  current plan model and expose no trustworthy base/head repository range; their version transitions and exact commands
  remain visible in the execution-plan section. If the comprehensive update contains only opaque provider work, show a
  calm empty state explaining that no repository commit ranges are available rather than fabricating a comparison.
- Honor `ace.updates.incoming_commits.enabled` and the existing confirmation limit. Within that safety bound, show every
  fetched commit. When a repository exceeds the bound, retain its authoritative total and render an explicit “+N more”
  marker so the UI never claims a truncated list is complete. Document the confirmation-specific limit.

## Modal experience

- Generalize the reusable confirmation layout so any supplied incoming-commit preview appears before the action preview.
  This establishes a consistent “what changes → what will run → decide” hierarchy for comprehensive, SASE, and
  individual plugin updates; install/uninstall confirmations without commit data remain unchanged.
- Render the top section immediately with a lightweight checking state. Once loaded, give it a concise aggregate heading
  (repository and commit counts), followed by visually separated repository subsections. Each subsection should lead
  with the repository name and count, then aligned rows with a subdued short SHA and high-contrast subject. Avoid
  redundant duplicate repository summaries; unavailable data is an inline, subdued state for that repository.
- Preserve the modal's wide comprehensive layout and fixed decision buttons. Allocate bounded space to commits and the
  execution preview only when content overflows, and make `ctrl+d`/`ctrl+u` traverse the two regions in visual reading
  order: down exhausts the top commit region before the plan, while up reverses that sequence. Keep border scroll hints
  synchronized with actual overflow and ensure neither scrollbar nor either button escapes the modal at compact and
  normal terminal sizes.
- Keep `y`, `n`, `Esc`, and button clicks available while commit metadata loads. The section is best-effort context, so
  a slow GitHub response must not freeze first paint or prevent confirmation. A top-level loader error, a per-repo
  unavailable result, zero commits after revalidation, and dismissal during loading must all settle cleanly without a
  traceback or late mutation of a detached screen.
- Use the existing cyan update accent, border language, typography, and glyph-plus-color status vocabulary. Refresh the
  comprehensive visual golden from the actual grouped-commit state and inspect the PNG at both normal and compact
  dimensions for hierarchy, clipping, line wrapping, button placement, and visual noise.

## Implementation boundaries

- Add comprehensive-preview-to-commit-loader composition alongside the existing Updates-pane incoming-commit helpers,
  and wire it only after the comprehensive plan has passed its current runnable/blocker checks. Seed it from the core
  and plugin commit snapshots already loaded by the pane, refetching only incomplete previews at the larger confirmation
  limit.
- Keep all git, GitHub, config, and cache work in the existing thread worker. The UI thread should only compose the
  loading state and apply typed worker results; capture all loader inputs before starting the worker, ignore late
  results after teardown, and avoid adding async work to Textual's serial message-pump callbacks.
- Evolve the presentation API narrowly enough to express an explicit empty-state message and commit-first placement
  without coupling the generic modal to comprehensive-update models. Continue using the same immutable comprehensive
  preview for confirmation and tracked-task submission.
- This is TUI presentation/planning glue over the established incoming-commit domain primitives. Do not create a second
  commit-fetch implementation or move shared backend behavior into the TUI; no Rust-core wire change is needed unless
  implementation uncovers genuinely new cross-frontend domain semantics.

## Verification

- Unit-test source selection for managed, editable, and mixed comprehensive previews, including exact package scoping,
  stable ordering, shared-root/range deduplication, cache reuse, skipped/current exclusion, disabled/offline behavior,
  provider-only empty state, and partial source failures.
- Extend the comprehensive Updates-pane flow test to prove the modal receives the loader derived from the same captured
  preview and that confirming still submits that unchanged preview. Cover both direct `u` dispatch and the automatic
  comprehensive handoff path represented by the current screenshot.
- Extend modal interaction tests for loading, multi-repository rendering, truthful truncation, explicit empty/error
  states, commit-first widget order, dismissal during work, and directional scrolling when the commit and execution
  regions both overflow. Assert first paint and cancel/confirm remain responsive while a loader is held.
- Drive the comprehensive PNG snapshot through deterministic grouped commit data with at least two repositories and
  enough execution detail to exercise hierarchy and scrolling. Add or update a compact snapshot if the shared layout
  changes individual update confirmations, then inspect actual/expected/diff artifacts rather than accepting goldens
  blindly.
- Run `just install` before repository checks, execute the focused incoming-commit, modal, comprehensive-update, and PNG
  snapshot tests while iterating, then finish with `just check` as required for SASE source/style/test changes.

## Acceptance criteria

- Opening the Updates-tab comprehensive confirmation shows “Incoming commits” above the execution plan whenever commit
  previews are enabled, with commits grouped under distinct repository headings and sourced only from work the captured
  plan can perform.
- The preview clearly distinguishes complete, truncated, empty, and unavailable results and never invents commit ranges
  for opaque provider updates.
- Network and git inspection never block ACE's event loop; the modal remains cancellable/confirmable during loading and
  is safe to dismiss at any point.
- Both content regions and the decision buttons remain readable and reachable at supported compact and normal terminal
  sizes, and the final visual snapshots demonstrate a clean commit-first hierarchy.
