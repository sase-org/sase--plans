---
tier: tale
goal: Prevent the toobig_split chop from scanning or launching while an existing split_file-hood
  agent is starting, running, or waiting.
create_time: 2026-07-15 16:04:37
status: done
prompt: 202607/prompts/toobig_split_active_hood_guard.md
---

# Plan: Guard toobig split launches by active agent hood

## Context and root cause

The suspicion is confirmed. The chezmoi-managed `toobig_split` executable is configured as a script chop, so AXE's
agent-chop registry deduplication does not apply to it. Its current repository-scoped `flock` prevents only overlapping
executions of the scanner process; the lock is released as soon as the detached `sase run` command returns and does not
represent the lifetime of the launched agents. The `%w(runners=0)` directive in each generated prompt controls when a
new agent may consume a runner slot, but it does not stop a later chop tick from creating another batch. Consequently,
`run_locked()` proceeds directly to `toobig` discovery and scanning without consulting SASE agent state, even when an
earlier `split_file.<name>` agent is already `RUNNING` or `WAITING`.

SASE already exposes the necessary cross-project state through the stable JSON schema of `sase agent list -j`, including
agent `name` and `status`, so this does not require a new SASE CLI option or shared Rust-core behavior. The generated
names observed in durable artifacts use the expected `split_file.<slug>-<id>` form. For this guard, the `split_file`
hood consists of the exact root name `split_file` and every dotted descendant whose name starts with `split_file.`;
lookalikes such as `split_file_backup` are unrelated. Although the requested steady states are `RUNNING` and `WAITING`,
the guard must also treat `STARTING` as active because SASE reports a live claimed agent as `STARTING` before its run
timestamp is written. Omitting that pre-run state would leave a real duplicate-launch window.

## Implementation

1. In the linked chezmoi repository's `home/bin/executable_sase_chop_toobig_split`, add a small, explicit agent-state
   preflight that invokes `sase agent list -j`, validates the JSON response, and identifies blockers by the hood and
   active-state rules above. Keep the integration at the public CLI boundary rather than importing SASE internals into
   the standalone chop script.
2. Run the preflight while holding the existing repository lock but before locating or invoking `toobig`. When a blocker
   exists, return a successful no-op with a structured `launched=0` reason and bounded blocker details so scheduled runs
   remain healthy and diagnosable without scanning or calling `sase run`. Preserve normal scanning, prompt construction,
   wait chaining, and launch behavior when only unrelated or terminal agents are present.
3. Fail closed if the agent-list command fails or returns invalid/unexpected JSON: report a compact diagnostic, return
   nonzero, and do not scan or launch. An unavailable occupancy check must not be interpreted as an empty hood and
   create duplicate work.
4. Extend `tests/bash/toobig_split_chop_test.sh` so its fake `sase` implementation serves configurable agent-list JSON
   and failures. Add regression cases proving that `RUNNING`, `WAITING`, and `STARTING` members of the hood abort before
   scanner execution; the exact root and dotted-name boundary are recognized; unrelated names and terminal statuses do
   not block a valid launch; and command, malformed-JSON, or schema failures are visible, nonzero, and launch-free.
   Retain the existing repo-lock and multi-prompt assertions to ensure the new preflight does not weaken concurrency or
   sequencing behavior.

## Validation

- Prepare the linked chezmoi checkout's development environment, then run the focused `toobig_split_chop_test.sh`
  bashunit suite while iterating.
- Re-run the focused suite after reviewing the final diff, including assertions that blocker paths make no `toobig` or
  `sase run` call and that the normal path performs exactly one agent-state query before scanning.
- Run the chezmoi repository's full `just check` gate before completion.
- No source change is expected in the SASE repository, its xprompt renderer, the Athena chop configuration, or the Rust
  core: the existing public agent-list contract is sufficient, and the defect is local to the standalone script chop's
  missing preflight.
