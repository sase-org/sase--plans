---
tier: tale
goal:
  Ensure the standard SASE Python setup cannot silently run sase-63 against a stale
  published Rust binding, validate the complete notification-kill behavior, and finish
  landing epic sase-63.
create_time: 2026-09-09 19:53:22
status: wip
---

# Plan: Integrate and land sase-63 across the Python/Rust binding boundary

## Context

Epic `sase-63` added root-aware `UserQuestion` and `PlanApproval` identity matching in
the linked `sase-core` checkout, then made successful named-agent kills use that
Rust-backed notification update. The Rust workspace is clean and passes formatting,
clippy, and its full test suite. The current Python and ACE code also routes the CLI,
mobile, stale-agent, and TUI cleanup surfaces through the intended APIs.

Landing verification exposed a cross-repository setup gap. After dependency
synchronization restores the published `sase-core-rs` 0.4.0 wheel,
`tools/validate_sase_core_rs` accepts it because its probes cover older contracts but
not the new root-aware notification behavior. Consequently, the normal `_setup` recipe
skips rebuilding the linked Rust checkout, and the three new end-to-end regressions
fail: live named-root cleanup, stale named-root cleanup, and ACE root-only cleanup all
leave the child-routed question active. This makes the epic incomplete even though both
child beads are closed.

The only non-epic source commit made after this epic began was the documentation-only
sase-62 change to `docs/sdd.md` and `docs/sdd_storage.md`; it neither conflicts with nor
consumes notification cleanup. The integration work is the missing local-binding
compatibility sentinel described below.

## Implementation

Extend `tools/validate_sase_core_rs` with a focused behavioral probe for the existing
`apply_notification_state_update` binding. In a temporary notification store, write a
`UserQuestion` whose child and root timestamps differ, apply `dismiss_matching_agents`
using the root `(cl_name, raw_suffix)` identity, and require a well-shaped outcome that
reports and returns the row as dismissed. Add the binding to the validator's required
binding set if it is not already present. Keep the probe independent of SASE's
higher-level Python notification modules so the bootstrap validator remains usable while
`_setup` is deciding whether the installed extension is current.

Add focused unit coverage in `tests/test_validate_sase_core_rs_tool.py` for all relevant
outcomes: a compatible fake binding passes, a binding with pre-sase-63 child-only
matching behavior is rejected, malformed or exceptional binding results fail with a
useful validator diagnostic, and the required-binding audit includes the notification
update entry. Keep temporary files isolated and avoid weakening any existing validator
probes.

## Validation and revalidation

First reproduce the stale-wheel state and confirm the new validator rejects it. Then
exercise the normal Justfile setup path and confirm it rebuilds `sase_core_rs` from the
workspace-linked `sase-core` checkout. Re-run the validator to prove the rebuilt
extension is accepted.

Run the focused validator-tool tests and the notification-kill regressions in
`tests/notification_store/test_storage.py`, `tests/test_kill_named_agent_dismiss.py`,
and `tests/test_agent_kill_single.py`. Re-run Rust formatting, clippy, and the full Rust
workspace tests if the linked core checkout changes; otherwise preserve the
already-verified clean Rust commit. Finally run `just install` followed by the required
full `just check`, and re-run the focused Python regressions after the full gate to
catch dependency or backend selection drift.

## Final landing phase

Only after every validation above passes, recheck `sase bead show sase-63` and both
child beads and verify that no new post-epic source commit needs integration. Close the
epic with `sase bead close sase-63`. After closing, use the project's audited Symvision
memory workflow before fixing any reported issue, run `just symvision` if that recipe is
available, and remove only stale `sase-63` whitelist entries or genuinely unused code it
reports. Re-run `just check` after any source/configuration cleanup.

Lastly, change the linked epic plan `202607/question_notification_kill_cleanup.md` from
`status: wip` to `status: done`, preserving its approved content. Verify the epic is
closed, the plan frontmatter is done, all affected repositories are clean except for the
intended landing changes, and report the exact validation results. Do not commit unless
the user explicitly requests a commit.
