---
tier: tale
title: Rebuild plan and question statistics on durable gate bundles
goal:
  The Statistics "Plans & Questions" sub-tab reports every plan proposal and question session in the selected window,
  sourced from durable notification-gate bundles instead of the agent artifact index, so handed-off planner agents are
  no longer silently dropped.
proposed_by: bbugyi200.athena.rn
create_time: 2026-08-02 06:43:13
status: wip
---

- **PROMPT:**
  [prompts/202608/plan_question_stats_from_gates.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/plan_question_stats_from_gates.md)

# Rebuild Plan & Question Statistics On Durable Gate Bundles

## Problem

The **Statistics → Plans & Questions** sub-tab of the SASE Admin Center reports wildly low numbers. For the window
`2026-07-26 06:28 EDT → 2026-08-02 06:28 EDT` it showed:

```
Plans      Proposed 3 · Approved 3 · Rejected 0 · Pending 0
Tier       tale  3
           Mean phases per epic: 0.00
Questions  Sessions 7 · Asking agents 7 · Questions 0
           Mean questions per session: 0.00
```

Ground truth for that same window, measured from the durable notification-gate bundles under
`~/.sase/interaction_requests/`:

| Metric         | Reported | Actual                             |
| -------------- | -------- | ---------------------------------- |
| Plans proposed | 3        | **233**                            |
| tale plans     | 3        | **156**                            |
| epic plans     | 0        | **77**                             |
| Questions      | 0        | **7 sessions, non-zero questions** |

The panel is under-reporting plans by roughly **78x**. Three independent defects cause it.

## Root Causes

### 1. The `hidden = 0` filter silently drops handed-off planner agents (primary cause)

Both aggregation queries filter hidden rows out of the agent artifact index:

- `crates/sase_core/src/agent_stats/activity.rs:277` — `WHERE hidden = 0`
- `crates/sase_core/src/agent_stats/run.rs:205` — `WHERE hidden = 0`

A planner agent (`role_suffix == "--plan"`) submits its plan, then hands off to an implementer that runs in a **new**
artifacts directory. The planner's own directory therefore never receives a `done.json`. That makes it a terminalization
candidate under `record_is_terminalization_candidate` (`crates/sase_core/src/agent_scan/index.rs:1344`), so
`terminalized_abandoned_record` (`crates/sase_core/src/agent_scan/index.rs:1432`) synthesizes a done marker with
`outcome: "abandoned"` and `hidden: true`. `RecordSummary::from_record`
(`crates/sase_core/src/agent_scan/index.rs:2117`) then ORs `done.hidden` into the row's `hidden` column.

Measured on the live index for the 7-day window: **144 plan submissions exist, 141 of them sit on `hidden = 1` rows,
leaving exactly the 3 the panel displayed.** On every one of those 141 rows `agent_meta.hidden` is `false` — the user
never hid these agents. The index hid them as a side effect of the plan-chain handoff, and the statistics query
inherited that decision.

### 2. The agent artifact index is the wrong source, and it windows on the wrong timestamp

Even with the `hidden` filter removed, the index yields only 144 of the 233 real proposals, and what it does yield is
imprecise:

- **Wrong window key.** `scan_plan_activity` (`activity.rs:299-309`) windows on the agent's _launch_ time
  (`started_at`), not on when the plan was submitted. A long-running agent that proposes a plan just inside the window
  is dropped; one launched inside the window that proposes after it ends is counted.
- **Fragile tier resolution.** `resolve_plan_document_stats` (`activity.rs:365`) guesses at plan-document locations by
  probing `sdd_plan_path`, `plan_path`, marker paths, workspace-relative paths, and then every month shard under
  `~/.sase/plans/`. Any plan document that has since moved resolves to the `unknown` tier and inflates
  `unresolved_plan_files`.
- **Lossy metadata.** `plan_submitted_at` is only written by `handle_plan_marker`
  (`src/sase/axe/run_agent_exec_plan.py:92-109`) into the _then-current_ artifacts directory, which later steps in the
  chain supersede.

