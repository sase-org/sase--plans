---
tier: epic
title: Name-first `sase plan approve`/`reject` that can see every pending plan
goal: '`sase plan approve` and `sase plan reject` find every plan that is really awaiting
  review, accept the plan''s name in whatever form the user has on hand (`unrelated_red_gate_bead_close`,
  `202609/unrelated_red_gate_bead_close.md`, a path, a `plan:` ref, the planner agent,
  or the old notification ID), TAB-complete pending plan names with rich descriptions,
  and explain every miss precisely instead of printing "pending plan approval not
  found".

  '
phases:
- id: resolver
  title: Gate-owned visibility and the name-first selector resolver
  depends_on: []
  size: medium
  description: 'resolver: make pending-plan visibility gate-shell aware, add the lightweight
    plan-name module, replace the ID-only selector with the exact-then-prefix resolver
    and its miss diagnosis, and render approve/reject errors, ambiguity, and success
    beautifully.'
- id: completion
  title: TAB completion for pending plan names
  depends_on:
  - resolver
  size: medium
  description: 'completion: add the `pending_plan` value kind with a fast-path provider
    that mirrors the resolver''s visibility rule, bind it to the approve/reject PLAN
    slot, and keep candidates fresh with a short per-kind cache TTL in every shell.'
- id: surfaces
  title: Names everywhere plans are listed
  depends_on:
  - resolver
  size: small
  description: 'surfaces: lead `sase plan list` Proposed rows and `sase plan show`
    hints with the plan name, add a `name` JSON field, and finish the name-first docs.'
proposed_by: bbugyi200.athena.0qx
create_time: 2026-09-24 12:07:24
status: wip
bead_id: sase-17z
---

