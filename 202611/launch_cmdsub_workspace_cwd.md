---
tier: tale
size: medium
title:
  Run launch-time macro commands inside the assigned workspace, then finish landing
  sase-1tc
goal:
  For a launch whose workspace is already assigned, launch-time macro `$(cmd)` runs
  inside that workspace again, as it did before efc66a420b. The sase-1tc backlog chain
  then continues with a proposed next epic, and epic sase-1tc closes.
proposed_by: bbugyi200.athena.sase-1tc.land
proposed_with: claude/opus
bead: sase-1tc
create_time: 2026-11-19 15:46:59
status: wip
---

- **PARENT:**
  [202611/backlog_epic_cmdsub_retries_launches.md](https://github.com/sase-org/sase--plans/blob/main/202611/backlog_epic_cmdsub_retries_launches.md)
- **BEAD:**
  [sase-1tc](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1tc/README.md)

# Run Launch-Time Macro Commands Inside the Assigned Workspace, Then Finish Landing sase-1tc

## Context

Epic **sase-1tc** ("Backlog epic 1") is fully implemented except for one regression it
caused. Its plan is `plan:202611/backlog_epic_cmdsub_retries_launches.md`. The land
agent verified the following and recorded the evidence in two notes on sase-1tc: "LANDER
VERIFICATION + FOLLOW-UP TRIAGE" and "BACKLOG CHAIN PREP".

- All five phases are verified and closed, along with their nested child epics.
- The task beads are closed: sase-1sn, sase-1he, sase-1c2, sase-1kp, and sase-1ha.
- Epic-caused CI tasks sase-1th and sase-1tl are closed. This epic's own commits had
  already fixed both.
- Every `PROPOSED FOLLOW-UP:` has been dispositioned. New task sase-1u2 was created.
- `sase bead epic-symbols sase-1tc` reports no entries.
- sase-1tc has no `parent_bead`.

Read both notes first: `sase bead read sase-1tc -r "<why>"`.

**The remaining defect (epic notes #2 and #3).**

- efc66a420b (phase sase-1tc.1.1.2) moved authored `$(cmd)` execution to the source
  boundary. In particular, macro bodies now run their own commands during macro
  expansion.
- At HEAD, `bootstrap_agent_run` in `src/sase/axe/run_agent_runner_bootstrap.py` calls
  `preprocess_prompt_macros(...)` at about line 442. It only calls
  `enter_agent_workspace(...)` afterward, at about line 448.
- The runner subprocess inherits the launcher's cwd. `launch_spawn.py` sets no `cwd=`,
  so that is the TUI's, CLI's, or scheduler's directory.
- So every launch-time command sees an arbitrary directory instead of the agent's
  workspace. Before efc66a420b, these commands ran later, inside the workspace.

The concrete failure: `sase repo path research --ensure` found no workspace marker and
fell back to workspace #0, the primary checkout
(`repo_handler_common.resolve_list_workspace_num`). Research reports then landed where
the commit finalizer never looks. Epic sase-1tj patched only part of this:

- plugin 73b3350 fixed the research macros;
- 43ef590bb6 documented the current behavior in `docs/reference/macros/templating.md`;
- 03a48ab7fc and bd8616817d added an unscanned-writes warning.

The general hazard is still open, and sase-1tj.land routed it back to this epic.

**Chosen fix.**

- When the workspace is already assigned, enter it before macro expansion.
- Deferred-workspace launches (`%wait` launches with `SASE_AGENT_DEFERRED_WORKSPACE`)
  keep running before any workspace is claimed. For them, `launch_executor_workspace.py`
  (~lines 340-343) passes a provisional workspace 0 or workspace 1 directory, so
  entering it would be wrong. That path stays the documented contract.

**Out of scope.** Deferring command execution until after workspace claim and
preparation while preserving source provenance. Directives such as `%name`, `%model`,
and `%wait` must be extracted before the claim, so that redesign would need its own
epic. Do not attempt it here.

## Rules

- Read the `lint_and_test` reference memory before finishing:
  `sase memory read lint_and_test.md -r "<why>"`.
- Read `docs_writing` before editing `docs/`.
- Keep every guard: no skips, xfails, flake-baseline entries, or timeout increases.
- Before editing, run `sase agent list` and check that no live agent owns
  `src/sase/axe/run_agent_runner_bootstrap.py` or `docs/reference/macros/templating.md`.
  At planning time, live epic sase-1tz did not touch either file.
- Do not run `just check-full`.
- Never use `--force` to close anything in this tale.

## Step 1: Enter the assigned workspace before launch-time macro expansion

In `bootstrap_agent_run` (`src/sase/axe/run_agent_runner_bootstrap.py`):

1. Compute `deferred_workspace = bool(os.environ.get("SASE_AGENT_DEFERRED_WORKSPACE"))`
   before the `preprocess_prompt_macros(...)` call.
2. When the launch is not deferred, call
   `enter_agent_workspace(state.workspace_dir, state.workspace_num)` before
   `preprocess_prompt_macros(...)`, and do not call it a second time afterward.
   - Home-mode launches use the home directory. Today they enter it right after
     expansion anyway, so entering early is equivalent; keep them on the early path.
3. For deferred launches, keep today's order exactly: expand, then call
   `enter_agent_workspace` with the provisional values. `_prepare_workspace_and_repos()`
   in `run_agent_runner_launch.py` later claims the real workspace.
4. Audit everything `preprocess_prompt_macros` touches
   (`src/sase/axe/run_agent_runner_setup_prompt.py`) for dependence on the launch cwd:
   - `canonicalize_project_aliases_in_prompt`;
   - `resolve_macro_aliases`;
   - the raw-prompt write into `artifacts_dir`;
   - `write_used_macros`;
   - `capture_revival_inputs`;
   - `process_macro_references`, including macro discovery and any `local_macros_file`.

   Any relative path, or any discovery root derived from cwd, must resolve to the same
   target as before. If one would change, resolve it to an absolute path before the
   `chdir` instead of changing discovery semantics. Record what you checked in the
   commit message.

5. Keep the existing ordering of `print_agent_start_banner`, retry-handoff loading, and
   `_capture_commit_finalizer_baseline`. The baseline must still run after workspace
   entry.

**Tests.** Extend `tests/test_run_agent_runner_bootstrap_baseline.py` or add a focused
sibling module. Keep the tests side-effect-free, using `tmp_path` dirs and monkeypatched
env.

- A non-deferred launch whose prompt uses a macro body with `$(pwd)` expands to the
  assigned workspace directory, even though the process started elsewhere. This test
  must fail on the pre-fix ordering.
- The same check for a top-level human-authored `$(pwd)` that is executed at the launch
  source boundary, if one exists. Confirm which prompt sources execute during
  `process_macro_references` and cover each one.
- A deferred launch (`SASE_AGENT_DEFERRED_WORKSPACE=1`) still expands in the launch cwd
  and still enters the provisional workspace afterward.
- A project-local macro still resolves after the change.

## Step 2: Document the contract

Update the paragraph in `docs/reference/macros/templating.md` that currently reads "In a
macro body, command substitutions run at launch, before the agent's workspace is
claimed...". It should state:

- When the launch already has an assigned workspace, launch-time command substitutions
  run inside that workspace. This happens before workspace preparation syncs it, so
  revision-sensitive commands may see the workspace's previous state.
- `%wait` (deferred-workspace) launches run them before any workspace is claimed, so
  those commands must not depend on cwd.
- Keep the `sase repo open` advice for workspace-relative paths.

If `docs/contribute/llm-invocation.md` describes when launch-time substitution runs,
update it as well.

## Step 3: Verify and integrate

1. Run the focused tests from Step 1, plus `tests/test_cmdsub_source_provenance.py`,
   `tests/test_cmdsub_provenance_landing.py`, `tests/test_cmdsub_guard_nesting.py`, and
   `tests/test_workflow_routes_provenance.py`.
2. Run `just fix`, then `sase tool run check`. Route the check through `/sase_monitor`
   or a prepared-completion monitor if it would outrun the turn, as the `lint_and_test`
   memory describes.
3. Fetch `origin/master` and review the commits after `afb070580a`. Look for any new
   launch path, macro-expansion call site, or second `enter_agent_workspace` caller that
   should follow the same rule. Integrate any you find.
4. Record a note on sase-1tc with the fix commit evidence, the test names, the ToolRun
   id, and the integration result.

## Step 4: Continue the backlog chain (mandatory, owner's standing instruction)

Follow step 3, "Continue the backlog chain", in the "Landing and the next epic" section
of the epic plan (`plan:202611/backlog_epic_cmdsub_retries_launches.md`; read it via the
PLAN path that `sase bead read sase-1tc` prints).

1. **Start from the shortlist.** sase-1tc's "BACKLOG CHAIN PREP" note ranks sase-1sj,
   sase-11d, sase-1el, sase-1g1, and sase-v6, followed by the alternates sase-1ot,
   sase-yn, sase-175, sase-1cg, and sase-11g.
2. **Re-verify each pick at HEAD.**
   - It still reproduces.
   - It is open.
   - It is not named in, or clearly inside the scope of, any in-progress plan or phase
     bead. Check live epic sase-1tz in particular.
   - The owner has not snoozed it.
3. **Handle obsolete picks.** If a pick is verifiably fixed or obsolete, close it with a
   concrete evidence note (commit, `file:line`, or removed symbol) and promote the next
   alternate. Never close a flake because it "passes now".
4. **Recheck the exclusions.**
   - If sase-1ra is still `in_progress`, sase-1r4, sase-1qj, sase-12u, and sase-1jd stay
     with it, and sase-1sh, sase-1si, and sase-1sk stay deferred.
   - If sase-1ra has closed or been canceled, those three become eligible.
5. **Author the next epic with `/sase_plan`**, following the rules in that section:
   - at most 5 strictly sequential phases, each depending on the previous one;
   - at most 5 task beads;
   - a minimal set of decisions, for owner opt-out of feature, memory, or maybe-unneeded
     work;
   - the "Rules for every phase" section, adapted;
   - the whole "Landing and the next epic" section, copied in with its exclusions
     updated.
6. **Propose it.** Validate with `--explain`, then revalidate. Run `sase plan propose`.
   When it reports **Staged**, continue to Step 5 in this same turn. Do not wait for the
   proposal.
7. **If nothing is eligible.** If every remaining open task bead is excluded, do not
   create an epic. Instead, record a note on sase-1tc that lists each remaining bead and
   why it is excluded.

## Step 5 (final): Close out epic sase-1tc

1. Run `sase bead epic-symbols sase-1tc`. For each listed `--epic-symbol` entry, resolve
   it under the Symvision epic-whitelist policy: wire it up, privatize it, add a
   non-test pragma, or delete it. Re-key an entry only when a still-open later bead
   needs it. There were none at planning time.
2. Close the epic: `sase bead close sase-1tc --note "<verification>"`. The note
   summarizes:
   - the land agent's verification and follow-up notes;
   - the cwd fix commit, its tests, and its ToolRun;
   - the integration review;
   - the outcome of the backlog-chain proposal.

   If the close is rejected for leftover epic-symbol entries, finish that cleanup and
   close again. Never use `--force` merely to make the command succeed.

3. Run `just symvision` and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file. That is the PLAN path
   `sase bead read sase-1tc` prints, inside the plans sidecar; it currently says
   `status: wip`.
5. sase-1tc has no `parent_bead`, so the landing ends here. Finish through
   `/sase_final`.
