---
tier: tale
title: Repair the claimed-status show fixture
goal:
  The claimed-status show test exercises the routed local-view seam and the full landing
  suite passes.
size: xsmall
proposed_by: bbugyi200.athena.sase-116.5.land--1
bead: sase-116.5
create_time: 2026-09-15 16:06:00
status: wip
---

- **BEAD:**
  [sase-116.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-116/sase-116.5.md)

# Repair the claimed-status show fixture

## Context

Epic `sase-116.5` changed `bead show` to obtain its optional local view through
`sase.bead.cli_query._show_read_view()` so a command issued outside a project does not
initialize a caller bead store. Most mocked show tests were updated to patch that new
seam, but `tests/test_bead/test_claimed_status.py::test_show_explains_claim_owner` still
patches only `get_read_view`. It now fails deterministically with
`issue not found: sase-claimed`, including in the epic's full-tree landing check.

## Implementation

Update the `claimed_view` fixture to patch `_show_read_view` to the same synthetic
`_ReadView` context manager. Preserve the existing `get_read_view` patch because the
fixture is shared by list and stats coverage. Do not change production routing behavior.

## Verification

Run the previously failing test, then the complete claimed-status test module. The
resumed epic land agent remains responsible for the repository landing checks.
