---
tier: tale
size: small
title: Require the published directive-completion core bindings
goal:
  Ensure ordinary sase installs resolve a sase-core-rs release that provides the shared
  directive-completion contract used by ACE.
proposed_by: bbugyi200.athena.sase-rj.land
bead: sase-rj
create_time: 2026-08-21 05:48:36
status: wip
---

- **PARENT:**
  [202608/xprompt_directive_completion_parity.md](xprompt_directive_completion_parity.md)
- **BEAD:**
  [sase-rj](https://github.com/sase-org/sase--beads/blob/main/pages/sase-rj/README.md)

# Require the published directive-completion core bindings

## Context

Epic `sase-rj` added `directive_contract`, `directive_completion_context`, and
`directive_completion_candidates` to `sase-core` and made ACE call those bindings at
runtime. The bindings are published in `sase-core-rs` 0.29.6, as recorded by the 0.29.6
core release changelog. The primary `sase` repository still declares
`sase-core-rs>=0.29.5,<0.30.0`, and `uv.lock` still resolves 0.29.5. A normal published
install can therefore satisfy the declared dependency with a wheel that lacks the API
now required by ACE; editable development installs hide the problem by building the
linked `sase-core` checkout.

## Implementation

1. Raise the `sase-core-rs` lower bound in `pyproject.toml` from 0.29.5 to 0.29.6,
   preserving the existing `<0.30.0` upper bound.
2. Refresh only the affected dependency metadata in `uv.lock` and confirm the locked
   `sase-core-rs` package is the published 0.29.6 release on every recorded platform.
3. Install the workspace dependencies, then exercise the repository's core-floor probe
   so it verifies the declared published floor exports all three directive-completion
   bindings rather than relying on the linked checkout.
4. Run the focused runtime/ACE/LSP directive parity test and the repository-required
   `just check`. If the independently owned `sase-ri` closed-flag inconsistency still
   stops the whole-repo lint lane, record that known external blocker while preserving
   successful dependency-floor and directive-parity evidence.

## Acceptance criteria

- `pyproject.toml` cannot resolve `sase-core-rs` 0.29.5 for a normal install.
- `uv.lock` resolves `sase-core-rs` 0.29.6 and records the matching 0.29.6 artifacts.
- The published-floor validation confirms `directive_contract`,
  `directive_completion_context`, and `directive_completion_candidates` are present.
- `tests/test_xprompt_directive_completion_parity.py` passes against the refreshed
  environment.
- No source, documentation, or completion behavior is changed beyond the dependency
  floor needed to make the already-landed shared contract installable.
