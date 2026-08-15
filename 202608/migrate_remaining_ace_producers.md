---
tier: tale
title: Migrate remaining durable ACE producers
goal:
  Every non-patch/non-agent ACE producer is either a supervisor-owned argv operation
  with private typed sidecars or an ordinary UI worker with no durable proc row.
size: medium
proposed_by: bbugyi200.athena.sase-m9.3.1.3
bead: sase-m9.3.1.3
create_time: 2026-08-15 16:51:28
status: wip
---

- **PROMPT:**
  [prompts/202608/migrate_remaining_ace_producers.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/migrate_remaining_ace_producers.md)
- **BEAD:**
  [sase-m9.3.1.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m9/sase-m9.3.1.3.md)

# Migrate remaining durable ACE producers

## Goal

Complete phase `sase-m9.3.1.3` by moving every non-patch/non-agent durable ACE producer
onto the supervisor-owned argv contract introduced by `sase-m9.3.1.1`, while turning
immediate UI-only work into ordinary Textual workers. Preserve current UI feedback and
completion behavior, keep sensitive or large values in mode-0600 request sidecars, and
leave no duck-typed submit lookup in this phase's production surfaces.

## Scope and constraints

- Reconcile the machine-checked producer inventory at implementation time. The current
  phase owns AXE background-command launch, monitor stop, bead and external-issue
  mutations, notification state/gate actions, plugin and SASE/agent-CLI updates,
  xprompt/config commits, and the three UI-only sites for external-issue browser open,
  prompt stashing, and commit fetching.
- Reuse focused domain commands and services. Extend existing `bead`, `notify`, `gate`,
  `launch`, `monitor`, `plugin`, `run`, `agent-cli`, `update`, and `stitch` command
  paths where their current durable result support is incomplete; do not add an ACE-only
  dispatcher or serialize/import Python callables.
- Put commands, prompts, edit bodies, option selections, plugin sets, and other
  sensitive or large values in the private operation request payload. Keep argv, labels,
  fingerprints, concurrency keys, and human logs limited to stable identifiers and
  non-sensitive summaries.
- Preserve UI-side optimistic state, targeted refreshes, toasts, selection checks,
  history updates, slot-pending rollback, and failure behavior. Completion callbacks
  consume typed result envelopes and remain only a live-session convenience.
- The patch/agent producer phase `sase-m9.3.1.2` is running concurrently. Avoid removing
  shared legacy adapter definitions until the combined production inventory proves that
  phase has migrated its callers; this phase must nevertheless remove all duck-typed
  lookup/fallback behavior from the producer files it owns and update the catalog/static
  checks so unclassified production sites cannot appear silently.
- Do not change proc observation/`ProcQueue`/`ProcMirror` ownership or public detached
  CLI semantics; those belong to later phases.

## Implementation

1. Strengthen the durable operation command surface and result contracts.
   - Add stable operation names and focused handlers for the owned mutation families,
     loading the versioned request payload only where private data is needed and
     emitting operation-specific typed payloads on every success/failure path.
   - Route handlers into the existing domain services used by the current ACE callables.
     Keep required stable identifiers positional and optional/internal I/O controls
     sorted and consistent with CLI rules.
   - Cover direct CLI behavior, request validation, typed result emission, and failure
     envelopes without treating stdout/stderr as a protocol.

2. Migrate durable utility and mutation producers.
   - Replace AXE background launch and monitor stop callables with explicit argv,
     private request payloads, stable fingerprints, and `axe-slot:<slot>` or
     `monitor:<id>` concurrency keys. Preserve slot-pending cleanup and current
     navigation/history behavior from typed completion.
   - Replace bead/artifact and external-issue mutation callables with focused durable
     commands keyed by project, bead/issue identity, and operation. Carry titles,
     bodies, labels, refs, notes, or other edit content in the request sidecar and
     retain pane refreshes.
   - Replace notification state, gate response/action, question, plan, launch-approval,
     and legacy launch callables with durable command submissions. Keep response data
     private, retain conflict/already-handled semantics, and key each request by its
     stable notification/request identity.

3. Migrate plugin, update, and commit producers.
   - Submit plugin install/update/uninstall/mode switching, SASE/dev/combined updates,
     agent-CLI updates, and comprehensive updates through their owning command paths
     with stable namespaced concurrency keys. Reconstruct the existing browser state,
     progress/history, restart notices, and refresh behavior from typed result payloads.
   - Move xprompt and config commit/post-write work to explicit `stitch`/domain argv;
     keep file contents, messages, or derived payloads out of argv and fingerprints, and
     preserve post-write cleanup and UI notifications.
   - Replace `getattr(..., "_submit_tracked_proc", ...)` and callable fallbacks in all
     owned durable producers with the typed `_submit_durable_proc` boundary.

4. Reclassify immediate UI-only work.
   - Run prompt-stash persistence, external-issue browser opening, and explicit commit
     fetching as named thread-backed Textual workers with local completion/error
     delivery and no `ProcQueue` row, durable proc request, or submit-adapter lookup.
   - Ensure worker results are applied on the Textual/UI side and that blocking browser,
     filesystem, VCS, or collection work does not run on the event loop or serial
     message pump.

5. Enforce and verify the completed phase.
   - Update the producer catalog and AST/static tests to reflect durable argv or UI-only
     worker ownership, reject callable/duck-typed submissions in owned source, and
     require every newly discovered production producer to be classified.
   - Add focused ACE tests for argv/request privacy, operation names, fingerprints,
     cross-instance concurrency keys, typed success/failure completion, optimistic
     rollback/refresh, and absence of proc rows for UI-only work. Update existing test
     doubles to the durable adapter or ordinary worker boundary.
   - Run `just install`, focused command/operation/ACE tests while iterating, then
     `just check`. Because no rendered layout is intended to change, record that PNG
     visual snapshots are not required unless implementation unexpectedly changes a
     rendered surface.

## Acceptance criteria

- Every producer assigned to this phase is either supervisor-owned explicit argv with a
  versioned request/result contract or an ordinary thread-backed UI worker with no proc
  row.
- No owned producer accepts, submits, serializes, or discovers a Python callable via a
  duck-typed proc API; the inventory/static guard fails for regressions or new
  unclassified sites.
- Sensitive and large payloads do not appear in argv, proc labels, fingerprints,
  concurrency keys, or combined logs.
- Existing optimistic UI state, targeted refresh/navigation, durable conflict behavior,
  and typed success/failure feedback remain covered by tests.
- `just check` passes after installation, and only `sase-m9.3.1.3` is closed with a
  verification note; no ancestor bead is closed.
