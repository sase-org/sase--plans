---
create_time: 2026-07-11 14:31:05
status: done
prompt: 202607/prompts/hide_audit_chop_notifications.md
tier: tale
---
# Hide audit chop completion notifications

## Context

The scheduled `sase_recent_improvement_audit` and `sase_recent_bug_audit` chop launchers are configured in the chezmoi
SASE configuration, but they invoke the repo-local `#!sase/audit_recent_improvements` and `#!sase/audit_recent_bugs`
workflows. The workflow definitions therefore live under this repository's `xprompts/` directory; the chezmoi launcher
configuration does not need to change.

SASE suppresses a completed workflow notification when every step that actually ran is marked `hidden: true`. Both audit
workflows already hide their commit-count and child-agent launch steps, but leave their final marker-update step
visible. Once an audit is launched, that bookkeeping step runs and makes the otherwise hidden launcher workflow emit the
empty, unhelpful completion notification shown in the screenshot. The recent `refresh_docs` fix in commit `f1f5324e2`
solved the same problem by marking its analogous marker-update step hidden.

The request names the improvement audit, while the screenshot shows the bug audit. Because both workflows have the same
defect and are intended to behave symmetrically, this change should cover both audit launchers. This suppresses only the
parent chop workflow's bookkeeping notification; the independently launched audit agent and its meaningful results
remain unaffected.

## Implementation

1. In `xprompts/audit_recent_improvements.yml`, mark `update_recent_improvement_audit_marker` hidden, matching the
   existing `refresh_docs` marker-step fix. Do not change its condition, marker contents, launch behavior, or output
   contract.
2. In `xprompts/audit_recent_bugs.yml`, make the same change to `update_recent_bug_audit_marker` so the workflow
   demonstrated in the screenshot no longer emits the same pointless notification.
3. Add a focused regression assertion in `tests/test_workflow_loader_xprompts.py` that the bundled refresh-docs and
   audit launcher workflows keep every executable step hidden. This should protect the notification-suppression contract
   directly while retaining the existing validation and parent-row visibility checks.

## Validation

1. Run the focused xprompt loader tests to verify the bundled workflows parse, validate, and expose all launcher steps
   as hidden.
2. Run `just install` as required for an ephemeral SASE workspace, then run the repository-mandated `just check` suite.
3. Review the final diff to confirm it is limited to the two marker-step visibility flags and the targeted regression
   coverage, with no chezmoi or notification-engine changes.
