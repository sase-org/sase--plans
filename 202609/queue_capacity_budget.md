---
tier: epic
title: Make %queue capacity a per-launch capacity budget
goal: "`%q:N` gives that launch a capacity budget of N that replaces
  `max_running_agents` for its own admission decision, unsatisfiable capacity/weight
  combinations are rejected when they are authored instead of parking forever, and every
  agent node and agent family node shows the capacity it authored.

  "
phases:
  - id: core
    title: Rust admission contract — capacity is the limit
    depends_on: []
    size: medium
    description: "core: rename the persisted capacity field to `queue_capacity`,
      evaluate each waiter against its own admission limit instead of a shared global
      limit, reject unsatisfiable authored capacity at parse time, translate legacy
      `capacity=0` records, and gate all of it behind the sunset flag.

      "
  - id: admission
    title: Python adapters, launcher, and the sunset flag
    depends_on:
      - core
    size: medium
    description: "admission: register the `queue_capacity_budget` sunset flag, pass it
      into the Rust entry points, rebuild against the new core revision, and carry
      `queue_capacity` plus the per-waiter admission limit through the adapters, the
      slot-poll launcher, and the agent-listing wire.

      "
  - id: display
    title: The capacity badge and the live/authored split
    depends_on:
      - admission
    size: medium
    description: "display: add the `cN` capacity badge to agent nodes and agent family
      nodes beside the existing weight badge, accent an over-subscribing budget, move
      the queue ladder and detail pane onto each waiter's own admission limit, and
      restate the wait and epic-approval modals.

      "
  - id: docs
    title: Documentation sweep and the xprompts memory correction
    depends_on:
      - admission
      - display
    size: small
    description: "docs: restate capacity as a per-launch budget across the xprompt, ACE,
      configuration, and runner-slot troubleshooting docs, and correct the stale
      capacity paragraph in the `sase/memory/xprompts.md` reference memory note.

      "
  - id: verify
    title: Live admission and display smoke
    depends_on:
      - display
    size: xsmall
    description:
      "verify: launch real agents against a lowered runner limit to confirm a
      high-capacity launch is admitted, a low-capacity launch drains first, and both
      render the intended badge and accent."
proposed_by: bbugyi200.kellys_mbp.06.f0
create_time: 2026-09-12 10:33:25
status: wip
---

# Plan: Make `%queue` capacity a per-launch capacity budget

## Problem

`%queue(capacity=N)` / `%q:N` reads like "this agent's capacity is N". It is not. Today
it is a _pre-admission threshold_: the agent starts only when already-occupied weighted
load is at most `N`, and the global `max_running_agents` budget applies on top of it. As
`docs/xprompt.md` puts it, "Capacity can make a launch stricter, but it never overrides
`max_running_agents`."

The consequence is a directive that silently does nothing in the case users reach for it
most. With an effective limit of `1` and one running agent, a launch authoring `%q:100`
evaluates `1.0 ≤ 100` → satisfied → contributes nothing, and then parks on the global
budget anyway. The Agents row makes this worse rather than better: it renders
`QUEUED #2/2 ▶1→100` — an obviously-satisfied condition — in parked amethyst, because
`parked` is decided by a `capacity` blocker the row never shows. Three numbers are in
play (occupied load, the authored threshold, the global limit) and the row shows two of
them, neither of which is the one doing the blocking.

## Design

### One rule

> **`%queue(capacity=N)` replaces the effective `max_running_agents` budget with `N` for
> this launch's own admission decision.**

Admission becomes exactly one inequality per waiter:

```
occupied_capacity + effective_weight  ≤  admission_limit

admission_limit = authored capacity, when present
                  effective max_running_agents, otherwise
```

Nothing else moves. Priority, FIFO order, the deference window, lineage claims, weight
inheritance, liveness, and fail-closed behavior are untouched.

This collapses two admission stages into one and deletes the entire class of confusion
above: there is now a single limit per waiter, and it is the number the author wrote.

### What the rule buys

