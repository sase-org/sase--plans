- **PLAN:** [../202607/lane_name_hood_neighbors.md](../lane_name_hood_neighbors.md)

%xprompts_enabled:false
# Previous Conversation

**User:**

You are the land agent for epic bead sase-99: verify the epic is truly complete, integrate it with changes
that landed since it started, then close it out.

1. Verify. Run `sase bead show sase-99` (children, linked plan file), then `sase bead show` on each child
bead. Confirm every bead note was addressed, and read the actual source code and the epic's commits (bead IDs
appear in commit messages) to confirm the work previous agents reported complete really is.

2. Integrate. Changes committed since this epic started could not integrate with this epic's feature while it was
incomplete. Find them (e.g. `git log` since the first commit mentioning sase-99, excluding the epic's own
commits; in a PR workflow also review commits on the base branch) and update anything that should now use what
this epic added or that duplicates or conflicts with it. This integration is part of the epic's work.

3. Land. Close the epic with `sase bead close sase-99`. AFTER closing, run `just symvision` if available
(epic-symbol whitelist entries for sase-99 expire at close) and remove the stale entries and unused code
it reports. Finally, set `status: done` in the frontmatter of the epic's plan file (the PLAN path shown by
`sase bead show`).

If steps 1-2 uncover remaining work, use your /sase_plan skill to plan it and complete the skill's tier-aware
validate/revalidate/propose loop. Make step 3 the plan's final phase (close, run symvision, mark the plan file done)
so the agent that executes the plan finishes the landing. Otherwise do step 3 now.

**Assistant:**

I'll start by verifying the epic's state and its children.

Now let me verify the implementation against the plan.

Found two stale-documentation gaps the epic's polish phase missed. Fixing them:

4 failures. Let me isolate them.

Epic's symvision whitelist entries were already self-cleaned by phase 3. Now integrating the base branch, which has moved one commit ahead:

Verification complete. Running `just check` on the integrated tree:

The formatting failure is pre-existing on the base branch (from `899a257f2`), not from my edits. Fixing it so `just check` is green:

I'll wait for `just check` to finish before closing the epic. Summary of verification so far:

**Step 1 — Verify: all 5 phases confirmed in source, not just reported.**

| Phase | Plan requirement | Verified |
|---|---|---|
| 99.1 | `agent_owns_lane`, `lane_fold_scale`/`AGENT_FOLD_SCALE`, pure projection, `~` modal rebuilt on it | `models/agent_hoods.py:33`, `models/fold_scale.py:18,58`, `models/agent_lane_neighbors.py`, `_neighbors.py:275` builds modal from the projection (D1 is real, with parity test `test_agent_neighbor_modal_matches_shared_lane_projection`) |
| 99.2 | Document-wide numbering, group labels, entry limit, dismissed styling, roles, map merging | `_member_roster.py:131` `MemberJumpNumbering`, `:255` merge, `:161` roles, group lines are plain (not section anchors) |
| 99.3 | NEIGHBORS section, lane wiring on all render paths, one merged map | `_agent_display_neighbors.py`; wired in `_agent_display_render.py:226`, `_agent_display.py:219`, `_agent_display_hints.py:167`; single publish at `_agent_display_header.py:288` |
| 99.4 | Digit jump + revive, role-aware revalidation, footer, help | `_member_jump.py:66,116,187`, `_fold.py:264` `_notify_lane_fold_scope`, footer `("0-9","neighbor")`, help reworded |
| 99.5 | Fold contracts, 3 PNG goldens, docs | 5 lane cases in `test_summary_fold_contracts.py`, all 3 goldens present |

The two PNG goldens confirm the design visually: the single-agent lane renders `❖ NEIGHBORS · 5` with digits 0–2 and the `… +2 more neighbors (zz / za to show more)` tail; the family lane numbers `FAMILY MEMBERS` 0–2 then `NEIGHBORS` 3–4 on one shared ladder with the `… +2 also listed under FAMILY MEMBERS` suppression tail (D2/D5 both real).

