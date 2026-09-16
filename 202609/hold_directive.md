---
tier: epic
title: "%hold: a reverse-%wait admission barrier"
goal: "A launch (agent or stand-alone proc) can arm a durable, TTL-bounded, fail-open
  hold that makes selected WAITING/QUEUED agents and un-dispatched procs wait for it to
  settle — never touching running work — with a first-class CLI, a %hold prompt
  directive, ACE/LSP completion, and TUI visibility.

  "
phases:
  - id: queue-on-procs
    title: Allow %queue capacity on proc units
    depends_on: []
    size: large
    description:
      "queue-on-procs: allow %queue(capacity=) on %proc units with default weight 0 so a
      stand-alone proc can gate on runner load and drain the host before dispatch."
  - id: hold-store-core
    title: Rust hold-record store and bindings
    depends_on: []
    size: large
    description:
      "hold-store-core: add agent_hold.rs, a TTL-bounded fail-open flock-guarded hold
      store under SASE home with prune-on-read, armer-liveness pruning, the
      match/exclusion predicate, and pyo3 bindings."
  - id: hold-blocker-agents
    title: hold-barrier blocker at runner-slot admission
    depends_on:
      - hold-store-core
    size: large
    description:
      "hold-blocker-agents: consult hold records under runner_slots.lock via a new
      hold-barrier blocker, add agent_name to capacity records, park held waiters, and
      release holds on family and terminal-proc settlement."
  - id: hold-cli
    title: sase agent hold command group
    depends_on:
      - hold-blocker-agents
    size: large
    description:
      "hold-cli: add sase agent hold create/list/release/run/show plus arm and expiry
      notifications, proving the runtime behind a watchable CLI before any prompt
      surface exists."
  - id: hold-directive
    title: The %hold prompt directive
    depends_on:
      - queue-on-procs
      - hold-cli
    size: large
    description:
      "hold-directive: add %hold behind a new agent_holds beta flag — parsing, directive
      contract entry, arm-at-launch, kin exclusion, armer priority boost, approval
      preview, confirmation gating, and %repeat/%dispatch rejection."
  - id: hold-completion-lsp
    title: Completion and LSP for %hold
    depends_on:
      - hold-directive
    size: medium
    description:
      "hold-completion-lsp: complete %hold everywhere — new Hood value role, hood
      candidate rows, WAITING/QUEUED-first agent ranking, proc rows included, ACE
      wait-clause generalization, and xprompt LSP parity."
  - id: hold-proc-targets
    title: Hold un-dispatched proc units
    depends_on:
      - hold-cli
    size: medium
    description:
      "hold-proc-targets: evaluate the hold predicate at the admission-engine
      eligibility boundary so lexical and future selectors fence stand-alone procs
      before dispatch, while dispatched procs stay immune."
  - id: hold-visibility
    title: TUI, doctor, and deadlock visibility
    depends_on:
      - hold-cli
    size: medium
    description:
      "hold-visibility: surface holds — held_by on queue markers, Agents-tab and agent
      list -j rendering, a TUI hold panel modeled on the runner-limit override panel, a
      doctor stale-hold check, and the admission-deadlock notification."
  - id: wait-hood
    title: Hood selector for %wait
    depends_on:
      - hold-completion-lsp
    size: medium
    description:
      "wait-hood: give %wait a hood= keyword using the existing hood matcher, with
      contract, wait-resolution, completion, and parity-test updates."
  - id: hold-flag-removal
    title: Remove the agent_holds flag and close out
    depends_on:
      - hold-directive
      - hold-completion-lsp
      - hold-proc-targets
      - hold-visibility
      - wait-hood
    size: small
    description:
      "hold-flag-removal: delete the agent_holds Off branch, make %hold unconditional,
      close the flag bead, run the full landing gate, and propose the
      memory-documentation follow-up."
proposed_by: bbugyi200.athena.0ls
create_time: 2026-09-15 22:45:56
status: wip
---

- **PROMPT:**
  [prompts/202609/hold_directive.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/hold_directive.md)

# Plan: %hold — a reverse-%wait admission barrier

