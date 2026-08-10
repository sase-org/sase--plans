---
tier: tale
title: Auto-close task beads on sase commit with a do-not-close opt-out
goal:
  A successful sase commit closes the committing agent's in-progress task bead in its
  own primary repo, unless -B/--do-not-close-bead is passed, and never closes phase,
  plan, or epic beads or beads touched by linked-repo or sidecar commits.
size: medium
proposed_by: bbugyi200.athena.xa
create_time: 2026-08-10 10:45:53
status: wip
---

# Auto-close task beads on `sase commit`, with a `-B|--do-not-close-bead` opt-out

## Problem

The request: task beads assigned to SASE agents that make commits are sometimes left
open, so `sase commit` should close the agent's task bead automatically unless a new
`-B|--do-not-close-bead` flag is passed.

Two findings from investigating the premise must shape the implementation. Both are
recorded here because they are the reason this plan is narrower than "close the assigned
bead on commit".

### Finding 1 — this behavior existed and was deliberately removed two days ago

Commit `04e4a33b3` (2026-08-08, "fix(commit): stop closing the assigned bead on commit")
removed exactly this feature. Its message records why:

> The before-commit bead hook closed the workspace's assigned bead as soon as it saw a
> commit, without checking whether the agent had finished. A commit is not a completion
> signal: one workspace commits to its primary repo, to linked repos, and to SDD
> sidecars, and none of those say the agent's own deliverable is done. **Two confirmed
> occurrences closed a phase bead mid-flight — one from a linked-repo commit, one from a
> plans-sidecar commit** — leaving a close timestamp and note that described work which
> had not happened yet.

That commit replaced the close with a warning (`_report_unclosed_bead` in
`src/sase/workflows/commit/commit_hooks.py`) and left the docstring on `handle_beads`
stating the commit path never closes a bead in any repo.

So this plan is not adding a new capability; it is **re-introducing a reverted one under
guards that make the two confirmed regressions impossible**. Both regressions involved
(a) a **phase** bead and (b) a commit in a **non-primary repo**. The guards below
exclude both.

### Finding 2 — the current evidence does not show task beads being left open

Cross-referencing every `SASE_BEAD=` footer in this repo's git history against the bead
store (`sase/repos/beads/issues.jsonl`):

- 593 commits carry a bead footer; 563 distinct beads (478 phase, 61 task, 54 plan).
- All 61 task beads referenced by a commit were closed.
- The close landed **1–4 minutes before** the commit that carries the footer, in every
  case — the agent closes the bead, then commits.
- Independently: 71 task beads have reached `in_progress`; all 71 were closed, each by
  the same actor that took it in progress.
  `sase bead list --type task --status in_progress` is currently empty.
- The three task beads currently referenced by a commit and not closed (`sase-ct`,
  `sase-ii`, `sase-iq`) were each closed by their worker and later **reopened via +1**,
  which is the intended `+1` promotion path, not an agent forgetting.

Conclusion: since `04e4a33b3`, the warning-only mechanism appears to be working, and no
recorded task bead was left open by a committing agent. The feature in this plan is
therefore best framed as a **safety net** that removes a manual step, not as a fix for
an observed leak. This is called out so the reviewer can reject the plan on that basis
if they prefer to keep the current warn-only behavior.

If the reviewer wants this anyway — which the request states — the design below is the
one that does not reintroduce `04e4a33b3`.

## Scope decision: the `-B` short flag already exists

`-B` is currently `--bug-id` (`src/sase/main/parser_commit.py:37`), an int option added
by `sase/repos/plans/202604/bug_id_cli_option.md`. The requested
`-B|--do-not-close-bead` therefore collides with it.

Usage evidence from `~/.sase_git_commit.jsonl` (the agent commit path, 7,132 recorded
`sase commit` invocations): `-B` appears **once**. The long `--bug-id` form is untouched
by this change, and the `#pr` xprompt's `bug_id=` named argument sets `$SASE_BUG_ID`
rather than passing the CLI flag, so it is unaffected.

