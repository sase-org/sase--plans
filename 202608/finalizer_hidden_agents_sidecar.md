---
tier: tale
title: Stop the commit finalizer from failing runs over the shared agents sidecar
goal:
  A concurrent agents-sidecar sync can no longer make a successful agent run fail commit
  finalization, while genuine discarded or reset agent work still fails.
size: medium
proposed_by: bbugyi200.athena.03n
create_time: 2026-08-16 11:14:47
status: wip
---

# Stop the commit finalizer from failing runs over the shared agents sidecar

## Problem

A successful agent run is marked `failed` when the machine-wide, machine-managed
`agents` sidecar repository happens to be dirty while that agent's commit finalizer
runs. The finalizer treats another process's sidecar-sync payload as the agent's own
uncommitted work, then hard-fails the run when a concurrent `agents_sync` commit cleans
it.

Observed failure (agent `03i`, 2026-08-16):

```
WorkflowExecutionError: Step 'main' failed: Error: Commit finalizer failed: dirty
work vanished without an attributable commit. The finalizer will not treat
discarded, reset, or foreign-agent work as successful finalization.
- agents: /home/bryan/.sase/projects/gh_sase-org__sase/repos/agents (no newly
  reachable commit was attributed to this agent)
  - README.md
  - prompts/202608/README.md
  - users/bbugyi200/machines/athena/hoods/03j/snapshot.json
  - agents/bbugyi200.athena.03j--mon/chat.md
  - agents/bbugyi200.athena.03j--plan/meta.json
  - ... (55 total)
```

Every listed path is sidecar-sync payload for _other_ agents (`03b`, `03c`, `03j`) and
machine hood snapshots. None of it was ever `03i`'s work. `03i` had already finished
successfully: it committed bead state, authored and archived its plan, and its
implementation landed as `2a587fff7 fix: reconcile phantom running procs` (tagged
`SASE_AGENT=[bbugyi200.athena.03i]`). Only the finalizer verdict was wrong, and it cost
a held workspace plus a false `failed` outcome.

## Root cause

Two independent defects combine into a race that any concurrent agent can lose.

### Defect 1: the hidden `agents` sidecar leaks into finalizer dirty state

`agents` is a **hidden** sidecar role —
`HIDDEN_SIDECAR_ROLES = frozenset({AGENTS_SIDECAR_ROLE})` in
`src/sase/_linked_repo_config.py`. It is machine-managed by `sase.agents_sync`, shared
by every agent on the host, and stored outside any workspace at
`~/.sase/projects/<project_key>/repos/agents`. The codebase filters it out of launch
metadata, linked-repo resolution, repo inventory, `sase repo open`, sidecar auto-sync,
and memory init.

`src/sase/llm_provider/commit_finalizer_state.py::_dirty_sdd_store_repos` has no such
filter. It walks `sdd_commit_targets(resolve_sdd_store(...), None)`, which iterates
`store.sidecar_dirs`. The project's SDD store record lists `agents` as a configured
sidecar (`git@github.com:sase-org/sase--agents.git`), and
`sase/sdd/_sidecar_init.py::resolve_sidecar_clone_root` maps the `agents` role to the
hidden clone dir. `sdd_store_label` then labels it with its role name. The result is
exactly the `DirtyRepo` in the error above:

```
DirtyRepo(name="agents",
          path="~/.sase/projects/<project_key>/repos/agents",
          kind="sdd")
```

That this is unintended is visible in the same module:
`_dirty_agents_prompt_archive_repo` exists specifically to admit the agents sidecar in
one narrow case — "Return a dirty agents sidecar only for canonical prompt-file edits" —
and only when _every_ dirty path satisfies `is_prompt_archive_path`. That deliberate
narrowing is dead weight today, because `_dirty_sdd_store_repos` already admits the
whole repo unconditionally. Worse, when every dirty path _is_ a prompt archive path both
collectors fire, so one repository is entered into `DirtyState` twice under two
different names.

