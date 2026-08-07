---
tier: tale
title: Fail loudly when the built sase_core_rs is older than sase requires
goal:
  A dev install whose sase_core_rs is behind the pyproject floor stops the run with an
  actionable message instead of silently producing seven false test failures.
proposed_by: bbugyi200.athena.v1
create_time: 2026-08-07 16:53:17
status: wip
---

# Fail loudly when the built `sase_core_rs` is older than sase requires

## Problem

`just test` failed with 8 tests red on a clean `master` tree. Seven of them share one
cause: the `sase_core_rs` extension in the workspace `.venv` was built from a
`sase-core` checkout three releases behind the floor `pyproject.toml` declares, and
every layer that could have said so downgraded that fact to a warning.

## Diagnosis

### The failures split into two unrelated groups

| Failing test                                                                                                 | Cause                                                      |
| ------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------- |
| `tests/test_notification_modal_tags.py::test_a_gate_declared_panel_icon_reaches_the_classified_tab`          | stale core binding                                         |
| `tests/test_bead/test_snooze_gate.py::test_bead_snooze_gate_preview_carries_the_real_snooze_note`            | stale core binding                                         |
| `tests/test_bead/test_snooze_lifecycle.py::test_snooze_round_trips_through_every_persistence_surface`        | stale core binding                                         |
| `tests/test_bead/test_cli_snooze.py::test_snooze_leaves_one_note_naming_wake_time_length_target_and_reason`  | stale core binding                                         |
| `tests/test_bead/test_cli_snooze.py::test_bare_snooze_with_no_reason_or_target_still_leaves_a_note`          | stale core binding                                         |
| `tests/test_bead/test_cli_snooze.py::test_re_snoozing_appends_a_second_note_naming_the_replaced_wake_time`   | stale core binding                                         |
| `tests/test_bead/test_cli_snooze.py::test_a_multiline_reason_collapses_to_one_note_but_keeps_the_raw_reason` | stale core binding                                         |
| `tests/test_contract_manifest.py::test_contract_set_serial_runtime_stays_within_budget`                      | known flake, bead `sase-go` — **not** in this plan's scope |

### The stale binding, established

The workspace's linked `sase-core` checkout sat at core commit `ce2b828`
(`fix(core-py): bind the extension module explicitly`), whose
`[workspace.package] version` is `0.19.0`. `pyproject.toml` declares
`sase-core-rs>=0.19.3,<0.20.0`. The two core commits the seven tests pin were both
absent from that build:

- `ce8c04b feat(notifications): donate a per-tab icon from the newest declaring row`,
  released as core `v0.19.2` — the icon the modal-tags test expects on the tab record.
- `bfdc411 feat(bead): append a snooze note recording wake conditions`, released as core
  `v0.19.3` — the note every snooze test reads back.

Refreshing the checkout (`sase repo open sase-core`, which rebased it onto core
`4d1d05f` / `v0.19.3`) and rebuilding with `just install` turns all seven green: the
four affected modules run 87 passed, and a confirming full `just test` no longer reports
any of the eight.

### Why nothing stopped it

Dev installs deliberately build `sase_core_rs` from the local checkout with the
published version window lifted (`_core-overrides-arg` writes a `uv` overrides file
naming `sase-core-rs`). That is intentional — dev core normally runs _ahead_ of the
published window — but it means `pyproject.toml`'s floor constrains nothing locally, and
two guards that should have caught the _behind_ direction do not:

1. **`tools/validate_sase_core_rs_version` is right, and is ignored.** It already
   classifies the mismatch correctly and prints
   `sase-core checkout is behind: source version 0.19.0 ... Pull/rebuild sase-core`.
   Both callers throw that away:
   - `Justfile` `_setup` maps its `CORE_VERSION_ERROR` bit to
     `printf "[setup] WARNING: bump the published sase-core-rs window in pyproject.toml; dev installs build from ... regardless."`
     and continues.
   - `Justfile` `rust-install` runs it as
     `... validate_sase_core_rs_version ... || printf "[rust-install] WARNING: bump the published sase-core-rs window ..."`.

   Both warnings name only the _ahead_ remediation, so the one message an agent is
   likely to skim actively points away from the real fix. The validator's exit code is a
   bare `1` for behind, ahead, and internal errors alike, so no caller _can_ tell the
   fatal case from the benign one.

