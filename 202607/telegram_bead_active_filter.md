---
create_time: 2026-07-10 14:45:34
status: done
prompt: 202607/prompts/telegram_bead_active_filter.md
tier: tale
---
# Keep Telegram `/bead` Results Active-Only

## Context and root cause

The Telegram integration promises that `/bead` lists active beads, where active means `open` or `in_progress`. It
gathers results by running `sase bead list` once per known project, parsing the compact rows, and combining them into
one picker.

Plain `sase bead list` is intentionally optimized for interactive CLI use: when a project has no open or in-progress
beads, it falls back to showing recent closed beads. That fallback was added after Telegram switched from an explicit
`open` filter to the default list command so it could include in-progress work. As a result, Telegram's cross-project
aggregation mixes active rows from some projects with fallback closed rows from completed projects. The generic compact
output parser accepts all valid status icons, so those closed rows reach the picker unchanged.

This is an integration contract mismatch, not a defect in the SASE CLI's interactive fallback. The fix belongs in the
`sase-telegram` linked repository.

## Implementation plan

1. Define one canonical active-bead list invocation in the Telegram inbound command path using explicit, repeatable
   status filters for both `open` and `in_progress`. Use it for all no-argument `/bead` listing modes: the configured
   project override, aggregation across known project workspaces, and the legacy/current-context fallback. Leave
   `/bead <id>` detail lookup unchanged.

2. Keep compact bead-output parsing status-agnostic. Filtering belongs at the subprocess query boundary, where the CLI
   can return `No issues found.` for a closed-only project without triggering its implicit closed fallback. This also
   preserves the parser's usefulness for valid compact output while making Telegram's requested status scope explicit
   and auditable.

3. Update Telegram inbound tests to assert that every active-picker listing path passes both explicit status filters.
   Add or strengthen a multi-project regression case in which a project has only closed beads, proving it contributes no
   picker buttons while open and in-progress beads from other projects remain visible. Preserve coverage for project
   errors, empty active results, callback routing, and project-context behavior.

4. Adjust the Telegram integration documentation to describe the explicit active-status query and why closed-only
   projects produce no picker entries, keeping user-facing behavior and implementation details aligned.

5. Install the linked repository's development dependencies if needed, run its focused bead/inbound tests during
   iteration, then run the full `just check` suite. Confirm the final diff is limited to the Telegram integration and
   contains no changes to SASE's intended `sase bead list` fallback behavior.

## Expected outcome

`/bead` will show only open and in-progress beads across all projects. Projects whose bead stores are entirely closed
will be silently absent from the picker, and if every project is closed the bot will respond with `No active beads.`
Direct bead detail lookup will continue to work for any requested bead ID.
