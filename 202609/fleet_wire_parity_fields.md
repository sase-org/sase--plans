---
tier: tale
title: Core wire carries the missing presentation facts
goal:
  Owner-served fleet summaries carry real start/run/stop timestamps, workspace number,
  clan/tribe identity, the human project label, and the row-visible presentation and
  shell facts that a renderer-driven audit proves the local Agents-tab renderer uses,
  all as additive, backward-compatible sase-core wire fields, with the gateway contract,
  PyO3 binding coverage, and core tests updated, and the exact surface recorded for the
  adoption phase.
size: medium
proposed_by: bbugyi200.apollo.sase-xe.16.11.7.15.3
bead: sase-xe.16.11.7.15.3
create_time: 2026-09-13 18:53:51
status: wip
---

- **PARENT:**
  [202609/remote_agents_display_parity.md](https://github.com/sase-org/sase--plans/blob/main/202609/remote_agents_display_parity.md)
- **BEAD:**
  [sase-xe.16.11.7.15.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xe/sase-xe.16.11.7.15.3.md)

# Core wire carries the missing presentation facts (wire-parity-fields)

Phase bead `sase-xe.16.11.7.15.3` of epic `plan:202609/remote_agents_display_parity.md`
("Remote agents render as real agent nodes"). This phase changes **sase-core only**. It
adds the owner-resolved presentation facts that the local Agents-tab row renderer uses
but `ResolvedAgentSummaryWire` does not yet carry, and it fixes `labels.project_label`
so it holds the human project name. The SASE Python viewer does not change here:
publishing, pin ratchet, and adoption belong to `sase-xe.16.11.7.15.4`, and consuming
the facts belongs to `sase-xe.16.11.7.15.5`.

## Ground rules

- Open sase-core with `/sase_repo`: `sase repo open sase-core -r "<reason>"`. That
  command currently fails with
  `Unknown repo 'sase-core' for project 'gh_sase-org__sase'` (see follow-up F1). If it
  fails, run `sase repo open gh:sase-org/sase-core -r "<reason>"`, which sibling phase
  agents already use. Work only in the path it prints. Before editing, fast-forward that
  checkout to `origin/master` (audited at `3fa0a54`).
- Shared policy stays in Rust core. No Python changes and no feature flags.
- Never hand-edit crate versions; release-plz owns them. Use a conventional
  `feat(fleet): ...` commit. The host finalizer commits the opened core repo.
- Do not create beads. Record discoveries with
  `sase bead note sase-xe.16.11.7.15.3 'PROPOSED FOLLOW-UP: ...'`. Close only this bead;
  never close the parent epic or any ancestor.

## Step 1 — Renderer-driven audit (record it as a phase note)

Re-verify this table against the renderer sources, then record it as a bead note:
`src/sase/ace/tui/widgets/_agent_list_render_agent_prefix.py`,
`_agent_list_render_agent_status.py`, `_agent_list_render_agent.py`,
`_agent_list_render_layout.py`, `_agent_list_render_cache.py` (render key), and the
banner summaries. The viewer mapping it is compared against is `_agent_from_summary` in
`src/sase/ace/tui/models/_fleet_agents_rows.py` (in the sase repo; read it, do not edit
it).

| Agent field(s)                                                                                                                              | Renderer use                                                  | Owner source                                                                                                        | Decision                                                                                                                     |
| ------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| status, status_bucket                                                                                                                       | status parenthetical, banners, folds                          | `status`, `status_bucket`, `lifecycle`                                                                              | already on wire                                                                                                              |
| model, llm_provider                                                                                                                         | provider badge                                                | `model`, `provider`                                                                                                 | already on wire                                                                                                              |
| agent_name / family / parent lineage / shell kind                                                                                           | identity, tree, family chrome                                 | `labels.*`, `family_role`, `parent_timestamp`, `row_kind`, `current_instance`, `container_projected_concrete_agent` | already on wire (consumed by `.15.2`)                                                                                        |
| fleet freshness / health / intent                                                                                                           | fleet chrome                                                  | `freshness`, `connection_health`, `intent`                                                                          | already on wire                                                                                                              |
| start_time                                                                                                                                  | runtime timestamps, sort, clan/family aggregates              | record `timestamp` (14-digit, owner's local zone)                                                                   | **add** `started_at_unix`                                                                                                    |
| run_start_time                                                                                                                              | elapsed time, pre-run WAITING suppression                     | `agent_meta.run_started_at` (UTC ISO)                                                                               | **add** `run_started_at_unix`                                                                                                |
| stop_time                                                                                                                                   | DONE time, duration end, ticking                              | `agent_meta.stopped_at` (UTC ISO; the local loaders use only this)                                                  | **add** `stopped_at_unix`                                                                                                    |
| workspace_num                                                                                                                               | footer and detail header, file panel                          | `agent_meta.workspace_num`, then `done.workspace_num`                                                               | **add** `workspace_num`                                                                                                      |
| agent_clan, agent_clan_generation, clan_tribe, tribe, clan_context                                                                          | clan containers, `@tribe` chips (`project_clan_tree`)         | meta clan fields + scan `clan_context`                                                                              | **add** clan identity                                                                                                        |
| project_display_name                                                                                                                        | project column and group labels                               | project registry `display_name`                                                                                     | **fix** `labels.project_label`                                                                                               |
| approve, auto_approve_plan_action                                                                                                           | `⚡`/`⚡E`/`⚡T` icon                                         | `agent_meta.approve` or `done.approve`; `agent_meta.auto_approve_plan_action`                                       | **add**                                                                                                                      |
| retry_attempt                                                                                                                               | `↻N` badge and indent                                         | `agent_meta.retry_attempt`                                                                                          | **add**                                                                                                                      |
| agent_family_parallel, plan_chain_root, role_suffix                                                                                         | sequential-family lanes, plan-chain grouping, member identity | `agent_meta.*`                                                                                                      | **add** (viewer already probes these keys)                                                                                   |
| monitor/gate/proc id, state, label, gate_accent                                                                                             | lane glyph style, `⚙N` and gate settled/running counts       | `family_shell.{kind,id,state,label,accent}`, `agent_meta.proc_id`                                                   | **add** (viewer already probes these keys)                                                                                   |
| phase_bead_id                                                                                                                               | bead-linked glyph                                             | `agent_meta.phase_bead_id`                                                                                          | **add**                                                                                                                      |
| waiting_for, wait_for_beads, wait_until, wait_duration, wait_priority, wait_runners, runner-slot queue position and size, occupied capacity | WAITING/QUEUED detail text                                    | waiting marker + owner runner-slot runtime                                                                          | **deferred** (F4): queue position is owner runtime state, not a record fact; wait-target names need their own privacy review |
| retry_count, retry_next_at_epoch, fallback_model, using_fallback, monitor_exit_code, follow-up outcome/error                                | detail text for specific states                               | mixed                                                                                                               | **deferred** (F4)                                                                                                            |
| hidden                                                                                                                                      | hidden icon                                                   | `agent_meta.hidden`                                                                                                 | non-goal: the presentation scope never serves hidden records                                                                 |
| reverted                                                                                                                                    | reverted glyph                                                | local git state                                                                                                     | non-goal (owner-local)                                                                                                       |
| step_type, embedded_workflow_name, workflow-step child rows, proc_language                                                                  | workflow step rows                                            | step dirs and proc registry are not fleet records                                                                   | non-goal                                                                                                                     |
| fleet_followed                                                                                                                              | fleet badge                                                   | viewer follow state                                                                                                 | non-goal (viewer-local)                                                                                                      |
| source_machine, imported_source_owner                                                                                                       | owner badge                                                   | superseded by the host chip                                                                                         | non-goal                                                                                                                     |
| reasoning_effort, cl_name, clan_summary                                                                                                     | detail panel only                                             | meta                                                                                                                | non-goal for row parity (F4)                                                                                                 |
| artifact/workspace/output/response paths, pids, monitor_command                                                                             | none on rows                                                  | local                                                                                                               | non-goal (privacy: never serialized)                                                                                         |

If re-verification finds a row-visible fact that is missing from this table, add it
under the same rules or record it as a follow-up. Do not silently widen the scope.

## Step 2 — Contract changes (`crates/sase_core/src/fleet_contract.rs`)

### 2a. `ResolvedAgentSummaryWire` gains additive fields

Every new field uses `#[serde(default, skip_serializing_if = ...)]` (`Option::is_none`,
or a private `is_false` helper for bools). This keeps three things working: old owner
payloads still parse on new viewers, rows without a fact serialize exactly as they do
today, and existing Python fixtures and `tools/validate_sase_core_rs` dicts still
validate. Field names reuse the keys `_agent_from_summary` already probes wherever one
exists.

| Field                                                   | Type             | Source (projection)                                                                                                                                    |
| ------------------------------------------------------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `started_at_unix`                                       | `Option<f64>`    | owner fact `started_at_unix` (2b)                                                                                                                      |
| `run_started_at_unix`                                   | `Option<f64>`    | parse `agent_meta.run_started_at` as RFC 3339 (accept `Z` and `+00:00`); unparseable → `None`                                                          |
| `stopped_at_unix`                                       | `Option<f64>`    | parse `agent_meta.stopped_at` the same way                                                                                                             |
| `workspace_num`                                         | `Option<u32>`    | `agent_meta.workspace_num`, then `done.workspace_num`; negative or out of range → `None`                                                               |
| `agent_clan`                                            | `Option<String>` | `agent_meta.agent_clan`                                                                                                                                |
| `agent_clan_generation`                                 | `Option<String>` | `agent_meta.agent_clan_generation`                                                                                                                     |
| `clan_tribe`                                            | `Option<String>` | `agent_meta.clan_tribe` (this row's own declaration)                                                                                                   |
| `clan_context_tribe`                                    | `Option<String>` | owner fact `clan_context_tribe` (2b): the tribe the owner resolved for this row's clan generation, which may come from records outside the served page |
| `tribe`                                                 | `Option<String>` | `agent_meta.tribe`                                                                                                                                     |
| `approve`                                               | `bool`           | `agent_meta.approve \|\| done.approve`                                                                                                                 |
| `auto_approve_plan_action`                              | `Option<String>` | `agent_meta.auto_approve_plan_action`                                                                                                                  |
| `retry_attempt`                                         | `Option<u32>`    | `agent_meta.retry_attempt` when > 0                                                                                                                    |
| `agent_family_parallel`                                 | `bool`           | `agent_meta.agent_family_parallel`                                                                                                                     |
| `plan_chain_root`                                       | `bool`           | `agent_meta.plan_chain_root`                                                                                                                           |
| `role_suffix`                                           | `Option<String>` | `agent_meta.role_suffix`                                                                                                                               |
| `monitor_id` / `monitor_state` / `monitor_label`        | `Option<String>` | `family_shell.{id,state,label}` when `row_kind == Monitor`                                                                                             |
| `gate_id` / `gate_state` / `gate_label` / `gate_accent` | `Option<String>` | `family_shell.{id,state,label,accent}` when `row_kind == Gate`                                                                                         |
| `proc_id` / `proc_status` / `proc_label`                | `Option<String>` | `agent_meta.proc_id`, plus `family_shell.{state,label}` when `row_kind == Proc`. If a record has no proc status source, leave it `None` and note that  |
| `phase_bead_id`                                         | `Option<String>` | `agent_meta.phase_bead_id`                                                                                                                             |

Get `family_shell` the same way `project_resolved_agent_summary` already does (meta
first, then done). Trim every string with `trim_to_limit(.., MAX_LABEL_BYTES)`; an empty
string becomes `None`.

`labels.project_label` changes meaning: it is now the owner's human project display name
(owner fact `project_label`), falling back to the trimmed record `project_name`.
`project_name` and `logical_locator.project.project_id` stay the portable project id, so
`project_ids` catalog filters, `machine:`/`project:` queries, and cross-machine grouping
are unchanged. `summary_matches_catalog_query` already searches both strings, so text
queries match either the id or the human name. Update the struct doc comment to say
this.

### 2b. `OwnerResolutionFactsWire` gains owner-only facts

Add these fields, each with
`#[serde(default, skip_serializing_if = "Option::is_none")]`:

- `started_at_unix: Option<f64>`
- `project_label: Option<String>`
- `clan_context_tribe: Option<String>`

The pure layer cannot produce these: it has no clock or timezone (`chrono` is built
without the `clock` feature in `sase_core`), no project registry, and no scan clan
context. `normalized_owner_facts` validates them: timestamps must be finite and
non-negative, and labels go through `validate_label` + `reject_secretish`.

### 2c. Validation (`validate_resolved_agent_summary`)

- New timestamps: finite and non-negative (reuse `validate_timestamp`). Do not enforce
  ordering between them.
- Every new string: `validate_label(field, value, MAX_LABEL_BYTES)` +
  `reject_secretish`. Do not use the stricter `validate_identifier`, because real clan
  and bead names contain dots.
- `retry_attempt`, when present, must be > 0.
- `agent_clan_generation`, `clan_tribe`, and `clan_context_tribe` require `agent_clan`.
- Shell facts must match the row kind: any `monitor_*` field requires
  `row_kind == Monitor`, any `gate_*` requires `Gate`, and any `proc_*` requires `Proc`.
  Otherwise reject with a "… inconsistent with row_kind" error, like the existing
  family-role check.

### 2d. Schema version

Keep `FLEET_CONTRACT_SCHEMA_VERSION = 1`. Every fleet wire (enroll, hello, catalog,
mutations) checks this version for equality, so bumping it would cut off every peer on a
different version. The precedents `270e501` (family_role/parent_timestamp) and `8e491c3`
(queue weight) added summary fields under v1. For this phase, "bump the contract schema"
means: regenerate the committed gateway contract snapshot (Step 4), update the PyO3
binding coverage (Step 5), and use a `feat` commit so release-plz publishes a new
version. Record the skew consequence in the phase note: until both sides upgrade, an old
viewer will still reject (via `deny_unknown_fields`) any page from a new owner that
contains one of these fields. That was also true for the earlier additions, and
`live-acceptance` requires both machines on new builds.

## Step 3 — Owner facts in the gateway (`crates/sase_gateway/src/fleet_reads.rs`)

In `build_snapshot_blocking`:

1. **Project labels.** Read the project registry once per build with
   `list_project_records(&request.projects_root, &[], true, false)` (all states), and
   map each `project_name` to `display_name.unwrap_or(project_name)` after trimming. If
   the registry read fails, use an empty map; the snapshot must not fail, and labels
   fall back to the raw `project_name` exactly as today. Snapshot builds are already
   cached and coalesced, so this adds one registry read per rebuild and nothing to the
   request hot path.
2. **Clan context.** Take `scan.clan_context` _before_ the `for record in scan.records`
   loop consumes `scan`. Build a map from `(agent_clan, agent_clan_generation)` to
   `clan_tribe`, and pass the matching tribe into `resolve_record` for records whose
   meta clan key matches.
3. **Launch time.** Add a helper that parses the 14-digit `record.timestamp`
   (`%Y%m%d%H%M%S`) as owner-local wall time with `chrono::Local` (the gateway crate
   enables `clock`): take `from_local_datetime(..).earliest()`, fall back to `.latest()`
   for DST folds, and return `None` for gaps or bad input. `chrono::Local` honors `TZ`
   and `/etc/localtime`, the same resolution order as SASE's Python `system_timezone()`,
   which writes these timestamps when no `timezone` config key is set. Record the
   config-key mismatch as follow-up F3.
4. Extend `resolve_record` to accept these facts and set them on
   `OwnerResolutionFactsWire`. Do **not** change `parse_record_timestamp` or
   `completion_time_for_record` (presentation-window policy) in this phase; see F2.

## Step 4 — Gateway contract manifest

In `crates/sase_gateway/src/contract.rs` (the `ResolvedAgentSummaryWire` description
around the fleet contract snapshot builder, plus `OwnerResolutionFactsWire` if it is
described), add entries such as:

- `timestamps`: optional `started_at_unix` (owner launch time in the owner's zone),
  `run_started_at_unix`, `stopped_at_unix`
- `workspace`: optional `workspace_num`
- `clan`: optional `agent_clan`, `agent_clan_generation`, `clan_tribe`,
  `clan_context_tribe`, `tribe`
- `project_label`: `labels.project_label` is the owner's human project name, while
  `project_name` and the locator `project_id` stay portable
- `presentation`: `approve`, `auto_approve_plan_action`, `retry_attempt`,
  `agent_family_parallel`, `plan_chain_root`, `role_suffix`, `phase_bead_id`
- `shell`: `monitor_*`, `gate_*`, and `proc_*` facts, present only on the matching
  `row_kind`

Regenerate `crates/sase_gateway/contracts/api_fleet_v1/fleet_api_v1.json` with
`UPDATE_FLEET_CONTRACT=1 cargo test -p sase_gateway committed_fleet_contract_snapshot_is_current`,
inspect the diff, and confirm the test passes without the env var.

## Step 5 — Tests and fixtures

**`fleet_contract.rs` tests.** Extend the `projection_request` and `summary_done`
helpers so all new fields have explicit values, and update every struct literal that
constructs `ResolvedAgentSummaryWire` or `OwnerResolutionFactsWire` (including
`fleet_mutation.rs` tests). Add:

1. Projection carries `run_started_at_unix`/`stopped_at_unix` from RFC 3339 meta strings
   (`+00:00` with fractional seconds, and `Z`), `started_at_unix` from owner facts, and
   `workspace_num` with the done-marker fallback. Malformed ISO and a negative workspace
   both produce `None`.
2. `labels.project_label` uses the owner fact, while `project_name` and the locator
   `project_id` keep the raw id. A catalog text query matches both strings, and the
   `project_ids` filter still keys on the id.
3. Clan identity and `clan_context_tribe` are carried. The validator rejects clan
   sub-fields that have no `agent_clan`.
4. Presentation flags are carried: `approve` from either the meta or done marker,
   `auto_approve_plan_action`, `retry_attempt` (0 → `None`), parallel, plan-chain root,
   role suffix, and phase bead id.
5. Shell facts appear only on the matching row kind, and the validator rejects
   `monitor_state` on an `agent_shell` summary.
6. A summary JSON in today's serialized shape (none of the new keys) still deserializes
   and validates, and a projected row with no facts serializes without any new key.
7. The validator rejects non-finite or negative timestamps, `retry_attempt: 0`, and
   secret-looking labels.

**`fleet_reads.rs` tests.** Use the existing `seed_home`/`seed_project`/`seed_agent`
helpers to write real artifacts:

1. Seed a project spec whose display name differs from its key. Follow the
   `project_name_parse.display_name` source in `project_spec.rs::build_project_record`.
   Seed an agent whose meta has `run_started_at`, `stopped_at`, `workspace_num`, clan
   fields, `approve`, and `retry_attempt`. Assert the served catalog summary carries
   them, that `labels.project_label` is the human name, and that `started_at_unix`
   equals the `chrono::Local` conversion of the seeded timestamp (compute the expected
   value the same way; do not mutate `TZ` in tests).
2. A projects root without a readable spec still serves rows, and the label falls back
   to `project_name`.
3. A clan generation whose tribe declaration comes from a different seeded record still
   yields `clan_context_tribe` on the member row.
4. `assert_no_paths_or_pids` still passes on the new payload.

**PyO3 (`crates/sase_core_py/src/lib.rs`).** No new functions are needed, because the
dict-in/dict-out projection, validation, and federation-normalize bindings serialize the
new fields automatically. Extend the existing `fleet_project_resolved_agent_summary`
binding test (around the `owner_facts` request dict) with the new owner facts and meta
fields, and assert the new keys in the returned dict. Also add a validate-call on a
legacy summary dict that has none of the new keys.

## Step 6 — Verification

- Run the complete core gate through `/sase_monitor`: `just check` (which runs
  `scripts/check.sh all`: fmt-check, clippy `-D warnings`, `cargo test --workspace`
  including `sase_core_py`). Never fall back to `cargo test -p sase_core` alone.
- `rg` for every remaining struct literal of the two changed wires, so nothing depends
  on `..Default` hiding a field.
- `git diff` review: no paths, pids, or raw commands serialized, and no version edits.

## Step 7 — Phase note, follow-ups, close

1. Record the audit table (Step 1) as a note on `sase-xe.16.11.7.15.3`.
2. Record the exact committed surface for `published-core-adoption`: every new
   `ResolvedAgentSummaryWire` and `OwnerResolutionFactsWire` field with its type, serde
   attributes, source, and validation rule; the `labels.project_label` semantics; the
   unchanged `FLEET_CONTRACT_SCHEMA_VERSION = 1`; the binding names that carry the
   fields (`fleet_project_resolved_agent_summary`,
   `fleet_validate_resolved_agent_summary`, `fleet_normalize_federation_response`); a
   suggested wheel probe for `tools/validate_sase_core_rs` (project a request with
   `owner_facts.started_at_unix` and assert `started_at_unix` in the summary); and the
   skew note from 2d.
3. Record these follow-ups:
   - **F1**
     `PROPOSED FOLLOW-UP: sase repo open cannot open linked repos from numbered workspaces — "Unknown repo 'sase-core' for project 'gh_sase-org__sase'"; host_ctx.project_name is the project key while repo inventory records carry project "sase" (filter in src/sase/main/repo_open_external.py _unknown_repo_error and the lookup it guards); agents fall back to gh:sase-org/sase-core external opens.`
   - **F2**
     `PROPOSED FOLLOW-UP: owner-local 14-digit artifact timestamps are parsed as UTC in sase_gateway fleet_reads parse_record_timestamp/completion_time_for_record and sase_core agent_stats parse_artifact_timestamp — presentation windows and stats are skewed by the owner's UTC offset.`
   - **F3**
     `PROPOSED FOLLOW-UP: federation worker and gateway launchers do not export the configured sase timezone as TZ, so chrono::Local-derived started_at_unix drifts when config "timezone" differs from the system zone.`
   - **F4**
     `PROPOSED FOLLOW-UP: remote WAITING/QUEUED detail facts (wait targets, wait_until/duration/priority, runner-slot queue position) and detail-panel facts (reasoning_effort, clan_summary, retry_count, fallback model) are not on the fleet wire — decide privacy and an owner-runtime source before adding them.`
   - Add anything else you discover.
4. Run `sase bead epic-symbols sase-xe.16.11.7.15.3` (it was empty at planning time).
   Resolve or re-key any entries it lists.
5. `sase bead close sase-xe.16.11.7.15.3 --note "<what was verified: core just check green incl. PyO3, contract snapshot regenerated, tests added>"`.
   Do not close `sase-xe.16.11.7.15` or any ancestor, and do not touch the sibling
   phases.

## Acceptance

- A remote owner's served summary carries real `started_at_unix`/`run_started_at_unix`/
  `stopped_at_unix`, `workspace_num`, clan identity including the owner-resolved clan
  tribe, the human `labels.project_label` (for example `sase` rather than
  `gh_sase-org__sase`) with the portable project id unchanged, and the row-visible
  presentation and shell facts from the audit table.
- Summaries from older owners and older fixtures still parse and validate.
- The gateway contract snapshot test passes against the regenerated manifest.
- Core `just check` is green, including the PyO3 binding tests.
- The phase note records the audit table, the exact surface, the skew note, and
  follow-ups F1–F4.
