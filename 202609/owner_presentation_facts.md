---
tier: tale
title: Resolve production family presentation facts
goal:
  The gateway-served fleet catalog carries the owner's real per-record presentation
  facts — rich status with bucket, shell kind/state and chip facts, role, direct and
  inherited tribe, and family/run/completion timestamps — so the shared viewer renderer
  reproduces the owner's rows for the production fixture.
size: medium
proposed_by: bbugyi200.athena.sase-133.5.2
bead: sase-133.5.2
status: done
---

- **BEAD:**
  [sase-133.5.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-133/sase-133.5.2.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-133.5.2](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-133.5.2.md)
- **COMMITS:**
  - [92cf0ca](https://github.com/sase-org/sase-core/commit/92cf0ca230a436a6dd48c9cd09aeb70759119404)
    — feat(fleet): derive owner presentation facts in core and bump contract to v5

# Resolve Production Family Presentation Facts

Phase `owner-facts` of epic `sase-133.5`
(`plan:202609/remote_parity_landing_repairs.md`, bead `sase-133.5.2`). The preceding
phase `owner-roster` (`sase-133.5.1`) already made the gateway and the owner loader
agree on _which_ rows exist. This tale makes them agree on _what those rows say_.

Do not close, update, or symvision-sweep any ancestor bead. Record discovered follow-up
work with `sase bead note sase-133.5.2 'PROPOSED FOLLOW-UP: <summary — detail>'`.

## Problem

Authenticated apollo renders its own families as `TALE DONE ×7` and `EPIC CREATED √ ×7`.
The viewer, reading the same host through the fleet catalog, shows `0n--plan` and
`0k--plan` as unparented `DONE` gate rows and loses the family's rich status entirely.
The settled owner capture reports 20 agents (`@default=8`, `@epic=6`, `@research=6`);
the near-contemporaneous viewer capture filtered to `machine:apollo` reports 30 agents
with panel counts 1/23/6. Those three surfaces count different objects, so treat only
per-identity fact equality as the oracle, never raw row totals.

Root cause: `display_status_for_record()` in
`sase/repos/linked/sase-core/crates/sase_core/src/fleet_catalog.rs` derives a coarse
six-way status (`STOPPED`/`FAILED`/`DONE`/`QUESTION`/`WAITING`/`RUNNING`) and
`PresentationRecordFacts` carries no shell, plan, question, retry, or role facts at all.
The gateway therefore cannot express the owner's plan-chain lifecycle, and the viewer
cannot reconstruct it.

## Key discovery: do not port the family status rules

The viewer **already reuses** the owner's family status pipeline.
`normalize_remote_host_nodes()` in `src/sase/ace/tui/models/_fleet_agents_nodes.py`
calls `apply_status_overrides(agents, classify_diff_badges=False)` — the same
`_agent_status_apply.py` pass that produces `TALE DONE`, `EPIC CREATED`, `WORKING PLAN`,
`WORKING TALE`, `PLAN APPROVED`, `ANSWERED`, and root mirroring for local rows. `×N`
comes from the shared fold-count annotation and the bucket glyph from the shared
`status_bucket`.

So this tale must **not** reimplement family status policy in Rust. It must supply the
per-record _base_ facts that `apply_status_overrides` consumes, which today arrive
empty. The Rust work mirrors exactly one Python function's per-record half:
`enrich_agent_from_meta_wire()` in
`src/sase/ace/tui/models/_loaders/_meta_enrichment_wire.py` plus its helpers in
`_meta_enrichment_status.py`. That function already reads only wire types
(`AgentMetaWire`, `WaitingMarkerWire`, `PendingQuestionMarkerWire`, `DoneMarkerWire`,
`WorkflowStateWire`), which is why the port is tractable: identical inputs, same shapes.

Concretely, the container gets `TALE DONE` only when the remote rows carry
`plan_chain_root` (or the `--plan` role suffix that `mark_derived_plan_family_roots()`
derives), `agent_family_parallel`, `role_suffix`, `plan_action`, `plan_times`,
`gate_id`, and the answered-question signal. Every one of those is read speculatively
today by `_agent_from_summary()` in `src/sase/ace/tui/models/_fleet_agents_rows.py` and
always resolves to `None`, because `ResolvedAgentSummaryWire` never had the field.

## The shared seam

Both consumers already funnel through the same three core functions, so extending them
covers the gateway and the Python oracle at once:

- `select_fleet_presentation()` → `PresentationContext::facts_for_record()` →
  `project_resolved_agent_summary()`.
- Gateway: `crates/sase_gateway/src/fleet_reads.rs:864` (`resolve_record`).
- Python oracle: `assemble_fleet_catalog()` in
  `crates/sase_core/src/fleet_catalog.rs:260`, bound as `assemble_fleet_catalog` in
  `crates/sase_core_py/src/lib.rs`.

Keep the gateway a thin observer/cache wrapper. Put every decision in `sase_core`.

## Steps

### 1. Derive owner base facts in `sase_core`

Open the core repo with `/sase_repo` (`sase repo open sase-core -r "..."`); never assume
a sibling path. Add `crates/sase_core/src/fleet_owner_facts.rs` and register it in
`lib.rs`.

Implement a per-record derivation that mirrors `enrich_agent_from_meta_wire` and
`_meta_enrichment_status.plan_enrichment_status` / `pending_review_window_active` /
`pending_question_status_for_request_path`:

Base status, in the owner's order (later rules override earlier ones only where the
Python does):

