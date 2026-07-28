---
tier: tale
title: Integrate and land the lumberjack runner gate epic
goal: The published core dependency is truthful, the lane-level runner gate remains
  verified, and sase-af is closed with its durable plan marked done.
bead: sase-af
parent: plans:202607/lumberjack_wait_runners.md
create_time: 2026-07-28 11:04:28
status: wip
---

- **PROMPT:** [202607/prompts/land_lumberjack_wait_runners.md](prompts/land_lumberjack_wait_runners.md)

# Plan: Integrate and land the lumberjack runner gate epic

## Goal

Finish `sase-af` without closing over a dependency regression introduced by work that landed while the epic was running.
Preserve the completed lumberjack-level `wait_runners` feature, make the published `sase-core-rs` minimum truthful for
every Rust binding now used by `sase`, add coverage that prevents a linked-core development install from masking this
class of error again, and only then close the epic, run the post-close Symvision audit, and mark its durable plan done.

## Verified starting point

- `sase-af.1`, `sase-af.2`, and `sase-af.3` are closed with resolution `done`.
- Rust commit `717b5b9` accepts and validates non-negative lumberjack `wait_runners`; it shipped in `sase-core-rs`
  0.12.2.
- Sase commit `bd630ec` parses and displays the key, threads it through scheduled, CLI, and ACE chop execution, stamps
  every prepared proposal, preserves a proposal-authored `%wait(runners=...)`, and covers previews, independent
  launches, clan launches, directive scanning, schema validation, editor ordering, CLI output, and docs.
- Chezmoi commit `596cc5e` configures `code_quality.wait_runners: 0`. The live `sase axe lumberjack list` reports that
  value, and `sase axe chop run 'recent_bug_audit[sase]' --dry-run --force --chop-verbose` renders `%wait(runners=0)` in
  the proposal scaffold without launching an agent.
- The later ACE wait-lane commit `61013b2` consumes the existing runtime `Agent.wait_runners` state and therefore
  already renders these agents in the `[runners]` wait lane; it does not duplicate or conflict with proposal injection.
- The integration defect is in the published-core floor. Later commits `105b597` in `sase-core` and `8b2baa8` in `sase`
  added the `sdd_plan_header_block_*` Rust bindings and unconditional Python callers. The current main-repo dependency
  and lock still allow/install `sase-core-rs==0.12.2`, whose wheel has `validate_axe_config` but does not export
  `sdd_plan_header_block_wire_schema_version`. A linked-core `just install` masks this because it builds the newer
  checkout. At planning time, 0.12.2 is still the newest PyPI wheel and the 0.12.3 release workflow is not yet terminal,
  so do not assume a release version from a release branch alone.
- The land audit's `just check` passed formatting, Ruff, mypy, script lint, Symvision, and size lint, then stopped at
  the two already-recorded `plan_header_provenance.md` prompt/reverse-link validation errors owned by the active
  `sase-ag` epic. Recheck current state rather than weakening validation or folding that epic's content work into this
  tale; the final `just check` must pass after the owning work lands.

## Phase 1 — Repair and guard the published-core compatibility floor

1. Re-read the current base branch before editing. Inspect `sase-ag` and its active agents/commits so this plan does not
   duplicate a dependency bump that lands first. If the declared minimum has already moved, prove that exact published
   minimum contains both the AXE validation API and all `sdd_plan_header_block_*` bindings and retain the compatible
   newer result.
2. Through `/sase_repo`, inspect the current `sase-core` release state. Wait for the release containing both `717b5b9`
   and `105b597` to be genuinely published to PyPI, not merely present on a release-plz branch or in a successful CI
   build. It is expected to be 0.12.3, but use the actual published compatible version. Verify it in an isolated
   environment by installing that exact wheel and checking the AXE validation and plan-header binding names. Do not
   close `sase-af` while the only published compatible floor is still missing.
3. In `sase`, raise the inclusive `sase-core-rs` floor in `pyproject.toml` to that published version while preserving
   the existing `<0.13.0` ceiling, refresh `uv.lock`, and update the declared-minimum assertion in
   `tests/test_sase_core_rs_telemetry_smoke_tool.py`.
4. Add an exact-published-minimum smoke for the plan-header API beside the existing telemetry and bead-resolution smoke
   tools. It must at least require the complete binding family used by `src/sase/sdd/plan_header_block.py`, check wire
   schema version 1, and exercise a small parse/render or parse/mutation round trip. Add focused unit coverage for the
   smoke tool and run it from the existing `published-core-smoke` CI job after that job installs the version extracted
   from `pyproject.toml`. This check must execute against the isolated published wheel, not the linked core checkout.
5. Run `just install`, focused tests for the minimum-version and new smoke tool, the new smoke against an isolated
   environment containing the exact declared wheel, and `just check`. Re-run the live lumberjack list and the quoted
   verbose dry-run command above to ensure the dependency integration did not change the completed `%wait(runners=0)`
   behavior. Commit the main-repo integration through the required SASE commit workflow.

## Final phase — Close the epic, run Symvision, and mark the durable plan done

1. Re-run `sase bead show sase-af` and each child show. Confirm all descendants remain closed with resolution `done` and
   the published-minimum checks above are green, then run `sase bead close sase-af` without `--force`. If the close is
   rejected, resolve or reopen the named unfinished work; never force a `done` result.
2. After the close succeeds, run `just symvision`. Follow the audited `symvision.md` memory hierarchy for every finding:
   remove stale `sase-af(...)` epic-symbol entries if any now exist, delete genuinely unused code and its dead tests, or
   make internal-only symbols private. Do not suppress findings merely because they appeared after close. Re-run
   `just symvision`, and re-run `just check` if this cleanup changes the main repository.
3. Open the plans sidecar with `sase repo open plans -r "Mark the landed lumberjack wait-runners epic plan done"`. In
   `202607/lumberjack_wait_runners.md`, change only the frontmatter `status: wip` to `status: done` after the bead is
   closed. Validate the plan, commit the sidecar change through the required SASE commit workflow, and confirm the plans
   checkout is clean.
4. Finish with evidence: `sase bead show sase-af` reports `closed`/`done`; every child remains `closed`/`done`;
   `just symvision` and the required repository checks pass; the original plan frontmatter reports `status: done`; the
   declared exact published core minimum exports the AXE and plan-header APIs; and the live code-quality dry-run still
   contains exactly one `%wait(runners=0)`.

## Constraints

- Do not change `code_quality.wait_runners: 0`, broaden the feature to non-proposal launchers, or alter runner-slot
  semantics; those parts of `sase-af` are complete.
- Do not hand-edit or web-fetch another repository. Use `/sase_repo` and only the path it prints.
- Do not treat the linked-core development build as proof that the declared published minimum is sufficient.
- Coordinate with the still-running `sase-ag` epic by observing current state and accepting a compatible change that
  lands first; do not overwrite or duplicate its work.
- The close/Symvision/plan-status sequence above is the final phase and must stay in that order.

## Acceptance

- The exact inclusive `sase-core-rs` minimum declared by `sase` is available on PyPI, contains the lumberjack
  `wait_runners` validator, and exports every plan-header binding currently required by Python.
- CI has an exact-published-minimum plan-header smoke that would fail against 0.12.2.
- `just check` passes, and the live code-quality dry-run still scaffolds `%wait(runners=0)`.
- `sase-af` and all three children are closed with resolution `done`, post-close Symvision is clean, and
  `202607/lumberjack_wait_runners.md` has `status: done`.
