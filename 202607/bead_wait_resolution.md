---
tier: tale
title: Bead conditions in wait resolution
goal: '%wait(bead=...) conditions actually gate agent launches: the agent runner threads
  wait_beads into waiting.json as wait_for_beads, and dependency_resolution_status,
  the wait_checks chop, and the runner''s direct-resolution fallback all require the
  named beads to be closed in the waiting agent''s project bead store, failing closed
  when a bead or store is unavailable.

  '
create_time: 2026-07-20 12:09:12
status: wip
prompt: 202607/prompts/bead_wait_resolution.md
---

# Plan: Bead conditions in wait resolution (sase-87.3)

This implements the "Bead conditions in wait resolution" phase of the bead-gated `%wait` epic (parent bead `sase-87`,
plan `202607/bead_gated_wait.md` in the plans sidecar). Close bead `sase-87.3` when done; do NOT close the parent epic
and do NOT create new beads.

## Context

Phase `sase-87.2` (landed) taught the `%wait` grammar a repeatable `bead=<bead_id>` kwarg: `AgentDirectives.wait_beads`
(`src/sase/xprompt/_directive_types.py:157`) now carries an ordered, deduplicated, validated list of bead IDs, and
bead-only `%wait` occurrences already behave like `time=`-only ones for bare-wait rewriting and name templates. Nothing
downstream consumes the field yet:

- `extract_directives_and_write_meta` (`src/sase/axe/run_agent_directives.py`) builds `AgentInfo` with
  `wait_names`/`wait_identity_deps`/`wait_duration`/`wait_until`/`wait_runners` but ignores `directives.wait_beads`.
- The runner (`src/sase/axe/run_agent_runner.py:289-315`) gates `wait_for_dependencies` on
  `wait_names`/`wait_identity_deps`/`duration`/`wait_until` only.
- `wait_for_dependencies` (`src/sase/axe/run_agent_wait.py:351`) writes `waiting.json` with `waiting_for` (+ optional
  `wait_for_artifacts`/`wait_duration`/`wait_until`) and polls for `ready.json` with a 60s direct-resolution fallback
  (`_waiting_marker_dependencies_resolved`, `run_agent_wait.py:199`).
- The `wait_checks` chop (`src/sase/scripts/sase_chop_wait_checks.py`) skips any marker where
  `not waiting_for and not wait_for_artifacts` (lines 111-115) and resolves the rest through
  `dependency_resolution_status` (`src/sase/core/wait_dependency_resolution/_resolution.py:14`), which consults only
  agent artifacts.

Bead-store access outside a workspace has a precedent: `_canonical_beads_dir_for_project`
(`src/sase/integrations/_mobile_helper_beads.py:213`) resolves a project directory key to its canonical beads dir via
`get_project_beads_dirs_for_project` (`src/sase/bead/workspace.py:27`), and `_open_project_for_beads_dir`
(`_mobile_helper_beads.py:249`) maps a beads dir (all three layouts: `sdd/beads`, `.sase/sdd/beads`, custom) onto the
`BeadProject` Rust read facade. Both the chop (`project_dir.name`) and the runner (`project_name` passed into
`wait_for_dependencies`) hold the same `~/.sase/projects/<key>` directory key that `get_project_beads_dirs_for_project`
expects.

`waiting.json` is projected into `WaitingMarkerWire` (`src/sase/core/agent_scan_wire_markers.py:191`) by selecting known
fields only, so the additive `wait_for_beads` key cannot break the Rust scanner or ACE loaders. Displaying bead waits in
ACE/agent listings is a later phase (`sase-87.5`) and is out of scope here.

## Design

Wait semantics for a bead condition (from the epic design):

- A bead condition holds when the named bead exists in the **waiting agent's own project** bead store with status
  closed. All bead conditions AND all agent/artifact conditions must hold; the `time=` floor still starts only after all
  dependencies resolve.
- **Fail closed**: a missing bead, missing store, or store read error keeps the agent waiting (same posture as an
  unknown agent name). ACE wait cancel/resume remains the escape hatch.
- Resolution is edge-triggered at poll time; a later reopen does not re-park a released agent.
- `resolved_deps` memoization in `waiting.json` stays **agent-only** — bead checks are cheap and monotonic, so they are
  re-evaluated on every poll and never memoized.
- No cross-project bead waits; no bead conditions on `#fork` or family attachment (those code paths build
  `wait_names`/`wait_identity_deps` and are untouched).

### Shared bead-store locator

New module `src/sase/bead/store_locator.py` (promotion, not new behavior):

- `canonical_beads_dir_for_project(project: str) -> Path | None` — moved from `_canonical_beads_dir_for_project`
  (`src/sase/integrations/_mobile_helper_beads.py:213`).
- `open_bead_project_for_beads_dir(beads_dir: Path) -> BeadProject` — moved from `_open_project_for_beads_dir`
  (`_mobile_helper_beads.py:249`), keeping the three-layout dispatch.
- `closed_bead_ids_for_project(project: str) -> frozenset[str] | None` — new convenience used by both resolvers: resolve
  the dir, open the `BeadProject`, and return the IDs of issues with `Status.CLOSED` (one
  `list_issues(statuses=[Status.CLOSED])` read through the Rust facade). Return `None` on a missing store or any read
  error — the fail-closed signal.

Refit `_mobile_helper_beads.py` onto the first two (delete the private copies, import from the new module; callers and
tests that patch `sase.integrations.mobile_helpers.get_project_beads_dirs_for_project` keep working because the locator
continues to call `get_project_beads_dirs_for_project` by that import path only from the new module — update any such
test patch targets if they break).

### Resolution core (stays free of store I/O)

Extend `dependency_resolution_status` (`src/sase/core/wait_dependency_resolution/_resolution.py:14`) with:

