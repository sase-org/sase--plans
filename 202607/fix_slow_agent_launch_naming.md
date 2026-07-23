---
tier: tale
title: Fix slow agent launches caused by repeated machine-identity discovery
goal: Restore prompt, stable agent naming and launch latency by resolving machine
  identity once per registry rebuild.
create_time: 2026-07-23 07:31:29
status: done
---

- **PROMPT:** [202607/prompts/fix_slow_agent_launch_naming.md](prompts/fix_slow_agent_launch_naming.md)

# Fix slow agent launches caused by repeated machine-identity discovery

## Problem

Agent launches can remain unnamed in the Agents tab, disappear during refreshes, and take minutes to reach the actual
process spawn. Launch telemetry shows that the child spawn itself remains fast, while multi-agent name planning now
spends roughly 100 seconds before spawning.

The regression is in the agent-name registry rebuild path introduced with machine-qualified agent hoods. Creating a
planned artifact directory changes the registry source signature, so the next name allocation rebuilds the registry
while holding the global allocation lock. During that rebuild, `MachineHoodIdentity.current()` is called once for every
artifact and dismissed bundle record. Each call repeats configuration discovery. With roughly 4,500 registry entries,
the current rebuild takes about 88 seconds; resolving one identity snapshot for the same scan reduces the collection
work to about one second.

## Scope

Keep this as a Python-side registry-scan correction. The machine-hood facade already models an identity as a loaded
snapshot, so no Rust core API or TUI presentation change is needed. Preserve all current qualification behavior for
local, machine-qualified, foreign, explicit, and legacy auto-prefixed agent names.

## Implementation

1. Update `src/sase/agent/names/_registry.py` so `rebuild_name_registry()` resolves `MachineHoodIdentity.current()`
   exactly once for a rebuild and passes that immutable snapshot to both active-artifact and dismissed-bundle
   collection.
2. Update `src/sase/agent/names/_registry_scan.py` so the collection helpers and `_add_owner_names()` consume the
   supplied identity instead of performing per-record identity discovery. Keep direct helper usage compatible where
   useful, but ensure the production rebuild path shares one snapshot across every source.
3. Add deterministic regression coverage in `tests/test_agent_name_registry.py`:
   - Build a registry from both active artifacts and dismissed bundles.
   - Patch or instrument `MachineHoodIdentity.current()` and assert that a complete rebuild resolves identity only once,
     independent of the number and kinds of records scanned.
   - Assert the rebuilt registry still preserves the expected configured-machine qualification, legacy auto-prefix
     handling, and foreign-name behavior. Avoid wall-clock assertions; the identity-resolution call count is the stable
     complexity invariant that prevents this regression.

## Validation

1. Run `just install` before repository checks, as required for an ephemeral workspace.
2. Run the focused registry and machine-hood tests, including the new regression case.
3. Re-run a controlled registry rebuild against a representative large registry and confirm collection time returns from
   tens of seconds to approximately the sub-second/low-single-second range without changing the resulting name set.
4. Exercise a multi-model fanout launch and inspect launch timing telemetry: name planning should no longer dominate
   launch time, child processes should spawn promptly, and Agents-tab rows should receive stable names without the prior
   disappear/reappear cycle.
5. Run `just check` and resolve every failure before completion.

## Acceptance criteria

- A registry rebuild performs a constant number of machine-identity resolutions: exactly one in the production rebuild
  path, rather than one per scanned record.
- Active artifacts and dismissed bundles are processed with the same identity snapshot.
- Existing local, qualified, foreign, explicit, and legacy name semantics remain unchanged.
- Large-registry rebuild and fanout name-planning latency no longer account for minute-scale launch delays.
- The focused tests and full `just check` suite pass.
