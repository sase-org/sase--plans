- **PLAN:** [../202608/task_bead_default_priority.md](../task_bead_default_priority.md)

 
%xprompts_enabled:false
# Previous Conversations

You are forking from 1 prior source. Source sections are independent parents, and section order carries no priority. Members inside an agent family section are sequential: each member continued the previous member's work. Carry forward relevant goals, constraints, decisions, and unfinished work with attribution when it matters. The New Query is the active request and takes precedence over conflicting source instructions.

## Source 1 of 1 — agent family `r0`

- **Members shown:** 2 of 2 (sequential chain, oldest first)

Family members ran as one sequential chain: each member continued the previous member's work, and the last member reflects the family's final state. These are transcripts of prior agents' conversations, not your own — attribute decisions to the named member when it matters.

### Member 1 of 2 — agent `r0--plan`

- **Outcome:** `completed` · **Model:** `codex/gpt-5.6-sol` · **Launch:** `20260801080043`
- **Transcript:** `~/.sase/chats/202608/gh_sase_org__sase-ace_run-r0__plan-260801_080043.md`

**User:**

Can you help me stop adding `%wait(priority=15)` to epic phase / lander agent prompts? Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.

**Assistant:**

*Plan submitted for review.*

**Plan file:** `/home/bryan/.sase/plans/202608/default_epic_agent_priority.md`

> # Plan: Use Default Runner Priority for Epic Agents
> ## Current behavior and scope
> `sase bead work` renders each epic phase and the final lander as separate prompt segments. Phase segments already rely
> only on their authored agent/bead dependency waits: the bundled `bd/work_phase_bead` xprompt does not set a runner-slot
> priority. The bundled `bd/land_epic` xprompt still contributes `%wait(priority=15)` when its segment is expanded, so the
> lander is assigned an explicit, worse-than-default queue priority in addition to the genuine waits that the epic
> renderer adds for every phase agent and phase bead.
> Remove that lander-only priority override so both kinds of epic worker use the normal implicit priority (`10`) when they
> become eligible. Do not remove or weaken the phase DAG waits, the lander's waits on all launched phases and phase beads,
> runner-slot admission itself, or support for user/plugin xprompt overrides. Keep the separate `bd/work_task` priority

*See full plan file for details.*

### Member 2 of 2 — agent `r0--code`

- **Outcome:** `completed` · **Model:** `codex/gpt-5.5` · **Launch:** `20260801080527`
- **Transcript:** `~/.sase/chats/202608/gh_sase_org__sase-ace_run-r0__code-260801_080043.md`

**User:**

@sase/repos/plans/202608/default_epic_agent_priority.md

The above plan has been reviewed and approved. Implement it now.

**Assistant:**

I’ll read the SASE repo-access instructions first because the plan path points at the plans sidecar, then I’ll inspect the approved plan and map it to the current codebase.

I’m checking the registered repository inventory so the plan sidecar is opened through SASE’s audited path instead of guessing a filesystem location.

The plans sidecar is registered as `plans` and already materialized in this workspace. I’m opening it with an audit reason, then I’ll read the specific approved plan.

The sidecar open updated the plans checkout to `origin/main`; I’m reading the approved plan now and then I’ll inspect the code paths it calls out.

The change is narrowly in bundled xprompts, tests, and docs. Because this touches xprompt directives, I need to review the SASE xprompts memory through the required memory-read workflow before editing.

I’m using the audited memory read for `xprompts.md` now; after that I’ll inspect the exact templates and tests named by the plan.

The memory note confirms `%wait` directives are stripped launch metadata and that xprompt discovery can be overridden by project/user/plugin definitions. I’m now reading the bundled definitions, tests, and docs in parallel so the edits stay scoped.

The old priority assertion is exactly where the plan expected it. I’m pulling the surrounding YAML and doc sections next so the patch preserves the surrounding wording and indentation.

The YAML confirms only `bd/land_epic` and `bd/work_task` carry the explicit priority, and the docs have two user-facing references to update. Before editing the test, I’m checking the directive object shape so the new assertions verify authored waits precisely.

The directive parser reports `None` for absent priority and separate lists for agent/bead waits, which is enough for tight regression tests. I’m going to patch the YAML, retarget the test around epic versus task behavior, and update the two docs references.

One detail I’m checking before the patch: where `sase bead work` injects phase and lander dependencies. That decides whether the test should assert only “no priority” or “no authored wait directives at all” for the built-in template bodies.

