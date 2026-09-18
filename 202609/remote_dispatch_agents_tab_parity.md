---
tier: epic
title: Remote dispatch Agents-tab parity
goal: 'Remote machine nodes on the Agents tab are indistinguishable from local nodes
  except for the machine chip, and a viewer filtered to machine:X shows exactly the
  nodes machine X''s own TUI shows — same count, grouping, statuses, and chips — with
  stale owner-side rows retired and `sase screenshot` able to drive query filters
  unattended for the cross-machine evidence captures.

  '
phases:
- id: owner-served-set-parity
  title: Owner served-set parity
  depends_on: []
  size: large
  description: 'owner-served-set-parity: make the gateway snapshot serve exactly the
    rows the owner''s own TUI presents — retire dead-PID protected waiting rows, honor
    owner dismissal state, and harden liveness beyond bare kill(pid,0) — with the
    selection decision unit-tested in sase_core.'
- id: wire-presentation-facts
  title: Wire presentation facts
  depends_on:
  - owner-served-set-parity
  size: large
  description: 'wire-presentation-facts: extend the fleet contract additively so every
    row carries the owner-presented display status, family/parent linkage for historical
    shells, resolved tribe/clan tribe, and both runtime timestamps, populated by the
    gateway projection and covered by golden wire fixtures.'
- id: viewer-remote-node-parity
  title: Viewer remote-node render parity
  depends_on:
  - wire-presentation-facts
  size: large
  description: 'viewer-remote-node-parity: render remote rows through the same grouping
    and presentation paths as local rows — tribe panels, nested done shells, shell-count
    and proc/gate/monitor chips, owner status text, and banner counts that include
    done nodes — differing only by the machine chip.'
- id: screenshot-text-input
  title: sase screenshot text-input driving
  depends_on: []
  size: medium
  description: 'screenshot-text-input: add an ordered, repeatable --type option that
    sends literal text interleaved with -p presses and -w waits, forward it through
    the remote --host leg, and cover ordering and remote argv forwarding with tests,
    so query filters can be driven unattended.'
proposed_by: bbugyi200.athena.0na
create_time: 2026-09-18 16:37:55
status: wip
bead_id: sase-133
---

- **PROMPT:** [prompts/202609/remote_dispatch_agents_tab_parity.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/remote_dispatch_agents_tab_parity.md)
- **BEAD:** [sase-133](https://github.com/sase-org/sase--beads/blob/main/pages/sase-133/README.md)

# Remote Dispatch Agents-Tab Parity

## Problem

Remote nodes on the Agents tab must look identical to local nodes except for the
machine-name chip (for example `apollo`), and a viewer machine filtered to `machine:X`
must show exactly the same agent nodes that machine X's own TUI shows. Today both
properties are violated. Comparing live screenshots taken on 2026-09-18 (athena viewing
`machine:apollo` vs apollo's own TUI):

- apollo's TUI showed 14 nodes (`@default` tribe panel: 6 `[R1 W1 D4]`; `@epic` tribe
  panel: 8 `[R2 W1 D5]`); athena's `machine:apollo` view showed 17 matched rows with
  **no `@epic` tribe panel at all** and a `@default` banner counting only `5 [R2 W3]`
  (done rows rendered but not counted).
- The same running agent rendered as `(TESTING) ×10` with proc/gear chips and
  `27m44s / 3h22m` runtimes on apollo, but as bare `(RUNNING)` with no shell count, no
  chips, and collapsed runtimes on athena.
- Done agents rendered on apollo as family-grouped agent nodes with rich done statuses
  (`TALE DONE`, `EPIC CREATED √`); on athena they rendered as top-level per-shell rows
  named `<agent>--<shell>` (`0k--plan`, `0j--0`, `0e.f0.f0--plan`) marked
  `offline · aging`.
- athena showed rows apollo no longer shows at all: a `RUNNING` row 8h10m old (`0i`) and
  a `WAITING ×2` clan (`research.2`) 103h old whose recorded PID is dead, while two of
  apollo's three actually-running agents were missing from the remote feed.

## Diagnosis (verified against the live wire)

A raw `fleet catalog` pull from apollo's gateway (via the federation facade) plus direct
inspection of apollo's `~/.sase/agent_artifact_index.sqlite` established the following
root causes.

### RC1: The owner-side served set is a different universe than the owner's TUI

The gateway builds its snapshot in `build_snapshot_blocking()`
(`crates/sase_gateway/src/fleet_reads.rs` in the `sase-core` repo) from the agent
artifact index, then selects rows with `decide_fleet_presentation()`
(`crates/sase_core/src/fleet_presentation.rs`). That selection diverges from the owner
TUI's own listing rules:

- **Protected rows never retire.** Any record with a waiting marker or pending question
  is `protected` and stays `current` forever, regardless of liveness or age (see test
  `protected_dead_row_stays_current_despite_liveness`). apollo's index holds 10
  waiting-marker records; its gateway serves all 10 as current WAITING rows, while
  apollo's own TUI shows only 2 (the other 8 include `research.2.*` rows 103h old whose
  recorded PID 2056419 is dead).
