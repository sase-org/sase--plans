---
tier: tale
title: Write agent commit messages to .sase/commit_message.md
goal:
  The /sase_git_commit skill directs agents to write the temporary commit message file to the git-ignored
  .sase/commit_message.md path, so a leftover message file can no longer fail the post-completion commit finalizer or be
  swept into a whole-repository commit.
create_time: 2026-07-31 08:30:27
status: wip
---

- **PROMPT:** [202607/prompts/commit_message_in_dot_sase.md](prompts/commit_message_in_dot_sase.md)

# Plan: Write agent commit messages to `.sase/commit_message.md`

## Problem

The `/sase_git_commit` skill tells agents to create a commit message file "e.g. `commit_message.md`" at the repository
root. That file is untracked and NOT ignored, so any time it survives the agent's turn the post-completion commit
finalizer sees it as an uncommitted change and fails the agent.

How the finalizer sees it: `build_commit_details()` in `src/sase/commit_instructions.py` calls
`provider.diff_with_untracked()`, whose git implementation (`vcs_diff_with_untracked` in
`src/sase/vcs_provider/plugins/_git_query_ops.py`) runs `git ls-files --others --exclude-standard`. A root
`commit_message.md` is reported, the bounded finalizer passes exhaust, and `failure_message()` in
`src/sase/llm_provider/commit_finalizer_prompting.py` marks the run FAILED.

The file survives the turn in several ordinary situations:

- The agent runs `sase_git_commit` as a background command and its turn ends before the commit finishes.
  `src/sase/main/commit_handler.py` only calls `os.remove(message_file_path)` after `RunResult.OK`, so the file is still
  on disk during the whole commit window.
- The commit fails (exit 1). The handler deliberately preserves the file so the agent can retry with the same `-M` path,
  and prints "Commit message preserved at ...".
- A rebase conflict pauses the commit (exit 2) and the agent finalizes with `sase_git_commit --resume`. The
  `args.resume` branch in `src/sase/main/commit_handler.py` returns before any message-file cleanup, so the file is
  never deleted on that path.

### Evidence: failed agent `q2` (`q2--code`)

Artifacts: `~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202607/31/20260731073415/`

`commit_finalizer_result.json`:

```json
{
  "changed_files": ["commit_message.md"],
  "error": "Commit finalizer failed: uncommitted changes remain after 2 finalizer pass(es) ... commit_message.md.",
  "passes": 2,
  "reason": "dirty_after_max_passes",
  "status": "failed"
}
```

`tool_calls.jsonl` shows the agent wrote `<workspace>/commit_message.md` and then ran
`sase_git_commit -M commit_message.md -f ...`. `commit_result.json` shows that commit actually **succeeded**
(`0002a0590`, now on `master`), and the agent's own pass-2 response ends with "The commit is running in the background;
I'll wait for it to finish". The agent was failed purely because the temporary message file was still on disk when the
finalizer took its final dirty reading. The work was fine; the bookkeeping file broke the run.

### Secondary defect the same change fixes

When an agent omits every `-f` flag, the git dispatch stages with `git add -A`
(`src/sase/vcs_provider/plugins/_git_commit_dispatch.py`). A root `commit_message.md` is untracked and unignored, so a
whole-repository commit will silently commit the temporary message file into the project. Moving it under `.sase/` makes
that impossible, because `git add -A` honors ignore rules.

## Approach

Instruct agents to write the commit message file to `.sase/commit_message.md` instead of the repository root.

`.sase/` is ignored in **every** repository SASE hands to an agent, via per-clone `.git/info/exclude` entries
(`SASE_GIT_INFO_EXCLUDE_PATTERNS = (".sase/", "/sase/repos/")` in `src/sase/workspace_provider/git_exclude.py`):

| Repo kind             | Installer                                |
| --------------------- | ---------------------------------------- |
| agent workspace clone | `src/sase/axe/run_agent_runner_setup.py` |
| linked-repo checkout  | `src/sase/_linked_repo_workspaces.py`    |
| external repo         | `src/sase/main/repo_open_external.py`    |
| SDD sidecar clone     | `src/sase/sdd/_store_link.py`            |

The sase repo additionally tracks `.sase/` in its own `.gitignore`. Both `git ls-files --others --exclude-standard`
(finalizer detection) and `git add -A` (whole-repo staging) honor `.git/info/exclude`, so the move fixes the finalizer
failure and the accidental-staging defect at once, with no runtime code change required to make it work.

This is instruction-only for the primary fix; the CLI already accepts any `-M` path, so no `sase commit` behavior
changes.

## Scope Decisions