The renderer appends separate `%wait` lines around the xprompt references; the built-in bodies themselves do not need any wait directive for epic phase or lander prompts. I’m applying the scoped edits now.

I found the docs table that lists the built-in xprompt references, plus a separate tag table that just describes semantic roles. I’m updating only the priority-facing wording so the role docs stay concise.

I’m editing the built-in config, test expectations, and priority wording together now so the behavior and docs move in lockstep.

The first patch is in. I’m checking for stale `land_epic` priority references and reviewing the edited test block before running the focused tests.

The grep shows `priority=15` now only belongs to `bd/work_task`, as intended. I spotted one formatting wrinkle in the test and a small docs wording improvement, so I’m cleaning those before running tests.

Edits are ready for verification. Per the repo instructions I’m refreshing the editable install before running the focused pytest file, then I’ll run the required `just check`.

`just install` is still compiling the local Rust binding. It emitted an existing version-window warning for the linked `sase-core` checkout, but it is continuing with the dev build.

The install is still in the Rust build step. Once it finishes I’ll run the focused `tests/test_bead_xprompt_tags.py` test file first so failures are localized.

No new output yet; the build is still active. I’m leaving it running rather than starting parallel validation against a half-installed environment.

The build reached the Python binding crate. After install completes, I’ll run the exact focused test, then the full required check.

Still compiling without errors. I’m polling the same install process until it exits so we don’t leave a background session running.

The install is unusually long but still alive. I’ll keep waiting; there’s no actionable failure yet.

`just install` completed successfully. I’m running the focused bead xprompt test file now, which covers the new no-authored-wait regression and the unchanged task priority behavior.

The focused test file passed: 14 tests. I’m moving on to the repository-required `just check`; if it reports unrelated failures I’ll separate them from anything caused by this patch.

`just check` stopped at Python formatting only: ruff wants the long directive-extraction call wrapped. I’m applying the formatter to that test file, then I’ll rerun the focused test and `just check`.

The formatter only touched the test wrapping. I’m rerunning the focused file now before the full check.

The focused tests still pass after formatting. I’m rerunning `just check`; this will cover formatting, lint/type checks, and the configured test suite.

`just check` advanced past Python formatting and stopped on Markdown formatting in `docs/xprompt.md`. I’m running Prettier on the two docs files I touched, then I’ll verify the diff before rerunning the check.

Prettier only changed `docs/xprompt.md` and left the troubleshooting page unchanged. I’m checking the diff, then I’ll rerun the required check.

The diff is scoped to the intended YAML, test, and docs edits. I’m rerunning `just check` now after applying Python and Markdown formatting.

`just check` has passed formatting, keep-sorted, and ruff. It’s still running the later checks, so I’m polling the same process until it completes.

The lint and static-analysis stages are green so far, including mypy and symvision. The command has moved into the test portion.

`just check` now fails only in SASE validation: `init skills --check` wants to overwrite five provider `sase_gate` skill files in the chezmoi-managed skill outputs. That is outside this patch’s scope and the code/doc/test checks before it passed, so I’m recording it as discovered follow-up work per the repo instructions.

I reviewed the bead rules. I’m creating a ready task bead for the pre-existing `init skills --check` drift rather than modifying the chezmoi-managed skill files as part of this unrelated xprompt change.

The bead create command is running and emitted the same linked-core version warning. I’m waiting for the bead ID before marking it ready.

The first bead-create attempt used shell backticks in the description, so zsh treated parts of the description as command substitutions. I’m checking whether it created anything; if not, I’ll rerun with a safely quoted description.

The bead was created as `sase-bf`, but I need to inspect and likely repair its description because the shell expansion stripped some literal command text.

The created bead description captured the expanded command output, so I’m replacing it with a concise, literal description and then I’ll mark the bead ready.

I’m updating `sase-bf` in two steps: first correcting the description while it is still open, then setting it to `ready` for triage.

The bead description is corrected. I’m marking `sase-bf` ready now so it is available for task triage.

The status update is running; I’m waiting for the confirmation so the follow-up is fully recorded.

The follow-up bead is ready as `sase-bf`. I’m doing a final status and diff check now so the close-out can clearly separate implemented changes from the unrelated validation blocker.

The final grep shows `priority=15` remains only for `bd/work_task`; `bd/land_epic` is now documented as default-priority. I’m checking the bead sidecar status and a concise diff stat before closing out.

Implemented the approved plan.

