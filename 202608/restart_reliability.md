---
tier: tale
size: medium
title: Make sase agent restart reliable for every agent it offers to restart
goal:
  "`sase agent restart NAME` restarts the 42% of agents it currently refuses, never
  destroys the only copy of the prompt it needs to recover, and never lets a mutation
  escape as a traceback after the old agent is already dead."
proposed_by: bbugyi200.athena.05t.f0
create_time: 2026-08-18 09:42:01
status: wip
---

- **PROMPT:**
  [prompts/202608/restart_reliability.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/restart_reliability.md)

# Plan: Harden `sase agent restart`

## 1. The Reported Failure

```
❯ sase agent restart 061
Forced name reuse did not survive prompt rewrite; the stored prompt has no reusable %id.
Add an explicit %id or use ACE's ,x.
```

Agent `061`'s stored `raw_xprompt.md` is:

```
#gh:gh_bobs-org__bob-cli Describe this repo. #m_sonnet
```

`plan_agent_restart()` in `src/sase/agent/restart.py` calls
`prepare_kill_and_edit_prompt()`, which for a non-family agent calls
`force_name_reuse_in_prompt()` in `src/sase/agent/retry_prompt.py`. That function opens
with:

```python
if "%i" not in raw_prompt:
    return raw_prompt
```

The prompt has no `%id` directive, so it is returned unchanged,
`plan_force_reuse_launch()` sees nothing to rewrite and returns `None`, and the planner
raises `reason="name_not_reusable"`.

This is not an edge case. Scanning every `raw_xprompt.md` under
`~/.sase/projects/*/artifacts/ace-run/`:

| Population                                            | Count | Share |
| ----------------------------------------------------- | ----- | ----- |
| Agents with a stored prompt                           | 7146  | 100%  |
| Family-branch agents (directive injected by ACE path) | 1674  | 23%   |
| **No `%id` and no family branch — restart refuses**   | 2984  | 42%   |

Every plainly-launched agent — `061`, `037`, `02q`, `02x`, `025` — is in the refusing
set. The refusal hint is also wrong twice over: telling the user to "add an explicit
`%id`" asks them to edit an immutable historical artifact, and ACE's `,x` does not solve
it either. `,x` only _seeds the prompt bar_; with no `%id` it seeds the prompt unchanged
and a submit produces a **new** auto-named agent. `,x` never had name reuse here — it
just never had to say so, because it does not promise a name.

The CLI does promise a name. So the CLI has to earn it.

## 2. What Else Is Wrong

Three further defects found while tracing the mutation path. All are in
`execute_agent_restart()`.

### 2.1 The recovery command reads a file the command just deleted

`execute_agent_restart()` builds

```python
recovery = f'sase run "$(cat {plan.artifacts_dir}/raw_xprompt.md)"'
```

and returns it on the `partial` outcome. But step 3 of the same function calls
`apply_force_reuse_launch()` → `wipe_names_for_forced_reuse()` →
`wipe_agent_name_for_reuse()` in `src/sase/agent/names/_wipe.py`, whose docstring says
the wipe "is intentionally stronger than dismissal: artifact directories, dismissed
bundles/index rows, notifications, workspace claims, and registry reservations tied to
the owner and its descendants are removed". `_remove_artifact_dirs()` calls
`shutil.rmtree(path)`.

So `plan.artifacts_dir` is gone by the time the recovery command is printed. The one
output that exists to rescue the user from a half-finished restart cannot work.
`tests/test_agent_restart_execute.py:121-124` asserts only the command's _substrings_
with `apply_force_reuse_launch` stubbed out, so the suite never noticed.

### 2.2 The deletion is never disclosed

`_build_wipe_plan()` seeds from the owner and then iterates to a **transitive closure**
over `_artifact_related()` / `_bundle_related()`, following `relation_refs` and
`outgoing_suffixes`. A restart can therefore delete artifact directories belonging to
related agents, not just the target's own.

The preview panel says nothing about any of this. A user restarting a DONE agent to
re-run it has no way to learn, before confirming, that the previous run's
`live_reply.md`, `tool_calls.jsonl`, `done.json`, diffs, and workflow state — and
possibly a relative's — are about to be removed.

(Chat transcripts under `~/.sase/chats/` are stored separately and do survive. Say so
rather than overstating the loss.)

### 2.3 A mutation can escape as a traceback after the agent is dead

`apply_force_reuse_launch(plan.force_reuse_plan)` is called with no `try`. It raises
`RuntimeError` from `wipe_names_for_forced_reuse()` in two documented cases: a wipe
error (`Failed to wipe agent name '<name>': ...`) and a container name
(`Cannot force-reuse <kind> container '<name>'.`). Either one escapes
`execute_agent_restart()` after the kill has already succeeded, producing a raw
traceback with no step ledger, no `partial` outcome, no recovery command, and an exit
code outside the command's documented 0/1/2 contract — in the worst possible state, the
one where the old agent is dead and the new one was never launched.