- **Liveness is a bare `kill(pid, 0)`.** `owner_liveness_for_record()` treats any live
  PID as `Alive` — a recycled PID resurrects a long-dead agent (the `0i` row served as
  RUNNING 8h+ after apollo stopped showing it). The Python side at least detects zombies
  (`sase.ace.hooks.processes.is_process_running`) and additionally reconciles workspace
  claims (`_release_stale_running_claim` in
  `src/sase/ace/tui/models/_loaders/_running_loaders.py`).
- **Owner dismissal state is ignored for protected/live rows.** The snapshot consults
  family-dismissal lineage only to _exclude dead orphans_; the index's
  `dismissed_agents` table (1245 rows on apollo) — which the owner TUI honors — is never
  consulted for protected or alive rows.
- **Freshly launched shells lag or race the snapshot cache**, so currently running
  agents can be absent from a pull that includes hours-old dead rows (observed:
  `sase-12y.3--2` running on apollo but absent from the concurrent catalog pull).

### RC2: Wire rows drop the owner's presentation facts

`resolve_record()` (gateway) and `project_resolved_agent_detail()`
(`crates/sase_core/src/fleet_contract.rs`) only project what the index record carries,
and the index meta for most records lacks presentation facts:

- `status` on the wire is the coarse bucket (RUNNING/WAITING/DONE). The owner's
  presented statuses — `TESTING`, `WORKING TALE`, `TALE DONE`, `EPIC CREATED √`,
  `WAITING ▶1` — are derived owner-side by the TUI status pipeline and never cross the
  wire.
- Historical/done shells arrive with `family_id: null` and `parent_timestamp: null`, so
  the viewer cannot nest them under their agent or family node and renders them as
  top-level `<agent>--<shell>` rows.
- `tribe`/`clan_tribe` are null on most rows (on apollo only `*.land` records carry
  `clan_tribe: "epic"`), so remote rows cannot form the `@epic` tribe panel the owner
  shows. The tribe is resolvable owner-side from the clan/family root.
- No agent- or family-level container row is published for done agents, so shell counts
  (`×N`), proc/gate/monitor chips, and family aggregation cannot be reconstructed by the
  viewer.

### RC3: Viewer-side remote projection cannot compensate

`src/sase/ace/tui/models/_fleet_agents_nodes.py` already synthesizes missing family
containers and rewrites parent links, but only from what the wire gives it; with RC2
facts missing, done shells stay top-level, tribe panels never form, and banner counts
(`[R2 W3]` with no `D`) disagree with the owner. The row mapping in
`_fleet_agents_rows.py` already passes through `tribe`, `clan_tribe`,
`parent_timestamp`, and `family_role` when present, so most of RC3 unblocks once RC2
lands; remaining gaps are pure presentation (tribe panels, banner counting, shell-count
chips, done-status styling for remote rows).

## Goal And Acceptance

1. **Set parity:** for every enrolled machine X, a viewer's Agents tab filtered to
   `machine:X` lists exactly the agent nodes X's own TUI lists (same node count, same
   grouping into tribe panels, clans, families, and nested shells).
2. **Render parity:** each remote node renders identically to the corresponding local
   node on X — same presented status text, shell counts, chips, and runtime columns —
   except for the machine-name chip (and machine-specific detail-panel fields such as
   `Machine:`).
3. Stale rows the owner no longer presents (dead-PID waiting rows, dismissed agents,
   recycled-PID "running" rows) no longer appear on viewers.

## Constraints

- Shared backend/domain behavior belongs in the `sase-core` repo (`crates/sase_core`),
  per the rust-core boundary rule: the served-set decision and the row-projection
  contract are exactly such behavior. Python keeps only presentation glue. Phase workers
  must open that repo with the `/sase_repo` skill
  (`sase repo open sase-core -r "<why>"`) and make Rust wire/API, binding, and test
  changes there.
