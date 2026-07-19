---
tier: tale
title: Tier-aware pending plan review statuses (EPIC / TALE)
goal: 'Agents awaiting plan review on the Agents tab show TALE for tale plans and
  EPIC for epic plans (each with its own distinct color) instead of the flat PLAN
  label, with pending-plan semantics consolidated behind shared helpers so every consumer
  treats the three labels uniformly.

  '
create_time: 2026-07-19 08:33:50
status: wip
prompt: 202607/prompts/epic_tale_pending_statuses.md
---

# Plan: Tier-aware pending plan review statuses (EPIC / TALE)

## Context

When an agent proposes a plan and blocks on user review, the TUI shows the status `PLAN` regardless of the plan's
authored tier. The rest of the status taxonomy is already tier-aware (`TALE APPROVED`, `EPIC APPROVED`, `TALE DONE`,
`EPIC CREATED`, `WORKING TALE`, ...), so the flat pending label is the odd one out: the user cannot tell from the Agents
tab whether they are being asked to review a small tale or a multi-phase epic — the review with the most leverage.

Every gated plan is authored as tier `tale` or `epic` (`sase/sdd/plan_tiers.py` defines
`PLAN_TIERS = ("tale", "epic")`), and gate creation (`sase/plan_gate.py::create_plan_approval_gate`) rejects plans whose
frontmatter does not declare a valid tier. So at pending-review time the tier is always knowable, and the flat `PLAN`
label carries strictly less information than the system already has.

### New semantics

- A pending plan review displays **`TALE`** when the submitted plan's authored tier is `tale`, **`EPIC`** when it is
  `epic`.
- Bare **`PLAN`** remains only as the unknown-tier fallback (legacy rows whose plan file is gone/unreadable). It keeps
  its current color and all current behaviors, so old artifacts degrade gracefully — no data migration exists or is
  needed because statuses are never persisted; they are derived at load time from `agent_meta.json` signals,
  notifications, and family structure.
- All three labels form one semantic family, "pending plan review": same Stopped bucket, same
  approve/dismiss/notification behaviors, same footer keybindings — only label and color vary by tier.
- Approved/working/terminal statuses are untouched. Note the existing nuance that `PLAN APPROVED` / `WORKING PLAN` /
  `PLAN DONE` vs their `TALE` variants key off the _approval action_ (plain approve vs approve+commit), not the authored
  tier; this plan does not change that.

## Design

### 1. Shared pending-status vocabulary (consolidation)

`src/sase/agent/status_buckets.py` is already the canonical status-semantics module shared by the TUI and integrations.
Add the new vocabulary there:

```python
PENDING_PLAN_STATUS = "PLAN"    # unknown-tier fallback
PENDING_TALE_STATUS = "TALE"
PENDING_EPIC_STATUS = "EPIC"
# Display-priority order (largest review ask first).
PENDING_PLAN_REVIEW_STATUSES: tuple[str, ...] = (
    PENDING_EPIC_STATUS, PENDING_TALE_STATUS, PENDING_PLAN_STATUS,
)
PENDING_PLAN_REVIEW_STATUS_SET: frozenset[str]

def pending_plan_status_for_tier(tier: str | None) -> str: ...
def is_pending_plan_review_status(status: str | None) -> bool: ...
```

Fold the set into the existing semantics in the same module so most behavior follows automatically:

- `AGENT_ASKING_STATUSES` — `agent_is_asking()` gains TALE/EPIC, which fixes summary "stopped" counts, mobile
  needs-action, and every caller of the predicate.
- `_STOPPED_STATUSES` — Stopped bucket membership, which automatically covers status bucketing/sorting
  (`agent_groups/_buckets.py`), the `attention:` query filter, clan member counts, stopped-navigation, and
  `is_stopped_agent_status`.
- `_NEEDS_INPUT_STATUSES` deliberately excludes `PLAN` today; keep excluding TALE/EPIC so `needs:input` semantics are
  unchanged.
- `DISMISSABLE_STATUSES`, `_TERMINAL_STATUSES`, `ACTIVE_AGENT_STATUSES` do not contain `PLAN` and must not gain the new
  labels (pending reviews are not dismissable, terminal, or in-flight).

Consolidation rule for call sites: wherever code hardcodes `status in ("PLAN", "QUESTION")` with "the agent paused for
input" intent, switch it to `agent_is_asking()`; wherever a membership set lists `"PLAN"`, splice in
`*PENDING_PLAN_REVIEW_STATUSES` instead of adding three literals. This leaves exactly one place that defines what
"pending plan review" means.

