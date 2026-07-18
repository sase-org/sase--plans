---
tier: tale
title: Bare --plan family roots and family-name references
goal: 'Creating an agent family renames the first agent into its unnumbered role slot
  (cx--plan, not cx--plan-0), the top-level family row keeps the original agent name,
  family names work as #fork/%wait arguments, and every existing family-creation flow
  keeps its user-visible behavior.

  '
create_time: 2026-07-18 08:24:31
status: done
prompt: 202607/prompts/family_plan_root_rename.md
---

# Plan: Bare `--plan` family roots and family-name references

## Context

sase-6n.4 (commit 01da41927) introduced rename-on-attach: when the first member attaches to a bare agent `cx`, the
original is persistently renamed into a member slot and the bare name becomes a family container. The implementation
reused the old display-slot spellings — `--plan-0` for plan proposers and `--0` for generic roots — and deleted the
display-only collapse that used to keep the top-level row labeled with the bare name. Two user-visible regressions
resulted:

1. **Wrong root suffix.** A plan proposer is now persisted as `cx--plan-0`. That name is overloaded: `--plan-N` is the
   namespace for plan feedback rounds and plan-phase question follow-ups (the `--plan-@` allocation template), and the
   suffix classifier reads `--plan-0` as a feedback row. The proposer must instead take the canonical unnumbered planner
   slot `cx--plan` (`PLAN_CHAIN_PLAN_SUFFIX`), which `_reserved_agent_family_names` already reserves and which the
   pre-6n synthetic planner row and `planner_row_name()` aliasing already treat as canonical.
2. **Root row lost the original name.** The Agents-tab top-level family entry now renders the renamed member name
   (`cx--plan-0`) instead of the original agent name (`cx`). Families get no synthetic container row (only clans do);
   the top row is the promoted member itself, and nothing in the render path collapses it back to the family name
   anymore.

Target model (unchanged where not listed): the family takes the name of the first agent (`cx` is the container, reserved
in the name registry with `container_kind: "family"`); the first agent is renamed `cx--plan` (plan proposers) or
`foo--0` (generic roots, unchanged); feedback and plan-phase question follow-ups allocate `cx--plan-0`, `cx--plan-1`, …
again (pre-6n numbering restored, since the proposer no longer occupies `--plan-0`); phase children (`cx--code`,
`cx--commit`, …) and root-question continuations (`--1`, `--2`, …) are unchanged.

No sase-core changes are needed: the Rust family-parent resolver matches on `workflow_name` (which stays the bare base)
and sase-core has no root-suffix naming logic (verified).

## Design

### 1. Core rename: plan roots get bare `--plan`

`src/sase/agent/_family_promotion.py` is the single source of the root slot:

- `_PLAN_ROOT_SUFFIX` becomes `PLAN_CHAIN_PLAN_SUFFIX` (`--plan`); `_GENERIC_ROOT_SUFFIX` stays `--0`. Update the
  hard-coded validation set in `promote_agent_to_family`, the plan-wins precedence between caller-supplied and derived
  suffixes, and the `plan_chain_root=True` branch to key on the new value. The idempotence branch (`current_name`
  already family-prefixed) already handles `cx--plan`.
- Callers pick the new value up without signature changes, but each must be verified: derivation in
  `resolve_family_attach_plan` (`_family_attach_resolution.py`), the in-batch sibling suffix in
  `_prompt_family_root_role_suffix` (`_family_attach_launch.py`), and the axe-side `promote_to_workflow` path
  (`run_agent_helpers_artifacts.py`) used by plan accept (`run_agent_exec_plan_accept.py`), plan feedback
  (`run_agent_exec_plan.py`), and question handling.
- Consequences to assert, not code: the `--plan-@` template now allocates `--plan-0` as the first feedback/question
  round; `%n(cx, plan)` now raises the existing "reserved for the original parent" error; explicit `%n(foo, 0)` on a
  generic root still refuses to double-promote.

### 2. TUI: the top-level family entry keeps the original agent name

Restore a model-layer collapse (the inverse successor of the removed `assign_bare_family_root_zero_suffix`): a non-child
row that anchors a plan-chain family — the promoted root member (`plan_chain_root` / `agent_family_role == "root"` with
a plan suffix) — presents the family name (`agent.agent_family`, falling back to `agent_family_base(agent.agent_name)`)
as its row identity: the agent-name annotation in `_agent_list_render_agent.py` and the detail-panel Name for that row.
Member rows keep their real member names, including the synthetic planner child from
`ensure_synthetic_planner_children`, which follows the parent's `role_suffix` and therefore becomes `cx--plan`
automatically. The clipboard `%n` copy action already collapses this way — reuse its rule rather than inventing a new
one.

- The collapse keys on family metadata, not the suffix spelling, so records promoted during the short-lived `--plan-0`
  scheme (live agents exist on this machine) render correctly with no data migration.
