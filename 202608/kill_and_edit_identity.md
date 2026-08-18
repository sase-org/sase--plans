---
tier: tale
title: Make ,x kill-and-edit rebuild the agent's real identity
goal:
  ACE's ,x (and `sase agent restart`) always seeds a relaunch prompt whose %id directive
  names the agent being relaunched, keeps its clan membership, and forces name reuse --
  or refuses loudly without killing anything.
size: medium
proposed_by: bbugyi200.athena.06f
create_time: 2026-08-18 14:24:29
status: wip
---

# Plan: Make `,x` kill-and-edit rebuild the agent's real identity

## Problem

`,x` (leader-mode `kill_and_edit`) kills the selected agent and seeds its relaunch
prompt into the prompt bar. For some agents it seeds a prompt that describes a
_different_ identity than the agent it just killed. The agent is already dead by the
time the user notices, and the seeded prompt silently relaunches something else.

Reported instance: `,x` on the ACE row for `sase-pw.1` (phase 1 of epic clan `sase-pw`,
tribe `epic`) seeded

```text
%id(!plan, family=sase-pw.1, bead=sase-pw.1)
#gh:gh_sase-org__sase
%model:@medium
%auto
#bd/work_phase_bead:sase-pw.1
```

while the agent's stored `raw_xprompt.md` was

```text
#gh:gh_sase-org__sase
%id(sase-pw.1, bead=sase-pw.1)
%clan(sase-pw, tribe=epic, summary_script=sase_clan_summary_epic)
%model:@medium
%auto
#bd/work_phase_bead:sase-pw.1
```

The correct rewrite -- the one a later marked-set `,x` on the same agent produced, and
which relaunched successfully -- is `%id(!1, clan=sase-pw, bead=sase-pw.1)`.

Two things are wrong with the seeded prompt:

1. It attaches the agent to a family named after itself (`family=sase-pw.1`), so the
   relaunch is no longer phase 1 of the epic; it is a `plan` member of a family whose
   root does not exist.
2. `%clan(sase-pw, tribe=epic, summary_script=sase_clan_summary_epic)` is gone. Nothing
   in the seeded prompt carries the clan, the `epic` tribe, or the epic summary script,
   so the relaunched agent is orphaned from the epic entirely.

## Root cause

`prepare_kill_edit_agent_prompt` in
`src/sase/ace/tui/actions/agent_workflow/_entry_relaunch.py` takes the "exact family
member" branch whenever the row has `agent_family` and `role_suffix` and is not
`agent_family_parallel`:

```python
    family_name: str | None = None
    agent_family = getattr(agent, "agent_family", None)
    role_suffix = getattr(agent, "role_suffix", None)
    if (
        agent_family
        and not getattr(agent, "agent_family_parallel", False)
        and role_suffix
    ):
        family_name = (...)
```

That branch routes into `prepare_kill_and_edit_prompt`
(`src/sase/agent/relaunch_prompt.py`) which calls `rewrite_prompt_family_member_name`
(`src/sase/xprompt/_directive_edit_identity.py`). That rewriter deliberately drops clan
and tribe syntax; its docstring says "the family resolver restores inherited clan and
tribe context from the authoritative parent."

The reported agent is `sase-pw.1--plan` -- the plan-chain **root** of family
`sase-pw.1`. ACE presents a family root under the family's name, which is why the row
reads `sase-pw.1`. Its metadata is `agent_family: "sase-pw.1"`,
`agent_family_role: "root"`, `plan_chain_root: true`, `role_suffix: "--plan"`, so the
family branch fires.

For a family root there is no authoritative parent to inherit clan context from -- the
root _is_ the origin, and its own prompt carries the `%clan(...)` declaration. Applying
the member rewrite to a root therefore does two irreversible things: it points `family=`
at the root's own name, and it deletes the only copy of the clan declaration.

