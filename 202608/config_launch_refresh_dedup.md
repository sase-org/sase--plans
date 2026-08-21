---
tier: tale
size: small
title: Make Config Launch mutation refreshes exactly once
goal:
  Each successful embedded Launch mutation refreshes the affected top-bar indicators
  exactly once, including provider-routing writes, without replaying the accumulated
  change result when nested modals or the Admin Center close.
proposed_by: bbugyi200.athena.sase-rp.land
bead: sase-rp
create_time: 2026-08-21 08:41:22
status: wip
---

- **PARENT:** [202608/admin_center_launch.md](admin_center_launch.md)
- **BEAD:**
  [sase-rp](https://github.com/sase-org/sase--beads/blob/main/pages/sase-rp/README.md)

# Plan: Make Config Launch mutation refreshes exactly once

## Context

Epic `sase-rp` moved Launch settings into the Admin Center and added live indicator
refreshes through `LaunchPane._mark_changed()` and `ConfigHubPane.on_launch_changed()`.
Landing verification found two duplicate notification paths:

- `LaunchPane._mark_changed()` notifies the Config host immediately, but
  `ConfigHubPane.request_launch_close()` feeds the same accumulated changed result back
  through `on_launch_changed()` before closing.
- Every successful provider-routing write emits a snapshot to
  `ModelsPanelProvidersMixin._on_provider_modal_snapshot()`, which calls
  `_mark_changed()`, and the provider modal's dismissal callback calls `_mark_changed()`
  again for the same write.

A direct contract reproduction currently records two host change notifications for one
embedded mutation followed by close. The existing focused Launch/Config suite passes
because it tests change delivery and indicator refresh independently, not the combined
mutation-to-close lifecycle.

The epic's temporary flag has already been removed and its removal bead `sase-rq` is
closed. The unrelated Symvision private-import baseline is recorded on active epic
`sase-rm`, and repository-wide visual drift is recorded on `sase-r5` / `sase-rm.13`;
neither belongs in this tale.

## Implementation

1. Make an embedded successful mutation the sole point that notifies the Config host of
   indicator changes. Closing `ConfigHubPane` must consume the Launch host close request
   without replaying an already-delivered accumulated result. Preserve the host protocol
   and the standalone `ModelsPanelResult` dismissal contract.
2. Make the provider modal's successful snapshot callback the sole mutation signal for
   provider-routing writes. Do not mark the same write again on modal dismissal; retain
   changed-result accumulation so a standalone `ModelsPanel` still dismisses with
   `provider_routing_changed=True`.
3. Add focused regression coverage that exercises the complete sequences:
   - embedded `_mark_changed()` followed by Launch/Admin Center close refreshes the app
     indicators once, not twice;
   - a provider write snapshot followed by provider-modal dismissal notifies the host
     once while preserving the panel's changed result;
   - unchanged close/dismiss paths remain silent, and existing busy-write close guards
     continue to hold.

## Verification

- Run the focused Launch pane, Config hub, provider-routing, leader-routing, and three
  indicator test modules.
- Run the two Config Launch PNG snapshot nodes to ensure the lifecycle-only fix causes
  no visual drift.
- Run `tools/check_feature_flags` under the workspace virtual environment and confirm it
  is clean.
- Run `just check` as the repository-required post-change gate. If it stops at the
  already-recorded `sase-rm` Symvision private-import baseline, confirm all earlier
  gates passed and that no finding names a file changed by this tale.

## Acceptance criteria

- One successful embedded Launch mutation causes exactly one app-level indicator refresh
  notification, even after a provider modal and the Admin Center subsequently close.
- Provider-routing changes still invalidate and refresh the launch-default/provider
  indicators and remain represented in `ModelsPanelResult`.
- Standalone compatibility behavior, busy-write safety, focused functional tests, Config
  Launch snapshots, and feature-flag integrity remain intact.
