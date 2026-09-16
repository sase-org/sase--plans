---
tier: tale
title: Add the sase agent hold command group
goal:
  Durable agent holds can be created, inspected, released, and wrapped around commands
  through a safe CLI.
size: medium
proposed_by: bbugyi200.athena.sase-11l.4
bead: sase-11l.4
create_time: 2026-09-16 10:49:36
status: wip
---

- **PARENT:**
  [202609/hold_directive.md](https://github.com/sase-org/sase--plans/blob/main/202609/hold_directive.md)
- **BEAD:**
  [sase-11l.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11l/sase-11l.4.md)

# Plan: add the `sase agent hold` command group

## Goal

Expose the already-landed durable hold store and runner-slot blocker through a safe,
watchable CLI. The command group will support `create`, `list`, `release`, `run`, and
`show`, default a bare group to `list`, freeze `pending` targets at arm time, enforce
the configured TTL cap, identify agent and standalone CLI armers correctly, and emit
deduplicated lifecycle notifications.

## Implementation

1. Add `agent_hold_default_ttl` and `agent_hold_max_ttl` to the bundled configuration
   and schema, with validated accessors that fail safely to the approved two-hour and
   twelve-hour defaults. Reuse the existing duration parser for CLI TTL input, reject
   zero/invalid values, and reject values above the configured maximum.
2. Extend the Python hold facade into the shared service used by this CLI and the later
   `%hold` directive. Build armer wire payloads from current agent metadata when
   `SASE_ARTIFACTS_DIR` is present and from the current process otherwise; build project
   or host scope payloads; normalize names and `@tribe` selectors; snapshot WAITING and
   QUEUED artifact directories for `pending`; call the Rust bindings for arm/list and
   release; and summarize captured WAITING, QUEUED, and skipped RUNNING rows.
3. Add lifecycle notifications at the shared service boundary. Upsert one notification
   when a hold is armed and one when an explicit or liveness-driven release is observed,
   using stable per-armer deduplication keys and capture summaries. Keep store reads and
   admission fail-open while surfacing direct CLI errors clearly.
4. Register an alphabetically ordered `sase agent hold` nested command group and route
   it through the agent handler. Give every public option a short alias. Implement:
   `create` with names, tribes, hoods, future, pending, scope, and TTL; `list` with a
   colored table and stable JSON; `release` with an optional armer key that defaults to
   the current agent's hold; `show` with full selector and frozen artifact-dir detail;
   and `run`, which arms around an argv command, preserves the command's exit status,
   and releases in `finally` on success, failure, or interruption. For the documented
   selector-free quiesce recipe, `run` defaults to `pending` plus `future`; `create`
   continues to require at least one explicit selector.
5. Add focused tests for parser/help ordering and aliases; config defaults and caps;
   identity, selector normalization, pending snapshots, scope, JSON and rich output;
   create/list/show/release behavior; arm/run/release including nonzero command exit;
   notifications; and an admission integration proving a held waiter parks and resumes
   after release. Preserve existing agent-list and runner-slot behavior.

## Verification

Run focused hold CLI, config, parser, and runner-admission tests while iterating. Then
run `just fix`, `just check`, `sase bead epic-symbols sase-11l.4`, resolve or re-key any
remaining phase symbols, and close only `sase-11l.4` with a note naming the verified
commands and behavior.
