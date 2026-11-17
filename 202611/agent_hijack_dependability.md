---
tier: epic
title: 'Agent Hijack dependability: make v1 real, then hijack the agent, not the row'
goal: 'Pressing ,k (or the kill dialog''s h, or `sase agent hijack NAME`) on any local
  agent row, session container, or session name hijacks the session''s occupying turn
  when that is safe now, latches the hijack to the next safe hand-off when it is not,
  and otherwise answers with a precise reason and next action, never the generic "not
  eligible" toast. Ordinary running agent turns, --plan/--tale turns, retries, booting
  and WAITING/QUEUED turns that hold a numbered workspace, running monitor turns,
  finalizing turns, staged hand-offs, monitor follow-ups, and executing gates are
  all covered. A real-pipeline canary proves hijack then Continue works end to end,
  and concurrent or crashed hijacks converge to exactly one operation and at most
  one successor.

  '
decisions:
  glossary_strand:
    ask: Update the Agent Hijack glossary strand so it describes session-level, latched,
      monitor, and waiting-turn hijacks?
    memory:
    - glossary:agent-hijack
    corrects: An Agent Hijack stops a running agent turn while retaining its workspace
      claim, agent node, and recovery evidence.
    default: true
    answer: true
phases:
- id: core_decision_v2
  title: 'sase-core: eligibility decision v2, occupant selection, stop_blocked ledger
    state'
  depends_on: []
  size: medium
  description: 'core_decision_v2: in sase-core, add the v2 eligibility facts/decision
    wires (separate schema constant 2; mode immediate|latched|redirect|refused, code,
    message, next_action, redirect_op), the occupant-selection function, and the additive
    ledger fields plus the stop_blocked state and its events. Expose PyO3 bindings,
    add thin Python wrappers, and move sase''s sase-core-revision.txt pin.'
- id: meta_facts
  title: Keep the hijack capability stamp and read the model facts runners actually
    write
  depends_on: []
  size: small
  description: 'meta_facts: fix D3 (runner_capabilities erased by directive extraction)
    with one shared RUNNER_BOOT_META_KEYS tuple, a runner-side re-stamp, and an invariant
    test. Fix D4 (wrong model-fact keys, float-only booted_at, monitor/gate classified
    from live rows or never-written keys) using durable records and a fixture captured
    from a real runner meta.'
- id: snapshot_inventory
  title: Snapshot discovery that sees real linked-repo metadata
  depends_on: []
  size: small
  description: 'snapshot_inventory: make hijack stop/release snapshot discovery accept
    dict-shaped LinkedRepoResolution entries and the opened_linked_workspaces.json
    and opened_siblings.json markers by sharing one repository inventory with the
    commit finalizer, with fixtures built from real launcher metadata.'
- id: facts_resolver
  title: One facts builder, occupant resolution, and `sase agent hijack --explain`
  depends_on:
  - core_decision_v2
  - meta_facts
  size: medium
  description: 'facts_resolver: build one Python facts builder used by every hijack
    caller (no hard-coded facts), resolve names, session names, and session containers
    to the occupying member through the Rust occupant selection, and switch the stop
    orchestrator to the v2 decision. Add `sase agent hijack -e/--explain NAME` and
    the additive -j envelope keys, plus a screenshot-shaped fixture test.'
- id: tui_ingress
  title: 'TUI ingress uses the backend decision: precise toasts, container retargeting,
    redirects'
  depends_on:
  - facts_resolver
  size: medium
  description: 'tui_ingress: replace agent_can_offer_hijack with an asynchronously
    cached backend decision for the selected row; gate ,k, its footer chip, and the
    kill modal''s h on it; toast code + message + next action; run non-destructive
    redirects; mirror hijack_record onto collapsed session rows; log refusal codes;
    add a TUI/backend parity test.'
- id: stop_txn
  title: Transactional stop hardening and target-level dedup
  depends_on:
  - facts_resolver
  - snapshot_inventory
  size: medium
  description: 'stop_txn: target-level dedup that returns the active operation, verified
    intent persistence before any signal, pin restore on pre-signal abort, stop_blocked
    custody plus bounded retry on partial stops, pidfd signalling on Linux, stale-sweep
    reconciliation of abandoned requested/stopping records, and session promotion
    so a sessionless stop can still Continue.'
- id: canary
  title: Real-pipeline hijack canary
  depends_on:
  - tui_ingress
  - stop_txn
  size: medium
  description: 'canary: a fakey end-to-end test that launches through the normal pipeline
    (anonymous workflow, real bootstrap, real directive extraction), asserts the real
    loader row gets an immediate decision, hijacks through the TUI''s own submission
    path, asserts HIJACKED with the workspace held, runs Continue, and asserts the
    successor settles in the same workspace.'
- id: latch_core
  title: Durable hijack latch at agent-turn hand-offs
  depends_on:
  - canary
  size: large
  description: 'latch_core: latched ledger state and events in sase-core; the decision
    returns latched for finalizing turns and staged hand-offs; a Python latch store
    keyed by target; runner checkpoints at staged-handoff adoption, run end (moot),
    and retry boundaries; a pre- and post-spawn guard that the follow-up fallback
    chain cannot swallow; dead-owner reconciliation; a bounded wait with an explicit
    force-now path; CLI -c/--cancel and -n/--now; and the TUI pending overlay.'
- id: waiter_branch
  title: Hijack WAITING, QUEUED, and booting turns that hold a numbered workspace
  depends_on:
  - canary
  size: medium
  description: 'waiter_branch: allow WAITING/QUEUED/booting turns with a numbered
    claim (terminate the parked runner, or take the processless branch when it is
    dead), cancel the turn''s own wait marker while waiters of it stay parked, redirect
    claimless %wait turns to kill-and-edit with a reason, and make Continue relaunch
    a turn that never reached its provider.'
- id: gate_routing
  title: Latch hijacks behind executing gates
  depends_on:
  - latch_core
  size: small
  description: 'gate_routing: a gate executing its selected option yields latched;
    settle_gate_turn applies the latch after the commands settle and the decision
    is recorded, suppresses and records the follow-up, and holds the workspace, never
    replaying or reinterpreting the decision. Continue launches the recorded follow-up.'
