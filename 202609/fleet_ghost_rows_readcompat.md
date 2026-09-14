---
tier: epic
title:
  Fleet ghost rows — finish live acceptance, fix v1 read-compat, make broken feeds
  honest
goal: "Athena's ACE never silently renders stale ghost agents for a remote host: the
  sase-xe.16.11.7.15 live-acceptance phase is completed and closed on today's
  verified-live fleet view, capability-set validation honors the contract's claimed v1
  read-compatibility so mixed-version hosts do not collapse to zero rows behind a
  hello-ok status, and a host whose feed is invalid or stale is rendered loudly
  (host-level error surfacing plus honest staleness chrome on every cached row) instead
  of masquerading as healthy.

  "
parent_bead: sase-xe.16.11.7
phases:
  - id: close-live-acceptance
    title: Complete and close the sase-xe.16.11.7.15 live-acceptance phase
    depends_on: []
    size: small
    description:
      "close-live-acceptance: with SSH from Apollo to Athena now working and Apollo's
      gateway restarted onto the current installed binary, capture Athena-side `sase ace
      --tmux` pane evidence of the live Apollo machine group (family/clan nodes, host
      chips, honest chrome, no ghost rows), attach it to phase bead sase-xe.16.11.7.15.7
      with a root-cause note, and close that phase bead so the epic's land agent can
      land and close epic bead sase-xe.16.11.7.15. Leave the phase open on any unmet
      gate. Never close the epic bead itself."
  - id: caps-readcompat
    title: Capability-set validation honors claimed v1 read-compatibility
    depends_on: []
    size: medium
    description:
      "caps-readcompat: in sase-core fleet_contract.rs, stop rejecting
      older-but-readable capability sets — the normalized-equality check must ignore the
      schema_version stamp that CapabilitySetWire::normalized rewrites to the current
      version — audit the file for the same normalize-then-strict-equality pattern on
      other versioned wires, and add regression tests proving a v1 summary envelope
      yields rows instead of fleet_envelope_invalid while unnormalized content is still
      rejected. Full core checks including PyO3."
  - id: invalid-feed-honesty
    title: Invalid or stale host feeds render loudly in the viewer
    depends_on: []
    size: medium
    description:
      'invalid-feed-honesty: when a remote host''s snapshot is status "invalid" or
      served from cache beyond freshness thresholds, the ACE Agents tab must surface the
      feed error at the host/machine level (banner or host row with error state and
      cache age) and stamp honest staleness chrome on every cached row — no plain
      RUNNING rows frozen from an hours-old cache — with the fleet_envelope_invalid
      diagnostic reachable from the detail panel; regression tests cover the
      invalid-host and stale-cache render paths.'
  - id: gateway-skew-loudness
    title: Gateway version skew is visible, and the upgrade runbook says to restart
    depends_on: []
    size: small
    description:
      'gateway-skew-loudness: surface remote gateway service/contract version skew in
      `sase machine status` output (hello already carries service versions) so an
      outdated target gateway is visible instead of hiding behind "hello ok", and extend
      docs/remote_dispatch.md with the restart-after-upgrade requirement for supervised
      gateways, whose Restart=on-failure units keep running the old binary after an
      install upgrade.'
proposed_by: bbugyi200.apollo.01
create_time: 2026-09-14 16:25:22
status: wip
---

- **PROMPT:**
  [prompts/202609/fleet_ghost_rows_readcompat.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/fleet_ghost_rows_readcompat.md)

# Fleet ghost rows: finish live acceptance, fix v1 read-compat, make broken feeds honest

## Problem

On 2026-09-14 at ~15:33 EDT, Athena's ACE Agents tab
(`~/tmp/screenshots/20260914_153340.png`, sase 0.17.1+644) showed an `apollo` machine
group with 8 agents — two `lane` rows rendered plain `RUNNING … 6h01m`, two `attempt-0`
rows as `[agent] … unknown · aging`, a `research.2` WAITING row, and two `y--plan` rows
`offline · aging` — while Apollo's own ACE (`~/tmp/screenshots/20260914_153353.png`,
sase 0.17.1+628) showed none of those agents alive. The rows on Athena were ghosts: a
cached fleet snapshot frozen at roughly 10:35 EDT, rendered without any indication that
the live feed had been failing for hours. The user's suspicion that these remote agents
"just shouldn't be showing" on Athena is confirmed, and the issue is directly related to
epic bead `sase-xe.16.11.7.15` (remote agents display parity), whose final phase
`sase-xe.16.11.7.15.7` (live Athena-to-Apollo acceptance) is still open.

## Root cause (diagnosed live, 2026-09-14)