### 2. Tier resolution

The tier of a _pending_ plan cannot come from `plan_action` (only written at approval). Resolve it from the submitted
plan file's frontmatter, which the gate already validated:

- Add a small cached helper to `src/sase/sdd/plan_tiers.py`, e.g.
  `cached_plan_tier(path: str | Path | None) -> str | None`: a bounded LRU keyed by resolved path and invalidated by
  `(st_mtime_ns, st_size)`, with a short negative-result TTL — the same invalidation strategy as the existing
  `PlanFileCache` in `ace/tui/models/_agent_associated_plan_cache.py`. It returns a normalized tier or `None`
  (missing/unreadable/invalid file). Placing it in `sase/sdd/` keeps it importable by the TUI loader, family status
  logic, and integrations without TUI imports.
- Per the TUI performance rules: this helper must only be called from loader / status-derivation code (background
  refresh), never from per-keystroke render paths, and the cache means steady-state refreshes do exactly one `stat()`
  per pending-plan row (a handful of rows at most).
- Add a TUI-side convenience in `ace/tui/models/_agent_status_family.py`, e.g.
  `pending_plan_status_for_agent(agent) -> str`, which resolves the agent's freshest plan reference (`plan_path`,
  falling back to `sdd_plan_path` / `archived_plan_path`) through `cached_plan_tier` and maps it with
  `pending_plan_status_for_tier`. All family/override derivation sites use this one helper.
- The notification path knows the tier without any file read: gate kind `plan` ⇔ authored tale, `epic_plan` ⇔ authored
  epic, surfaced as notification action `PlanApproval` vs `EpicApproval`. Map `EpicApproval → EPIC` and
  `PlanApproval → TALE` for non-legacy bundles; for legacy bundles fall back to `pending_plan_status_for_agent` (which
  itself falls back to bare `PLAN`).

Considered and rejected: persisting an explicit `plan_tier` field in `agent_meta.json`. That would require extending the
Rust scan wire in the sibling `sase-core` repo plus a fallback path for pre-existing agents anyway. Deriving from the
(gate-validated) plan file gives one code path for old and new agents with no schema evolution.

### 3. Producers — where the pending label is emitted

Update every site that emits the flat `"PLAN"` pending status:

1. `ace/tui/models/_loaders/_meta_enrichment_common.py::plan_enrichment_status` — add a `plan_tier: str | None`
   parameter; the `plan_submitted and not auto_approved` branch returns `pending_plan_status_for_tier(plan_tier)`. Its
   two callers pass `cached_plan_tier(...)` of the plan path they already carry: `_meta_enrichment_filesystem.py` (meta
   dict `plan_path`) and `_meta_enrichment_wire.py` (`meta.plan_path`).
2. `ace/tui/actions/agents/_notification_status_overrides.py` — the `PlanApproval`/`EpicApproval` override currently
   writes `"PLAN"`; compute the tiered label per §2 first, then keep the existing no-churn guard by comparing against
   that computed label. (The same file's external-response auto-dismiss path already calls `plan_enrichment_status` for
   _approved_ states and needs no tier plumbing.)
3. `ace/tui/models/_agent_status_family.py::planner_child_status` — the `is_awaiting_plan_review(parent)` branch returns
   `pending_plan_status_for_agent(parent)`.
4. `ace/tui/models/_agent_status_apply.py` — the feedback-round awaiting-review branch and the DONE→PLAN catch-all
   (`has_unreviewed_submitted_plan`) both use `pending_plan_status_for_agent(agent)`; the workflow-step demotion guard
   `parent.status == "PLAN"` → `is_pending_plan_review_status(...)`.
5. `integrations/_agent_list_entry_builder.py::_plan_status` — return
   `pending_plan_status_for_tier(cached_plan_tier(meta.plan_path))` (mobile summary title-casing then shows
   "Tale"/"Epic" naturally).

### 4. Consumers — full edge-case inventory

Membership sets and checks that list the pending `"PLAN"` status; each gets the shared set/predicate from §1:

