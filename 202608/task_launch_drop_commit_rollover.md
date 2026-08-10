---
tier: tale
title: Drop the
goal:
  "A `sase bead work <task-id>` launch prompt no longer includes the `#commit` xprompt,
  and the bundled `bd/work_task` xprompt body no longer carries the `IMPORTANT: Do not
  commit your changes unless/until the finalizer asks you to.` line, making task
  launches match the epic phase launch path."
size: small
proposed_by: bbugyi200.athena.x0
create_time: 2026-08-10 09:46:04
status: wip
---

# Drop the `#commit` rollover and the redundant no-commit line from task-bead launches

## Goal

Two coupled changes to how `sase bead work <task-id>` launches a task worker:

1. Remove the
   `IMPORTANT: Do not commit your changes unless/until the finalizer asks you to.` line
   from the bundled `bd/work_task` xprompt body.
2. Stop prefixing the rendered task-launch prompt with the `#commit` xprompt.

After this change a task-bead launch prompt is exactly the VCS ref, the identity / model
directives, the `bd/work_task` reference, an optional `#plan`, and optional feedback.

## Background and why these two go together

`render_task_prompt` (`src/sase/bead/work.py:353-363`) currently emits a first line of
`#<vcs>:<project> #commit`. The `#commit` xprompt (`src/sase/xprompts/commit.yml`) does
three things: it sets `SASE_COMMIT_METHOD=create_commit`, it injects a prompt part
telling the agent not to create a commit/branch/PR itself, and it runs post-turn steps
that read `commit_result.json`.

The `IMPORTANT:` line in `bd/work_task` (`src/sase/default_config.yml:1090`) restates
`#commit`'s injected no-direct-commit rule. So today the task worker receives that same
instruction twice. Removing `#commit` alone would leave the duplicate-turned-orphan line
referring to a finalizer contract the prompt no longer establishes; removing the line
alone would leave the redundancy in place. Doing both lands the task path on one
coherent shape.

### Key finding: this does not disable committing

The commit finalizer is **not** gated on `#commit`. `run_commit_finalizer`
(`src/sase/llm_provider/commit_finalizer.py:130-190`) runs for any SASE agent — it skips
only on `SASE_DISABLE_COMMIT_STOP_HOOK`, disabled config, or a missing
`SASE_AGENT_TIMESTAMP` — and fires whenever the workspace is dirty. When
`SASE_COMMIT_METHOD` is unset, `build_commit_instruction_message`
(`src/sase/commit_instructions.py:109`) and `sase commit` itself
(`src/sase/main/commit_handler.py:85`) both default to `create_commit`. So the effective
commit method for a task worker is unchanged.

### Key finding: this matches what epic phase agents already do

`render_multi_prompt`'s `_segment_prefix` (`src/sase/bead/work.py:600-614`) emits only
`#<vcs>:<ref>` (plus `#pr(...)` on the first Patch segment). Epic phase and land agents
have never received `#commit`, and `bd/work_phase_bead` carries no no-commit line. They
rely on the dirty-repo finalizer defaulting to `create_commit`. This change makes the
task path consistent with the phase path rather than introducing a new mode.

### Accepted consequences

