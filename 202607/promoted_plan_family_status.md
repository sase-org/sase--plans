---
tier: tale
title: Project plan-family status onto promoted (rename-on-attach) family roots
goal:
  A lane whose plan chain started in a family continuation shows the plan-family status (TALE / TALE APPROVED / WORKING
  TALE / TALE DONE) on its root row instead of a stale DONE.
create_time: 2026-07-31 07:23:59
status: done
---

- **PROMPT:** [prompts/202607/promoted_plan_family_status.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/promoted_plan_family_status.md)

# Plan: Project plan-family status onto promoted family roots

## Problem

In the Agents tab the `pv` lane row renders `DONE` while its newest family member `pv--1` renders `TALE` (a submitted
tale awaiting review). The lane's own fold header already counts the pending review correctly
(`pv · 2 agents · 1 awaiting`), so the lane is simultaneously reported as finished _and_ as awaiting input. Because
grouping is by status bucket, the lane sorts into **Done** and disappears from the user's "needs my attention" view even
though a tale is sitting unreviewed.

Observed rows (screenshot state, 2026-07-31 07:00:52):

```
✓ Done ─────────────────────────── 3 agents · 1 awaiting
  ▶ pv                             2 agents · 1 awaiting
    bob-cli (DONE) ×6 −3 pv        07:00:39 · 6m38s     <-- should be TALE
      main    (ANSWERED) pv--0     06:54:20 · 2m15s
      bob-cli (TALE)     pv--1     07:00:39 · 4m22s
      diff    (DONE)
```

The same family is mislabeled at every later plan-chain phase too. After the tale was approved and the coder ran, the
live loader reports:

| row        | actual | correct         |
| ---------- | ------ | --------------- |
| `pv` root  | `DONE` | `TALE DONE`     |
| `pv--1`    | `DONE` | `TALE APPROVED` |
| `pv--code` | `DONE` | `TALE DONE`     |

By contrast the `pw` lane in the same screenshot — a family whose _root_ was launched as a plan agent — correctly shows
`WORKING TALE` while its coder runs and `TALE DONE` afterwards.

## Reproduction

`apply_status_overrides` reproduces the bug from model state alone; no fixtures or artifact directories are needed. The
three rows below mirror the real `agent_meta.json` records for the `pv` family
(`~/.sase/projects/gh_bobs-org__bob-cli/artifacts/ace-run/202607/31/{20260731065155,20260731065616,20260731070142}`):

- root: `agent_type=WORKFLOW`, `raw_suffix="20260731065155"`, `role_suffix="--0"`, `agent_family="pv"`,
  `agent_family_role="root"`, `plan_chain_root=False`, `status="DONE"`, `questions_times=[06:54:20]`,
  `question_response_path` set.
- main workflow step (passed as `workflow_agent_steps`): `parent_workflow` set, `parent_timestamp="20260731065155"`,
  `step_type="agent"`, `parent_step_index=None`, `role_suffix="--0"`, `agent_family_role="q"`, `status="DONE"`.
- family member: `raw_suffix="20260731065616"`, `parent_timestamp="20260731065155"`, `role_suffix="--plan"`,
  `agent_family_role="q"`, `status="DONE"`, `plan_times=[07:00:39]`, `plan_path`/`sdd_plan_path` pointing at a
  `tier: tale` plan, `plan_action=None`.

Result today: root `DONE`, main step `ANSWERED`, member `TALE` — byte-for-byte the screenshot.

## Root cause

### 1. `plan_chain_root` is decided once, at family-promotion time, and never revisited

`sase/agent/_family_promotion.py::family_root_role_suffix` picks the original agent's family-member slot when the first
`%n(...)` member attaches. A root that was launched as a plan agent (canonical `--plan` role suffix, `plan_chain_root`,
`approve`, or `plan` meta) is promoted to `--plan` and gets `plan_chain_root: True`; every other agent is promoted to
`--0` with no `plan_chain_root`.

The `pv` lane took the second path: it was launched as a plain agent, asked a question via `/sase_questions`, and was
promoted to `pv--0` at that moment — before any plan existed. The plan chain only started later, in the question
continuation `pv--1`. When that continuation submitted its plan, `sase/axe/run_agent_exec_plan.py::handle_plan_marker`
rewrote **the continuation's** `role_suffix` to `--plan`
(`update_meta_suffix(state.current_artifacts_dir, PLAN_CHAIN_PLAN_SUFFIX)`), which is correct — but nothing marks the
_family_ as having become a plan family. The root's `plan_chain_root: False` / `role_suffix: "--0"` is durable metadata
that was accurate when written and is still literally accurate: `pv--0` really was a plain agent.

### 2. The whole plan-family status projection is gated on that stale root flag

`sase/ace/tui/models/_agent_status_family.py::is_root_plan_workflow` answers "is this a plan family?" purely from the
root row:

```python
if agent.plan_chain_root:
    return True
return agent.agent_type == AgentType.WORKFLOW and (
    canonical_plan_chain_suffix(agent.role_suffix) == PLAN_CHAIN_PLAN_SUFFIX
)
```

For the `pv` root that is `False` (`canonical_plan_chain_suffix("--0") == "--0"`), which switches off every plan-family
projection in `sase/ace/tui/models/_agent_status_apply.py::apply_status_overrides`:

- `approved_followup_planner_status` gate — planner member never becomes `TALE APPROVED`.
- `active_approved_plan_handoff_status` gate — running coder never becomes `WORKING TALE`.
- `is_completed_plan_handoff_child` gate — finished coder never becomes `TALE DONE`.
- the final root-mirroring block: `is_plan_root = is_root_plan_workflow(parent)` is `False`, so the root falls through
  to "plain-agent roots keep their own terminal status" and stays `DONE`. The active/waiting rules above it do not fire
  because every member has finished.

That final fallback was introduced deliberately in `7b53fec4b` (plan `~/.sase/plans/202607/root_agent_status_mirror.md`,
"Fallback: … plan-workflow roots mirror the newest child verbatim; other roots keep their own status"). It is not wrong
in itself — the bug is that `pv` _is_ a plan family and is not being recognized as one.

Verified end to end: forcing `is_root_plan_workflow` to return `True` for this root turns the repro into root `TALE`,
main step `ANSWERED`, member `TALE`, and the post-approval state into root `TALE DONE`, member `TALE APPROVED`, coder
`TALE DONE`.

### 3. The tale/plan flavor lives on the planner _member_, not on the root