The container case is also knowable _before_ any mutation: `lookup_registered_name()` is
exported from `sase.agent.names` and returns the owner record carrying `container_kind`.

## 3. Fix 1 — Force Name Reuse By Injecting The Directive

Add to `src/sase/agent/relaunch_prompt.py`:

```python
def ensure_forced_name_reuse(prompt: str, agent_name: str) -> str:
    """Return *prompt* with a forced-reuse ``%id`` for *agent_name*."""
```

It composes two already-tested primitives rather than formatting a directive by hand:

1. `set_prompt_name(prompt, agent_name)` from `sase.xprompt.directive_edit`. Its
   no-directive branch is exactly `insert_directive(protected, f"%id:{name}")`, and its
   has-directive branch replaces the value while preserving kwargs — so this both
   inserts a missing `%id` and overwrites a template-valued one.
2. `force_name_reuse_in_prompt(result, replacement_name=agent_name)` to add the `!`.

Wire it into `plan_agent_restart()` between the existing
`prepare_kill_and_edit_prompt()` call and `plan_force_reuse_launch()`:

- Run `plan_force_reuse_launch(rewritten)` as today.
- If it returns `None`, call `ensure_forced_name_reuse(rewritten, meta_name)` and run
  `plan_force_reuse_launch()` once more on the result.
- Only if _that_ still returns `None` raise `name_not_reusable` — now a genuine internal
  invariant failure, not the common case. Reword its message accordingly and drop the
  "add an explicit `%id`" hint, which was never actionable.

Record which path was taken on the plan as `name_reuse_source: "prompt" | "injected"`,
so the preview and the JSON envelope can both report it honestly.

### 3.1 The leading VCS tag survives injection — verified

Injection prepends a directive line, so the prompt becomes:

```
%id:!061
#gh:gh_bobs-org__bob-cli Describe this repo. #m_sonnet
```

This is safe, and the reason must not be re-derived by guesswork.
`extract_vcs_workflow_tag()` in `src/sase/xprompt/_parsing_vcs_tags.py` skips a leading
directive run before matching the tag, using

```python
_DIRECTIVE_PREFIX_RE = re.compile(r"(%[^\s(]+(?:\((?:[^()]|\([^()]*\))*\))?[\s]+)+")
```

whose trailing `[\s]+` matches the newline. The `#gh:` tag therefore still reads as the
_leading_ tag, and the restart still targets the same project. **Pin this with a test**
(§7): if that regex ever stops spanning newlines, restart would silently relaunch every
injected agent as a home agent in the wrong project, and nothing else in the suite would
catch it.

### 3.2 Refuse fan-out before injecting, not after

`plan_force_reuse_launch()` raises `_ForceReuseLaunchFanoutError` when a forced `!` name
meets an alt/repeat fan-out, and `plan_agent_restart()` currently reports that verbatim
under `reason="preflight"`. Once we inject the `!` ourselves, that message would blame
the user for a directive we added.

So before injecting, check `plan_prompt_fanout_variants(prompt) is not None`
(`sase.xprompt.directives` — the same predicate `_prompt_has_launch_fanout()` uses) and
raise `reason="fanout"` with a message that names the real cause: the stored prompt fans
out, so it has no single agent to restart. Leave the existing `preflight` mapping alone
for prompts whose _own_ `%id` triggers it.

### 3.3 Names that cannot be reused

Before planning any wipe, call `lookup_registered_name(meta_name)`. If the record
carries a truthy `container_kind`, raise `reason="container"` with a message naming the
kind (clan or family container) and stating that only concrete members restart. This
converts §2.3's post-kill `RuntimeError` into a clean pre-mutation refusal at exit 2.

## 4. Fix 2 — Survive And Disclose The Wipe

### 4.1 A read-only wipe preview

`_build_wipe_plan()` already computes the full closure without mutating. Expose it: add
`preview_agent_name_wipe(name) -> AgentNameWipePreview` to
`src/sase/agent/names/_wipe.py` and export it from `src/sase/agent/names/__init__.py`
next to `wipe_agent_name_for_reuse`. The dataclass carries `artifact_dirs`,
`bundle_paths`, `names`, and `container_kind`, built by the existing
`lookup_registered_name()` + `_build_wipe_plan()` pair with no further changes to the
wipe logic itself.

`plan_agent_restart()` calls it and stores the result on the plan. This is read-only and
belongs in the planning half, consistent with the module's stated plan/apply split.