- id: monitor_adapter
  title: Monitor hijack adapter (closes the screenshot case)
  depends_on:
  - latch_core
  size: large
  description: 'monitor_adapter: running monitor turns become immediate through stop_monitor
    with a hijack intent; the settlement funnel suppresses --next, holds the workspace,
    and writes a hijacked done marker for proc-backed and legacy monitors; a settled
    monitor with a pending follow-up latches. Recovery offers Continue (recorded follow-up
    with untrusted partial output), Escalate, Re-run command, View, Shell, and Dismiss.'
- id: ops_polish
  title: Doctor check, resolving watchdog, and remote referral tracking
  depends_on:
  - stop_txn
  - tui_ingress
  size: small
  description: 'ops_polish: a `sase doctor` check for running agents lacking the hijack
    capability, a watchdog that reconciles resolving records whose successor never
    dispatches, and local tracking of remote hijack referrals so the mirror row shows
    the request until the owner host reports an outcome.'
- id: chaos
  title: Boundary chaos test, crash convergence, and hand-off inventory guard
  depends_on:
  - gate_routing
  - monitor_adapter
  - waiter_branch
  size: medium
  description: 'chaos: a scripted fakey session (agent, sleep monitor, agent) that
    fires hijack at every checkpoint and requires HIJACKED, latched, or a defined
    redirect, never a generic refusal; crash-between-steps convergence tests; and
    a static guard test that fails when a new successor-launch path lacks a latch
    checkpoint.'
- id: land
  title: Docs sweep, glossary strand, and final verification
  depends_on:
  - chaos
  - ops_polish
  size: small
  description: 'land: reconcile the hijack Reference docs with the final behavior,
    update the Agent Hijack glossary strand per the glossary_strand decision, run
    the final check, and record deferred follow-ups.'
proposed_by: bbugyi200.athena.1ct
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-15 20:00:00
status: done
bead_id: sase-1t2
---

- **PROMPT:** [prompts/202611/agent_hijack_dependability.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/agent_hijack_dependability.md)
- **BEAD:** [sase-1t2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1t2/README.md)

# Plan: Agent Hijack dependability

## Context and source

Every phase worker should skim the consolidated research first:

```bash
sase artifact read research:202611/agent_hijack_dependability/agent_hijack_dependability.md "<why>"
```

