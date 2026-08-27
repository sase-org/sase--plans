---
tier: tale
size: medium
title: Project the heavy record_json leaves off the list-render path
goal:
  A Tier 1 ACE agent load asks the artifact index for a list-shaped record that omits
  ~48% of the record_json payload, the omitted leaves are a typed `not loaded` state
  that the detail panel hydrates on demand instead of silently rendering as empty, and
  the abandoned-row repair scan stops full-scanning a 138 MB text column.
proposed_by: bbugyi200.athena.sase-uv.7
bead: sase-uv.7
create_time: 2026-08-27 14:49:59
status: wip
---

- **PARENT:** [202608/ace_tui_responsiveness.md](ace_tui_responsiveness.md)
- **BEAD:** sase-uv.7

# Plan: Project the heavy `record_json` leaves off the list-render path

## Scope and provenance

This is the `projection` phase of epic `sase-uv` (Restore ACE TUI responsiveness), bead
`sase-uv.7`. The epic plan is `plan:202608/ace_tui_responsiveness.md`; read its
`Correction 3` section and its `Phase projection` section before starting.

The work crosses the Rust core boundary. The Rust changes go in the `sase-core` linked
repo; the Python changes go in this repo. Open the core repo with the `/sase_repo` skill
(`sase repo open sase-core -r "<why>"`) and use only the path it prints. Do not clone or
web-fetch it any other way.

## Re-measurement: the epic plan's leaf table is stale, and two of its four leaves cannot be omitted

The epic plan says to omit four whole leaves — `done.step_output`, `prompt_steps`,
`agent_meta.linked_repos`, `workflow_state.steps` — quoted at 77.7% of the payload. Both
halves of that were re-checked against the live index (195 MB, 9,283 rows) before this
plan was written, and both need correcting. The corrections are load-bearing; follow
this plan, not the epic plan's letter, where they disagree.

**Correction A — the share is 61.6%, not 77.7%.** Measured over a selection shaped like
the TUI's actual Tier 1 query (1,000 visible active rows + 200 visible recent-completed
rows, 15.74 MB total):

| Leaf                       |     bytes | share of payload |
| -------------------------- | --------: | ---------------: |
| `prompt_steps` (whole)     | 2,989,757 |           19.00% |
| `agent_meta.linked_repos`  | 2,492,905 |           15.84% |
| `done.step_output` (whole) | 2,451,484 |           15.58% |
| `workflow_state.steps`     | 1,755,202 |           11.15% |
| **four leaves combined**   |           |       **61.56%** |

**Correction B — two of the four leaves produce list rows.** They are not detail-only
data:

- `record.prompt_steps` is iterated by `_build_workflow_agent_steps_for_record`
  (`src/sase/ace/tui/models/_loaders/_workflow_snapshot_loaders.py:301`), and **each
  element becomes an `Agent` row in the list**. Omitting the list deletes workflow-step
  rows from the Agents tab.
- `workflow_state.steps` is iterated by `load_workflow_states_from_snapshot` (same file,
  lines 84/112/120) to compute the entry's `total_steps`, its `diff_path`, and its
  FAILED-status error message, and by `load_workflow_agents_from_snapshot` (line 203) to
  derive the parent row's `step_output` and therefore its `workspace_num`. Omitting it
  changes visible list state.

So a whole-leaf projection of those two is not "a detail field renders empty"; it is
"rows disappear and statuses change". Do not do it.

**What is actually large is one sub-leaf: `_raw`.** `_raw` is the agent's full raw
response text (`src/sase/xprompt/workflow_executor_steps_prompt.py:470`), and inside the
ACE TUI it is read only by the prompt panel's display path — through `format_output()`
(`widgets/prompt_panel/_helpers.py:57`, which unwraps `_data`/`_raw`) and the two
`"_raw" in ... or "_data" in ...` presence checks. Nothing on the load or list-render
path reads it. Measured on the same TUI-shaped selection:

| Projected leaf                        |     bytes | share of payload |
| ------------------------------------- | --------: | ---------------: |
| `agent_meta.linked_repos`             | 2,492,035 |           15.83% |
| `done.step_output._raw`               | 1,922,890 |           12.22% |
| `workflow_state.steps[*].output._raw` | 1,580,865 |           10.04% |
| `prompt_steps[*].output._raw`         | 1,572,015 |            9.99% |
| **projection total**                  | 7,567,805 |       **48.08%** |

`_data` did not occur in any sampled row, but project it alongside `_raw` anyway:
`format_output` treats the two interchangeably, so a future writer that emits `_data`
instead must not silently escape the projection.