- **`%q:100` under a global limit of `1` admits.** `1.0 + 1.0 ≤ 100`. This is the case
  that motivated the change.
- **Strictness survives, and reads better.** `%q:2` with the default weight means "start
  only while at most one other unit is busy", regardless of a global limit of 10.
- **`%q:1` is the drain barrier**, and a clearer one than `%q:0` ever was: "my budget is
  one unit, so nobody else may hold any." A launch that wants to run alone asks for
  exactly enough room for itself.
- **Nothing can bypass accounting.** An admitted over-subscriber holds an ordinary claim
  of its weight, so global occupied capacity simply rises above `max_running_agents`.
  Free capacity already clamps at zero, so every other waiter keeps waiting on its own
  limit. Over-subscription is opt-in, per launch, and never reserves anything against
  anyone else.

### What the rule costs, and what we do about it

`capacity=0` becomes unsatisfiable: no positive weight fits in a budget of zero. Rather
than let that deadlock, it is rejected where it is authored and translated where it was
already persisted.

- **Authoring `capacity=0` is a parse error** naming its replacement. A budget must be
  at least `1`.
- **A persisted `queue_capacity: 0`** — only reachable from a launch that parked before
  this change landed — is read as `admission_limit = effective_weight`, which is the
  exact translation of "drain to zero, then run", and emits a `legacy-capacity-zero`
  diagnostic. An in-flight waiter is never wedged by the upgrade.

We also reject a second unsatisfiable combination that today parks forever in silence:
**an authored `weight` greater than an authored `capacity`** on the same launch. Both
numbers are present at parse time, so the failure is static and belongs at the keyboard,
not in the queue.

### Capacity stays an integer

Capacity is now literally a per-launch stand-in for `max_running_agents`, which is an
integer configuration field, so it keeps the integer domain — narrowed from non-negative
to positive. Weight remains the fractional quantity; `w=` already covers fine-grained
claims, and a fractional per-launch budget has no demonstrated use. Reopen this if a
real case appears for a budget that is not a whole number of standard agents.

### Rejected alternative: a configured ceiling on authored capacity

A `runner_slots.max_authored_capacity` knob would stop `%q:1000` from over-subscribing
the host. It is rejected for now: it reintroduces exactly the
second-limit-you-cannot-see problem this epic exists to delete, and over-subscription is
already explicit per launch and loud on screen (gold badge on the row, red `C/L` in the
header). Reopen if real over-subscription incidents appear that the operator did not
intend.

### Display: rows carry authored intent, chrome carries live state

The screenshot's `▶1→100` mixed a live number and an authored number into one token next
to a verdict driven by a third number that was not shown. The fix is a clean split:

- **Agent rows carry what the launch authored**: `w2`, `cN`, `p20`.
- **Header, queue ladder, and detail pane carry what is live**: occupied capacity, free
  capacity, rank, blockers.

So the new capacity badge is a sibling of the existing weight badge, in the same place,
with the same suppression rules, and the `▶occupied→threshold` suffix on `QUEUED` rows
goes away entirely — the authored limit is now two characters to the left in the badge,
and live occupancy is already in the header, the ladder, and the detail pane.

```
sase-zz.3  w2 c100  (QUEUED #2/2 p20)
```

The badge uses `c` in the same dim prefix style as `w`, with the number in `#87AFD7` —
the same `87-` family as the weight badge's `#87D7D7` and the detail-pane label's
`#87D7FF`, so the three read as one system, and far enough from the QUEUED cornflower
`#5F87FF` and the parked amethyst `#AF87FF` that it never reads as a status.

When an authored capacity **exceeds the current effective global limit**, the number
takes the header's capacity-pressure gold `#FFD700` instead. That single accent is the
whole feature at a glance: this launch is authorized to push the host past its budget.
In the motivating screenshot the user would have seen `c100` in gold and known
immediately that the directive had been honored. When the global limit is unknown
(`—/—`), the badge stays quiet rather than guessing.

### Naming