### 4.2 Persist recovery state before mutating anything

In `execute_agent_restart()`, **before** the stop, write a recovery bundle to a location
the wipe does not touch:

```
~/.sase/restarts/<timestamp>-<name>/
    raw_xprompt.md      # the original stored prompt, byte for byte
    rewritten.md        # the prompt the restart actually launches
    agent_meta.json     # copied from the old artifacts dir
    restart.json        # name, project, artifacts_dir, timestamps
```

Use `sase_subdir()` from `sase.core.paths` so the location follows the same root as the
rest of SASE's state. Only these small files are copied — not the whole artifacts tree.

The `partial` outcome's `recovery_command` then becomes

```
sase run "$(cat ~/.sase/restarts/<timestamp>-<name>/rewritten.md)"
```

which points at a file that still exists. Carry `recovery_dir` on the outcome and in the
JSON envelope. Writing the bundle is best-effort: if it fails, degrade to carrying the
prompt text inline on the outcome and print it, rather than aborting a restart the user
asked for.

### 4.3 Show the cost in the preview

Add two preview rows, populated from §4.1:

- `Name reuse` — extend the existing row to distinguish `forced (%id(!061)) · injected`
  from `forced (%id(!061)) · from prompt`.
- `Deletes` — `N artifact dirs · M bundles`, listing up to three paths and `+K more`.

Add a warning when the closure reaches beyond the target's own artifacts dir:

```
Releasing '061' also deletes 2 related agents' artifacts: <path>, <path>.
```

and a standing note on every non-dry run:

```
The previous run's artifacts are deleted; the chat transcript under ~/.sase/chats is kept.
```

### 4.4 Widen the confirmation gate — but only where it is surprising

Today `cli_restart.py` confirms only when `plan.preview.is_live`. Restarting a DONE
agent deletes its artifacts with no prompt at all.

New rule: confirm when the agent is live **or** when the wipe closure includes artifact
directories beyond the target's own. Both are the surprising cases. A plain DONE agent
whose wipe touches only itself still restarts with one keystroke, because that is the
common case and the deletion is already disclosed in the preview.

Keep the existing `-y`, `--dry-run`, TTY, and `EOFError` handling unchanged.

## 5. Fix 3 — No Unguarded Mutation

Restructure `execute_agent_restart()` so that every step after the stop runs inside
error handling, and each failure maps to a documented outcome:

| Step                       | Failure                         | Status          | Exit |
| -------------------------- | ------------------------------- | --------------- | ---- |
| recovery bundle            | any                             | (warn, proceed) | —    |
| kill / dismiss             | `success=False`                 | `kill_failed`   | 2    |
| `apply_force_reuse_launch` | `RuntimeError` or any exception | `wipe_failed`   | 1    |
| launch                     | exception or empty results      | `partial`       | 1    |

`wipe_failed` is new. It means the old agent was stopped but its name was never
released, so the correct advice differs from `partial`: the name is still taken, and the
user should inspect with `sase agent show` before retrying. Both `wipe_failed` and
`partial` print the recovery command and the recovery directory.

Emit a ledger line for each — the failure path currently emits nothing for the wipe
because it cannot fail gracefully.

## 6. Fix 4 — Verify The Restart Actually Kept The Name

`execute_agent_restart()` takes `results[0]` and reports success without checking that
the new agent got the name the whole command is built around.

After a successful launch, compare the launched agent's resolved name against
`plan.name`. On mismatch, still report success — the agent is running and killing it
would be worse — but set `renamed_to` on the outcome, print a yellow ledger line, and
include it in the JSON envelope:

```
  ✓ launched   PID 492011 · workspace #14
  ! name       launched as '062', not '061'
```

Silence here is what turns a subtle launcher regression into a mystery.

## 7. Tests

Extend the three existing files; add none.

`tests/test_agent_restart_plan.py`:

- **The reported bug.** An agent named `061` whose stored prompt is
  `#gh:gh_bobs-org__bob-cli Describe this repo. #m_sonnet` plans successfully, and the
  rewritten prompt carries a forced-reuse `%id` for `061`.
- **VCS tag survives injection** (§3.1). Assert `extract_vcs_workflow_tag(rewritten)`
  still returns the `#gh:` tag — the regression that would silently retarget the
  project.
- `name_reuse_source` is `"injected"` there and `"prompt"` for an agent whose stored
  prompt already carries `%id`.
- A prompt with an existing `%id` is **not** re-injected; the rewritten prompt is
  byte-identical to today's output. This pins that the fix is additive.
- A family member still takes the family branch and is not double-rewritten.
- A fan-out prompt raises `reason="fanout"`, not `preflight`.
- A container name raises `reason="container"` before any mutation.
- Planning still touches no marker, claim, registry entry, or process.

