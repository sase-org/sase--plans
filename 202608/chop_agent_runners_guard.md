---
tier: epic
title: Add an agent_runners chop guard and idle-gate bugyi_chop_ci_watch
goal: "A declarative `inhibit_if: {agent_runners: {max: N}}` chop guard exists end to
  end, and `bugyi_chop_ci_watch` uses it so the chop only runs while no SASE agent holds
  a runner slot.

  "
phases:
  - id: core-guard
    title: Rust agent_runners guard provider
    depends_on: []
    size: medium
    description: "core-guard: add the `agent_runners` guard variant, the
      `holds_runner_slot` agent snapshot field, the decision and validation logic, and
      config-authority acceptance in `../sase-core`.

      "
  - id: host-guard
    title: Host snapshot, schema, and docs for agent_runners
    depends_on:
      - core-guard
    size: medium
    description: "host-guard: teach the Python preflight host to build runner-slot agent
      snapshots for the new provider, accept it in `sase.schema.json`, document it, and
      cover it with tests including count parity.

      "
  - id: guard-cadence
    title: Guard skips stop consuming run_every cadence
    depends_on: []
    size: small
    description: "guard-cadence: stop advancing a chop's `run_every` clock when the skip
      came from an `inhibit_if` guard rather than from the configured trigger, so a
      guarded chop retries on the next tick.

      "
  - id: ci-watch-idle
    title: Enable the idle guard on bugyi_chop_ci_watch
    depends_on:
      - host-guard
    size: xsmall
    description:
      "ci-watch-idle: add `inhibit_if: {agent_runners: {max: 0}}` to the `ci_watch` chop
      in the chezmoi-managed config, refresh its description body, and verify the live
      runtime accepts and honors it."
proposed_by: bbugyi200.athena.yx
create_time: 2026-08-12 15:59:53
status: wip
---

- **PROMPT:**
  [prompts/202608/chop_agent_runners_guard.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/chop_agent_runners_guard.md)

# Plan: Add an `agent_runners` chop guard and idle-gate `bugyi_chop_ci_watch`

## Problem

`bugyi_chop_ci_watch` should only do its work when the machine is idle — no SASE agents
running. Today AXE has no way to express that. The chop's own design already reaches for
this idea twice, in both cases with a narrower workaround:

- Its LaunchApproval gate prompt carries `%w(runners=0)` so an approved CI-fix agent
  waits for an idle machine (`src/bugyi_chops/ci_watch.py:1126`).
- It hand-rolls "is a `ci_fix` agent live?" suppression in the script
  (`src/bugyi_chops/ci_watch.py:1916`).

Neither gates the chop itself, and neither generalizes to "any agent".

## What already exists, and why none of it covers this

Guard support is real but identity-scoped. `ChopConfig.inhibit_if`
(`src/sase/axe/_config_types.py:62`) accepts exactly three providers, enforced by the
Rust config authority (`crates/sase_core/src/axe_chop/config.rs:711`) and evaluated by
the Rust decision engine (`crates/sase_core/src/axe_chop/decision.rs:13`):

| Provider     | Answers                                        | Covers "no agents running"? |
| ------------ | ---------------------------------------------- | --------------------------- |
| `patch`      | Is a Patch with this name prefix non-terminal? | No                          |
| `agent_hood` | Is an agent in _this_ hood active?             | No — one named hood only    |
| `agent_clan` | Is an agent in _this_ clan prefix active?      | No — one named clan prefix  |

The nearby capacity knobs do not help either:

- Lumberjack `wait_runners` gates **agents a chop proposes**, not the chop itself, and
  `docs/axe.md:477` says so explicitly: it "does not gate mentor, hook, or CRS workflow
  launchers."
- `axe.max_agent_runners` caps concurrent agent runners; it never suppresses a chop.

So the capability genuinely does not exist and belongs in the declarative guard
vocabulary, next to its two `agent_*` siblings.

## Design

### The guard

```yaml
inhibit_if:
  agent_runners: { max: 0 }
```

`agent_runners` inhibits the chop while **more than `max`** SASE agents hold a runner
slot. `max` is an optional non-negative integer defaulting to `0`, so
`agent_runners: {}` means "inhibit while anything is running". Both spellings the other
guards support work: keyed (above) and tagged (`- provider: agent_runners`, `max: 0`).

### Why "runner slots" is the right population

The counted population is exactly the one SASE already calls _running_: agents that
`running_root_agent_count` counts (`src/sase/core/runner_slots/_admission.py:174`),
which is the `R` in the ACE header capacity chip (`docs/ace.md:1663`) and the number
`%wait(runners=N)` compares against. Per-row it is `holds_runner_slot` on
`RunningAgentInfo` (`src/sase/agent/running_listing.py:253`): admitted (`run_started_at`
is set) and not parked on a pending question.