- Scoping decision (flagging for review): generic dynamic families keep the `--0` annotation on their root row — that
  was the deliberate display both before and after sase-6n.4; only the plan-chain root entry regressed. The family name
  still appears as the container/banner name.
- Respect the TUI perf rules: compute the collapse at enrichment/status-apply time (no render-path I/O); if any newly
  visible input feeds rendering, extend `agent_render_key`/`_runtime_signature` so patch-row updates stay correct.
  Add/refresh a PNG snapshot for the expanded family tree (root row `cx`, children `cx--plan` and `cx--code`).

### 3. `#fork` accepts family names

`#fork` resolves names through `sase/scripts/agent_chat_from_name.py` → `find_named_agent`, which today only happens to
match a family base via `workflow_name` and picks the newest record. Make family-name resolution explicit: when the
requested name is a family container (family metadata lookup succeeds and no exact agent has that name), resolve to the
most recent completed member via the existing family-aware helpers (`resolve_resume_agent_name` /
`most_recent_completed_family_member` in `agent/names/_lookup.py`). Exact member names, `cx--plan` included, and
template references behave as today. The `#fork`-implied `%wait` (fork targets are appended to `wait_names` in
`run_agent_directives.py`) keeps whole-family wait semantics.

### 4. `%wait` guarantees

`%wait(cx)` already resolves as wait-on-family (`family_candidate` in `core/wait_dependency_resolution/_index.py`: every
member of the generation must complete). Lock this in with regression tests against renamed `--plan` roots, covering the
recursive descendant-chain generation walk and still-queued members. For `%wait(cx--plan)`: the renamed proposer and the
submitted-plan planner-row alias (`planner_row_name`) now share one name — assert that a wait on the planner row still
unblocks while the plan is under review (submitted-plan candidate) and resolves via the exact member after approval;
guard against the not-yet-done proposer record shadowing the review-window candidate in the newest-timestamp merge.

### 5. Flow preservation matrix

Each flow keeps its user-visible behavior; each gets a regression test asserting the resulting names:

| Flow                                                      | Expected names after this change                                          |
| --------------------------------------------------------- | ------------------------------------------------------------------------- |
| Plan approved → coder                                     | proposer renamed `cx--plan`; coder `cx--code`                             |
| Plan feedback rounds                                      | `cx--plan-0`, then `cx--plan-1`, … (allocator and no-name fallback paths) |
| Question answered (root agent)                            | root promoted `foo--0`; follow-ups `foo--1`, `foo--2`, …                  |
| Question answered (plan phase)                            | follow-up `cx--plan-0`                                                    |
| User `%n(parent, suffix)` / `%n(parent, @)`               | child `foo--<suffix>`; generic root promoted `foo--0`                     |
| In-batch siblings (multi-prompt `pending_family_parents`) | plan-auto siblings' root slot `--plan`; generic `--0`                     |
| Families inside clans                                     | clan metadata propagation unchanged                                       |

## Testing

- Update sase-6n.4 expectations: `tests/test_dynamic_agent_family_attach_metadata.py`, `..._attach_resolution.py`,
  `..._attach_inbatch_launch.py`, `tests/test_dynamic_agent_family_root_zero_suffix.py`,
  `tests/test_axe_chop_wait_checks_plan_families.py`, `tests/test_agent_name_registry.py`,
  `tests/test_agent_names_lookup.py`, `tests/test_axe_run_agent_helpers_artifacts.py`, plus a repo-wide sweep for
  `--plan-0` expectations in tests.
- New regressions: `#fork` with a family name (latest completed member) and with `cx--plan`; `%wait` family/planner-row
  cases from section 4; first-feedback and plan-phase-question allocation landing on `--plan-0`; promotion idempotence
  and re-promotion with `--plan`; TUI root-entry label collapse (model-level test + PNG snapshot).
- `just check` and the visual snapshot suite must pass.

## Risks and compatibility

- **Legacy `--plan-0` roots (no migration).** Records promoted while the `--plan-0` scheme was live keep working:
  grouping, waits, and the TUI collapse key on `agent_family`/`agent_family_role`/ `plan_chain_root` metadata rather
  than the suffix. Verify the classifier-driven surfaces (feedback-round badges, members panel) don't mislabel such a
  root as a feedback row where the stored role is available.
- **Name sharing on `cx--plan`.** The wait-index merge of the proposer record and the submitted-plan alias is the main
  behavioral seam — covered by dedicated tests in section 4.
- **Docs.** Update the promotion suffix in `docs/agent_families.md`, `docs/ace.md`, and `docs/xprompt.md` (each
  currently documents `--plan-0` promotion). `sase/memory/glossary.md` needs no change (its wording is suffix-agnostic),
  and memory files are not to be edited without explicit user permission.
