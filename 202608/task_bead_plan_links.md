---
tier: tale
title: Link task-bead workers' proposed plans to their task bead
goal: "A plan proposed by an agent that is working a task bead always archives with a
  managed association to that bead, exactly as a plan proposed from epic bead work
  already does.

  "
size: small
proposed_by: bbugyi200.athena.x4
create_time: 2026-08-10 10:14:18
status: wip
---

# Plan: Link task-bead workers' proposed plans to their task bead

## Problem

`sase plan propose` stamps a plan's managed bead association from **epic-work fields
only**. `handle_plan_propose_command` in `src/sase/main/plan_propose_handler.py`
resolves:

```python
phase_bead = os.environ.get(SASE_PHASE_BEAD_ID_ENV, "") or meta["phase_bead_id"]
epic_bead  = os.environ.get(SASE_EPIC_BEAD_ID_ENV, "")  or meta["epic_bead_id"]
active_bead = phase_bead or epic_bead
if target_tier == "tale" and active_bead:
    stamps["bead"] = active_bead
elif target_tier == "epic" and active_bead:
    stamps["parent_bead"] = active_bead
```

An agent launched to work a **task bead** has neither variable.
`sase bead work <task-id>` renders its prompt in `render_task_prompt`
(`src/sase/bead/work.py`) as:

```
#<vcs>:<project> #commit
%id(!<task-id>, bead=<task-id>)
%m:<model>
#<work_task_xprompt>:<task-id>
#plan          <- appended when phase_requires_plan(size), i.e. size large or xlarge
```

and `task_work_segment_env` (`src/sase/bead/work.py`) exports only
`SASE_BEAD_ID=<task-id>` plus the internal name bypass. There is no
`SASE_PHASE_BEAD_ID`, no `SASE_EPIC_BEAD_ID`, and no `SASE_EPIC_PLAN_REF`.

Consequences today, for exactly the `large`/`xlarge` task beads that are _supposed_ to
plan first:

- A proposed **tale** archives with no `bead:` frontmatter, so
  `refresh_bead_plan_section` is never called and the plan gets no `BEAD` header bullet.
  Nothing in the archived plan records which task bead it came from.
- A proposed **epic** archives with no `parent_bead:` frontmatter, so
  `selected_epic_parent_id` returns `None` and `create_and_launch_epic_from_plan`
  (`src/sase/bead/epic_from_plan.py`) creates a **top-level** epic that is unrelated to
  the task bead that spawned it. The task bead stays `in_progress`, assigned to an agent
  that has already exited with `epic_approved`, with nothing in the store tying it to
  the epic now doing its work.

## Approach

Add exactly one more source to the existing resolution chain: the proposing agent's own
bead. Do not introduce a new mechanism, a new frontmatter field, or a new link
direction. Tier rules (`bead` for tales, `parent_bead` for epics) stay exactly as they
are.

The agent's own bead is already available in two places, and both are already the
established pattern in this handler:

