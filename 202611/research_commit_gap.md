---
tier: epic
title: Close the research report commit gap
goal: "Every research-macro report lands in a checkout the host commits. A run that
  writes into a checkout the commit finalizer does not scan is told so before it stops.
  The ten reports already stranded in apollo's primary research checkouts get committed.

  "
decisions:
  recovery:
    ask:
      How should the stranded apollo reports be handled once their copies are committed?
    choices:
      commit_and_park:
        Move apollo originals to a backup dir so apollo's fast-forward sync keeps
        working
      commit_only:
        Leave apollo originals for you to delete after landing; sync stays blocked until
        then
    default: commit_and_park
    why: Parking keeps a backup and unblocks sync without deleting anything
    answer: commit_and_park
  artifact_create_guard:
    ask:
      How should sase artifact create -p react to a source in a checkout the host will
      not commit?
    choices:
      warn: Register as today and print one stderr warning naming the fix
      refuse: Exit nonzero and register nothing until the file is moved
    default: warn
    why: The finalize-time check is the real safety net; a warning keeps the snapshot
    answer: warn
phases:
  - id: research-macro-paths
    title: Resolve research paths at agent run time
    depends_on: []
    size: small
    description:
      "research-macro-paths: make the five research macros resolve the research checkout
      through an agent-run sase repo open research instead of a launch-time sase repo
      path substitution; update docs, add a regression test, and note the general
      launch-time cwd hazard."
  - id: unscanned-write-guard
    title: Flag writes the commit finalizer cannot see
    depends_on: []
    size: medium
    description:
      "unscanned-write-guard: add a shared detector for run writes and registered
      artifact sources that sit in checkouts the commit finalizer does not scan, and
      surface it in sase final context, sase artifact create, and the /sase_final and
      /sase_repo skills."
  - id: stray-report-recovery
    title: Recover stranded research reports from apollo
    depends_on:
      - research-macro-paths
    size: small
    description:
      "stray-report-recovery: copy the untracked reports from apollo's primary research
      checkouts into opened clones so the host commits them, then handle the originals
      per the recovery decision."
proposed_by: bbugyi200.athena.1dz
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-18 11:26:24
status: wip
---

- **PROMPT:**
  [prompts/202611/research_commit_gap.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/research_commit_gap.md)

# Plan: Close the research report commit gap

## Background

The research report `research:202611/research_report_finalized_without_commit.md`
explains why agent `ex` finished with `success` but never committed its report. I
re-checked each claim against the code at master and against the live machines:

- **Order of work.** The runner bootstrap expands macros (`preprocess_prompt_macros` in
  `src/sase/axe/run_agent_runner_bootstrap.py`) before `enter_agent_workspace`. For a
  deferred-workspace launch, the real workspace is claimed even later, in
  `_prepare_workspace_and_repos` (`src/sase/axe/run_agent_runner_launch.py`). That
  happens after dependency and runner-slot admission.
- **When commands run.** Since `efc66a420b` (phase `sase-1tc.1.1.2`),
  `expand_single_macro` (`src/sase/macro/_processor_render.py`) runs the command
  substitutions written in a macro body during expansion, for human launches. They run
  in the runner's cwd from before the workspace exists.
- **Where `sase repo path` points.** It infers the workspace from the cwd's checkout
  marker. With no marker it falls back to workspace 0, the primary checkout
  (`resolve_list_workspace_num` in `src/sase/main/repo_handler_common.py`).
- **What the commit finalizer scans.** `collect_dirty_state`
  (`src/sase/llm_provider/commit_finalizer_state/__init__.py`) scans the workspace,
  configured sibling targets, repos this run opened, and the SDD and agents stores. The
  research sidecar is scanned only when the run opened it. So a write into the primary
  research checkout is invisible, and `build_context_requirements` returns
  `not_triggered`.
- **Affected macros.** Five sase-research-artifacts macros compute the research checkout
  at launch with a `sase repo path research --ensure` command substitution:
  `research.md`, `research_swarm.md`, `research_more.md`, `research_audio.md` and
  `research_diagrams.md`, all under `src/sase_research_artifacts/xprompts/`.