**Decision:** honor the request literally — reassign `-B` to `--do-not-close-bead` and
move `--bug-id` to the free short alias `-b`. Because `-B` becomes `store_true`, a stale
`sase commit -B 12345` now fails loudly with `unrecognized arguments: 12345` rather than
silently changing meaning.

This is a **breaking CLI change** and the landing commit MUST use a `feat!:` header with
a `BREAKING CHANGE:` footer naming the `-B` reassignment and the `-b` replacement.

**Alternative if the reviewer prefers no break:** keep `-B|--bug-id` and give the
opt-out `-b|--do-not-close-bead` instead. Everything else in this plan is unchanged;
only the two letters swap. The implementing agent must not make this substitution on its
own — it is a reviewer decision.

## Goal

After a `sase commit` that actually lands a commit, close the committing agent's
assigned bead automatically **only when every guard below holds**, and print a clear
line saying so. Otherwise fall back to today's warning. Never fail a commit because of
the bead step.

## Design

### Guards (all must hold, evaluated together)

1. **Opt-out not used.** `payload["do_not_close_bead"]` is falsy.
2. **A bead is assigned.** `payload["bead_id"]` is set (it comes from `$SASE_BEAD_ID`
   via `_resolve_env_bead_id` in `src/sase/main/commit_handler.py`).
3. **Method commits.** `self._method` is `create_commit` or `create_pull_request`.
   `create_proposal` already skips `handle_beads` and must keep skipping this.
4. **Bead is a task bead.** `sase bead show <id> --format json` reports
   `issue.issue_type == "task"`. Phase, plan, and epic-tier beads are never auto-closed.
   This directly excludes the phase-bead regression from `04e4a33b3`, and matches
   `sase/memory/sase_beads.md`: "Never close the parent epic bead; its land agent does
   that."
5. **Bead is actively being worked.** `issue.status == "in_progress"`. A launched
   worker's bead is already `in_progress` before it reads its prompt, so this admits
   exactly the intended case and makes re-runs and stray invocations no-ops. `open`,
   `ready`, `claimed`, `snoozed`, and `closed` are all skipped.
6. **Commit is in the agent's own primary repo.** The repo root for `cwd` must not match
   a linked-repo checkout or an SDD sidecar checkout. This excludes the linked-repo and
   plans-sidecar regressions from `04e4a33b3`.

   Implement as an explicit **denylist**, not an allowlist: resolve
   `_get_repo_root(cwd)` and skip when it matches any of
   - the `workspace_dir` values in `SASE_LINKED_REPOS_JSON` (fall back to the legacy
     `SASE_SIBLING_REPOS_JSON`; use `linked_repo_metadata_from_env` in
     `src/sase/_linked_repo_env.py`, and the `SASE_LINKED_REPO_*_DIR` /
     `SASE_LINKED_REPO_*_PRIMARY_DIR` pairs it also exposes), or
   - `$SASE_SDD_DIR`, `$SASE_SDD_PLANS_DIR`, `$SASE_SDD_BEADS_DIR`,
     `$SASE_SDD_RESEARCH_DIR`.

   A denylist is required rather than "repo root == workspace checkout dir" because
   `src/sase/workspace_provider/marker.py` documents that markers are deliberately
   **not** written into the user's primary checkout, so an allowlist would silently
   disable the feature there. Compare with `os.path.realpath` on both sides; ignore
   unset/empty env values.

   If any resolution step raises, treat the repo as non-primary and skip. Failing closed
   is correct: a skipped close costs a warning, a wrong close corrupts bead history.

Any unresolvable value (`sase` not on PATH, non-zero exit, unparseable JSON, payload
with no `issue`) means **skip and warn**, matching the existing `_resolve_bead_status`
behavior.

### Ordering: close after the commit lands, not before

Today `handle_beads` runs **pre-dispatch**
(`src/sase/workflows/commit/workflow.py:133`), which is where the reverted
implementation closed the bead. Closing there closes on an attempt, not a result. The
new close runs **post-dispatch**, as the last step of `_run_tracking_steps`, after
`append_commits_entry` / `final_result_marker` and after `_run_agent_publication_step` —
i.e. only once every other bookkeeping step succeeded.