1. `SASE_BEAD_ID` (`SASE_BEAD_ID_ENV` in `src/sase/bead/work.py`). Unlike the epic-work
   variables, this one is **deliberately not popped** by `epic_work_metadata_from_env`
   (`src/sase/axe/run_agent_directive_metadata.py`: _"`SASE_BEAD_ID` is intentionally
   left untouched: it remains the commit workflow's bead attribution."_), so it is still
   set in the agent's environment when the agent runs `sase plan propose`. It is also
   set by `run_agent_directives` for any agent whose prompt carries a
   `%id(..., bead=<id>)` association, not just `sase bead work` launches.
2. `agent_meta.json`'s `bead_id` field, written by `build_agent_meta` and carried across
   a runner re-exec by `preserved_agent_metadata` (both in
   `src/sase/axe/run_agent_directive_metadata.py`). This is the durable fallback,
   mirroring how the handler already falls back for `phase_bead_id` / `epic_bead_id`.

Precedence must stay `phase_bead → epic_bead → agent_bead`. That is a strict superset of
today's behavior: a phase worker's `SASE_BEAD_ID` _is_ its phase bead and a land agent's
_is_ its epic bead, so the new fallback can never change an existing outcome — it only
fires where there is currently no association at all.

### Why `parent_bead` for a task-proposed epic is correct and safe

Checked against the Rust core (`sase-core`, `crates/sase_core/src/bead/`):

- `schema.rs` constrains parents as
  `(issue_type='phase' AND parent_id IS NOT NULL) OR (issue_type='plan') OR (issue_type='task' AND parent_id IS NULL)`.
  A `plan` bead may take **any** parent; the task restriction only forbids a task from
  being a _child_.
- `wire.rs` `IssueWire::validate` likewise rejects only `Task` issues that carry a
  `parent_id`.
- `mutation.rs` `next_child_id` yields `<parent>.<n>`, so a task-proposed epic becomes
  `sase-iq.1` with phases `sase-iq.1.1`, `sase-iq.1.2`, … This nesting shape already
  ships: a phase agent proposing a child epic produces `sase-5.2.1` today (see
  `docs/sdd.md`).

The useful consequence is the non-cascading close rule: `sase bead close <task-id>` is
rejected while the child epic or any of its phases is unclosed. A task that spawned an
epic can no longer be quietly closed before the work it spawned actually lands.

### Deliberately out of scope

- **The reverse (bead → plan) link.** `sase bead show` / `sase plan show` resolve a
  bead's plan through `issue.design` (`_resolve_plan_link` in
  `src/sase/bead/cli_detail_resolution.py`), which is only written for epic plan beads
  at creation. Phase beads whose workers author tales do not get a `design` write-back
  today either, so adding one only for task beads would be a new, inconsistent mechanism
  rather than the one this plan is asked to reuse. File it as a follow-up if wanted.
- **The ACE agent-detail PLAN lane for task agents.** `resolve_agent_plan_enrichment` in
  `src/sase/ace/tui/models/agent_associated_plan.py` early-returns
  `_AgentPlanEnrichment("task", association.bead_summary, None, ())` once a bead lookup
  confirms the `task` role, and `_task_bead_summary` builds an intentionally _plan-free_
  summary. That display choice predates task beads having plans and is a separate
  change.
- **Rust core changes.** None are needed: `bead` is already an accepted optional field
  on both tiers and `parent_bead` on epics (confirmed via `plan_frontmatter_schema`).
  Only the host-side resolution order changes, which is where the existing stamping
  already lives.
- **Memory files.** No `sase/memory/*.md`, `AGENTS.md`, or provider shim edits; that
  needs separate explicit user permission.

## Implementation

### 1. `src/sase/main/plan_propose_handler.py`

**a.** Extend `_read_agent_meta_associations` to also surface the agent's own bead:

```python
for key in ("phase_bead_id", "epic_bead_id", "epic_plan_ref", "bead_id"):
```

**b.** In `handle_plan_propose_command`, import `SASE_BEAD_ID_ENV` alongside the
existing three names from `sase.bead.work`, resolve the agent bead with the same
env-then-marker pattern used for the others, and extend the precedence chain:

```python
agent_bead = os.environ.get(
    SASE_BEAD_ID_ENV, ""
).strip() or meta_associations.get("bead_id", "")
active_bead = phase_bead or epic_bead or agent_bead
```

