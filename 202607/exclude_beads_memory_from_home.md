---
tier: tale
title: Exclude sase_beads.md from home root AGENTS.md and memory
goal:
  Ensure sase/memory/sase_beads.md is generated and included in Tier 2 long-term memory exclusively for sase-managed
  project repos, and excluded from home root memory and ~/AGENTS.md.
proposed_by: bbugyi200.athena.qr
create_time: 2026-07-31 16:27:57
status: wip
---

- **PROMPT:** [prompts/202607/exclude_beads_memory_from_home.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/exclude_beads_memory_from_home.md)

# Plan

1. **Root Planning and Rendering Modifications**
   - Update `memory_root_context` and `plan_memory_root` in `src/sase/main/init_memory/root_planning.py` to accept
     `include_beads_memory: bool = False`.
   - In `src/sase/main/init_memory_handler.py` (`_memory_root_plans`), pass
     `include_beads_memory=inputs.is_sase_managed` for `project_root` and `include_beads_memory=False` for `home_root`.
   - In `memory_root_context`, only render generated bead memory and pass `generated_long_notes` to `_amd_sync_plan`
     when `include_beads_memory` is `True`.
   - In `src/sase/main/init_memory/root_rendering.py` (`render_expected_memory_files`), only add
     `sase/memory/sase_beads.md` to expected files when `generated_beads_content` is provided.
   - In `root_planning.py`, if `include_beads_memory` is `False` and `sase/memory/sase_beads.md` exists in the target
     root, add it to `memory_delete_paths` so any existing home copy is cleaned up.

2. **Documentation Updates**
   - Update `docs/init.md` and `docs/memory.md` to specify that `sase/memory/sase_beads.md` is generated and included in
     Tier 2 long-term memory for SASE-managed project repos only, and omitted from home memory and `~/AGENTS.md`.

3. **Test Suite Updates**
   - Update `tests/main/test_init_memory_plan.py` to assert that `home_root` does not generate `sase_beads.md` or
     reference it in `home_root/AGENTS.md`, while `project_root` does.
   - Update `tests/main/test_init_memory_formatting.py`, `tests/main/test_init_memory_handler_outputs.py`,
     `tests/main/test_init_memory_commit.py`, and related init_memory tests to align with the home vs project memory
     split.

4. **Verification**
   - Validate the plan with `sase plan validate`.
   - Run `just check` to verify code quality, typing, formatting, and test suite.