- Input-pause checks (`("PLAN", "QUESTION")` → `agent_is_asking()`): `integrations/_agent_list_entry_models.py`
  (`needs_user_action`), `ace/tui/actions/agent_workflow/_leader_mode.py`,
  `ace/tui/actions/agents/_display_detail_footer.py`, `ace/tui/actions/agents/_notification_modal_flow.py`,
  `ace/tui/modals/agent_neighbor_modal.py` (also its style branch).
- Approve eligibility: `ace/tui/actions/agents/_approve.py` (`_APPROVE_ELIGIBLE`),
  `ace/tui/widgets/_keybinding_bindings.py` (inline copy), `ace/tui/commands/availability.py` (`accept_proposal`),
  `ace/tui/actions/proposal_rebase.py` (inline copy). Consider deduplicating these four copies into one shared constant
  while touching them.
- Active-status sets: `ace/tui/modals/zoom_panel_rendering.py` (`ACTIVE_STATUSES`), `ace/tui/widgets/agent_detail.py`
  (`_ACTIVE_STATUSES`), `ace/tui/models/_loaders/_workflow_loaders.py` (`ACTIVE_STATUSES`).
- Live refresh: `ace/tui/actions/event_refresh/_constants.py` (`_LIVE_FILE_REFRESH_STATUSES`).
- Kill eligibility: `integrations/_mobile_agent_summary.py` (`_KILLABLE_ACTIVE_STATUSES`).
- Revival artifacts: `ace/tui/actions/agents/_revive_artifacts.py` (`_PLAN_LIKE_STATUSES` and the two inline status
  lists).
- Time anchoring and navigation (`status == "PLAN" and agent.plan_times` → `is_pending_plan_review_status`):
  `ace/tui/models/agent_time.py`, `ace/tui/actions/agents/_unread_navigation.py`.
- Clan rollup: `ace/tui/models/_agent_clan.py::aggregate_clan_status` — after the QUESTION check, return the first
  member of `PENDING_PLAN_REVIEW_STATUSES` present among member statuses (EPIC outranks TALE outranks PLAN), so a clan
  holding a pending epic surfaces the bigger review ask.