**c.** Update the explanatory comment block above that code (currently _"Plans proposed
from epic bead work inherit managed associations …"_) so it states the full contract:
the association is the proposing agent's active bead — its phase bead, else its epic
bead, else the bead the agent itself is working (a task bead from
`sase bead work <task-id>`, or any `%id(..., bead=<id>)` association). Keep the existing
note that the runner pops the epic variables in `epic_work_metadata_from_env` while
`SASE_BEAD_ID` survives, and that the tier rules are unchanged.

Everything downstream already works unmodified: `stamps["bead"]` triggers the existing
`refresh_bead_plan_section` call that writes the `BEAD` bullet, and
`stamps["parent_bead"]` flows through `set_frontmatter_fields` into
`selected_epic_parent_id` at launch.

Do **not** touch the `parent_plan` / `PARENT` bullet logic. A task worker has no
`SASE_EPIC_PLAN_REF`, and that is correct — its plan has no parent _plan_, only a parent
bead.

Do not add bead-existence validation. The handler does not validate the phase/epic ids
today, and an unresolvable `parent_bead` already fails at launch with an actionable
remedy from `require_epic_parent`.

### 2. `tests/plan_command_handler_helpers.py`

`clear_bead_work_association_env` must also
`monkeypatch.delenv("SASE_BEAD_ID", raising=False)`, otherwise a test run started from
inside a bead-associated agent leaks that agent's bead into the "no association"
assertions.

### 3. `tests/test_plan_command_handler_associations.py`

Add cases to `test_plan_command_stamps_associations_from_bead_work_env` (or a sibling
parametrized test that also sets `SASE_BEAD_ID`) covering:

| case              | env                                                       | expected                            |
| ----------------- | --------------------------------------------------------- | ----------------------------------- |
| task worker, tale | `SASE_BEAD_ID=sase-iq` only                               | `{"bead": "sase-iq"}`               |
| task worker, epic | `SASE_BEAD_ID=sase-iq` only                               | `{"parent_bead": "sase-iq"}`        |
| phase precedence  | `SASE_PHASE_BEAD_ID=sase-7z.5` + `SASE_BEAD_ID=sase-7z.5` | `{"bead": "sase-7z.5"}` (tale)      |
| epic precedence   | `SASE_EPIC_BEAD_ID=sase-7z` + `SASE_BEAD_ID=sase-7z`      | `{"parent_bead": "sase-7z"}` (epic) |
| no association    | nothing set                                               | `{}` (unchanged)                    |

The existing `id="outside-bead-work"` case must keep asserting `{}` with `SASE_BEAD_ID`
absent — that is the regression guard that an unassociated planner still archives clean.

`assert_archived_associations` already asserts the `BEAD` header section for `bead` and
the absence of a `PARENT` section, so the task-tale case gets `BEAD`-bullet coverage for
free.

Add a marker-fallback case mirroring `tests/test_plan_command_handler_metadata.py`:
write `agent_meta.json` containing `{"bead_id": "sase-iq"}` with `SASE_BEAD_ID`
**unset**, and assert the tale still archives `bead: sase-iq`. This is the re-exec path.

### 4. Documentation

- `docs/sdd.md`, the paragraph beginning _"When an epic is proposed from a bead-work
  phase agent, SASE also records that phase's ID as the managed `parent_bead` …"_. It
  currently ends with _"Agents outside bead work have no association and continue to
  create top-level epics."_ Correct that: an agent working a task bead records that
  task's ID, so an epic it proposes becomes a child of the task (`sase-iq` →
  `sase-iq.1`); only agents with no bead association at all still create top-level
  epics.
- `docs/beads.md`, the paragraph beginning _"When an epic-tier plan is proposed from
  bead work, `sase plan propose` automatically stamps `parent_bead` from the phase
  agent's `SASE_PHASE_BEAD_ID`, or from the land agent's `SASE_EPIC_BEAD_ID`."_ Add the
  task-worker source (`SASE_BEAD_ID`) and state that a task bead cannot then be closed
  until its child epic closes.
- Mention the tale side once in `docs/sdd.md` where the managed `bead` field is
  described, so both tiers' behavior is documented rather than only the epic one.

## Verification

```bash
just install
just check
```

Targeted:

```bash
.venv/bin/pytest tests/test_plan_command_handler_associations.py \
                 tests/test_plan_command_handler_metadata.py -q
```

End-to-end sanity, without launching anything:

1. `sase bead work <a-large-task-id> --dry-run` and confirm the rendered prompt contains
   both `%id(!<id>, bead=<id>)` and `#plan`.
2. Confirm `SASE_BEAD_ID` really is visible to `sase plan propose` inside a live agent
   (it is the same variable `sase commit` reads for its `SASE_BEAD=` footer — see
   `src/sase/main/commit_handler.py`); a `sase bead work` launch on a small throwaway
   task plus `env | grep SASE_BEAD_ID` in the agent shell is the cheapest confirmation.
3. For the epic path, confirm parenting resolves before any store mutation with
   `sase bead work <plan-file> --dry-run`, which previews the parented epic ID
   (`<task-id>.1`) without creating beads or launching agents.

Report `PROPOSED FOLLOW-UP:` items for the two out-of-scope surfaces (bead → plan
`design` write-back, and the ACE task-role PLAN lane) rather than expanding this change.
