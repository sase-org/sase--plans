---
tier: tale
title: Keep Agent Conversation Sections Unfolded
goal: Agent and agent-family xprompt, prompt, and reply sections always show full
  content and never participate in metadata folding.
create_time: 2026-07-28 07:02:35
status: done
---

- **PROMPT:** [202607/prompts/unfold_agent_conversation_sections.md](prompts/unfold_agent_conversation_sections.md)

# Keep Agent Conversation Sections Unfolded

## Goal

Remove folding from the `AGENT XPROMPT`, `AGENT PROMPT`, and `AGENT REPLY` sections in the Agents-tab metadata panel for
ordinary agents and real agent-family container rows. These sections should always render their available content
without a fold glyph, bounded fold preview, or section-level fold override. Keep the existing compact fold scales for
the metadata that benefits from them: family members, lane neighbors, output/workflow variables, SASE context, slow tool
calls, errors, and the clan/tribe aggregate summaries.

The user-visible contract is:

- An ordinary agent continues to show full xprompt, prompt, and live/consolidated reply content, as it does today.
- A family container now follows that same conversational-content contract at every family panel fold level.
- `zz`, `zZ`, and direct level chords may still change a family roster and its other fold-aware metadata, but never
  shorten or otherwise change these three conversation bodies.
- `za` and `zA` do not create or change a per-section override when the viewport is on one of these three fold-inert
  headings. They continue to work for genuinely foldable sections and numbered member rows.
- The three headings remain semantic metadata-navigation anchors, so `Ctrl+J`/`Ctrl+K`, scroll preservation, search, and
  hint-mode section tracking continue to work.
- Missing xprompt/prompt sections remain omitted for families, pending replies retain their placeholder, the family
  reply heading may retain its useful phase count, and terminal ordinary-agent `AGENT CHAT` behavior is unchanged.

## Current Behavior and Boundaries

`src/sase/ace/tui/widgets/prompt_panel/_agent_display_render.py` and
`src/sase/ace/tui/widgets/prompt_panel/_agent_display_hints.py` already render ordinary-agent xprompt, prompt, and reply
sections with plain headings and full content. Family containers take the separate
`AgentFamilyDisplayMixin._update_family_display()` path in
`src/sase/ace/tui/widgets/prompt_panel/_agent_display_family_render.py`; that path currently resolves the shared family
fold level and per-section overrides for all three sections, uses fold glyph headings, and chooses between 12-line /
four-line previews and full bodies.

The shared fold level is still needed by family headers and non-conversation metadata. Do not remove
`FAMILY_FOLD_SCALE`, the panel fold header, the per-section override registry, member/neighbor fold behavior, or
clan/tribe folding. Do not make synchronous disk reads or extra render-path discovery calls: use the content already
loaded by the existing family render path, preserve lazy Markdown rendering, and retain the total hint-content budget
and annotated-document cache. The hint budget is a performance/safety cap, not a user fold, and should continue to show
its existing truncation notice when reached.

## Implementation

1. Make family conversation rendering fold-inert.
   - In `src/sase/ace/tui/widgets/prompt_panel/_agent_display_family_render.py`, stop resolving effective family fold
     levels or consulting `section_fold_overrides` for `AGENT XPROMPT`, `AGENT PROMPT`, and `AGENT REPLY`.
   - Render those headings through the plain semantic section-heading path, without `▸`/`▾`/`▼`, while preserving the
     stable `agent-xprompt`, `agent-prompt`, and `agent-reply` anchor identities. Preserve the consolidated reply phase
     count without presenting it as a fold indicator.
   - Always use the existing full xprompt highlighting, full/lazy prompt Markdown, and full per-phase reply rendering
     branches in normal mode. In hint mode, annotate the same full logical content while retaining `HintContentBudget`,
     phase-specific workspace resolution, cache wrapping, and the truncation notice.
   - Keep the family error traceback and all header-owned sections on their current effective fold levels. Keep absent
     xprompt/prompt omission, reply ordering/dividers, timestamped chunks, and pending/no-response placeholders intact.

2. Remove obsolete family-conversation fold helpers without disturbing helpers still used by real metadata folds.
   - In `src/sase/ace/tui/widgets/prompt_panel/_agent_display_family.py`, remove the preview-only helpers and constants
     that become unused (`bounded_content_preview`, `reply_tail_preview`, and any conversation-specific fold-heading
     identifiers no longer needed by production code).
   - Retain `append_family_fold_heading()` and `effective_family_fold_level()` for `FAMILY MEMBERS`, output/workflow
     variables, context, slow tools, errors, and other remaining family fold owners.
   - Prefer the default semantic identities produced by `append_section_heading()`, or a small shared non-folding
     heading helper if the reply count needs one; do not conflate a navigable section marker with fold eligibility.