The feature that introduced this branch (commit `330c25856`, "fix: restart exact agent
family members") was designed for non-root members such as `--code` and `--1`, which
attach to a family root that still holds the clan.
`tests/ace/tui/test_family_member_relaunch.py` only ever exercises a `--code` child; no
test covers a root.

`sase agent restart` has the identical defect through `_family_rewrite_args`
(`src/sase/agent/restart.py`), which reads `agent_family` / `role_suffix` /
`agent_family_parallel` from `agent_meta.json` and likewise ignores
`agent_family_role == "root"` and `plan_chain_root`.

### Reproduction (verified against the current tree)

```python
from sase.agent.relaunch_prompt import prepare_kill_and_edit_prompt

raw = """#gh:gh_sase-org__sase
%id(sase-pw.1, bead=sase-pw.1)
%clan(sase-pw, tribe=epic, summary_script=sase_clan_summary_epic)
%model:@medium
%auto
#bd/work_phase_bead:sase-pw.1"""

prepare_kill_and_edit_prompt(
    raw, "sase-pw.1--plan",
    family_name="sase-pw.1", role_suffix="--plan", phase_bead_id="sase-pw.1",
)
# -> '%id(!plan, family=sase-pw.1, bead=sase-pw.1)\n#gh:...' (clan lost)
```

### Second, independent defect: forced reuse silently no-ops

`force_name_reuse_in_prompt` (`src/sase/agent/retry_prompt.py`) returns the prompt
unchanged when it contains no `%i`-family directive. Most agents are launched from a
bare prompt with no `%id`, so:

```python
prepare_kill_and_edit_prompt("#gh:gh_sase-org__sase Describe this repo.", "068")
# -> '#gh:gh_sase-org__sase Describe this repo.'   (no %id at all)
```

`,x` on such an agent kills it and seeds a prompt that will allocate a **brand new**
name instead of reusing `068`. `,x` means "relaunch this agent", so this is wrong, and
it is silent. `ensure_forced_name_reuse` already produces the right thing
(`%id:!068\n#gh:...`); the kill-and-edit path just does not use it.

## Approach

All fixes land in the shared, non-TUI seam (`src/sase/agent/relaunch_prompt.py` plus its
two callers) so ACE `,x` and `sase agent restart` cannot drift. Do not fork TUI-only
identity logic.

### 1. Rewrite family roots under the family name, not as a member of themselves

Teach `prepare_kill_and_edit_prompt` about family roots. Add an explicit keyword (for
example `is_family_root: bool = False`); when it is true, skip the
`rewrite_prompt_family_member_name` branch entirely and instead name the prompt with the
family reference name and force reuse, via the existing `ensure_forced_name_reuse`.

Callers supply the flag:

- `prepare_kill_edit_agent_prompt` (`_entry_relaunch.py`) passes
  `agent.is_family_root_entry` (already defined on `Agent` in
  `src/sase/ace/tui/models/agent.py` as
  `not is_workflow_child and (plan_chain_root or agent_family_role == "root")`). Use the
  row's family reference name (`presented_family_reference_name()`, falling back to
  `prompt_facing_agent_name(agent_family)`) as the reuse name.
- `_family_rewrite_args` (`restart.py`) derives the same condition from
  `agent_meta.json`:
  `meta.get("plan_chain_root") is True or meta.get("agent_family_role") == "root"`.

Verified outputs of this approach against the current tree:

| Row                                              | Stored prompt                                                  | New `,x` output                                                |
| ------------------------------------------------ | -------------------------------------------------------------- | -------------------------------------------------------------- |
| `sase-pw.1--plan` (epic clan root)               | `%id(sase-pw.1, bead=...)` + `%clan(sase-pw, tribe=epic, ...)` | `%id(!1, clan=sase-pw, bead=sase-pw.1)`                        |
| `06d--plan` (plain plan root, no clan, no `%id`) | `#gh:... #plan`                                                | `%id:!06d`                                                     |
| `sase-8u.4.2--code` (non-root member)            | fork body                                                      | `%id(!code, family=sase-8u.4.2, bead=sase-8u.4.2)` (unchanged) |

The epic case now matches the prompt that actually relaunched the agent successfully,
and the plain-root case is strictly better than today's `%id(!plan, family=06d)`
self-attach. The non-root member case -- the one commit `330c25856` was written for --
is untouched.

### 2. Never let a rewrite drop clan membership

Defence in depth, independent of the root flag. Before returning, if the stored prompt
carried clan membership (a `%clan(...)` declaration or a `clan=` kwarg on `%id`) and the
rewritten prompt carries none, that rewrite is wrong. `%id` treats `clan=`, `family=`,
and `tribe=` as mutually exclusive, so the family form can never carry the clan: the
correct response is to refuse the family rewrite and use the clan-preserving path
instead.

### 3. Verify the rewritten identity before anything is killed

`,x` currently seeds whatever text the rewriter returned. Add a verification step at the
end of `prepare_kill_and_edit_prompt` that re-parses its own output
(`sase.xprompt.directives.extract_prompt_directives`) and requires:

