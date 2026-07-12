---
create_time: 2026-07-11 12:42:11
status: done
prompt: 202607/prompts/agents_view_hint_race.md
tier: tale
---
# Stabilize Agents-tab view hints against deferred detail repaints

## Context and root cause

Pressing `v` on the Agents tab synchronously replaces the selected agent's prompt with a hint-annotated render and
mounts the `View:` input bar. App-level detail refreshes already notice `_hint_mode_active` and preserve that annotated
render, but entering hint mode does not advance `AgentDetail`'s render generation.

The normal detail render may already have launched deferred prompt work: detail-header enrichment, linked-delta
resolution, bead metadata resolution, workflow rendering, or the periodic slow-tool metadata tick. Those callbacks use
the selected agent identity and `AgentDetail` generation to reject stale results. Because the hint render currently
leaves the generation unchanged, a callback started before `v` still looks current and can run the plain prompt renderer
after the hint-annotated renderer. The numbered markers then disappear while the `View:` input bar and its original
mappings remain active. The timing and the selected running agent in the screenshot are consistent with this race,
particularly the five-second slow-tool repaint.

This is presentation-only Textual state, so the fix belongs in the Python TUI rather than the shared Rust backend.

## Implementation plan

1. Make the Agents detail widget's hint-annotated render an explicit render-generation transition. Entering or
   refreshing hint mode will invalidate callbacks and timer contexts created by the preceding plain render before the
   prompt is annotated. Keep the existing generation/identity checks as the single authority for whether deferred work
   may repaint, avoiding a parallel hint-mode check in every worker and preserving the current fast-path architecture.

2. Add focused regression coverage around the generation boundary and deferred repaint sources. Verify that entering
   hint mode advances the generation, that work captured by the prior plain render is rejected, and that the slow-tool
   tick cannot replace an active hint render. Retain coverage for the existing app-level immediate, debounced, and full
   refresh paths so repeated hint-preserving renders continue to update the active file/commit/tool mappings.

3. Verify the interaction end to end at the appropriate layers: run the targeted Agents detail, async prompt-worker,
   slow-tool tick, and view-hint regression tests; then run `just install` followed by the repository-required
   `just check`. Inspect any visual snapshot impact rather than updating goldens unless the intended stable-hint
   behavior genuinely changes a captured state.

## Expected result

Once `v` opens the selector, numbered markers and their mappings remain stable until the user submits or cancels the
input. Deferred metadata may finish and populate caches, but callbacks from the superseded plain-render generation
cannot repaint the prompt; normal dynamic rendering resumes through the existing full refresh when hint mode closes.
