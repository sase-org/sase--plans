---
tier: tale
title: Install local SASE for sase-telegram development
goal:
  Fresh sase-telegram development environments use the workspace-matched editable SASE
  checkout and pass all checks.
size: medium
proposed_by: bbugyi200.athena.sase-cj
create_time: 2026-08-10 09:33:06
status: wip
---

# Install the local SASE checkout for sase-telegram development

## Objective

Make `just install` and the automatic `_setup` path in the linked `sase-telegram`
repository install the workspace-matched local `sase` source checkout in editable mode,
so development and tests use current APIs instead of whichever older `sase` release is
available from PyPI.

## Implementation

1. Add a documented Justfile variable for the local `sase` source directory. Support an
   explicit environment override and deterministic defaults for SASE linked workspaces,
   sibling development checkouts, and the CI dependency checkout.
2. Refactor the setup/install recipes so the project and development dependencies are
   installed into the repository venv and then the discovered local `sase` checkout is
   installed editable with `--no-deps`, matching CI's dependency-resolution order.
   Missing or invalid local source must fail with an actionable error instead of
   silently retaining a PyPI package.
3. Add focused regression coverage for Justfile path selection and command behavior,
   including override precedence and the workspace default.

## Verification

1. Start from a fresh `sase-telegram` virtual environment and run `just install`.
2. Verify from that venv that `sase` imports from the intended local source checkout and
   that APIs previously absent from the PyPI release are importable.
3. Run `just check` in `sase-telegram` and confirm lint, type checking, and the full
   test suite pass.
4. Review the final diff and repository status to ensure only task-scoped files changed
   and no commit was created.