This matters because `list_running_agents()` returns more than that: `STARTING` agents
not yet admitted, `WAITING` agents parked on a time/dependency/bead wait, agents queued
for capacity, and agents stopped on a question. Counting _every live row_ would be the
obvious-looking choice and is the wrong one — a single agent stopped overnight on a
question, or parked on `%wait(time=8h)`, would inhibit the chop indefinitely.
Runner-slot semantics also make the guard say the same thing the ci_watch gate prompt
already says with `%w(runners=0)`, which is the property the user actually wants.

Accepted consequence: a just-launched agent that has not yet been admitted does not
inhibit the chop. That is the same narrow race `%wait(runners=0)` already has, and
accepting it keeps one definition of "running" across ACE, waits, and guards instead of
inventing a second.

### Naming

`agent_runners` keeps the `agent_*` guard family and reuses vocabulary already in the
config surface (`wait_runners`, `%wait(runners=N)`, "runner slots"). `max` reads
correctly at the use site and matches how `patch` supplies optional narrowing fields
rather than requiring them.

### Wire shape and cross-version compatibility

The engine decides; the host observes. `ChopAgentSnapshotWire`
(`crates/sase_core/src/axe_chop/wire.rs:263`) gains `holds_runner_slot: bool` with
`#[serde(default)]`, and the host sets it per row. A per-row flag rather than a
host-computed total keeps the split intact and lets the engine name an offending agent
in the skip reason, exactly like the sibling guards do.

`ChopAgentSnapshotWire` is `deny_unknown_fields`, so the host must **only** emit
`holds_runner_slot` when the chop actually configures an `agent_runners` guard. That
confines the new field to configs that already require a newer core — such a config
fails `unknown_guard_provider` validation on an older core anyway — so a new `sase`
against an older published `sase-core-rs` wheel keeps working for every existing guard.
No `pyproject.toml` pin edit is part of this work: per `docs/rust_backend.md:306`, the
`sase-core-rs` window is owned by the `sync-release-metadata` release job, and
`release-core-floor-smoke` is where the not-yet-published-capability invariant is
enforced.

### Rejected alternatives

- **Evaluate the guard in Python.** Violates the stated boundary —
  `src/sase/axe/chop_policy.py:1` and the repo's Rust-core rule put deterministic
  decisions in `sase_core`.
- **A `scope: runner_slots | live` knob.** Two meanings of "running", one of which
  starves the chop. Ship one correct meaning.
- **An `ignore_own: true` self-exclusion option.** Real, but not needed here: ci_watch's
  fixer is launched through a LaunchApproval gate, and `agent_hood` already covers
  "ignore my own hood" precisely. Left as a non-goal so the first version of the
  provider stays one idea.
- **Making it a lumberjack-level field.** `inhibit_if` is chop-scoped today, and the
  ci_watch lane has one chop. No need to widen the config surface.

## Rust agent_runners guard provider

All work is in the sibling core repo. Open it with the skill and use only the printed
path:

```bash
sase repo open sase-core -r "Add the agent_runners chop guard provider"
```

1. `crates/sase_core/src/axe_chop/wire.rs`
   - Add `AgentRunners { #[serde(default)] max: u64 }` to `ChopGuardConfigWire` (line
     ~203), renamed `agent_runners`.
   - Add `#[serde(default)] pub holds_runner_slot: bool` to `ChopAgentSnapshotWire`
     (line ~263).
2. `crates/sase_core/src/axe_chop/decision.rs`
   - Handle the new variant in the guard loop (line 13). Count rows where
     `row.active && row.holds_runner_slot`; if the count exceeds `max`, return `skip`
     with `provider = "agent_runners"`.
   - Reason format, so the host, CLI, TUI, and tests all agree on one string:
     ``inhibited by 2 agents holding runner slots (e.g. `foo.cld`); max is 0``. Use the
     first counted row by request order for the example name, and singular "agent" when
     the count is 1.
   - `validate_request` (line 121) needs no new case: `max` is unsigned and `0` is
     meaningful. Do not add a blank-value check here.
3. `crates/sase_core/src/axe_chop/config.rs`
   - In `validate_guard_provider` (line ~699) add `"agent_runners" => vec!["max"]` to
     the allowed-key table and extend the `unknown_guard_provider` message's
     supported-provider list (line ~721).
   - Validate `max` when present: must be an integer `>= 0`. Emit the existing
     diagnostic vocabulary — a `type_mismatch` for a non-integer and a
     `non_positive_threshold`-style code for a negative value, with the guard's config
     path.
