---
tier: tale
title: Make ACE post-write init actions non-interactive so they cannot hang the TUI
goal:
  Saving an xprompt memory note or skill source from ACE runs `sase memory init` / `sase
  skill init` as a fully non-interactive background task that either completes or fails
  with a visible error, and can never block on stdin or steal the TUI's terminal.
proposed_by: bbugyi200.athena.wo
create_time: 2026-08-09 13:53:58
status: wip
---

# Plan: Make ACE post-write init actions non-interactive

## Problem

Saving an xprompt memory note through the new xprompt target mode (epic `sase-hp`)
offers a `sase memory init` post-write action. Selecting it hangs the ACE TUI: the
terminal stops responding to keystrokes and the tracked task never finishes.

## Diagnosis

### The `--yes` hypothesis is denied

`sase memory init` has no `--yes` option. Its complete option set is `-c/--check`,
`-d/--diff`, `-M/--enable-project-memory`, `-m/--message`, `-C/--no-commit`
(`src/sase/main/parser_memory.py`, mirrored for the `sase init memory` alias in
`src/sase/main/parser_init.py`). Verified directly:

```
$ sase memory init --yes
sase: error: unrecognized arguments: --yes
```

`-y/--yes` exists only on the top-level `sase init` parser
(`src/sase/main/parser_init.py`), where it suppresses that command's generic "run this
initializer?" confirmations. It would not have helped even there: `run_init_apply` still
forwards the real `sys.stdin` as `_init_stdin` (`src/sase/main/init_onboarding.py`), and
the prompt that actually hangs consults only `--message` and stdin TTY-ness. Adding
`--yes` to the invocation would have turned the hang into an argparse error (exit 2),
not a working action.

### The actual root cause

`_run_post_write_action_sync` in
`src/sase/ace/tui/actions/agent_workflow/_prompt_bar_save_xprompt_git.py` runs the
selected action as:

```python
subprocess.run(list(offer.command), capture_output=True, text=True, check=False)
```

`capture_output=True` redirects **stdout and stderr only**. Stdin is inherited, so the
child gets ACE's terminal on fd 0. Confirmed empirically: a child spawned with
`capture_output=True` from a parent whose stdin is a pty reports
`sys.stdin.isatty() == True`; the same child spawned with `stdin=subprocess.DEVNULL`
reports `False`.

Both init commands gate their interactive prompts on exactly that check, so in the child
the gate opens:

1. **Memory.** `run_init_memory` captures pre-init git state, then
   `deploy_to_project_repo` computes `fold_dirty = memory_dirty + source_dirty`
   (`src/sase/main/init_memory/project_deploy.py`). The memory note ACE just saved under
   `sase/memory/` is dirty at capture time, so `fold_dirty` is **always** non-empty on
   this path. With no `--message`, `_resolve_fold_commit_message` checks
   `stdin_is_tty()`, sees `True`, prints a Rich panel to the captured stderr
   (invisible), and calls `input()`. That blocks forever.
2. **Skills.** `run_init_skills` sets `is_tty = sys.stdin.isatty()`
   (`src/sase/main/init_skills_handler.py`) and, for every existing generated target
   whose content changed, calls `_prompt_overwrite` -> `input()` unless `--force` was
   passed. Same blocking `input()`.

Two consequences make it present as a TUI hang rather than a stuck task:

- The blocked child is reading the same TTY that Textual's input reader is reading.
  Keystrokes are split nondeterministically between them, so the TUI appears frozen.
- `subprocess.run` has no timeout, so the tracked worker never completes. The
  `xprompt-memory-init` dedup key then rejects every retry with "Another sase memory
  init is already running."

### The mistaken belief that produced this

`apply_chezmoi` in `src/sase/config/targets.py` documents the incorrect assumption
explicitly: "these run as captured subprocesses with no interactive stdin". They are
not. The same latent exposure exists in `run_git_commit_push_sync`, where
`git pull --rebase` / `git push` can block on a credential or passphrase prompt;
`apply_chezmoi` is currently masked only by its `--force` flag.

### Why the skill path is also broken, differently

