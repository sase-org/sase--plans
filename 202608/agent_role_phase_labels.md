---
tier: tale
title: Render agent-family phase headers as AGENT (<role>)
goal:
  Every agent-family phase header in the ACE prompt panel reads `AGENT (<role>)`, so
  plan, coder, question, epic, commit, and monitor members use the same shape custom
  family members already use.
size: small
proposed_by: bbugyi200.athena.05z
create_time: 2026-08-18 08:44:35
status: wip
---

# Plan: Render agent-family phase headers as `AGENT (<role>)`

## Goal

The AGENT REPLY section of the ACE prompt panel splits a family's reply into phases and
labels each one with a purple divider. Six family roles currently get bespoke, all-caps
headers (`PLANNER`, `CODER`, `QUESTIONS`, `EPIC`, `COMMIT`, `MONITOR`) while every other
family member already renders as `AGENT (<token>)` — for example `AGENT (bar)` for a
`--bar` member and `AGENT (0)` for a promoted bare root.

Replace the bespoke headers with the uniform `AGENT (<role>)` shape so the panel has one
naming rule instead of two.

## Current Behavior

`get_phase_label()` in `src/sase/ace/tui/widgets/prompt_panel/_agent_display_content.py`
is the single source of truth. It resolves a member's family role from
`agent.role_suffix` / `agent.agent_family_role` via `sase.plan_chain`, then returns:

| Family role (canonical suffix)                                     | Label today                |
| ------------------------------------------------------------------ | -------------------------- |
| `q` — `--q`, root-question `--0`/`--1`/`--N` (not a promoted root) | `QUESTIONS`                |
| `code` — `--code`, `--code-N`                                      | `CODER`                    |
| `epic` — `--epic`                                                  | `EPIC`                     |
| `commit` — `--commit`                                              | `COMMIT`                   |
| `monitor` — `--mon`, `--mon-N`                                     | `MONITOR`                  |
| `plan` — `--plan`                                                  | `PLANNER`                  |
| `feedback` — `--plan-N`, legacy `--N`/`-N`/`.N`                    | `PLANNER (round K)`        |
| custom member — `--bar`, promoted bare root `--0`                  | `AGENT (bar)`, `AGENT (0)` |
| no suffix                                                          | `AGENT`                    |

Legacy dotted (`.plan`) and single-dash (`-plan`) suffixes canonicalize to the same
labels.

Five call sites consume that label; all of them are display-only:

- `_agent_display_render.py` — AGENT REPLY phase dividers (the panel in the request).
- `_agent_display_hint_render.py` — the same dividers in hint mode.
- `_agent_display_family_render.py` — family-conversation phase dividers.
- `_agent_display_family.py` — `kind` for FAMILY MEMBERS roster rows.
- `_agent_display_neighbors.py` — `kind` for lane-neighbor roster rows.

The two roster call sites both do
`kind = "agent" if phase_label == "AGENT" else phase_label`, so a roster row reads
`--plan · PLANNER · ✓ DONE` today and will read `--plan · AGENT (plan) · ✓ DONE` after
this change — which is exactly the shape custom members already render
(`--bar · AGENT (bar) · ✓ DONE`). That is the intended outcome, not a side effect; no
separate change to the roster adapters is needed.

## Decisions

**Use the resolved family role, not the raw suffix token.** `--code-0` (a coder
question-continuation) has suffix token `code-0` but role `code`; it must keep rendering
as one coder phase, so the label comes from the role. The existing token fallback stays
for members with no known role, which is what keeps `AGENT (bar)` and `AGENT (0)`
working unchanged.

**`monitor`, not `mon`.** The monitor suffix is `--mon` but its family role is
`monitor`. Roles supply the label, so the header is `AGENT (monitor)`.

**Feedback rounds become `AGENT (plan round K)`.** `PLANNER (round 2)` today; keeping
the round number matters because consecutive feedback phases are otherwise
indistinguishable. Putting the qualifier inside the parentheses keeps the `AGENT (...)`
shape intact. (`AGENT (feedback K)` was considered and rejected: the visible round
number is already plan-relative — round 2 is the second planner pass — and
`plan round K` preserves that reading.)

**No feature flag.** This is a complete rename that is correct the moment it lands: no
beta to prove out, no partially landed path, and no old branch users must be able to
reach. Per `sase/memory/sase_flags.md` that is not flag-shaped work.

**No Rust core change.** `crates/sase_core` in the linked `sase-core` repo has no
phase-label or role-display logic (`agent_family.rs` only resolves family parents over
the wire). Per `sase/memory/rust_core_backend_boundary.md`, a role-to-display-string map
consumed only by Textual widget rendering is presentation and stays in this repo.