- **`/sase_hg_commit` is out of scope.** It carries the same "e.g. `commit_message.md`" wording, but the `.sase/` ignore
  plumbing is git-only — `git_exclude.py` writes `.git/info/exclude` and there is no Mercurial equivalent wired up.
  Changing the hg skill without that plumbing would move the file to a path that is still dirty under `hg status`. File
  a `sase bead create -T task` follow-up proposing the Mercurial equivalent instead of editing that skill here.
- **Do not change `sase commit` / `sase_git_commit` argument handling.** The fix is where agents are told to put the
  file, not what the CLI accepts. Keep `-M` accepting arbitrary paths.
- **Do not add cleanup logic** for a stale `.sase/commit_message.md`. Once ignored, a leftover file is harmless
  workspace-local scratch, and the retry-after-failure behavior documented in the skill depends on it persisting.

## Implementation Steps

### 1. Update the skill source `src/sase/xprompts/skills/sase_git_commit.md`

This is a generated-skill source template; see the "Deployment" section below before running any deploy command.

Three places reference the path. Change all of them to `.sase/commit_message.md`:

- **Step 3 heading paragraph** (currently "Create a file (e.g., `commit_message.md`) containing the commit message.").
  Rewrite so the path is prescriptive rather than an example, and say why:
  - Instruct agents to create the file at `.sase/commit_message.md`, relative to the repository being committed.
  - Instruct them to create the `.sase/` directory first if it does not exist (`mkdir -p .sase`), since linked-repo,
    external-repo, and sidecar checkouts may not have one yet. Do not rely on a file-writing tool auto-creating parents
    — the skill is rendered for every runtime, and per the "Uniform Agent Runtimes" convention it must not assume one
    runtime's tool behavior.
  - State the reason in one sentence: `.sase/` is git-ignored in every SASE-managed checkout, so the temporary message
    file never shows up as an uncommitted change to the post-completion commit finalizer and can never be swept into a
    whole-repository commit.
  - Keep the existing "NEVER mention {{ provider_name }}" sentence and the existing "Do not preemptively stash,
    fast-forward, pull, or hand-sync" sentence intact, including the Jinja conditional
    `{% if provider_name != provider_tool_name %}`.
- **Step 4 command block**: `sase_git_commit -M .sase/commit_message.md -f file1.py -f file2.py`.
- **The `## Example` block**: `sase_git_commit -M .sase/commit_message.md -f src/auth.py -f src/login.py`.