**48.1% of the payload for zero list-row loss** is the trade this plan takes, against
61.6% that would require hydrating row-producing structure. Re-measure on the
implementing host and record the number on the bead — the corpus moves.

## What this projection does and does not save

The GIL is the reason this is worth doing, and it also bounds the win. In
`py_query_agent_artifact_index` (`crates/sase_core_py/src/lib.rs`) the SQL read and the
`serde_json::from_str` decode already run inside `py.allow_threads(...)`; only
`serialize_to_py` (and, back in Python, `agent_scan_wire_from_dict`) holds the GIL.
Projecting the record **after** it is decoded therefore removes ~48% of the GIL-held
marshalling and ~48% of the Python dataclass construction, which is exactly the work
that stalls the UI, while leaving the off-GIL parse untouched.

Do **not** try to make the parse cheaper as part of this phase. Two tempting variants
were considered and rejected:

- _Deserialize with `IgnoredAny` on the heavy fields._ Needs either a duplicate mirror
  of `AgentArtifactRecordWire` (which drifts — that struct has taken 20+ migrations) or
  a thread-local projection flag threaded through `serde` (spooky action at a distance
  on any other `from_str` on the same thread).
- _Split the heavy leaves into a second `record_detail_json` column at upsert time._
  This is the strictly better fix and would also shrink the read and the parse, but it
  costs a full-table schema migration and touches every `record_json` reader in
  `index.rs`. Its extra win is off-GIL, so it is worth less than it looks. Record it as
  a `PROPOSED FOLLOW-UP:` note on the bead rather than doing it here.

Budget expectation, per the epic plan: do **not** promise a large fraction off
`load_tiered_agents`. Measure and report what you actually get.

## Design

### The projection is opt-in on the query wire, and visible on the record

Two new wire values, both defaulting to the current behavior so every existing caller is
byte-for-byte unchanged:

```rust
#[derive(Debug, Clone, Copy, Default, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum AgentArtifactRecordShapeWire {
    #[default]
    Full,
    List,
}
```

- `AgentArtifactIndexQueryWire` gains
  `#[serde(default)] pub record_shape: AgentArtifactRecordShapeWire` (default `Full`).
- `AgentArtifactRecordWire` gains
  `#[serde(default, skip_serializing_if = "<is-full predicate>")] pub record_shape: AgentArtifactRecordShapeWire`.

`skip_serializing_if` is load-bearing, not cosmetic. `upsert_record` stores
`serde_json::to_string(record)` into `record_json`, and skipping the default keeps
stored bytes identical, so **no index schema migration and no
`migrate_record_json_refresh_v*` is needed for the projection itself**. It also keeps
the agent-scan golden corpus (`tests/agent_scan_golden/`,
`tests/test_core_agent_scan_*.py`) unchanged for full-shape output.

Do **not** bump `AGENT_SCAN_WIRE_SCHEMA_VERSION` (currently 7 in both
`crates/sase_core/src/agent_scan/wire.rs` and
`src/sase/core/agent_scan_wire_records.py`). The addition is additive and opt-in;
`known_field_kwargs` already tolerates unknown keys in the other direction. Add a test
that a default (`Full`) query's payload contains no `record_shape` key at all.

### Why a record-level enum and not a per-map marker

The epic plan requires the omission be "explicit in the type, not by convention". A
record-level `record_shape` enum satisfies that. A marker key injected into the output
map (e.g. `{"_omitted": ["_raw"]}`) does not: it is a convention, and
`_agent_clan_sections.py:384` iterates _every_ key of `step_output` and renders it, so
the marker would paint itself into the UI.

### Hydration

One new core function and binding, because there is no by-artifact-dir record query
today:

```rust
pub fn load_agent_artifact_records(
    index_path: &Path,
    artifact_dirs: &[String],
) -> Result<Vec<AgentArtifactRecordWire>, String>
```

- Opens read-only (`open_index_read_only`), resolves each input through
  `resolve_index_artifact_dir` so alias paths work, chunks the
  `SELECT record_json FROM agent_artifacts WHERE artifact_dir IN (...)` the way
  `delete`/`repair` already chunk their `IN` lists, and returns full (unprojected)
  records. Unknown dirs are skipped, not an error.
- Exposed as `load_agent_artifact_records` from `crates/sase_core_py`, and as
  `sase.core.agent_scan_facade.load_agent_artifact_records()` returning
  `list[AgentArtifactRecordWire]` under the existing
  `agent_artifact_index_operation_lock()`.

## Implementation

### Part 1 — `sase-core` (open with `/sase_repo`)

All paths below are relative to the core checkout.