### 3. Question statistics read a store that has been dead since 2026-07-16

`scan_question_sessions` (`crates/sase_core/src/agent_stats/activity.rs:197-251`) reads
`~/.sase/user_question/<uuid>/question_request.json`. The unified-notification-gates work moved question sessions to
`~/.sase/interaction_requests/question/<request_id>/`. The legacy directory holds 23 sessions, **none newer than
2026-07-16 and zero inside the reported window** — hence `Questions 0`.

The `Sessions 7 · Asking agents 7` line comes from a different source (`run.rs:710`, reading
`meta.questions_submitted_at`), which is why the panel contradicts itself. That 7 exactly matches the 7 question gate
bundles in the window, confirming the gate store is the accurate one.

## Approach

Stop deriving plan and question activity from the agent artifact index. Read it from the durable gate bundles that the
notification-gate system already writes, which carry every field the panel needs _directly_ and require no inference:

```
~/.sase/interaction_requests/plan/<request_id>/       # tale proposals   (513 bundles, 156 in window)
~/.sase/interaction_requests/epic_plan/<request_id>/  # epic proposals   (164 bundles,  77 in window)
~/.sase/interaction_requests/question/<request_id>/   # question sessions ( 15 bundles,   7 in window)
```

Each bundle contains `request.json`, `plan.md` (plan kinds only), and `response.json` once answered:

| Needed value                          | Source                                                                       | Replaces                                         |
| ------------------------------------- | ---------------------------------------------------------------------------- | ------------------------------------------------ |
| Proposal time                         | `request.json` → `payload.timestamp`                                         | agent launch time (wrong key)                    |
| Tier                                  | `request.json` → `payload.authored_tier`                                     | `resolve_plan_document_stats` path probing       |
| Phase count                           | bundled `plan.md` frontmatter `phases:`                                      | month-shard path guessing                        |
| Approve / reject / feedback / pending | `response.json` → `selected_option_ids` / `choice_id`; absent file = pending | `plan_approved` / `plan_action` / `done.outcome` |
| Agent attribution                     | `request.json` → `producer.agent_name`                                       | index `agent_name`                               |
| Project                               | `request.json` → `producer.artifacts_dir` → `~/.sase/projects/<key>/…`       | not currently possible                           |
| Question count                        | `request.json` → `payload.questions` array length                            | dead `user_question` store                       |

This source is strictly better than the index: it is written at proposal time by the gate creator, it is never hidden by
agent-lifecycle bookkeeping, and it survives the plan-chain handoff that destroys the index signal.

**Verified properties of the gate store** (checked on live data before writing this plan):

- All 164 `epic_plan` bundles have `plan.md` with a `phases:` frontmatter key — mean-phases-per-epic becomes exact.
- `authored_tier` is `tale` for all 513 `plan` bundles and `epic` for all 164 `epic_plan` bundles.
- Bundles are **durable and never time-pruned**. The only `shutil.rmtree` calls in
  `src/sase/notification_gates/service.py` (lines 78 and 151) clean up failed or half-initialized creations.
- 10 of 513 plan bundles have no `response.json` — these are genuine pending proposals.
- `response.json` has two shapes that both must be handled: a `selected_option_ids` list (schema_version 2, 467 bundles)
  and a `choice_id` scalar (36 bundles).

**Accepted limitation:** the gate store begins at 2026-07-16, when unified notification gates landed. Ranges reaching
further back have no gate data. Handle this explicitly rather than silently reporting zero (see step 6).

## Implementation

Steps 1–5 are in the **`sase-core`** repo (open it with `/sase_repo`; the aggregation is core backend logic per the
`rust_core_backend_boundary` memory). Steps 6–8 are in the **`sase`** repo.

### Step 1 — Add a gate-bundle reader in `sase-core`

