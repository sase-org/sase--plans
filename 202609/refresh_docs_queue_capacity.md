---
tier: tale
title: Give documentation-refresh agents a valid run-alone queue budget
goal:
  Every configured refresh_docs proposal uses %queue(capacity=1) while preserving its
  existing scope and dependency chain.
size: small
proposed_by: bbugyi200.athena.cm
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.cm](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.cm.md)
  - [bbugyi200.athena.sase-10h.2](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-10h.2/README.md)
  - [bbugyi200.athena.sase-zt.6.5.4.2](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-zt.6.5.4.2/README.md)
  - [bbugyi200.athena.toobig-5c.loading_disk.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-5c.loading_disk.0/README.md)
  - [bbugyi200.athena.toobig-5c.run_agent_wait_slots.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-5c.run_agent_wait_slots.0/README.md)
- **COMMITS:**
  - [78570c0](https://github.com/sase-org/sase/commit/78570c06110f05075acf44ac3d1cc847ee0caa1d)
    — fix(gate-shell): do not block gate execution on runner capacity
  - [5a280bd](https://github.com/sase-org/sase/commit/5a280bddbce83dcb65209c0a0c171150b0732064)
    — refactor(tui): split agent disk loading module
  - [66a46b8](https://github.com/sase-org/sase/commit/66a46b8cc39312b0a8c347e1b92a1f45af6d661c)
    — refactor(axe): split run_agent_wait_slots.py into candidate/state modules
  - [1dd9160](https://github.com/sase-org/sase/commit/1dd9160fdb98c187bbd09abf38f465df4345b0c1)
    — feat(ace/fleet): restore queue capacity parity for remote fleet agent rows

# Plan: Give documentation-refresh agents a valid run-alone queue budget

## Goal

Make every proposal produced by the configured `refresh_docs` lumberjack use
`%queue(capacity=1)` instead of `%queue(capacity=0)`, without changing the builtin chop,
the shared proposal scaffolder, or queue behavior for any other lumberjack.

## Current behavior and root cause

The personal Athena SASE overlay in the configured linked `chezmoi` repository sets
`axe.lumberjacks.refresh_docs.wait_runners` to `0`. The AXE runner passes that
lumberjack-level value through `process_script_chop_result()` into
`prepare_chop_proposals()`, and `scaffolded_prompt()` renders it as a canonical
`%queue(capacity=0)` directive on each proposal that does not already declare a runner
threshold. The builtin `sase_chop_refresh_docs` emits two such plain prompts—`update`
and `polish`—so every expanded target (`refresh_docs[actstat]`, `refresh_docs[bob-cli]`,
`refresh_docs[sase]`, and future enabled-project instances) inherits the invalid
zero-capacity directive.

A focused reproduction through the proposal preparation/preview API confirms that a
lumberjack value of `0` renders `%queue(capacity=0)` on both proposals, while `1`
renders `%queue(capacity=1)` on both. The latter is the documented run-alone barrier for
a default-weight launch. The existing `polish` proposal's `wait_on: update` dependency
is independent and must remain intact.

## Implementation

1. From the SASE workspace, open the configured linked `chezmoi` repository through
   `/sase_repo`; do not edit a deployed file under `~/.config` directly.
2. In `home/dot_config/sase/sase_athena.yml`, change only
   `axe.lumberjacks.refresh_docs.wait_runners` from `0` to `1`. Preserve the
   lumberjack's cadence, project fan-out, commit threshold/checkpoint policy,
   descriptions, and the separate `wait_runners: 3` setting on the maintenance lane.
3. Do not modify `sase_chop_refresh_docs`, shared AXE proposal scaffolding, the SASE
   configuration schema, or SASE documentation: they already accept positive capacity
   values and correctly translate this inherited setting into `%queue(capacity=1)`.

## Validation

Before handing the change to the host finalizer:

1. Parse the edited YAML and inspect the relevant source path to confirm the
   `refresh_docs` lumberjack has `wait_runners == 1` and no remaining zero-valued
   setting, while the maintenance lane remains unchanged.
2. Run the linked repository's applicable YAML ordering/lint check
   (`just lint-keep-sorted`) and `git diff --check`; this configuration-only edit does
   not require the unrelated Python, Lua, Markdown, or test suites.
3. Exercise SASE's proposal preparation/preview path with the configured value and two
   representative `update`/`polish` proposals. Assert that each rendered prompt contains
   exactly one `%queue(capacity=1)`, neither contains `%queue(capacity=0)`, and the
   polish prompt still contains its resolved `%wait` dependency on the update agent.
4. Review the final diff and confirm that the only source change is the intended
   one-line value update in `home/dot_config/sase/sase_athena.yml`.

## Deployment note

The `chezmoi` repository requires `chezmoi update -a --force` after its change is
committed. Let the SASE host own the commit, then apply the committed source so the live
Athena overlay also reports `wait_runners: 1`; a subsequent dry-run preview of one
configured `refresh_docs[...]` chop should show `%queue(capacity=1)` on both proposals
without launching agents.