- Query language (`ace/agent_query/`): `status:` is substring matching, so `status:tale` / `status:epic` match the new
  labels with no code change (alongside `TALE DONE` / `EPIC CREATED` etc. — same pre-existing ambiguity `status:plan`
  has today; document, don't fix). `attention:` and Stopped grouping follow automatically from `_STOPPED_STATUSES`. Only
  test updates needed here.

Explicit non-status usages of the words PLAN/EPIC/TALE that must NOT be touched: prompt-panel context-lane labels
(`_agent_plan_section.py`, `_agent_context.py`, `_agent_clan_sections.py`, `agent.py` timeline lanes), the ChangeSpec
`PLAN:` field (`workflows/commit/commit_hooks.py`), commit-log tag styles (`vcs_log/_tag_style.py`), ChangeSpec fold
labels (`ace/tui/widgets/commits_builder.py`), and the auto-approve action alias display (`⚡ PLAN`, driven by
`auto_approve_plan_action`, which names the approval action, not the status).

After wiring the inventory, do a final sweep: grep the codebase for remaining comparisons against the bare `"PLAN"`
status string and confirm each is either converted or one of the documented non-status usages.

### 5. Colors

Palette decision (Agents-tab row renderer, `ace/tui/widgets/_agent_list_render_agent.py`):

- **`TALE` → `bold #FF87AF`** (the classic pending-plan pink). Tale is the direct successor of today's `PLAN` in ~all
  real cases; users' eyes are already trained that this pink means "a plan is waiting for you". The label change carries
  the new information; the hue preserves recognition.
- **`EPIC` → `bold #D787FF`** (orchid). A regal purple that visually signals "bigger review ask", clearly distinct from
  TALE's pink, from FEEDBACK's magenta `#FF5FD7`, from WAITING's amethyst `#AF87FF`, and from the epic terminal green
  `#5FD7AF` (`EPIC CREATED`/`EPIC APPROVED`) — pending stays in the warm/attention family, terminal stays green.
- **`PLAN` (fallback) → unchanged** everywhere.

Apply per-surface: in each status→style map, `TALE` mirrors that surface's existing `PLAN` style and `EPIC` uses
`#D787FF` (bold where the surface bolds). Surfaces to update:

- `ace/tui/widgets/_agent_list_render_agent.py` (main row renderer — add the two branches beside the `PLAN` branch).
- `ace/tui/modals/zoom_panel_rendering.py::status_text` (there `PLAN` is `bold #FFD787`, so `TALE` mirrors that; `EPIC`
  gets `bold #D787FF`; both also join its `ACTIVE_STATUSES` icon set per §4).
- `ace/tui/widgets/prompt_panel/_workflow_render.py` status style map.
- `ace/tui/modals/agent_neighbor_modal.py::_status_style`.
- `ace/tui/modals/jump_all_modal.py` has no `PLAN` entry today (empty-style fallback) — leave as-is for all three
  pending labels.
- Revive/saved-group revival modals only ever show dismissable statuses, so pending labels cannot appear there; skip.

### 6. Explicitly out of scope

- **Rust core (`sase-core`) is intentionally untouched.** Agent statuses are derived presentation-shared logic that
  lives in Python (`status_buckets.py` et al.); the Rust scan wire needs no new field under this design, and the Rust
  cleanup planner's `DISMISSABLE_STATUSES` mirror is unaffected because pending statuses are not dismissable.
- Approved/working/terminal status naming (the action-vs-tier nuance noted in Context).
- `needs:input` query semantics (continues to exclude pending plan reviews).
- Notification titles/copy ("Plan ready for review" / "Epic ready for review" already tier-aware in `plan_gate.py`).

## Testing

- Unit: `pending_plan_status_for_tier` / `is_pending_plan_review_status` / `cached_plan_tier` (tier read, cache
  invalidation on rewrite, missing file → `None`); `plan_enrichment_status` tier branches; bucket membership
  (`tests/ace/tui/models/test_agent_status_stopped.py`, grouping-mode/tree status tests); clan rollup priority
  (`tests/test_agent_clan.py`, `tests/ace/tui/widgets/test_agent_clan_aggregation.py`).
- Derivation: extend the loader/status-override suites (`tests/test_enrich_agent_plan_meta.py`,
  `tests/test_agent_loader_status_override_*.py`, `tests/ace/tui/test_agent_notification_status_overrides.py`) with tale
  and epic pending fixtures, including: tier read from a real plan file frontmatter; unreadable/missing plan file → bare
  `PLAN`; EpicApproval notification → `EPIC` without any plan file on disk.
- Consumers: `tests/test_agent_list_entries.py` (`needs_user_action`), `tests/test_agent_query_evaluator.py`
  (`attention:true` matches TALE/EPIC; `needs:input` still excludes them; `status:tale`/`status:epic` substring cases),
  footer/approve-eligibility tests around the touched keybinding paths.
- Visual: add a PNG snapshot mirroring `test_agent_plan_handoff_status_colors_png_snapshot` in
  `tests/ace/tui/visual/test_ace_png_snapshots_agents.py` with rows in `EPIC`, `TALE`, and fallback `PLAN` states to
  lock the new colors; run `just test-visual` and accept intentional changes with `--sase-update-visual-snapshots` only
  for the new/changed goldens.
- Full gate: `just install` then `just check` before finishing.

## Risks & edge cases

- **Legacy artifacts**: rows whose plan file no longer resolves show bare `PLAN` — intentional graceful degradation,
  covered by tests.
- **Perf**: tier reads are loader-side, bounded, mtime-invalidated, and limited to pending-plan rows; no render-path I/O
  is introduced (consult `sase memory read tui_perf.md` before altering any refresh path).
- **Query ambiguity**: `status:epic` also matches `EPIC CREATED`/`EPIC APPROVED` via substring matching — pre-existing
  behavior for `status:plan`; documented, not changed.
- **Mixed clans**: rollup priority (EPIC > TALE > PLAN, after QUESTION) is explicit and tested.
- **Override churn**: the notification override's write-once guard must compare against the computed tiered label, not
  the literal `"PLAN"`, or every refresh re-marks the row changed.

## Acceptance criteria

1. An agent awaiting review of a tale plan shows `TALE` (pink); an epic plan shows `EPIC` (orchid); a row whose plan
   tier is unresolvable shows `PLAN` exactly as today.
2. All three labels behave identically for bucketing (Stopped), `attention:`, approve keybindings/footer hints,
   notification jump/dismiss flows, clan counts, kill/refresh eligibility, and mobile needs-action.
3. Pending-plan semantics are defined once in `status_buckets.py` and every touched call site consumes the shared
   helpers — no site enumerates the three labels locally.
4. `just check` and the full test suite (including the new PNG snapshot) pass; no `sase-core` changes are required.
