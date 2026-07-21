---
tier: tale
title: Restore view hints for agent-family details
goal: 'Pressing `v` on an agent-family container opens the normal numbered view-hint
  flow for the files and commits shown in its fold-aware detail document, without
  flattening the family view or introducing synchronous artifact discovery.

  '
create_time: 2026-07-21 08:10:23
status: wip
prompt: 202607/prompts/agent_family_view_keymap.md
---

# Plan: Restore view hints for agent-family details

## Context and root cause

The Agents-tab `v` action obtains the selected row and asks `AgentDetail` to re-render it through
`update_display_with_hints()`. For an ordinary agent, that path builds a `HeaderHintState`, passes the cached
detail-header summary through `build_header_text()`, collects numbered file and commit mappings, and keeps the hint
document live while deferred header enrichment completes.

Family-container rows diverge at that boundary. The fold-aware family rollout added an explicit early return in
`src/sase/ace/tui/widgets/prompt_panel/_agent_display_hints.py`: it calls the plain family renderer to preserve the
compact family document, then returns an empty `AgentHintRender`. This guard solved a real presentation problem—the
ordinary agent renderer must not replace a family container's fold-aware roster, context, prompt previews, and phase
replies—but it also makes the action report `No files or commits found in agent details` even when the plain family
header is visibly showing an ARTIFACTS lane. The missing capability is therefore a family-native hint renderer, not a
missing key binding or missing artifact data.

The fix should retain the existing `v` action and configurable keymap. It should make the hint renderer honor the same
family fold state and cached enrichment as the plain renderer, so the visible document and the returned hint mappings
stay in agreement.

## Family-native hint rendering

Replace the family-container empty-result branch with a dedicated path that builds the same fold-aware family document
in hint mode. Reuse the existing panel fold level and per-section overrides, family member jump-map publication,
wait-status metadata, slow-tool threshold, and cached `DetailHeaderSummary` used by the plain render. Construct a
`HeaderHintState` and pass it to `build_header_text()` so visible plan paths, deltas, artifact files, slow-tool reports,
and commits receive the established numbering and return through `AgentHintRender.file_hints`, `tool_call_reports`, and
`commit_views`.

Refactor or parameterize the shared family renderer as needed so hint mode does not duplicate or drift from its
fold-aware layout. The family roster, compact previews, section headings, phase ordering, fold overrides, and
fully-expanded content must remain identical to the plain family document apart from numbered hints. Annotate file paths
in visible family xprompt, prompt, and member-reply content using the existing path-hint rules; resolve member content
relative to that member's effective workspace rather than assuming every phase uses the family root workspace. Do not
allocate hints for content hidden by the current fold state.

Mark the render as active hint mode and cancel the slow-tool repaint timer just as the ordinary-agent path does,
preventing an unannotated refresh from overwriting the numbered family document. Keep keypress work cache-backed: do not
add filesystem scans, subprocesses, or family aggregation to the UI event path beyond the content reads already
performed by the existing family display.

## Deferred enrichment and action behavior

Preserve the ordinary two-phase header contract for families. If the selected family has no cached detail-header summary
yet, render whatever hints are immediately available, return `header_enrichment_pending=True`, and start the existing
background enrichment. This prevents `_view_agent_files()` from showing a false empty warning or abandoning hint mode
while artifacts are still loading. When `AgentDetailHeaderEnriched` arrives, the existing active-hint refresh must
rebuild the family-native hint document and replace all mapping dictionaries atomically, without clobbering typed hint
input or changing the selected family/fold state.

No changes are needed to the `v` binding, default keymap configuration, footer label, or file/commit viewer dispatch.
Once the family renderer returns truthful mappings, the existing action and processing paths should mount the hint bar
and open the selected file or commit exactly as they do for ordinary agents.

## Regression coverage

Add focused tests at both the renderer and action integration boundaries:

- A family with a cached header summary containing representative plan, delta, artifact-file, and commit entries renders
  numbered hints and returns the exact file and `CommitViewSpec` mappings instead of an empty result.
- Family xprompt/prompt/reply path hints preserve compact and fully-expanded fold contracts, enumerate only visible
  content, and resolve a child phase's relative path against the child workspace.
- A family without a cached summary reports enrichment pending, remains in hint mode, and gains the cached
  artifact/commit mappings on the existing enriched repaint path.
- Invoking the Agents-tab view action for a family with displayed artifacts mounts/refocuses the hint input rather than
  emitting the empty-details warning; the ordinary-agent empty-state behavior remains unchanged.
- Existing fold-aware family rendering tests continue to prove that plain and hint modes share the same roster, section
  order, preview bounds, overrides, and phase ordering.

Keep the fixtures synthetic and cache-controlled so tests do not depend on a live agent, repository state, or background
timing. Add a visual assertion only if the implementation changes an intentional non-hint rendering contract; the
expected fix is interaction behavior plus transient `[N]` annotations, not a new persistent family layout.

## Validation

Bootstrap the ephemeral workspace first with `just install`. Run the focused family-display,
header-artifact/delta/commit-hint, async-enrichment, and view action tests while iterating. Re-run the existing
hint-survives-refresh and family fold suites to catch repaint or layout regressions. Finally run the repository-mandated
`just check` and manually exercise `v` on a collapsed family whose detail pane shows artifacts, confirming that numbered
choices appear and that selecting both a file and a commit reaches the existing viewers.

## Risks and guardrails

The main risk is allowing the plain and hint family documents to evolve separately. Prefer one shared family layout path
with optional hint state over a second copied renderer. Numbering must be deterministic across cached repaints, and
mappings must describe only the currently rendered numbers. Child-relative paths and commit repository metadata must
retain their own workspace/repository attribution. Finally, the fix must not weaken the original family guard by routing
containers through the ordinary full-agent document or by doing new disk discovery on the keystroke path.