1. `crates/sase_core/src/agent_scan/wire.rs`
   - Add `AgentArtifactRecordShapeWire` and the `record_shape` field on
     `AgentArtifactRecordWire` exactly as above. Export it from `agent_scan/mod.rs` and
     `lib.rs` alongside the other record wire types.

2. `crates/sase_core/src/agent_scan/index.rs`
   - Add `record_shape` to `AgentArtifactIndexQueryWire` and to its `Default` impl
     (`Full`).
   - Add a private `project_record_for_list(record: &mut AgentArtifactRecordWire)` that:
     - clears `agent_meta.linked_repos` (when `agent_meta` is `Some`),
     - removes the `_raw` and `_data` keys from `done.step_output`, from every
       `prompt_steps[*].output`, and from every `workflow_state.steps[*].output`,
     - sets `record.record_shape = List`. Removing the keys rather than nulling the
       whole map is deliberate: the `meta_*` keys in those maps are read on the load
       path (`_done_loaders.py`, `_meta_enrichment_prompt_markers.py`,
       `_agent_status_apply.py`) and must survive.
   - Apply it in `query_agent_artifact_index` **after** `by_dir` has been collected into
     `records` and before `select_clan_context`, gated on `query.record_shape == List`.
     Applying at the end (rather than inside `select_records`) keeps it provably
     selection-neutral: `record_matches_selection`, the project filter, and the
     revalidate/upsert path all still see full records.
   - Add `load_agent_artifact_records` as described under **Hydration**.

3. `crates/sase_core/src/agent_scan/index.rs` — abandoned-row repair (epic plan item 4).
   `repair_abandoned_agent_artifact_index_rows` (~line 704) currently ORs three
   `record_json LIKE` predicates, forcing a full scan of the 138 MB text column to match
   ~36 rows. It runs from `terminalize_stale_active_agent_artifact_index_rows`, which is
   background index maintenance, not a hot query path — fix it for the scan cost, and do
   not claim it as a UI speed win.
   - Add a `done_outcome TEXT` column via the existing `ensure_agent_artifacts_column`
     helper, populate it in `upsert_record` from `record.done.outcome`, and index it
     (`CREATE INDEX IF NOT EXISTS idx_agent_artifacts_done_outcome ON agent_artifacts(done_outcome)`).
   - Bump `AGENT_ARTIFACT_INDEX_SCHEMA_VERSION` 23 → 24 and add a v24 arm to the
     migration ladder in `open_index_with_busy_timeout` that ensures the column and
     backfills it. Model the backfill on `migrate_model_alias_projection_v22`, which
     already does a full-table decode-and-update pass. Note the interaction with
     `open_index_read_only`: it falls back to the migrating read-write open whenever the
     stored version is not current, so the first open after this lands migrates and
     later read-only opens are clean.
   - Replace **all three** LIKE predicates with
     `WHERE has_done_marker = 1 AND done_outcome = ?1`. The OR-clause was only a
     pre-filter; the loop body already re-derives `changed` from the decoded record, so
     narrowing to abandoned rows (an indexed equality over ~36 rows) is sufficient and
     the remaining conditions can be evaluated in Rust.

4. `crates/sase_core_py/src/lib.rs`
   - Register `load_agent_artifact_records`. Return the records through the same
     `serialize_to_py` path the query binding uses (the direct serializer landed in
     phase `marshal`); release the GIL around the SQL/decode with `py.allow_threads` as
     `py_query_agent_artifact_index` does.

5. Rust tests (`crates/sase_core/src/agent_scan/index.rs` test module, plus
   `crates/sase_core_py/tests/` for the binding)
   - A `List`-shaped query drops `_raw`/`_data` and `linked_repos` and sets
     `record_shape = "list"`, while every other field — including the `meta_*` keys in
     the same output maps, `prompt_steps` element count, and `workflow_state.steps`
     element count — is identical to the `Full` result.
   - A `Full`-shaped query serializes no `record_shape` key, and `record_json`
     round-trips unchanged through `upsert_record` (guards the golden corpus).
   - `load_agent_artifact_records` returns full records for a projected dir, resolves an
     alias path, and skips an unknown dir.
   - `repair_abandoned_agent_artifact_index_rows` still repairs the same rows after the
     LIKE removal, and the v24 migration backfills `done_outcome` on a v23 fixture
     index.

6. `refresh_stale_rows` (epic plan item 3) already selects only the `*_sig` columns and
   never reads `record_json` — verify this rather than changing it, and add a regression
   test asserting its SQL does not mention `record_json`, so the 2026-08-12 flag stays
   closed. Say so explicitly in the bead close note; do not report it as newly fixed.