Three stacked causes, verified today:

1. **Stale gateway binary (operational trigger).** Apollo's user-systemd `sase-gateway`
   had been running since 2026-09-11 on an old binary reporting gateway 0.34.9, emitting
   fleet-contract schema-v1 capability sets, while Apollo's installed sase had moved on.
   Recorded as a DISCOVERED ISSUE on bead `sase-xe.16.11.7.15` (note #2). The gateway
   was restarted at 19:25:57 UTC and again — after Apollo's install upgrade to sase
   0.17.1+645 / sase-core-rs 0.34.28 — at 19:55:35 UTC, so it now serves current
   schema-v3 envelopes.
2. **Broken v1 read-compatibility in the validator (code defect, confirmed in sase-core
   `crates/sase_core/src/fleet_contract.rs`).** The contract declares
   `FLEET_CONTRACT_MIN_READABLE_SCHEMA_VERSION = 1` and `validate_schema` accepts 1..=3,
   but `CapabilitySetWire::normalized()` (~line 3620) rewrites `schema_version` to
   `FLEET_CONTRACT_SCHEMA_VERSION` (3), and summary validation (~line 2916) requires
   `normalized == original`. Any capability set carrying an older-but-readable schema
   version therefore fails with "summary capabilities are not normalized" **by
   construction**, the whole host envelope is normalized to an `invalid_federation_host`
   (status `invalid`, zero summaries, error `invalid_envelope`), and the host's live
   projection collapses to zero rows. This is why Athena rejected Apollo with
   `fleet_envelope_invalid` once Athena's core reached 0.34.28 while Apollo's gateway
   still emitted v1.
3. **Silent failure in the viewer (presentation defect).** With the live feed rejected,
   Athena's ACE kept rendering the last-good cached rows. The machine banner showed
   healthy counts ("8 agents · 3 running · 5 awaiting"), two cached rows rendered plain
   `RUNNING` with no staleness chrome despite the snapshot being ~5 hours old, and the
   `fleet_envelope_invalid` diagnostic surfaced nowhere in the UI. `sase machine status`
   reported `hello ok` the whole time. Nothing in
   `src/sase/ace/tui/models/_fleet_agents_rows.py` or
   `src/sase/ace/tui/actions/agents/_fleet_projection.py` consumes host
   `status: "invalid"` today.

**Current live state (verified 2026-09-14 ~16:05 EDT):** driving Athena's own federation
path (`build_federation_facade().catalog_sync(...)` over SSH) now returns
`apollo | status: ok | cached: False` with 8 rows, all capability sets at schema v3,
matching Apollo's real agent population. The live symptom is resolved by the gateway
restart; the ghosts should already have been replaced on Athena's next auto-refresh.
What remains is the epic's evidence-gated close plus the durable fixes so this failure
mode cannot recur silently.

## Prior state of the epic

Epic bead `sase-xe.16.11.7.15` has phases 1–6 closed; phase 7 (`sase-xe.16.11.7.15.7`,
live acceptance) is `in_progress`. Its previous worker was blocked solely on access: SSH
from Apollo to Athena failed public-key auth and no remote machines were enrolled on the
Apollo controller, so it recorded the unmet gate (evidence artifact
`file:explicit:b79d37e46dc8755ef4d5daef`) and left the phase open. That blocker is gone:
SSH `athena` from Apollo now works (verified today). The epic's land agent
(`sase-xe.16.11.7.15.land`) is alive and WAITING on the phase beads; it — and only it —
closes the epic bead after verifying the acceptance evidence.

## Scope and coordination

- This plan completes and closes **phase bead `sase-xe.16.11.7.15.7` only**. No worker
  may close epic bead `sase-xe.16.11.7.15` (its land agent owns that), nor any other
  epic's beads.
- Bead `sase-xe.16.11.7.15` note #1 records that a child of `sase-zt.6.5` owns adding
  `queue_capacity`/explicitness to `ResolvedAgentSummaryWire`. The `caps-readcompat`
  phase touches validation logic only and must not add, remove, or re-shape summary wire
  fields; if a conflict appears, coordinate rather than absorbing that work.
- Release-plz owns sase-core versions: never hand-pin an unpublished core version. The
  `caps-readcompat` fix lands in core and reaches installs through the normal release
  flow; no consumer in this plan depends on it being published first.
- The gateway systemd unit itself is machine-managed (chezmoi), not sase-repo code;
  automating restart-on-upgrade at the unit level is out of scope here. The
  `gateway-skew-loudness` phase makes the skew visible and documents the manual restart
  requirement; if unit-level automation seems warranted, record it as a
  `PROPOSED FOLLOW-UP:` note, do not implement it.

## Constraints (binding for every phase)

- Open sase-core and any other repository through `/sase_repo`; read reference memory
  through `/sase_memory_read`: `tailnet.md` before any cross-machine work, `tui_perf.md`
  before TUI changes, `lint_and_test.md` before finishing any sase change,
  `symvision.md` for lint failures, `cli_rules.md` before changing CLI output.
- Baseline honesty: record the pre-existing failing-test baseline for any test files
  touched before changing them; never absorb pre-existing failures silently.
- No new feature flags: this is repair of unconditional behavior. If a flag seems
  unavoidable, read `sase_flags.md` first and record the reasoning.
- Preserve TUI performance budgets: keystroke paths stay read-only, j/k p95 under 16 ms,
  render-cache keys stay honest for every field that can change visible state.
- Phase workers never create beads; record discoveries as `PROPOSED FOLLOW-UP:` notes on
  their own phase bead.
- Long commands (core checks, `just check-full`) run through `/sase_monitor`.

## Phases

### close-live-acceptance

size: small — dependencies: none

Finish `sase-xe.16.11.7.15.7` exactly as its design specifies, now that the access
blocker is gone:

- Read `tailnet.md` through `/sase_memory_read` first. Verify preconditions from Apollo:
  `ssh athena` works noninteractively; both machines run builds containing every prior
  phase (Athena ≥ 0.17.1+644, Apollo ≥ 0.17.1+645, both core 0.34.28); Apollo's gateway
  is the current binary (`systemctl --user status sase-gateway` active since ≥
  2026-09-14 19:55 UTC).
- Over SSH on Athena, launch a fresh `sase ace --tmux` instance (do not disturb the
  user's existing tmux session), focus the Agents tab, and capture pane evidence
  (`tmux capture-pane`) of the Apollo machine group in mixed and BY_MACHINE groupings:
  family/clan-grouped agent nodes with `apollo` host chips, one row per logical agent
  with expandable member shells, human project name `sase`, real timestamps/durations,
  no `[agent]` brackets, no `here` chips on Athena-local rows, no `online · aging`
  chrome on healthy rows, and — critically — none of the ghost rows from
  `~/tmp/screenshots/20260914_153340.png` (`lane`, `attempt-0`, stale `y--plan`). Kill
  the test tmux window afterward.
- Register the captured evidence as explicit artifacts on phase bead
  `sase-xe.16.11.7.15.7` via `/sase_artifact`, cross-referencing the retained "before"
  evidence (`~/tmp/screenshots/20260913_180154.png`, the 2026-09-14 ghost screenshots,
  and prior blocker artifact `file:explicit:b79d37e46dc8755ef4d5daef`).
- Append a closing note to the phase bead summarizing the root-cause chain (stale 0.34.9
  gateway → v1 envelopes rejected by the 0.34.28 validator's normalized-equality defect
  → viewer froze cached rows silently) and the resolution (gateway restarted onto
  current binary after the install upgrade; SSH access repaired), then close
  `sase-xe.16.11.7.15.7` with `sase bead close`.
- The epic's land agent then wakes, verifies, lands, and closes epic bead
  `sase-xe.16.11.7.15`. Do not close, poke, or wait on the land agent; just confirm it
  is no longer blocked on open phase beads.
- If any acceptance gate is unmet (for example the Athena view still renders ghosts or
  parity chrome regressions), record the exact unmet gate on the phase bead, leave it
  open, and stop.

### caps-readcompat

size: medium — dependencies: none

In sase-core (open through `/sase_repo`), make capability-set validation honor the
contract's claimed read range:

- `crates/sase_core/src/fleet_contract.rs`: summary validation (~line 2916) currently
  rejects when `capabilities.normalized() != capabilities`. Because `normalized()`
  stamps `schema_version: FLEET_CONTRACT_SCHEMA_VERSION`, any set with a readable older
  version (validated 1..=3 by `validate_schema`) fails unconditionally. Fix the check to
  enforce content normalization (sorted, deduplicated, valid capability strings) while
  accepting any readable `schema_version` — e.g. compare against a normalized copy that
  preserves the original version, or compare the normalized content fields directly.
  Genuinely unnormalized content must still be rejected with the existing message.
- Audit `fleet_contract.rs` for the same normalize-then-strict-equality pattern on other
  versioned wire types (any `x.normalized()? != x` or equivalent where `normalized`
  rewrites `schema_version`) and apply the same treatment where the wire is validated on
  ingest; record the audit result in the phase note.
- Decision, recorded here deliberately: **preserve** v1 read-compatibility rather than
  dropping the claim. Bryan's machines upgrade at different times; the harm today came
  from a readable envelope being silently discarded. Unreadable versions (outside 1..=3)
  keep failing loudly.
- Regression tests: a summary whose capability set carries `schema_version: 1` (and 2)
  with sorted content passes validation and a host envelope containing it yields rows
  rather than an `invalid_federation_host` with `fleet_envelope_invalid`; unsorted or
  duplicated capability content still fails; version 0 and 4 still fail
  `validate_schema`.
- Do not add, remove, or re-shape `ResolvedAgentSummaryWire` fields (coordination with
  the `sase-zt.6.5` child noted above). Run the full core check
  (`just check`/`scripts/check.sh` including PyO3) through `/sase_monitor`.

### invalid-feed-honesty

size: medium — dependencies: none

In the sase repo viewer, make a broken or stale host feed impossible to mistake for a
healthy one. Evidence: on 2026-09-14 Athena rendered a ~5-hour-old cached Apollo
snapshot with a healthy-looking banner, plain `RUNNING` rows, and no surfaced error
while every live fetch was failing `fleet_envelope_invalid` behind `hello ok`.

- Read `tui_perf.md` first. Trace how federation snapshots reach
  `src/sase/ace/tui/models/_fleet_agents_rows.py` and
  `src/sase/ace/tui/actions/agents/_fleet_projection.py`, and where host-level `status`
  (`ok`/`invalid`), `cached`, `age_seconds`, `freshness.error`, and normalization
  diagnostics are dropped today.
- Host-level surfacing: when a host's latest snapshot is `status: "invalid"` (or carries
  `freshness.error`), the machine banner/host grouping header must render a qualified
  error state — feed error plus the age of the data being shown — instead of bare
  healthy counts. The `fleet_envelope_invalid` diagnostic text must be reachable from
  the detail panel for that host.
- Row-level honesty: every row rendered from a cached snapshot older than the freshness
  thresholds carries staleness chrome (stale/offline/`WAS RUNNING · last seen …` per the
  design vision) — a cached row must never render as plain `RUNNING`/`WAITING`. Healthy
  fresh rows keep the chrome-free rendering the remote-render-integration phase
  established; freshness details stay in the detail panel.
- Keep render-cache keys honest for every newly consumed field; route updates through
  the existing selective-update fast paths.
- Regression tests: an invalid-host snapshot (use the real normalized shape
  `invalid_federation_host` produces: status `invalid`, empty summaries, freshness error
  `invalid_envelope`) renders the host error state and no phantom healthy rows; a
  cached-stale snapshot renders every row with staleness chrome; a healthy snapshot
  renders unchanged. Extend the existing fleet fixtures
  (`tests/ace/tui/_fleet_summary_fixture.py`, `_fleet_response_fixture.py`) rather than
  inventing envelope schemas; refresh PNG snapshots only if banner rendering changes
  them.

### gateway-skew-loudness

size: small — dependencies: none

Make version skew visible where the operator already looks, and document the operational
requirement:

- Read `cli_rules.md` before changing CLI output. `sase machine status` today prints
  `hello ok` for a host whose gateway is days out of date. Discovery/hello responses
  already carry service version information (the prior phase-7 worker observed both
  gateways "report service versions older than this workspace's installed core").
  Surface that in `sase machine status` (human and `--json` forms): show the remote's
  reported gateway/service version and flag when it is older than the viewer's minimum
  fully-supported surface or otherwise skewed, without turning the hello check into a
  hard failure for readable-but-old peers.
- Extend `docs/remote_dispatch.md` (Linux/macOS supervision sections): supervised
  gateways use `Restart=on-failure`, which never picks up an upgraded binary — after
  upgrading a target's sase install, restart its gateway service, and note the symptom
  of forgetting (viewers reject or misread envelopes from the old process; before the
  caps-readcompat fix this silently emptied the host's rows).
- If unit-level restart-on-upgrade automation looks worthwhile, record it as a
  `PROPOSED FOLLOW-UP:` note on this phase's bead (the unit files are chezmoi-managed,
  outside this repo).

## Verification and landing

Every sase phase: `just check` green with the recorded pre-existing-failure baseline
compared explicitly; `lint_and_test.md` obligations honored. Core phase: full core check
including PyO3 through `/sase_monitor`. The `close-live-acceptance` phase is
evidence-gated: pane captures attached to the phase bead are the deliverable, and the
phase bead stays open if any gate is unmet. Landing runs the combined-tree
`just check-full` through `/sase_monitor` and closes this plan's beads through the
normal landing flow; this plan's land agent never touches `sase-xe.16.11.7.15` or its
descendants.