2. **The installed binding's version is never compared to anything.**
   `tools/validate_sase_core_rs` checks that every required binding _name_ exists and
   probes a few wire schemas. Neither core feature added a binding name or bumped a
   schema version, so a `0.19.0` build passes it cleanly. This is the load-bearing hole:
   `_setup` only rebuilds when that check fails, so refreshing the checkout and then
   running `just check` (rather than `just install`, which always rebuilds) reproduces
   the identical seven failures with the version check now _passing_ and no warning
   printed at all.

`~/.sase/plans/202608/test_suite_tier1.md` lists "stale `sase_core_rs` binding
detection" under **Tier 2 — explicitly out of scope**, which is why the hole is open.

### Why CI is green

`.github/workflows/ci.yml` checks out `sase-org/sase-core` at HEAD and builds a fresh
wheel every run, so CI can never be behind. Published-wheel installs resolve
`pyproject.toml` normally and cannot be behind either. Only a local checkout is exposed
— and `just install` never updates that checkout, so a long-lived workspace builds
against a frozen core indefinitely.

Commit
`7bdeee08e fix(deps): raise sase-core-rs floor to 0.19.3 for snooze note contract` (bead
`sase-h0`) already treated six of these same snooze tests once. That bead's environment
had **no** buildable core checkout, so it resolved the published `0.19.2` wheel, and
raising the floor was the right fix _there_. It cannot help a dev build, whose whole
point is that the floor is overridden — which is why the same six tests are red again
here for a different reason.

