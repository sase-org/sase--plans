---
tier: tale
title: Render TUI child memories as linked references
goal:
  TUI memory reads stay concise and point agents to the screenshot and performance notes
  on demand.
size: small
proposed_by: bbugyi200.athena.0mn
create_time: 2026-09-18 05:30:55
status: wip
---

# Render TUI child memories as linked references

## Goal

Stop recursively inlining the `tui_screenshot.md` and `tui_perf.md` child-note bodies
when `tui.md` is shown or read. Keep both child notes discoverable at the bottom of the
parent output as numbered linked references that agents can read individually.

## Implementation

1. In `sase/memory/tui.md`, change the two forced-inline links `![[tui_screenshot.md]]`
   and `![[tui_perf.md]]` to reference links `[[tui_screenshot.md]]` and
   `[[tui_perf.md]]`. Preserve their order and the parent note's existing entry-point
   guidance. Do not change the linked child notes or the memory renderer: existing link
   semantics already make plain links use the default reference rendering.
2. Run `sase memory init` to republish memory-derived files. Review the resulting diff
   and retain only the canonical note change and expected generated updates; do not
   hand-edit generated `AGENTS.md` or provider instruction shims.
3. Verify both user-facing paths. `sase memory show tui.md` and an audited
   `sase memory read tui.md --reason "Verify TUI child memories render as references"`
   must keep the short `# TUI` entry-point body, place `tui_screenshot.md` and
   `tui_perf.md` in the final numbered `## Linked References` section after
   `## Children`, and omit the expanded `# TUI Screenshot Capture` and
   `# TUI Performance Gotchas` bodies. Confirm each child remains independently readable
   with
   `sase memory read tui_screenshot.md tui_perf.md --reason "Verify direct TUI child-memory reads"`.
4. Run `sase memory init --check`,
   `pytest tests/memory/test_memory_selector.py tests/memory/test_memory_selector_render.py`,
   and the required repository-wide `just check`. Escalate into renderer changes or
   broader tests only if these checks expose an actual renderer defect.

## Acceptance Criteria

- The canonical `tui.md` source uses plain `[[...]]` links for both TUI child notes and
  no longer forces either child inline.
- Default Markdown output from both `sase memory show tui.md` and
  `sase memory read tui.md` lists both child selectors under `## Linked References` at
  the bottom, after `## Children`, without embedding either child body.
- Direct audited reads of `tui_screenshot.md` and `tui_perf.md` still resolve and print
  their complete content.
- Memory initialization is clean, generated instruction surfaces are refreshed only
  through `sase memory init`, and all required focused and project checks pass.