The persisted and wire spelling is `wait_runners`, two renames stale already, and under
the new semantics it is actively wrong — it is neither a wait nor a count of runners. It
becomes `queue_capacity` / `queue_capacity_explicit`, matching the public `capacity=`
keyword. Readers accept the legacy spelling so records written by any older build still
load, which also means the rename can land in the same core release as the semantics
without an ordering hazard.

### Flag

This is user-reaching behavior whose old branch must stay reachable while prompts
migrate, so it ships behind a **`sunset`** flag, `queue_capacity_budget`, default on.
Off is today's pre-admission-threshold behavior, including `capacity=0` as a drain and
the `▶occupied→threshold` row suffix. Removing the flag deletes the Off branch.

## Phases

### `core`: Rust admission contract — capacity is the limit

Worker: open the core checkout with `sase repo open sase-core` and work there.
Everything in this phase is in `crates/sase_core/`.

`runner_capacity.rs`:

- Rename `RunnerCapacityRecordWire.wait_runners` / `wait_runners_explicit` to
  `queue_capacity` / `queue_capacity_explicit`, each with
  `#[serde(alias = "wait_runners")]` / `#[serde(alias = "wait_runners_explicit")]` so
  older records and older Python still deserialize. Do the same for
  `RunnerCapacityWaiterWire.wait_runners`.
- Add `feature_flags: Vec<String>` to `RunnerCapacityRequestWire`, matching the existing
  convention where Python passes enabled flag keys into Rust entry points (see
  `agent_launch_facade.py` and `launch_feature_flag_keys()`). The new behavior is active
  only when `queue_capacity_budget` is present.
- Compute a per-waiter `admission_limit` and evaluate `weight-exceeds-limit`,
  `insufficient-capacity`, and `capacity-overflow` against it rather than against
  `request.effective_limit`. Apply the same limit in `waiter_capacity_shortfall` and in
  `build_candidate_decision`.
- Drop the separate `capacity-condition` blocker and `wait_capacity_shortfall` when the
  flag is on; keep both on the Off branch.
- Translate a persisted `queue_capacity == Some(0)` to
  `admission_limit = effective weight` and push a `legacy-capacity-zero` diagnostic.
- Add `admission_limit: f64` to `RunnerCapacityWaiterWire` and
  `admission_limit: Option<f64>` to `RunnerCapacityBlockerWire`, so display never has to
  re-derive which limit blocked a waiter.
- Leave `RunnerCapacitySnapshotWire.effective_limit`, `occupied_capacity`, `claims`,
  `occupied_lanes`, `first_eligible_artifact_dir`, priority/FIFO ordering, and the
  deference window exactly as they are. `occupied_capacity` is now allowed to exceed
  `effective_limit`; that is the honest signal, not an error.
- Bump `RUNNER_CAPACITY_POLICY_SCHEMA_VERSION` from `3` to `4`.

`queue_directive.rs`:

- Rename the `capacity` field's persisted spelling to match, keeping the existing
  `runners=` migration error untouched.
- With the flag on, reject `capacity=0` with code `invalid-queue-capacity-zero` and a
  message that names the replacement: capacity is this launch's capacity budget and must
  be at least 1; use `%q:1` to run alone.
- With the flag on, reject an authored `weight` greater than an authored `capacity` on
  the same launch with code `queue-weight-exceeds-capacity`, explaining that the launch
  could never be admitted.
- Keep the u32 upper bound and every existing duplicate-field, colon-form, and
  unknown-keyword rule.

Tests: extend the in-file Rust tests to cover, in **both** flag states, admission above
and below the global limit, `%q:1` as a drain barrier, the legacy-zero translation, the
two new parse rejections, and the `wait_runners` deserialization alias. Land the change
and record its commit SHA on the phase bead so the next phase can pin it.

### `admission`: Python adapters, launcher, and the sunset flag