- **Stranded reports.** A read-only check on apollo found 10 untracked reports in the
  primary research checkouts. The report checked only sase and found one.
  - sase (1): `202611/overnight_backlog_epic_chains_audit.md`
  - bob-cli (7): `202611/daily_aim_for_daily_notes__cld.md`,
    `daily_commitments_design__mus.md`,
    `daily_highlight_and_commitments_design__gem.md`, `record_album_nav__gem.md`,
    `record_card_vs_schema_field_jumps.md`, `record_prev_next_walk__mus.md` and
    `record_walk_prev_next_links__cld.md`
  - tailnet (2): `202611/recording_keystroke_overlay__cld.md` and
    `screen_recording_keystroke_overlay__gem.md`

  The bob-cli and tailnet names are also under `202611/`. Athena's primary research
  checkouts are clean.

### Which of the report's fixes are valid

1. **Fix the path: valid.** Its second option, having `sase repo path` take the runner's
   workspace number, cannot work for deferred launches, because no workspace number
   exists at expansion time. Delaying all authored commands until after the claim would
   rework the source-boundary security model of the active `sase-1tc` epic. So this epic
   fixes the five macros, the only definitions that compute a workspace-relative path at
   launch, and records the general hazard as a follow-up.
2. **Safety net at finalize: valid.** It is built as a model-visible advisory with an
   early warning. The host does not commit into the foreign checkout, for the reasons
   under "Rejected alternatives".
3. **Skill rule conflict: valid.** It needs a wording change in `/sase_repo` and
   `/sase_final`.
4. **Cleanup: valid,** and larger than the report knew: 10 files across 3 projects.

## Phase research-macro-paths: Resolve research paths at agent run time

Work in the linked plugin repo (`sase repo open sase-research-artifacts`; read its
`AGENTS.md`).

1. In `research.md`, replace the launch-time research path in all three branches
   (`report_target`, `suffix`, default) with a run-time step. The agent first runs
   `sase repo open research -r "<reason>"` and treats the printed path as the research
   checkout, then writes under `<research checkout>/<YYYYMM>/`.
   - Keep the launch-time `date +%Y%m` substitution for the month, since it does not
     depend on the cwd.
   - Keep the `report_target` and `suffix` rules and the no-overwrite rules as they are.
   - In the registration paragraph, say the absolute path must lie under the opened
     checkout.
2. In `research_swarm.md`, the "Month directory" block becomes
   `<research checkout>/<YYYYMM>`, created if missing. `<research checkout>` is the path
   `sase repo open research` printed in step 1, which already opens the research repo
   with `/sase_repo`.
3. In `research_more.md`, read `README.md` conventions from the opened checkout, and
   edit the report at its repo-relative path inside that checkout, never in another
   checkout.
4. In `research_audio.md` and `research_diagrams.md`, define the research checkout as
   the path `sase repo open research` prints.
5. Update the `#research` entry in `docs/macros.md` to match.
6. Add a regression test, such as `tests/test_research_paths.py`, that reads every
   packaged xprompt source's raw text. It asserts:
   - no source contains a `sase repo path` command substitution;
   - `research.md` and `research_swarm.md` mention `sase repo open research`.
7. Run `sase tool run check` in the plugin repo.
8. In sase, add a short note where the docs explain command substitution in macro bodies
   (`rg -n "command substitution" docs`; read the `docs_writing` memory note first). The
   note says these commands run at launch, before the agent's workspace is claimed, so
   they must not depend on the cwd; workspace-relative paths should be left for the
   agent to resolve with `sase repo open`.
9. Record a `PROPOSED FOLLOW-UP:` note on this phase's bead about the general hazard.
   Since `efc66a420b`, any cwd-dependent command in a macro body (`sase repo path`,
   `git rev-parse`, `pwd`) resolves against the runner's cwd from before the workspace
   claim. Options to raise with `sase-1tc`: have bootstrap enter the known workspace
   before expansion for non-deferred launches, or delay running authored commands until
   after the claim.

## Phase unscanned-write-guard: Flag writes the commit finalizer cannot see

Read the `lint_and_test` and `generated_skills` memory notes before starting.

1. **Scan-root helper.** Move the target enumeration in `collect_dirty_state` into one
   function that makes no status calls, for example
   `commit_scan_roots(project_dir, artifact_root) -> tuple[str, ...]` in
   `src/sase/llm_provider/commit_finalizer_state/`. It returns these paths, normalized:
   - the main project dir;
   - configured sibling target dirs;
   - opened linked workspace dirs, with name-only opens resolved through sibling targets
     exactly as `_blocking_sibling_candidates` does;
   - opened external workspace dirs;
   - known SDD and agents-archive paths.

   `collect_dirty_state` keeps its current scan order, dedupe and output.

