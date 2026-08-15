---
tier: tale
title: Complete the sase-agent taxonomy epic landing
goal:
  Finish the missed terminology and generated-skill integration for sase-m9.1.1,
  disposition every proposed follow-up, verify the combined tree, and close the epic
  with its post-close cleanup complete.
size: medium
proposed_by: bbugyi200.athena.sase-m9.1.1.land
bead: sase-m9.1.1
create_time: 2026-08-14 21:15:26
status: wip
---

- **PARENT:** [202608/shell_taxonomy.md](shell_taxonomy.md)
- **BEAD:**
  [sase-m9.1.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m9/sase-m9.1.1.md)

# Plan: Complete the sase-agent taxonomy epic landing

## Verified starting point

The epic `sase-m9.1.1` has three closed phase children. Its committed implementation is
on `master` as:

- `4280bc990c59dd3c2558af442673b0c037015281` (`sase-m9.1.1.1`): canonical `SaseAgentRef`
  projection and narrow `AgentLaneRef` / `lane_*` compatibility aliases.
- `e923dcb5d104705db58ffdf402309b85aac160b5` (`sase-m9.1.1.3`): monitor `--agent` CLI
  terminology with hidden `--lane` compatibility and historical wire fields preserved.
- `2265f2618c149e6c29cada008d8121c7544b9332` (`sase-m9.1.1.2`): glossary, generated
  memory surfaces, ACE terminology, and visual snapshot updates.

Current-tree verification already passed 283 focused projection, provenance, monitor
parser/handler, glossary, ACE terminology, and keymap tests. `sase memory init --check`
also passed. The workspace entry point (`.venv/bin/sase`) advertises
`monitor start -a/--agent` and `monitor list -l/--agent`, while suppressing `--lane`
from help. The host-level `sase` executable is an older installed release and is not
evidence about the checked-out parser.

Three non-epic commits landed after this epic started: `9c66dafee` (Antigravity Gemini
3.7 Flash), `33180daf1` (typed Artifacts-pane row identities), and `97e12b29e` (the
Antigravity model in `@cheaper`). Their diffs contain no agent-lane terminology or
projection callers, so no source integration is required for them. The Artifacts-pane
commit plausibly explains one independent visual-test proposal below, but it predates
the epic commits in their ancestry and is already present in the combined tree.

## Phase 1: Finish taxonomy and generated-skill integration

Correct the genuine migration gaps without renaming unrelated AXE scheduling, scoped
test, ACE context/display-layout, or launch-routing lanes, and without changing monitor
record/runtime behavior:

1. Rewrite the user-facing agent-ownership language in `docs/monitors.md`: replace “The
   lane picture,” the `lane "acme"` diagram label, follow-up ownership, stalled-lane
   prose, and planner-lane prose with the canonical sase-agent / agent-family / agent-
   shell terms. Keep the documented hidden `--lane` compatibility alias because it is an
   intentional migration boundary.
2. Fix nearby retired terminology that clearly denotes a sase agent rather than a
   generic layout lane, including the `Agent-lane count` argument description in
   `src/sase/ace/tui/widgets/agent_info_panel.py` and contradictory docstrings such as
   `sase_agent_name()` returning a “lane name.” Prefer small, reviewable wording or
   local identifier corrections; do not perform a broad mechanical rename of internal
   ACE presentation-lane machinery.
3. Re-run a repository terminology audit. Classify every retained relevant occurrence in
   the landing note as either an explicit compatibility boundary (`sase.agent_lanes`,
   `AgentLaneRef` / `lane_*`, monitor `lane` records and JSON, internal monitor engine
   APIs, hidden `--lane`) or an explicitly out-of-scope generic lane (AXE, tests,
   context/display layout, launch routing). Fix any remaining user-facing use of the
   retired agent-lane concept.