## 1. Context

The user asked for a `%sink` directive that is `%wait` in reverse: agents selected in
the arming prompt wait for the arming agent/proc to complete. The consolidated research
report (linked artifact ref
`research:202609/reverse_wait_hold_barrier/reverse_wait_hold_barrier.md` — read it with
`sase artifact read` before implementing any phase) verified every load-bearing code
claim and recommended a set of modifications, **all of which the user accepted**:

- **Rename `%sink` → `%hold`** (a sink is the thing that runs last; this directive makes
  its author run first). No short alias in v1 (`%h` is `%hide`).
- **Pull model, not push**: never write a blocking dependency into another launch's
  `waiting.json`/`ready.json`. The hold is one durable record per armer, evaluated by
  each candidate at admission boundaries it already passes through. A RUNNING agent has
  left those boundaries forever, so "running work is never affected" holds by
  construction.
- **First-class store + CLI first, directive as sugar**: prompt text is replayed
  (`#fork`, history, pipes), so the barrier must be a durable object with its own
  list/show/release CLI and a recovery path.
- **Fail-open, TTL-mandatory**: a hold is a scheduling courtesy, not a correctness
  invariant. The wrong failure mode is a frozen host, never an early release.
- **Split the drain out**: "run when nothing is running" = drain + hold. The drain
  primitive is `%queue` capacity (already the documented run-alone barrier for agents);
  phase `queue-on-procs` extends it to proc units. `%hold` is only the hold.
- **Freeze `pending` at arm time**; only lexical selectors (`names`, `hood=`, `tribe`)
  plus the `future` flag stay live, because only they can describe agents that do not
  exist yet.

Related bead: **sase-11k** (same-plan `%wait` on a proc unit resolves at supervisor
submission, not terminal status). The hold design must not inherit that defect: hold
release keys off **terminal settlement only**, never `Launched`.

## 2. Design at a glance

**Store** — `sase-core:crates/sase_core/src/agent_hold.rs`, persisting
`$SASE_HOME/agent_holds.json` (+ `agent_holds.lock`). Modeled on
`runner_limit_override.rs` / `provider_disable.rs` (schema version,
`deny_unknown_fields`, `expires_at > created_at` validation, prune-on-read, atomic
tempfile replace) but using the shared `store_lock.rs` instead of a private flock
helper. One record per **armer key**; re-arming replaces, never stacks.

Record fields:

- `armer`: kind (`agent` | `proc` | `cli`), key (agent family name / proc shell name or
  proc id / creating pid), display name, project, liveness data (pid + artifact dir for
  agents and cli, proc ref for procs).
- `created_at`, `expires_at` — TTL mandatory. New config fields `agent_hold_default_ttl`
  (default `2h`) and `agent_hold_max_ttl` (default `12h`, hard cap) in
  `src/sase/default_config.yml` plus the config schema.
- `scope`: `project` (default) | `host`.
- Match block: frozen `artifact_dirs` (the `pending` snapshot), live `names` (agent |
  family | clan | workflow | proc shell), `hoods`, `tribes`, `future: bool`.

A record is **active** iff schema-current, unexpired, and the armer is still live (dead
pid without a done marker, or terminal proc status ⇒ prune on read). Unreadable or
malformed store ⇒ **no hold**.

**Semantics** (each phase must preserve these; they are the contract):