Changed:
- [src/sase/default_config.yml](/home/bryan/.local/state/sase/workspaces/sase-org/sase/sase_23/src/sase/default_config.yml): removed `%wait(priority=15)` from bundled `bd/land_epic`.
- [tests/test_bead_xprompt_tags.py](/home/bryan/.local/state/sase/workspaces/sase-org/sase/sase_23/tests/test_bead_xprompt_tags.py): added directive-extraction regression coverage for default-priority epic phase/lander prompts, while keeping `bd/work_task` priority-15 coverage.
- [docs/xprompt.md](/home/bryan/.local/state/sase/workspaces/sase-org/sase/sase_23/docs/xprompt.md) and [docs/troubleshooting/runner-slots.md](/home/bryan/.local/state/sase/workspaces/sase-org/sase/sase_23/docs/troubleshooting/runner-slots.md): updated priority guidance.

Verification:
- `just install` completed.
- `.venv/bin/python -m pytest tests/test_bead_xprompt_tags.py` passed: 14 tests.
- `just check` passed fmt, ruff, mypy, symvision, and other lint stages, then failed in unrelated SASE validation because `init skills --check` reports stale generated `sase_gate` provider skill files. I filed ready follow-up bead `sase-bf` for that drift.

I made the listed changes in this session, so I’m going to use the required SASE commit workflow now and include exactly those four files.

I’ve recorded the commit skill usage. Next I’m checking the working tree and diff so the commit message only describes the files being staged.

The diff is scoped to the approved xprompt priority change, its regression test, and docs. I’m preparing a conventional commit message now, then I’ll run `sase_git_commit` with one `-f` per changed file.

I’m writing the commit message file under `.sase/`, which is ignored by the repo and used only by the SASE commit wrapper.

The message is ready. I’m invoking the SASE commit wrapper now with the four files from the finalizer list.

The commit wrapper has created the commit and is still finishing its post-commit bookkeeping. I’m waiting for the command to exit before verifying branch status.

No failure output from the commit process; it’s still active. I’m continuing to poll the same session so no background command is left hanging.

The wrapper still hasn’t exited after reporting successful commit creation. I’m keeping the session attached and waiting for final completion before running status checks.

The commit command appears to be stuck in quiet post-commit work after the provider success message. I’m going to inspect its process state from another shell while keeping the original session alive.

The active child is `sase commit`; it’s CPU-running rather than waiting on a prompt. I’ll give it another interval, then verify whether the repo is already clean and whether the wrapper can be allowed to finish normally.

It’s still running silently. I’m checking repository state and the process stack indicators now so I can tell whether this is bookkeeping still in progress or a stuck wrapper after a completed commit.

The commit is present at `origin/master`, and `git status` is clean, but the wrapper child is spinning at high CPU. I’m going to capture a stack/process clue before stopping that already-completed command.

The spinning process is holding a SASE store write lock under the plans sidecar. I found `py-spy`, so I’m taking a Python stack dump to identify the loop before terminating or recovering it.

The stack is in SASE’s commit publication step rebuilding plan association headers, not in git itself. Since the commit is already at `origin/master` and the main worktree is clean, I’ll wait one more short interval for normal exit; if it keeps spinning, I’ll stop that stuck post-commit process and report it plainly.

It’s still spinning in the same post-commit path, so I’m stopping the already-completed wrapper process now. After that I’ll re-check that the branch is clean and not ahead of upstream.

The child process received TERM. I’m polling the original wrapper session and then I’ll verify git state.

The wrapper has exited after termination. I’m verifying the final branch state and whether the temporary message file was cleaned up.

The commit is at both `HEAD` and `origin/master`, and the repo is clean. The wrapper left its ignored message file behind because it was terminated after push; I’m removing that temporary file and confirming no commit process remains.

The only remaining `sase_git_commit` processes belong to a different commit operation with different files, so I’m leaving them alone. I’m checking the original PIDs are gone and then I’ll close this out.

Committed the changes as:

`8b6f8fdfe fix: keep epic bead agents at default priority`

`master` is clean and aligned with `origin/master`.

Note: `sase_git_commit` created and pushed the commit, then spun in a post-commit plan-header publication step. I terminated that stuck process after verifying the commit was at `origin/master` and the worktree was clean.

---

%xprompts_enabled:true
# New Query

 Can you help me make sure we removed this from task bead worker agent prompts too? If not, use your /sase_plan skill to plan the appropriate changes.