`done_handoff_status`, `active_approved_plan_handoff_status`, and `_approved_planner_status` all decide "tale vs plan"
by reading `parent.plan_action` (plus the parent's own sticky status). In a native plan family the root's artifact
directory _is_ the planner's, so its `agent_meta.json` carries `plan_action: "tale"` — confirmed on the local `pw` and
`pz` families. In a promoted family the planner is a member (`pv--1` carries `plan_action: "tale"`), and the root has no
`plan_action` at all.

Consequence: fixing only the gate makes the same rows report `PLAN DONE` / `WORKING PLAN` instead of the correct
`TALE DONE` / `WORKING TALE` — a different wrong label. Both halves are needed.

### 4. Member projection must move with the gate, or the "awaiting" count double-counts

`sase/ace/tui/models/agent_family_members.py` projects a family container into concrete member rows.
`_concrete_planner_child` and `_root_represents_member` branch on `Agent.is_plan_family_root_entry`, which is a second,
independent spelling of the same stale question (`plan_chain_root` or a `--plan*` role suffix). Today the `pv` root is
counted as concrete member #0 while the main step is skipped; that is why the header reads `2 agents · 1 awaiting`.

If the status gate is widened but `is_plan_family_root_entry` is not, the root row keeps standing in as member #0 while
now carrying the _mirrored_ `TALE` status, and the header becomes `2 agents · 2 awaiting`. Verified: with both
predicates widened together the projection becomes `[main step (ANSWERED → Done), pv--1 (TALE → Stopped)]` and the
header stays `2 agents · 1 awaiting`.

## Fix plan

The fix is read-side and presentation-only. `plan_chain_root: False` on the `pv` root stays correct durable metadata;
what changes is that the TUI derives "this family has plan-family projection semantics" from the family's _members_
rather than from a snapshot of the root taken before the plan chain existed. This also repairs already-recorded history,
which a write-side metadata change could not.

### Step 1 — a shared "plan-chain family member" predicate

In `src/sase/ace/tui/models/_agent_status_family.py`, add a module-level predicate:

```python
PLAN_CHAIN_MEMBER_ROLES = frozenset({"plan", "code", "epic", "commit", "feedback"})
_PLAN_CHAIN_MEMBER_SUFFIXES = frozenset({
    PLAN_CHAIN_PLAN_SUFFIX,
    PLAN_CHAIN_CODER_SUFFIX,
    PLAN_CHAIN_EPIC_SUFFIX,
    PLAN_CHAIN_COMMIT_SUFFIX,
})


def is_plan_chain_family_member(agent: Agent) -> bool:
    """Return True when a family member row belongs to a plan chain."""
```

It returns `True` when the row is a non-parallel family-member child and any of:

- `agent_family_role(agent)` (from `._agent_status_roles`) is in `PLAN_CHAIN_MEMBER_ROLES`;
- `canonical_plan_chain_suffix(agent.role_suffix)` is in `_PLAN_CHAIN_MEMBER_SUFFIXES` or starts with
  `PLAN_CHAIN_PLAN_SUFFIX` (covers `--plan-<token>` feedback suffixes);
- `agent.plan_times` is non-empty (the member submitted a plan).

The suffix clause is what catches `pv--1`, whose stored `agent_family_role` is `"q"` while its rewritten `role_suffix`
is `--plan`. `PLAN_CHAIN_QUESTION_SUFFIX` is deliberately **not** a member suffix: a question continuation on its own
does not make a family a plan family.

### Step 2 — a runtime-derived plan-family marker on the root row

Add to `AgentState` in `src/sase/ace/tui/models/_agent_state.py`, next to `wait_display_source`, a runtime-only field:

```python
# Set when a family root's members reveal a plan chain that started after the
# root was promoted. Derived during status normalization; not serialized.
derived_plan_family_root: bool = field(default=False, compare=False, repr=False)
```

Confirm it is excluded from `agent_bundle` serialization, `repro` capture, and `_dedup` merge, exactly like
`wait_display_source` and `is_synthetic_planner`.

Add a setter in `_agent_status_family.py`:

```python
def mark_derived_plan_family_roots(
    children_by_parent: dict[str, list[Agent]],
    parent_by_suffix: dict[str, Agent],
) -> None:
```

For each `(parent_timestamp, children)` pair, resolve the parent and set `parent.derived_plan_family_root = True` when
the parent `is_family_root_entry` and any child satisfies `is_plan_chain_family_member`.

The marker is **sticky**: only ever set, never cleared. `apply_status_overrides` runs repeatedly over the same in-memory
rows (including after an artifact-delta merge that may carry a partial agent list), and a family that has entered a plan
chain never leaves it. This mirrors the existing "repeated normalization must not overwrite the root projection"
handling in the same pass.

### Step 3 — teach both plan-family predicates about the marker

- `_agent_status_family.py::is_root_plan_workflow`: after the existing `if agent.plan_chain_root: return True`, add
  `if agent.derived_plan_family_root: return True`. The leading
  `if agent.is_child_row or agent.agent_family_parallel: return False` guard stays first.
- `agent.py::Agent.is_plan_family_root_entry`: after the `plan_chain_root` branch, treat `self.derived_plan_family_root`
  as satisfying the property (the `is_family_root_entry` guard already runs first).

Update both docstrings to say the answer is "root metadata _or_ a plan chain the family entered later".

Leave `plan_chain_root` reads alone everywhere else. In particular
`src/sase/ace/tui/widgets/prompt_panel/_agent_display_content.py` and `src/sase/ace/tui/agent_context_members.py`
compute `is_promoted_root` from `plan_chain_root` directly and must keep doing so: `pv` genuinely _is_ a promoted root,
and its `--0` first member must keep its identity and display slot.

### Step 4 — call the derivation from `apply_status_overrides`

In `src/sase/ace/tui/models/_agent_status_apply.py`, immediately after
`children_by_parent = children_by_parent_timestamp(all_agents)` and **before** the existing parent→child
`copy_missing_plan_metadata` loop, call `mark_derived_plan_family_roots(children_by_parent, parent_by_suffix)`.

Placing it _after_ `ensure_synthetic_planner_children` is deliberate and load-bearing: that helper is also gated on
`is_root_plan_workflow`, and running the derivation first would let it synthesize a phantom `--0` planner row for a
derived plan family whose concrete main workflow step is not loaded (incomplete-history tiers). Deriving afterwards
means derived families never gain synthetic planner rows; they keep the promoted-root-as-first-member projection that
those tiers already use. Record this as an explicit non-goal in a comment.

Every consumer of `is_root_plan_workflow` inside the pass runs later than this call site, so all of them observe the
marker.

### Step 5 — resolve the tale/plan flavor from the family's planner member

Add to `_agent_status_family.py`:

```python
def pull_plan_metadata_from_family_members(
    children_by_parent: dict[str, list[Agent]],
    parent_by_suffix: dict[str, Agent],
) -> None:
```

For each family root where `is_root_plan_workflow(parent)` holds, iterate its `is_plan_chain_family_member` children
sorted by `child_launch_time` **descending** and call the existing `copy_missing_plan_metadata(parent, member)` for
each. Because that helper only fills fields that are `None`, the newest member wins per field and native plan roots
(which already carry their own values) are a no-op — verified: every local plan root with `plan_times` already has
`plan_action`.

Call it from `apply_status_overrides` directly after `mark_derived_plan_family_roots`, still before the parent→child
copy loop, so the pass performs a family-wide union: pull missing plan metadata up to the root, then push it down to
members that lack it.

Deliberately do **not** propagate `plan_times` upward. `copy_missing_plan_metadata` does not touch it, and it must stay
that way: `planner_child_status` returns `ANSWERED` for the root's own logical child via the
`has_followup_child and parent.questions_times and not parent.plan_times` branch, and giving the root borrowed
`plan_times` would flip `pv--0` from `ANSWERED` to `TALE APPROVED`. Add a comment recording this constraint.

With the root's `plan_action` populated, `done_handoff_status`, `active_approved_plan_handoff_status`, and
`_approved_planner_status` all resolve the tale flavor with no further changes.

## Expected outcome

For the `pv` family, verified against a monkeypatched prototype of the design above:

| state                        | root row       | `pv--0` step | `pv--1`         | `pv--code`     |
| ---------------------------- | -------------- | ------------ | --------------- | -------------- |
| tale submitted, unreviewed   | `TALE`         | `ANSWERED`   | `TALE`          | —              |
| tale approved, coder running | `WORKING TALE` | `ANSWERED`   | `TALE APPROVED` | `WORKING TALE` |
| coder finished               | `TALE DONE`    | `ANSWERED`   | `TALE APPROVED` | `TALE DONE`    |

Identical to what the natively-rooted `pw` plan family already shows. The lane's fold header stays
`2 agents · 1 awaiting`, and the lane moves from the **Done** group into the **Stopped** group while the tale awaits
review.

## Blast radius

Measured against the full local agent history (1920 loaded rows, 875 roots): exactly **one** family root — `pv--0`
itself — is newly recognized as a plan family by the Step 1 predicate. No native plan root changes behavior, because
`is_root_plan_workflow` already returned `True` for those and `copy_missing_plan_metadata` is a no-op on them.

A script equivalent to the following should be re-run after the change and reported in the commit message:

```python
roots = [a for a in load_all_agents() if a.raw_suffix and not a.is_child_row]
```

counting roots where `is_root_plan_workflow` flips from `False` to `True`.

## Tests

Extend `tests/test_agent_loader_status_override_followup_roots.py` (or add a sibling
`tests/test_agent_loader_status_override_promoted_plan_family.py`) with a shared builder for the promoted family shape
from the Reproduction section:

1. Promoted root `DONE` + `--plan` member with `plan_times` and no `plan_action` → root `TALE`, member `TALE`, main step
   `ANSWERED`.
2. Same family with `plan_action="tale"` on the member and a `RUNNING` `--code` member → root and coder `WORKING TALE`,
   member `TALE APPROVED`.
3. Same with the `--code` member `DONE` → root and coder `TALE DONE`, member `TALE APPROVED`.
4. Tier fidelity: a `tier: plan` plan file on the member yields `PLAN` / `PLAN APPROVED` / `WORKING PLAN` / `PLAN DONE`,
   proving the flavor is read from the member and not hardcoded.
5. Regression: a promoted family whose only member is a plain question continuation (no `--plan`/`--code` suffix, no
   `plan_times`) keeps today's behavior — root stays `DONE`, `is_root_plan_workflow` stays `False`, and no synthetic
   planner row is appended to the agent list.
6. Stickiness: running `_apply_status_overrides` twice over the same objects, the second time with only the root in the
   agents list, keeps the root's derived status (guards the artifact-delta merge path).
7. `plan_times` isolation: assert the root's `plan_times` stays empty after the pass and its logical `--0` child stays
   `ANSWERED`.

Add to `tests/ace/tui/models/test_agent_family_members.py`:

8. `concrete_agent_statuses` on the promoted plan family returns exactly two rows — the main workflow step bucketed
   `Done` and the `--plan` member bucketed `Stopped` — so the lane header stays `2 agents · 1 awaiting` and the root is
   no longer double-counted.

Keep green without edits: `tests/test_agent_loader_status_override_followup_roots.py` (existing cases),
`tests/test_agent_loader_status_override_plan_entry.py`, `tests/test_agent_loader_status_override_question_families.py`,
`tests/test_agent_loader_status_override_question_continuations.py`,
`tests/ace/tui/models/test_agent_summary_status_counts.py`. These 59+ tests pass on the current tree and must still pass
unchanged.

## Notes

- **Rust core boundary**: this is Agents-tab status projection and member counting, which already live in this repo's
  Python (`apply_status_overrides`, `agent_family_members`). No `sase-core` wire or API change. The durable
  `agent_meta.json` schema is unchanged — no new persisted field, no migration, no writer change in `sase/axe/`.
- **TUI perf**: the derivation and the metadata pull-up are two linear passes over the already-built
  `children_by_parent` index inside the existing `apply_status_overrides` pass. No new refresh path, no new render-cache
  key component, no per-second invalidation.
- **Why not a write-side fix**: marking the family root's `agent_meta.json` with `plan_chain_root: True` when a member
  submits a plan would leave every already-recorded family broken, would make `plan_chain_root: True` coexist with
  `role_suffix: "--0"` (a combination `family_root_role_suffix` never produces), and would flip the promoted-root
  display path in `_agent_display_content.py`, renaming the lane's first member. The read-side derivation avoids all
  three.
- Run `just install` before `just check`, per the workspace instructions.