1. `done` present → `STOPPED` when `repeat_stopped`; else `done.status_label`; else
   `FAILED` when `error` is non-blank or `outcome` starts with `failed`; else `DONE`.
2. Otherwise `STARTING` / `RUNNING` from `workflow_state` and `running`, promoted from
   `STARTING` to `RUNNING` by `meta.run_started_at` or `meta.wait_completed_at`
   (`ACTIVE_ENRICHMENT_STATUSES` is `{STARTING, RUNNING}`; only those two are eligible
   to be overridden below).
3. `waiting` marker → `WAITING` when the status is still active.
4. `pending_question` marker → `QUESTION`, or `ANSWERED` when the owner observes a
   persisted response (see step 2). A simultaneous `waiting` marker wins, matching the
   Python comment: the answered root is queued to reacquire its slot.
5. `meta.plan` → the plan enrichment status: `EPIC FAILED` / `PLAN FAILED` for
   `plan_action in {epic_failed, failed}`; for `plan_approved`, `PLAN COMMITTED`
   (suppressed when `plan_committed is false`), `TALE APPROVED`, `EPIC APPROVED`, or
   `PLAN APPROVED` by `plan_action`; otherwise, when submitted and not auto-approved,
   the tier-pending label `EPIC` / `TALE` / `PLAN`. Reopen this for a `DONE` row only
   through the `pending_review_window_active` conditions the wire can see
   (`plan_submitted`, not approved/actioned/auto-approved, no `stopped_at`, no done
   marker, live pid) — keep the same deliberate tolerance for `gate_id` /
   `gate_member_agent_name` that the Python wire path documents.

Facts, from `meta.family_shell` (`FamilyShellWire`, discriminated by `kind`):
`monitor_id`, `monitor_state`, `monitor_label`, `monitor_command`, `gate_id`,
`gate_kind`, `gate_state`, `gate_label`, `gate_accent`, `proc_id`, `proc_status`,
`proc_label`, and the shell `start_status` / `stop_status` pair.

Topology and lifecycle facts: `role_suffix`, `agent_family_parallel`, `plan_chain_root`,
`plan_action`, `plan_committed`, `plan_tier`, plan-submitted times, question-submitted
times, `epic_started_at`, `retry_of_timestamp`, `retry_attempt`, `retry_terminal`, and
`reasoning_effort`. Direct and inherited `tribe` / `clan_tribe` already resolve through
`PresentationContext`; keep that and add the shell/plan facts alongside them in
`PresentationRecordFacts`.

Delete `display_status_for_record()` from `fleet_catalog.rs` and route
`direct_presentation_facts_for_record()` through the new module, so exactly one status
derivation exists.

### 2. Make the two owner-side filesystem observations injectable

Two owner facts are not in the record and must not become viewer work:

- Whether a pending question already has a persisted response (the Python checks for a
  sibling `question_response.json` next to `pending_question.request_path`).