- the result has a top-level `%id` naming an identity;
- that identity resolves to the agent being relaunched -- clan form: same clan and
  member suffix; family form: same family, and the family name is **not** the relaunched
  agent's own presented name (no self-attach); plain form: same name;
- forced reuse (`!`) is present;
- clan membership is preserved per item 2.

On failure, raise a typed error rather than returning a prompt. Wire the two callers:

- ACE: the resolver already runs inside `schedule_relaunch_prompt_resolution` _before_
  the kill/confirm step, and an exception there notifies and kills nothing. Keep that
  ordering and make the notification name the reason (agent name plus what the rewrite
  produced) instead of the current generic "Unable to prepare agent relaunch prompt".
  The bulk marked-set path (`_bulk_kill_marked_agents_and_edit` in
  `src/sase/ace/tui/actions/agents/_marking_kill.py`) already refuses the whole batch
  when any prompt is missing; extend that to verification failures so one bad row never
  kills the rest.
- CLI: `plan_agent_restart` turns it into an `AgentRestartError` with a `reason` and a
  hint, alongside its existing `not_found` / `no_prompt` / `multi_segment` cases.

### 4. Always force name reuse when the agent's name is known

Route the non-family path through `ensure_forced_name_reuse` (name the prompt, then
force reuse) rather than bare `force_name_reuse_in_prompt`, so a prompt with no `%id`
gets one instead of silently allocating a new name. Keep the existing clan-demotion
behaviour: `ensure_forced_name_reuse` already delegates to `force_name_reuse_in_prompt`,
which handles the clan-declaration case.

If a reuse name genuinely cannot be determined, item 3's verification must surface that
as a refusal, not a silent new-agent launch.

### 5. Refuse `,x` on a clan-container row from the focused path

`sase agent restart` refuses container names (`_refuse_container_name`). ACE's focused
`,x` (`_kill_and_edit_agent`) has no such guard -- only the bulk path expands containers
via `clan_members_for_container`. A clan container row is a synthetic selection target
whose artifacts belong to its first member, so `,x` on it seeds that member's prompt
renamed to the clan. Refuse it with a message pointing at marking the members instead.

## Non-goals

- No feature flag. This is a bug fix to existing behaviour, not a disabled beta, an
  early-landed path, or a deprecation with an old branch to keep reachable.
- Do not move relaunch-prompt rewriting into the Rust core repo. The behaviour already
  lives in a shared Python seam consumed by both the TUI and the CLI; relocating it is a
  separate migration and is out of scope here.
- Do not change `rewrite_prompt_family_member_name`'s existing semantics for genuine
  non-root members. Its clan-dropping behaviour is correct there.
- Do not change how `,x` chooses between the focused row and the marked set.

## Tests

Add regression coverage that would have caught the reported bug:

- `tests/ace/tui/test_family_member_relaunch.py`: a family **root** row
  (`agent_family_role="root"`, `plan_chain_root=True`, `role_suffix="--plan"`,
  `agent_clan` set) whose stored prompt declares `%clan(...)`. Assert `,x` seeds
  `%id(!1, clan=sase-pw, bead=sase-pw.1)` and that `%clan`/`family=` do not appear. Keep
  the existing `--code` child assertions green and unmodified.
- Same file: a plain plan root with no clan and no `%id` seeds `%id:!<family name>`.
- `tests/test_agent_restart_plan.py`: `sase agent restart` on the same root metadata
  produces the clan form, not `family=`. The existing `family=` assertions for non-root
  members must stay green.
- New coverage for the verification gate: a rewrite that would emit a self-attaching
  `family=<own name>` or that loses clan membership raises, ACE notifies, and
  `app.killed == []` and `app.dismissed == []`.
- `tests/ace/tui/test_agent_bulk_kill_edit.py`: one unverifiable marked row aborts the
  whole batch without killing anything.
- Forced-reuse coverage: `,x` on an agent whose stored prompt has no `%id` seeds
  `%id:!<name>`.
- A clan-container row rejects focused `,x` with a warning and kills nothing.

Reuse the real prompts from the incident as fixtures so the regression is anchored to
observed behaviour rather than a synthetic shape.

## Verification

- `just check` while iterating.
- `just check-full` before landing, run only through `/sase_monitor` with a `--next`
  action, per the repo's two-speed verification rule.
- Manual smoke in ACE: `,x` on an epic phase row that is showing its plan-chain root,
  and confirm the seeded `%id` names that phase and keeps `clan=`.
