---
tier: tale
size: medium
title: 'Land E4: finish project-root shim retirement and the sase-core pin, then close
  sase-1lj'
goal: Project roots keep only AGENTS.md and CLAUDE.md (instruction-file count 2),
  sase memory init retires identical GEMINI.md/QWEN.md/OPENCODE.md copies safely and
  idempotently while home roots keep four, sase pins sase-core 181072b5, and epic
  sase-1lj is closed with its plan file marked done.
proposed_by: bbugyi200.athena.sase-1lj.land
bead: sase-1lj
status: done
---

- **PARENT:**
  [202610/e4_readable_agents_md.md](https://github.com/sase-org/sase--plans/blob/main/202610/e4_readable_agents_md.md)
- **BEAD:**
  [sase-1lj](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1lj/README.md)

# Land E4: finish project-root shim retirement, then close epic sase-1lj

## Why this exists

Epic `sase-1lj` (E4, plan `plan:202610/e4_readable_agents_md.md`) has every phase bead
closed. But phase `sase-1lj.2` (shim-retirement) landed **only** its sase-core half.
That is sase-core commit `181072b5` ("feat(content-layout): list only AGENTS.md and
CLAUDE.md in project agent documents"), already on sase-core `origin/master`. Its
recorded commit diffs hold no sase-side changes, so nothing can be recovered;
re-implement the sase side from scratch.

Phase notes `sase-1lj.2` #6/#7, `sase-1lj.3` #1/#2, and `sase-1lj.5` #1/#2 describe the
gap. The land agent confirmed it on master `f6ff20ef2a`:

- `git ls-files | grep -cE '(^|/)(AGENTS|CLAUDE|GEMINI|QWEN|OPENCODE)\.md$'` prints
  **5**. The root `GEMINI.md`, `QWEN.md`, and `OPENCODE.md` are still tracked, and all
  are byte-identical to `AGENTS.md`. The nested `tools/`, `src/sase/ace/`, and
  `demos/tapes/` sets are already gone (phase `sase-1lj.3`), so the target is **2**.
- `src/sase/amd/constants.py::PROVIDER_SHIM_FILES` still lists all four names, so
  `sase memory init` recreates the three root copies.
- `sase-core-revision.txt` is still `594faf31d80af79a148cb67968318b2bfb1a5770`.
  `tests/test_content_layout.py::test_path_classes_separate_tracked_generated_and_runtime_content`
  fails locally, because the locally built `sase_core_rs` already has `181072b5`.

Everything else in E4 landed and was verified: the projection header, neutral wording,
parity regex, CI drift gate, nested-guide fold, launch row and Files deck, decision
record, glossary, and docs. Follow-up triage is finished and recorded on `sase-1lj`
("LAND TRIAGE" note). **Do not re-triage it, and do not create beads.**

The epic's design decisions bind this work. Read them with
`sase bead read sase-1lj -r "<why>"` (DECISIONS) and in the plan file's "Design
decisions" and "shim-retirement" sections. In short:

- **D3. Project roots only.** Project roots and the directory of any nested `AGENTS.md`
  keep only `CLAUDE.md` beside `AGENTS.md`.
  - Home and chezmoi roots keep all four copies **until E5**.
  - Production lists shrink. Recognition lists (commit guard, pager detection, patch
    stitch audit) keep the retired names, so leftover copies in other projects are still
    classified as generated.
- **D5.** No feature flag.
- **D7.** sase-core is already done. Only the pin moves.

## Steps

Read `lint_and_test` memory (`sase memory read lint_and_test.md -r "<why>"`) and
`docs/contribute/rust-backend.md` ("The CI source revision pin") before starting.
Regenerate instruction files only through `sase memory init -C` (never by hand, and
never without `-C`: host finalizers own commits).

1. **Constants (D3).** In `src/sase/amd/constants.py`, split the provider list so one
   place owns each role, and export the new names:
   - project and nested production: `("CLAUDE.md",)`;
   - home and chezmoi production: today's four, with an "until E5" comment;
   - retired at project roots: `("GEMINI.md", "QWEN.md", "OPENCODE.md")`;
   - a recognition set that is the union of the two, for classifiers.

   Rewrite the comment that says agy reads only `GEMINI.md`. agy 1.4.0 reads both
   `GEMINI.md` and `AGENTS.md`, Qwen reads `QWEN.md` and `AGENTS.md`, and OpenCode never
   reads `OPENCODE.md` (E4 PROBES, `sase-1lj.2` note #1). Keep `PROVIDER_SHIM_FILES`
   only if callers still need the four-name tuple; otherwise replace every use with the
   role-appropriate constant.

2. **Route by root kind.**
   - `amd/_shared.py::provider_shim_specs`: the `prefer_templates` branch and the
     `is_chezmoi_home_root` branch keep the four-name home list. A plain project root,
     or the parent of any nested `AGENTS.md`, uses the `CLAUDE.md`-only list.
   - Home roots that are not chezmoi (the plain `~` root) must also keep four copies.
     Check how `plan_amd_memory_sync` / `memory_root_context` distinguish home from
     project (phase `sase-1lj.1` threaded `include_project_memory` for the projection
     header). Reuse that signal rather than inventing a new one.
   - `main/init_memory/root_planning_files.py::agent_doc_shim_plans` and
     `root_planning.py`'s `provider_shim_plan(...)` call must pass the root kind
     through.

3. **Safe retirement in `sase memory init`.** In `provider_shim_plan` (or a sibling
   helper), at project roots and beside every nested `AGENTS.md`, plan a `delete` for
   each retired file that matches either case:
   - it is byte-identical to the sibling `AGENTS.md`, either on disk or as about to be
     rendered;
   - it is a recognized legacy `@AGENTS.md` or `.tmpl` shim (`_is_provider_shim_text`).

   Leave a non-identical (custom) retired file in place. Report it as a warning, never
   as a blocker.

   Make sure the plan's deletes flow through the existing delete machinery:
   - `_PlannedDelete` and `_apply_provider_shim_plan`;
   - `--check` drift reporting, so pending deletions show as drift and exit non-zero;
   - the project commit-path collection.

   `main/init_memory/tracking.py` must stop treating retired names as managed at project
   roots. It must keep them managed at home roots.

4. **Consumers.** Use the role-appropriate list in:
   - `amd/inventory.py` (`sase instructions list` and `_shim_summary`'s "all shims"
     count). Show a pending `retired` state for a still-present identical copy and a
     kept `custom` file. Stop reporting retired names as `missing` at project roots.
   - `memory/history/scopes.py`. Project subjects stop listing retired paths;
     `build_home_scope` and `map_deployed_home_path` keep the four.
   - `axe/launch_evidence.py::capture_instruction_snapshot`. The workspace candidates
     use the project list; the home candidates keep four.

   Make the hard-coded copies use the recognition constant, keeping retired names
   recognized:
   - `memory/inventory_models.py` (`INSTRUCTION_ROOT_FILENAMES`, and fix its stale "agy
     reads GEMINI.md" comment);
   - `memory/history/pager_provider_core.py` (`_INSTRUCTION_NAMES`);
   - `finalizers/commit_memory_guard.py` (`_GENERATED_ROOT_BASENAMES`);
   - `patch_stitch_audit.py`.

   Also correct the provider-shim list in
   `src/sase/memory/assets/memory-directory-map.prompt.md`.

5. **Tests.** Add tests for:
   - idempotence: run the init plan twice, and the second plan is empty with no retired
     file recreated;
   - the custom retired-file case: kept, with a warning and no blocker;
   - the home-root case: nothing retired, four copies still written;
   - the chezmoi-root case: unchanged;
   - a nested-root case.

   Update the shim-set assertions and fixtures in:
   - `tests/test_memory_inventory.py`
   - `tests/main/test_instructions_list_inventory.py`
   - `tests/main/test_init_memory_agent_docs.py`
   - `tests/main/test_init_onboarding_reporting.py`
   - the other `tests/main/test_init_memory_*` suites that assert shim sets
   - `tests/ace/tui/modals/test_memory_pane_instructions.py`
   - `tests/_macro_terminology_common.py` and `tests/_macro_terminology_docs.py` (drop
     allowlisted paths that no longer exist; `_macro_terminology_docs` checks that
     listed files exist)
   - `tests/test_gemini_active_surface_guard.py`

   If the memory-pane history visual snapshots
   (`tests/ace/tui/visual/_ace_memory_pane_history_png_snapshot_*.py`) derive project
   shim rows from the constant, update their wires. Regenerate only the affected goldens
   with `just fix-tui-screenshots -- <selectors>` (read `tui` memory first).

6. **sase-core pin (D7).** Move `sase-core-revision.txt` to
   `181072b58e82a92c88f57a3d58952b049a50ddfe`.
   - Follow `docs/contribute/rust-backend.md`. Do not move it further than needed unless
     that doc requires it.
   - Update `tests/test_content_layout.py` so the project layout expects exactly
     `{"AGENTS.md", "CLAUDE.md"}`. Assert that the home and chezmoi layouts still list
     the five names, if the test covers them.
   - Confirm `tools/check_sase_core_rs_bindings` passes.

   Do not change sase-core.

7. **Apply.** Run `sase memory init -C`. It deletes the root `GEMINI.md`, `QWEN.md`, and
   `OPENCODE.md`. Then confirm:
   - `git ls-files | grep -cE '(^|/)(AGENTS|CLAUDE|GEMINI|QWEN|OPENCODE)\.md$'` counts
     only `AGENTS.md` and `CLAUDE.md`, so it prints `2` once the deletions are staged
     (`git status` shows the three as deleted);
   - `cmp AGENTS.md CLAUDE.md` is clean;
   - `sase memory init --check` exits 0, and a second `sase memory init -C` plans
     nothing and recreates no retired file;
   - `sase instructions render -p` exits 0;
   - `sase instructions list` shows no `missing` rows for retired names.

8. **Docs.** Correct the shim lists in place; don't grow legacy pages. Files:
   - `docs/reference/agents/initialization.md`
   - `docs/reference/agents/memory-initialization.md`
   - `docs/reference/memory/memory.md`
   - `docs/reference/providers/agent-clis.md` (the provider-file table)
   - `docs/reference/config/core-behavior.md`
   - anything else `git grep -n -E 'GEMINI\.md|QWEN\.md|OPENCODE\.md' -- docs` shows
     describing project roots

   State that agy, Qwen, and OpenCode read `AGENTS.md`, and that home roots keep four
   copies until E5. Leave `sdd/history/` untouched. Run `just fix`.

9. **Verify.** Run `sase tool run check` (never `just check-full`). It must be green
   except for the known failures that are not this tale's:
   - `tests/test_project_display_presentation_audit.py::test_project_labels_never_derive_from_paths`
     (task `sase-1ly`);
   - `tests/main/test_parser_root_help.py::test_root_help_renders_compact_help`
     (recorded on active epic `sase-1le.20.1`);
   - flakes that pass in isolation.

   Fix any other failure this work causes. Confirm `just symvision` is clean. Retire any
   dead public symbol left by the constants split (delete or privatize it). Do not add
   whitelist entries keyed to `sase-1lj`.

10. **Close out epic `sase-1lj`.** This is the final step and runs in this same turn. Do
    not wait for this work's commit, push, or CI.
    1. Run `sase bead epic-symbols sase-1lj`. The land agent found none. If any entry
       appears now, resolve it: wire it up, privatize it, add a non-test pragma, or
       delete it per the Symvision epic-whitelist policy. No still-open later bead needs
       an E4 exemption, so do not re-key entries.
    2. Check the chezmoi source:
       `git -C "$(chezmoi source-path)/.." diff --cached --stat`. The land agent found
       seven home files staged but uncommitted since 2026-10-16 18:22:
       - `home/AGENTS.md.tmpl`, `home/CLAUDE.md.tmpl`, `home/GEMINI.md.tmpl`,
         `home/QWEN.md.tmpl`, `home/OPENCODE.md.tmpl`;
       - `home/sase/memory/README.md`, `home/sase/memory/sase.md`.

       They hold exactly the `sase-1lj.1` neutral-wording regeneration that epic note #1
       asked for. Do not commit them yourself. Record their state in the close note as a
       human follow-up to commit and apply.

    3. Close the epic. Never use `--force`.

       ```bash
       sase bead close sase-1lj --note "<verification>"
       ```

       The note must summarize:
       - phases 1, 3, 4, and 5 verified in code and commits (`97825eee1d`, `552b38f7ed`,
         `8fde75d7ae`, `f6ff20ef2a`);
       - the sase-side shim retirement finished here: constants split, safe retirement,
         consumers, tests, the 3 root copies deleted, and the instruction-file count 5 →
         2;
       - the pin moved to `181072b5`, and `test_content_layout` updated;
       - `sase memory init --check` and `render -p` exit 0;
       - the `sase tool run check` result, with known failures named;
       - integration: commits since `97825eee1d` only regenerated instruction files, and
         stale nested-guide and shim references were fixed;
       - follow-up triage: `sase-1lz` new (agy explicit-channel spike), `sase-1ly` new
         (project-label audit), `+1` on `sase-1kr` (stale-lock flake), and root-help
         declined because it is recorded on `sase-1le.20.1`;
       - the staged chezmoi home files still need a human commit and apply.

       If the close is rejected for leftover `--epic-symbol` entries, fix them and close
       again.

    4. Run `just symvision` and confirm the whitelist is clean.
    5. Set `status: done` in the frontmatter of the epic's plan file. Its path is the
       PLAN path shown by `sase bead read sase-1lj -r "Need the plan path"`
       (`plan:202610/e4_readable_agents_md.md`); today it reads `status: wip`. Change
       nothing else in that file.
    6. `sase-1lj` has no `parent_bead`, so no parent handling is needed.

## Out of scope

- Home or chezmoi copies and `.chezmoiremove` (E5).
- Explicit delivery for agy, Qwen, or OpenCode (task `sase-1lz`).
- The two known pre-existing test failures above.
- Any memory-note edit. No memory note names the retired files.
