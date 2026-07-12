---
create_time: 2026-07-08 14:55:05
status: wip
prompt: 202607/prompts/fix_plan_chain_sdd_refs.md
tier: tale
---
# Fix Approved Plan-Chain SDD References

## Problem

Approved plan-chain coder launches are still failing after the recent SDD-store work. The failing child agent exits with
`SystemExit: 1` while validating the generated coder prompt. The prompt contains a relative SDD file reference like:

```text
#gh:sase @.sase/sdd/tales/202607/agent_reply_subsection_id.md

The above plan has been reviewed and approved. Implement it now.
```

That file exists in the primary SDD store:

```text
/home/bryan/projects/github/sase-org/sase/.sase/sdd/tales/202607/agent_reply_subsection_id.md
```

but it does not exist in the managed workspace clone where the coder runs:

```text
<managed workspace>/.sase/sdd/tales/202607/agent_reply_subsection_id.md
```

`process_file_references()` validates relative `@` paths from the current working directory and calls `sys.exit(1)` on
missing files, so the coder child dies before it can begin implementation.

## Diagnosis

There are two interacting issues.

First, the plan reference builders in `src/sase/axe/run_agent_exec_plan_sdd.py` still emit `.sase/sdd/<kind>/...` for
non-version-controlled SDD plans whenever the saved plan lives under the resolved SDD store. This assumes the runner CWD
has a fresh `.sase/sdd` view of the store.

Second, recent commits added `ensure_workspace_sdd_link()` so managed workspaces try to symlink `.sase/sdd` to the
primary store. That is useful, but it is a setup-time mitigation, not a safe reference contract:

- the failing agent was already in flight before the stale-clone replacement commit landed;
- existing workspace-local `.sase/sdd` clones can be stale at handoff time;
- preserved dirty/ahead clones intentionally remain real directories;
- the approved-plan code writes the new SDD plan and immediately builds the coder prompt in the same process, so the
  prompt builder should not rely on a prior workspace setup side effect.

The local evidence matches this exactly: the primary store has the new `agent_reply_subsection_id.md` tale at commit
`a88208f...`, while this managed workspace's real `.sase/sdd` clone is still at `117a5e...` and does not know that path.
The current tests also encode the fragile behavior by expecting `.sase/sdd/...` for sibling/managed non-VC store
layouts.

## Fix Strategy

Make the generated follow-up reference resolve from the follow-up agent's actual workspace CWD. Keep the short
`.sase/sdd/<rel>` form only when it really exists from that workspace; otherwise use the absolute committed plan path in
the primary SDD store.

This is narrower and more robust than trying to refresh every workspace clone at the point of handoff. Absolute paths
are already accepted by `process_file_references()`, and home-rooted absolute files are copied into the agent's
`.sase/home` context when needed.

## Implementation Plan

1. Update `src/sase/axe/run_agent_exec_plan_sdd.py`.
   - Add a shared helper for committed plan references.
   - For version-controlled/in-tree SDD plans, preserve the current `sdd/...` workspace-relative behavior.
   - For local/separate-repo SDD plans, compute the existing candidate `.sase/sdd/<rel>`, but return it only if
     `<workspace_dir>/.sase/sdd/<rel>` exists.
   - If that candidate is not reachable, return `sdd_plan_path.as_posix()`.
   - Route both `build_saved_plan_ref()` and `build_sdd_plan_ref()` through this helper so coder, epic, and legend
     follow-ups use the same resolution rule.

2. Update tests that currently assert the broken behavior.
   - Change the managed/sibling non-VC epic test to expect the absolute primary SDD-store path when the workspace-local
     `.sase/sdd` path does not resolve.
   - Add a non-VC symlinked/co-located case that still expects `.sase/sdd/<rel>` to prove existing working layouts keep
     their short refs.

3. Add coder-specific regression coverage.
   - Cover `build_saved_plan_ref()` for:
     - managed separate-repo store with stale/missing workspace-local path: expect the absolute committed plan path;
     - co-located or symlinked `.sase/sdd`: expect `.sase/sdd/<rel>`;
     - in-tree version-controlled SDD: expect `sdd/<rel>`;
     - no committed plan: expect the archived fallback plan file.
   - Add an integration-style `handle_plan_marker()` assertion for a normal approved coder handoff under separate-repo
     SDD where the `@` reference in `state.current_prompt` resolves from the workspace CWD.

4. Leave `ensure_workspace_sdd_link()` in place.
   - It is still valuable for workspace-local SDD browsing and for already valid short references.
   - The prompt builder becomes the final safety boundary, so stale or preserved workspace clones no longer crash
     approved-plan coder launches.

5. Verification.
   - Run the focused plan-chain SDD reference tests.
   - Run the existing approved-plan follow-up tests.
   - Run `just install` and then the required `just check` after code changes.

## Expected Outcome

New approved `Tale`/coder follow-ups will no longer die with missing `@.sase/sdd/...` validation errors in managed
workspaces. When the workspace has a valid local SDD view, prompts stay concise. When it does not, prompts point at the
authoritative primary SDD-store file and still pass validation.