- Create the flag with `sase flag new queue_capacity_budget -k sunset`, authoring
  `--when-enabled`, `--when-disabled`, and `--remove-when` from the Flag section above.
  This phase is explicitly authorized to create that flag bead; `sase flag new` is the
  only supported way to add a flag, so do not record it as a proposed follow-up and do
  not use `sase bead create` or `/sase_new_task`. Paste the printed registry entry into
  `src/sase/feature_flags/registry.py`.
- Add the key to `launch_feature_flag_keys()` in `src/sase/xprompt/queue_directive.py`
  and pass the enabled keys into the runner-capacity request built in
  `src/sase/core/runner_slots/_admission.py`.
- Bump `sase-core-revision.txt` to the SHA recorded by `core`, update the pinned schema
  version and probe expectations in `tools/validate_sase_core_rs`, and run
  `just install` so the workspace virtualenv builds the new extension. Leave the
  published `sase-core-rs` window in `pyproject.toml` alone — dev builds ignore it and
  the release-branch reconciler ratchets it at release time.
- Rename `wait_runners` / `wait_runners_explicit` to `queue_capacity` /
  `queue_capacity_explicit` through the Python surfaces that mirror the wire:
  `_admission.py`, `agent_scan_wire_markers.py`, the `agent_launch_wire_*` modules,
  `run_agent_wait_slots.py` and its marker writer, `monitor/continuation_delivery.py`,
  `integrations/agent_list_entries.py` and `_agent_list_entry_*`, `agents/cli_list.py`,
  `ops/commands/agent.py`, the ACE agent-state and meta-enrichment loaders, and
  `config/sase.schema.json`. Every reader accepts the legacy spelling; every writer
  emits the new one.
- Update `validate_queue_capacity()` in `queue_directive.py` for the positive-integer
  domain and surface the new parse error codes.
- Update the two Python helpers that duplicate admission logic with the old threshold
  shape: `may_start()` and `runner_slot_queue_display_key()` in
  `src/sase/core/runner_slots/_admission.py`. Prefer deleting a helper over keeping a
  second, divergent copy of a rule Rust already owns; if a caller still needs it,
  re-express it in terms of the waiter's own admission limit.
- Confirm that a serial continuation keeps inheriting its parent's authored capacity
  through `continuation_delivery.py`'s reconstructed `%queue` prefix — same launch, same
  budget.

Tests: both flag states across `tests/test_run_agent_runner_slot_capacity*.py`,
`tests/test_capacity_gate_to_admission.py`, `tests/test_capacity_snapshot_parity.py`,
`tests/test_runner_slots_queue.py`, and `tests/test_agent_wait_live.py`. Add the case
this epic exists for: effective limit `1`, one live claim of `1.0`, a waiter authoring
`capacity=100` — admitted with the flag on, parked with it off.

### `display`: the capacity badge and the live/authored split

- Add `src/sase/ace/tui/widgets/_queue_capacity_badge.py` next to
  `_queue_weight_badge.py`, exposing `append_queue_capacity_badge()` and
  `append_agent_queue_capacity_badge()`. Reuse the weight badge's suppression rule
  verbatim so the two stay in lockstep: no badge on clan containers, proc shells, gates,
  monitors, or serial child rows. That is what puts the badge on the agent node and the
  agent family node — a serial family shares one claim, so the family's authored
  capacity belongs on the family node, while a live parallel family member holds its own
  claim and gets its own badge.
- Render it in `_agent_list_render_agent_status.py` immediately after the weight badge:
  dim `c` prefix, number in `#87AFD7`, or `#FFD700` when the authored capacity exceeds
  the current effective global limit, and quiet when that limit is unknown. Show it
  whenever a capacity is authored, in every status — not only while `QUEUED`.
- Remove the `▶occupied→threshold` suffix from the `QUEUED` branch when the flag is on;
  keep it on the Off branch. The priority suffix is unchanged.
- Add the matching `Capacity:` field to the agent detail pane beside `Weight:` in
  `prompt_panel/_agent_display_header_metadata.py`, using the same label/value styles.