`tests/test_agent_restart_execute.py`:

- **Recovery survives the wipe.** With a fake `apply_force_reuse_launch` that actually
  `rmtree`s the artifacts dir (the real behavior), a failing launch still yields a
  `recovery_command` naming a file that **exists on disk**. Assert the file is readable
  and its contents are the prompt — the assertion the current substring test could not
  make.
- The recovery bundle is written _before_ the kill, not after.
- `apply_force_reuse_launch` raising `RuntimeError` yields `status="wipe_failed"`, not a
  propagated exception, and emits a failed ledger line.
- A launch whose agent takes a different name yields success plus `renamed_to`.
- Existing ordering assertions (kill before wipe, failed kill aborts, DONE dismisses)
  stay green.

`tests/test_agent_restart_cli.py`:

- `wipe_failed` exits 1 and prints the recovery directory.
- Confirmation is requested when the wipe closure reaches a related agent, and skipped
  for a self-only DONE wipe.
- The preview shows the `Deletes` row and the related-agent warning.
- The JSON envelope carries `name_reuse.source`, `deletes`, `recovery_dir`, and
  `renamed_to`; bump `schema_version` to `2` and assert it.

Assert on structure and substrings, never on Rich box geometry.

## 8. Files

Modify:

- `src/sase/agent/relaunch_prompt.py` — `ensure_forced_name_reuse()`.
- `src/sase/agent/restart.py` — injection fallback, `fanout` / `container` refusals,
  wipe preview on the plan, recovery bundle, guarded execution, `wipe_failed`,
  `renamed_to`.
- `src/sase/agent/names/_wipe.py` — `AgentNameWipePreview`, `preview_agent_name_wipe()`.
- `src/sase/agent/names/__init__.py` — export both (`__all__` is sorted; keep it
  sorted).
- `src/sase/agents/_restart_render.py` — `Deletes` row, extended `Name reuse` row, new
  warnings, `wipe_failed` output, `renamed_to` ledger line, envelope v2.
- `src/sase/agents/cli_restart.py` — widened confirmation, `wipe_failed` → exit 1.
- `src/sase/main/parser_agent.py` — epilog note that restart deletes the previous run's
  artifacts.
- `docs/cli.md` — document the deletion, the recovery directory, and exit 1 covering
  both `partial` and `wipe_failed`.

No new modules. Every touched file stays far under the `toobig` 1000/850/700 thresholds.

## 9. Decisions Already Made

Do not relitigate these.

- **Reuse the name; do not fall back to a fresh one.** The command's contract is "same
  prompt, same name" and the operator's handle must stay stable. Relaunching `061` as
  `06a` would make the command a slower `sase run`. The cost is the wipe, so the wipe is
  disclosed and made survivable instead of avoided.
- **No new ACE/TUI behavior.** `,x` keeps seeding the prompt bar and keeps its current
  no-`%id` behavior. `ensure_forced_name_reuse()` lands in `sase/agent/` and ACE is not
  rewired to call it; changing `,x` is a separate decision with its own UX.
- **No change to wipe semantics.** `wipe_agent_name_for_reuse()` stays exactly as
  destructive as it is. This plan only adds a read-only preview of it. Narrowing the
  closure is a much larger change to name-registry correctness and is out of scope.
- **No Rust core change.** Same reasoning as the original plan: this composes existing
  Python services and adds no new domain rule. `sase_core` has no force-reuse or
  relaunch-prompt rewriting to extend.
- **No feature flag.** This is a bug fix to a landed command, not a staged rollout.
- **`-j` still bypasses confirmation.** Machine callers cannot answer a prompt. This is
  existing behavior; document it in `docs/cli.md` rather than changing it.

## 10. Verification

Run `just install` first — workspace virtualenvs go stale, and this workspace's is
currently broken (`sase_core_rs is not importable`).

Then `just check` inline. Because this touches agent-name machinery and the launcher,
hand `just check-full` to `/sase_monitor`
(`sase monitor start --command 'just check-full' --next ...`) before landing; never run
it inline.

Two failures are **preexisting on master** and already have beads. Do not fix them here:

- `sase-pm` — symvision unused public `long_memory_entry_path` and
  `normalize_long_memory_description_lines` in `src/sase/amd/_agents_doc.py`.
- `sase-pn` —
  `tests/main/test_init_memory_glossary.py::test_memory_plan_renders_glossary_terms_block_in_tier2`.

Finally, confirm the original report is fixed against a real agent:

```
sase agent restart 061 --dry-run
```

It must print a preview — including the `Deletes` row — instead of the
`name_not_reusable` refusal.