## Target Behavior

| Family role (canonical suffix)               | Label today                | Label after            |
| -------------------------------------------- | -------------------------- | ---------------------- |
| `q` — `--q`, root-question `--0`/`--1`/`--N` | `QUESTIONS`                | `AGENT (q)`            |
| `code` — `--code`, `--code-N`                | `CODER`                    | `AGENT (code)`         |
| `epic` — `--epic`                            | `EPIC`                     | `AGENT (epic)`         |
| `commit` — `--commit`                        | `COMMIT`                   | `AGENT (commit)`       |
| `monitor` — `--mon`, `--mon-N`               | `MONITOR`                  | `AGENT (monitor)`      |
| `plan` — `--plan`                            | `PLANNER`                  | `AGENT (plan)`         |
| `feedback` — `--plan-N`, legacy `--N`        | `PLANNER (round 2)`        | `AGENT (plan round 2)` |
| custom member — `--bar`, promoted root `--0` | `AGENT (bar)`, `AGENT (0)` | unchanged              |
| no suffix                                    | `AGENT`                    | unchanged              |

Legacy dotted and single-dash spellings keep mapping onto the same labels. The bare
`AGENT` return value is unchanged, so the roster `"agent"` fallback keeps working.

## Implementation

### 1. `src/sase/ace/tui/widgets/prompt_panel/_agent_display_content.py`

Add one formatter and drive every branch of `get_phase_label()` through it. Keep the
existing branch order and the `is_promoted_root` guard exactly as they are — only the
returned strings change — so no edge case shifts.

- Add a module-private helper that wraps a token:

  ```python
  def _agent_role_label(token: str | None) -> str:
      """Format one family member's phase header as ``AGENT (<role>)``."""
      return f"AGENT ({token})" if token else "AGENT"
  ```

- Replace the `_PHASE_LABELS` suffix-to-header dict with a suffix-to-role-token dict
  named `_PHASE_SUFFIX_TOKENS`, mapping `PLAN_CHAIN_PLAN_SUFFIX → "plan"`,
  `PLAN_CHAIN_CODER_SUFFIX → "code"`, `PLAN_CHAIN_QUESTION_SUFFIX → "q"`,
  `PLAN_CHAIN_EPIC_SUFFIX → "epic"`, `PLAN_CHAIN_COMMIT_SUFFIX → "commit"`. This branch
  is only reachable for a `--q` member whose stored `agent_family_role` is `root` with
  `plan_chain_root` false; keep it so that case keeps resolving to `AGENT (q)`.
- Return `_agent_role_label("q")`, `_agent_role_label("code")`,
  `_agent_role_label("epic")`, `_agent_role_label("commit")`,
  `_agent_role_label("monitor")`, and `_agent_role_label("plan")` from the six role
  branches.
- Return `_agent_role_label(f"plan round {feedback_round}")` from the feedback branch.
- Collapse the trailing token branch and the bare fallback into
  `return _agent_role_label(agent_family_suffix_token(agent.role_suffix))`.

Update the `get_phase_label` docstring to say it maps a member's family role to its
`AGENT (<role>)` phase header.

### 2. `src/sase/ace/tui/widgets/prompt_panel/_agent_display_parts.py`

This compatibility-export module re-exports `_PHASE_LABELS`. Rename that import and its
`__all__` entry to `_PHASE_SUFFIX_TOKENS`, keeping `__all__` sorted the way it is now.

### 3. No other source changes

The remaining `PLANNER`/`CODER` matches in `src/` are internal identifiers, not user
text — `APPROVED_PLANNER_ACTIONS` / `PLANNER_FAMILY_ROLES` in
`ace/tui/models/_agent_status_family_policy.py`, `_PLANNER_PHASE_ENDED_STATUSES` in
`ace/tui/models/agent_time.py`, and re-exports in
`ace/tui/models/_agent_status_family.py`. Leave them alone.

## Explicit Non-Goals

- `_SUFFIX_LABELS` / `_FAMILY_ROLE_LABELS` in
  `src/sase/ace/tui/agent_context_members.py` produce lowercase compact attribution
  tokens (`plan`, `coder`, `q`, `fb`) for context lanes, not phase headers. They are
  already lowercase and are not the `PLANNER`/`CODER` headers in the request. Do not
  touch them here, even though `coder` is inconsistent with the `code` role name.
- The `MONITOR` heading built by `_agent_monitor_section.py` is a prompt-panel _section_
  heading (alongside `AGENT PROMPT`, `AGENT REPLY`) that lists the monitored command and
  state. It is not a family-member phase header. Leave it as `MONITOR`.