- In `prompt_panel/_agent_queue_section.py`, replace the `≤N` threshold column with the
  shared `cN` badge, and make the per-entry `needs X · Y free` detail derive `Y` from
  that waiter's own `admission_limit` rather than the global limit.
- Point the detail pane's `capacity: N/M in use` denominator at the row's admission
  limit, so an over-subscribing row honestly shows its own budget.
- Leave the header `C/L` prefix alone. It is global by definition and already escalates
  to red once occupied capacity reaches the limit, which is exactly the right signal
  when an over-subscriber has been admitted.
- Restate the wait modal: the Capacity field's help becomes this launch's capacity
  budget, replacing `max_running_agents`; `wait_spec_label()` in
  `actions/agents/_wait_helpers.py` changes from `waiting for weighted load ≤ N` to a
  budget phrasing; `wait_modal_values.validate_capacity_token()` picks up the new domain
  and error text.
- Restate the epic approval capacity control in `modals/approve_options_modal.py`,
  including `_format_capacity_display()`'s `capacity == 0` special case, which no longer
  has a meaning to display.

Tests: `tests/ace/tui/widgets/test_agent_info_panel_state.py`, the wait-modal and
approve-modal suites, and the queue-section tests. Refresh the PNG goldens with
`just test-visual --sase-update-visual-snapshots` for
`test_ace_png_snapshots_agents*.py` and `test_ace_png_snapshots_models_panel_edit.py`,
and eyeball the diffs in `.pytest_cache/sase-visual/` before accepting them — the badge
hue and the gold accent are the point of this phase.

### `docs`: documentation sweep and the xprompts memory correction

Restate capacity as a per-launch budget everywhere it is currently described as a
threshold:

- `docs/xprompt.md` — the `%queue` admission section, including the sentence "Capacity
  can make a launch stricter, but it never overrides `max_running_agents`", the
  `capacity=0` drain paragraph, and the four-claims-of-0.25 example.
- `docs/ace.md` — the Queued status description and its `▶7→0` example, the wait modal's
  Capacity field description, the new badge, and the Launch Control paragraph asserting
  that an explicit capacity "cannot bypass the global capacity budget".
- `docs/configuration.md` — the `max_running_agents` and `runner_slots` entries.
- `docs/troubleshooting/runner-slots.md` — the admission description, the `wait_runners`
  diagnosis step (now `queue_capacity`, with the legacy spelling named as still
  readable), and the ladder's `≤N` column.

**Memory change.** `sase/memory/xprompts.md` currently states that capacity is checked
"before admission, excluding the candidate's weight", that "the global
`max_running_agents` budget still applies", and that "`capacity=0` is a true drain". All
three are false after this epic. Use the `/sase_memory_write` skill, rewrite that
paragraph in place — do not append a caveat — to state the one rule, the
positive-integer domain, and `%q:1` as the drain barrier, then run `sase memory init` to
regenerate `AGENTS.md` and the provider shims. This is the only memory file this epic
touches.

### `verify`: live admission and display smoke

With the flag in its default state, use Launch Control's `Ctrl+R` to set a temporary
`max_running_agents` of `1`, then:

1. Launch a normal agent so the single unit is occupied.
2. Launch a second agent authoring `%q:100` and confirm it is admitted rather than
   queued, that the header shows `2.0/1.0` in red, and that its row shows `c100` in
   gold.
3. Launch a third agent authoring `%q:1` and confirm it parks until both drain, and that
   its row shows `c1` in the quiet hue.
4. Confirm a prompt authoring `%q:0` is rejected at launch with the migration message,
   and that a prompt authoring `%q(capacity=1, w=2)` is rejected as unsatisfiable.

Clear the temporary limit afterwards. Record what was observed on the phase bead.

## Verification

Every phase in the `sase` repo runs `just check` before finishing. The `docs` phase, as
the last phase to touch the tree, runs `just check-full` through the `/sase_monitor`
skill with the `TESTING` / `TESTED` status pair. The `core` phase runs `cargo test` and
`cargo clippy` in the core checkout.
