---
tier: tale
goal: 'Successful host-owned epic launches transition the completed planner family
  root and its planner child row from approval/review states to EPIC CREATED.

  '
create_time: 2026-07-15 20:39:12
status: wip
prompt: 202607/prompts/epic_created_status.md
---

# Plan: Mark host-launched epics as created

## Context

Epic approval now hands `sase bead work <epic_plan_file> --yes` to a host-owned tracked or detached task. While that
task is in flight, the completed planner's `epic_approved` outcome correctly renders as `EPIC APPROVED`. On success, the
task persists authoritative launch evidence (`epic_bead_id`, `epic_started_at`, the archived plan path, and
`plan_committed`) into the original agent artifacts and the TUI reloads them. The status normalizer, however, still
produces `EPIC CREATED` only for the retired `.epic` follow-up-agent shape. With no such child in the new flow, the root
remains `EPIC APPROVED` and its logical `--plan` row can fall back to `PLAN` even though the epic DAG has been created
and launched.

## Implementation

- Extend the existing plan-family status derivation to recognize a completed host-owned epic handoff from persisted
  agent metadata. Treat a non-empty epic bead id on an epic-approved plan as the successful-completion boundary: it is
  written only after `sase bead work` exits successfully and emits the required `Epic: <id>` line. Keep `EPIC APPROVED`
  while that evidence is absent, and do not infer success from task submission, plan approval, or plan archiving alone.
- Feed that semantic terminal state through the same root/planner-child normalization used by other plan-family outcomes
  so both the family root (for example `a1`) and its concrete or synthesized planner row (for example `a1--plan`) render
  `EPIC CREATED`. Preserve the legacy completed `.epic` child behavior and the existing root-mirroring rules.
- Keep the completion handoff on the current artifact-index and background reload path. Do not add synchronous I/O, a
  second refresh path, or task-callback-only UI mutation; this lets TUI, detached/headless launches, and later reloads
  derive the same status from durable metadata.

## Verification

- Add focused status-normalization regressions for the host-owned shape shown in the screenshot: an epic-approved root
  plus its main workflow planner child stays nonterminal before launch metadata arrives, then both become `EPIC CREATED`
  once a successful epic bead id is present. Cover synthesized planner children as appropriate, and retain the legacy
  `.epic` child expectations.
- Bridge the tracked-launch and loader contracts in test coverage: successful launch output must backfill the metadata
  used by status derivation, while failed or incomplete output must not falsely produce `EPIC CREATED`.
- Try a deterministic fakey-backed smoke through the real artifact/agent-loader boundary. If fakey cannot originate the
  external `sase plan propose` tool action, use it to produce the real executor artifacts and exercise the same
  post-launch metadata/reload transition, documenting that boundary rather than replacing the core regression tests with
  mocks.
- Run the focused agent-status, epic-launch, and loader tests, then run the full repository gate with `just install`
  followed by `just check`.

## Risks and constraints

- Key the terminal state to persisted successful launch evidence so an approved, queued, failed, or killed epic-launch
  task cannot be mislabeled as created.
- Keep the derivation constant-time and in-memory during refresh; all filesystem work remains in the existing
  scan/enrichment path to protect Agents-tab responsiveness.