4. The monitor skill template change is committed but not deployed:
   `.venv/bin/sase skill init --diff` currently shows only the intended monitor
   terminology update in every generated runtime destination. Follow `/sase_repo` before
   accessing the linked `chezmoi` repo, then, only from the clean canonical committed
   template source, run the generated-skill deployment prescribed by
   `generated_skills.md` (`.venv/bin/sase skill init --force`, followed by
   `chezmoi apply` when required). Do not hand-edit a generated destination. Verify the
   deployed monitor skill no longer differs from the source.

Run `just install` first if the continuation has a fresh environment. Re-run the focused
suites for the projection, commit provenance, monitor parser/handlers, glossary, ACE
terminology, and keymaps. Check `.venv/bin/sase monitor start --help` and
`.venv/bin/sase monitor list --help`, `sase memory init --check`, and the
generated-skill preview. Run `just check` after all repository edits.

Before landing, run `just check-full` only through `/sase_monitor`, with a concrete
`--next` instruction that resumes this plan and performs the follow-up disposition and
final landing below. Treat any failure caused by this epic as epic work and fix it
before continuing.

## Phase 2: Disposition every proposed follow-up

Review the three distinct `PROPOSED FOLLOW-UP:` reports below. They are not caused by
the taxonomy epic, so invoke `/sase_new_task` separately for each and let that workflow
corroborate a semantic duplicate, attach it to a causally related active epic, or create
an intentionally sized task. Every report must identify its proposing child bead and the
evidence. Record the exact outcome for the final close note, including the reason if any
proposal is declined.

1. From `sase-m9.1.1.1`: CLI/TUI tests compare raw strings with ANSI-styled output under
   forced color. This was independently reproduced on the combined tree with
   `CI=1 FORCE_COLOR=1 uv run pytest -q tests/test_file_hook_cli.py tests/test_plan_validate.py`
   (10 failures, 29 passes); the same tests pass without `FORCE_COLOR=1`. The original
   note also names bead CLI, plugin panes, and commit- publication warnings from the
   escalated suite.
2. From `sase-m9.1.1.2`: the full visual suite's Artifacts/Beads snapshots fail before
   PNG comparison because `select_entry_target` cannot find `alpha-1` / `alpha-open`.
   Keep this distinct from renderer drift because the failure is fixture/selection
   behavior, not a pixel mismatch.
3. From `sase-m9.1.1.2`: the Models panel effort-picker snapshot has a repeatable local
   111-pixel PNG drift unrelated to shell terminology. Keep this distinct from the
   Artifacts/Beads selection failure.

Do not fix these independent issues as part of `sase-m9.1.1`; only complete their
required task-bead disposition.

## Final phase: Close the epic and finish post-close cleanup

1. Re-run `sase bead show sase-m9.1.1` and every child immediately before close to
   ensure every descendant remains closed and no new note was missed.
2. Close with
   `sase bead close sase-m9.1.1 --note "<detailed verification and integration note>"`.
   The note must cover all three implementation commits, the current-tree tests and
   help/memory/skill checks, the three later non-epic commits and why they required no
   source integration, the retained-lane classification, generated skill deployment, and
   every follow-up disposition. Do not force a successful close. If the command names
   unfinished descendants, finish or reopen them; use `--force` only with a deliberate
   canceled/superseded resolution for genuinely abandoned work.
3. Only after the epic is closed, read `symvision.md` through `/sase_memory_read`, run
   `just symvision` if the recipe is available, and remove expired `sase-m9.1.1`
   whitelist entries plus any unused code it reports. Run `just check` again after
   repository cleanup, and use `just check-full` through `/sase_monitor` if the cleanup
   touches the broadening set or otherwise warrants exhaustive coverage.
4. Set `status: done` in the frontmatter of the epic's canonical plan,
   `plan:202608/shell_taxonomy.md`, using the artifact-resolved path. Confirm the final
   epic state is closed, the plan status is done, and the repository worktree contains
   no unintended changes.