Create `crates/sase_core/src/agent_stats/gate_bundles.rs` with a reader that, given `sase_home` and a kind, iterates
`sase_home/interaction_requests/<kind>/*/` and returns a parsed struct per bundle:

```rust
struct GateBundle {
    request_id: String,
    timestamp: f64,          // payload.timestamp, falling back to created_at_unix then created_at
    authored_tier: Option<String>,
    producer_agent: Option<String>,
    project_key: Option<String>,   // parsed from producer.artifacts_dir
    questions: u64,                // question kind only
    phase_count: Option<u64>,      // parsed from the bundled plan.md frontmatter
    outcome: GateOutcome,          // Approved | Rejected | Feedback | Pending
}
```

Follow the existing error discipline in `activity.rs`: a missing or malformed bundle increments a `malformed_*_skipped`
counter and is skipped independently, never aborting the scan.

Parse `outcome` from `response.json`:

- file absent → `Pending`
- `selected_option_ids` contains `reject` → `Rejected`; contains `feedback` → `Feedback`; contains `approve` or `commit`
  → `Approved`
- otherwise fall back to the scalar `choice_id` with the same mapping

Reuse `crate::plan::read::split_frontmatter` (already imported at `activity.rs:13`) to read `phases:` from `plan.md`.

Derive `project_key` by locating the `projects/<key>/` segment in `producer.artifacts_dir`.

### Step 2 — Rewrite `scan_plan_activity`

Replace the body of `scan_plan_activity` (`crates/sase_core/src/agent_stats/activity.rs:253-363`) to read the `plan` and
`epic_plan` gate kinds instead of opening the index. It no longer needs the `index_path` argument.

- Window on the bundle `timestamp`, using the existing `in_window` helper (`activity.rs:535`).
- `proposed` counts in-window bundles; `approved` / `rejected` / `pending` come from `GateOutcome`. Add a `feedback`
  count to `AgentPlanActivityStatsWire` — feedback rounds are currently invisible and were being miscounted as
  `pending`.
- `tiers` counts `authored_tier` directly.
- `phases_per_epic` and `mean_phases_per_epic` use `phase_count` from the bundled `plan.md`, over `epic_plan` bundles.
- Apply `request.project` against `project_key` so plans finally honor the project filter.

Delete the now-dead `resolve_plan_document_stats`, `read_plan_document_stats`, `plan_month_candidates`,
`is_month_shard`, `expand_home_reference`, `push_unique`, and `sorted_subdirs` helpers (check each for other callers
first — `sorted_subdirs` is used by `scan_question_sessions`, which step 3 also rewrites). Retire the
`PlanDocumentStats` struct (`activity.rs:38-42`) and the `unresolved_plan_files` response field if nothing else reads
it; otherwise leave it wired at zero and note it as deprecated in the wire doc comment.

### Step 3 — Rewrite `scan_question_sessions`

Point `scan_question_sessions` (`activity.rs:197-251`) at the `question` gate kind. Window on `payload.timestamp`; count
`payload.questions` array length per session; keep the existing `questions_per_session` distribution and
`mean_questions_per_session` shape. Apply the project filter via `project_key`.

Do **not** keep a fallback to `~/.sase/user_question/`. That store is dead and blending it in would double-count any
overlap.

### Step 4 — Reconcile the run-payload plan and question counts

`run.rs` independently derives `plans.proposed/approved/rejected/pending` (`run.rs:683-687`) and
`questions.sessions/asking_agents` (`run.rs:710-713`) from index metadata behind the same `WHERE hidden = 0` filter
(`run.rs:205`). Leaving it as-is guarantees the panel keeps contradicting itself, because `build_plans_questions_view`
mixes both payloads.

Take the simpler, correct route: make the **activity payload authoritative** for every plan and question figure, and
have step 7 stop reading the run payload for them. Keep the run-payload fields populated for wire compatibility, but add
a doc comment on `AgentPlanStatsWire` and `AgentQuestionStatsWire` (`wire.rs:163-184`) stating that they are
index-derived, under-count handed-off planners, and must not be used for user-facing totals.