4. Tests
   - `crates/sase_core/src/axe_chop/tests.rs`: guard fires above `max`; does not fire at
     exactly `max`; ignores rows with `holds_runner_slot: false` even when `active`;
     ignores `active: false` rows; accepts both keyed and tagged config spellings;
     default `max` is `0`; rejects an unknown key, a non-integer `max`, and a negative
     `max` with the right path.
   - `crates/sase_core/src/axe_chop/tests.rs`: guard ordering — an earlier
     `patch`/`agent_hood`/`agent_clan` guard still wins, since guards short-circuit in
     listed order.
   - `crates/sase_core_py/src/lib.rs`: extend the chop round-trip test near line 9837 so
     `agent_runners` travels through `py_evaluate_chop_decision` and
     `py_validate_axe_config` like `agent_clan` does.

Verify with `just check` from the core repo root, per `AGENTS.md` there.
`cargo test -p sase_core` alone is explicitly insufficient because it skips the
`sase_core_py` binding tests.

Land this on `sase-core` master before starting `host-guard`: `sase` CI builds
`sase_core_rs` from `sase-org/sase-core` HEAD (`.github/workflows/ci.yml:41`), so the
host phase cannot go green without it.

## Host snapshot, schema, and docs for agent_runners

Work in the `sase` workspace checkout.

1. `src/sase/axe/chop_policy.py`
   - `evaluate_chop_preflight` (line ~132): add `"agent_runners"` to the provider set
     that triggers building agent snapshots.
   - `_agent_snapshots()` (line 498) takes the configured guards so it can decide
     whether to emit the new key. When any guard is `agent_runners`, add
     `"holds_runner_slot": bool(agent.holds_runner_slot)` to each row; otherwise emit
     today's shape byte for byte. Keep the existing `active: True`.
2. `src/sase/config/sase.schema.json` — `definitions.axeChop.properties.inhibit_if`
   - Add `agent_runners` to the `provider` enum in the tagged-array branch and to
     `propertyNames.enum` in the keyed-object branch.
   - Add a `max` property (`{"type": "integer", "minimum": 0}`) to the tagged-array
     branch's properties.
   - Update the field `description` so it names all four providers. The Axe entry editor
     derives its form from this schema
     (`src/sase/ace/tui/modals/axe_entry_editor_types.py:114`) and already treats
     `inhibit_if` as a generic advanced field (line 25), so no TUI change is needed.
3. Docs — keep the tables and prose in sync in all four places that enumerate providers:
   - `docs/axe.md:493` (chop field table row) and `docs/axe.md:824` (the guards bullet
     under "Triggers, Guards, Dedupe, and Targets"). Explain the runner-slot population,
     the `max` default, and that a `STARTING` agent does not yet count.
   - `docs/configuration.md:2117` (chop field table row) and
     `docs/configuration.md:2182` (the `inhibit_if` paragraph).
   - Note that manual `sase axe chop run` still honors guards, so a manual run while
     agents are working skips unless `-f/--force` is passed. That is existing behavior
     (`docs/axe.md:843`) but is newly surprising with this guard, so it deserves a
     sentence.
   - Respect `markdown.print_width` (88) when rewrapping.
4. Tests
   - `tests/test_axe_chop_preflight_policy.py`: an `agent_runners` guard skips a
     scheduled run while a slot-holding agent exists; fires when the only live agent
     does not hold a slot; fires at exactly `max`; `-f/--force` bypasses it; and the
     recorded `ChopRunEntry.reason` carries the engine's reason string.
   - A count-parity test: for one synthetic artifact snapshot, the number of
     `holds_runner_slot` rows the preflight host emits equals `running_root_agent_count`
     (`src/sase/core/runner_slots/_admission.py:174`) over the same records. This is the
     regression that stops the guard's idea of "running" from silently drifting from the
     ACE capacity chip's.
   - Assert `_agent_snapshots()` omits `holds_runner_slot` entirely for a chop whose
     only guard is `agent_hood` or `agent_clan`, which is what protects older published
     cores.
   - `tests/test_config_schema_automation.py`: extend
     `test_config_schema_accepts_declarative_chop_policies` (line 60) with
     `agent_runners`, and add a rejection case for a negative `max`.

Verify with `just install` then `just check-full` — this change touches config schema
and docs, so the scoped lane is not the right gate.

## Guard skips stop consuming run_every cadence

Independent of the new provider, and listed separately so it can be dropped at approval
without affecting the rest of the epic.

A guard skip and a trigger skip are both recorded as `status="skipped"`
(`src/sase/axe/chop_runner_policy.py:58`), and the lumberjack advances the cadence clock
for either (`src/sase/axe/lumberjack.py:290`,
`update_timestamp=chop.run_every is not None`). So a chop with `run_every: 3h` that is
inhibited once does not retry for three hours, even if the guard clears a minute later.
`run_every` means "run at most this often"; a run that never happened should not spend
it. A trigger skip is different — the condition was evaluated and not met — and keeps
today's behavior.