| Rule               | Decision                                                                                                                                                                                                                                                                                                               |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Activation         | Active from **arm time** (launch submission / CLI create), not run start                                                                                                                                                                                                                                               |
| Armer scheduling   | Arming implies a wait-priority boost below `DEFAULT_WAIT_PRIORITY` (=10, lower is better) unless `%q(p=…)` is authored, so the armer clears admission quickly and is never held by its own or others' deference                                                                                                        |
| Exclusion          | The armer, its family generation, its clan, and its dotted descendants never match any hold predicate (checked at arm time for explicit names, and again at evaluation)                                                                                                                                                |
| Release            | Armer's **family generation settles** (any terminal outcome: success, failure, kill, skip, loss) for agent armers; **terminal proc status** for proc armers; explicit `hold release`; TTL expiry; dead-armer prune. A settled armer's `future` rule expires with the record — it never blocks later arrivals vacuously |
| Enforcement points | Agents: one `hold-barrier` blocker inside `waiter_blockers` at runner-slot admission. Un-dispatched proc units: the same predicate at the admission-engine eligibility transition. Gate 2 (`waiting.json`) is untouched — no third party ever writes a blocking dependency into another launch's marker                |
| Running work       | Never affected, by construction (both enforcement points are pre-run)                                                                                                                                                                                                                                                  |
| Scope              | `scope=project` default (runner slots are host-wide; "all waiting agents" must not silently mean every project); `scope=host` explicit in the text                                                                                                                                                                     |
| Failure            | Fail-open everywhere: TTL mandatory, armer-liveness pruning on read, unreadable store means no hold                                                                                                                                                                                                                    |
| Frozen vs live     | `pending` snapshot frozen at arm; `names`/`hood=`/`tribe`/`future` live                                                                                                                                                                                                                                                |
| Deadlock           | (a) arm-time self/kin rejection, (b) plan-time in-plan cycle rejection, (c) admission-time detection of candidate⇄armer mutual block raising a deduped notification; the TTL guarantees forward progress                                                                                                               |

**Syntax** (repeatable and unioned, like `%wait`):

```text
%hold:planner                               # exact agent/family/clan/workflow/proc-shell
%hold:@nightly                              # tribe
%hold(pending)                              # all WAITING|QUEUED now, frozen at arm
%hold(hood=sase-s7)                         # lexical hood match (component boundary)
%hold(pending, future, ttl=90m)             # freeze now + fence future launches
%hold(pending, future, scope=host, ttl=30m) # widest form, explicit in the text
```

Bare `%hold` and `%hold(all)` are hard errors that name the selectors. `%hold` with
`%dispatch` or `%repeat` is rejected. `future` + `scope=host`, or a frozen capture above
a configured threshold, raises a confirmation gate on interactive launches and is always
enumerated in `approval_preview` ("captures 4 waiting + 2 queued; skips 3 running").

## 3. Cross-repo protocol

Phases marked _cross-repo_ change `sase-core` first. Workers open it with
`sase repo open sase-core -r "<why>"` (the `/sase_repo` skill) and treat both checkouts
as declaration obligations. After the Rust change lands, bump `sase-core-revision.txt`
per `tools/ratchet_core_revision`; new bindings must also be added to the
`crates/sase_core_py/src/lib.rs` module doc index and pass
`tools/check_sase_core_rs_bindings` / `tools/validate_sase_core_rs`. Shared backend and
domain behavior belongs in `sase-core` (rust_core_backend_boundary); Python here is
plumbing, CLI, and presentation only.