### Part 2 — this repo (Python)

7. Wire mirrors
   - `src/sase/core/agent_scan_wire_records.py`: add
     `record_shape: Literal["full", "list"] = "full"` to `AgentArtifactIndexQueryWire`
     and to `AgentArtifactRecordWire`. Leave `AGENT_SCAN_WIRE_SCHEMA_VERSION` at 7.
   - `src/sase/core/agent_scan_wire_conversion.py`: emit `record_shape` from
     `agent_artifact_index_query_to_dict`, and read it in `_record_from_dict` with a
     `"full"` default so an older core that does not send the key still rehydrates.
   - `src/sase/core/agent_scan_facade.py`: add `load_agent_artifact_records`.

8. Ask for the list shape on exactly one query
   - `src/sase/ace/tui/models/_agent_loader_artifacts.py`,
     `query_artifact_index_for_loader` — the Tier 1 TUI query — sets
     `record_shape="list"`. Every other `query_agent_artifact_index` caller
     (`bead_pages`, `sdd`, `monitor/store.py`, `gate_shell/store.py`,
     `agents/cli_index.py`, `main/var_cli.py`, `agents_sync`, `completion`,
     `agent/_family_attach_candidates.py`) keeps the default full shape.
     `artifact_snapshot_for_live_plan_load` also stays full — it feeds CLI plan
     notification matching, not a list.

9. Carry the projection onto the `Agent`
   - `src/sase/ace/tui/models/_agent_state.py`: add
     `record_shape: Literal["full", "list"] = "full"`,
     `index_record_dir: str | None = None`, and
     `prompt_step_file_name: str | None = None`. `index_record_dir` is the _record's_
     `artifact_dir`, which for a workflow-step row is the parent record's dir, not the
     step's own `artifacts_dir`. `prompt_step_file_name` is
     `PromptStepMarkerWire.file_name`, the only stable key for picking one element back
     out of `record.prompt_steps`.
   - Set all three in the snapshot loaders that build agents from index records:
     `_workflow_snapshot_loaders.py` (`_build_workflow_agent_steps_for_record` and
     `load_workflow_agents_from_snapshot`), `_done_loaders.py`, and
     `_running_loaders.py`.
   - `_meta_enrichment_wire.py:82` currently does
     `agent.linked_repos = parse_linked_repos(meta.linked_repos)` unconditionally. Under
     a list shape that assignment is exactly the "renders as empty" bug the epic plan
     warns about: skip the assignment when the record is projected and leave the field
     for the resolver below.

10. Two hydration resolvers, then swap the call sites
    - New module (suggested `src/sase/ace/tui/models/_projected_record.py`):
      `resolve_step_output(agent)` and `resolve_linked_repos(agent)`. Both return the
      full value immediately when `agent.record_shape == "full"`; otherwise they fetch
      the record for `agent.index_record_dir` via `load_agent_artifact_records`,
      memoized per artifact dir with a small bounded cache keyed on the artifact dir,
      and select the right leaf:
      - a step row (`prompt_step_file_name` set) → that element's `output`,
      - a workflow parent row → the last `workflow_state.steps[*].output` that is a
        dict, matching the existing rule at `_workflow_snapshot_loaders.py:203-206`,
      - otherwise → `done.step_output`. `resolve_step_output` must **merge, not
        replace**: `merged = dict(hydrated); merged.update(agent.step_output or {})`.
        The projected dict is authoritative for every key it has, because
        `_done_loaders.py:151-175`, `_meta_enrichment_prompt_markers.py`, and
        `_agent_status_apply.py:266-275` all write derived `meta_*` keys into it after
        load; only `_raw`/`_data` come from the hydrated copy.
    - The hydration must not run on the event loop. Resolve on selection change through
      the same off-thread pattern the detail panel already uses for other per-selection
      work, and never from a synchronous render call — `tui_perf.md` rules 1, 8 and 10
      apply. Read `sase memory read tui_perf.md` before wiring it.
    - Swap these reads (they need `_raw`/`_data` or iterate all keys):
      - `widgets/prompt_panel/_agent_display_render.py:499-502`
      - `widgets/prompt_panel/_agent_display_hint_render.py:564-567`
      - `widgets/prompt_panel/_agent_display_step_render.py:71-72` and `94-95`
      - `widgets/prompt_panel/_agent_clan_member_content.py:113-114`, `124-125`
      - `models/_agent_clan_sections.py:381-396`
    - Leave these alone — they read only `meta_*`/scalar keys, which are never
      projected: `widgets/prompt_panel/_agent_commits.py:500,526`,
      `widgets/prompt_panel/_agent_display_header_metadata.py:378`,
      `models/agent.py:185`, `models/_dedup.py:365-370`,
      `models/_agent_status_apply.py:266-275`,
      `models/_loaders/_done_loaders.py:140-175`,
      `models/_loaders/_meta_enrichment_prompt_markers.py`.
    - Swap the three `linked_repos` iterations to `resolve_linked_repos(agent)`:
      `widgets/file_panel/_linked_deltas.py:98` (the chokepoint the whole file panel
      funnels through), `widgets/prompt_panel/_agent_commits.py:353`, and
      `src/sase/ace/revert_agent_resolution.py:396`.