3. Make section-level fold commands explicitly ignore the three fold-inert anchors.
   - In `src/sase/ace/tui/actions/navigation/_fold.py`, classify `agent-xprompt`, `agent-prompt`, and `agent-reply` as
     fold-inert before calling `SectionFoldStateManager.cycle()` or `.toggle()`. Leave the registry and current panel
     level untouched for these headings, refresh normally, and preserve current behavior for foldable family metadata,
     member rows, clan sections, tribe sections, and `NEIGHBORS`.
   - Keep panel-wide fold chords active: they still control the remaining metadata and clear real per-section overrides.
     Do not disable fold mode for regular-agent lanes because their `NEIGHBORS` section still owns the three-level agent
     scale.
   - Update the Agents fold-mode labels/help in `src/sase/ace/tui/commands/_mode_commands.py` and
     `src/sase/ace/tui/modals/help_modal/agents_bindings.py` to say that `za`/`zA` act on foldable sections or members,
     without changing the configured keys.

4. Replace preview-oriented coverage with invariants for always-full conversation sections.
   - Rewrite the focused cases in `tests/ace/tui/widgets/test_agent_display_family_render.py` to render
     collapsed/expanded/fully-expanded shared levels and explicit conversation-section overrides, then assert identical
     full xprompt, prompt, and per-phase reply bodies, plain headings with stable section IDs, no fold glyphs, preserved
     reply counts, omitted empty sections, and pending reply placeholders.
   - Update `tests/ace/tui/widgets/test_agent_display_family_hints.py` so both the default and fully expanded family
     levels expose hints from the full conversation content, while the aggregate hint cap and phase workspace mapping
     remain enforced.
   - Update affected expectations in `tests/ace/tui/widgets/test_agent_display_xprompt.py` and
     `tests/ace/tui/widgets/test_agent_display_family_roster.py`; remove tests for production constants/helpers that no
     longer exist, but retain direct assertions that the rendered navigation identities are stable.
   - In `tests/ace/tui/test_agents_panel_fold_mode.py`, prove `za` and `zA` leave overrides untouched on all three
     fold-inert identities and still mutate a representative foldable family/clan section.
   - In `tests/ace/tui/widgets/test_summary_fold_contracts.py`, stop using `AGENT PROMPT` as the family section expected
     to change between adjacent fold levels. Use a genuinely foldable family section (such as `FAMILY MEMBERS`) for the
     cross-kind ladder contract, and add a family-specific assertion that the three conversational bodies do not change
     across the scale.
   - Preserve or extend `tests/ace/tui/widgets/test_prompt_panel_section_navigation.py` coverage to verify the plain
     family headings remain reachable anchors. Update/add an intentional family-panel PNG snapshot that visibly covers a
     conversation heading/body at both family levels; accept only the goldens caused by removing these folds.

5. Align user documentation and performance verification.
   - Update the family-fold descriptions and key tables in `docs/ace.md` and `docs/agent_families.md`: family fold
     levels still shape compact metadata, but xprompt, prompt, and reply are always full and `za`/`zA` skip their
     headings. Keep the documented ordinary-agent and `NEIGHBORS` behavior accurate.
   - Update the family view-hints scenario wording in `docs/perf_runbook.md` so default and fully expanded metadata
     states are not described as differing in conversational-content visibility. Keep the committed JSON baseline as the
     documented synchronous pre-optimization reference rather than rewriting historical measurements.
   - Adjust the current scenario assertions in `tests/perf/bench_tui_trace.py` only as needed to prove default family
     rendering now scans full-but-capped content as well as the fully expanded state. Keep the existing cache-hit,
     auto-refresh, latency, and 200 KB aggregate scan gates intact.

## Validation

Run from the repository workspace after `just install`:

1. Focused functional tests:

   ```bash
   pytest \
     tests/ace/tui/test_agents_panel_fold_mode.py \
     tests/ace/tui/widgets/test_agent_display_family_render.py \
     tests/ace/tui/widgets/test_agent_display_family_hints.py \
     tests/ace/tui/widgets/test_agent_display_family_roster.py \
     tests/ace/tui/widgets/test_agent_display_xprompt.py \
     tests/ace/tui/widgets/test_prompt_panel_section_navigation.py \
     tests/ace/tui/widgets/test_summary_fold_contracts.py
   ```

2. Dedicated visual coverage, inspecting generated actual/expected/diff artifacts before accepting intentional family
   goldens:

   ```bash
   just test-visual
   ```

3. TUI performance regression floor for the now-always-full family conversation path:

   ```bash
   just view-hints-perf-check
   ```

   Confirm the family render remains within the shared hint scan cap and the hint bar stays off the synchronous keypress
   path. Do not refresh the committed pre-optimization baseline.

4. Full required repository gate:

   ```bash
   just check
   ```

Manual acceptance in `sase ace`:

- Select an ordinary agent with xprompt, prompt, and live/follow-up reply content. Cycle panel levels and press
  `za`/`zA` with each heading at the top; the bodies and override registry must remain unchanged.
- Repeat on a real family container at levels 1 and 2. The three plain headings and full bodies must be unchanged, while
  `FAMILY MEMBERS`, `NEIGHBORS`, and another foldable metadata section still respond to their fold chords.
- Enter hint mode on both family levels and confirm file hints cover the same visible conversation content, numeric
  section navigation still reaches all three headings, and large family content shows the existing cap notice without
  freezing navigation.
