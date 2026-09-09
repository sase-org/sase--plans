---
tier: epic
title: Raise an EpicResume gate when a failed phase agent stalls an epic
goal: "When an epic's phase agent fails and the epic stops making progress, SASE raises
  exactly one human-only EpicResume gate whose single option relaunches that epic with
  `sase bead work <epic_bead_id> --yes-to-all`, and reconciliation cancels the gate as
  soon as the epic resumes or closes.

  "
phases:
  - id: policy
    title: Epic stall detection policy
    depends_on: []
    size: medium
    description:
      "policy: add the pure stalled-epic predicate, its typed clan-member input records,
      and the fingerprint helper the gate lane keys on."
  - id: launch
    title: Detached epic resume launch
    depends_on: []
    size: small
    description:
      "launch: add the leased, detached submission helper that runs `sase bead work
      <epic_id> --yes-to-all` and reuses an in-flight resume instead of
      double-launching."
  - id: gate
    title: The EpicResume gate kind
    depends_on:
      - launch
    size: medium
    description:
      "gate: register the EpicResume gate kind end to end — request spec, preview,
      side-effect-free command, trusted response translation, kind validation, adapter
      routing, and notification classification."
  - id: chop
    title: The epic_resume chop and its feature flag
    depends_on:
      - policy
      - gate
    size: medium
    description:
      "chop: add the checks-lane chop that detects stalled epics, raises and reconciles
      one gate per epic behind a beta feature flag, plus its config knobs and schema."
  - id: docs
    title: User-facing documentation
    depends_on:
      - chop
    size: small
    description:
      "docs: document the gate, the chop, the feature flag, and the config knobs across
      the notification, AXE, bead, and configuration docs."
proposed_by: bbugyi200.athena.05e
status: done
bead_id: sase-p4
create_time: 2026-09-09 19:50:13
---

- **PROMPT:**
  [prompts/202608/epic_resume_gate.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/epic_resume_gate.md)
- **BEAD:**
  [sase-p4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-p4/README.md)

# Plan: Raise an EpicResume gate when a failed phase agent stalls an epic

## Why

An epic launched by `sase bead work` runs its phases as one agent clan
(`%clan(<epic_id>, tribe=epic)`). Every later phase waits on its predecessor by agent
name (`%w:<agent>`) and by phase bead (`%w(bead=<id>)`), and the land agent waits on all
of them. A phase agent that reaches outcome `failed` therefore never satisfies those
waits: its siblings sit in `WAITING` forever and the epic makes no further progress.
`sase_chop_wait_checks` already observes this exact condition — it logs
`Terminal dependency still blocks waiter` — but nothing acts on it.

Today the owner notices the stall by eye in ACE (a clan row like
`(FAILED) [W8 F1] sase-p1`) and types `sase bead work sase-p1 -Y` by hand. The failure
is usually transient, and sometimes the agent actually finished its work and only the
runtime marked it failed, so the same one-line remedy applies either way. This epic
turns that manual observation into a durable gate: the host detects the stall, and one
keypress relaunches the epic.

The gate is deliberately a _single-option_ gate, matching `BeadStaleCleanup`. Declining
is dismissing the notification; there is no second command worth authoring for "leave it
stalled".

## Design constraints this epic must hold

- **One gate per epic, and never a duplicate.** Lane state is keyed by
  `(project, epic_id)` and carries the fingerprint of the stall it already offered.
- **A declined stall stays declined.** Once a gate for a given fingerprint is answered
  or cancelled, the same fingerprint must never raise another gate. Only a _new_ failure
  — a different failed-member set or a newer clan generation — re-gates. This is the one
  place this epic deliberately diverges from `bead_stale_cleanup`, which regenerates
  gates for an unchanged roster.
- **Newest clan generation only.** Re-running `sase bead work` starts clan generation
  N+1 and leaves generation N's failed records on disk forever. Detection that ignored
  generation would re-gate an epic that is already running again.
- **Failure means outcome `failed`.** `killed` and `stopped` are deliberate user
  actions, and `build_agent_list_entry` already renders only `failed` as `FAILED`, so
  this trigger matches what the owner sees. `epic_launch_failed` belongs to the launch
  proc rather than to a clan member and is out of scope.
- **No gate while a relaunch is in flight.** Between submitting the resume proc and
  generation N+1 appearing on disk, generation N still looks stalled. Defer, exactly as
  `bead_task_triage` defers a bead whose detached launch is still in flight.

## Boundary note

Under `sase/memory/rust_core_backend_boundary`, shared backend behavior belongs in
`../sase-core/crates/sase_core`. This epic stays in Python, matching the sibling
gate-triage predicate `sase.bead.task_triage_policy` and every existing gate chop: the
notification-gate machinery, the chop runtime, and the lane-state reconciliation are all
Python here, and the new predicate is a pure function over data the existing Rust-backed
facades (`agent_scan_facade`, `bead_read_facade`) already return. It introduces no new
storage, wire contract, or bead mutation. Should another frontend later need it, a pure
predicate is cheap to promote.

## Epic stall detection policy

Add `src/sase/bead/epic_stall_policy.py`, modeled on
`src/sase/bead/task_triage_policy.py`: pure functions of their arguments, with the
caller owning the clock, the configured settle window, and every filesystem read.
Nothing in this phase imports the chop, the gate, or the scanner.

Define frozen input records:

- `EpicClanMember` — `agent_name`, `bead_id`, `artifact_dir`, `timestamp`, `outcome`
  (`str | None`), `has_done_marker`, `is_live` (the agent is
  `RUNNING`/`STARTING`/`QUEUED`), and `finished_at` (`datetime | None`).
- `EpicClanSnapshot` — `project`, `epic_id`, `clan_generation`, and the member tuple.

Define the outputs:

- `EpicStall` — `project`, `epic_id`, `clan_generation`, the ordered failed members, the
  ordered waiting members, and `stalled_since` (the newest failure's `finished_at`).
- `stalled_epic(snapshot, *, epic_open, now, settle_seconds) -> EpicStall | None`,
  returning a stall only when **all** of these hold:
  1. `epic_open` is true (the caller resolved the epic bead's status; a closed epic is
     done).
  2. At least one member has `outcome == "failed"`.
  3. No member is live.
  4. The newest failure is at least `settle_seconds` old, so a handoff race or a fast
     retry does not gate.
- `epic_stall_fingerprint(stall) -> str`, a `sha256_bytes(canonical_json_bytes(...))`
  digest over `project`, `epic_id`, `clan_generation`, and the sorted
  `(agent_name, artifact_dir)` pairs of the failed members. `stalled_since` and any
  wall-clock value are deliberately excluded: including them would make every tick a new
  fingerprint and defeat the dedupe.

Note that condition 3 intentionally does not inspect _why_ a member is waiting. A clan
with no live member and at least one failed member cannot advance, because every wait
edge in `render_multi_prompt` points at a clan sibling. A clan whose only non-terminal
members are waiting on a still-running sibling fails condition 3 and is not a stall.

Also export `latest_generation_snapshot(snapshots)`, which picks the newest
`clan_generation` for a given `(project, epic_id)` so callers cannot accidentally
evaluate a superseded generation.

Tests (`tests/test_epic_stall_policy.py`) must cover: a plain stall; a stall suppressed
by a live member; a stall suppressed by a closed epic; a stall suppressed inside the
settle window; `killed` and `stopped` members never triggering; an all-terminal clan
whose land agent failed still counting as a stall while the epic bead is open;
fingerprint stability across ticks; fingerprint change when a second member fails; and
generation selection preferring the newest generation.

## Detached epic resume launch

Add `src/sase/bead/epic_resume_launch.py`, mirroring `src/sase/bead/task_launch.py`
closely enough that a reader of one recognizes the other.

- `build_epic_resume_argv(epic_id)` returns
  `["sase", "bead", "work", epic_id, "--yes-to-all"]`. The long flag is the argv
  spelling of the owner's `-Y`; these commands run without a shell.
- `submit_epic_resume_task(epic_id, *, project, origin) -> Proc` takes the
  `epic-resume-submit` file lock under `procs_dir()`, returns any active resume proc for
  the same epic instead of submitting a second one, then acquires an operational lease
  (`workflow="epic-resume"`, `holder=f"epic-resume:{epic_id}"`) and submits through
  `submit_via_lease` with `label=f"Epic resume · {epic_id}"`, `cwd=lease.checkout_dir`,
  the resolved `project`, and tags `("epic", "resume")`.
- `active_epic_resume(epic_id) -> Proc | None` is public, because the chop needs it to
  defer while a relaunch is in flight. Model it on
  `_active_task_launch`/`_active_epic_launch_for_plan`.
- `epic_resume_origin_from_gate_source` re-exports the shared
  `epic_launch_origin_from_gate_source` mapping so a gate answered in ACE, Telegram, or
  AXE attributes its launch correctly.

`sase bead work` resolves its project from the working directory, which is exactly why
the lease supplies `cwd`; do not pass a project flag.

Tests (`tests/test_epic_resume_launch.py`) cover argv construction, single-submission
under concurrent calls, reuse of an in-flight resume, and origin mapping.

## The EpicResume gate kind

Register a first-party gate kind. Follow `BeadStaleCleanup`'s module split exactly,
because gate validation rebuilds every persisted spec from these helpers and compares
byte for byte, so each helper must stay a pure function of its arguments.

New modules under `src/sase/bead/`:

- `_epic_resume_gate_spec.py` — constants and `build_epic_resume_gate_spec(...)`. Kind
  `epic_resume`; continuation mode `epic_resume`; query `resume`; primary branch
  `("resume",)`; preview path `epic.md`; command path `commands/resume`. Presentation:
  `sender: "bead"`, `icon: "🔁"`, `title` from `bounded_gate_title` reading
  `Resume stalled epic <epic_id>`, one note naming the epic and its failed agents,
  `tags: ["bead", "epic", "resume"]`, `panel: "beads"`, `panel_icon: "◈"` (reusing the
  existing Beads tab rather than introducing a new panel), `files`/`preview` pointing at
  `epic.md`. Payload carries `project`, `epic_id`, `epic_title`, `clan_generation`,
  `failed_members`, `waiting_members`, `remaining_phase_count`, `resume_argv`, and
  `stalled_since`. The single option is `resume` — label `Resume epic`, icon `▶️`,
  `feedback: "optional"` — with a `result_schema` requiring
  `{"action": {"const": "resume"}}`. `auto: false`; no `gate_timeout_seconds`, because
  chop reconciliation owns cancellation. Add the `gate_command_entrypoint`-decorated
  `execute_epic_resume_gate_command`, which rejects any non-empty command input and
  prints `{"action": "resume"}` — side-effect free, like every other first-party gate
  command.
- `_epic_resume_gate_preview.py` — the Markdown preview and the presentation note. The
  preview must state the epic id and title, each failed agent with its phase bead and
  finish time, the count of phases still waiting, and the exact argv that the option
  will run. The reviewer decides from this text, so write it for them.
- `_epic_resume_gate_response.py` —
  `translate_epic_resume_response(bundle_path, response)` returning a trusted
  `EpicResumeResponse` (`project`, `epic_id`, `action`, `feedback`, `source`) read from
  the persisted request payload rather than from anything the client sent.
- `_epic_resume_gate_actions.py` — `resume_stalled_epic(decision, *, origin)`, which
  re-checks `decision.action == "resume"` and raises
  `GateError("invalid_epic_resume_action", ...)` otherwise, then delegates to
  `submit_epic_resume_task`. Also
  `cancel_epic_resume(project, epic_id, *, reason, source)` for the chop, modeled on
  `cancel_task_triage`.
- `epic_resume_gate.py` — the public facade re-exporting the four private modules,
  mirroring `stale_cleanup_gate.py`.

Wiring:

- `src/sase/notification_gates/kind_validation/epic_resume_payload.py` and
  `epic_resume.py`, plus their exports in that package's `__init__.py`, following
  `bead_stale_cleanup*`. Validation must rebuild the option spec, the command script,
  and the preview from the spec helpers and reject any drift.
- `src/sase/notification_gates/validation.py` — import and dispatch
  `validate_epic_resume_spec`.
- `src/sase/notification_gates/adapters.py` — a
  `GateAdapter(kind="epic_resume", display_title="Epic Resume", action="EpicResume", pending_action_kind="epic_resume", sender="bead", request_filename="request.json", response_filename="response.json", legacy_directory_key="bundle_path", auto_policy="forbidden", neutral_only=True, default_feedback="optional", generic_form=True)`
  entry, and an `apply_side_effects` branch that translates the response, calls
  `resume_stalled_epic`, and writes the returned proc id back into `response.json` under
  `epic_resume_task_id` via `atomic_write_json` — the same shape `task_triage` uses for
  `task_launch_task_id`.
- `src/sase/notifications/priority.py` — add `"EpicResume"` to `_PRIORITY_ACTIONS`.
- `src/sase/notification_gates/debug.py` — add `"EpicResume": "🔁"` to the action icon
  map.

`generic_form=True` means ACE, mobile, and the headless CLI render and answer this gate
through the shared generic path, so no TUI work is required. Confirm that by running the
existing kind-parametrized suites (`tests/test_notification_gates.py`,
`tests/test_mobile_notifications_bridge.py`), which enumerate `registered_gate_kinds()`
and will pick the new kind up automatically.

Add `tests/test_epic_resume_gate.py` covering spec construction, byte-for-byte
validation of a round-tripped bundle, rejection of a tampered option/command/preview,
the command's empty-input contract, response translation, and `apply_side_effects`
submitting exactly one resume proc.

## The epic_resume chop and its feature flag

Create the feature flag first, from within this phase, so the registry entry and its
reader land together:

```bash
sase flag new epic_resume_gate -k beta -d 'Opt-in beta: the epic_resume chop raises an EpicResume gate when a failed phase agent stalls an epic.' -z small
```

`sase flag new` is the only sanctioned way to add a flag, and it files the dedicated
`flag` removal bead itself — that bead creation is required by `sase/memory/sase_flags`,
not discovered follow-up work, so it does not conflict with the phase-worker prohibition
on creating beads. Paste the printed registry entry into
`src/sase/feature_flags/registry.py`. The flag is `beta` and defaults to `false` because
a detector over live agent state needs to soak against real epics before it starts
producing notifications unprompted; the owner enables it with one command after this
epic lands. Per the flags memory, both branches need tests: flag off means the chop
makes no gate and says so in its summary reason; flag on means it gates.

Add `src/sase/scripts/sase_chop_epic_resume.py`, structured like
`sase_chop_bead_stale_cleanup.py`: a `@builtin_chop("epic_resume")` entry that takes a
`file_lock` on `epic_resume.lock` in the chop state dir and reconciles under
`epic_resume.json`.

Each pass:

1. Return early with reason `dry_run` when `runtime.context.dry_run`, and with reason
   `flag_disabled` when `epic_resume_gate` is off.
2. Load the enabled-project inventory through
   `sase.scripts._bead_gate_projects.enabled_project_stores` (update that module's
   docstring, which currently names only the two existing bead-gate chops).
3. Take one `scan_agent_artifacts` snapshot restricted to `ace-run` records — the
   options `sase_chop_bead_claim_checks` uses are the right starting point, minus the
   exclusions this chop needs. Group records into `EpicClanSnapshot`s by
   `(project_name, agent_clan, agent_clan_generation)` for records whose `clan_tribe` is
   `epic`, keeping only the newest generation per `(project, clan)` via
   `latest_generation_snapshot`.
4. For each candidate clan, resolve the epic bead from that project's store to get
   `epic_open`, `epic_title`, and the count of phase beads not yet closed. A store that
   cannot be read skips that project with a warning and leaves its pending gates
   untouched — never sweep on an incomplete inventory.
5. Evaluate `stalled_epic(...)` with the configured settle window.
6. Skip any epic with an active resume proc (`active_epic_resume`), reason
   `resume_in_flight`.
7. Reconcile lane state per `(project, epic_id)`:
   - Current fingerprint equals the recorded `settled_fingerprint` → do nothing (the
     owner already answered or dismissed this exact stall).
   - Current fingerprint equals the recorded pending fingerprint and the gate is still
     `pending` → do nothing, reason `unchanged_stall`.
   - Gate pending for a different fingerprint → cancel it (`epic_stall_changed`) and
     raise a new one at `generation + 1`.
   - Gate no longer pending → record its fingerprint as `settled_fingerprint` and do not
     re-raise.
   - No stall, or the epic bead is closed, or a live member reappeared → cancel any
     pending gate (`epic_resumed` / `epic_closed`) and drop the entry. Prune entries
     whose epic no longer exists so lane state stays bounded.
8. Emit a `runtime.emit_summary` with `gated`, `canceled`, `skipped`, `stalled`,
   `epics`, `projects`, and a `reason`.

Request ids are deterministic: `epic-resume-{fingerprint[:12]}-g{generation}`, so a
crashed pass re-derives the same id instead of orphaning a bundle.

Register the chop in `src/sase/default_config.yml` under the `checks` lumberjack
(five-minute interval) with `timeout: "2m"`, next to `bead_task_triage` — the other
gate-raising chop. Gate creation is not latency-sensitive, and a five-minute cadence
composes with the settle window instead of racing it. Write the `description` in the
same voice as its neighbors: one headline line and a paragraph stating what it scans,
what it defers on, and when it cancels.

Add config knobs under a new `bead.epic_resume` section: `settle_seconds` (default 120)
and `enabled` is _not_ one of them — the feature flag owns on/off during the beta. Add
the accessor alongside the `task_triage` accessors in `src/sase/bead/config.py` with the
same defensive fallback-to-default behavior, document the key inline in
`default_config.yml`, and add it to `src/sase/config/sase.schema.json`.

Tests (`tests/test_axe_chop_epic_resume.py`, with fixtures in
`tests/_axe_chop_epic_resume_helpers.py` following the
`_axe_chop_bead_stale_cleanup_helpers.py` pattern) must cover: flag off makes no gate; a
stall raises exactly one gate; a second tick with an unchanged stall makes no second
gate; an answered or cancelled gate is never re-raised for the same fingerprint; a new
failure re-gates; a resumed epic (live member in generation N+1) cancels the pending
gate; a closed epic cancels it; an in-flight resume proc defers; an unreadable project
store leaves other projects working and pending gates untouched; and lane state is
pruned for vanished epics.

## User-facing documentation

Update the docs that already enumerate gate kinds and chops, matching each file's
existing table and prose conventions:

- `docs/notifications.md` — add `EpicResume` to the gate-action lists (the Panel row,
  the privileged-action enumerations, the `action` field table, and the
  kind/action/producer table), and add a short section describing when the gate appears,
  that its single option runs `sase bead work <epic_id> --yes-to-all` as a detached
  leased proc, that dismissing it declines the stall until a _new_ failure occurs, and
  that `sase axe chop run epic_resume` raises or refreshes it on demand.
- `docs/axe.md` — add `epic_resume` to the chop table and a prose subsection describing
  detection, the settle window, deferral while a relaunch is in flight, and the
  cancellation conditions.
- `docs/beads.md` — a short cross-reference from the epic-work section explaining that a
  failed phase agent no longer requires the owner to notice the stall by hand.
- `docs/configuration.md` — document `bead.epic_resume.settle_seconds` and the
  `epic_resume_gate` feature flag, including how to enable the beta.

Do not hand-edit `CHANGELOG.md`; release-please owns it.

## Verification

Every phase runs `just install` first (workspaces are ephemeral), then `just check`
before reporting. The `chop` and `gate` phases touch shared registries and generated
config surfaces, so they must run `just check-full` through `/sase_monitor` with a
`--next` action rather than inline.

New public symbols must be reachable, or Symvision will fail the build; read
`sase/memory/symvision.md` with `/sase_memory_read` before pragmatizing anything.

Finally, exercise the gate end to end by hand once the flag is on: stall a throwaway
epic by killing a phase agent so it settles as `failed`, run
`sase axe chop run epic_resume`, confirm the notification and its preview, answer it,
and confirm a new clan generation starts.
