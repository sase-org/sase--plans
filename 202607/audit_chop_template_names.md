---
tier: tale
title: Use `@` template agent names for the recent-audit chops
goal:
  The `recent_bug_audit` and `recent_improvement_audit` chops propose agents named `audit_bugs.<project>.@` /
  `audit_improvements.<project>.@` so SASE allocates a short alphanumeric token (`audit_bugs.sase.0`, `.1`, ... `.z`,
  `.00`) instead of embedding a 12-character git revision in the agent name.
create_time: 2026-07-29 06:40:33
status: done
---

- **PROMPT:** [prompts/202607/audit_chop_template_names.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/audit_chop_template_names.md)
- **AGENTS:**
  - bbugyi200.athena.ns--code
- **COMMITS:**
  - [eff60e6](https://github.com/bbugyi200/bugyi-chops/commit/eff60e64bcef9d9e3eb042f318ab605e5fc4714e) — fix: use templates for audit agent names

# Plan: Use `@` template agent names for the recent-audit chops

## Context

Today the two recent-commit audit chops build their agent name from the target project and the abbreviated HEAD,
producing names such as `audit_bugs.sase.7270b986bf6f`.

- **Definition (the file to change):** the `bugyi-chops` GitHub repo, `bbugyi200/bugyi-chops`. The name is built in
  `src/bugyi_chops/recent_audits.py`, in `build_audit_result`:

  ```python
  safe_project = safe_fragment(project)
  revision = safe_fragment(head_short or "current", fallback="current")
  pr_name = f"{kind.pr_prefix}_{safe_project.replace('.', '_')}_{revision}"
  agent_name = f"{kind.agent_hood}.{safe_project}.{revision}"
  ```

  `AuditKind.agent_hood` is `audit_bugs` for `BUG_AUDIT` and `audit_improvements` for `IMPROVEMENT_AUDIT`.

- **Configuration:** the `chezmoi` linked repo, `home/dot_config/sase/sase_athena.yml`, under the `code_quality` lane.
  It sets `interval: 60`, `wait_runners: 0`, and per-chop `run_every: 60m`, `inhibit_if.agent_hood.hood` (`audit_bugs` /
  `audit_improvements`), and a `git.commits_since` trigger with `threshold: 200` and `checkpoint: on_action_success`.

  **This file needs no change.** Its chop descriptions reference the _hood_ inhibit guard and the checkpoint policy,
  never the agent-name format, and the hood names are unchanged by this plan. Do not edit the chezmoi repo.

### How the `@` marker works

An agent-name template contains exactly one `@`. SASE renders it with tokens from the shared auto-name sequence
`0, 1, ... 9, a, ... z, 00, 01, ...`. Because the marker here follows a `.`, no auto-ID dash is inserted, so
`audit_bugs.sase.@` renders directly as `audit_bugs.sase.0`, `audit_bugs.sase.1`, ... `audit_bugs.sase.z`,
`audit_bugs.sase.00`.

## Why no `sase` repo change is required

This was traced end to end and exercised against the live name registry before writing this plan. The implementer should
not need to touch the `sase` repo; if something below turns out to be wrong, stop and report rather than expanding
scope.

1. **Rust proposal validation accepts it.** For a non-clan proposal, `validate_chop_proposal` checks `agent_name` only
   with `validate_token` (non-blank, no whitespace, no NUL). The stricter `validate_dotted_agent_name` /
   `validate_clan_member_identity` path runs only when `clan` is set, and these proposals set no clan.
2. **Chop planning passes the template through.** `prepare_chop_proposals` keeps the configured name verbatim and marks
   `explicit_agent_name=True`. `plan_chop_proposals` only allocates concrete names for _clan_ groups; a non-clan
   proposal keeps `proposal.agent_name` as-is.
3. **The scaffolded prompt carries the template.** `scaffolded_prompt` emits `%id(audit_bugs.sase.@, tribe=chop)`.
   Directive extraction parses this as `name_explicit=True`, `name_template='audit_bugs.sase.@'`, `tribe='chop'`.
4. **Launch-name validation tolerates templates.** `_preflight_launch_name_requests` rejects force-reuse-plus-template
   and otherwise parses the template and validates the token-`0` render (`audit_bugs.sase.0`);
   `validate_launch_name_requests` skips reservation/collision checks for template requests.
5. **The parent allocates the concrete name before spawn.** `launch_agent_from_cwd` calls `plan_single_agent_name` →
   `PlannedNameAllocator.planned_name_for_prompt`, which sees the explicit templated `%id`, allocates the lowest free
   token under the allocation lock, reserves it against the child's artifacts dir, and injects
   `SASE_AGENT_PLANNED_NAME`. The child's `run_agent_directive_identity` then adopts that planned name because it
   matches the template.
6. **Bookkeeping records the concrete name.** `AgentLaunchResult.agent_name` is the planned concrete name, so the chop
   launch descriptor stores `audit_bugs.sase.3`, not the template. Lifecycle reconciliation matches launches to agent
   records by artifacts timestamp, not by name, so it is unaffected either way.
7. **Verified interactively.** `audit_bugs.sase.@` parses (prefix `audit_bugs.sase.`, empty suffix), renders the
   expected token series, allocates around already-reserved names, passes preflight and full launch-name validation, and
   `planned_name_for_prompt` on the exact scaffolded prompt returns `audit_bugs.sase.0`.

## Changes

All changes are in the `bugyi-chops` repo. Open it first with the `/sase_repo` skill
(`sase repo open gh:bbugyi200/bugyi-chops -r "<reason>"`) and use the printed path as the only path for reads and
writes. It is not a registered SASE project, so there is no project workspace for it.

### 1. `src/bugyi_chops/recent_audits.py`

In `build_audit_result`, replace the revision suffix in the agent name with the `@` marker:

```python
agent_name = f"{kind.agent_hood}.{safe_project}.@"
```

Keep `revision` exactly as it is — it is still used by `pr_name`, and `head` is still used by the prompt's
`head_description`. Only the `agent_name` assignment changes.

Add a brief comment explaining that `@` is the SASE agent-name template marker and that SASE allocates the concrete
alphanumeric token at launch, matching the surrounding comment density (this module currently has none, so keep it to
one short line or omit it — do not add a paragraph).

### 2. `tests/test_recent_audits.py`

- `test_bug_audit_ports_recent_commit_prompt_and_head_context`: change
  `assert proposal["agent_name"] == f"audit_bugs.demo-project.{head[:12]}"` to
  `assert proposal["agent_name"] == "audit_bugs.demo-project.@"`.
- `test_improvement_audit_uses_vars_and_current_head_fallback`: change
  `assert proposal["agent_name"] == "audit_improvements.demo_project.current"` to
  `assert proposal["agent_name"] == "audit_improvements.demo_project.@"`.
- Add regression coverage in the bug-audit test that the revision moved out of the name but is still carried elsewhere:

  ```python
  assert head[:12] not in proposal["agent_name"]
  assert f"#pr(recent_bug_audit_demo-project_{head[:12]})" in proposal["prompt"]
  ```

  The existing `#pr(recent_bug_audit_demo-project_` prefix assertion can be replaced by the exact form above, or kept
  alongside it — do not delete revision coverage outright.

- The existing `test_audit_fails_closed_for_missing_workspace_dir` needs no change.

### 3. `README.md`

Three places assert that the revision-bearing name is what gives AXE its dedupe, which stops being true. Correct only
these naming claims; leave the rest of the embedded sample config alone (it intentionally differs from the live chezmoi
config, e.g. it omits `inhibit_if`).

- In the `recent_bug_audit` sample description: replace "Its stable `audit_bugs.*` name lets AXE dedupe repeated
  proposals for the same project and revision." with wording that says the agent is named `audit_bugs.<project>.@`, so
  SASE assigns a short alphanumeric suffix at launch and the agent stays in the `audit_bugs` hood for hood-based inhibit
  guards.
- Make the matching edit in the `recent_improvement_audit` sample description (`audit_improvements.<project>.@`,
  `audit_improvements` hood).
- Update the prose after the sample ("uses a stable `audit_bugs.*` or `audit_improvements.*` agent name, and includes a
  project- and revision-specific `#pr(...)` rollover") so it states that the agent name is a `@` template resolved by
  SASE at launch while the `#pr(...)` rollover remains project- and revision-specific.

## Behavior change to accept

The revision suffix currently doubles as an accidental idempotency guard, and this plan deliberately removes it. That is
the correct outcome, not a regression:

- With `checkpoint: on_action_success`, a _failed_ audit agent leaves the commit window open, so the chop re-proposes at
  the same HEAD on its next eligible tick.
- Today that re-proposal computes the identical `audit_bugs.sase.<head12>` name, `launch_chop_proposals` catches
  `AgentNameLaunchCollisionError`, and because `explicit_agent_name` is true the proposal is _skipped_ instead of
  retried — which directly contradicts the configured intent recorded in the chezmoi description ("preserving failed
  reviews for retry").
- With `@`, the retry launches under a fresh token. The rate stays bounded by `run_every: 60m`, the lane's
  `interval: 60` and `wait_runners: 0`, `git.commits_since` `threshold: 200`, and the `inhibit_if.agent_hood` guard that
  already blocks concurrent audits in the same hood.

If true one-successful-audit-per-HEAD semantics are ever wanted, the correct mechanism is a proposal-level `dedupe_key`
(e.g. `f"{kind.name}:{safe_project}:{revision}"`): `apply_chop_once_per` honors a proposal `dedupe_key` even with no
chop-level `once_per` block, and `chop_lifecycle` releases the key when the launched agent fails, so retries still work.
**This is out of scope for this tale — do not add it.** Raise it with the user instead if it comes up.

## Known caveats (no action required)

- **Legacy names still resolve as "latest".** 46 legacy `audit_bugs.sase.<12-hex>` / `audit_improvements.*` names remain
  registered, and a 12-character lowercase-hex suffix is itself a _valid_ template token. Allocation is unaffected
  (`allocate_agent_name_template` scans `0, 1, 2, ...` and takes the first free candidate), but token comparison orders
  by length first, so resolving `audit_bugs.sase.@` as a _reference_ (`%wait:`, `sase agent chat`) keeps returning a
  legacy hex name until those are wiped. Nothing in these chops resolves the template as a reference, so there is no
  functional impact. Mention it to the user; do not wipe anything as part of this tale.
- **Preview cosmetics.** For non-clan proposals, `proposal_previews` and the chop render table
  (`sase axe chop run ... --dry-run --chop-verbose`) display the raw `audit_bugs.sase.@` rather than a speculative
  concrete name, because only clan groups get concrete names during planning. The recorded launch shows the real name.
  Cosmetic only, in the `sase` repo, and out of scope here.

## Verification

In the `bugyi-chops` checkout obtained through `/sase_repo`:

```bash
just install
just check   # ruff format --check, ruff check, mypy, pytest, build
```

`just check` must pass before reporting completion. Optionally, from a `sase` workspace, confirm the scaffolded prompt:

```bash
sase axe chop run 'recent_bug_audit[sase]' -L code_quality --dry-run --chop-verbose
```

and check the previewed prompt contains `%id(audit_bugs.sase.@, tribe=chop)`.

No files change in the `sase` repo, so `just check` in the `sase` repo is not required.

## Deliverable

A single focused change in `bbugyi200/bugyi-chops` covering `src/bugyi_chops/recent_audits.py`,
`tests/test_recent_audits.py`, and `README.md`, verified by `just check`. Commit only if the user asks or a
post-completion finalizer triggers it; in that case use the `/sase_git_commit` skill from inside the `bugyi-chops`
checkout.