**Step 2 — Integrate:**
- Fast-forwarded to `origin/master`, which had one new commit (`899a257f2`, queued-agent status) authored after the epic. It already integrates — it added `"Queued"` to `_member_roster.py`'s status styles, so NEIGHBORS rows inherit queued rendering through the shared renderer.
- Reviewed every other non-epic commit since the epic's first (`1d6c95e60`). The tribe fold-hint rewrite, collapsed-tribe panel jump fix, and wait-priority work (sase-9k.2/3/4) touch disjoint surfaces — no duplication or conflict.
- Fixed two stale-doc gaps the epic's polish phase missed: `docs/agent_families.md:184` still said a fold chord on a regular agent changes nothing, and the help modal still called the 3-level scale `"clan/session"`. Both now describe the lane behavior the epic shipped (help-popup sync is required by `src/sase/ace/CLAUDE.md`).
- Also reformatted `tests/ace/tui/widgets/test_agent_list_status_indicators.py` — `899a257f2` left it failing `ruff format --check`, which blocked `just check`. Confirmed pre-existing by re-running with my edits stashed.

The epic's symvision `--epic-symbol` entries were already self-cleaned by phase 3, so nothing expires at close. Earlier full-suite run: 22018 passed, 4 failed — all 4 pass in isolation; the machine is running three other agents' pytest suites concurrently, and `test_suite_gate_integration` fails because its nested pytest can't win worker tokens under that load.

All four uncommitted files are mine (the two doc/help fixes, the keymap test that asserts the help wording, and the pre-existing format fix). Closing the epic first, per the finalizer.

Epic closed. Running symvision now that the epic-symbol whitelist entries have expired.

The plan file lives in the `plans` sidecar repo, so I need to open it through the repo skill before editing.

Plan file marked done. Now validating my diff before committing.

All green — lint, format, and the affected tests. Committing now.

The main repo is clean and pushed, but the status-indicator test isn't in the commit stat — let me confirm where that change went.

A concurrent commit (`9f8f04a4a`) landed the identical reformat, so that hunk became a no-op on rebase — the file is correctly formatted in HEAD. The plans sidecar still holds the plan-file change; let me commit that.

The auto-commit guard requires the dirty repo to be the **main** repo, so it won't cover a sidecar-only change. Let me confirm how the finalizer classifies the plans repo.

`sase/repos/` is gitignored by the main repo and the finalizer only auto-commits `kind="main"` plan-status changes — so the sidecar needs a direct commit, the same way sase's own tooling commits it. Let me match its message convention.

Epic sase-99 is landed. Both repos are clean and pushed.

**Landing (step 3):**
- `sase bead close sase-99` — closed and verified; all 5 phase children were already closed.
- `just symvision` — clean (`All public/private classes/functions are used properly!`). The epic's `--epic-symbol` whitelist entries had already been self-cleaned by phase 3, so nothing expired at close and there was no dead code to remove. Confirmed no lingering `sase-99` whitelist entries anywhere in config.
- Plan file `sase/repos/plans/202607/lane_neighbors_section.md` — `status: wip` → `status: done`.

**Commits:**
- `ffc7aa874` (main repo) — `docs(ace): describe lane fold behavior in help and agent docs (sase-99)`. Carries the two stale-doc fixes I found during verification: `docs/agent_families.md` still claimed a fold chord on a regular agent changed nothing, and the help modal still labeled the fold scales `"family"` / `"clan/session"` instead of naming the lane scope the epic shipped, plus the `tests/test_keymaps_defaults.py` assertions that keep the help popup and default keymap in sync.
- `e52522d1` (plans sidecar) — `chore: Mark SDD plan done`, using the same message the finalizer's own `auto_commit_done_sdd_plan_status` generates.

Two things worth flagging:

The `tests/ace/tui/widgets/test_agent_list_status_indicators.py` reformat is **not** in my commit. A concurrent agent's commit `9f8f04a4a` (sase-9m.2) landed the byte-identical fix first, so my hunk became a no-op during the rebase. The file is correctly formatted in HEAD and `ruff format --check` passes — nothing was lost.

The plan-status commit had to be a direct git commit rather than going through `sase commit`. `sase/repos/` is gitignored by the main repo, and the finalizer's auto-commit path (`commit_finalizer_git.py:144`) only fires for `repo.kind == "main"` — the plans sidecar is classified `kind="sdd"`, so it would have been left dirty. I matched the tooling's own convention by generating the message through `apply_auto_commit_tags_with_runtime`, the same helper the finalizer uses.

**Validation:** `just lint` fully green (ruff, mypy across 2364 files, pyscripts, symvision, toobig, keep-sorted), `ruff format --check` clean on all 4644 files, and the 74 tests in the two files my diff touches all pass.

---

%xprompts_enabled:true
# New Query

 Can you now help me make sure that agents that are named the same as one of an agent's hoods (and vice-versa) are matched as neighbors? This applied to both the new `NEIGHBORS` metadata panel section as well as the `~` (neighbors) keymap. Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