2. **Detector.** Add a new `_unscanned_writes.py` in the same package with
   `find_unscanned_writes(project_dir, artifact_root, *, extra_paths=(), include_run_paths=True)`.
   - **Candidates.** Take the absolute paths from `written_paths_from_tool_calls`; skip
     relative entries, because they are workspace-relative. Add the `source_path` of
     explicit artifacts this run registered (`list_explicit_artifact_files` from
     `sase.core.artifact_file_facade`), skipping continuation-portable labels as
     `_completion_explicit_artifact_paths` does. Add `extra_paths`. Dedupe and cap at
     `MAX_TOOL_CALL_PATHS`.

     The artifact source is the signal that catches non-Claude providers, because
     `tool_calls.jsonl` records only Claude's Edit, Write and NotebookEdit calls.

   - **Checkout check.** For each existing file, find its git toplevel with
     `git -C <parent> rev-parse --show-toplevel`, using a short timeout and the
     non-interactive git env. Skip it when it is not in a work tree, or when its
     normalized toplevel is a scan root. Compare the toplevels for equality, not by path
     prefix: nested `sase/repos/<name>` clones are separate work trees that the main
     scan excludes.
   - **Dirty check.** Keep only paths that
     `git status --porcelain --untracked-files=all -- <path>` reports as untracked or
     modified. Ignored files never appear.
   - **Suggested fix.** Match the toplevel against the project's repo inventory, reusing
     the `repo_handler_common` and `RepoRecord.clones` helpers:
     - this run's own clone of a known repo: action `open` (`sase repo open <name>`
       registers it; nothing moves);
     - another clone of a known repo, such as the primary or another workspace's clone:
       action `open_and_move` (open `<name>` and move the file to the same repo-relative
       path under the printed checkout);
     - anything else: action `relocate_or_discard` (open the repo with `/sase_repo` and
       move the file there, or delete it if it was scratch).
   - **Entry fields.** Each entry carries `path`, `checkout`, `repo_relative_path`,
     `status` (`untracked` or `modified`), `repo_name` (nullable), `action` and a
     one-sentence `instruction`.
   - **Fail-open.** An error or timeout drops that candidate. The function never raises.

3. **Final context.** In `_context_payload`
   (`src/sase/finalizers/_declaration_context.py`), add a top-level `unscanned_writes`
   list. It is always present and may be empty. It is advisory only: the context wire,
   requirements, obligations and digests do not change.
4. **`/sase_final` skill** (`src/sase/macros/skills/sase_final.md`). Step 2 becomes:
   - if `unscanned_writes` is non-empty, follow each entry's `instruction`, then repeat
     step 1;
   - stop only when `submission_required` is false and `unscanned_writes` is empty;
   - if an entry must stay where it is (for example, the user asked for that location),
     say so in the final response.
