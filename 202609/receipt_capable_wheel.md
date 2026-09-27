---
tier: epic
title: Publish the receipt-capable core wheel and raise the sase floor
goal: Unblock the sase-core release that contains the receipt report binding, publish
  a complete sase-core-rs wheel, and raise sase's declared floor so a fresh wheel
  install accepts the catalog receipt policy.
parent_bead: sase-1ah.8
phases:
- id: unblock-core-release
  title: Let release-plz select the gateway across a minor bump
  depends_on: []
  size: small
  description: 'unblock-core-release: drop the caret pin on the unpublished sase_gateway
    workspace path dependency so release-plz can cut the 0.35.0 release that contains
    the receipt binding.'
- id: ratchet-receipt-wheel
  title: Publish the wheel and raise the sase floor
  depends_on:
  - unblock-core-release
  size: medium
  description: 'ratchet-receipt-wheel: cut the urgent sase-core release, confirm a
    complete receipt-capable sase-core-rs wheel, and raise sase''s floor with a fresh-install
    proof.'
proposed_by: bbugyi200.athena.sase-1ah.8.land
create_time: 2026-09-26 18:37:43
status: done
bead_id: sase-1ah.8.4
---

- **PROMPT:** [prompts/202609/receipt_capable_wheel.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/receipt_capable_wheel.md)
- **PARENT:** [202609/e4_landing_remainder.md](https://github.com/sase-org/sase--plans/blob/main/202609/e4_landing_remainder.md)
- **BEAD:** [sase-1ah.8.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ah/sase-1ah.8.4.md)

# Publish the receipt-capable core wheel and raise the sase floor

Parent epic `sase-1ah.8` already landed the Rust opportunity report and the Python
adapter. This child only finishes the published-wheel requirement that phase
`sase-1ah.8.2` deferred. It does not close `sase-1ah.8`, retire symbols, or edit that
epic's plan status. The `parent_bead` link returns those decisions to the parent land
agent.

## What is already done

Do not redo this work.

- `sase-core` commit `0cf5147` adds `tool_run_receipts_report` in
  `crates/sase_core/src/tool_run/store/receipts_report.rs`, with the PyO3 binding
  `tool_run_receipts_report`. Later core commits did not edit that file. Source pin
  `sase-core-revision.txt` is `e44af7d`, which contains `0cf5147`, receipt contract
  `9f86897`, and no-new accept `e654e7c`.
- sase commit `f7886b1a6` (`sase-1ah.8.2`) makes `src/sase/tool/receipt_report.py` a
  thin CLI adapter over `sase.core.tool_run.tool_run_receipts_report`. Commits after it
  do not touch the receipt modules.
- `sase/sase.yml` already publishes `receipt:` on the `check` tool. An older core
  rejects that field. Dev installs build the pin from the linked checkout, so they hide
  the gap. A fresh wheel install does not.

## Why the floor is still `>=0.34.71,<0.35.0`

PyPI's newest `sase-core-rs` is `0.34.73` (tag `v0.34.73`, commit `e4ddcb48`), which
does not contain `tool_run_receipts_report`. No newer tag exists.

`release-plz` on `sase-org/sase-core` master has failed on every run since the breaking
commits after that tag. Run `36272464168` (2026-09-26) is the latest example. It decides
the next `sase_core` / `sase_core_py` version is `0.35.0` because of pre-existing
breaking commits, including `2a0fc2a` (`feat(core)!`), `cfe1902` (`feat!`), and
`44dbc91` (`feat!`). Those commits are not part of `sase-1ah.8`. The receipt commits are
additive.

`cargo update` then fails:

```text
failed to select a version for the requirement `sase_gateway = "^0.34.0"`
candidate versions found which didn't match: 0.35.0
required by package `sase_core_py v0.35.0`
```

Cause, in the workspace `Cargo.toml`:

- `[workspace.package] version` is `0.34.73`.
- `sase_gateway` the package uses `version.workspace = true`, so a release-plz bump of
  the workspace version moves the package to `0.35.0`.
- `[workspace.dependencies] sase_gateway` is
  `{ path = "crates/sase_gateway", version = "0.34.0" }`. Cargo treats that as
  `^0.34.0`, which excludes `0.35.0`.
- `release-plz.toml` sets `sase_gateway` to `release = false`, so release-plz rewrites
  the released `sase_core` requirement and does not rewrite this one.
- `sase_core_py` depends on `sase_gateway` with `workspace = true`.

Open release PR `sase-org/sase-core#313` (`chore: release v0.34.74`) is stale.
release-plz can no longer update it while this selection error persists. Dispatching
`release-plz.yml` before the dependency fix fails the same way. Do not hand-edit
`[workspace.package] version`, crate versions, or any `CHANGELOG.md`. release-plz owns
those. The only manifest edit is the gateway requirement below.

Standing task `sase-10d` already tracks "cut a core release and ratchet the sase floor"
for earlier unpublished commits, most recently `f55c63b`. A release of current master
contains that commit and the receipt binding. Leave `sase-10d` open: it also covers
publishing a `sase` distribution to PyPI, which this plan does not do. Note the
published core version on `sase-10d` when phase 2 finishes.

## Phase 1: Let release-plz select the gateway across a minor bump

Open the linked `sase-core` repo with `sase repo open sase-core` and follow its
`AGENTS.md`. In the workspace `Cargo.toml`, change the `sase_gateway` workspace
dependency from a caret `0.34.0` requirement to a path-only dependency:

```toml
sase_gateway = { path = "crates/sase_gateway" }
```

`cargo metadata --offline --no-deps` then reports `sase_core_py`'s `sase_gateway`
requirement as `*` while the `sase_gateway` package version stays equal to
`[workspace.package] version`. That `*` matches both the current `0.34.73` package and
the `0.35.0` package release-plz will write into its temporary tree. Do not change the
`sase_core` workspace dependency; release-plz already rewrites that requirement because
`sase_core` is in the released version group.

Do not dispatch the release workflow in this phase. This phase's commit reaches GitHub
only when the phase turn finishes, and a dispatch before that push still runs against
the broken manifest.

Verify with `sase tool run check` in `sase-core` (not a bare `just check`). A
metadata-only manifest edit still has to pass that gate.

## Phase 2: Publish the wheel and raise the sase floor

Start only after phase 1's commit is on `sase-org/sase-core` `master`. Fetch and confirm
the path-only `sase_gateway` dependency is in the remote tree.

Cut the release with the documented urgent path from `sase-core`
`docs/pypi-retention.md`. `dry_run` defaults to true and skips the merge; the real cut
is:

```bash
gh workflow run release-plz.yml --repo sase-org/sase-core -f dry_run=false
```

That updates the release PR, waits for its checks, and squash-merges it. The merge push
tags the version and builds the wheel matrix. If that push starts no publish run, the
next six-hourly heal publishes it. Keep this to the one cut this floor is waiting on.

Wait until PyPI has a stable `sase-core-rs` version strictly newer than `0.34.73` whose
status is `complete` (all five files in `EXPECTED_DIST_SUFFIXES`, none yanked). From a
`sase-core` checkout:

```bash
EXPECTED_DIST_SUFFIXES="$(python3 -c "import yaml; print(yaml.safe_load(open('.github/workflows/release-plz.yml'))['env']['EXPECTED_DIST_SUFFIXES'])")"
python3 .github/scripts/pypi_release_files.py status <version>
```

Confirm the published tree contains `tool_run_receipts_report` (the release commit must
be a descendant of `0cf5147`, `9f86897`, and `e654e7c`). A partial upload is not a
floor. Do not heal an older version and do not delete anything from PyPI.

In the sase checkout, raise the declared window with the sanctioned ratchet, not a hand
edit:

```bash
just ratchet-core-window --report-only
just ratchet-core-window
```

`tools/ratchet_core_window` selects the newest complete release and writes
`>=<version>,<0.<minor+1>.0`. A `0.35.x` wheel therefore moves the requirement off
`<0.35.0`. Update `uv.lock` only through that tool. Do not publish `sase` itself.

Prove a fresh wheel install, with no linked-checkout override and no `SASE_CORE_WHEEL`
pointing at a local build:

- Install current sase into a new virtualenv so `sase-core-rs` resolves from PyPI at the
  ratcheted floor.
- Importing the catalog that contains `sase/sase.yml`'s `receipt:` block succeeds.
- `sase tool receipt check` and `sase tool receipts` both run on that install. A typed
  `no_receipt` from an empty ledger is a successful command. An unknown `receipt:` field
  or a missing `tool_run_receipts_report` binding is not.
- `tools/check_sase_core_rs_bindings` passes against that wheel.

Append the published version, the ratchet specifier, and the fresh-install result to
this phase and to `sase-10d`.

## Out of scope

- Closing `sase-1ah.8`, editing `plan:202609/e4_landing_remainder.md` status, or running
  `just symvision` as a landing step.
- Rebuilding the Rust report or the Python adapter.
- `just check-full`.
- Stale `sase.shells.followup` and `proc-shell` contract tests. Active epic `sase-1ab`
  already owns them.
- A live prepared `accept: pass` receipt. `just check` is still red on those `sase-1ab`
  failures, so a pass receipt cannot be minted on this tree.
- Publishing a `sase` wheel to PyPI (`sase-10d`).