The user accepted all of that report's requirements. **This plan is authoritative where
the two differ**; the differences are listed in
[Deviations from the research](#deviations-from-the-research). The v1 design lives in
`plan:202611/agent_hijack.md` (read it with `sase artifact read` if a phase needs v1
background). The v1 ledger, custody, and recovery design is sound. Most defects are
integration glue, so this epic fixes and extends the subsystem in place; it does not
rewrite it.

**What the screenshot showed.** On apollo the user pressed `,k` on the `tailnet-t.10`
session container row. The row's Python object is the session's first member (`--plan`).
It loads as `AgentType.WORKFLOW`, so `agent_can_offer_hijack`
(`src/sase/ace/tui/actions/agents/_hijack_action.py:23`) refused before Rust was ever
consulted, with the fixed toast "Selected agent is not eligible for hijacking". At that
moment the live work was a monitor turn (`--mon-3`), while agent turn `--5` was still in
host completion. v1 refuses both of those by design.

**Verified defects** (numbering from the research):

| #   | Defect                                                                                                                                                                                                                                                                                                                                                          | Fixed in                                                         |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| D1  | The TUI predicate requires `AgentType.RUNNING`; every raw-prompt launch loads as `WORKFLOW`, so `,k`, its footer chip, and the kill modal's `h` never appear                                                                                                                                                                                                    | `tui_ingress`                                                    |
| D2  | Session rows and session names resolve to the wrong member (the container row is the root; `find_named_agent` always returns the root for a session name); `_check_eligibility` hard-codes `is_llm_agent`, `is_named_proc`, and `is_clan_container`                                                                                                             | `facts_resolver`, `tui_ingress`                                  |
| D3  | `runner_capabilities` is written at bootstrap (`run_agent_runner_bootstrap.py:150-160`) and then dropped, because `preserved_agent_metadata` (`run_agent_directive_metadata_preserved.py`) does not list it; Rust then refuses everything with `missing_capability`                                                                                             | `meta_facts`                                                     |
| D4  | `_stop_target.py` reads `agent_llm_provider`/`agent_effort` and a float `booted_at`; runners write `llm_provider`/`exec_llm_provider`, `reasoning_effort`, and an ISO `booted_at`. Gate-ness comes from `gate_turn`/`is_gate_turn` keys nothing writes; monitor-ness comes from a live-row name match. With D1–D3 fixed, Continue and Escalate would still fail | `meta_facts`                                                     |
| D5  | Refusal reasons are hidden behind one string, and the TUI and Rust rule sets have drifted in both directions                                                                                                                                                                                                                                                    | `facts_resolver`, `tui_ingress`                                  |
| D6  | Finalizing, monitor, gate, and waiting states are refused by design                                                                                                                                                                                                                                                                                             | `latch_core`, `monitor_adapter`, `gate_routing`, `waiter_branch` |

**Rust/Python boundary** (`rust_core_backend_boundary`): sase-core owns the decision
table, occupant selection, ledger state machine, and recovery-preview action rows, so
the CLI, TUI, web, and fleet agree. Python owns effects: file I/O, locks, signals,
launches, CLI, and Textual. **Each phase that changes policy also implements the
matching mechanism.** The Rust decision never returns a mode (for example `latched` for
an executing gate) before the Python code that honours it lands in the same phase. This
is why no feature flag is needed. Every sase-core phase works in the linked checkout
(`sase repo open sase-core`; read its `AGENTS.md`), adds Rust tests and PyO3 bindings,
updates the sase-core CHANGELOGs per that repo's convention, and moves sase's
`sase-core-revision.txt` pin per `docs/rust_backend.md` ("the CI source revision pin").
Phases that run in parallel and both touch sase-core rebase the pin; the later one wins.

## The target behavior (the contract every phase serves)

### The unit is the agent, resolved to its occupant

`,k` on a member row, a session container row, or `sase agent hijack NAME` (member name
or session name) resolves to the **occupying member**. That resolution is atomic per
request and runs again inside the stop or latch transaction:

1. the member whose `artifacts_timestamp` matches the session's numbered workspace claim
   (the claim says who owns the workspace now, so a monitor supervisor that took over
   the claim is the occupant);
2. otherwise the newest in-flight member (STARTING/RUNNING/RETRYING, provider running,
   or finalizing);
3. otherwise the earliest WAITING/QUEUED member;
4. otherwise none (the session is idle or terminal).

The toast and CLI output always name the member actually acted on, for example
`⏏ hijacked tailnet-t.10--mon-3 · ws #4 held`.

### Per-state decisions

`immediate` stops now. `latched` is durable and is applied at the next safe hand-off.
`redirect` is a **non-destructive** navigation the TUI performs for the user: open the
recovery panel, open the gate review, select a member, or copy a remote command.
`refused` carries a code, a one-line message, and a `next_action`. **No path shows the
generic toast again.** The last column names the phase that delivers each row.

| Occupying state                                                                                                      | Decision                                                         | Mechanism                                                                                                                                                                         | Phase                                           |
| -------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------- |
| Agent turn STARTING/RUNNING/RETRYING, including `--plan`/`--tale`                                                    | immediate                                                        | v1 stop orchestrator, now fed true facts                                                                                                                                          | P0 phases                                       |
| Agent turn finalizing (lifecycle `provider_done`/`finalizing`/`handoff`, or finalizer phase `declaring`/`executing`) | latched                                                          | The finalizer settles and commits land; at staged-handoff adoption the runner settles `hijacked` instead of adopting; with no hand-off, the latch settles `finished_first` (moot) | `latch_core`                                    |
| Staged hand-off not yet adopted                                                                                      | latched                                                          | Checked under the adoption lock before any adopter runs                                                                                                                           | `latch_core`                                    |
| Retry boundary (in-process or spawned retry)                                                                         | immediate on the live runner; spawn guard if a retry child races | Pre-spawn and post-spawn guards                                                                                                                                                   | `latch_core`                                    |
| Running monitor turn _(the screenshot)_                                                                              | immediate (monitor adapter)                                      | `stop_monitor` with a hijack intent; the settlement funnel suppresses `--next` and holds the workspace                                                                            | `monitor_adapter`                               |
| Monitor settled, follow-up not yet launched                                                                          | latched                                                          | Applied in `settle_claim_and_followup`                                                                                                                                            | `monitor_adapter`                               |
| Gate awaiting a human                                                                                                | redirect `gate_review`                                           | "Awaiting your decision — Enter to review"                                                                                                                                        | `core_decision_v2` / `tui_ingress`              |
| Gate executing its selected option                                                                                   | latched                                                          | Applied in `settle_gate_turn` after the commands settle and the decision is recorded                                                                                              | `gate_routing`                                  |
| WAITING/QUEUED/booting with a numbered claim and a live runner                                                       | immediate                                                        | Terminate the parked runner; its exit-143 path already records `hijacked` and holds the workspace                                                                                 | `waiter_branch`                                 |
| The same, with the runner dead                                                                                       | immediate (processless)                                          | Pin by claim identity, skip `terminate_tree`, write the marker                                                                                                                    | `waiter_branch`                                 |
| WAITING/QUEUED with no numbered claim (for example a `%wait` placeholder on #0)                                      | refused `nothing_to_keep`                                        | next_action: "use x, then edit and relaunch"                                                                                                                                      | `waiter_branch` (P0 refuses with the same code) |
| Clan container                                                                                                       | redirect `select_member`                                         | Auto-retarget when exactly one member is live                                                                                                                                     | `tui_ingress`                                   |
| Already HIJACKED                                                                                                     | redirect `open_recovery`                                         | Open the recovery panel                                                                                                                                                           | `tui_ingress`                                   |
| Hijack already requested, stopping, or latched for this target                                                       | redirect `show_active_hijack`                                    | Show the active operation; `,k` again on a latched row offers cancel                                                                                                              | `stop_txn`, `latch_core`                        |
| Terminal (DONE/FAILED/…)                                                                                             | refused `already_completed`                                      | next_action: fork or retry                                                                                                                                                        | `core_decision_v2`                              |
| Remote fleet row                                                                                                     | redirect `copy_remote_command` + tracked referral                | Copy `sase agent hijack NAME`; the mirror row shows the referral                                                                                                                  | `tui_ingress`, `ops_polish`                     |
| Named proc, home mode, bulk mark, multi-step (non-anonymous) workflow step                                           | refused with code                                                | Each names its alternative                                                                                                                                                        | `core_decision_v2`                              |
| Runner without the capability stamp                                                                                  | refused `missing_capability`                                     | "runner did not record hijack support; use x, or wait for its next turn"                                                                                                          | `meta_facts` makes this rare                    |

Until the phase that delivers a row lands, that row stays `refused` with an accurate
message (for example "monitor turns cannot be hijacked yet; stop it with x or wait for
its follow-up"). It is never generic.

### CLI contract (per `cli_rules`: sorted options, short aliases, excellent `-h`)

`sase agent hijack [-c|-e|-n] [-j] [-r REASON] NAME`:

- `-c, --cancel`: cancel a pending latched hijack (`latch_core`);
- `-e, --explain`: read-only; print the resolved member, why it was chosen, the full
  facts, and the decision (`facts_resolver`);
- `-j, --json`: one stable envelope. Existing keys keep their meaning, and new keys are
  additive: `member`, `mode`, `code`, `next_action`, `redirect_op`, `latched`;
- `-n, --now`: force a latched hijack whose bound has passed (effect uncertain;
  `latch_core`);
- `-r, --reason`: unchanged.

`-c`, `-e`, and `-n` are mutually exclusive. Exit codes: 0 = HIJACKED (or the request
coalesced onto an already-held hijack); 1 = stop failed or error; 2 = refused or
redirect, nothing changed; **3 = accepted and latched** (applies at the next safe
hand-off). `--explain` exits 0 whenever it computed a decision and 2 when no agent
resolves.

### TUI

- `,k`, its leader-footer chip, and the kill modal's `h` are driven by the cached
  backend decision for the selected row (modes `immediate`, `latched`, `redirect`).
  `refused` hides the kill-modal `h`; `,k` then toasts the reason plus `next_action`.
- Latched rows show **`⏏ HIJACK PENDING · after <boundary>`** (for example
  `after finalizer`, `after gate`, `after monitor`). `,k` on a pending row offers
  **Cancel hijack**. After the force bound has passed it also offers **Hijack now —
  interrupts the finalizer (effect uncertain)**.
- A latch mooted by clean completion toasts "finished before the hijack; nothing was
  held".

## Architecture shared by all phases

### Decision wire v2 (sase-core, `core_decision_v2`)

- Add a **new** constant, `AGENT_HIJACK_ELIGIBILITY_WIRE_SCHEMA_VERSION = 2`, for facts
  and decision only. The ledger record keeps `AGENT_HIJACK_WIRE_SCHEMA_VERSION = 1` with
  additive `#[serde(default)]` fields, so records already on disk still advance. Today
  one constant guards both; split them.
- **Facts v2:**
  - row kind: `is_local`, `is_remote`, `is_bulk`, `is_named_proc`, `is_home_mode`,
    `is_clan_container`, `is_session_container`, `live_member_count`;
  - occupant: `occupying_kind` (`llm|wait|queue|monitor|gate|workflow_step|none`),
    `status`, `turn_phase` (runner lifecycle phase), `finalizer_phase`, `gate_state`,
    `monitor_state`;
  - workspace and process: `workspace_num`, `has_numbered_claim`, `pid`,
    `process_alive`;
  - history: `has_hijack_capability`, `successor_attachable`, `done_outcome`,
    `active_hijack_state` (the state of any non-terminal ledger record for this target).
- **Decision v2:** `mode`, `eligible` (true for immediate/latched, kept for older
  callers), `code`, `message`, `next_action`, `redirect_op`, and `adapter`
  (`llm|processless|monitor`, set when the mode is immediate).
- **Occupant selection:** a pure function over member facts (`name`, `status`,
  `holds_claim`, `in_flight`, `waiting`, `settled`, `order`) that applies the four-step
  rule above and returns the chosen index plus a reason string. `--explain` prints that
  reason.

### Target key and dedup (`stop_txn`)

The target key is `sha256(project_file + "::" + (session name or agent name))`. A target
index under the hijack ledger root maps each key to the one non-terminal hijack id. It
is reserved by exclusive create under the ledger lock. A second request for the same
target (TUI + CLI, double press, retry after a lost response) gets the active operation
back; it never starts a second stop. The TUI proc concurrency key becomes
`agent-hijack:<target_key>`.

### Ledger additions

- `core_decision_v2`:
  - fields `target_key` and `scope` (`llm|processless|monitor|gate`);
  - state `stop_blocked`, entered by event `stop_blocked` from `stopping` when a
    surviving writer remains. Custody stays and the pin stays;
  - event `retry_stop`: `stop_blocked` → `stopping`;
  - event `abandon_stop`: `stop_blocked` → `stop_failed`. Only this explicit user choice
    unpins.
- `latch_core`:
  - state `latched`, entered by event `latch` from `requested`;
  - event `apply_latch`: `latched` → `stopping`;
  - event `cancel_latch`: `latched` → new terminal state `cancelled`;
  - event `finished_first` is also legal from `latched` (moot);
  - fields `latched_at`, `applied_at`, `boundary`, `boundary_outcome` (the member's real
    outcome, preserved verbatim), `suppressed_followup` (`{kind, ref}`), `forced`,
    `owner_pid`, and `owner_identity`.

### Why the latch is a checkpoint inventory, not a claim flag (verified deviation)

The research proposed storing the latch on the workspace claim, on the premise that
every hand-off transfers the claim. **That premise is false.** Verified against the
code:

- Same-process successors do no claim operation. These are pipe, plan→coder, and
  questions: `axe/run_agent_successor_inprocess.py:53-133`, called from
  `run_agent_exec_plan.py`, `run_agent_exec_plan_accept.py`,
  `run_agent_exec_questions.py`, and `run_agent_exec_pipe_handoff.py`.
- Same-process retries do no claim operation (`run_agent_exec.py:385-387`). A failed
  retry-spawn transfer falls back to an in-process retry
  (`run_agent_retry_spawn.py:326-333`).
- `launch_turn_followup` turns a claim-transfer `WorkspaceClaimError` into a fresh
  claim, then a pool workspace, then #0 (`turns/followup_launch.py:225-295`). A claim
  refusal there would launch the successor elsewhere.
- Rust spawns the child **before** the claim callback runs (`agent/launch_spawn.py`
  around 492-579). A claim refusal therefore means spawn-then-kill.
- The Rust claim reader drops unknown suffix fields, and `release_workspace` deletes the
  row.

So the latch is a **ledger record in state `latched`, found through the target index**.
Every hand-off decision point calls one helper,
`observe_hijack_latch(target_key, boundary) -> LatchVerdict`. The decision points:

| Decision point                                                                                                                | Where                                                                                                                                                                                                        | Phase             |
| ----------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------- |
| Staged-handoff adoption and run end (covers every in-process successor, monitor/gate staged adoption, and successor delivery) | `dispatch_completed_staged_handoff` under the adoption lock (`axe/run_agent_exec_staged_dispatch.py`), plus the no-handoff run end in `run_agent_exec.py`                                                    | `latch_core`      |
| Retry boundaries                                                                                                              | the in-process retry loop and `run_agent_retry_spawn.py`                                                                                                                                                     | `latch_core`      |
| Generic spawn guard                                                                                                           | the pre-spawn check in `launch_turn_followup`, `launch_spawn`, and successor delivery; a post-spawn re-check inside the claim callback under `patch_lock` kills the child (existing callback-raise behavior) | `latch_core`      |
| Gate follow-up                                                                                                                | `settle_gate_turn` (`gate_turn/settlement.py`)                                                                                                                                                               | `gate_routing`    |
| Monitor follow-up                                                                                                             | `settle_claim_and_followup` (`monitor/_settlement_followup.py`), shared by the proc, reconcile, and legacy supervise paths                                                                                   | `monitor_adapter` |

The latch refusal raises a distinct `HijackLatchedError`. It is **not** a
`WorkspaceClaimError`, so no fallback chain can swallow it. `chaos` adds the static
guard that keeps this inventory complete.

**Lock order.** The order is adoption lock → gate-lane or monitor-lane lock → gate
follow-up lock → hijack ledger/target-index lock → `patch_lock`. `patch_lock` stays
innermost and cannot be taken twice in one process. Document this order in the hijack
package docstring and add an assertion-style test in `latch_core`.

**Applying a latch at a boundary** reuses one function shared with the orchestrator, the
processless branch, and dead-owner reconciliation:

1. write `done.json` with outcome `hijacked`, preserving the member's real outcome in
   `boundary_outcome`;
2. hold (pin) the claim;
3. record the suppressed follow-up or hand-off;
4. collect evidence and snapshots;
5. CAS `latched → stopping → held`.

The successor's continuation brief names the suppressed hand-off and its payload path.

## Phase: core_decision_v2

Work in sase-core (`crates/sase_core/src/agent_hijack/`: `wire.rs`, `eligibility.rs`,
`ledger.rs`; bindings in `crates/sase_core_py/src/agent_hijack/`). Then update sase.

1. Split the schema constants as described in
   [Decision wire v2](#decision-wire-v2-sase-core-core_decision_v2). Add
   `AgentHijackEligibilityFactsV2Wire` and `AgentHijackEligibilityDecisionV2Wire`, and
   `agent_hijack_decide(facts_v2)`. Keep the v1 function only as long as a sase caller
   needs it. `facts_resolver` moves every caller to v2, so delete v1 there or in this
   phase if nothing else calls it.
2. Encode today's truthful table:
   - `immediate`: an `llm` occupant in STARTING/RUNNING/RETRYING with a numbered claim,
     a live process, and the capability;
   - redirects: `open_recovery` for HIJACKED, `select_member` for clan containers,
     `gate_review` for a pending gate, `copy_remote_command` for remote rows,
     `show_active_hijack` when `active_hijack_state` is non-terminal;
   - refusals for everything else, each with its own code, message, and `next_action`.
     Finalizing: "finalizing; try again in a moment". Monitor: "monitor turns cannot be
     hijacked yet; stop it with x or wait for its follow-up". Executing gate, waiting or
     queued, `nothing_to_keep`, `no_session` (when `!successor_attachable`),
     `workflow_step`, `missing_capability`, terminal, named proc, home, and bulk each
     get their own.
3. Add the occupant-selection function, and the ledger additions listed for this phase
   under [Ledger additions](#ledger-additions). Write exhaustive table tests:
   - every refusal and redirect code;
   - every legal and illegal transition;
   - a v1 record without the new fields still advancing.
4. In sase:
   - bump the pin;
   - add thin wrappers in `src/sase/agent/hijack/core.py`;
   - mirror any Python constants (ledger state names in `ledger.py`) and add a binding
     test.

   Python callers keep using v1 until `facts_resolver`. This phase changes no user
   behavior.

## Phase: meta_facts

1. **D3, the class not just the instance.**
   - Define `RUNNER_BOOT_META_KEYS` in one module and use it from both the bootstrap
     writer (`axe/run_agent_runner_bootstrap.py`) and `preserved_agent_metadata`
     (`axe/run_agent_directive_metadata_preserved.py`). Include `runner_capabilities`,
     `code_identity`, `booted_at`, `launch_scratch_key`, `prompt_origin`, and
     `prompt_source_surface`.
   - In `write_agent_meta` (`axe/run_agent_markers.py`), re-stamp the hijack capability
     when `pid == os.getpid()`, following the `launch_scratch_key` precedent there.
   - Grep every writer that creates or rebuilds a successor's or follow-up's
     `agent_meta.json` and make sure none drops boot keys.
   - Add an **invariant test**: run the real bootstrap stamp, then the real directive
     extraction/rebuild, and assert every `RUNNER_BOOT_META_KEYS` key survives.
2. **D4.**
   - In `agent/hijack/_stop_target.py`, read the provider from `llm_provider` or
     `exec_llm_provider` (keep the legacy keys as fallbacks), the effort from
     `reasoning_effort`, and the model from `model`.
   - Parse `booted_at`/`started_at` as ISO-8601 or epoch.
   - Classify monitor turns with `monitor/models.py::is_monitor_member_record` and
     `plan_chain.turn_kind_value`. Classify gate turns with
     `gate_turn/state.py::is_real_gate_member` and expose `gate_state`. Never use
     live-row name matches or the never-written `gate_turn`/`is_gate_turn` keys.
3. Capture a **real** runner `agent_meta.json` from a fakey run, sanitize its paths, and
   commit it as a test fixture. Write the D4 tests against it, not against a
   hand-written meta. Update `tests/fakey/test_agent_hijack_e2e.py` so it stops
   hand-writing capability and legacy model keys.
4. Change the `missing_capability` message to "runner did not record hijack support; use
   x, or wait for its next turn". Do not infer capability from `code_identity` (see
   Deviations).

## Phase: snapshot_inventory

1. In `agent/hijack/_stop_evidence.py`, `_release_snapshot.py`, and
   `_release_effects.py`, stop accepting only `list[str]` for linked repos. Find the
   commit finalizer's repository-inventory helper and reuse it, so hijack and finalizer
   see the same repo set: primary, linked (`LinkedRepoResolution` dicts), external, and
   sidecar, plus the dynamically opened repos recorded in
   `opened_linked_workspaces.json` and `opened_siblings.json`. If no single helper
   exists, extract one and point both callers at it.
2. Tests must use metadata produced by the real launcher code path, not string-path
   fixtures. Fix the e2e fixture that hid this. Cover a dirty primary, a linked repo, an
   opened sibling, and a sidecar, and assert each one is snapshotted before release.

## Phase: facts_resolver

1. **New module** `src/sase/agent/hijack/facts.py`:
   - `resolve_hijack_subject(name | row) -> HijackSubject` resolves a member name,
     session name, or session-container identity to the occupant. It uses
     `find_agent_session` (`agent/names/_lookup_groups.py`) and the claims from
     `running_field.get_claimed_workspaces`, then the Rust occupant selection.
   - `build_hijack_facts(subject) -> facts_v2` computes every fact from durable records,
     including `process_alive` (a real liveness and identity check, not "pid present")
     and `successor_attachable` (true when the member has an agent session; `stop_txn`
     widens this).
   - `decide(subject)` calls the v2 binding.
   - `HijackTarget` gains the occupant and the decision.
2. `_stop_orchestrator.py` stops calling `_check_eligibility` with hard-coded facts and
   uses `decide`. `resolve_target` resolves through the subject, so the CLI no longer
   acts on `find_named_agent`'s root for a session name. Re-resolve inside the
   transaction: if the occupant changed between decision and pin, re-decide once, then
   refuse with `occupant_changed`.
3. Add `-e/--explain` and the additive `-j` keys to `sase agent hijack`
   (`main/parser_agent_lifecycle.py`, `agents/cli_hijack.py`). Use colored Rich output
   for `--explain`. Regenerate the CLI reference and completion snapshots with
   `just fix`.
4. Log every decision (`code`, `mode`, `member`) at INFO in one stable format, so real
   use can be measured.
5. **Screenshot-shaped fixture test.** Build a session with:
   - a workflow root plus historical DONE members;
   - an agent member in host completion with a staged monitor hand-off;
   - an active monitor member holding the numbered claim.

   Assert that the session name and the container identity resolve to the monitor member
   and yield the precise monitor refusal. `monitor_adapter` later flips it to immediate.

## Phase: tui_ingress

Read the `tui_perf` memory note (via `tui.md`) before starting.

1. Replace `agent_can_offer_hijack` with a per-selection decision cache. Compute the
   decision with `resolve_hijack_subject` + `decide` **off the UI thread**, keyed by row
   identity and roster generation, and refresh it on selection change and agents
   refresh. Never block Textual rendering on filesystem scans. Until a decision arrives,
   the footer chip uses only the cheap row-local refusals (remote, named proc, bulk,
   terminal). Pressing `,k` with no cached decision computes it in the worker and then
   acts.
2. Re-gate `_start_hijack_agent`, the leader `,k`
   (`actions/agent_workflow/_leader_mode.py`), the footer chip
   (`actions/agents/_display_detail_footer.py`), and the kill modal's `h`
   (`actions/agents/_kill_action_flow.py`) on the decision:
   - submit the hijack for the **occupant's** name;
   - toast `code`, `message`, and `next_action`;
   - perform redirects: open the recovery panel, open the gate review the way Enter
     does, auto-select the single live clan member and re-decide, or copy the remote
     command (existing helper).
3. Make `_mirror_root_from_child` (`models/_agent_status_apply.py`) mirror
   `hijack_record` and the hijack overlays, so Enter opens recovery from a collapsed
   session.
4. **Parity test.** Load real loader rows (anonymous-workflow agents, a session
   container, a clan container, a remote row, a monitor member, a gate member). Assert
   the TUI-cached decision equals the CLI `--explain` decision for each.
5. Update the `?` help popup, `docs/reference/tui/agents-tab.md`, and
   `agents-tab-modes.md` for the new semantics. Run the targeted
   `just fix-tui-screenshots` capture for any changed render, through `/sase_monitor`.

## Phase: stop_txn

1. **Target-level dedup** per [Target key and dedup](#target-key-and-dedup-stop_txn).
   Include a concurrency test in which TUI and CLI request the same agent at once: one
   record, one stop, and both callers report the same hijack id.
2. **Intent persistence.** Make `write_user_kill_intent` failures fatal before any
   signal (`intent_write_failed`; abort cleanly).
3. **Pin restore on abort.** If `begin_stop` fails after `pin_claim`, the abort path
   restores the prior pin state.
4. **Partial stop retains custody.**
   - Before giving up, retry termination a bounded number of times (2), escalating to
     SIGKILL.
   - With a surviving writer, advance to `stop_blocked`: keep the pin and the intent,
     and report "N processes survived; workspace kept; `,k` retries, or release with x".
   - `,k` on a `stop_blocked` target sends `retry_stop`. Only an explicit choice sends
     `abandon_stop` and unpins.
5. **pidfd.** On Linux, signal through `os.pidfd_open` + `signal.pidfd_send_signal`
   after the identity check, to close the check-then-signal race. Fall back to today's
   identity-checked kill elsewhere. Cover PID reuse and permission-denied.
6. **Stale sweep.** Teach `ace/scheduler/stale_running_sweep.py` to reconcile abandoned
   `requested`/`stopping` records:
   - process dead and no marker → apply the fallback settlement and publish `held`;
   - process alive and no orchestrator → move to `stop_blocked`.
7. **Sessionless runs.** Make keep-work recovery promote a sessionless predecessor into
   a session using the existing promotion in `agent/_agent_session_promotion.py`, at the
   point `_recover_prepare.py` raises `no_session` today. Set `successor_attachable` to
   true whenever promotion's preconditions hold. A stop is never accepted when its
   Continue would be refused.

## Phase: canary

1. In `tests/fakey/`, write a real-pipeline canary that drives:
   - the normal launch path (anonymous `tmp_*` workflow, real bootstrap, real directive
     extraction, fakey provider held on a barrier) until the agent is RUNNING;
   - the real TUI loader, asserting its row gets an `immediate` decision;
   - a hijack through the TUI's own submission path (`submit_agent_hijack` →
     `sase agent persist-hijack`);
   - assertions that the row is HIJACKED, the claim is pinned, and the snapshots exist;
   - Continue through `sase agent recover`;
   - assertions that the successor runs in the **same workspace**, settles, and the
     predecessor shows CONTINUED.
2. Add a session variant: hijack by session name while member `--1` is running, and
   assert the resolver chose `--1`.
3. Keep it within the fakey e2e budget. If it is too slow for the default lane, mark it
   the way the other `tests/fakey` e2e tests are marked; never skip it silently.

## Phase: latch_core

This is a large phase: plan before implementing. Read the
[latch architecture](#why-the-latch-is-a-checkpoint-inventory-not-a-claim-flag-verified-deviation)
first.

1. **sase-core:**
   - add the ledger additions listed for this phase;
   - make the decision return `latched` (adapter none) for the finalizing phases and for
     staged-not-adopted, with a `boundary` hint;
   - add a pure reconcile function for latches: owner alive → keep; owner dead with no
     successor → apply; real outcome with a successor launched → `finished_first`.
2. **Latch store and primitives** in `agent/hijack/latch.py`: `request_latch`,
   `observe_hijack_latch`, `apply_latch_at_boundary` (the shared apply function),
   `cancel_latch`, and `moot_latch`. `stop_agent_for_hijack` returns exit 3 with mode
   `latched`.
3. **Checkpoints:**
   - staged-handoff dispatch under the adoption lock, before any adopter: apply and
     record `suppressed_followup = {kind: <handoff kind>, ref: <staged payload path>}`;
   - the no-handoff run end: moot;
   - the in-process retry loop and the retry spawn;
   - the spawn guard, pre-spawn and post-spawn inside the claim callback, in
     `launch_turn_followup`, `launch_spawn`, and successor delivery.

   Raise `HijackLatchedError` and make sure the follow-up fallback chain in
   `turns/followup_launch.py` re-raises it.

4. **Never wait on a dead or stuck owner.**
   - The stale sweep calls the reconcile function for latched records.
   - After a 120 s bound with the owner still alive, `,k` and `-n/--now` offer a forced
     immediate stop (`forced: true`). Recovery shows "effect uncertain — finalizer
     interrupted; inspect the snapshot".
   - Automatic preemption never happens.
5. **CLI:** `-c/--cancel` and `-n/--now`, exit 3, `latched` in `-j`. **TUI:** the
   `⏏ HIJACK PENDING · after <boundary>` overlay, `,k` again to cancel, and the moot
   toast.
6. The continuation brief (Rust `continuation.rs` evidence) names the suppressed
   hand-off and its payload path, labeled untrusted.
7. **Tests:**
   - each checkpoint latched → held exactly once;
   - cancel;
   - moot;
   - dead-owner apply;
   - forced path;
   - a race of latch vs adoption with one winner and at most one successor;
   - the lock-order test;
   - an extended canary that hijacks during finalizing.

   Update `docs/reference/tui/agent-statuses.md` (the new overlay) and
   `docs/reference/agents/agent-sessions.md` (session-level hijack).

## Phase: waiter_branch

1. **sase-core:**
   - `wait`/`queue` occupants with a numbered claim → `immediate`, adapter `llm` if the
     runner is alive, `processless` if not;
   - without a numbered claim → refused `nothing_to_keep`, next_action "use x, then edit
     and relaunch".
2. **Live runner:** terminate it as today; the parked-runner SIGTERM path
   (`axe/run_agent_wait_slots.py`, `run_agent_runner.py` `_record_user_kill`) already
   settles `hijacked`. Add tests for WAITING, QUEUED (`slot_requested_at`), and booting.
3. **Processless branch** in the orchestrator: no pid required; pin by claim identity
   (`artifacts_timestamp` + workspace), not by live pid; skip `terminate_tree` and the
   capability check, which only guards against signalling old runners; write the marker
   with the shared apply function.
4. **Cancel the turn's own wait marker.** Mark `waiting.json` cancelled-by-hijack so a
   later `ready.json` cannot resume it. Waiters **of** the agent stay parked: they key
   off its outcome, and `hijacked` keeps them parked. Test both directions.
5. Make Continue/Escalate work for a turn that never reached its provider (no
   transcript): the successor gets the stored prompt and instructions, and the brief
   says the turn never started.

## Phase: gate_routing

1. **sase-core:** a `gate` occupant with `gate_state == "settling"` → `latched`
   (boundary `gate`). Pending stays `gate_review`.
2. In `gate_turn/settlement.py::settle_gate_turn`, after the selected option's commands
   settle and the decision is written, but before any follow-up launch, call
   `observe_hijack_latch`. If latched:
   - suppress and record the follow-up (`suppressed_followup.kind = "gate_followup"`);
   - hold the claim. For `workspace: release` gates there is no claim to hold; record
     the boundary and keep the follow-up's planned workspace in the record;
   - apply the latch, keeping the gate's real outcome in `boundary_outcome`. The gate
     bundle's `response.json` stays the source of truth for the decision; never rewrite
     or replay it.
3. Recovery Continue launches the recorded follow-up in the held workspace. Tests cover
   plan-approval and question gates.

## Phase: monitor_adapter

This is a large phase: plan before implementing.

1. **sase-core:**
   - a `monitor` occupant with `monitor_state == "running"` → `immediate`, adapter
     `monitor`;
   - settled with a follow-up pending → `latched` (boundary `monitor`);
   - recovery preview rows for scope `monitor`: Continue (launch the recorded follow-up
     now), Escalate (the follow-up on a higher rung), Re-run command, View, Shell, and
     Dismiss. Hide Restart/Edit; Re-run replaces Restart.
2. **Stop.** For monitor targets, write the hijack intent and call `stop_monitor`
   (`monitor/store.py`) instead of `terminate_tree`. Cover both proc-backed and legacy
   supervisor monitors.
3. **Settlement.** At the top of `settle_claim_and_followup`, check the hijack intent or
   latch first:
   - suppress `--next`;
   - skip `release_monitor_claim` and pin the claim, re-owning it from the exiting
     supervisor pid by claim identity;
   - write `done.json` with outcome `hijacked` and `boundary_outcome` set to the
     monitor's own state.

   Cover `proc_adapter.py`, `reconcile.py`, and legacy `supervise.py`.

4. **Recovery.** Continue/Escalate launch the recorded follow-up with the command's
   partial output injected, bounded by the continuation budget and labeled untrusted.
   Re-run starts the same monitor command in the held workspace.
5. Flip the screenshot fixture test to `immediate`. Extend the canary with a running
   sleep monitor. Update `docs/reference/automation/monitor-operations.md`.

## Phase: ops_polish

1. Add a `sase doctor` check that lists running agents whose meta lacks the hijack
   capability, under the existing doctor check registry (`src/sase/doctor/`).
2. Add a watchdog to recovery reconciliation for `resolving` records whose successor
   never dispatches: after a bound, reconcile to `held` with an error, using the
   existing Rust reconcile decision. This prevents `RECOVERING…` from holding a
   workspace forever.
3. **Remote referral tracking.** When the TUI copies a remote hijack command, record a
   small local referral (alias, agent, requested_at, TTL). The mirror row shows
   `⏏ hijack requested on <alias>` until fleet facts report HIJACKED or a terminal
   status, or the TTL lapses.

## Phase: chaos

1. **Boundary chaos test.** Run a scripted fakey session (agent → sleep monitor →
   agent). Fire hijack at every checkpoint: provider running, finalizing, staged,
   monitor running, monitor settling, follow-up spawn, and retry. Every attempt must end
   `held`, `latched`→`held`, or a defined redirect. A generic or `ineligible` refusal
   fails the test. Each case asserts at most one successor per progression point.
2. **Crash convergence.** Kill the hijack worker between every transaction step (record,
   intent, pin, begin_stop, terminate, settle, evidence, publish, latch apply). Assert
   that the stale sweep and reconciliation converge with no orphaned pin, no second
   stop, and no surviving writer allowed to release.
3. **Static hand-off inventory guard.** Write a test that enumerates every call site of
   the successor-launch primitives (`launch_turn_followup`, `launch_followup_agent`,
   staged adopters, `run_agent_successor_inprocess`, retry spawn, `launch_spawn` with a
   transfer pid). Assert each one is either behind an `observe_hijack_latch` checkpoint
   or in an explicit allow-list with a one-line reason. A new hand-off path then fails
   CI until it is covered.

## Phase: land

1. Make the Reference docs match the final behavior, routed per `docs_writing`:
   - `docs/reference/tui/agents-tab.md`, `agents-tab-modes.md`, and `agent-statuses.md`;
   - `docs/reference/agents/agent-sessions.md` and `workspaces.md`;
   - `docs/reference/automation/monitor-operations.md`;
   - `docs/reference/glossary.md` if it restates the hijack definition.

   Regenerate the CLI, keymap, and config references with `just fix`. Fix only guide
   steps that became wrong.

2. **Memory, per the `glossary_strand` decision.** Through `/sase_memory_write`, rewrite
   `sase/memory/glossary/agent-hijack.md` so it says:
   - a hijack targets the session's occupying turn: an agent turn, a running monitor
     turn, or a waiting/queued turn that holds a numbered workspace;
   - it stops that turn now, or latches to the next safe hand-off and applies exactly
     once;
   - it retains the workspace claim, node, and evidence; settles `hijacked`; and is
     resolved exactly once.

   Keep the "distinct from" sentence. Then run `sase memory init`.

3. Run `sase tool run check`. Record these as `PROPOSED FOLLOW-UP:` notes on the land
   bead:
   - a pending-gate hijack adapter (only if real use shows demand);
   - routed execution on the owner host for remote rows;
   - home-mode custody;
   - bulk hijack;
   - a "Resume suppressed hand-off" recovery action;
   - Telegram/mobile ingress.

> [!decision] glossary_strand = no Skip step 2 and record the strand update as a
> `PROPOSED FOLLOW-UP:` on the land bead.

## Acceptance criteria (the epic is done when all hold)

- The canary passes: real pipeline → hijack → HIJACKED → Continue succeeds in the same
  workspace. The screenshot-shaped fixture resolves to the monitor member and yields
  `immediate`.
- Concurrent TUI + CLI requests for one target coalesce into one operation. Lost
  responses and retries never create a second stop or successor. Races with admission,
  retry, finalizer completion, staged adoption, monitor completion, and gate acceptance
  produce at most one successor per progression point (`chaos`).
- No surviving writer ever permits workspace release. No unrelated process is ever
  signalled; PID reuse and permission-denied are covered.
- Snapshot discovery runs against real launcher metadata and opened-repo markers.
- Every refusal, in the TUI or the CLI, carries a code and a next action. TUI and
  backend decisions agree on identical inputs (parity test).

## Deviations from the research

1. **Latch carrier.** The latch is a ledger record found through the target index and
   enforced at an explicit checkpoint inventory, not a flag on the workspace claim. The
   claim-transfer premise was checked and is false; the reasons are listed
   [above](#why-the-latch-is-a-checkpoint-inventory-not-a-claim-flag-verified-deviation).
   The static inventory guard in `chaos` replaces the "every hand-off transfers the
   claim" assumption.
2. **WAITING/QUEUED are not processless.** A parked WAITING/QUEUED turn is a live runner
   process that holds the claim, and its SIGTERM path already settles `hijacked`. The
   "processless" branch exists only for a dead runner that still holds a numbered claim.
   A `%wait` launch holds a placeholder claim on #0, so the screenshot's neighbouring
   `tailnet-t.land` row gets `nothing_to_keep` with a next action.
3. **The occupant is whoever holds the claim first.** The research said "session
   roster's active member". The claim's `artifacts_timestamp` names the member that owns
   the workspace now, which is exactly the screenshot's case: the monitor supervisor
   took the claim while `--5` was still finalizing.
4. **No capability inference from `code_identity` ancestry.** Agents turn over within
   hours, git-ancestry checks do not belong on a TUI refresh path, and `ops_polish`'s
   doctor check makes the residue visible.
5. **The record schema stays at 1.** Only the facts/decision wire moves to 2, so held
   records already on disk keep advancing.
6. **No feature flag.** Each phase lands policy together with its mechanism, so no
   landed phase exposes an unbacked decision.
7. **Scope.** The research suggested shipping P0 alone and measuring. The user asked for
   the full coverage up front, so all phases are planned. The P0 phases still land first
   and the canary gates every widening phase.

## Risks and mitigations

- **Hand-off holes.** The checkpoint inventory may miss a successor-launch path. The
  `chaos` static guard and the boundary chaos test catch it, and `HijackLatchedError` is
  never a claim error.
- **Finalizer hangs.** Latches inherit them. The 120 s bound and the explicit, forced
  "effect uncertain" path mitigate this; automatic preemption never happens.
- **Lock inversions.** `patch_lock` stays innermost, and the documented order plus its
  test guard the rest.
- **TUI responsiveness.** Decisions are computed off-thread and cached per selection
  (see the `tui_perf` note).
- **Parallel sase-core phases.** `core_decision_v2`, `latch_core`, `waiter_branch`,
  `gate_routing`, and `monitor_adapter` all touch `eligibility.rs` and the pin. Parallel
  pairs rebase; the table tests make a bad merge loud.