Leave the `-M` flag description ("The file is deleted only after a successful commit. If the command fails, retry with
the same `-M` path; do not recreate the message.") semantically unchanged — it is still accurate.

Markdown in this repo is prettier-formatted with `--prose-wrap=always --print-width=120`; run `just fmt` rather than
hand-wrapping.

### 2. Update `tests/main/test_init_skills_sources.py`

In `test_git_commit_skill_invokes_observable_wrapper`:

- Change `assert "sase_git_commit -M commit_message.md" in body` to
  `assert "sase_git_commit -M .sase/commit_message.md" in body`.
- Replace `assert "sase commit -M commit_message.md" not in body` with `assert "sase commit -M" not in body`. That keeps
  the original intent (the skill must call the wrapper, never raw `sase commit`) without depending on the old filename.
  The body legitimately contains the prose "delegates to `sase commit`", so only the `-M` form is asserted absent.
- Add a regression assertion that the root-relative form is gone: `assert "-M commit_message.md" not in body`.

  **Trap to avoid:** do NOT write `assert "commit_message.md" not in body` — `.sase/commit_message.md` contains that
  substring, so such an assertion can never pass. Anchor on the `-M ` prefix.

- Add an assertion that the skill explains the ignore rationale, anchored on a short stable phrase from the wording
  chosen in step 1 (for example `assert "git-ignored" in body`). Pick the anchor from the text actually written so the
  test and source agree.

### 3. Update `docs/commit_workflows.md`

Two agent-facing examples describe what generated skills do and should match the skill:

- The `sase_git_commit -M commit_message.md -f src/example.py` block → `-M .sase/commit_message.md`.
- The following sentence's low-level equivalent `sase commit -M commit_message.md -f src/example.py -t <method>` →
  `-M .sase/commit_message.md`.

Add one sentence near that block noting that the skill writes the message under `.sase/` because that directory is
git-ignored in every SASE checkout, so the temporary file cannot trip the commit finalizer's dirty check.

Leave the later "CLI Inputs and Internal Payload" examples (`sase commit -M commit_message.md ... -t commit` and the
`-M pr_description.md` PR example) unchanged — that section documents raw CLI flags that accept any path, and rewriting
it would wrongly imply `.sase/` is a CLI requirement.

### 4. Defense in depth: agent-delta bookkeeping filter

`_is_commit_message_bookkeeping_path()` in `src/sase/ace/tui/widgets/prompt_panel/_agent_deltas.py` currently hides a
root `commit_message.md` from displayed agent deltas by exact match on the normalized path. Extend it to also match
`.sase/commit_message.md`, keeping the existing root-path match for older agent runs whose diffs are already recorded.

Implementation note: `_normalized_agent_delta_path()` already strips leading `./` and normalizes separators, so the
change is comparing the normalized path against a two-element frozenset of
`{"commit_message.md", ".sase/commit_message.md"}` rather than a single literal. Keep the helper private and keep
`visible_agent_delta_entries` / `visible_agent_linked_delta_groups` unchanged.

This path is unreachable when the ignore rules are in place (an ignored file cannot appear in `diff_with_untracked`
output), so treat it as insurance for checkouts missing the exclude entry — not as the fix.

Add a test in `tests/ace/tui/widgets/test_agent_deltas.py` mirroring
`test_completed_agent_filters_root_commit_message_from_commit_diffs`, but with the diff paths written as
`a/.sase/commit_message.md` / `b/.sase/commit_message.md`. Assert the entry is filtered out of the header while a
sibling real file (e.g. `src/foo.py`) still renders. Do not delete the existing root-path tests — the old behavior is
still required for historical diffs.

### 5. Verify

```bash
just install    # required: workspaces are ephemeral and deps may be stale
just fmt
just check
```

Targeted runs while iterating:

```bash
.venv/bin/pytest tests/main/test_init_skills_sources.py -k git_commit_skill
.venv/bin/pytest tests/ace/tui/widgets/test_agent_deltas.py
```

Preview the rendered skill without deploying (read-only, no guard applies):

```bash
sase skill init --diff
```

## Deployment (post-land, do NOT run from a dirty tree)

Chezmoi `SKILL.md` files are generated from `src/sase/xprompts/skills/` and the destination is global and shared by
every workspace. Deploying from a dirty or unmerged tree publishes content that exists in no sase commit and reverts
whatever another agent deployed.

Correct order:

1. Iterate with `sase skill init --diff` / `--dry-run` only.
2. Commit the template change and land it on the canonical branch.
3. Only from that clean, merged tree, run `sase skill init --force`, then `chezmoi apply` if it was skipped.

Do not reach for `--allow-dirty`. If `sase skill init --force` refuses, the source is not canonical yet — land it
instead of overriding.

Until deployment happens, running agents keep using the previously deployed skill text, so the fix takes effect for new
agents only after step 3.

## Acceptance Criteria

- `src/sase/xprompts/skills/sase_git_commit.md` prescribes `.sase/commit_message.md`, tells the agent to create the
  `.sase/` directory if missing, and explains that `.sase/` is git-ignored so the file cannot trip the commit finalizer.
  No occurrence of `-M commit_message.md` remains in that file.
- `tests/main/test_init_skills_sources.py` asserts the new path, asserts `-M commit_message.md` is absent, and still
  asserts the skill calls the wrapper rather than raw `sase commit -M`.
- `docs/commit_workflows.md` skill examples use `.sase/commit_message.md` and state the reason.
- `_is_commit_message_bookkeeping_path` matches both the legacy root path and `.sase/commit_message.md`, covered by a
  new test, with the existing root-path tests still passing.
- `just check` passes.
- A follow-up task bead exists proposing the Mercurial (`/sase_hg_commit`) equivalent, noting that it needs `.sase/`
  added to Mercurial ignore handling first.

## Risks and Non-Risks

- **Not a risk:** the CLI accepts arbitrary `-M` paths, so no `sase commit` change is needed and old in-flight agents
  using the root path keep working.
- **Not a risk:** `git clean -fd` in the `#gh` pre-step does not remove ignored files (that needs `-x`), and it runs at
  checkout time, long before an agent writes a message file.
- **Real risk (small):** a git checkout that SASE did not prepare would not have `.sase/` excluded, and the message file
  would then be dirty under a subdirectory instead of the root. Every repo-opening path listed above installs the
  exclude entry, and step 4 keeps the TUI display clean if it ever happens. If the implementing agent finds a
  repo-opening path with no `ensure_sase_git_info_excludes` call, file a task bead rather than expanding this change.
- **Sequencing risk:** the behavior change only lands for agents after the chezmoi deploy in the Deployment section. Do
  not conclude the fix is live based on the repo diff alone.