### Defect 2: machine-managed sidecar commits carry no agent provenance

`src/sase/agents_sync/git_sync_ops.py::commit_agents_payload_if_dirty` commits with
`apply_auto_commit_type_tag(...)`, which stamps only `SASE_TYPE=agents_sync`. The
prompt-archive commit path does the same. Confirmed in the sidecar's history around the
failure:

```
2e41e0cb1 2026-08-16 10:55:10  chore(agents): sync from bbugyi200.athena
                               SASE_TYPE=agents_sync          # no SASE_AGENT
b2a5fe4f7 2026-08-16 10:10:23  chore(agents): archive prompt for ...03i--plan
                               SASE_TYPE=agents_sync          # no SASE_AGENT
```

This is correct in itself — one sync commit covers many agents, so attributing it to a
single agent would be a lie. But
`src/sase/llm_provider/commit_finalizer_git_progress.py::discarded_dirty_work_evidence`
demands `SASE_AGENT` provenance from every commit that cleans a repo it was tracking.
When it finds none it emits `reason="missing_agent_provenance"` and
`_fail_on_discarded_dirty_work` raises.

### The race

1. `03i`'s finalizer collects dirty state; the shared agents sidecar is dirty with other
   agents' sync payload, so it enters `DirtyState` (Defect 1).
2. The finalizer asks the model to commit it, and snapshots `fingerprint_before` with
   the sidecar's HEAD at `d3f639305`.
3. A concurrent `agents_sync` (TUI, another runner, or a monitor) commits the payload as
   `2e41e0cb1`, tagged `SASE_TYPE=agents_sync` only (Defect 2).
4. The next `_collect_dirty_state` finds the sidecar clean, HEAD advanced, and no new
   commit tagged `SASE_AGENT=03i` → `missing_agent_provenance` → the run fails.

Fixing only Defect 1 is not sufficient. The surviving narrow path
(`_dirty_agents_prompt_archive_repo`) races against the very same untagged
`chore(agents): archive prompt for ...` commits, so an agent that edits a canonical
prompt file can still lose the race. Both layers are required.

## Approach

### Layer 1 — exclude hidden sidecar roles from SDD dirty-state collection

In `src/sase/llm_provider/commit_finalizer_state.py::_dirty_sdd_store_repos`, skip
targets whose sidecar role is in `HIDDEN_SIDECAR_ROLES`. Determine the role from the
target store (`target_store.sidecar_role`) rather than from the label string, and import
`HIDDEN_SIDECAR_ROLES` from `sase.linked_repos`, which already re-exports it, to stay
consistent with the other call sites.

After this change `_dirty_agents_prompt_archive_repo` becomes the single, intentional
entry point for the agents sidecar, which also removes the duplicate-`DirtyRepo`
condition described above.

Do not change `_auto_commit_bead_state` or `_separate_sdd_store_repo_may_exist` in
`commit_finalizer.py`: the former resolves `beads` explicitly, and the latter only
performs an early-out existence probe.

### Layer 2 — accept machine-managed sidecar auto-commits as attributable

In `src/sase/llm_provider/commit_finalizer_git_progress.py`, a repo that goes clean
because a machine-managed SASE auto-commit landed is finalized, not discarded. Extend
the attribution predicate used by `discarded_dirty_work_evidence` so a new commit counts
as attributable when either:

- `parse_trailing_commit_tags(message).get("AGENT")` matches the current agent (today's
  behavior), **or**
- that commit carries `TYPE == "agents_sync"`.

Keep this narrow and deliberate:

- Only `agents_sync` is accepted. Do **not** whitelist auto-commit types generally
  (`sdd`, `beads`, `stitch`, ...). The `missing_agent_provenance` check exists to catch
  an agent resetting or discarding its own work, and broad acceptance would gut it.
- Accept the commit if _any_ new commit in `before_head..after_head` qualifies, which
  matches how `_new_commits_include_agent` already scans that range.
