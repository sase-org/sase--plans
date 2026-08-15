---
tier: tale
title: Complete powerful-variable landing integration
goal:
  Published installs and detailed documentation expose the complete sase-mg variable
  contract.
size: small
proposed_by: bbugyi200.athena.sase-mg.land
bead: sase-mg
create_time: 2026-08-15 18:41:02
status: done
---

- **PARENT:** [202608/powerful_variables.md](powerful_variables.md)
- **BEAD:**
  [sase-mg](https://github.com/sase-org/sase--beads/blob/main/pages/sase-mg/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-mg.land](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-mg.land.md)
- **COMMITS:**
  - [9d9d499](https://github.com/sase-org/sase/commit/9d9d49959146740f171753547ad32145fbcb0d3e)
    — build(deps): require powerful variable core release

# Complete powerful-variable landing integration

## Context

Epic `sase-mg` added indexed output-variable history and selectors in `sase-core`, then
exposed `sase var show`, historical `sase var list`, and selector-based `sase var get`
in the Python CLI. The core work is now published as `sase-core-rs` 0.27.9, but the host
repository still declares and locks 0.27.7. That older wheel lacks the new query
contract required by this feature. The configuration guide also still describes the
former current-agent `var list` behavior even though the CLI and the concise command
reference have moved to the new contract.

The landing audit reproduced `just install` against the linked 0.27.9 checkout: the
existing override path retained the local editable extension, and the history and
selector bindings imported successfully. The separate provider-disable smoke proposal
also passes on the current tree, so neither is part of this remaining implementation.

## Implementation

1. Ratchet the `sase-core-rs` dependency minimum in `pyproject.toml` to 0.27.9,
   retaining the existing `<0.28.0` compatibility ceiling. Refresh `uv.lock` so ordinary
   published-wheel installs resolve 0.27.9 rather than 0.27.7. Keep the existing
   local-core override behavior intact.
2. Update the `sase var` section of `docs/configuration.md` to match the shipped
   interface: `show` owns current/named snapshots, `list` performs filtered historical
   discovery, `get` resolves selectors, and `set` remains the mutation command. Document
   the machine-format and major filtering options accurately enough that this detailed
   guide no longer contradicts CLI help or `docs/cli.md`.
3. Add or adjust focused assertions only where needed to prevent regression of the core
   floor and detailed command documentation. Reuse the existing core version/binding
   validators and variable CLI tests rather than duplicating domain behavior in Python.

## Verification

- Run `just install` before repository checks.
- Verify the declared published minimum and the installed binding surface with the
  existing core validation tools/tests.
- Run the focused variable CLI and documentation/source-content tests affected by the
  edits.
- Run `just check` for the completed repository changes.

Do not close `sase-mg`, run its post-close Symvision cleanup, or mark the original epic
plan done here. Those remain the parent land agent's post-handoff responsibilities.