- The plan tier for a submitted, unapproved plan (`cached_plan_tier(meta.plan_path)`
  reads the plan file's frontmatter).

Resolve both on the owner side behind a trait, following the existing
`OwnerLivenessObserver` / `HostOwnerLivenessObserver` precedent in
`crates/sase_core/src/host_liveness.rs`: a host implementation that stats/reads, and an
injectable map for hermetic tests, threaded through `AssembleFleetCatalogRequestWire`
the way `observations` already is. Bound and cached like the liveness observer; do not
read a plan file per record per refresh.

### 3. Extend the wire additively, with no paths

Add the new facts to `OwnerResolutionFactsWire` and `ResolvedAgentSummaryWire` in
`crates/sase_core/src/fleet_contract.rs`. Every new field is `Option`/`Vec`/`bool` with
`#[serde(default)]` so an older payload still deserializes under the existing
`deny_unknown_fields`; a legacy omission must render degraded, never raise.

Honor the summary's documented invariant: it never serializes artifact directories,
checkout paths, marker paths, or output/response paths. So ship **no** path fields.
Where the owner's rules key off a path's existence, ship the derived boolean:
`question_answered` instead of `question_response_path`, `plan_tier` instead of
`plan_path`. Trim labels through the existing `MAX_LABEL_BYTES` helper and validate the
new timestamps in `validate_resolved_agent_summary()`.

Bump `FLEET_CONTRACT_SCHEMA_VERSION` from 4 to 5 and leave
`FLEET_CONTRACT_MIN_READABLE_SCHEMA_VERSION` at 1. Phase `version-diagnostics`
(`sase-133.5.3`) already advertises `fleet_contract_schema_version` independently of
capabilities; update the contract fixture in
`crates/sase_gateway/contracts/api_fleet_v1/fleet_api_v1.json` and any pinned
schema-version fixtures to match.

### 4. Invalidate caches when presentation facts change

`stable_revision()` in `fleet_catalog.rs` hashes the raw record only, so a row whose
_derived_ facts changed (inherited tribe, resolved parent linkage, answered question,
newly read plan tier) can keep a stale revision and a stale viewer cache entry. Fold the
resolved `PresentationRecordFacts` into the revision hash. Keep the hash stable for an
unchanged row — a revision that churns every refresh defeats incremental refresh.

### 5. Carry the facts through the Python adapters

In `src/sase/ace/tui/models/_fleet_agents_rows.py`, `_agent_from_summary()` already
reads most of the new names speculatively (`role_suffix`, `gate_id`, `gate_kind`,
`gate_state`, `gate_label`, `monitor_id`, `monitor_state`, `monitor_command`,
`monitor_label`, `proc_id`, `proc_status`, `proc_label`, `agent_family_parallel`,
`plan_chain_root`, `reasoning_effort`). Match those exact key spellings in the wire so
they light up, and keep each existing fallback for legacy payloads — the current
`monitor_state` / `gate_state` liveness guesses must stay as the degraded path.

Add the plan, question, retry, and epic facts, converting unix times to the local-aware
datetimes the models use (`sase.core.time.to_local`, as
`_meta_enrichment_status.parse_utc_to_local` does).

Add an `Agent` field for the answered-question signal (e.g. `question_answered`) and
teach `is_answered_continuation_asker()`, `is_answered_root_asker_step()`, and
`approved_followup_planner_status()` in
`src/sase/ace/tui/models/_agent_status_family_policy.py` to accept it in place of a
`question_response_path`. Local rows keep setting the path and must behave identically;
prefer OR-ing the new flag with the existing check over rewriting the predicates.

Do not touch Textual rendering, layout, or keybindings. `normalize_remote_host_nodes()`
keeps calling `apply_status_overrides`; preserve `classify_diff_badges=False` (remote
rows have no readable diff artifact) and preserve the concrete action/content identity a
family container summarizes.

### 6. Extend the production fixture

`tests/ace/tui/owner_roster_fixture.py` currently writes gate/monitor facts as
_top-level_ `agent_meta.json` keys (`gate_id`, `gate_state`, `monitor_id`). The real
persisted shape is the nested `family_shell` object the wire reads (`kind`, `id`,
`state`, `label`, `start_status`, `stop_status`, plus a `gate` / `monitor` sub-object).
Correct the fixture to the real shape first — a fixture that does not match production
is how a second synthetic oracle gets built by accident.

Then add, keeping every existing identity and its assertions intact:

- A tale plan-chain family reproducing `0n`: promoted root, a `--plan` gate shell, and a
  completed `--code` member, expected to render `TALE DONE` on the container.
- An epic plan-chain family reproducing `0k`: root, `--plan` shell, completed `--epic`
  member, expected to render `EPIC CREATED`.
- An active family whose coder is running (expect `WORKING PLAN` / `WORKING TALE`) and a
  waiting family (expect the waiting root projection).
- A pending-review row with a submitted, unapproved plan, to exercise the tier-pending
  label and the injected plan-tier observation.
- An answered-question continuation, to exercise `ANSWERED` through the injected
  response observation rather than a shipped path.

### 7. Prove parity on facts, not just identities

Extend the phase-1 oracle rather than adding a parallel one. Either grow
`tests/ace/tui/test_owner_roster_oracle.py` or add a sibling
`tests/ace/tui/test_owner_facts_oracle.py` that reuses the same fixture and the same two
real paths: `load_tiered_agents()` on one side, and `assemble_fleet_catalog()` →
`project_fleet_agents()` on the other. Compare per-identity facts, asserting each
dimension independently so a failure names the dimension:

- `status` and `status_bucket`
- `agent_family_role`, `role_suffix`, `parent_timestamp`, nesting
- `tribe` and `clan_tribe`, direct and inherited
- shell kind, shell state, and the chip ids (`monitor_id` / `gate_id` / `proc_id`)
- `start_time`, `run_start_time`, `stop_time`
- panel grouping and per-panel counts

Run it over both the current and the compact-index (`agents_list_projection=True`)
paths, as the phase-1 oracle already does.

Add targeted regressions: the `0n` / `0k` containers carry the rich status and no bare
`--plan` shell appears as an unparented visible `DONE` row; a legacy summary missing
every new field still projects rows without raising, with the documented degraded
rendering.

On the Rust side add unit tests in `fleet_owner_facts.rs` for each status rule and shell
kind, and a `fleet_contract.rs` test that a v4-shaped payload still deserializes at v5.

### 8. Verify

In `sase/repos/linked/sase-core`: `just check` from the repo root. Never verify with
`cargo test -p sase_core` alone — it skips the `sase_core_py` binding tests, which is
exactly where stale schema-version fixtures have reached master before.

In the primary repo: `just install` first (dev installs build `sase_core_rs` from the
linked checkout, so the Rust change is not visible to pytest until then), then
`just fix` inline, then `just check`. Hand `just check` to `/sase_monitor` with the
`TESTING` / `TESTED` status pair if it runs long. Do **not** run `just check-full` —
nothing in this tale explicitly instructs it.

This tale changes no rendered TUI layout, so PNG goldens should not move. If
`just check` or a visual run reports golden drift, inspect the retained report and every
changed golden before accepting anything; generation is not approval.

## Constraints

- Shared selection and presentation decisions belong in `crates/sase_core`. The gateway
  observes and caches; Python adapts and renders.
- Preserve dismissal, recycled-PID protection, resource capabilities, content identity,
  and older-host tolerant readers. A pending gate shell can legitimately outlive its
  creator PID — never infer that a dead-PID shell is obsolete.
- Do not treat terminal process liveness as transport-offline health.
- Keep shared-state decisions out of the event loop and preserve incremental refresh.
- No viewer filesystem access may be needed to reconstruct remote facts.
- Compare actual production payloads, not fixture-only fields.

## Acceptance

The production fixture's owner rows and remote rows have matching normalized rendered
rows apart from the machine chip and explicitly preserved remote affordances. The `0n` /
`0k` family failures and the active/waiting families reproduce as tests and then pass.
Legacy payloads degrade without exceptions. `just check` passes in `sase-core` and in
the primary repo.

This tale does not deploy, does not capture live screenshots, and does not close the
parent epic — phase `parity-acceptance` (`sase-133.5.4`) owns that evidence.
