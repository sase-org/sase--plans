---
tier: tale
goal: Epic approval launches from the TUI and other host transports resolve the canonical
  SASE project and start bead work in the correct primary workspace, even when notification
  metadata uses a relative project directory or the repository display name differs
  from the canonical project key.
create_time: 2026-07-15 17:04:12
status: wip
prompt: 202607/prompts/epic_approval_project_identity.md
---

# Plan: Preserve canonical project identity during epic approval launches

## Context and root cause

The failed TUI task did not reach `sase bead work`. Its retained output reports that primary-workspace resolution tried
to load `/home/bryan/.sase/projects/sase/sase.sase` and then failed because no workspace plugin could identify that
nonexistent project record.

The plan-approval notification already carried the authoritative project file for `gh_sase-org__sase`, but the tracked
epic-launch path passed only `project_dir` (in this case the relative value `.`) to `resolve_epic_launch_cwd`. That
helper resolved `.` against the long-lived ACE process directory, asked the workspace provider for a repository/display
name, received `sase`, and used that value directly as though it were the canonical project-storage key. The actual
lifecycle record is keyed by `gh_sase-org__sase`; `sase` is only its effective/display identity. Existing tests mocked
workspace resolution at the TUI boundary, so they verified task streaming and ownership without exercising the lossy
identity conversion.

The shared resolver is also used by foreground and detached/headless plan approvals, so the correction should live in
the shared host-side epic-launch boundary rather than as a TUI-only special case.

## Implementation

1. Extend the shared epic-launch workspace resolver to accept the notification's canonical project-file identity in
   addition to its workspace-directory hint. Prefer the canonical project key derived from `agent_project_file`; when
   older notification producers do not provide it, retain workspace-provider and basename fallback discovery but
   normalize any discovered display name through the project-alias/lifecycle mapping before asking the running-field
   workspace API for the primary checkout. Keep the final directory existence check and actionable resolution errors.

2. Thread `agent_project_file` through every host-owned epic approval launch path:
   - the ACE tracked background task used by the TUI;
   - detached headless/mobile approval;
   - foreground CLI approval.

   Continue running the actual `sase bead work <plan> --yes` subprocess inside the tracked/background launch path, and
   preserve the existing ownership, deduplication, metadata backfill, failure toast, and manual-resume contracts.

3. Add regressions at both layers. Exercise the shared resolver with a canonical project file whose project key differs
   from the provider's repository/display name, and verify the canonical key is sent to primary-workspace resolution.
   Cover the compatibility fallback for notifications without a project file, including alias normalization. Update TUI
   and headless/foreground approval tests to assert that the canonical project-file field reaches the resolver rather
   than mocking away the argument contract. Include a failure-path assertion that missing or invalid identity still
   produces the existing recoverable behavior instead of falsely reporting a launched epic.

## Validation

- Run the focused epic-launch, TUI notification, plan-approval, CLI approval, and mobile bridge test modules that cover
  all resolver callers and launch ownership modes.
- Run `just install` as required for an ephemeral workspace, followed by the repository-mandated `just check` to cover
  formatting, lint/type checks, the full test suite, and visual snapshots.
- Re-run the focused regression after the full check if any formatting or generated adjustments touch the launch code,
  and confirm the worktree contains only the intended implementation, tests, and this approved plan.

## Risks and boundaries

The canonical project file must take precedence over the relative directory because the latter is intentionally
transport-friendly and may be interpreted in a different host process. Alias normalization remains a compatibility
fallback, not the primary identity source. The change should not alter bead DAG creation, phase/land agent rendering,
SDD plan archival, or workspace-provider behavior outside approved-epic launch resolution.