- a `wait_beads: Iterable[object] = ()` positional-after-`resolved_deps` (or keyword-only) parameter, and
- a keyword-only `closed_bead_ids: Collection[str] | None = None` prebuilt lookup produced by callers.

Rules: every entry of `wait_beads` must be a non-empty `str`, else `waiting` (defensive, matching the existing non-`str`
handling); when any bead conditions exist and `closed_bead_ids` is `None`, return `waiting` (store unavailable = fail
closed); a bead resolves iff its ID is in `closed_bead_ids`. Bead checks ignore `resolved_deps`. The
`wait_dependency_resolution` package performs no store reads — callers inject the set.

### wait_checks chop

In `src/sase/scripts/sase_chop_wait_checks.py`:

- `_WaitingMarker` gains `project_name: str` (from `project_dir.name` in the scan loop).
- Parse `wait_for_beads` from the marker (non-list coerced to `[]`) and make the marker actionable when **any** of
  `waiting_for`/`wait_for_artifacts`/`wait_for_beads` is non-empty (today's lines 104-115).
- Maintain a per-cycle cache `dict[str, frozenset[str] | None]` keyed by project, filled lazily via
  `closed_bead_ids_for_project` only when a marker of that project actually carries bead waits — one store read per
  project per cycle, zero reads when no bead-wait markers exist.
- Pass `wait_beads` + the project's `closed_bead_ids` into `dependency_resolution_status`; keep writing `ready.json` as
  `{"resolved_deps": waiting_for}` (agent-only memoization). Include the bead IDs in the "Dependencies satisfied" log
  line (e.g. append `beads: ...` when present).

### Runner threading

- `AgentInfo` (`src/sase/axe/run_agent_directives.py:94`) gains `wait_beads: list[str]` (update the keyword construction
  at the `return AgentInfo(...)` site and the `AGENT_INFO` fixture in
  `tests/_axe_run_agent_runner_retry_helpers.py:29`). Populate from `directives.wait_beads` and, when non-empty, write
  `agent_meta["wait_for_beads"]` alongside the existing `wait_for`/`wait_duration`/... fields so later surface phases
  can read it. Bead waits must NOT influence wait-derived agent naming (`wait_name` uses `wait_names` only — already
  true, keep it that way).
- `run_agent_runner.py`: include `bool(info.wait_beads)` in `has_dependency_wait` and pass `wait_beads=info.wait_beads`
  to `wait_for_dependencies`.
- `wait_for_dependencies` (`src/sase/axe/run_agent_wait.py:351`) gains `wait_beads: list[str] | None = None`:
  - the has-dependencies branch triggers when `wait_names or wait_identity_deps or wait_beads`;
  - `waiting.json` gains `"wait_for_beads": [...]` when non-empty (keep `waiting_for` present even if empty — ACE
    loaders and the chop expect the key);
  - the "Waiting for ..." message builds its parts conditionally: `agents: ...` only when `wait_names` is non-empty,
    plus `beads: ...` when bead waits exist (a bead-only wait must not print an empty `agents:` part);
  - `_initial_dependencies_resolved` (`run_agent_wait.py:89`) accepts `wait_beads`, builds `closed_bead_ids` for its own
    `project_name` via the shared locator (lazy import inside the function, matching the file's style), and passes both
    through to `dependency_resolution_status`;
  - `_waiting_marker_dependencies_resolved` (`run_agent_wait.py:199`) re-reads `wait_for_beads` from the marker,
    includes it in the "nothing to wait for" gate, and threads it into the fallback resolution — so the 60s fallback
    re-checks bead closure exactly like the chop.

## Testing

Extend, don't rewrite, the existing suites; bead-less `%wait` behavior, marker shapes, and chop skip-reasons must stay
green.

- **Resolution unit cases** (alongside the existing `dependency_resolution_status` unit coverage in
  `tests/test_axe_chop_wait_checks.py`): closed bead in set → resolved; open/unknown bead → waiting;
  `closed_bead_ids=None` with bead waits → waiting; non-string bead entry → waiting; mixed agent+bead where only one
  side holds → waiting; bead conditions unaffected by `resolved_deps` memoization.
- **Chop scenarios** via `tests/_axe_chop_wait_checks_helpers.py` (extend `make_waiting_agent` with a `wait_for_beads`
  kwarg): bead-only marker with open bead → parked (`unresolved`); closed bead → `ready.json` written; missing bead →
  parked; missing/unresolvable store → parked; mixed agent+bead marker requiring both. Seed real stores with
  `BeadProject.init(...)` + create/close (see `seed_bead_project` in `tests/_mobile_helper_bridge_helpers.py` for the
  pattern) and point the locator at them (monkeypatch the locator's beads-dir resolution, mirroring
  `tests/test_mobile_helper_beads.py:82`).
- **Runner coverage** in `tests/test_run_agent_wait.py` / `tests/test_run_agent_wait_fallback.py`: a bead-only wait
  writes a marker containing `wait_for_beads` and blocks; the fallback releases once the bead closes and stays parked
  while it is open; the "Waiting for" output names beads and omits the empty agents part; `wait_for_beads` appears in
  `agent_meta.json`; bead-less calls behave exactly as before.
- **Mobile helper refit**: existing `tests/test_mobile_helper_beads.py` suite stays green.

Run `just check` (full lint + mypy + tests) before finishing.

## Out of scope

- Emitting `%w(bead=...)` lines from `sase bead work` and the `sase_core_rs` floor bump (phase `sase-87.4`).
- ACE waiting-description/wait-modal display of bead waits and the docs sweep (phase `sase-87.5`).
- Any sase-core (Rust) change; this phase is Python-only and store reads go through the existing facade.
- Cross-project bead waits, TUI affordances for adding bead waits, `%wait` grammar changes.
