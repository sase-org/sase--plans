---
tier: tale
title: Restore sase-telegram CI after EpicResume removal
goal:
  Make sase-telegram pass against current SASE sources while retaining coverage for
  every supported generic notification gate.
size: small
proposed_by: bbugyi200.athena.0ag
create_time: 2026-08-22 11:18:03
status: wip
---

# Restore sase-telegram CI after EpicResume removal

## Goal

Make the `sase-telegram` GitHub Actions matrix pass against the live `sase` and
`sase-core` master branches by removing stale test-only coverage for the deleted
EpicResume notification gate, without weakening coverage for any gate kind that remains
registered.

## Diagnosis

The failing release commit is `e70410e` (`v0.4.8`). GitHub Actions run `32525817396`
fails while collecting `tests/test_custom_gates.py` on Python 3.12:

```text
ModuleNotFoundError: No module named 'sase.bead.epic_resume_gate'
```

The Python 3.13 matrix leg is cancelled by fail-fast after the 3.12 failure; it does not
expose a distinct failure.

The CI workflow deliberately checks out and installs the current `sase` master branch.
Upstream `sase` commit `5938b6dce` is a breaking change that removes the automatic
EpicResume chop, the `EpicResume` notification-gate kind, and
`sase.bead.epic_resume_gate`. The `sase-telegram` runtime source has no reference to
that deleted API. The only stale references are an import, a fixture helper, and gate
construction inside the registry-driven generic-form test.

## Implementation

1. Update `tests/test_custom_gates.py` to remove the deleted `create_epic_resume_gate`
   import and the now-unused EpicResume member fixture.
2. Remove construction of the obsolete EpicResume gate and its notification-map entry
   from `test_registry_declared_generic_forms_render_keyboards`.
3. Preserve the test's registry-driven iteration. It must still render and assert a
   keyboard for every gate whose current adapter declares `generic_form`; the fixture
   map should contain every such currently registered kind.
4. Do not pin `sase` to an older release or recreate compatibility code for the removed
   gate. CI is intentionally an integration check against the live SASE sources, and
   EpicResume no longer exists upstream.

## Validation

1. Install the development environment through the repository's `just install` workflow
   so the local SASE and Rust-core bindings match CI.
2. Run the targeted registry test in `tests/test_custom_gates.py` to verify that all
   current generic gate forms still render.
3. Run `just check` to exercise lint, type checks, and the complete test suite on the
   repository's default Python environment.
4. Re-run the targeted test after the full check, confirming that environment or
   formatting steps did not mask the original collection failure.
5. Inspect the final diff and worktree status to ensure the patch is limited to the
   stale EpicResume test fixture and contains no generated or environment artifacts.

## Expected result

`tests/test_custom_gates.py` collects successfully with current `sase`, the
registry-driven generic-form coverage remains intact for supported gates, and the same
test command used by both GitHub Actions matrix legs can complete.
