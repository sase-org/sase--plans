---
tier: tale
goal: 'sase-github CI and release validation use a mutually compatible SASE Python
  checkout and Rust core binding, eliminating failures caused by publication lag between
  the two repositories.

  '
create_time: 2026-07-15 13:37:36
status: wip
prompt: 202607/prompts/github_ci_core_source_alignment.md
---

# Plan: Align sase-github CI with the SASE Rust core

## Context and diagnosis

The failing `sase-org/sase-github` Actions runs began immediately after SASE's project lifecycle vocabulary moved from
`active` / `inactive` to `enabled` / `disabled`. The plugin workflow checks out `sase` from `master`, but its install
path resolves `sase-core-rs` from PyPI. Each failing run installed `sase-core-rs==0.3.4`, whose `list_project_records`
binding still rejected the new state names. The result was the same `ValueError` in twelve workspace-provider tests on
both supported Python versions; the latest plugin change was not the source of those failures.

This is a moving compatibility boundary rather than a one-off bad package resolution. Re-running after a matching core
wheel is published may turn CI green temporarily, but tracking live SASE Python source alongside the latest published
Rust wheel can fail again during the next coordinated cross-repository change. The established sibling-plugin
integration pattern avoids that window by testing the Python and Rust repositories from source together.

## Implementation

1. Update the GitHub Actions dependency setup used by `sase-github` integration checks so it checks out both
   `sase-org/sase` and `sase-org/sase-core` into dedicated dependency paths. Install the Rust toolchain and `maturin`,
   build the `sase_core_rs` binding from the checked-out core into each job's selected virtual environment, and overlay
   the editable SASE checkout without allowing a published wheel to replace that source-built binding. Preserve the
   Python 3.12 lint lane and the Python 3.12/3.13 test coverage.

2. Keep dependency setup reproducible outside Actions by consolidating any shared source-checkout installation behavior
   in the repository's task runner where it materially reduces duplicated shell logic. Use unambiguous environment names
   for the SASE Python checkout and the Rust core checkout, while retaining the ordinary published-package install path
   when those development overrides are absent.

3. Apply the same compatibility contract to release installation validation where it currently overlays a live SASE
   checkout. A release smoke test should either exercise the built plugin wheel against its declared published
   dependencies, which represents the user installation path, or deliberately build both SASE source components together
   as CI does; it must not combine live Python source with an independently published stale Rust binding. Keep release
   creation, artifact build, entry-point assertions, and trusted publishing behavior unchanged.

4. Add focused guard coverage for the installation contract where practical—for example, assertions over task-runner
   setup or workflow structure—so a future edit cannot silently restore the `sase@master` plus arbitrary PyPI core
   combination. Avoid tests that depend on network access or mutable remote branches.

## Validation

Run the repository's clean installation and full `just check` suite after the workflow/task-runner changes. Recreate the
integration environment with local SASE and sase-core checkouts, build the Rust binding into fresh Python 3.12 and 3.13
virtual environments, and run the full plugin tests in both; confirm the twelve lifecycle-related workspace tests pass
and that the installed binding reports the source-compatible lifecycle behavior. Build the plugin distribution and
repeat the release install-smoke assertions in a fresh virtual environment.

Finally, review the workflow diff for valid Actions expressions, path consistency, cache inputs, least-privilege
permissions, and absence of accidental publishing. If an Actions-level validation tool is available, run it before
relying on the next hosted CI execution.

## Risks and boundaries

Building the Rust binding adds CI time and requires a Rust toolchain, but it makes the integration lane test the
coordinated source contract it claims to cover. Source checkout paths and virtual-environment interpreter selection must
be kept explicit so `maturin` cannot install into a runner-global Python. This change is limited to dependency setup and
validation; it should not alter GitHub workspace provider behavior or weaken the package's declared SASE version floor.