This is a recurring shape, not a one-off: `sase-cn` ("Raise sase-core-rs floor to 0.17.3
for bead_resolve_id"), `sase-d7` ("Raise sase-core-rs floor for notification snooze
contract") and `sase-h0` are all the same symptom triaged three separate times, and
`sase-h5`'s closing note records yet another agent tripping over "an unrelated stale
`sase_core_rs` build (needed a rebuild)" on the same day. Nothing distinguishes the two
causes at the point of failure, which is what this plan fixes.

## Scope

Make both directions of core-version skew impossible to run past silently, on the
dev-build path, without disturbing the intentional "dev core is ahead of the published
window" workflow.

## Steps

### 1. Give `tools/validate_sase_core_rs_version` a machine-readable verdict

Replace the single non-zero exit with distinct codes so callers can act on direction:

- `0` — the checkout satisfies the specifier (unchanged).
- `1` — internal/parse error: missing `Cargo.toml`, missing dependency, malformed
  specifier or version (unchanged for those cases).
- `3` — `sase-core checkout is behind` the specifier's inclusive minimum.
- `4` — `sase-core checkout is ahead of sase's compatibility window`.

Keep every stderr message byte-identical; only the exit code gains meaning.
`_mismatch_kind` already computes exactly this distinction — thread its result out of
`validate_sase_core_rs_version` instead of collapsing it to a `bool`. Leave
`--published-minimum` on `0`/`1`.

Update the two existing assertions in `tests/test_validate_sase_core_rs_version_tool.py`
that currently pin `result.returncode == 1` for the behind and upper-bound cases
(`test_sase_core_rs_version_validation_fails_when_source_is_behind`,
`test_sase_core_rs_version_validation_fails_when_source_hits_upper_bound`), and keep the
error-path tests pinned to `1`.

### 2. Carry the direction through `tools/validate_test_environment`

Add `CORE_VERSION_BEHIND_ERROR = 16` alongside the existing bits (`1`, `2`, `4`, `8`,
`64`). `_run_validator` currently collapses any non-zero into one caller-supplied bit;
let the `core-version` check map its exit code onto `16` for `3` and `1` for every other
non-zero value. Bit `1` keeps its present meaning of "mismatch that is only a warning"
(the ahead case and internal errors).

Bump `CACHE_SCHEMA_VERSION` to `2` so a cache written under the old single-bit meaning
is discarded rather than replayed with the new one.

`_fingerprint_inputs` already hashes `core-cargo` (`<sase-core-dir>/Cargo.toml`) and
every validator script, so a checkout refresh or a change to these tools invalidates the
cache correctly. No fingerprint change is needed.

### 3. Make `_setup` and `rust-install` fatal on _behind_ only

In `Justfile` `_setup`, before the existing bit-`1` warning, handle bit `16` by printing
an actionable message and exiting non-zero. The message must name the actual remedy —
refresh the checkout, then reinstall:

```
[setup] ERROR: the sase-core checkout is behind the sase-core-rs floor in
pyproject.toml; the extension built from it will not satisfy sase's tests.
In a SASE workspace run `sase repo open sase-core`; otherwise update the checkout
directly. Then rerun `just install`.
Set SASE_ALLOW_STALE_CORE=1 to proceed anyway (intentional bisects only).
```

Honour `SASE_ALLOW_STALE_CORE=1` by downgrading it back to a warning, so bisecting an
older core stays possible.

Apply the same treatment in `rust-install`, whose `|| printf ... WARNING` currently
swallows every failure including this one. Leave the `setup-demos` path alone.

While in `_setup`, note that its post-rebuild `tools/validate_sase_core_rs` call has no
effect on the recipe's exit status: `just` runs recipes under `sh -cu` (no `-e`), and
the following `if [ $((validation_status & 12)) -ne 0 ]` evaluates to `0` when that bit
is clear, masking the failure. Propagate it (capture the status and exit non-zero) so
step 4's new check is not swallowed the same way.

### 4. Detect a stale _installed_ binding, not just a stale checkout

This is what makes step 3 more than a speed bump: without it, refreshing the checkout
and running `just check` reproduces the whole failure with no warning at all.

Extend `tools/validate_sase_core_rs` to check the installed distribution's version
before it probes bindings:

- Read `importlib.metadata.version("sase-core-rs")` — `maturin develop` writes the Cargo
  workspace version into the dist metadata, so this is the built extension's true
  version. The extension module itself exposes no `__version__`; do not add a dependency
  on one.
- Fail if it does not satisfy `pyproject.toml`'s inclusive minimum, reusing the
  specifier parsing already in `tools/validate_sase_core_rs_version` rather than writing
  a second parser.
- Accept a new optional `--sase-core-dir`. When it is given _and_ has a `Cargo.toml`,
  also fail when the installed version differs from that checkout's
  `[workspace.package] version` — that is precisely "the checkout moved and nobody
  rebuilt". Have `_setup` pass `--sase-core-dir` through the `core-bindings` check.

Failing here returns `1`, which `validate_test_environment` already maps to
`CORE_BINDINGS_ERROR` (bit `2`), which `_setup` already handles by rebuilding. So the
stale-build case self-heals through the existing path and needs no new Justfile branch.

**State the limitation in the module docstring:** between core releases the Cargo
workspace version does not move, so a checkout advanced to an unreleased core commit
still reads as the same version and this check will not notice. Bead `sase-bh` hit
exactly that blind spot from the other side and concluded "version metadata cannot
distinguish the two cores ... any feature detection must probe behavior, not the version
string." That remains true, and this step does not claim otherwise: it catches
release-boundary skew — which is the skew that produced these seven failures — and the
existing binding-name and wire-schema probes keep covering what a version number cannot.
Do not replace those probes with the version check.

### 5. Tests

Cover the new behaviour in the modules that already own these tools:

- `tests/test_validate_sase_core_rs_version_tool.py` — the `3`/`4` exit codes, plus the
  error paths still on `1`.
- `tests/test_validate_test_environment_tool.py` — a behind verdict sets bit `16` and
  not bit `1`; an ahead verdict still sets bit `1`; a cache written at schema version
  `1` is rejected.
- `tests/test_validate_sase_core_rs_tool.py` — an installed version below the floor
  fails; an installed version that disagrees with `--sase-core-dir` fails; the in-range,
  in-agreement case passes.
- `tests/test_justfile_lint.py` — `_setup` and `rust-install` exit non-zero on the
  behind bit, still warn on the ahead bit, and honour `SASE_ALLOW_STALE_CORE`.

**Cost constraint — read before writing these.** All four modules are in
`tests/contract_manifest.txt`, and the contract set is added to _every_ diff-scoped
selection. Its budget guard is already the marginal failure in the same `just test` run
(bead `sase-go`): measured on this host at 24.2–25.2s normalized against a 30.0s
ceiling, up from the 22.6–23.2s recorded at `d66101e8f` because the set has gained ~19
tests since. Keep the new tests to in-process assertions and `tmp_path` fixtures; do not
add tests that spawn a nested `pytest`. After the change, run
`tests/test_contract_manifest.py` alone on a quiet host and record the normalized figure
in the commit message. If it crosses 30s, trim per the curation procedure in
`~/.sase/plans/202608/test_suite_tier1.md` — do not raise `_BUDGET_SECONDS`.

## Verification

1. `just install` first — workspaces are ephemeral and the linked `sase-core` checkout
   may be stale. Confirm it reports the core version it built.
2. `just check-full`. This change touches `Justfile` and `tools/`, which are
   selection-tooling and broadening paths, so the scoped lane is not sufficient.
3. Prove the guard fires. With a throwaway `Cargo.toml` declaring an old version:
   ```
   mkdir -p /tmp/core-behind
   printf '[workspace.package]\nversion = "0.19.0"\n' > /tmp/core-behind/Cargo.toml
   .venv/bin/python tools/validate_sase_core_rs_version \
       --sase-core-dir /tmp/core-behind --pyproject pyproject.toml; echo "exit=$?"
   ```
   Expect exit `3` and the unchanged "checkout is behind" text. Then run
   `just --set sase_core_dir /tmp/core-behind _setup` and confirm it exits non-zero with
   the new ERROR message, and that `SASE_ALLOW_STALE_CORE=1` makes it a warning.
4. Prove the ahead direction is untouched: repeat with `version = "0.99.0"` and confirm
   exit `4`, a warning, and a zero exit from `_setup`.
5. `just _lint-symvision` is known-red on `master` for an unrelated reason
   (`progress_fingerprint` in `src/sase/llm_provider/commit_finalizer_git.py`, bead
   `sase-fj`). If that is the only symvision failure it is pre-existing.

## Out of scope

- **`test_contract_set_serial_runtime_stays_within_budget`.** Already tracked as bead
  `sase-go` ("flakes despite its calibration probes"). A `+1` carrying this reproduction
  and its measurements has been recorded there; that met the bead's `+1` wake target, so
  it is now `ready` and will reach the owner through its own `TaskTriage` gate. The
  measurements, for context: quiet host 24.2s / 24.7s / 25.2s normalized; under 96 CPU
  spinners 23.3s / 23.6s — the probe absorbs pure-CPU contention almost perfectly, so
  the fault is the _shape_ of the probe workload against a memory/IO-heavy parallel
  lane, compounded by ~2s of headroom lost to the contract set gaining ~19 tests since
  its baseline measurement.
- **Two parallel-only flakes** observed on the confirming rerun and not in the original
  report:
  `tests/ace/tui/test_agent_metadata_search.py::test_inline_metadata_search_yank_and_frozen_refresh`
  and
  `tests/ace/tui/test_notification_custom_gate.py::test_tracked_executor_reports_terminal_and_extra_commands_live`.
  Both pass in isolation. Deliberately **not** filed: the 2026-08-05 backlog triage
  consolidated this exact class onto umbrella bead `sase-ct` and set the bar for a new
  `+1` at "a specific node turns into a deterministic failure". These are one-off and
  non-deterministic, and the confirming run was contended by an unrelated concurrent
  process, so reopening `sase-ct` again would be noise against that stance.
- **Automatically refreshing the linked `sase-core` checkout** during `just install`.
  That is workspace/linked-repo machinery, not a test-environment guard; the guard here
  deliberately stops and tells the agent to run `sase repo open sase-core`.
- **Raising the `pyproject.toml` floor again.** The floor is already correct at
  `>=0.19.3`; it is the dev-build path that ignores it.

## Proposed follow-up (needs explicit user permission)

`sase/memory/build_and_run.md` tells agents to run `just install` because workspaces are
ephemeral, but says nothing about the linked `sase-core` checkout being the thing that
goes stale, nor about `sase repo open sase-core` being how to refresh it. Adding that
sentence would have saved this whole investigation.

IMPORTANT: You have my explicit approval to make this memory update and run the
appropriate `sase init` command to apply the agent instruction file changes.
