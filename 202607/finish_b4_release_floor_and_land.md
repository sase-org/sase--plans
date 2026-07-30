---
tier: tale
title: Finish the sase-b4 published-core floor and land the epic
goal:
  The artifact-reference file-row gate is required by an installable published sase-core-rs minimum, protected by an
  exact-wheel smoke, and landed with verified child beads, post-close Symvision, and a completed epic plan.
bead: sase-b4
create_time: 2026-07-30 08:12:51
status: wip
---

- **PROMPT:** [202607/prompts/finish_b4_release_floor_and_land.md](prompts/finish_b4_release_floor_and_land.md)
- **PARENT:**
  [202607/at_reference_file_row_gate.md](https://github.com/sase-org/sase--plans/blob/main/202607/at_reference_file_row_gate.md)
- **BEAD:** [sase-b4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-b4/README.md)

# Finish the `sase-b4` published-core floor and land the epic

## Goal

Complete the one criterion that the land audit found was closed prematurely, preserve it as a tested published-package
contract, and then perform the original epic landing sequence. The target epic is `sase-b4`; its incomplete child is
`sase-b4.3`, and its canonical epic plan is `plans:202607/at_reference_file_row_gate.md`.

## Audit baseline

- `sase-b4.1` is implemented by sase-core commit `4e61ad05ed30824e827e50a3d2d99cfca82200ef`. The current core code gates
  Kind-stage path rows only for tier-0 kind-prefix matches, exposes `files_suppressed`, supports the additive
  `include_files` option through `sase_core_rs`, and maps LSP `CompletionTriggerKind::INVOKED` to that option.
- Later sase-core commit `24e773ec2789199613e53b32d9169dce0423d6c7` changed the LSP payload-inventory/cache path. It
  retained `AtReferenceMenuOptionsWire` and still calls `editor_build_at_reference_menu_with_options`, so it is already
  integrated with the gate.
- `sase-b4.2` is implemented by sase commit `9ba92b09a7cacd192f59ccc0756970d8ca67526d`. The prompt owns sticky per-menu
  reveal state, the first `Ctrl+T` reveals suppressed files without accepting a kind, the second press keeps the old
  force-completion behavior, and the completion panel advertises `[^T] files`.
- The land audit reran 92 focused Python/TUI/help tests, 18 Rust at-reference tests, the Rust-to-Python binding test,
  and the grouped LSP completion test; all passed at the integrated repository tips.
- A fresh `just check` passed formatting and every lint stage, including Symvision, then stopped at two pre-existing
  plan-link diagnostics: `202607/editor_artifact_ref_parity.md` has a duplicate `PROMPT` header and
  `202607/prompts/editor_artifact_ref_parity.md` consequently lacks its reverse link. The older diagnostic involving the
  `sase-b4` plan itself is already resolved.
- sase-core release commit `493a632924c4b5109f9fe9162999cd46059b7916` labels the carrying release as `0.12.19`, but at
  audit time PyPI returned 404 for `sase-core-rs==0.12.19` and reported `0.12.18` as latest. Consequently
  `pyproject.toml` still accepts `sase-core-rs>=0.12.18,<0.13.0` and `uv.lock` still resolves `0.12.18`. That wheel
  predates the new `options` argument, so `sase-b4.3` was reopened and must not be reclosed until a carrying wheel is
  actually installable.

## Phase 1: Publish-aware dependency floor and durable wheel smoke

1. Run `sase bead show sase-b4` and `sase bead show sase-b4.3`, then move `sase-b4.3` from `open` to `in_progress` when
   work starts.
2. Confirm from PyPI, not only from the sase-core source checkout, that `sase-core-rs==0.12.19` is installable. The
   release changelog must still show that this version contains the file-reference gate. If `0.12.19` is not published
   yet, leave the child open and wait; do not substitute the older wheel, close the phase, cancel it, or force the epic
   merely to make progress.
3. Raise the inclusive requirement in `pyproject.toml` to `sase-core-rs>=0.12.19,<0.13.0` and regenerate `uv.lock` so
   both the recorded requirement and resolved package version are at least `0.12.19`. Do not change the `<0.13.0`
   compatibility ceiling.
4. Add a small published-wheel smoke executable under `tools/` and invoke it from the `.github/workflows/ci.yml`
   `published-core-minimum-smoke` job. The smoke must import only the installed `sase_core_rs` wheel and directly prove
   all of the new contract:
   - a default `@f` Kind-stage menu with a prefix-matching `file` kind suppresses matching path rows and reports
     `files_suppressed`;
   - passing `options={"include_files": True}` returns both kind and path groups;
   - a query with no kind-prefix hit returns matching path rows without the option. This is required because the
     existing static binding-name check can prove that `at_reference_menu` exists but cannot detect an older function
     signature that lacks the new `options` argument.
5. In a temporary Python 3.12 virtual environment with no local-core override, install the exact new minimum from PyPI
   and run the new smoke plus `tools/check_sase_core_rs_bindings`. Keep this verification independent of `SASE_CORE_DIR`
   and the editable core checkout.

## Phase 2: Revalidate the integrated repositories and finish `sase-b4.3`

1. Recheck the main and linked sase-core histories for commits after the original `sase-b4` commits. Confirm that no
   newer at-reference, completion, dependency, or published-core-smoke changes require another integration edit.
2. Refresh the canonical plans sidecar. If the two `editor_artifact_ref_parity.md` link diagnostics recorded in the
   audit baseline still exist, repair the duplicate `PROMPT` header/reverse-link pair according to the plan-link
   validator rather than ignoring the failure. If another owner has already repaired them, inspect that landed change
   and continue.
3. Run `just install` before repository checks. Run the focused gate tests, `just test-visual`, and the full
   `just check`; resolve any failures caused by this work. Do not treat the known plan-link diagnostics as permission to
   report a partial check: the final `just check` must pass. In the linked sase-core checkout, run its formatting,
   clippy, and workspace-test commands if Phase 1 required any core change (a core change is not expected).
4. Run `tools/validate_sase_core_rs_version --pyproject pyproject.toml --published-minimum` and repeat the exact-minimum
   temporary-wheel smoke after the final lock state.
5. Close `sase-b4.3` with `sase bead close sase-b4.3 --note "<evidence>"`. The note must name the published minimum, the
   dependency/lock changes, the exact-wheel smoke, the later-commit integration review, and the successful checks.

## Phase 3: Land `sase-b4`

1. Run `sase bead show` for the epic and every child one last time. Confirm all children are closed with resolution
   `done` and that their plan criteria and notes are supported by the current code and tests.
2. Close the epic without `--force`:
   `sase bead close sase-b4 --note "<verified implementation, later-commit integration, published floor, and checks>"`.
   If normal close is rejected, finish or reopen the named work; never force a `done` resolution.
3. Only after the epic closes, run `just symvision`. Remove any expired `--epic-symbol sase-b4(...)` entries it names
   and fix genuine unused/private-symbol findings according to the Symvision memory hierarchy: delete dead code first,
   make file-local code private when appropriate, and do not add a new suppression for a closed epic. Re-run
   `just symvision`, then run `just check` after any repository file change.
4. Use `sase repo open plans -r "<reason>"` to access the canonical plans sidecar. In
   `202607/at_reference_file_row_gate.md`, change only the frontmatter `status` from `wip` to `done`, then validate the
   plan and inspect the plans-sidecar diff. This status update is the final landing action.

## Completion criteria

- PyPI serves the carrying `sase-core-rs` version and sase no longer accepts or locks a pre-gate minimum.
- CI has an exact-published-minimum behavioral smoke for the additive menu-options contract.
- `sase-b4.3` and then `sase-b4` close normally with evidence; no descendant is swept or force-closed.
- Post-close Symvision and the required repository checks pass.
- `plans:202607/at_reference_file_row_gate.md` has `status: done`.