Every phase: run `just check` before finishing; `just check-full` gates landing (the
land agent's responsibility, plus explicitly in `hold-flag-removal`). Workers must read
`sase/memory/lint_and_test.md` via `/sase_memory_read` before finishing.

## 4. Phases

### 4.1 queue-on-procs — allow `%queue(capacity=)` on `%proc` units

The drain half of "run a command when nothing is running". Today `%queue` is forbidden
on proc units (`proc_forbidden_directives` push at
`sase-core:crates/sase_core/src/agent_launch/mod.rs:1460`; `ProcUnitWire` at `:497` has
no queue fields), so a stand-alone proc cannot gate on host load at all.

- _Rust (cross-repo)_: stop forbidding `%queue` on proc units; carry authored
  `capacity`/`priority`/`weight` for proc units (default **`weight=0`** — a proc holds
  no runner slot, so by default it drains without occupying); keep `capacity=0` rejected
  exactly as for agents (`queue_directive.rs`). Update `approval_preview` and digests
  for proc units with queue fields.
- _Python_: before dispatching an eligible proc unit
  (`src/sase/agent/launch_admission_engine.py:190` proc branch →
  `src/sase/agent/launch_proc_runtime.py::dispatch_proc_unit`), when queue fields are
  authored, gate dispatch on the same runner-capacity admission agents use
  (`src/sase/core/runner_slots/_admission_snapshot.py`), surfacing blockers instead of
  dispatching. `%q:1` on a `%proc` unit must mean: dispatch only when occupied load is
  zero (weight 0 fits capacity 1 only when occupation is 0... verify semantics against
  `waiter_blockers`' `insufficient-capacity` math and add a proc-specific test that a
  weight-0 candidate with `capacity=1` is blocked while one weight-1 agent runs and
  admitted when the host is idle).
- _Tests_: `%proc … %q:1` waits for load to drain then dispatches; authored weight
  participates; `%queue` on procs no longer raises `agent-directive-on-proc`; contract
  parity (`tests/test_xprompt_directive_contract.py`) still green; existing agent
  `%queue` behavior unchanged.
- _Read first_: `sase/memory/xprompts.md`, `sase/memory/lint_and_test.md`.

### 4.2 hold-store-core — Rust hold store and bindings

_Cross-repo (Rust-only change + pin bump)._ New
`sase-core:crates/sase_core/src/agent_hold.rs` implementing the store described in §2.

- Follow `runner_limit_override.rs` (single-record template) and `provider_disable.rs`
  (keyed map, migration, first-writer-wins) for validation, prune-on-read, and the
  atomic-replace recipe; use the shared `store_lock.rs` for locking (better factored
  than the private `with_lock` copies those two carry).
- Public API: `arm_agent_hold` (relative + until variants; re-arm replaces),
  `release_agent_hold(armer_key)`, `list_agent_holds(now, liveness)` with prune-on-read,
  and a **pure predicate**
  `hold_blocks_candidate(record, candidate) -> Option<HoldBlock>` where candidate
  carries name, project, artifact dir, and launch/eligible timestamps. The predicate
  implements: frozen `artifact_dirs` membership; live name matching
  (agent/family/clan/workflow/proc shell); hood matching via
  `agent_identity::agent_name_in_hood` (`identity.rs:440` — component boundary: `foo`
  matches `foo.bar`, never `foobar`); tribe matching; `future` (candidate created after
  arm time); scope filtering; and armer-kin exclusion via `agent_name_ancestors` /
  `historical_family_scope` (handles `--` shell suffixes).
- Armer liveness is an input (pid-alive flag, done-marker flag, proc status), so the
  core stays I/O-pure beyond its own store file; the Python caller supplies liveness.
- pyo3 bindings in `crates/sase_core_py/src/lib.rs` mirroring the runner-limit-override
  binding block (error mapper → `PyValueError` / `PyTimeoutError` / `PyRuntimeError`;
  schema-version fn; doc-index rows), plus `pub mod agent_hold` re-exports in
  `crates/sase_core/src/lib.rs`.
- _Tests_ (model: `runner_limit_override.rs:275` and `provider_disable.rs:597` suites):
  expiry boundary; malformed/stale self-clean; bounded lock wait; re-arm replaces; kin
  exclusion (self, family generation, clan, dotted descendant); hood component boundary;
  `future` on/off around arm time; scope filtering; unreadable store treated as empty.
- _Read first_: `sase/memory/lint_and_test.md`.

### 4.3 hold-blocker-agents — enforcement at runner-slot admission

_Cross-repo._ One new typed blocker, evaluated where candidates already queue.

- _Rust_: add `agent_name: Option<String>` to `RunnerCapacityRecordWire`
  (`runner_capacity.rs:44`; `deny_unknown_fields` ⇒ schema-version bump) and an optional
  `holds` payload on `RunnerCapacityRequestWire` (`:24`). Extend `WaiterEvaluation`
  (`:211`) and emit a `hold-barrier` blocker from `waiter_blockers` (`:608`) using the
  phase-4.2 predicate; add optional `held_by` / `hold_expires_at` fields to
  `RunnerCapacityBlockerWire` (`:130`) with serde defaults; include `hold-barrier` in
  `has_resource_blocker` (`:930`) so held waiters **park** instead of spinning.
  `waiter_blockers` stays pure: hold records and liveness arrive in the request.
- _Python_: populate `agent_name` in
  `src/sase/core/runner_slots/_admission_capacity_records.py`; read the hold store once
  per admission poll **under the existing `runner_slots.lock`** in
  `src/sase/axe/run_agent_wait_slots.py::_try_claim_runner_slot` and pass records +
  liveness into the request built by `_admission_snapshot.py`. Because `claim()`
  (`record_run_started_at`, wired at `src/sase/axe/run_agent_runner.py:193`) runs under
  that same lock, "whichever happens first wins" is linearized with **no new lock**: a
  candidate that claimed before the arm is running and immune; one that hasn't is held.
- _Release hooks_: agent armers — release the record where the armer's **family
  generation** settles with any terminal outcome (the family/handoff machinery in
  `src/sase/core/wait_dependency_resolution/_index_queries.py::family_candidate_for_root`
  defines "settled"; hook the settlement/done-marker path, and the kill path in
  `src/sase/ace/tui/actions/agents/_killing_utils.py` follows automatically if release
  keys off done markers + liveness — worker verifies and covers kill with a test). Proc
  armers — hook terminal settlement in `src/sase/procs/settlement.py::settle_proc_shell`
  (`TERMINAL_PROC_STATUSES` in `procs/models.py:15`). TTL and dead-armer pruning already
  release store-side.
- _Blocker surfacing_: the unknown-code fallthrough at
  `src/sase/ace/tui/widgets/prompt_panel/_agent_queue_section.py:333` renders the
  message automatically; set message text `held by <armer> (expires in <t>)`. Full
  `held_by` presentation is phase 4.8.
- _Tests_: WAITING and QUEUED candidates held; a RUNNING agent never re-evaluates
  (defining invariant); arm-vs-claim race in both orders under the lock; multiple holds
  on one candidate (all must release); fail-open on malformed store and dead armer;
  release on success, failure, and kill; `future` matches a launch submitted after arm;
  parity updates to `tests/test_capacity_snapshot_parity.py` and the
  `run_agent_runner_slot_*` suites.
- _Read first_: `sase/memory/lint_and_test.md`, `sase/memory/tui_perf.md` (the admission
  poll is hot).

### 4.4 hold-cli — `sase agent hold` command group

Full capability behind a watchable CLI before any prompt surface (the risky runtime
proves itself here; this is also the cheap experiment for whether prompt sugar is worth
it — but the user has asked for the directive, so subsequent phases proceed).

- Subcommands (alphabetical; group gains `list` as bare default automatically via
  `default_list_subcommands` — just document it, per `sase/memory/cli_rules.md`):
  - `create` — positional selectors (names, `@tribe`); options `--future`, `--hood`,
    `--pending`, `--scope`, `--ttl` (each with a short alias; none required; at least
    one selector or a hard error naming them). Freezes the `pending` snapshot
    (WAITING|QUEUED per `PRE_RUN_WAIT_STATUSES`, `src/sase/agent/status_buckets.py:31`)
    at arm.
  - `list` — active holds with armer, selector summary, capture counts, expiry
    countdown; `--json`.
  - `release` — positional armer key, or the caller's own hold by default when run
    inside an agent.
  - `run` — arm, exec a command, release in `finally`; the quiesce recipe is
    `sase agent hold run --scope host --ttl 30m -- <cmd>` (optionally after
    `sase agent wait -a -t 2h` to drain named running work).
  - `show` — one hold in full, including the frozen artifact-dir set.
- Armer identity: inside an agent, the agent (family) is the armer; outside, a `cli`
  armer keyed and liveness-checked by pid.
- Notifications via `upsert_notification` (`src/sase/notifications/store.py:177`) with
  `dedup_key`s: one on arm (enumerating the capture: "captures 4 waiting + 2 queued;
  skips 3 running") and one on expiry/dead-armer release. Emit from the shared
  arm/release helpers so phase 4.5 reuses them.
- Register in `src/sase/main/parser_agent.py` (`_AGENT_SUBCOMMAND_ORDER`), nested-group
  template `parser_agent_tribe.py`. Follow every rule in `sase/memory/cli_rules.md`
  (sorted help, short aliases, colored output, options never required).
- _Tests_: each subcommand; bare-group delegation notice; arm→run→release-in-finally
  including command failure; a held launch actually parks and releases end-to-end.
- _Read first_: `sase/memory/cli_rules.md`, `sase/memory/lint_and_test.md`.

### 4.5 hold-directive — the `%hold` prompt surface

_Cross-repo._ The directive is sugar over the phase-4.2 store using the phase-4.4
helpers, gated by a new **beta** flag.

- Create the flag with `sase flag new agent_holds -k beta …` (epic scaffolding — the
  sanctioned `sase flag new` door, not `/sase_new_task`; removed by phase 4.10). Wire it
  into `launch_feature_flag_keys()` (`src/sase/xprompt/queue_directive.py:264`), the
  collector gate (pattern: `_collect_code_directive` at
  `src/sase/xprompt/_directive_collect.py:246` + the message pattern in
  `code_value.py`), and Rust `directive_feature_flag` (`editor/wire.rs:636`:
  `"hold" => Some("agent_holds")`).
- _Python parsing_: `_KNOWN_DIRECTIVES`, `_MULTI_VALUE_DIRECTIVES` (+`hold`), a
  `PromptDirectives` field (`src/sase/xprompt/_directive_types.py`), a collector for
  positional names/`@tribe`/`pending`/`future` and keywords `hood=`, `tribe=`, `ttl=`,
  `scope=`. Bare `%hold` / `%hold(all)` → `DirectiveError` naming the selectors. No
  alias.
- _Rust contract_: `DIRECTIVES` entry (`editor/directive.rs:532`; keyword spec beside
  `WAIT_KEYWORDS:479`) with `positional_role: Agent`, positional suggestions
  `pending`/`future`, keywords `hood=` (new `DirectiveValueRole::Hood` variant,
  `editor/wire.rs:444` — candidate synthesis is phase 4.6), `tribe=` (Tribe), `ttl=`
  (Duration), `scope=`; update the sibling per-name matches (`directive_body_kind`,
  `directive_synopsis`, `directive_examples`, snippet recipes) and the parity-test
  keyword tuples in `tests/test_xprompt_directive_contract.py`.
- _Launch integration_: arm at launch submission (active from arm), through the shared
  helper — armer kin excluded, explicit self/kin names rejected with a clear error;
  arming defaults the launch's wait priority below `DEFAULT_WAIT_PRIORITY` unless
  `%q(p=…)` is authored. `%hold` is allowed on `%proc` units (do **not** add it to
  `proc_forbidden_directives`) so stand-alone proc prompts can arm; pre-dispatch the
  armer key is the proc shell name, rebound to the proc id at dispatch; release relies
  on the phase-4.3 settlement hook.
- _Safety_: reject with `%dispatch` (`validate_dispatch_combinations`,
  `agent_launch/mod.rs:2898` — new participant) and `%repeat`
  (`src/sase/agent/repeat_launcher.py::_validate_repeat_prompt_directives`); reject
  in-plan cycles at plan time (typed planner sees both units); `approval_preview`
  (`plan_typed_launch_units` → `src/sase/agent/launch_preview.py:143`) always enumerates
  the capture; `future && scope=host` or a frozen capture above a new config threshold
  raises a confirmation gate on interactive launches (`/sase_gate` pattern host-side).
- _Tests_: both flag states (off: parse error names the flag; on: full behavior); every
  selector shape; rejection matrix; arm-at-submission while the armer is itself parked
  on `%wait` (the quiesce composition `%proc … %q:1 %hold(pending, future)` drains and
  fences correctly — uses phase 4.1); digest/preview snapshots.
- _Read first_: `sase/memory/xprompts.md`, `sase/memory/sase_flags.md`,
  `sase/memory/lint_and_test.md`.

### 4.6 hold-completion-lsp — one shared contract, every editor

_Cross-repo._ The prompt-widget and external-editor support the user asked for.

- _Rust_: candidate dispatch for `DirectiveValueRole::Hood` (`editor/completion.rs:1760`
  role match) — synthesize hood rows from live agent-name prefixes in the existing
  inventories (`DirectiveCompletionInventories`, `editor/wire.rs:1121`; add a field only
  if synthesis from `agents` is insufficient); `%hold` agent rows ranked
  WAITING/QUEUED-first with status in `detail`; **procs included** in `%hold` candidates
  (unlike `%wait` — the exclusion at
  `src/sase/ace/tui/widgets/directive_completion.py:58` is wait-specific and must not
  apply to hold).
- _ACE_: generalize the wait-style clause handling in
  `src/sase/ace/tui/widgets/_directive_completion_tokens.py` (the `== "wait"` literals
  at :58, :153, :194, :336) to a contract-driven set covering `hold` rather than a
  seventh literal; extend `_directive_completion_agents.py` candidate building.
- _LSP_: `sase-core:crates/sase_xprompt_lsp` — inventories and catalog plumbing
  (`server.rs:723/:763`, `catalog_cache.rs:306`); editor-helper catalog
  (`src/sase/integrations/_editor_helper_agents.py`) if a `hood` row kind is added to
  `AgentCompletionEntry` (`editor/wire.rs:254`; kind doc currently
  agent/family/clan/tribe).
- _Tests_: contract/completion tests beside `editor/directive.rs`'s existing suites; ACE
  token tests; LSP catalog tests; parity test tuples.
- _Read first_: `sase/memory/lint_and_test.md`.

### 4.7 hold-proc-targets — fence un-dispatched procs

Stand-alone procs as **targets**. A pending proc unit has no proc id yet
(`launch_proc_runtime.py::_submit_unit` mints it at dispatch), so it is fenced at the
plan-admission boundary, not the runner-slot gate.

- Evaluate the hold predicate at the admission-engine eligibility transition
  (`sase-core:crates/sase_core/src/agent_launch/admission.rs::next_admission_actions`,
  the Waiting→Eligible arm at `:281-298` / Eligible dispatch at `:307`), fed hold
  records the same way wait facts flow in
  (`src/sase/agent/launch_admission_engine.py:100` / `launch_admission_runtime.py`).
  Lexical selectors (the proc's `%id` shell name, hoods, tribes) and `future` hold it;
  the frozen `pending` snapshot cannot capture un-dispatched procs — a **documented
  limitation**, stated in CLI/directive help.
- No `proc=<id>` selector exists anywhere in the feature: a proc id exists only once
  dispatched (running), and running work is never held, so it could never match anything
  holdable. Proc targeting is by shell name.
- A dispatched (running/terminal) proc is immune, mirroring agents.
- _Tests_: all four armer/target kind combinations complete the matrix (agent→proc,
  proc→proc, cli→proc); a pending proc named by a hold parks pre-dispatch and dispatches
  on release; a dispatched proc is untouched; `future` fences a proc submitted after
  arm.
- _Read first_: `sase/memory/lint_and_test.md`.

### 4.8 hold-visibility — see every hold, everywhere

Visibility is part of the feature, not polish.

- Queue marker gains `held_by` (+ expiry), written by the candidate's **own** runner
  (same writer, same file — no third-party writes); surface through `AgentWaitInfo`
  (`src/sase/integrations/_agent_list_entry_models.py:18`) and `agent_list_entries.py`;
  Agents tab renders `QUEUED · held by <armer>` in the QUEUED branch of
  `src/sase/ace/tui/widgets/_agent_list_render_agent_status.py:103-130` (extend the
  render cache key at `_agent_list_render_cache.py:317`); `sase agent list -j` exposes
  it.
- TUI hold panel modeled on the runner-limit override panel
  (`src/sase/ace/tui/modals/models_panel_runner_limit*.py`, registered via
  `config_hub_catalog.py`): list active holds with capture counts and expiry, release
  action.
- `sase doctor` stale-hold check (`CheckSpec` pattern, `src/sase/doctor/runner.py:112`;
  template `checks_flags.py`): flags active holds with dead armers or past-expiry
  records that failed to prune.
- Deadlock notification: at the runner-slot gate, when a candidate is held by an armer
  that is itself pre-run and blocked on that candidate (directly or via its `%wait`
  set), upsert a terminal-blocked-style notification with a `dedup_key` (precedent:
  `src/sase/scripts/sase_chop_wait_checks.py:358-395`); the TTL remains the
  forward-progress guarantee.
- _Tests_: render snapshot for the held row; cache-key invalidation; panel actions;
  doctor check both clean and stale; deadlock notification dedups.
- _Read first_: `sase/memory/tui_perf.md`, `sase/memory/lint_and_test.md`.

### 4.9 wait-hood — `%wait(hood=…)`

_Cross-repo._ Hood is the one grouping `%wait` cannot target even though the matcher
exists (`agent_name_in_hood`). With `Hood` role and hood candidates landed (4.5/4.6),
close the gap.

- Rust: `hood` keyword in `WAIT_KEYWORDS` (`editor/directive.rs:479`) with the Hood
  role; contract siblings; parity tuples
  (`wait: ("agent","bead","hood","proc","time","unit")`).
- Python: wait parsing accepts `hood=`; resolution semantics: a hood wait resolves when
  **every current member of the hood at waiter-launch time** has settled (reuse the
  cutoff pattern `newer_than=waiter_launch_cutoff` at
  `src/sase/core/wait_dependency_resolution/_resolution.py:76`, and the family/tribe
  candidate queries in `_index_queries.py`); an empty hood at launch resolves
  immediately with a diagnostic.
- _Tests_: hood wait across family/clan members; component boundary; empty hood;
  completion rows; contract parity.
- _Read first_: `sase/memory/xprompts.md`, `sase/memory/lint_and_test.md`.

### 4.10 hold-flag-removal — make it unconditional and close out

Per `sase/memory/sase_flags.md`, the `agent_holds` beta flag is epic scaffolding: this
phase deletes the Off branch, makes the On branch unconditional, removes the registry
entry (`src/sase/feature_flags/registry.py` + `launch_feature_flag_keys`), and closes
the flag bead in the same change.

- Run `just check-full` as the epic's landing gate and fix anything it surfaces.
- Append to this phase's bead:
  `PROPOSED FOLLOW-UP: memory — document %hold and %queue-on-procs in sase/memory/xprompts.md (directive table row, queue-on-proc note, hold semantics one-liner) and record the hold-store design (pull-model, fail-open, TTL, family- scoped release) as a decisions web strand.`
  The land agent turns this into a `memory` task bead; phase workers never create beads
  and never edit memory files directly (memory edits were not user-authorized in this
  epic).
- _Read first_: `sase/memory/sase_flags.md`, `sase/memory/lint_and_test.md`.

## 5. Global testing matrix

Every phase adds its own tests; before the epic lands, the union must cover: all four
armer/target kind combinations; WAITING, QUEUED, and pending-proc targets held while
RUNNING is never affected (the defining invariant); the arm-versus-claim race in both
lock orders; every selector including the hood component boundary; self / family / clan
/ dotted-descendant exclusion; `future` before and after armer settlement; TTL,
dead-armer, kill, and malformed-store fail-open release; multiple holds on one target;
the deadlock notification; both feature-flag states; ACE/LSP completion parity;
`%repeat` / `%dispatch` rejection; the quiesce composition
`%proc … %q:1 %hold(pending, future)`.

## 6. Non-goals and recorded limitations

- No `drain=` keyword on `%hold` — draining is `%queue`'s vocabulary (v2 candidate if
  selector-scoped drain of unnamed running agents proves needed).
- No `proc=<id>` selector, ever (see 4.7).
- The frozen `pending` snapshot cannot capture un-dispatched procs (documented in help;
  lexical + `future` selectors cover them).
- No global completion-graph cycle detection — the three proportionate layers plus TTL
  (see §2 Deadlock).
- No short alias for `%hold` in v1; broad scheduling effects should be visible.
- Hold release keys off terminal settlement only; it must not inherit the sase-11k
  `Launched`-counts-as-done defect.