5. **`/sase_repo` skill** (`src/sase/macros/skills/sase_repo.md`). After "Use that
   printed path as the only path…", add a rule. When a prompt or other instruction names
   a path inside a different checkout of a repo you opened (its primary checkout or
   another workspace's clone), use the same repo-relative path under the printed path.
   The host commits only your workspace and the checkouts you opened.
6. **`sase artifact create -p`** (`src/sase/artifact_cli/create.py`). Inside an agent
   run (artifact root found from the environment), check only the source path, with
   `extra_paths=(source,)` and `include_run_paths=False`. Outside an agent run, nothing
   changes.

   > [!decision] artifact_create_guard = warn Register the artifact as today, then print
   > one stderr warning carrying the entry's `instruction`. Exit status is unchanged.

   > [!decision] artifact_create_guard = refuse Run the check before storing anything.
   > On a hit, print the entry's `instruction`, exit 2, and register nothing. The
   > research macro already tells agents to report a failed registration.

7. **Docs.** Update the docs that describe final-context fields and
   `sase artifact create`. Find them with
   `rg -n "repository_evidence|artifact create" docs`, and follow the `docs_writing`
   memory note.
8. **Tests** (pytest with temporary git repos and no network). Detector cases:
   - a write into an unopened second checkout is flagged `open_and_move`, with the right
     repo-relative path;
   - the same write is not flagged once that checkout is recorded as opened;
   - a file in the main workspace is not flagged;
   - an unchanged tracked file, an ignored file and a non-git path are not flagged;
   - an unopened git repo nested under the workspace dir is flagged `open`;
   - a git error or timeout drops the candidate without raising;
   - an explicit-artifact source is flagged even when there are no tool-call rows.

   Other cases:
   - the final-context payload carries `unscanned_writes`, and requirements, obligations
     and digests are unchanged;
   - `sase artifact create` follows the decision inside an agent run and is unchanged
     outside one;
   - tests that pin skill text still pass (`rg -n "submission_required" tests`).

9. Run checks as the `lint_and_test` note directs.
10. Do not run `sase skill init --force` from the phase workspace. Generated skills
    deploy only from a landed tree, so say in the final response that a redeploy is
    needed after landing.

### Rejected alternatives

- **Host commits the stray file into the checkout it found.** That checkout is usually
  the user's primary checkout. It can hold the user's own edits and is kept current by
  fast-forward sync, and host commits there go against the rule that machine writes stay
  off the primary.
- **A new obligation kind that makes `submission_required` true.** It changes the
  finalizer wire schema just to carry an advisory.

## Phase stray-report-recovery: Recover stranded research reports from apollo

Read the `tailnet` memory note and use `ssh apollo`, falling back to `apollo-do`. If
apollo is unreachable, stop and report.

1. **Inventory (read-only).** List every project with a `research` sidecar from
   `sase repo list --all --json`. On apollo, run
   `git -C <same home-relative path> status --porcelain --untracked-files=all` for each
   and collect untracked files under `20*/`. Skip paths that do not exist on apollo.
   Record each checkout's `origin` URL. Expect the 10 files listed in Background, and
   recover any new ones too.
2. **Open a committable checkout per repo.** For sase, run
   `sase repo open research -r "<reason>"`. For other projects, run
   `sase repo open gh:<owner>/<repo>` with the owner and repo from the origin URL; for
   example, bob-cli's research repo is `bobs-org/bob-cli--research`. Use only the
   printed paths.
3. **Copy.** Copy each stray file to the same repo-relative path under the printed
   checkout with `scp apollo:<abs> <printed>/<rel>`, creating parent dirs. Never
   overwrite: if the target exists, compare sha256. Identical means nothing to copy.
   Different means keep the existing file, skip the copy, and list the collision.
4. **Verify.** Check that each copy's sha256 matches the apollo original. Leave the
   report contents as they are. Do not re-register artifacts: `research:` refs resolve
   by repo-relative path once the files are committed.
5. **Handle the originals.**

   > [!decision] recovery = commit_and_park On apollo, `mv` (never `rm`) each verified
   > original to
   > `~/.local/state/sase/recovered-research/<project>/<repo-relative path>`. This stops
   > the untracked files from blocking apollo's fast-forward sync once the commits land.
   > Leave collisions in place and report them.

   > [!decision] recovery = commit_only Leave the originals on apollo. In the final
   > response, list the exact `rm` commands to run after the commits land. Warn that
   > fast-forward sync of those checkouts fails until then.

6. **Check apollo's installed plugin.** On apollo, check whether the installed plugin
   already has the macro fix: run `sase macro show research --format raw` and look for
   `sase repo open research`. If it does not, say so, because new strays can still
   appear there.
7. **Wrap up.** The host commits each opened checkout when the turn ends. The final
   response lists, per repo: files recovered, collisions, and where the originals went.

## Verification

- Plugin: the `sase tool run check` run passes, and the new regression test fails if a
  launch-time `sase repo path` substitution returns to any research macro.
- sase: the detector, final-context and artifact-create tests pass, and existing
  finalizer and skill tests still pass.
- End to end, after landing, redeploying the skills and reinstalling the plugin: a human
  launch of the research macro opens `research`, writes into the opened clone, and gets
  a host commit of the report. A write into another checkout shows up in
  `sase final context` under `unscanned_writes`.
- Recovery: each repo's opened checkout has the recovered files in the host commit, and
  `git status` on apollo's primary research checkouts no longer lists them (with
  `commit_and_park`).