`ChopPreflight.decision["provider"]` already distinguishes the two, so:

1. `src/sase/axe/chop_runner_types.py`: add a field to `ChopRunOutcome`, e.g.
   `advances_cadence: bool = True`.
2. `src/sase/axe/chop_runner_policy.py`: in `record_preflight_outcome`, set it `False`
   when the outcome is `skip` and the decision's `provider` is one of the guard
   providers (not `always` and not `git.commits_since`). Deriving from the provider
   rather than from the reason string keeps this mechanical.
3. `src/sase/axe/lumberjack.py`: in the `skipped` branch of `_outcome_to_result` (line
   290), honor the flag:
   `update_timestamp=chop.run_every is not None and outcome.advances_cadence`.
4. Tests in `tests/test_axe_chop_preflight_policy.py`: a guard-skipped run leaves the
   chop eligible on the next tick; a `git.commits_since` trigger skip still advances the
   clock.

Consequence worth stating: a guarded chop now re-evaluates its guard every tick until it
fires. For an `agent_runners` guard that is one agent-artifact scan per tick on that
lane, which is the same cost the existing `agent_hood`/`agent_clan` guards already pay.
Note in `docs/axe.md` that guard evaluation is not free and that guards belong on lanes
whose interval matches the cost.

## Enable the idle guard on bugyi_chop_ci_watch

Only start this once `host-guard` has landed **and** the live runtime has it installed.
Config loading is fail-closed, so applying a config with `agent_runners` to a runtime
that does not know the provider takes AXE down with `unknown_guard_provider`.

```bash
sase repo open chezmoi -r "Gate the ci_watch chop on an idle machine"
```

1. In `home/dot_config/sase/sase_athena.yml`, add to the `ci_watch` chop (line ~122,
   alongside its existing `env:` and `vars:`):

   ```yaml
   inhibit_if:
     agent_runners: { max: 0 }
   ```

   Write `max: 0` explicitly rather than relying on the default; this config is read far
   more often than it is written.

2. Update that chop's `description` body (line ~124) to name the new knob, as the
   authoring guide requires (`docs/axe.md:594`). The existing body already documents
   gate suppression; add that the whole chop is now inhibited while any agent holds a
   runner slot. Keep the summary line unchanged, keep the body within the 2000-character
   description cap, and keep source lines inside the configured prose width.

   Do **not** change the lumberjack-level `ci_watch` description at line 109 beyond what
   is needed to stay accurate.

3. Verify against the live runtime, in order:
   - `sase axe chop doctor` reports no config diagnostics and lists the chop with
     `guards=agent_runners`.
   - `sase axe chop show ci_watch -v` (or the JSON inventory) shows the guard with its
     chezmoi provenance.
   - With at least one agent running, `sase axe chop run ci_watch` records a `skipped`
     run whose reason is the engine's runner-slot message, and
     `sase axe chop run ci_watch -f` still runs.
   - With nothing running, a scheduled tick executes normally and refreshes the release
     report.

## Accepted consequences

- **The release report goes stale while agents run.** `bugyi_chop_ci_watch` republishes
  `ci_watch_releases.report.json` on every tick (`src/bugyi_chops/ci_watch.py:2247`),
  and an inhibited tick publishes nothing. The report's timestamps are absolute, so a
  reader sees the staleness rather than being misled by it. This is the direct cost of
  what was asked for, and it is the right trade: the alternative is a chop that merges
  release PRs underneath working agents.
- **Release merges and CI-fix gates wait for idle.** On a continuously busy machine the
  chop may not run for long stretches. If that turns out to be too strict in practice,
  `max: 1` or `max: 2` is a one-line change — which is a good reason for `max` to be a
  number rather than a boolean.
- **Manual runs are gated too.** `sase axe chop run ci_watch` skips while agents are
  running; `-f` is the escape hatch.

## Non-goals

- No `ignore_own`/self-exclusion option, no `scope` selector, and no idle-debounce
  ("agents have been idle for at least N minutes") field. Each is defensible later; none
  is needed for this outcome, and an idle-debounce in particular would need new
  host-owned state that the current stateless guard contract does not have.
- No lumberjack-level `inhibit_if`.
- No `pyproject.toml` `sase-core-rs` pin edit — that window is release-owned
  (`docs/rust_backend.md:306`).
- No change to what `bugyi_chop_ci_watch` itself does. Its in-script `ci_fix` hood
  suppression stays; the new guard is a strictly broader outer gate, and removing the
  inner one is a separate call in a separate repo.
