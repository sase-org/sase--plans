---
tier: tale
goal: Make local SASE installation reliably select the compatible workspace-local
  sase-core checkout when linked-repository environment variables are unavailable.
create_time: 2026-07-15 08:22:27
status: done
prompt: 202607/prompts/workspace_local_core_fallback.md
---

# Plan: Prefer the Workspace-Local Rust Core Checkout

## Context and root cause

`just install` intentionally builds a local `sase_core_rs` extension whenever it finds a Rust core source checkout. The
current SASE dependency and lockfile require `sase-core-rs>=0.4.0,<0.5.0` because the plan-validation command calls the
new `plan_validate` and `plan_frontmatter_schema` bindings. Lowering that constraint would therefore hide a real API
incompatibility rather than repair the installation.

SASE-launched agents receive `SASE_LINKED_REPO_SASE_CORE_DIR`, and their workspace-local linked checkout is already at
the compatible `sase-core` v0.4.0 release. A shell without SASE's injected linked-repository variables instead makes the
Justfile fall back directly to `../sase-core`. In a numbered workspace that path is not the workspace-matched linked
checkout; it can refer to an older shared checkout, which is why the version guard correctly rejects the observed v0.3.4
source before Maturin runs.

## Implementation

Update the Justfile's environment-free core-directory resolution so it recognizes the conventional workspace-local
linked checkout under `sase/repos/linked/sase-core` before using the legacy `../sase-core` fallback. Keep the existing
precedence intact for the explicit `SASE_CORE_DIR` override, the canonical linked-repository variable, and both legacy
sibling-repository variables. Retain the sibling fallback when no workspace-local linked checkout is present so the
primary development checkout and older layouts continue to work, and update the nearby documentation to explain the two
fallback layouts.

Extend the Justfile resolution tests to isolate the filesystem layout rather than relying on whichever linked checkout
happens to exist beside the test process. Cover both environment-free cases: a numbered-workspace-style local linked
checkout wins when present, while an absent local checkout preserves the sibling fallback. Continue exercising the full
explicit/canonical/legacy environment precedence so the compatibility contract cannot regress.

## Validation

Run the focused Justfile resolution tests first and inspect the evaluated core path with all related environment
variables removed. Confirm that the exact source-version validator accepts the selected v0.4.0 checkout. Then run
`just install` before the repository-required `just check`, followed by the user's original `just all` workflow, and
recheck the worktree so formatting or generated-file side effects are understood. The installation is fixed when the
source version guard passes, the local extension builds from the workspace-matched checkout, all checks and tests pass,
and explicit environment overrides still resolve in their documented order.