- **PROMPT:** [prompts/202609/plan_approve_names.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/plan_approve_names.md)
- **BEAD:** [sase-17z](https://github.com/sase-org/sase--beads/blob/main/pages/sase-17z/README.md)

# Plan: Name-first `sase plan approve`/`reject` that can see every pending plan

## Diagnosis

The reported failure has three separate causes. Only fixing all three makes the command
trustworthy.

1. **The CLI can't see shell-owned proposals at all.** This is the root cause. Since
   `32da1f3d2` (`feat(plan): add shell-backed approval handoff`, 2026-08-27), a planner
   hands its proposal to a gate shell and then writes `done.json` with status DONE.
   `visible_pending_plan_notifications()` (`src/sase/main/plan_candidates.py`) still
   counts a proposal as pending only when its notification matches a _live planner_ row
   (`_agent_is_live_plan_candidate`). So every modern proposal is invisible to
   `sase plan list`, `approve`, `reject`, and `show`.

   Reproduced on 2026-09-24: `sase gate list` showed pending gates `0qw--gate` (tale)
   and `0qv--gate` (epic). At the same moment, `sase plan list -s proposed` showed 0,
   and `sase plan show 35553907` printed `unknown plan`. Both plans had to be approved
   in the TUI. With the notification ID in hand, `sase plan approve 35553907` would have
   printed the same `pending plan approval not found` the user saw.

2. **The selector accepts only notification IDs.** `_matching_selector_notifications`
   (`src/sase/main/plan_pending.py`) compares against `notification.id` only. Every
   surface a user actually reads names plans by file name instead. That includes the
   notification text (`Tale ready for review: foo.md`), the gate-shell label, the
   archive path, and the plan list's Plan column.

3. **Misses explain nothing.** The user's own plan,
   `~/.sase/plans/202609/unrelated_red_gate_bead_close.md`, never had an approval gate.
   Agent `0qr` archived it, then crashed while creating the gate with
   `GateError: next.fork must be family, shell, or none`. That was a mid-rollout
   mismatch in the agent-session rename. It is already fixed on master
   (`notification_gates/model_shell.py` now reads `session`). So even after fixes 1 and
   2, that plan cannot be approved. The CLI must say exactly that, not "not found".

There is also no completion for the selector. The existing `plan` kind would offer about
2,600 archived `plan:` refs, which is the wrong set for an approval.

## Design

### The plan name

A proposal's **name** is its archive file stem. For
`~/.sase/plans/202609/unrelated_red_gate_bead_close.md`, the name is
`unrelated_red_gate_bead_close`.

- `move_plan_to_sase` already de-duplicates names across all month shards (`foo_1.md`,
  `foo_2.md`, …). So the bare name is unique in the local archive and is the simplest
  handle possible.
- Every user-facing string calls it the plan's **name**, never "slug" or "handle".
- The display form is the _shortest unique form_ within the set being shown: the bare
  name, then `<shard>/<name>` if two names ever collide (defensive), then the full path.
- A new lightweight module, `src/sase/plan_names.py`, owns the name helpers. It may
  import only the stdlib and `sase.core.paths`, because the completion fast path imports
  it. Helpers:
  - `plan_name(path)`
  - `plan_display_names(paths)` (shortest unique form)
  - selector normalization (strip whitespace, a `plan:` or legacy `plans:` prefix, a
    leading `./`, and a trailing `.md`)

### Accepted PLAN forms

`sase plan approve [PLAN]` and `sase plan reject [PLAN]` resolve PLAN identically:

| Form                           | Example                                                   |
| ------------------------------ | --------------------------------------------------------- |
| name (what TAB completes)      | `unrelated_red_gate_bead_close`                           |
| name with `.md`                | `unrelated_red_gate_bead_close.md`                        |
| `<shard>/<name>[.md]`          | `202609/unrelated_red_gate_bead_close.md`                 |
| `plan:` reference              | `plan:202609/unrelated_red_gate_bead_close.md`            |
| path (absolute, `~`, relative) | `~/.sase/plans/202609/unrelated_red_gate_bead_close.md`   |
| gate bundle's `plan.md` path   | `~/.sase/interaction_requests/plan/<id>/plan.md`          |
| planner agent                  | `0qw`, `@0qw`, `0qw--plan`, `0qw--gate`, `sase-17p.2`     |
| notification ID or prefix      | `35553907` (unchanged, for Telegram/mobile muscle memory) |

Matching rules. The first matching tier wins.

1. **Exact tier.** One of these:
   - a full notification ID;
   - a path that resolves (`expanduser`, cwd-relative, `resolve(strict=False)`) to the
     proposal's archive file or bundle `plan.md`;
   - a normalized selector equal to `<name>` or `<shard>/<name>`;
   - an agent spelling equal to the notification's `agent_name`, with `@` and a
     `--plan`/`--gate` role suffix tolerated.

   Name, ref, and agent comparisons are case-insensitive.

2. **Prefix tier.** Used only when the exact tier is empty: a notification-ID prefix, as
   today.
3. **One match wins.** Exactly one distinct proposal resolves. Two or more is an
   ambiguity error listing each candidate and _which form matched it_. The CLI never
   guesses.
4. **No name-prefix or fuzzy matching.** SASE's naming habits make it dangerous. Today's
   archive holds `agent_session_wire_cutover` next to
   `agent_session_wire_cutover_finish`. A user naming the first, already-approved plan
   must never approve the second. Close matches appear only as "did you mean"
   suggestions on a miss.
5. **Omitted PLAN** still requires exactly one pending proposal.

### What counts as pending (the visibility fix)

Visibility is decided by the **gate**, not by the planner process or the inbox.

- Candidates are plan-approval notifications (`PlanApproval`/`EpicApproval`) whose
  action state is still `available`.
- **Shell-backed gates** (all gates since the shell migration): look up the gate shell
  with the O(1) indexed
  `sase.gate_shell.store.find_gate_shell_by_gate_id(None, gate_id)`. `gate_id` is
  `action_data["request_id"]`, falling back to the bundle directory name. The proposal
  is pending exactly when that shell is not terminal. This holds _even if the
  notification was dismissed_, because dismissing an inbox row doesn't cancel the gate,
  and the Agents tab still shows the gate shell awaiting a decision.
- **Legacy gates** (no gate-shell record): keep today's non-dismissed, live-planner-row
  rule unchanged.

This one change repairs `plan list` Proposed, `approve`, `reject`, and `show`'s proposal
rung at once. They all read `visible_pending_plan_notifications()`, and nothing else
calls it.

### Misses explain themselves

When nothing pending matches, diagnose before printing:

1. Locate the plan the user named. Use the local archive by name, `<shard>/<name>`,
   path, or `plan:` ref, with the same machinery `sase plan show` uses. Import it lazily
   to avoid the `plan_show` ↔ `plan_pending` cycle.
2. If found, read its gate history from the pending-action store
   (`read_pending_action_store(include_legacy=True)`). Take entries whose
   `action_data.original_plan_file` is that file, newest `created_at_unix` first, and
   classify:
   - `already_handled` with a `handled_action`: say "was already approved as a tale",
     "approved as an epic", "approved and committed", "rejected", "sent back for
     revision", or "cancelled", plus a relative age (`38m ago`).
   - stale or past `stale_deadline_unix`: say the approval request expired (requests go
     stale after 24h).
   - available but not visible: say its approval gate is orphaned, meaning no live gate
     shell or planner owns it.
   - no entry at all: say no approval gate was ever opened for it. This is the user's
     `unrelated_red_gate_bead_close` case.
3. Otherwise: "no pending plan matches `X`", plus up to three `difflib` suggestions
   drawn from pending names.

Every error ends with the **Awaiting approval** list, or `Nothing is awaiting approval.`
Keep error codes stable so exit statuses don't change:

- 2 for selection errors;
- `conflict_already_handled` for handled plans;
- `gone_stale` for expired plans;
- `not_found` otherwise.

### Beautiful output

All selector output goes through a new shared renderer,
`src/sase/main/plan_pending_render.py`, printing to stderr for errors. Color follows the
shared contract (`sase.core.term_color.should_colorize`), so `NO_COLOR` and
`FORCE_COLOR` behave. Styles:

- name: bold cyan;
- tier: green for tale, magenta for epic, the same palette as `plan_inventory_render`;
- title: default;
- agent and age: dim;
- `✗` red, `✓` green.

```text
$ sase plan approve -k tale 202609/unrelated_red_gate_bead_close.md
✗ unrelated_red_gate_bead_close is not awaiting approval
  Stop unrelated red gates from leaving assigned beads open · tale
  ~/.sase/plans/202609/unrelated_red_gate_bead_close.md
  No approval gate was ever opened for this plan, so there is nothing to approve.
  Inspect it with: sase plan show unrelated_red_gate_bead_close

Awaiting approval (2)
  updates_tab_cached_open           tale  Cache the updates tab open state            @0qw   4m
  bead_relocation_safe_epic_launch  epic  Relocation-safe bead IDs and epic launches  @0qv  10m
```

On an ambiguity, the list shows a "matched by" note per row: `agent`, `ID prefix`, or
`name`. On success, the output leads with the name:

```text
✓ Approved as tale · updates_tab_cached_open
  Cache the updates tab open state
  35553907 → ~/.sase/interaction_requests/plan/979f0756-…/response.json
```

The existing epic monitor/proc follow hints stay below it. `reject` mirrors this format,
including its cleanup warning and error lines.

### Completion

- Add a new value kind, `pending_plan`. Candidates are the display names of pending
  proposals, newest first. Each description reads `tier · title · @agent · age`:

  ```text
  $ sase plan approve <TAB>
  updates_tab_cached_open           -- tale · Cache the updates tab open state · @0qw · 4m
  bead_relocation_safe_epic_launch  -- epic · Relocation-safe bead IDs and epic launches · @0qv · 10m
  ```

- It is bound to the approve/reject PLAN positional through `PATH_OVERRIDES`, the same
  mechanism as `(("plan", "show"), "target")`.
- The provider obeys the fast-path import contract: no `sase.notifications`,
  `sase.gate_shell`, `sase.sdd`, `sase.ace`, or `rich`. Those package imports pull
  `rich`/`sase.ace`. Reading this state that way was measured on this host at about 75
  ms of import plus about 150 ms per snapshot read.
  - Read notifications through the Rust binding directly (`read_notifications_snapshot`,
    including dismissed rows).
  - Look gate shells up through
    `sase.core.agent_scan_facade.find_gate_shell_by_gate_id`.
  - Derive gate state from the wire record (`agent_meta.agent_session_shell.state`,
    defaulting to `pending`).
  - For the title, lightly scan the plan's frontmatter. Take the tier from
    `action_data.plan_tier`.
- **Freshness is a feature.** A user approving plans back-to-back must not be offered
  the plan they just approved, or miss the one that just arrived. One constant in
  `sase.completion.kinds`, `VOLATILE_KIND_TTL_SECONDS = {PENDING_PLAN: 5}`, drives the
  disk-cache TTL and the zsh and bash in-shell cache TTLs. The emitters read it from
  Python, so the three layers can't drift.

### Boundary

This is CLI selector grammar and presentation. It follows the `plan_show/resolve.py`
precedent: Python sequences existing bindings, while `plan:` reference parsing and
resolution stay Rust-owned through `sase.sdd.plan_refs`. No new shared backend semantics
cross the `sase-core` boundary, and no `sase-core` change is needed.

### Non-goals

- **Approving plans that have no gate.** The orphaned `0qr` proposal is one example.
  Approval answers a live gate, and fabricating one is a separate feature. The miss
  message points at `sase plan show <name>`.
- **Accepting the consumed `sase_plan_<name>.md` scratch filename.** The archive may
  have renamed it to `<name>_1.md`, so accepting it could select the wrong plan.
- **Changing notification IDs or the Telegram/mobile action protocol.**

## Phase: resolver

This phase covers visibility, the name-first resolver, and approve/reject UX.

- Add `src/sase/plan_names.py` (see "The plan name"). Add a unit test that imports it in
  a subprocess and asserts that no `rich`, `sase.ace`, `sase.notifications`, or
  `sase.sdd` module was loaded.
- In `src/sase/main/plan_candidates.py`, implement the gate-owned visibility rule. For
  shell-backed gates, include dismissed notifications. Keep the `agents=` parameter's
  legacy semantics.
- Rework `src/sase/main/plan_pending.py` around a structured, never-raising resolver:
  - `PendingPlan` holds the notification, name, display name, archive path, bundle plan
    path, title, tier, agent, and age.
  - `pending_plans()` returns proposals newest first.
  - `resolve_pending_plan_selector(raw) -> PendingPlanMatch | PendingPlanAmbiguity | PendingPlanMiss`
    carries the diagnosis. Put the diagnosis in its own module if `plan_pending.py`
    would exceed the toobig limit.
  - Keep `resolve_pending_plan(selector)` as the raising wrapper, using the stable
    codes, for `plan_show`.
- Build the plan-path and title lookups on the existing
  `_plan_approval_artifacts.durable_plan_file_for_context` and
  `plan_inventory_paths.plan_metadata_for_path`. Do not re-derive them.
- Switch `plan_approve_handler.py` and `plan_reject_handler.py` to the structured API
  and the shared renderer in `src/sase/main/plan_pending_render.py`. Keep
  `_validate_wait_spec_for_cli` running before resolution.
- In `src/sase/main/parser_plan.py`, for approve and reject:
  - Use positional metavar `PLAN`. Keep `dest="selector"` so callers and tests keep
    working.
  - Rewrite the help to "Pending plan: name (TAB completes), <shard>/<name>[.md], path,
    plan: ref, planner agent, or notification ID/prefix".
  - Make the descriptions state the exact-then-prefix rule and the omitted-PLAN rule.
  - Use name-first epilog examples, 5–7 lines, covering a bare name,
    `202609/<name>.md --kind tale`, a `~/.sase/plans/...` path, an agent, an ID prefix,
    and `--wait`.
  - Keep options sorted. Every option already has a short alias.
- In `src/sase/plan_show/resolve.py`:
  - The `proposal` rung resolves bare words through the structured resolver:
    - match → `_Resolved`;
    - ambiguity → `_Ambiguous` built from its candidates;
    - miss → fall through.

    Path-shaped input still skips this rung.

  - `_finalize` reuses the matched notification instead of resolving twice.
  - `_final_miss` suggests pending names instead of ID prefixes.

- Tests:
  - Extend `tests/test_plan_approve_cli.py` and `tests/plan_show/test_resolve.py`. Add a
    focused selector matrix test module covering:
    - every form in the table, including the user's exact
      `202609/unrelated_red_gate_bead_close.md` and `unrelated_red_gate_bead_close.md`
      invocations;
    - exact beating prefix;
    - `foo` never selecting `foo_finish`;
    - agent/ID ambiguity with "matched by" output;
    - case-insensitivity.
  - Visibility cases:
    - a pending shell with a DONE planner is visible;
    - a pending shell with a dismissed notification is visible;
    - an answered or cancelled shell is invisible;
    - a legacy live planner is visible;
    - a legacy orphan is invisible.
  - Miss diagnosis cases: never-gated (the user's case), approved as tale, rejected,
    stale, orphaned, unknown with suggestions, and zero/multiple pending with an omitted
    PLAN.
  - Success rendering, and `NO_COLOR`/`FORCE_COLOR` behavior.
- Update the selector docs to be name-first:
  - `docs/cli.md`: the `sase plan approve`/`reject` paragraph and the table row;
  - `docs/configuration.md`: the `sase plan` table rows and the "Use the Proposed row's
    `id_prefix`" paragraph;
  - `docs/sdd.md`, `docs/xprompt.md`, and `docs/ace.md`: their `<id-prefix>` mentions.

## Phase: completion

This phase adds TAB completion for pending plan names.

- In `src/sase/completion/kinds.py`:
  - add `ValueKind.PENDING_PLAN = "pending_plan"`;
  - add `PATH_OVERRIDES` entries for `(("plan", "approve"), "selector")` and
    `(("plan", "reject"), "selector")`;
  - add `VOLATILE_KIND_TTL_SECONDS`.
- Add the provider `pending_plan_candidates` / `pending_plan_source_path` to
  `src/sase/completion/candidates/catalog_sdd.py`, or to a new `catalog_plans.py` if
  that is cleaner. The source path is `notifications.jsonl`. Register it in
  `catalog.py`. Make `providers.candidates_for` honor the per-kind TTL.
- Mirror the resolver's visibility rule exactly. Use `plan_names.plan_display_names` for
  values. If the terminal gate-state set can't be imported without heavy modules, pin a
  local copy to `sase.gate_shell.state.TERMINAL_GATE_STATES` with a parity test.
- In `emit_zsh_preamble.py` and `emit_bash.py`, apply per-kind in-shell TTLs generated
  from `VOLATILE_KIND_TTL_SECONDS`. Keep the default `SASE_COMPLETION_CACHE_TTL` for
  every other kind. Fish already relies on the disk cache. Keep the zsh
  value/description escaping intact.
- Tests:
  - add `pending_plan` to `_SHIPPED_KINDS` in
    `tests/main/test_completion_candidates_contract.py` (import-set and CPU probes);
  - provider tests in `tests/completion/test_candidates_providers.py`: newest-first
    order, descriptions, collision display, and exclusion of answered, stale, and
    cancelled gates;
  - a parity test: the same fixture state yields identical name sets from
    `sase completion candidates pending_plan` and phase 1's `pending_plans()`;
  - kind-resolution tests in `tests/completion/test_kinds.py`;
  - regenerate the `tests/completion/snapshots/cli_spec.json` snapshot and any emitter
    goldens.
- Update `docs/completion.md`: add `pending_plan` to the kinds list, plus a short worked
  example next to the memory-selector paragraph explaining the name-only candidates and
  the 5s freshness window.

## Phase: surfaces

This phase puts names everywhere plans are listed.

- `sase plan list` Proposed table (`src/sase/main/plan_inventory_render.py`):
  - Replace the leading `ID` column with `Name`: the display name in bold cyan, folded
    and never ellipsized so it stays copy-pasteable, with the dim 8-character ID prefix
    on a second line.
  - Under a non-empty Proposed section, add one dim hint line:
    `approve: sase plan approve <first name>  ·  reject: sase plan reject <name>  ·  TAB completes names`.
- Add a `name` field to `ProposedPlan` (`plan_inventory_models.py`), populated in
  `plan_inventory_collectors.py` via `plan_names`, and emit it in `--json` rows.
  `id_prefix` stays.
- `sase plan show`: `_hint` in `src/sase/main/plan_show_render.py` prints
  `sase plan approve <name>` / `sase plan reject <name>`. The proposal context carries
  `name`, and the JSON envelope includes it.
- Update tests covering Proposed table rendering, list JSON, and the show hint and JSON.
  Refresh any affected snapshots.
- Update docs: the `sase plan list` JSON field list and Proposed-row description in
  `docs/cli.md` and `docs/configuration.md`, pointing users at the name.

## Verification

- Each phase runs `sase tool run check` and fixes what it breaks.
- Once phases 1 and 2 have landed, check against a real pending proposal (for example,
  launch a throwaway `#plan` agent):
  - `sase plan list -s proposed` shows it;
  - `sase completion candidates pending_plan` prints its name with a description;
  - `sase plan approve <TAB>` offers it within 5s of it appearing, and stops offering it
    within 5s of approval;
  - `sase plan approve <name> -k tale` approves it.
- `sase plan approve -k tale 202609/unrelated_red_gate_bead_close.md` prints the
  never-gated diagnosis above and exits 2.
- Users with stamped completion installs pick up the new grammar through the runtime
  cache on upgrade. `sase completion refresh` forces it.
