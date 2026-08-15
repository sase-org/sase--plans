---
tier: tale
title: Prevent tribe persistence from leaking an artifact directory
goal:
  Tribe-only persistence updates the canonical store without creating agent-tribes or
  any other synthetic artifact directory.
size: small
proposed_by: bbugyi200.athena.02z
create_time: 2026-08-15 19:39:40
status: wip
---

# Prevent tribe-only persistence from creating `agent-tribes/`

## Problem and diagnosis

The suspicion is confirmed. Several SASE workspaces contain an untracked `agent-tribes/`
directory whose only entry is the zero-byte `.agent_directive_persistence.lock` file.
The regression came from the durable-argv migration in commit `0835b38d2`: ACE's
tribe-assignment action supplies the relative string `agent-tribes` when a standalone
agent has no artifacts directory, so the new `sase agent persist-directive` transport
has a positional value. The worker then builds an `AgentDirectivePersistenceSpec` with
that value.

The actual filesystem mutation is in
`src/sase/ace/tui/actions/agents/_directive_persistence.py`.
`persist_agent_directive_update()` currently acquires `_agent_directive_lock()` whenever
the spec has any non-`None` `artifacts_dir`; the lock helper creates that directory
before opening `.agent_directive_persistence.lock`. A standalone tribe assignment only
updates the canonical `agent_tribes.json` store and does not touch prompt, metadata,
waiting, or ready artifacts. The tribe store already serializes its own
read-modify-write cycle with `_agent_tribes_file_lock()`, so creating an artifact
directory and taking the artifact lock is both unnecessary and the source of the leak.
The synchronous test double in `tests/ace/tui/test_agent_tribe_assignment.py` exercises
the same production persistence function, which explains why test runs leave the
directory behind; the real ACE durable command can do so as well.

## Implementation

1. In `persist_agent_directive_update()`, distinguish artifact-backed mutations from
   store-only mutations. Acquire `_agent_directive_lock()` only when at least one field
   that can read or write an agent artifact is present: `prompt_mutator`, `meta_patch`,
   `waiting_marker`, or `ready_marker`. Keep the existing per-artifact lock behavior
   unchanged for those operations, including mixed artifact-plus-tribe updates. Allow a
   tribe-only update to proceed directly to the canonical tribe-store helper, whose own
   lock remains responsible for cross-process serialization.

2. Add focused regression coverage in
   `tests/ace/tui/actions/test_agent_directive_persistence.py` around the existing
   set/unset tribe-store test. Give the spec a nonexistent sentinel artifact path,
   verify the tribe assignment is still persisted and cleared, and assert that neither
   the sentinel directory nor `.agent_directive_persistence.lock` is created. Preserve
   the existing artifact-backed tests so they continue proving prompt, metadata, and
   marker updates work with their lock path.

3. Add or strengthen an ACE tribe-assignment integration assertion in
   `tests/ace/tui/test_agent_tribe_assignment.py` using an isolated temporary current
   directory. Exercise a standalone agent with no artifacts directory through the
   durable test double and assert that the canonical tribe store changes without an
   `agent-tribes/` directory appearing. This protects the exact caller/transport path
   that caused the observed workspace residue.

Do not hide `agent-tribes/` in `.gitignore` and do not add post-test cleanup: those
would mask production creation instead of enforcing the correct persistence boundary.

## Validation

1. Run the focused regression tests:

   ```bash
   pytest -q tests/ace/tui/actions/test_agent_directive_persistence.py tests/ace/tui/test_agent_tribe_assignment.py
   ```

2. Confirm the focused run leaves no `agent-tribes/` directory in the repository root.

3. Because repository files changed, first run `just install`, then run the required
   `just check`. If scoped selection escalates, reports unusual coverage, or the change
   enters the broadening set, run `just check-full` through `/sase_monitor` with a
   concrete `--next` follow-up action, as required by the repository instructions.