- The `#commit` inject text no longer reaches the task worker. The no-direct-commit rule
  still reaches it through the `/sase_git_commit` skill description ("NEVER invoke this
  skill unless the user explicitly asks you to commit or a post-completion finalizer
  triggers it"), which is how phase agents already get it.
- `#commit` is `rollover`-tagged, so `_get_embedded_workflow_refs`
  (`src/sase/axe/run_agent_exec.py`) will no longer reconstruct `#commit` into follow-up
  agent prompts spawned from a task worker. Those follow-ups fall back to the same
  finalizer default. This matches phase-agent follow-ups today.
- `#commit`'s post-turn `report` step no longer emits `meta_new_commit`,
  `meta_commit_message`, `meta_changespec`, or `diff_path` for task agents. Phase agents
  already run without these. Verify (step 6) that ACE still shows a task agent's commit
  after a dry-run-then-real launch, and if anything regresses, record it rather than
  silently reintroducing `#commit`.

## Out of scope

- Epic phase, land, and plan-file launch paths — they already omit `#commit`.
- The `#commit` xprompt itself, its tags, and every other `#commit` caller.
- `sase/memory/*.md` and the generated `AGENTS.md` / `CLAUDE.md` / `GEMINI.md` /
  `OPENCODE.md` / `QWEN.md` shims. `sase/memory/xprompts.md:91` describes the general
  rollover contract, not the task-launch prompt, so it stays accurate. Do **not** edit
  memory files: no user permission has been granted for that in this work.
- `CHANGELOG.md` — release-please generates it.

## Steps

### 1. Remove the `IMPORTANT:` line from `bd/work_task`

In `src/sase/default_config.yml`, inside the `bd/work_task:` `content:` block (around
line 1090), delete the trailing line

```
IMPORTANT: Do not commit your changes unless/until the finalizer asks you to.
```

and the blank line that separates it from the preceding `/sase_new_task` paragraph. The
block must still end with the `/sase_new_task` paragraph and stay valid YAML at the same
indentation; keep the `tags:` and `input:` keys untouched.

### 2. Drop `#commit` from the rendered task prompt

In `src/sase/bead/work.py`, in `render_task_prompt`, change the first entry of `lines`
from

```python
f"#{vcs_context.vcs_workflow}:{vcs_context.project_name} #commit",
```

to

```python
f"#{vcs_context.vcs_workflow}:{vcs_context.project_name}",
```

Leave everything else in the function unchanged: `_validate_vcs_context`, the
top-level-`---` feedback guard, `%id`, `%m`, the `bd/work_task` reference, and the
`#plan` append for large/xlarge. Update the function or module docstring only if it
names `#commit`.

### 3. Update the prompt-shape tests

- `tests/test_bead/test_work_task_rendering.py:28` — in
  `test_task_prompt_has_exact_single_segment_order_and_feedback_tail`, change the
  expected first line from `"#gh:sase #commit\n"` to `"#gh:sase\n"`. This is an exact
  full-prompt equality assertion, so it is the primary regression guard.
- `tests/test_bead/test_cli_work_task.py:176` — change the expected first line from
  `f"#git:sase #commit\n"` to `f"#git:sase\n"`.

Add an explicit negative assertion in `tests/test_bead/test_work_task_rendering.py` so
the removal is guarded by intent and not only by the equality check — e.g. a test
asserting `"#commit" not in rendered` for a task prompt rendered with a
`VCSLaunchContext`.

### 4. Replace the xprompt-body assertion

In `tests/test_bead_xprompt_tags.py`, the test
`test_builtin_task_prompt_defers_commits_to_finalizer` (lines 133-138) asserts the
removed sentence is present. Invert it rather than deleting it, so the removal stays
guarded: rename it to reflect the new contract (e.g.
`test_builtin_task_prompt_omits_commit_deferral_line`) and assert

```python
assert "Do not commit your changes unless/until the finalizer asks you to." not in body
```

Keep the surrounding `get_all_prompts()["bd/work_task"].steps[0].prompt_part` lookup and
the `assert body is not None` guard. Leave every other test in the file alone —
`test_builtin_work_task_resolves`, the wait-directive parametrization at line 106, and
the tag assertions are all unaffected.

### 5. Update the docs

- `docs/xprompt.md:1345` — the `#bd/work_task` row currently reads "Task-agent prompt
  used by `sase bead work`; defers commits to the finalizer". Replace the trailing
  clause so it no longer claims a prompt-level commit deferral. Keep the markdown table
  column alignment consistent with the neighboring rows in that table.
- Grep `docs/` for any other statement that a task launch includes `#commit` or that
  `bd/work_task` instructs the agent about committing, and fix what you find.
  `docs/beads.md:320` ("instructed to close the task with verification evidence") and
  `docs/beads.md:1427` ("Renders one VCS-aware prompt ending in
  `#bd/work_task:<task-id>`") are already accurate and should not need changes — confirm
  rather than assume.

### 6. Verify

- `just install` first (ephemeral workspace).
- `just check`.
- Run the directly affected suites explicitly, since the scoped lane selects on the
  import graph and `default_config.yml` is data, not an import:
  `just test tests/test_bead/test_work_task_rendering.py tests/test_bead/test_cli_work_task.py tests/test_bead_xprompt_tags.py`
  (or the equivalent pytest invocation for this repo).
- Sanity-check the real rendered prompt with
  `sase bead work <some-ready-task-id> --dry-run` and confirm the printed prompt's first
  line is the bare `#<vcs>:<project>` with no `#commit`, and that the `bd/work_task`
  expansion no longer contains the `IMPORTANT:` line.
- Because `docs/` and `src/sase/default_config.yml` are data/prose that the scoped test
  selector may not map to tests, run `just check-full` before landing.

## Done when

- `render_task_prompt` emits no `#commit` and `bd/work_task`'s body has no `IMPORTANT:`
  commit line.
- The three test files assert the new shape, including at least one negative assertion
  per removal.
- `docs/xprompt.md` no longer claims `bd/work_task` defers commits to the finalizer.
- `just check-full` passes.
