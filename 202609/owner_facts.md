---
tier: tale
title: Resolve production family presentation facts from real owner records
goal:
  The production fixture's owner loader and remote catalog produce the same normalized
  rendered Agents rows — rich status, nested shells, tribes, chips, runtimes, and
  done-inclusive banners — except the machine chip and preserved remote affordances,
  with no viewer filesystem access required.
size: medium
proposed_by: bbugyi200.athena.sase-133.5.2
bead: sase-133.5.2
create_time: 2026-09-19 12:13:25
status: wip
---

- **PARENT:**
  [202609/remote_parity_landing_repairs.md](https://github.com/sase-org/sase--plans/blob/main/202609/remote_parity_landing_repairs.md)
- **BEAD:**
  [sase-133.5.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-133/sase-133.5.2.md)

# Resolve production family presentation facts

This tale implements phase **owner-facts** (`sase-133.5.2`) of
`plan:202609/remote_parity_landing_repairs.md`. Phase **owner-roster** (`sase-133.5.1`)
already shares inclusion/retirement and serves dead members of presented families. The
remaining failure is presentation: topology, rich status, tribe inheritance, shell/chip
facts, and timestamps are still coarse or missing on the production path.

Do not close `sase-133.5`, `sase-133`, or any other ancestor. Live apollo/athena
captures and deployment belong to `sase-133.5.4`. Version-skew diagnostics are already
closed in `sase-133.5.3`.

## Evidence already in hand

Read these through `sase artifact read` before coding; they are the production failures
this phase must regress:

- Child epic: `plan:202609/remote_parity_landing_repairs.md`
- Original epic: `plan:202609/remote_dispatch_agents_tab_parity.md`
- Live catalog (schema v4): `file:explicit:d1d475c9a14db30b47241095`

Confirmed production gaps after roster parity:

1. **`0k--plan` / `0n--plan` arrive as unparented `DONE` gate rows** with `family_id`
   set (`0k` / `0n`), `parent_timestamp: null`, `family_role: gate`,
   `tribe`/`clan_tribe` null, and `connection_health: offline` because process liveness
   is `dead`. Apollo itself renders those families as `EPIC CREATED √ ×7` and
   `TALE DONE ×7`. A real family root and its concrete `--plan` shell are different
   objects. Viewer `_materialize_missing_family_containers` only groups members that
   already have an unresolved `parent_timestamp`, so a top-level gate with
   `parent_timestamp: null` never becomes a family container.
2. **`display_status_for_record` in `fleet_catalog.rs` is coarse** (`RUNNING` /
   `WAITING` / `QUESTION` / `DONE` / `FAILED` / `STOPPED`). It does not run the owner's
   plan/monitor/gate/family pipeline (`apply_status_overrides`, gate/monitor enrichment,
   plan-action handoff).
3. **`status_bucket` is derived from lifecycle + process liveness**, not from the rich
   display status, so `TALE DONE` would still bucket wrong even if the label were fixed.
4. **Process `Dead` is mapped to `connection_health: Offline`** in both
   `assemble_fleet_catalog` and gateway `fleet_reads.rs`. Terminal process liveness is
   not transport-offline health.
5. **Chip and wait facts are absent from `ResolvedAgentSummaryWire`.** The index already
   stores `family_shell.{kind,state,id,start_status,stop_status,label}`, `plan_action`,
   `plan_chain_root`, and wait vectors on `AgentMetaWire`. The viewer currently infers
   monitor/gate state from dead/stop_time.

The hand-written test `tests/ace/tui/test_fleet_agents_display_parity.py` remains a
useful unit fixture. It is **not** the production oracle. Keep it. The production oracle
is `tests/ace/tui/test_owner_roster_oracle.py`, which today compares only identity
signatures.

## Architecture

Open `sase-core` with `/sase_repo` before any Rust edit:

```bash
sase repo open sase-core -r "Implement owner-facts topology, status, and wire projection"
```

Shared selection and presentation decisions belong in `crates/sase_core`. The gateway
stays an observer/cache wrapper. Python keeps Textual rendering, grouping, folding, and
layout. Do not re-derive remote facts by reading the viewer's filesystem.

Before touching TUI rendering paths, read:

```bash
sase memory read tui.md tui_screenshot.md tui_perf.md --reason "Need TUI rendering, screenshot, and perf constraints before owner-facts viewer consumption"
```

### 1. Resolve facts in shared core, once

Add a dedicated module (preferred name `fleet_owner_facts.rs`) rather than growing
`display_status_for_record` in place. `PresentationContext` today only indexes
**served** records and treats “no parent_timestamp and not a concrete shell” as a family
root. That is the 0n/0k bug: a `--plan` gate with `family_id` and a null parent is
classified as a nested gate by `fleet_family.rs` but never linked to a distinct root,
and no container row is emitted.

The resolver takes the served roster from phase 1 plus the full scanned record set
needed for family linkage (already available to `assemble_fleet_catalog`) and produces
per-row `PresentationRecordFacts` **and**, when required, extra projected summaries that
are not 1:1 with a disk record.

**Topology (do this first; status depends on it):**

- Keep `sase_core::fleet_family` as the shell classifier. A concrete `--plan` gate with
  `family_id` and a null parent is a nested shell, never a family root.
- Group by `family_id_for_record` / `family_key_for_record`.
- When a presented family has a real non-shell root record, set each shell's
  `parent_timestamp` to that root's record timestamp even if the shell omitted it.
- When a presented family has **only** concrete shells (the 0n/0k shape), emit a
  distinct `row_kind = container_header` / `family_role = root` summary whose stable
  identity is the family id, and point every shell at that container. Do **not**
  collapse the concrete `--plan` shell into the container.
  `container_projected_concrete_agent` stays false unless the owner actually presents
  the container as the actionable shell. Preserve the shell's action/content identity.
- Deduplicate: never emit both a real root and a synthetic container for the same
  family. Never emit two containers for one family_id.
- Trace the owner loader (`load_tiered_agents` + `_apply_status_overrides` +
  family-container attachment in `sort_and_reorder`) against the expanded fixture so the
  catalog identity for the container matches the owner identity (label `0n` / `0k` /
  `lane`, not `0n--plan`).

**Inheritance:**

- `tribe`: record meta, else family root, else clan tribe.
- `clan_tribe`: record meta, else clan-context map already built by
  `PresentationContext`, else inherited from the root.
- `started_at_unix`: family-root start (family clock).
- `run_started_at_unix`: this shell/run start.
- `stopped_at_unix`: this record's completion.

**Rich status — extract owner rules, do not copy a label list:**

Port the decision tree, not a hardcoded map of desired strings. The source of truth is
the current owner pipeline:

- `src/sase/ace/tui/models/_agent_status_apply.py`
- `src/sase/ace/tui/models/_agent_status_family_policy.py`
- `src/sase/ace/tui/models/_loaders/_meta_enrichment_gate.py`
- `src/sase/ace/tui/models/_loaders/_meta_enrichment_status.py`
- `src/sase/agent/status_buckets.py` (`status_bucket_for_values`)

Cover, with fixture records and Rust unit tests for each:

| Owner situation                           | Expected rich status / bucket                                           |
| ----------------------------------------- | ----------------------------------------------------------------------- |
| Active monitor shell                      | monitor `start_status` (e.g. `TESTING`), Running                        |
| Settled monitor shell                     | monitor `stop_status` / completed pair, Done                            |
| Pending gate (including dead creator PID) | gate `start_status` (`PLAN` / `TALE` / `EPIC` / `QUESTION`), Stopped    |
| Settled gate                              | gate `stop_status` (`TALE DONE` / `EPIC CREATED` / `PLAN APPROVED` / …) |
| Active approved-tale coder                | `WORKING TALE`, Running                                                 |
| Completed tale code handoff               | `TALE DONE`, Done                                                       |
| Approved epic follow-up                   | `EPIC CREATED`, Done                                                    |
| Question / answered handoff               | `QUESTION` / `ANSWERED`                                                 |
| Waiting with deps                         | `WAITING` plus wait-count facts for `▶N`                                |
| Stopped / failed                          | `STOPPED` / `FAILED`                                                    |

Family roots **mirror** the active, waiting, or newest concrete child/shell using the
same policy as `apply_status_overrides` (live child first, then waiting, then newest
shell for plan-workflow roots; pending gate shells remain visible and can be the
mirrored status).

`status_bucket` on the wire must follow the rich display status through the extracted
bucket rules, not `bucket_for_lifecycle(lifecycle, liveness)`.

Derive from index metadata first (`done.status_label`, `family_shell.*`, `plan_action`,
`plan_approved`, `plan_chain_root`, waiting/question markers, role/suffix). Persist a
bounded fact through lifecycle/index mutation hooks **only** when a real production
record cannot reconstruct the owner status without a filesystem read the gateway must
not perform. If a hook is required, document which fact and why in the phase close note.

List-shape (`agents_list_projection`) already keeps `agent_meta` aside from heavy
leaves. Do not force a Full scan on every refresh to obtain these facts.

**Shell / chip facts** to carry on each summary (optional, additive):

- monitor: id, state, start_status, stop_status, label
- gate: id, kind, state, start_status, stop_status, label, accent
- proc: id, status, label
- `plan_action`, `plan_chain_root`
- bounded wait facts already needed for `WAITING ▶N` (counts/ids that the owner row
  shows; no local paths)

**Connection health:** `Alive` → `Online`. `Dead` / `NotProcess` / `Unknown` must
**not** become `Offline`. Absent a real transport observation, use `Unknown`. Viewer
`WAS RUNNING` / feed-staleness markers stay remote affordances; do not invent them from
PID death.

**Revisions:** `stable_revision` currently hashes the raw record. Include the resolved
presentation facts in the revision input so a status, parent link, tribe, or chip change
invalidates snapshot caches.

### 2. Wire, gateway, and bindings

`FLEET_CONTRACT_SCHEMA_VERSION` is **4**. Bump additively to **5** if any new
summary/owner-facts field is required. Keep `#[serde(default)]` on new fields. Older
payloads remain readable; omitted facts degrade to today's rendering without exceptions.
Current hosts emit complete facts.

Update every constructor of `OwnerResolutionFactsWire` / `ResolvedAgentSummaryWire`,
including:

- `crates/sase_core/src/fleet_catalog.rs` (`project_summary_for_record`)
- `crates/sase_gateway/src/fleet_reads.rs` (`resolve_record` owner_facts)
- `crates/sase_core/src/fleet_contract.rs` tests
- `crates/sase_core_py` only if it hand-builds these structs

Do not fork the facts builder: gateway and `assemble_fleet_catalog` must call the same
`fleet_owner_facts` API. `sase_core_py::assemble_fleet_catalog` already returns serde
JSON; new fields flow through if the Rust summaries carry them.

`sase-133.5.3` already stopped comparing capability schema to fleet data schema. A bump
to fleet schema 5 is expected; do not reintroduce that false warning.

### 3. Python adapters and the existing renderer

Consume the new facts in `src/sase/ace/tui/models/_fleet_agents_rows.py`. Populate
`Agent` the same way the owner loader would: `status`, `status_bucket`, family
role/parent, tribe/clan_tribe, gate/monitor/proc fields, plan_action, timestamps.

Keep using:

- `normalize_remote_host_nodes` / grouping / folding
- `format_agent_option` / `compute_fold_annotation` (`×N`)
- `agent_panel_counts` (done-inclusive banners)
- `apply_status_overrides` on the remote path **only if** the Agent fields it needs are
  present so it cannot clobber an owner-resolved `TALE DONE` back to `DONE`. If a remote
  row already carries a complete owner `display_status`, the override pass must be a
  no-op or an identical re-derivation.

Do **not** add a remote-only renderer. Do not treat `connection_health: offline` as
transport failure when the host feed is ok. Preserve the machine chip, feed-staleness
markers, `WAS RUNNING`, and capability-gated actions.

Viewer reconstruction of remote facts must not open owner artifact directories.

### 4. Production fixture and rendered-row oracle

Extend `tests/ace/tui/owner_roster_fixture.py` (one shared on-disk lifecycle, not a
second synthetic oracle) so it includes:

- The existing pending dead-creator gate, settled monitor, active coder, done family,
  dismissed identity, recycled PID, and fresh launch
- A **0n-like completed tale** whose durable records are a `--plan` gate plus nested
  shells, owner-rendered as `TALE DONE ×N` under family id `0n` (or the fixture's
  equivalent label)
- A **0k-like completed epic** owner-rendered as `EPIC CREATED` with a check glyph and
  nested shells, not as a top-level `0k--plan` `DONE` gate
- An **active/waiting family** with rich status, monitor/gate chips, and a wait-count if
  the owner shows `▶N`
- Direct and inherited tribe/clan tribe on at least one done family

Extend `tests/ace/tui/test_owner_roster_oracle.py` (or a sibling in the same module) to
compare **normalized rendered rows**, not just identity signatures:

1. Owner: `load_tiered_agents` → fold → `format_agent_option` / panel titles / runtimes
   at a fixed `now`
2. Remote: `assemble_fleet_catalog` (current **and** compact-index paths) → wrap as a
   catalog response → `project_fleet_agents` → same renderer
3. Assert independently: visible-node count, panel counts including Done, nested-shell
   count, rich status text, `×N`, monitor/gate/proc chips, runtime columns, and
   machine-filter counts
4. Strip only the remote machine chip and explicitly preserved remote affordances before
   comparing

Add focused regression names for the 0n and 0k shapes so a future coarse `DONE` gate
cannot slip through as an identity-only match.

Keep `test_fleet_agents_display_parity.py`. Add Rust tests next to the new module for
topology, each status case, Dead≠Offline, revision changes on fact updates, and
older-payload omission.

## Implementation order

1. Open `sase-core`. Expand the production fixture until the owner loader reproduces
   0n/0k/active/waiting rendering locally. Do not invent catalog expected rows by hand.
2. Implement topology + inheritance + timestamps in `fleet_owner_facts`. Prove parent
   links and container-vs-shell identities against the fixture **before** status work.
3. Extract status + bucket + chip facts. Drive them through `project_summary_for_record`
   and gateway `resolve_record`.
4. Fix connection health. Include facts in row revisions. Bump schema if new fields
   landed.
5. Carry fields through Python adapters. Confirm `format_agent_option` is the only
   renderer under test.
6. Grow the production oracle to rendered-row parity on both catalog paths.
7. `just rust-fmt` / `just rust-test` in the opened sase-core checkout, then
   `just rust-install` so Python tests see the new extension. In the primary repo:
   `just fix`, then the focused oracle/display tests, then `just check`. If TUI goldens
   change, run `just fix-tui-screenshots` and inspect the visual report; include
   generated goldens in the host commit. `just check-full` is required before landing
   the parent epic, not before closing this phase, unless scoped `just check` escalates.

## Acceptance

- Production fixture owner vs remote normalized rendered rows match except the machine
  chip and preserved remote affordances.
- `0n`/`0k` families render as family containers with `TALE DONE` / `EPIC CREATED`,
  nested shells, and `×N` — not as top-level `--plan` `DONE` gates.
- Active and waiting families keep rich status, chips, and wait details.
- No viewer filesystem access is required to reconstruct remote facts.
- Dead process liveness does not mark the row transport-offline.
- Current and compact-index catalog paths both satisfy the oracle.
- This phase does not close `sase-133.5` or `sase-133`.