Also add `proposing_agents` (distinct `producer_agent`) and `asking_agents` (distinct `producer_agent`) to the activity
wire structs so the panel can source those from the accurate side too.

### Step 5 — Rust tests

In `crates/sase_core/src/agent_stats/activity.rs` tests, build a temporary `interaction_requests` tree and cover:

- a `plan` bundle and an `epic_plan` bundle in window produce `proposed = 2` with `tiers = {tale: 1, epic: 1}`
- **regression for the primary bug:** a plan bundle whose producing agent's index row is `hidden = 1` with
  `done.outcome = "abandoned"` is still counted — this is the exact shape of the 141 dropped proposals
- windowing uses `payload.timestamp`, not the producer agent's launch time
- each of `Approved` / `Rejected` / `Feedback` / `Pending` (missing `response.json`) maps correctly, through **both**
  the `selected_option_ids` and the `choice_id` response shapes
- `mean_phases_per_epic` computes from bundled `plan.md` frontmatter
- the project filter scopes on `producer.artifacts_dir`
- a malformed `request.json` increments the skip counter without aborting the scan

Update the existing `aggregates_logs_questions_and_plan_documents` test (`activity.rs:594`), which fixtures the old
index-and-plan-document layout and will no longer be meaningful.

### Step 6 — Handle the pre-2026-07-16 coverage floor

Add a `coverage_start_ts` field to `AgentActivityStatsResponseWire`, set to the earliest gate-bundle timestamp observed.
In the `sase` repo, render a note in the Plans & Questions view when the requested range starts before
`coverage_start_ts` — something like `plan/question data begins <date>`. This keeps an honest signal instead of silently
reporting zero for older ranges.

### Step 7 — Update the Python view builder

In `src/sase/stats/_view_builders.py:524-546`, source `plans_proposed`, `plans_approved`, `plans_rejected`, and
`plans_pending` from `activity_payload["plans"]` rather than `run_payload["plans"]`, and `question_sessions` /
`asking_agents` from `activity_payload["questions"]` rather than `run_payload["questions"]`. Add the new
`plans_feedback` and `coverage_start_ts` fields to `PlansQuestionsView` (`src/sase/stats/_view_models.py:252`).

### Step 8 — Update the TUI view and legend

Render the new `Feedback` count alongside Proposed / Approved / Rejected / Pending in
`src/sase/ace/tui/modals/statistics_pane_views.py`.

Update `src/sase/ace/tui/modals/statistics_pane_legends.py:102`: plans and questions are now project-scopable, so
`"global durable activity the project filter cannot scope"` must no longer claim to cover them. Narrow that legend to
the metrics that genuinely remain global (skills and memories).

## Verification

1. `just check` in the `sase` repo, and the crate's own test suite in `sase-core`.
2. Rebuild the binding (`just install`) and re-query the exact window from the screenshot:

   ```python
   from sase.stats.query import query_activity_stats
   payload = query_activity_stats(start_ts=<2026-07-26 06:28 EDT>, end_ts=<2026-08-02 06:28 EDT>)
   ```

   Expect `plans.proposed ≈ 233` with `tiers ≈ {tale: 156, epic: 77}` and a non-zero `questions.questions` over 7
   sessions — not 3 and 0. Small drift from re-proposals landing after this plan was written is expected; an answer
   still in the single or double digits means the fix did not take.

3. Open the panel and confirm the Plans & Questions sub-tab agrees with those numbers, that the tier table lists both
   `tale` and `epic`, and that `Mean phases per epic` is non-zero.
4. Switch the project filter (`p`/`P`) and confirm plan and question counts now change with it.

## Out Of Scope

- Changing the index's `hidden`/terminalization behavior. Marking handed-off planners abandoned is arguably wrong on its
  own terms and worth a separate task bead, but this plan routes around it rather than perturbing agent-list semantics
  that the TUI depends on.
- Backfilling plan or question activity from before 2026-07-16.