- The `⚡ PLAN` badge in `_agent_display_header_metadata.py` renders a plan _action_,
  not a family role. Leave it alone.
- Renaming the internal `PLANNER_*` Python identifiers listed above.

## Tests

Update the assertions that pin the old header strings, and add coverage for the two
labels that had none:

- `tests/ace/tui/widgets/test_agent_display_phase_labels.py` — retarget every
  `TestGetPhaseLabel` case to the new strings (`AGENT (plan)`, `AGENT (code)`,
  `AGENT (q)`, `AGENT (epic)`, `AGENT (commit)`, `AGENT (plan round 2)`,
  `AGENT (plan round 10)`). The custom-member, promoted-root, `--code-0`, no-suffix, and
  unknown-suffix cases already assert `AGENT (...)` shapes and must keep passing
  unchanged — that is the regression guard proving the two naming rules merged into one.
  Add a case for the `monitor` role (`role_suffix="--mon"` and `role_suffix="--mon-1"` →
  `AGENT (monitor)`) and for the promoted-root `--q` fallback (`role_suffix="--q"`,
  `agent_family_role="root"`, `plan_chain_root=False` → `AGENT (q)`), since neither is
  covered today.
- `tests/ace/tui/widgets/test_agent_display_family_roster.py:43` — the roster `kind`
  assertion becomes `["AGENT (plan)", "AGENT (code)"]`.
- `tests/ace/tui/widgets/test_agent_display_xprompt.py` — two
  `assert "QUESTIONS" not in plain` checks guard against custom members being mislabeled
  as question members. Restate them as `assert "AGENT (q)" not in plain` so they still
  test that.
- `tests/ace/tui/widgets/test_agent_display_phase_divider.py` and
  `tests/test_timezone_runtime_consistency.py:234` pass a label string straight into
  `render_phase_divider`, so they pass either way. Update the literals to `AGENT (plan)`
  / `AGENT (code)` anyway so no stale header text survives in the suite.

Every other `PLANNER`/`CODER`/`QUESTIONS` hit under `tests/` is a `PLAN_CHAIN_*_SUFFIX`
constant import; leave those alone.

## Docs

`docs/ace.md` (in the AGENT REPLY bullet, near line 3826) spells out the old mapping:
`--plan` as `PLANNER`, `--code` as `CODER`, `--q` as `QUESTIONS`, `--epic` as `EPIC`,
`--commit` as `COMMIT`, and `--2` as `PLANNER (round 2)`. Rewrite it to describe the
single `AGENT (<role>)` rule, list `--mon` as `AGENT (monitor)` (missing today), state
that custom members render as `AGENT (<suffix token>)`, and keep the existing sentence
that legacy dotted and single-dash suffixes render the same way. Keep the file's prose
wrapping style.

No `sase/memory/*.md`, `AGENTS.md`, or generated provider shim mentions these headers,
so no memory edit is needed — and none may be made without separate explicit user
permission.

## Visual Snapshots

Family and lane-neighbor PNG goldens under `tests/ace/tui/visual/snapshots/png/` render
roster `kind` cells and phase dividers, so several will legitimately change (the
`agents_family_*`, `agents_clan_*`, and `agent_neighbor_*` families are the likely
movers). After the code and test changes:

```bash
just test-visual                                  # see which goldens moved
just test-visual -- --sase-update-visual-snapshots  # accept the intended changes
```

Then rerun `just test-visual` clean and inspect at least one regenerated golden (or its
`.pytest_cache/sase-visual/` diff artifact) to confirm the only delta is the header text
— a wider `AGENT (plan)` cell must not have pushed other columns off-screen or truncated
a roster row. If any golden changed for an unrelated reason, stop and investigate
instead of accepting it.

## Verification

1. `just install` first — workspaces are ephemeral and dependencies may be stale.
2. `just check` while iterating.
3. `just test-visual` plus the golden refresh described above.
4. `just check-full` before landing, run through `/sase_monitor` with a `--next` action
   (it routinely outruns a single agent turn); this change touches shared display
   helpers, so the full suite is required rather than the scoped lane.

## Done When

- `get_phase_label()` returns `AGENT (<role>)` for every known family role and keeps
  returning `AGENT (<token>)` / `AGENT` for everything else.
- AGENT REPLY phase dividers, family-conversation dividers, and both roster `kind`
  columns show the new headers.
- `docs/ace.md` documents the single rule.
- PNG goldens are regenerated and reviewed; `just check-full` passes.