`build_post_write_action_offers` in `src/sase/xprompt/write_targets.py` returns **only**
a `SKILL_INIT` offer for a written skill source: no commit, no push. But
`sase/memory/generated_skills.md` requires the opposite order - commit the template
change and land it on the canonical branch, _then_ deploy - and `sase skill init`
enforces it by refusing a chezmoi deploy from a dirty source tree. So with chezmoi on
(this user's configuration), the shipped skill offer reliably fails at
`skill_source_integrity_error()` before it can reach `_prompt_overwrite`; it never
deploys anything. With chezmoi off, the integrity check is skipped entirely and the
`_prompt_overwrite` hang is reachable.

That same memory note is also why the fix must **not** pass `--force`: `--force` and
`--allow-dirty` are documented escape hatches that can revert another agent's
deployment.

## Design

One invariant, enforced in two independent layers:

> A post-write action is a background task. It must never inherit an interactive stdin
> or a controlling terminal (**belt**), and it must pass every argument the underlying
> command needs so it has no reason to prompt (**suspenders**).

The belt alone converts the hang into a clean failure. The suspenders make the action
actually do its job. Both are required: without the belt a future prompt reintroduces
the hang; without the suspenders `sase memory init` fails every time with "no commit
message was provided and stdin is not a TTY", and `sase skill init` silently skips every
target and reports success.

Layering note (`rust_core_backend_boundary`): the _policy_ of which follow-up actions
exist and what argv each runs already lives in the UI-free
`src/sase/xprompt/write_targets.py`; keep it there. Process-launch hygiene is Python
runtime plumbing and stays in Python. No `sase-core` change is needed.

## Implementation

### Step 1 - Shared non-interactive captured-run helper

Add `src/sase/noninteractive_subprocess.py` exporting `run_noninteractive`:

```python
DEFAULT_NONINTERACTIVE_TIMEOUT_SECONDS = 900.0

def run_noninteractive(
    argv: Sequence[str],
    *,
    cwd: str | Path | None = None,
    env: Mapping[str, str] | None = None,
    timeout: float | None = DEFAULT_NONINTERACTIVE_TIMEOUT_SECONDS,
) -> subprocess.CompletedProcess[str]: ...
```

Behavior:

- `stdin=subprocess.DEVNULL` - the direct fix for the inherited TTY.
- `start_new_session=True` - a child that opens `/dev/tty` itself (git credential
  helpers, GPG pinentry, chezmoi prompts) still cannot reach ACE's terminal, and the
  whole tree becomes killable as one process group. Precedent:
  `src/sase/ace/tui/bgcmd.py` and `_stream_subprocess` in
  `src/sase/ace/tui/task_subprocess.py` already do this.
- `capture_output=True, text=True, check=False`.
- Implement with `Popen` + `communicate(timeout=...)` rather than
  `subprocess.run(timeout=...)`: because of `start_new_session=True`, `run`'s timeout
  handling would kill only the direct child and leak the group. On `TimeoutExpired`,
  terminate the group (SIGTERM, brief wait, SIGKILL) and re-raise `TimeoutExpired` with
  whatever output was captured. Mirror the SIGTERM/SIGKILL handling in
  `_terminate_process_group` / `_kill_process_group`
  (`src/sase/ace/tui/task_subprocess.py`); factor those out for reuse rather than
  copying if that stays clean.
- Export in `__all__` so symvision sees it used.

The 900s ceiling is chosen to sit well above a legitimate `sase memory init` run that
does `git pull --rebase` and `git push` over the network, while still guaranteeing the
tracked task terminates.

### Step 2 - Route every post-write subprocess through it

In `src/sase/ace/tui/actions/agent_workflow/_prompt_bar_save_xprompt_git.py`:

- `_run_post_write_action_sync`, generic `offer.command` branch: call
  `run_noninteractive(offer.command, cwd=offer.cwd)`. Catch `TimeoutExpired` alongside
  the existing `FileNotFoundError` and return
  `GitCommitPushResult(False, f"{label} timed out after ...")` so the task queue reports
  a real failure instead of pinning a worker.
- `run_git_commit_push_sync._run_git`: add `stdin=subprocess.DEVNULL`,
  `start_new_session=True`, and `env=non_interactive_git_env()` (reuse the existing
  public helper in `src/sase/workspace_provider/utils.py`; do not add a third copy of
  that env-building logic). Keep the `run_with_git_lock_retry` wrapper as-is.

In `src/sase/config/targets.py`:

- Route `apply_chezmoi` through `run_noninteractive` and correct the docstring so the
  "no interactive stdin" claim becomes true instead of aspirational. Keep `--force`.

### Step 3 - Give the memory-init offer a commit subject

In `build_post_write_action_offers` (`src/sase/xprompt/write_targets.py`),
`WrittenFileKind.MEMORY_NOTE` branch, build the command as:

```python
("sase", "memory", "init", "--message", f"{'Add' if is_new else 'Update'} memory note {xprompt_name}")
```

`build_fold_commit_message` (`src/sase/main/init_memory/commit_message.py`) prefixes a
non-conventional subject with `docs(memory): `, yielding
`docs(memory): Update memory note foo`. Passing `--message` unconditionally is safe:
when `fold_dirty` is empty the flag is ignored and the default
`chore: run sase init memory` subject is used.

Note the residual failure mode this deliberately does **not** paper over: if the repo
has dirty files unrelated to the memory work, `_print_foreign_dirty_refusal` returns
exit 1 with an explanatory message. That is correct behavior and now surfaces as an
error notification instead of a hang.

### Step 4 - Scope `sase memory init` to the right repository

`sase memory init` is cwd-scoped: `_load_memory_inputs` uses `Path.cwd()` as the project
root. The post-write subprocess currently inherits whatever directory ACE was launched
from, which need not be the repo that owns the note.

Add an optional `cwd: str | None = None` field to `PostWriteActionOffer` and set it for
`MEMORY_INIT` (and `SKILL_INIT`) to `get_git_root(target.write_path)`, falling back to
`None`. Pass it through in Step 2. For a project memory note this is the workspace clone
that owns the note; for a chezmoi-managed home note it is the chezmoi repo, which is not
a SASE project directory, so init correctly does home-only work.

This is a deliberate behavior change. It is in scope because it is on the same code path
and the current behavior is wrong, but it is isolated in its own step so it can be
dropped without affecting Steps 1-3.

### Step 5 - Add a non-interactive overwrite flag to `sase skill init`

`--force` cannot be used here (see Diagnosis). Add a separate, narrower flag in
`add_skills_init_arguments` (`src/sase/main/parser_init.py`), so both `sase skill init`
and the `sase init skills` alias get it:

```
-y, --yes    Answer the overwrite confirmation for existing generated targets
             with yes; unlike --force this does not override the source-integrity
             or provenance-manifest guards
```

Per `sase/memory/cli_rules.md`: `-y` is free on this parser (taken:
`-h -D -c -d -n -f -A -C -P -p`), every public long option gets a short alias, and
options are listed alphabetically by long name - so `--yes` is added last, after
`--provider`.

In `run_init_skills` (`src/sase/main/init_skills_handler.py`):

- Read `assume_yes = getattr(args, "yes", False)`.
- Change the overwrite gate to
  `if target.path.exists() and not force and not assume_yes:`.
- Do **not** thread `assume_yes` into `prepare_skill_manifest(force=force)` or the
  `skill_source_integrity_error()` check. Keeping those on `force`/`allow_dirty` alone
  is the whole point of the new flag.
- Update the existing non-TTY skip warning to mention `-y` as well as `-f`, so the
  message stays accurate.

### Step 6 - Make the skill offer commit before it deploys

In `build_post_write_action_offers`, `WrittenFileKind.SKILL_SOURCE` branch: return
`(COMMIT_PUSH, SKILL_INIT)` in that order instead of `SKILL_INIT` alone.

- Build the commit offer with the existing `_commit_push_offer` helper (as the generic
  xprompt branch already does), so it is omitted when the file has no git changes.
- `SKILL_INIT` command becomes `("sase", "skill", "init", "--yes")` - never `--force`,
  never `--allow-dirty`.
- `submit_post_write_action_sequence` already runs offers in order and halts the
  sequence on the first failure, so a failed commit correctly prevents the deploy.
- `PostWriteActionsModal` already binds `c` and `s` independently, so no modal change is
  needed.
- Update the `SKILL_INIT` subtitle, which currently claims skill init "commits and
  pushes for you" - with the commit offer split out, that is no longer what it does.

## Tests

New and updated tests, all of which must fail before the corresponding change:

**`tests/test_noninteractive_subprocess.py`** (new)

1. A child running `sys.stdin.isatty()` through `run_noninteractive` reports `False`.
2. A child that calls `input()` returns promptly with an `EOFError` traceback rather
   than blocking - assert the call completes under a short wall-clock bound. This is the
   direct regression test for the hang.
3. A child printing `os.getpgrp()` reports a group id different from the test process's,
   proving `start_new_session=True`.
4. A sleeping child with `timeout=0.5` raises/returns the timeout path and leaves no
   live process in the child's group.

**`tests/xprompt/test_write_targets.py`** (extend)

5. The memory offer command is
   `("sase", "memory", "init", "--message", "Add memory note foo")` for a new note and
   `"Update memory note foo"` for an existing one.
6. The skill offers are `(COMMIT_PUSH, SKILL_INIT)` in that order when the source is
   dirty inside a git repo, and only `(SKILL_INIT,)` when it is not dirty.
7. Regression guard: the skill command contains `--yes` and contains neither `--force`
   nor `--allow-dirty`, with a comment citing `sase/memory/generated_skills.md`.
8. `offer.cwd` is the git root of the written path for both init offers.

**`tests/ace/tui/actions/test_prompt_save_xprompt_git.py`** (extend)

9. `_run_post_write_action_sync` for a generic command delegates to `run_noninteractive`
   with the offer's `cwd`.
10. `run_git_commit_push_sync` passes `stdin=subprocess.DEVNULL`,
    `start_new_session=True`, and a `GIT_TERMINAL_PROMPT=0` env to every git invocation.
11. A timed-out action produces a failed `TrackedTaskResult` with a message naming the
    timeout, so the tracked task completes and the dedup key clears.

**`tests/main/test_skills_handler.py`** (extend)

12. With stdin not a TTY and `--yes`, an existing changed target is overwritten
    (previously: skipped with a warning).
13. `--yes` does not imply `--force`: `prepare_skill_manifest` still receives
    `force=False`, and a dirty source tree is still refused by
    `skill_source_integrity_error()`.

**`tests/main/test_parser_command_help.py`** (extend)

14. `sase skill init --help` and `sase init skills --help` both list `-y, --yes`, and
    the option list stays alphabetical by long name.

Also re-check `tests/main/test_init_memory_*` for any test that asserts the fold prompt
fires with a TTY-shaped stdin double; those stay valid, since the CLI behavior is
unchanged - only ACE's invocation of it changes.

## Verification

```bash
just install          # ephemeral workspace: required before anything else
just check-full       # every lint gate + the full suite
```

`check-full` rather than `check`: this change touches CLI parser surface, the shared
`sase memory init` / `sase skill init` handlers, and ACE, which is well inside the
broadening set.

Manual smoke test, in this order, from a real terminal:

1. `sase ace`, load an xprompt memory definition into the prompt bar, edit it, save to
   the target, accept the `sase memory init` action. Expect: the task completes or fails
   with a visible notification, and the TUI stays responsive throughout.
2. Repeat with a skill source. Expect the commit action to run first, then
   `sase skill init`, with the sequence halting visibly if the commit fails.
3. Negative check: leave an unrelated dirty file in the repo and save a memory note.
   Expect a clear "refusing to commit - uncommitted changes unrelated to the generated
   memory work" error notification, not a hang.

## Out of scope

Note these as follow-ups; do not fix them here.

- `sase skill init` exits 0 after skipping every target on a non-TTY. With `--yes` wired
  up this no longer bites ACE, but a non-interactive caller that omits both flags still
  gets a silent no-op reported as success.
- The `MEMORY_NOTE` branch offers no `APPLY_CHEZMOI` action, so a chezmoi-managed home
  memory note is written to the source tree and `sase memory init` then reads a stale
  applied copy. This is an independent correctness bug in the same epic's surface.
- `sase/memory/generated_skills.md` should document the new `-y/--yes` flag alongside
  `--force` and `--allow-dirty`. Memory-file edits require explicit user permission in
  the implementing conversation; propose it, do not make it unprompted.