- The fleet contract is versioned; additive fields need a schema-version bump and
  tolerant readers (older gateways without the new fields must still render, with
  today's degraded behavior, rather than error).
- Cross-machine verification requires the upgraded `sase`/`sase-core` install on the
  remote machine and a gateway restart (see `docs/remote_dispatch.md`, "restart target
  gateway" / version-skew guidance).

## Phases

### Owner served-set parity

Make the gateway serve exactly the rows the owner's own TUI presents.

Steps:

1. Open the `sase-core` linked repo with `/sase_repo`. Study
   `decide_fleet_presentation()` (`crates/sase_core/src/fleet_presentation.rs`),
   `build_snapshot_blocking()` / `owner_liveness_for_record()`
   (`crates/sase_gateway/src/fleet_reads.rs`), and the owner TUI's local listing rules
   (`src/sase/ace/tui/models/_loaders/_running_loaders.py`,
   `_done_filesystem_loaders.py`, and the dismissed-bundle handling under
   `src/sase/ace/dismissed_bundle_index/`) to enumerate every rule by which the local
   listing includes/excludes a record.
2. Extend the owner-side selection so it matches the local listing:
   - Retire protected (waiting/question) rows whose recorded process is dead, mirroring
     however the local TUI resolves them (demote to recent-terminal, not silent
     deletion, if that is what the owner shows).
   - Consult owner dismissal state (the index's `dismissed_agents` table / dismissal
     lineage) for all candidates, not only dead unprotected orphans.
   - Harden liveness beyond bare `kill(pid, 0)`: at minimum zombie detection to match
     `is_process_running`, plus a process-identity guard (start-time or cmdline check)
     so recycled PIDs do not resurrect dead agents.
3. Keep the selection decision in `sase_core` (pure, fact-driven, unit-tested) with the
   gateway supplying observations, following the existing candidate/decision split. If
   local-listing rules currently live only in Python, either port the decision rule into
   `sase_core` and call it from the Python listing path through the binding, or (only
   where porting is disproportionate) have the Python lifecycle hooks persist the needed
   fact into the index so both sides read one source of truth. Document which option
   each rule took in the phase notes.
4. Add/extend Rust unit tests for each new selection rule and a gateway-level test that
   a snapshot excludes: dead-PID protected rows, dismissed rows, and recycled-PID
   running rows.

Acceptance: with owner and viewer on the same build, the catalog row set for a host
equals the row set the host's own TUI lists (verified by an integration test that builds
both from one fixture index, and manually against apollo).

### Wire presentation facts

Carry the owner's presentation facts on every wire row so a viewer can render the node
identically.

Steps:

1. Open `sase-core` with `/sase_repo`. Extend the resolved-summary contract
   (`ResolvedAgentSummaryWire` in `crates/sase_core/src/fleet_contract.rs`) with the
   owner-presented facts that are currently dropped (exact field list to be confirmed
   during implementation against the local render pipeline):
   - `display_status` (owner-presented status text, e.g. `TESTING`, `TALE DONE`,
     `EPIC CREATED`) plus any status-detail the local row shows (e.g. waiting follow-up
     count for `WAITING ▶1`).
   - Family/parent linkage for historical shells (`family_id`, `parent_timestamp`,
     `family_role`) so done shells nest under their agent.
   - Owner-resolved `tribe`/`clan_tribe` for every row (resolved from the clan or family
     root when the record's own meta lacks it).
   - Both lifecycle timestamps needed for the local `elapsed / total` runtime rendering
     (run start and family/agent start), not one collapsed value.
2. Bump the fleet contract schema version additively; older viewers must ignore the new
   fields and current viewers must tolerate hosts that omit them.
3. Populate the new fields in the gateway's `resolve_record()` /
   `project_resolved_agent_detail()`. Where a fact is only known to the Python lifecycle
   (for example presented status transitions), persist it into the index record at write
   time (`sase.core.agent_artifact_index_lifecycle`) so the Rust projection reads it —
   same one-source-of-truth rule as phase 1 step 3.
4. Update `sase_core_rs` bindings and Rust + Python tests covering the new projection,
   including a fixture where a done family's shells arrive fully linked and
   tribe-tagged.

Acceptance: a catalog pull for a host contains, for every row, the presented status
text, family linkage, and tribe facts needed to reproduce the owner's rendering; golden
wire-fixture tests assert the new fields.

### Viewer remote-node render parity

Render remote rows through the same grouping and presentation paths as local rows,
differing only by the machine chip.

Steps:

1. Read the `tui.md` reference memory chain
   (`sase memory read tui.md tui_screenshot.md tui_perf.md`) before touching TUI
   rendering paths.
2. Consume the new wire facts in `src/sase/ace/tui/models/_fleet_agents_rows.py` and
   `_fleet_agents_nodes.py`: nest historical shells under their family/agent nodes, use
   owner `display_status` for the status cell, and stop synthesizing lossy containers
   when real linkage is present.
3. Group remote rows into tribe panels (`@epic` etc.) exactly as local rows are grouped,
   and make tribe-panel banner counts include done nodes so a banner like `[R1 W1 D4]`
   matches the owner's.
4. Restore shell-count (`×N`) and proc/gate/monitor chips for remote family and agent
   nodes from the nested wire rows.
5. Keep the machine chip and remote-only affordances (feed staleness markers,
   `WAS RUNNING` last-seen labels, capability-gated actions) unchanged; they are
   deliberate remote additions, not parity violations.
6. Extend TUI tests: unit tests over the projection, plus visual snapshot coverage for a
   remote family node (running with `×N`, and done nested shells) under
   `tests/ace/tui/visual/` following the existing golden-lane rules.

Acceptance: with a fixture feed built from a local snapshot, the rendered remote node
list is pixel-identical to the local rendering of the same agents except for machine
chips; `machine:<alias>` filter counts equal the owner's visible node count.

### sase screenshot text-input driving

Improvements discovered while using `sase screenshot` for this diagnosis (the command
otherwise worked well locally, for `--window` recapture, and for remote `--host` capture
with `-p`/`-w` forwarding). Called out explicitly per the request:

1. Read the `cli_rules.md` reference memory before adding options.
2. Add a repeatable `--type TEXT` option that sends literal text to the TUI (tmux
   `send-keys -l` semantics). It must interleave with `-p` presses and `-w` waits in
   argv order (one ordered input script, e.g. via a shared argparse action list), so a
   single command can do:
   `sase screenshot -p slash --type "machine:apollo" -p enter -w "17/17" -o out.png`.
   Today typing a query requires `--keep`, manual `tmux send-keys`, and a second
   `--window` invocation.
3. Forward `--type` steps (in order) through the remote leg (`_remote_screenshot_argv()`
   in `src/sase/screenshot/remote.py`), and fail with the existing actionable contract
   error when the remote `sase` is too old to support them.
4. Update `sase screenshot` help text and the `?` help popup if applicable. Do NOT edit
   SASE memory notes in this phase: `sase/memory/tui_screenshot.md` will be stale after
   this change, so file a `memory` task bead via `/sase_new_task` describing the needed
   note update instead.
5. Cover the new option with unit tests for ordering (press/type/wait interleaving) and
   remote argv forwarding.

Acceptance: a single local command and a single `--host apollo` command can set the
Agents query filter and capture, with no manual tmux driving.

## Lander Instructions

The epic lander must demonstrate completion with live evidence from both machines and
save it as sase artifacts:

1. Deploy the finished work everywhere it is needed: land through the normal flow, then
   upgrade `sase` on apollo (`uv tool install --reinstall sase` per
   `docs/remote_dispatch.md`), restart apollo's gateway service and AXE, and confirm
   `sase machine status apollo` reports no version skew.
2. On apollo (via `sase screenshot --host apollo`), capture its Agents tab.
3. On this machine, capture the Agents tab filtered with the `machine:apollo` Agents
   query (using the phase-4 `--type` flow, e.g.
   `-p slash --type "machine:apollo" -p enter`).
4. Take both captures as close together in time as practical, and verify before saving:
   the `machine:apollo` node count on this machine equals apollo's own visible node
   count, tribe panels match, and spot-check one running and one done family for
   identical rendering modulo the machine chip.
5. Save both PNGs as sase artifacts with descriptive labels, e.g.
   `sase artifact create -p <local>.png -l "Agents tab on athena filtered machine:apollo (parity evidence)"`
   and
   `sase artifact create -p <apollo>.png -l "Agents tab on apollo (parity evidence)"`,
   and reference both artifacts in the landing notes.