- Leave the `head_not_advanced` branch untouched. A repo that goes clean with no new
  commit at all is still a genuine discard and must keep failing.

Name the new predicate to say what it means (for example
`_new_commits_are_attributable`, wrapping the existing `_new_commits_include_agent`),
and keep `discarded_dirty_work_evidence`'s structure and return type unchanged so
`_fail_on_discarded_dirty_work` and `discarded_dirty_work_message` need no edits.

Introduce the `agents_sync` tag value as a shared constant rather than a bare literal in
two modules; `src/sase/agents_sync/git_sync_ops.py` currently passes the string
`"agents_sync"` to `apply_auto_commit_type_tag`. Placing the constant beside that
producer and importing it in the finalizer keeps producer and consumer from drifting.

### Explicitly out of scope

- Adding `SASE_AGENT` trailers to `agents_sync` commits. One sync commit spans many
  agents; single-agent attribution would be incorrect.
- Any change to how a genuine discard fails the run, holds the workspace, or reports
  itself.
- Retroactively repairing the `03i` run record.

## Files

- `src/sase/llm_provider/commit_finalizer_state.py` — hidden-role filter in
  `_dirty_sdd_store_repos`.
- `src/sase/llm_provider/commit_finalizer_git_progress.py` — attribution predicate
  accepting `TYPE=agents_sync`.
- `src/sase/agents_sync/git_sync_ops.py` — export the `agents_sync` commit-type constant
  used by both sides.
- `tests/llm_provider/` — new regression coverage (see below).

## Testing

Add regression tests; both defects need coverage that fails before the change.

1. **Hidden sidecar excluded.** Build an SDD store whose sidecar roles include `agents`
   plus a visible role, make both dirty, and assert `collect_dirty_state` returns the
   visible sidecar and no `DirtyRepo` for the agents clone. Follow the fixture style in
   `tests/llm_provider/test_commit_finalizer_auto_sdd_qa.py`, which already builds
   sidecar-storage stores.

2. **No duplicate agents entry.** With the agents sidecar dirty and _every_ dirty path a
   canonical prompt-archive path, assert exactly one `DirtyRepo` points at the agents
   clone (the `_dirty_agents_prompt_archive_repo` entry).

3. **`agents_sync` commit is attributable.** In a temp git repo: dirty state before,
   then commit with a message tagged `SASE_TYPE=agents_sync` and no `SASE_AGENT`, clean
   state after, HEAD advanced. Assert `discarded_dirty_work_evidence` returns empty.

4. **Genuine discard still fails.** Two cases that must keep producing evidence: HEAD
   unchanged with the dirt gone (`head_not_advanced`), and HEAD advanced by a commit
   tagged with a _different_ agent and no `agents_sync` type
   (`missing_agent_provenance`).

Existing `tests/llm_provider/test_commit_finalizer_no_progress.py` covers the current
discard behavior — keep it green rather than relaxing it.

## Verification

```bash
just install                                    # ephemeral workspace may be stale
just check                                      # lint gates + diff-scoped tests
```

Then confirm the specific suites:

```bash
.venv/bin/python -m pytest tests/llm_provider -q
```

This change touches finalizer behavior that every agent run depends on, so finish with
`just check-full` through `/sase_monitor` (never inline), handing it a `--next` action:

```bash
sase monitor start --command 'just check-full' --label 'finalizer hidden sidecar' --next '<follow-up>'
```

## Risks

- **Under-reporting real dirt.** If a future non-hidden sidecar were misclassified as
  hidden, its dirty work would stop blocking finalization. Mitigated by keying off
  `HIDDEN_SIDECAR_ROLES`, which contains only `agents`.
- **Weakening discard detection.** Accepting `TYPE=agents_sync` means an agent could in
  principle hand-craft such a commit to escape the check. Accepted: the agents sidecar
  holds no source code, and the alternative is the false failure this plan fixes.
- **Test flakiness.** The tests must construct git state deterministically rather than
  racing a real sync; use fixed commit messages and explicit HEAD manipulation.