11. Three correctness hazards that are not render call sites
    - **Dismissed bundles persist the Agent.** `to_bundle_dict`
      (`models/agent_bundle.py`) serializes every `init` field, and
      `actions/agents/_dismiss_memory.py:171` writes that to disk;
      `src/sase/ace/dismissed_bundle_index/_summary.py:102` later reads `step_output`
      back out of it. Bundling a projected agent would persist a truncated `step_output`
      and an empty `linked_repos` **permanently**. Hydrate before bundling, and never
      persist `record_shape="list"` into a bundle.
    - **Dedup merges two agents.** `models/_dedup.py:365-370` merges `step_output`
      across duplicates. Merge `record_shape` too: the result is `"full"` if either
      input is full, `"list"` only if both are.
    - **Fallback paths are always full.** `artifact_snapshot_for_tui_load`'s source-scan
      fallbacks and the artifact-delta path do not go through the index query, so agents
      from them stay `record_shape="full"`. Make sure the loaders default the field
      rather than inheriting a stale value, so a fallback load never mislabels a full
      record as projected.

12. Python tests
    - Wire round-trip: `record_shape` survives `agent_artifact_index_query_to_dict` /
      `_record_from_dict`, and defaults to `"full"` when the key is absent (extend
      `tests/test_core_agent_scan_wire_artifact_index.py` /
      `tests/test_core_agent_scan_records_index.py`).
    - Loader: a projected snapshot produces the **same number of agent rows, the same
      statuses, and the same `total_steps`** as the equivalent full snapshot — this is
      the regression gate for Correction B. Put it beside the existing snapshot-loader
      tests under `tests/`.
    - Hydration: an agent built from a projected record renders the same response body
      as one built from a full record, with `load_agent_artifact_records` stubbed; and
      the loader-written `meta_*` keys survive the merge (i.e. hydration does not
      clobber `meta_commits`/`meta_commit_message` added by `_done_loaders.py`).
    - Bundle: dismissing a projected agent round-trips a full `step_output` and full
      `linked_repos`.
    - Guard: the non-TUI `query_agent_artifact_index` callers still request the full
      shape.

## Verification

- `just check` in this repo before handing off. If the scoped run escalates or reports
  an unusual selection, run `just check-full` through `/sase_monitor` with the `TESTING`
  / `TESTED` status pair — it routinely outruns one turn.
- In the core checkout: `cargo test -p sase_core -p sase_core_py`, plus `just check`
  there (`./scripts/check.sh all`).
- Rebuild the extension before running Python tests that exercise the new binding:
  `just install` builds `sase_core_rs` from the local `sase-core` checkout.
- Measure and record on bead `sase-uv.7`, against the `baseline` phase numbers: the
  projected payload share on the implementing host, the warm `load_tiered_agents` time
  before and after, and the `serialize_to_py`/wire rehydration split. Report what you
  measured, not what was budgeted.

### Expect the pinned-core-bindings gate to flag this

This phase adds a binding (`load_agent_artifact_records`) that this repo then calls. CI
builds `sase_core_rs` from the SHA in `sase-core-revision.txt`, not from `sase-core`
HEAD, so `tools/check_sase_core_rs_bindings` in the `lint` job will fail by name until
that pin is ratcheted past the core commit (`just ratchet-core-revision`; see
`docs/rust_backend.md`, "The CI source revision pin"). `just check` only runs
`tools/probe_core_floor --advisory`, so it will pass locally. This is the epic's normal
two-repo sequencing, not a defect: leave a `PROPOSED FOLLOW-UP:` note on the bead so the
epic's land agent bumps the pin before landing.

## Out of scope

- The `record_detail_json` column split described under "What this projection does and
  does not save". Note it; do not build it.
- Anything the `viewport` phase (`sase-uv.8`) owns. This phase must not start wiring
  `AgentsViewport` through `DirectAgentsDataProvider`.
- Read-only index opens and the index vacuum tooling — the `hygiene` phase already
  landed `open_index_read_only`; reuse it, do not rework it.