This is safe with respect to what gets committed: `sase/repos/` is git-ignored (see
`.gitignore:63`), and `sase bead close` commits and pushes the bead store itself
(`close_mutation_commit_message` in `src/sase/bead/mutation_commit.py`), so the close's
own writes never need to be part of the code commit.

`_run_tracking_steps` is shared by the normal and `--resume` paths, so the close is
reached on resume too. Guard it with `"close_bead" not in cp.completed_steps` and append
that marker plus a `checkpoint_save(cp)` after a successful close, exactly like the
neighbouring steps. `CommitCheckpoint.payload` is serialized to JSON verbatim, so
`do_not_close_bead` survives a resume with no dataclass change.

### The close command

```
sase bead close <bead_id> --resolution done --note "<note>"
```

The note must state only what actually happened. Do **not** claim verification — that is
the specific dishonesty `04e4a33b3` called out ("a close timestamp and note that
described work which had not happened yet"). Use text of the form:

> Auto-closed by `sase commit` after `<method>` landed `<short-sha>` ("<subject>"). No
> verification is implied by this note. Reopen with `sase bead open <id>`, or pass
> `-B|--do-not-close-bead` on mid-flight commits.

Resolve `<short-sha>` best-effort from the provider/`git rev-parse --short HEAD`; omit
the sha clause rather than failing if it cannot be read. Do not pass `--reason` and do
not pass `--force` — a task bead with an unclosed descendant must fail the close and
produce the warning, never sweep children closed.

### Reporting

- Auto-close ran: `print_status(...)`, `"success"`, naming the bead and how to undo it
  (`sase bead open <id>`).
- Auto-close was skipped or failed while the bead is still open: keep today's
  `_report_unclosed_bead` warning. Its current wording ("committing does not close it")
  is now conditionally false and must be reworded to say the commit did not close _this_
  bead and why (opt-out used / not a task bead / not in this repo / close failed), still
  naming the `sase bead close` command.
- The close is best-effort throughout: it never changes `RunResult` and never turns a
  successful commit into a failure.

### Avoiding a double status lookup

Factor a single helper, e.g.
`resolve_task_bead_autoclose(payload, cwd) -> AutocloseDecision`, returning the bead id
when every guard passes and otherwise a skip reason string. The pre-dispatch
`handle_beads` uses it to decide whether to warn (it must not warn when the
post-dispatch close is armed); the post-dispatch step uses it to decide whether to
close. Keeping one helper keeps the two call sites from drifting.

## Files to change

Implementation:

- `src/sase/main/parser_commit.py` — move `--bug-id` to `-b`; add
  `-B/--do-not-close-bead` (`action="store_true"`, `dest="do_not_close_bead"`). Per
  `sase/memory/cli_rules.md`, keep the help text scannable and place the two options
  adjacent where `-B/--bug-id` sits today; do not reorder the rest of the parser.
- `src/sase/main/commit_handler.py` — set `payload["do_not_close_bead"] = True` when the
  flag is passed, alongside the existing `bug_id` / `bead_id` payload wiring.
- `src/sase/workflows/commit/commit_hooks.py` — add the autoclose decision helper, the
  repo denylist, the close call, and the reworded skip warning; generalize
  `_resolve_bead_status` to return the whole `issue` dict (status **and** `issue_type`)
  so one `sase bead show` call serves every guard. Update the `handle_beads` docstring,
  which currently asserts the commit path never closes a bead.
- `src/sase/workflows/commit/workflow.py` — call the new close step at the end of
  `_run_tracking_steps`, gated on `cp.completed_steps`.

Docs (all three currently document `-B` as `--bug-id`):

- `docs/commit_workflows.md` — flag table row (~line 108) and the payload-mapping
  sentence (~line 236).
- `docs/configuration.md` — the `sase commit` option row (~line 3201).

Skill source — **minimal, high-value edits only**, per the request. Only two changes in
`src/sase/xprompts/skills/sase_git_commit.md`:

1. Add one bullet to the existing `Flags:` list in step 4:
   `- `-B`: Do not auto-close your assigned task bead; use it for mid-flight commits.`
2. Fix the now-wrong exit-code-1 text, which currently reads "Committing never closes a
   bead — it only syncs the bead store — so re-runs are safe. Close your own bead with
   `sase bead close` when the work is actually done; a commit made while the bead is
   still open prints a reminder, not a close." Replace it with one or two sentences: a
   successful commit auto-closes an `in_progress` **task** bead in this repo (phase and
   plan beads are never auto-closed, and the agent still closes those itself); re-runs
   stay safe because re-closing is a no-op.

Do not add a new section, an example, or a rationale paragraph to the skill — every
token there is loaded into every commit-capable agent's context.

Deployment (from `sase/memory/generated_skills.md`): the chezmoi `SKILL.md` files are
generated, never hand-edited. Land the template change on the canonical branch first,
then run `sase skill init --force` from that clean, merged tree. Preview with
`sase skill init --diff` while iterating. `sase_hg_commit.md` is out of scope — the
request names the git skill, and per that memory Claude and Codex do not install the hg
skill.

## Tests

- `tests/test_commit_cli.py` — `-b/--bug-id` still yields `payload["bug_id"]`; bare `-B`
  yields `payload["do_not_close_bead"] is True`; absent flag omits the key; `-B 12345`
  is rejected by the parser.
- `tests/test_commit_hooks_artifacts.py` (already covers `handle_beads`) — one case per
  guard: task + `in_progress` + primary repo closes; phase bead does not; plan bead does
  not; `open`/`ready`/`claimed`/`closed` task does not; opt-out set does not; repo root
  equal to a `SASE_LINKED_REPOS_JSON` `workspace_dir` does not; repo root equal to
  `$SASE_SDD_PLANS_DIR` does not; unresolvable `sase bead show` does not and warns;
  non-zero close exit warns and does not raise.
- `tests/workflows/test_commit_workflow.py` and `tests/test_commit_workflow_dispatch.py`
  — the close runs only after a successful dispatch, does not run when dispatch fails or
  returns `CONFLICT`, does not run for `create_proposal`, and appends `"close_bead"` to
  `completed_steps`.
- `tests/test_commit_workflow_resume.py` — `--resume` reaches the close step, and a
  checkpoint that already has `"close_bead"` does not close twice.
- `tests/test_sase_git_commit_wrapper.py` — the wrapper forwards `-B` unchanged (it
  already forwards `"$@"`; assert it rather than assume it).

Per `sase/memory/generated_skills.md`, any `sase commit` argument change requires
same-turn updates to in-repo callers/wrappers, the skill files documenting those
arguments, and tests covering both CLI parsing and skill invocation examples — the list
above is that set. Search for any other in-repo caller passing `-B` before landing.

## Verification

- `just install` first (workspace virtualenvs are ephemeral).
- `just check` during development.
- `just check-full` before landing: this touches the commit workflow, which is broad
  enough that the scoped lane should not be trusted alone.
- Manual smoke in a scratch task bead: launch or simulate an agent with `SASE_BEAD_ID`
  set to an `in_progress` task bead, run `sase commit`, confirm the bead closes with the
  generated note; repeat with `-B` and confirm it stays open with the reworded warning.

## Out of scope

- Changing what closes **phase**, **plan**, or **epic** beads. Land agents keep that
  job.
- Auto-closing from linked-repo or SDD-sidecar commits — deliberately excluded; this is
  the `04e4a33b3` regression.
- `sase_hg_commit.md` and any other runtime's skill wording.
- Reordering the rest of the `sase commit` parser to be fully alphabetical.
- Moving any of this into the Rust core. The decision logic is commit-workflow glue that
  shells out to the existing `sase bead` CLI, and both `handle_beads` and the bead CLI
  boundary already live in Python here. If the implementing agent finds itself
  reimplementing bead status or close semantics rather than calling the CLI, stop and
  reconsider against `CLAUDE.md`'s Rust-core boundary rule.
