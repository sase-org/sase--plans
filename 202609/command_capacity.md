---
tier: epic
title: Command capacity reservations and fleet load meters
goal:
  Account for expensive commands through reliable shared admission, require explicit
  machine capacities, and make every enabled machine's load clear in Agents.
phases:
  - id: baseline
    title: Establish integration prerequisites and acceptance fixtures
    size: medium
    depends_on: []
    description:
      "baseline: record the current weighted-capacity and monitor contracts, add the
      external bead dependencies described below to the affected materialized phases,
      and establish representative accounting, verification, and UI fixtures without
      enabling new behavior."
  - id: contracts
    title: Define command reservations and ownership in Rust
    size: medium
    depends_on:
      - baseline
    description:
      "contracts: implement the versioned Rust reservation, shell-role accounting,
      numeric, transition, snapshot, and profile-input contracts with PyO3 parity tests.
      Keep launch-time queue_weight immutable."
  - id: leases
    title: Supervise command reservations and recover their lifetime
    size: medium
    depends_on:
      - contracts
    description:
      "leases: implement host locking, durable publication, argv execution,
      descendant-aware cleanup, nested grant validation, and standalone owners using the
      shared Rust policy. Prove cancellation, startup failure, and crash recovery with
      real subprocesses."
  - id: fair-queue
    title: Admit agents and tool requests through one fair queue
    size: medium
    depends_on:
      - leases
    description:
      "fair-queue: extend the Rust queue and Python admission adapter to include parked
      tool requests, bounded bypass, persistent aging protection, impossible-request
      handling, capacity changes, and ordinary-agent readmission."
  - id: monitors
    title: Integrate queued tools and independent monitor weights
    size: medium
    depends_on:
      - fair-queue
    description:
      "monitors: integrate with the completed sase-zl lifecycle, add explicit monitor
      weight, release provider capacity only after an acknowledged handoff, count
      wrapped monitor demand without the agent baseline, and preserve result delivery
      and prepared-completion semantics."
  - id: cli
    title: Deliver the tool command and useful admission diagnostics
    size: medium
    depends_on:
      - monitors
    description:
      "cli: implement tool run/status, argv separation, queue/force choices, dry-run
      explanations, structured outcomes, and CLI completion/help using the tested
      lifecycle and policy contracts."
  - id: initialization
    title: Require persistent machine capacity during initialization
    size: medium
    depends_on:
      - contracts
    description:
      "initialization: extend config init and sase init completion, source-aware
      capacity validation, doctor, and temporary-override reporting while preserving
      machine identity and YAML. Prepare but do not activate the requested 32/8/4
      configuration changes."
  - id: profiles
    title: Price command plans from real verification work
    size: medium
    depends_on:
      - cli
      - initialization
    description:
      "profiles: implement versioned SASE-repository profiles, the specified initial
      demand table, setup/build preflight, and selection-driven check stages. Record
      estimated work, granted workers, and fingerprints; keep policy in Rust and
      repository evidence collection in thin tooling."
  - id: pytest-admission
    title: Connect pytest concurrency to the shared capacity grant
    size: medium
    depends_on:
      - profiles
    description:
      "pytest-admission: adapt scoped, full, coverage, cost, direct pytest, and nested
      test runs to validated shared grants on managed hosts; enforce worker limits,
      preserve required selection, and implement the deliberate serial fallback for
      refused optional middle gear."
  - id: recipe-admission
    title: Govern recipes before setup and make bootstrap safe
    size: medium
    depends_on:
      - pytest-admission
    description:
      "recipe-admission: put the seven public recipes and expensive setup/build entry
      points behind admission before dependencies execute, reuse outer grants without
      recursion, bound build concurrency, preserve diagnostics, and implement the
      managed-host migration/version barrier."
  - id: fleet-snapshots
    title: Publish cached capacity for each enabled machine
    size: medium
    depends_on:
      - leases
      - initialization
    description:
      "fleet-snapshots: extend the Rust fleet worker and wire contracts with
      machine-wide load/capacity snapshots, freshness, source and diagnostics; consume
      them through existing bounded background refresh and invalidation paths without
      deriving load from visible agents."
  - id: meters
    title: Render accessible machine badges and load meters
    size: medium
    depends_on:
      - fleet-snapshots
    description:
      "meters: replace the top-right fleet sentence with one compact badge/meter/ratio
      per enabled machine, remove the old left capacity prefix, preserve standalone
      visibility and exact fractional formatting, and verify responsive layout,
      accessibility, PNG scenes, and navigation/idle cost."
  - id: guidance
    title: Replace static workload weights and synchronize guidance
    size: medium
    depends_on:
      - recipe-admission
      - monitors
      - meters
    description:
      "guidance: remove the four research-swarm quarter weights and lander weight two,
      update the requested lint_and_test memory and authoritative monitor skill,
      regenerate memory instructions, and prepare canonical post-landing skill/config
      deployment with package compatibility tests."
  - id: acceptance
    title: Validate and activate the coordinated fleet rollout
    size: medium
    depends_on:
      - guidance
      - recipe-admission
      - meters
    description:
      "acceptance: run the integrated lifecycle, managed-pytest, bootstrap,
      fleet/visual, packaging and performance matrix; collect representative cost
      evidence; drain legacy admission before the per-machine cutover; activate athena
      32, apollo 8 and the existing Mac identity 4 only with the complete protocol."
proposed_by: bbugyi200.athena.0jh
create_time: 2026-09-11 11:21:35
status: wip
---

- **PROMPT:**
  [prompts/202609/command_capacity.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/command_capacity.md)

# Command capacity reservations and fleet load meters

## Outcome and scope

Expensive commands consume temporary capacity for exactly their supervised lifetime.
Ordinary agents keep their declared base weight, normally one. On athena, a full check
adds 15 to an inline agent, so its live weight is 16. When the same check runs in a
monitor, the provider has ended and the monitor consumes 15. An unwrapped monitor
defaults to one, independently of its predecessor's weight. All enabled fleet machines
have their own prominent badge, utilization meter and integer-preferred load/capacity
ratio in the top-right Agents header.

This is an epic: ownership and admission belong in Rust core, process control in host
adapters, test/build integration in repository tooling, and rendering in Textual. The
phases are bounded direct implementation work. This authoring turn changes no product,
configuration, memory or skill source; it creates and proposes this scratch plan only.
An immutable copy of the requested existing research was registered solely to enable an
audited read after its sidecar reference failed to resolve.

Capacity means reserved execution demand, not measured CPU utilization, physical cores,
provider quota, or a pooled fleet budget. The machine capacities are athena 32, apollo
8, and the Mac 4. Preserve its existing identity, currently `kellys_mbp`; do not rename
it to `mac`. No remote dispatch, automatic migration between machines, cgroup resource
enforcement, or general multidimensional scheduler is included.

## Evidence, repositories, and concurrent work

Read the original sources through the required skills when implementing:

- `plan:202609/weighted_queue_capacity.md`,
  `plan:202609/weighted_capacity_landing_repairs.md`, and current
  `sase bead show sase-z4` / `sase-z4.6`: immutable launch weights, explicit claim
  lineage, atomic admission and compensated decimal summation are the baseline. Parent
  acceptance is still in progress at authoring; old failure notes are not fresh
  reproductions.
- `plan:202609/monitor_continuations.md` and current `sase bead show sase-zl`:
  structured results, continuation graph, durable delivery and prepared host completion
  are owned by that epic. Its result/dispatch/completion/experience/release phases are
  still running at authoring.
- Requested report
  `research:202609/command_capacity_and_load_meter/command_capacity_and_load_meter.md`,
  read as `file:explicit:ac538c08bf1b84d3a9491b12` after configured sidecar lookup
  failed. This is an unchanged source snapshot, not newly measured evidence.
- Current `Justfile`, `tools/run_pytest`, `tests/_suite_gate_budget.py`,
  `tests/_test_selection*.py`, `src/sase/core/runner_slots/`,
  `src/sase/axe/run_agent_wait_slots.py`, `src/sase/monitor/`,
  `src/sase/main/config_init_handler.py`, and Agents fleet/header adapters.

Work in the current SASE checkout and open every other repository with `/sase_repo`:
`sase-core` (or `gh:sase-org/sase-core`), `sase-research-artifacts` (or its GitHub
external ref), and `chezmoi`. Use only returned paths, read each repo's AGENTS.md, and
use `sase artifact read` for sidecar artifacts. Linked aliases failed during authoring;
core, the research plugin and research repository were opened through the external
fallback. Re-resolve chezmoi through the supported repo command before its source
changes; never substitute a guessed home checkout. Use repository-relative locations in
durable descriptions and configure `SASE_CORE_DIR` from the opened core path for local
verification.

The baseline worker must add real external dependencies to the materialized phase beads
before completing baseline: `contracts` depends on `sase-z4`; `monitors` depends on
`sase-zl`. Use `sase bead dep add <phase-bead-id> <prerequisite-bead-id>` after reading
the bead skill/memory. These are external prerequisites in addition to frontmatter
dependencies; do not place external IDs in `phases[].depends_on`. All later consumers
inherit these barriers. Baseline may record fixtures and source evidence while those
epics run, but no worker should edit their contested runtime/monitor sources early. Do
not message their authors, close their beads, or relitigate their acceptance work.

Rebase/revalidate against their accepted revisions. In particular, keep sase-zl's exact
result identity, context budgeting, deduplicated next actions, stop semantics and
prepared completion. Teach its command recognizer that a validated tool envelope wraps
the original verification argv; never infer success from a shell string containing
`just check-full`. No second continuation graph or output-retention policy is
introduced.

## Public command contract

Canonical usage:

```sh
sase tool run -- just check
sase tool run -- just check-full
sase tool run -q -- just check-full
sase tool run -f -- just check-full
sase tool run -w 2.5 -- expensive-command --its-own-option
sase tool run -n -- just check
sase tool status
sase tool status -j
```

Require the `--` delimiter. Before it, options belong to SASE; after it, preserve argv
exactly and execute without an implicit shell. Thus `sase tool run -f -- command -f`
forces admission and passes a separate flag to the child. An explicit `sh -c` command is
supported with an authored weight; arbitrary shell text is not profile-matched. Unknown
commands need `-w/--weight` or a recognized `-p/--profile`. Match automatic profiles
using repository identity, recipe, cwd, and supported relevant arguments, including
`just` overrides; reject ambiguous variants with a useful explicit-weight remedy instead
of guessing by executable basename. Preserve custom justfiles and recipe arguments
without silently assigning the SASE repository's workload price.

`-w/--weight` is command demand: additional units for an active agent, total units when
there is no active provider baseline. Positive finite decimals use the existing Rust
weight validator. Reject bool, zero, negatives, NaN, infinity, overflow and underflow to
zero. Built-in profiles may request zero extra for bounded serial work already covered
by an inline baseline; a live standalone/monitor command still counts at least one.
Weight and profile are mutually exclusive. Explicit weight never changes meaning by
machine and is never silently clamped.

`-q/--queue` and `-f/--force` are mutually exclusive. Ordinary invocation attempts
admission once at each required stage. If initial admission does not fit, it starts no
child, retains the inline caller's base claim, exits 75, and emits a structured
`capacity_unavailable` record with `child_started: false`. A later stage refusal also
exits 75 but reports the earlier completed stages and `child_started: true`; the blocked
stage has `stage_started: false`. Example of an initial refusal:

```text
Cannot start just check-full on athena.
Load 20/32 includes your base 1.
Needs +15; your running weight would be 16, machine load 35/32.

Queue:  sase tool run -q -r <request-id> -- just check-full
        Hands off this agent; the running monitor will use 15.
Force:  sase tool run -f -r <request-id> -- just check-full
Details: sase tool status
```

These are executable choices, not a stdin prompt or new approval gate. Preserve the
original cwd and correctly shell-quote the reproduction argv. A request larger than
capacity cannot enter an impossible queue: explain exact demand and offer a smaller
profile/weight or force. Queued agent calls are checked against their eventual monitor
demand, not the inline total; a command that only fits after shedding the provider base
can queue successfully. Force bypasses only capacity and fair-queue admission, still
publishes the whole demand and records force provenance. It cannot bypass bad metadata,
invalid ownership, an unavailable ledger, or nested concurrency restrictions.

`-r/--resume REQUEST` identifies the exact refused operation for either retry. Error
output substitutes the actual request ID. Validate owner, cwd, original argv and
fingerprint; reserve resumption atomically so duplicate retry calls cannot execute it
twice. This reuses valid completed-stage receipts without replaying mutation. A normal
run without `--resume` is a new operation, never an implicit reuse of old success.
Resume applies to capacity-refused operations, not an already-started arbitrary failed
command; terminal monitor delivery continues to use sase-zl's resume facilities.

`-n/--dry-run` performs read-only preflight and explains profile/version, setup/build
requirements, test selection/reasons, predicted time, worker ceiling, baseline, command
demand, inline/monitor totals and projected local load. It never acquires a claim,
starts compilers/tests, creates a monitor, or repairs an environment. Unknown setup
facts are shown as conditional conservative costs. `-j/--json` on preflight and status
emits a versioned envelope; reject `run -j` without dry-run so child stdout stays
unambiguous. Runtime metadata provides structured outcomes for other integrations.
Wrapper messages go to stderr and child stdout/stderr remain usable, streaming and
bounded only by the monitor's established capture rules.

Add optional `-t/--timeout` for execution and `-T/--queue-timeout` for waiting. Queue
wait does not consume execution or idle timeout; omitted queue timeout waits until
grant/cancellation, and the existing monitor execution default applies to queued work.
Explain defaults in help. Inside an existing monitor, its execution deadline applies
once work starts; if both supply execution deadlines, the earlier one wins. Never
interpret intentional silence while queued as a hung command.

`tool status` shows this machine's effective load/capacity and source, holders, command
labels, base/demand, queued target and age, forced/uncertain state, and monitor/request
IDs. Use `sase monitor show/stop` to inspect/cancel durable queued work; do not build
another queue-management command group. Bare `sase tool` prints concise group help. Sort
public subcommands/options and give every new public long option a short alias; update
shell and editor completion and default config where appropriate.

Preserve child exit status; record child signal, exit and whether it started separately
from wrapper admission failure. A child's exit 75 is a child failure, not a capacity
refusal. `just` and shell layers may rewrite exit codes, so monitor/verification
integration consumes typed records rather than interpreting a numeric 75 alone.

## Ownership, lifetime, and queue policy

Keep `queue_weight` immutable and retain its authored/inherited provenance. Add a
versioned command-reservation model linked to Rust's authoritative claim owner, not a
display family name. For one active serial lineage:

| Executing state                     |                                       Capacity held |
| ----------------------------------- | --------------------------------------------------: |
| Provider agent with base B          |                                                   B |
| Provider running tool demand D      |                                               B + D |
| Wrapped monitor, provider ended     | D; one only when a built-in serial profile uses D=0 |
| Unwrapped monitor                   |              Explicit monitor weight M, otherwise 1 |
| Parked monitor waiting for capacity |                                                   0 |
| Standalone tool                     | D; one only when a built-in serial profile uses D=0 |

For fractional explicit tool demand, wrapped monitors also use exact D; the minimum-one
rule is only for built-in profiles whose computed extra demand is zero. For example,
explicit `-w 0.25` contributes 0.25 in a monitor and B+0.25 inline. Terminal monitors
hold zero. A successor provider resumes with its family's original B, never the
monitor's demand. Sum independently admitted parallel lineages and standalone owners;
deduplicate overlapping records only by verified claim identity/generation. Do not apply
the old maximum-base-weight aggregation to temporary command increments or count a
provider merely because its family row looks running.

Expose `sase monitor start -w/--weight N` as an explicit total for an unwrapped monitor.
Do not inherit the starter's queue weight. For a directly wrapped tool invocation,
derive monitor demand from its tool plan and reject a simultaneous explicit monitor
weight, naming the tool's `-w` as the single override. Prevent an unwrapped monitor
script from silently double-counting a nested tool: negotiate an atomic transition from
monitor base to tool demand before its first expensive child starts. An explicitly
weighted shell script owns that fixed outer envelope; nested tools may only reuse
compatible capacity within it. Show requested/inherited agent base separately from the
monitor's effective held weight in rows and details.

Rust owns validation, numeric aggregation, ownership, legal transitions, fairness,
profile decisions and serializable projections. Host adapters own OS process identity,
bounded locking, filesystem/IPC, process groups, acknowledgement and durable writes. Use
the existing host admission lock and change-token signaling for scan/check/publish,
transfer and release; no Python scheduler or independent tool counter. Never hold the
admission lock while spawning, waiting, performing network I/O or probing a large tree.

The durable reservation contains machine/boot identity, owner lineage and shell role,
reservation ID/generation, original base, requested/granted demand, capacity source,
worker allowance, profile and source fingerprints, command argv/cwd identity, supervisor
and child birth identities, lifecycle timestamps, requested priority, force flag and
typed outcome. Publication makes an acquired claim visible before spawning work. Spawn
failure releases only that reservation, idempotently. Preserve original command argv
under existing artifact access rules; omit secret-bearing environment dumps and redact
displayed values using existing command-rendering policy.

Initially allow one executing top-level tool per serial lineage. Reject a second
concurrent sibling command before spawn; users can launch independent parallel agents
for independent work. Nested recipe/test invocations may reuse a parent grant only after
verifying the live ID, generation, command scope and process ancestry through core. An
environment variable is a locator, not admission proof. A nested request that exceeds
its envelope fails before starting and asks for a larger outer reservation. No in-place
hold-and-wait upgrades. Sequential stage transitions are allowed only after all
preceding stage descendants have exited and their tool increment is released.

Inside an agent, `-q` always means a durable monitor handoff, even if it fits
immediately. Reserve the request and supervisor, persist the continuation checkpoint
through sase-zl, receive acknowledgement, then end the provider. Only after confirmed
handoff may the host atomically release B and grant the monitor D (or park at zero). Do
not run the command while the predecessor is still doing uncounted work. Failed
acknowledgement returns an error to the live agent, restores/retains its exact claim and
workspace, and starts no command. Retain workspace ownership throughout a parked wait.
Inside a monitor, wait within that supervisor; do not recursively launch monitors.
Standalone queueing uses the same durable command supervisor without fabricating an
agent or launching a provider successor.

Publish waiting state as `WAITING FOR CAPACITY`, with requested demand, current free
capacity and queue age. Queued reservations do not fill the load meter. When admitted,
switch to the monitor's actual execution label and start the execution timer. Queued
generic commands use `RUNNING`/`FINISHED`; recognized checks use `TESTING`/`TESTED`.
Agent-origin queue handoffs record a concise automatic next instruction to inspect the
result and continue the interrupted user task through sase-zl's existing result graph;
do not copy the transcript or rerun the tool in that instruction. Preserve prepared host
completion when eligible. Stopping a queued/running monitor follows sase-zl's stop
policy, including no new follow-up after explicit stop.

One queue orders agent launches and tool requests. Preserve priority/FIFO and bounded
deference initially; allow fitting younger work to pass a blocked heavy request until
120 seconds of capacity-blocked eligible time. Then protect that oldest aged request
from all newer competing admissions, including new agents, until its frozen target fits.
Aging excludes unresolved dependencies/time floors. Persist age/protection across polls
and supervisor restart, expose it in status, and do not repeatedly reset it.
Cancellation, impossible requests and dead owners remove protection. Earlier protected
requests retain precedence over subsequently aged ones; force remains an explicit
exception. Existing running work is never preempted. Progress assumes running jobs
eventually release capacity and force is not used continually.

Cancellation/queue timeout does not restore a provider base into a full machine. Any
eligible follow-up uses ordinary admission. Inline completion releases only D and
retains B. Monitor completion releases monitor demand; a subsequent provider reacquires
B through shared admission, with an atomic transfer only when policy proves it fits.
Lowering capacity never kills admitted work; show overload and block new demand.
Revalidate queued demand/fingerprints before granting, retaining request age for the
same operation; never execute a stale cheaper plan. If a changed plan becomes
impossible, settle a useful refusal instead of waiting forever.

Keep deterministic compensated summation and the existing at-most-four-ULP comparison
bound on proposed total versus capacity. Formatting never participates in admission.
Invalid live reservations or unrepresentable aggregate load fail closed with a
diagnostic. Missing fields in valid legacy records retain their documented old meaning;
malformed present fields cannot become zero or a default claim.

Supervise foreground process trees. SIGINT/SIGTERM, timeout, failed spawn, wrapper
death, supervisor death, and reboot must not leak capacity or release it while
descendants still run. Record boot and process-birth identity to avoid PID reuse.
Reconciliation retains uncertain demand while it investigates or terminates surviving
descendants; confirm tree exit before release. No age-only expiry for a living holder.
Kernel holder locks can support this protocol, but are insufficient proof of child
death: Linux flock handles can survive exec and duplicated descriptors. See the
[flock manual](https://man7.org/linux/man-pages/man2/flock.2.html). Test the supported
process/holder mechanisms on Linux and macOS; daemonizing escape from the supervised
foreground tree is unsupported and documented.

## Initial command demand and dynamic check policy

These are explicit initial policy choices, not measured peak-resource claims. D is
command demand, normally the increment on an agent with B=1; the inline total is 1+D.
The monitor/standalone total is D, except zero-extra serial profiles use one there. The
requested athena +15 is fixed in the initial full profile.

| Recipe/path                               | Athena D (inline total) | Apollo D (inline total) |                      Mac D (inline total) |
| ----------------------------------------- | ----------------------: | ----------------------: | ----------------------------------------: |
| install: cached wheel/no source build     |                   1 (2) |                   1 (2) |                                     1 (2) |
| install/setup: Rust or other source build |                   7 (8) |                   7 (8) |                                     3 (4) |
| fmt: prepared dependencies                |                   1 (2) |                   1 (2) |                                     1 (2) |
| lint: preferred envelope                  |                   3 (4) |                   3 (4) |                                     3 (4) |
| check: bounded serial scoped stage        |                   0 (1) |                   0 (1) |                                     0 (1) |
| check: optional parallel scoped stage     |            k for k=2..4 |            k for k=2..4 | k for k=2..3 inline, up to 4 in a monitor |
| check: required full escalation           |                 15 (16) |                   3 (4) |                                     1 (2) |
| check-full                                |                 15 (16) |                   3 (4) |                                     1 (2) |
| test                                      |                 15 (16) |                   3 (4) |                                     1 (2) |
| test-cov                                  |                 15 (16) |                   3 (4) |                                     1 (2) |

For other initialized machines, the default full profile uses reference total
`T = min(C, 16, max(2, floor(C/2)))` and D=T-1. At C=1, T=1: run serial with zero inline
extra or one unit as a monitor. C here is the persistent machine capacity; temporary
admission overrides change the fit limit, not the profile price. Source-build reference
total is min(8,C), cached install/fmt min(2,C), and lint min(4,C), with D=T-1 and the
same serial rule. Positive nondefault B is never shrunk or overwritten: inline admission
needs B+D and must refuse if it cannot fit. Explicit weight is exact and bypasses
profile price selection, while still enforcing any recognized adapter's worker envelope;
it does not disable admission.

Athena's current automatic pytest ceiling is 14 workers for a 32-token pool. Start with
that ceiling under D=15; apollo uses at most 3 workers under D=3, and the Mac is serial
under D=1. Additional CPU and available-memory checks may reduce concurrency. Treat D as
the entire monitor command envelope, including its controller. Reuse the existing
measured per-worker memory estimate as a conservative starting input; tune from fresh
evidence rather than claiming a fixed physical meaning for a capacity unit. For
arbitrary profile inputs, cap worker concurrency to the admitted command envelope with
at least one serial worker when serial work is covered by the provider base. Explicit
conflicting `-n`, job-count arguments or environment overrides must be rejected or
deliberately replanned before spawn, never silently widen workers.

At 32, at most two full checks can run concurrently. Two inline checks use all 32; two
full-check monitors use 30 and can coexist with two ordinary agents. More unrelated load
may reduce the number admitted. This is not a guarantee of two available slots.

Keep generic explicit-weight commands and `check-full`/`test`/`test-cov` under one fixed
peak envelope for their child lifetime in v1. If cold setup needs more than that
envelope (notably on smaller machines), preflight reserves the larger peak and explains
it; tests still use their own smaller worker limit. This avoids a hidden nested build
upgrade. `check`, install and setup integrations can use explicit safe stage boundaries.

For dynamic `check`, adapt existing `Selection`, `gear_candidate`, timing estimates,
rule reasons and manifest fingerprints. Do not implement a changed-file-count heuristic
or move the repository's import-graph selector into product core. Repository tooling
collects evidence; Rust chooses demand and a legal lane from typed inputs.

1. Preflight setup first; a read-only probe must not invoke `_setup`. Account for
   dependency installation and any core/LSP/plugin source builds before executing them.
   Compute/validate test selection after the resulting source/core identity is known.
2. Prefer the lint envelope D=3. When that extra cannot fit, allow bounded serial lint
   with D=0 inline (one as a monitor), but only after auditing/enforcing thread limits
   of every participating lint tool. Do not make the standalone `lint` profile cheaper
   silently; explain any chosen fallback in the check plan.
3. An empty selection skips pytest. A cheap serial selection uses D=0 inline/one in a
   monitor, without acquiring the legacy pytest pool.
4. Only when serial-duration excess is the sole escalation cause may the middle gear run
   the same valid selection. Choose the smallest k in 2..4 predicted to meet the
   existing serial-time budget, bounded by selected work, current resources and machine
   capacity. Obtain a nonblocking grant for D=k. If no width fits, retain the same
   selection and run serial (or monitor that long serial stage). This deliberately
   replaces today's refused-middle-gear escalation to the full lane.
5. Broadening/correctness rules, invalid selection evidence or changed source identity
   retain the required full coverage. Request the full profile; unavailable capacity
   returns queue/force choices or parks within an already-authorized queued monitor.
   Never replace required full coverage with a cheaper subset. Missing timing evidence
   alone does not invalidate a sound selected set; it disables optimistic parallel
   pricing and uses the existing conservative selection decision.

Persist stages, argv/cwd, selected-set/source/config/profile fingerprints, completed
stage receipts, predicted duration, requested/granted widths, refusal/fallback reasons,
queue delay and actual duration. Resume a queued late stage only after its prerequisites
and fingerprints are validated. Do not automatically replay already-successful mutating
install/fmt/setup stages after a late refusal; invalidate dependent read-only
verification receipts when inputs changed and report any mutating prerequisite that
requires an explicit new operation. `run_silent` must forward admission control records
and keep their readable diagnostics visible. Feed sase-zl's structured stage evidence
and prepared completion recognizer the actual inner command and stage outcomes.

Before tuning the initial table, collect warm/cold install, fmt/lint, empty/small/medium
and full-escalating check, full test and coverage samples. Record wall/CPU time, maximum
actual concurrency, process-tree memory peak, queue time, force incidence and TUI
responsiveness with platform-specific measurement methods disclosed. Reuse required
landing checks for heavy samples; do not run many exhaustive suites just to calibrate.
Use fixture plans and existing timing history first. An unavailable remote machine gets
explicit pending measurement, not an invented benchmark. Keep athena's requested +15
default unless the user explicitly approves a later change; tighten actual worker limits
when safety/resource evidence warrants it. Record other tuning as versioned
profile/config changes with before/after evidence.

## Mechanical enforcement, bootstrap, and rollout safety

All seven named public recipes must acquire/reuse admission before expensive Just
dependencies execute. A wrapper only inside the body of `check` is too late because
`_setup` can compile Rust. Refactor into public admitted entry points and private
implementation stages; audit independently callable `_setup`, `rust-install`,
`rust-lsp-install`, required-plugin setup and other expensive bootstrap paths. Nested
`just` invocations validate and reuse the same grant. Profile matching and internal
recipe routing must avoid recursion and preserve invocation cwd and user arguments.

Use a stable installed SASE launcher/core runtime outside the workspace environment
being rebuilt for admission. Do not require importing a broken/missing workspace
extension to authorize its repair. Managed-host first bootstrap without a compatible
launcher fails with a bootstrap/install remedy, never unrestricted compilation. Document
a minimal isolated/unmanaged bootstrap using a released wheel and bounded build/install
concurrency, then require normal initialization before joining a managed host. Do not
add a Python fallback scheduler to solve bootstrap.

Bound compilers, source-build fan-out, install extraction and test workers according to
the grant. Cargo supports jobs controls and defaults to logical CPU count; uv exposes
separate build/install concurrency controls. Coordinate outer and nested limits so their
product cannot exceed the intended envelope. See
[Cargo build controls](https://doc.rust-lang.org/cargo/commands/cargo-build.html#miscellaneous-options)
and
[uv concurrency settings](https://docs.astral.sh/uv/reference/environment/#uv_concurrent_builds).
The limits constrain known adapters; an arbitrary explicitly weighted command remains
the caller's demand estimate, not an OS resource jail.

On a managed host, supported direct `just`, `tools/run_pytest` and `pytest` entry points
(including xdist setup in conftest) must obtain the same authoritative ledger grant,
whether invoked by an agent, human, monitor or shared-host CI. A standalone owner has no
imaginary provider baseline. A validated outer grant replaces the old suite-gate charge
and carries an enforceable worker allowance. Test children exercising SASE must use
isolated test state and narrowly verified descendant grants rather than inheriting real
admission authority into unrelated work. A bare environment bypass is never sufficient
to disable the ledger.

Isolated CI without initialized managed-host state may retain the existing pytest pool;
this is a distinct runtime environment, not a configurable second scheduler on a managed
machine. Represent this boundary explicitly and test it. Use the required sunset-flag
workflow if an old managed-host branch must remain during migration.

Cutover is machine-local and must be coordinated: prepare compatible core/SASE/plugin
releases, pause new legacy launches through host controls, allow legacy agents, monitors
and pytest holders to finish, verify the old pool and incompatible runner processes are
drained, replace/restart admission owners, and retire/reprepare old workspace entry
points through host-owned workspace preparation. Check protocol capabilities at managed
launch/bootstrap/test entry points and reject unsupported workspaces with an upgrade
remedy. A compatible launcher alone does not make old repository scripts participate.
The rollout must demonstrate no supported path can acquire the old independent budget
after activation. Do not silently kill active work to reach the barrier.

Only then activate the new ledger and that machine's explicit capacity. Do not raise
athena to 32 while old test admission can still coexist. If rollout cannot prove the
drain/version barrier, keep the previous capacity and stop activation with a concrete
diagnostic. Rollback also drains new holders before restoring legacy admission; never
switch ledgers underneath live commands. Persist migration state so a reboot cannot
forget which ledger is authoritative. This is an operational capability/version barrier,
separate from temporary feature scaffolding.

Earlier landed phases must be dormant library code/test seams. If partial public routing
is unavoidable, create a beta flag through `sase flag new` after reading
`sase_flags.md`; test both states and remove its Off branch/registry/bead before final
landing. Do not hand-author flags or create one during plan authoring. Host completion
owns commits/releases/deployment. Follow the approved release flow, preserve release-plz
ownership of Rust crate versions, and verify actual compatible published package floors.

## Initialization and configuration

Retain `max_running_agents` as the stored positive-integer capacity key for this change;
call it Machine capacity throughout UI/help. Decimal weights remain supported. Avoid an
unrelated key rename or fractional capacity schema migration. Add source-aware
validation: a packaged default, generic inherited default, malformed value, or temporary
override does not satisfy explicit selected-machine initialization.

Extend `plan_config_init` and its run path's currently identity-only early return.
Identity without a valid capacity in the selected machine overlay remains incomplete.
`sase init --check`, `sase config init --check`, onboarding and doctor explain the same
missing/invalid state. Gather and validate the capacity alongside identity before
writing, preserve comments/unrelated YAML, and keep the read-only plan/TTY/chezmoi
workflow. Repeated init is idempotent. Noninteractive init needs preconfigured capacity
or a supported explicit config edit; `--yes` never guesses machine policy. Do not add a
required option: required configuration is collected by the existing init interaction.

Prepare source edits for `home/dot_config/sase/sase_athena.yml` = 32, `sase_apollo.yml`
= 8 and the actual Mac overlay (currently `sase_kellys_mbp.yml`) = 4 in the opened
chezmoi repository. Verify identity in source before editing; retain machine-specific
deployment/ignore rules. These values come from the user, not an automatic CPU or
fluctuating free-memory calculation. General initialization may suggest a value with a
clearly identified source; an unknown machine still requires an explicit accepted value.

Persistent capacity and active temporary override stay separate. The denominator is the
same effective value used for admission; details show persistent value, override and
expiry/source. Init does not clear overrides. Keep the legacy ordinary-agent default 10
during any controlled preactivation compatibility interval and mark it as unconfigured
in details; new heavy tool requests require explicit initialization. After managed
activation, malformed/missing selected capacity produces unknown/error state and refuses
new capacity admissions, including forced ones, without interrupting running work or
breaking read-only inspection. Never silently increase an uninitialized machine's
concurrency.

## Fleet meter design and performance contract

Replace the top-right sentence with one capsule per enabled machine. Local comes first,
then other enabled machines in stable configured order. Deduplicate by authenticated
machine identity, not alias text. Fleet membership must not be limited to followed
agents, fetched catalog pages or machines with active work. These are independent
budgets: never render a misleading combined denominator 44.

```text
[athena · here] ▰▰▱▱▱▱▱▱  8/32   [apollo] ▰▰▰▱▱▱▱▱  3/8   [kellys_mbp] ▱▱▱▱▱▱▱▱  0/4
[athena · here] ▰▰▰▰▰▰▰▰ 35/32 ! [apollo] ▰▰▰▱▱▱▱▱ ~3/8  [kellys_mbp] ────────  —/4
```

The plain-text specimens specify hierarchy/state, not mandatory padding at every
terminal width. Use a subtle filled machine-name badge, a theme-aware eight-cell
single-fill bar, and bold load with quiet slash/denominator. Local `here` is a small
badge accent, not the old sentence. Normal usage is a calm cyan/green accent; near-full
and exactly-full are amber, overload is red with `!`. Bar fill clamps at 100%; the
number never does. Use pressure thresholds consistently (normal below 75%, amber from
75% through capacity). A forced reservation is disclosed in detail, and an actual
overload is marked even if formatted values would otherwise look equal. No animation,
base/tool textures, blinking or color-only state distinctions.

At reduced width, drop bars before ratios, then the local `here` accent. Preserve a
named ratio for each enabled machine and wrap capsules onto bounded rows when needed;
never hide machines behind `+N`. For unusually large fleets, allow the header region to
scroll with keyboard access rather than consume the entire Agents pane. Preserve a
stable header height for a given width/fleet membership so routine load updates do not
move selection. Long names get ellipsis with the complete name in keyboard-accessible
detail. Provide ASCII bar/status fallback and respect light/dark/NO_COLOR contexts. Keep
the ratio readable at 70/80/120 columns with the configured three-machine fleet.

Clicking a capsule or focusing/activating it by keyboard opens that machine's existing
detail surface (or a small reusable capacity detail panel if Machines has no compatible
surface). Show source/freshness, agent bases versus tools, top holders, waiting target,
forced/uncertain state and overload reason. Keep enabled-machine reachability warnings
accessible there, with a small stale/unknown marker on the capsule. Healthy local
capacity does not turn red because a remote machine failed. Preserve attention/status
counts in their existing left-side strip and details. Remove
`ProviderInfoPanel._append_capacity_prefix` and its call; the top-right capsules become
the only aggregate load/capacity meters on Agents.

Use integer-preferred formatting for both numbers: `8/32`, `6.75/32`, `0/4`.
Reuse/refine `format_capacity_value(..., minimum_decimal=False)` and common formatting
tests; suppress binary sum noise without rounding a small positive load to zero or
losing meaningful fractional precision. `—/32` means unavailable load with known
capacity, `—/—` means capacity also unknown. Last-known stale values get an explicit `~`
and stale detail; never show stale/unknown as idle zero. Formatting is presentation
only, and overload state comes from unformatted policy facts.

Keep the local capsule visible with fleet mode disabled, no remotes, empty agent list,
loading or federation failure. The existing `_update_agents_header` fleet-visibility
condition and fixed one-line CSS must change. Reuse `_app_layout.py`,
`widgets/agent_info_panel.py`, `models/fleet_agents.py`,
`actions/agents/_fleet_header.py`, `_fleet_refresh.py`, and `styles.tcss`.

Publish a versioned capacity object per machine through the existing Rust fleet worker
summary path: machine identity, occupied load, effective/persistent capacity, source,
timestamp, change token, freshness, diagnostics and optional compact holder totals. The
remote owner computes this from the same authoritative base+tool snapshot as admission.
Never sum remote visible rows or follow snapshots to infer its load. Handle older peers
through capability negotiation and absent-capacity unknown state; do not let new
optional data make otherwise usable agent summaries unreadable. Missing remote capacity
cannot be filled from a local overlay guess presented as authoritative.

Snapshot building, serialization, network and disk work run off both event loop and
Textual's serial pump. Reuse `spawn_pump_free_task`, coalescing, generation checks,
change tokens, navigation gates and cached first paint. Lease acquisition, parking,
release/recovery and cap changes invalidate local capacity; remote publication follows
the existing bounded refresh cadence and timeouts. No per-keystroke/per-cell RPC, scans,
stat/glob or new polling loop. Repaint only changed capsules. Filtering, hiding,
grouping, project/tribe changes and following remotes cannot change a machine's load.

## Phase deliverables and acceptance

Each phase description above is its scope; the shared contracts in this body are
binding. Additional phase-specific completion requirements:

| Phase            | Required evidence before completing                                                                                                                                                                                                                                                                                                                                  |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| baseline         | Current prerequisite revisions/status, external dependency edges, immutable accounting/selection fixtures and current accepted visual baseline; distinguish historical notes from reproduced failures. No fabricated benchmarks.                                                                                                                                     |
| contracts        | Rust and binding parity for B+D versus monitor D, fractional/overflow handling, invalid present metadata, serial/parallel owner conservation, monotonic generation transitions, legacy wire compatibility and exact snapshot semantics.                                                                                                                              |
| leases           | Real subprocess tests for spawn failure, success/nonzero/child-75, signals, descendant survival, supervisor loss, restart/reboot, PID reuse, nested verified reuse, forged/stale grants and rejected sibling tools. No leaked or early-released claims.                                                                                                              |
| fair-queue       | Concurrent multi-process admission cannot oversubscribe; 32 base-one callers can park without a full-capacity deadlock; a 120-second aged heavy request blocks later agents and eventually runs; cancellation, impossible demand, deference, queue timeout and cap reduction preserve accounting. Use fake clocks for policy tests.                                  |
| monitors         | Real parent-to-monitor-to-provider transitions with B=1 and nondefault B; full-check monitor is exactly 15 on athena, ordinary monitor 1, explicit decimal monitor works; no provider baseline during waiting/monitor execution; failed ACK rolls back; one command/result/delivery across retries and stop races, including sase-zl prepared completion.            |
| cli              | Parser/help/completion contracts, mandatory delimiter, quoted argv/cwd, unknown profiles, decimal/invalid values, queue/force exclusion, impossible-request diagnostics, dry-run no side effects, stdout preservation and typed refusal versus child failure.                                                                                                        |
| initialization   | Missing/invalid/bool/fractional capacity, already-complete identity, repeated init, selected-overlay provenance, non-TTY, cancelled prompt, override, YAML preservation, doctor/check parity and chezmoi write/deploy fixtures. No live capacity increase yet.                                                                                                       |
| profiles         | Deterministic prices for 32/8/4/1 and custom B; build miss versus cached wheel; all check branches including missing timing evidence; stale queued fingerprint and safe stage resume; actual selected-set/worker facts recorded.                                                                                                                                     |
| pytest-admission | Managed direct human/agent/CI pytest all share ledger; optional middle-gear refusal preserves scoped selection serially; correctness broadening remains full; worker floor no longer forces four workers onto the Mac; fake/stale inherited grants cannot bypass admission; isolated CI keeps its governed path.                                                     |
| recipe-admission | None of the seven recipes or supported bootstrap entry points starts expensive setup before admission; nested grants do not double-charge; broken workspace venv uses stable launcher; build/test worker options cannot exceed grants; quiet wrappers preserve diagnostics; version barrier rejects stale managed launchers/workspaces.                              |
| fleet-snapshots  | Local and remote snapshot fixtures include tools and unwrapped monitors once; show all enabled machines including idle/unfollowed; old peer, stale/partial/offline, identity dedup and cap-source handling; same snapshot agrees across admission, CLI and TUI.                                                                                                      |
| meters           | Deliberately reviewed PNGs for 70/80/120 columns, dark/light, ASCII, fractions, tiny positives, exact-full, overload, zero, stale/unknown, long names, standalone and three-machine fleet. Remove the old left prefix across all affected accepted goldens; do not bulk-accept unrelated diffs. Keyboard/pointer details and attention information remain reachable. |
| guidance         | Four swarm launch units expand at default weight one with omitted/explicit runners and priority combinations (no empty `%q()` or leading comma); lander omits weight two; custom user weights still parse. Memory and monitor examples are executable and match CLI. Plugin wheel install verifies actual floors without source overrides.                           |
| acceptance       | Combined Linux/macOS lifecycle, fakey concurrency, packaging, cold/warm bootstrap, version cutover/rollback and stale-peer matrix; cost observations and UI performance comparison; source configuration 32/8/4 and staged deployment verified per machine; required full verification and post-landing publication completed.                                       |

For meters, measure existing navigation benchmarks before/after: target p95 key-to-paint
below 16 ms, with no meaningful regression under representative fleet sizes. Quiet
auto-refresh reloads no unchanged surfaces and opens near-zero axe files. Count RPCs and
snapshot recomputations to demonstrate bounded/coalesced work; render-path tests must
fail if filesystem or network access is attempted. Do not interpret a host-load noisy
benchmark as proof without controlling the comparison.

For any tracked SASE changes, read `lint_and_test.md` through `/sase_memory_read` and
run required `just check`; while developing, use the working preactivation verification
path until the new wrapper is complete. At combined landing, run the real
`sase tool run -- just check-full` through `/sase_monitor` with TESTING/TESTED, then the
dedicated affected visual suite. It is possible for wrapper changes to trigger the
broadening set; follow the memory's full-verification rule. Core verification is its
repository-wide `just check` including PyO3 with Python >=3.12, never only
`cargo test -p sase_core`. Research plugin needs `just check` and real
`just test-wheel`. Use actual minimum-version wheel smoke and required binding checks,
not a test whose PYTHONPATH hides incompatible published wheels. Do not manually change
release-plz crate versions or commit through raw git.

In guidance, edit only the user-authorized canonical `sase/memory/lint_and_test.md` and
authoritative `src/sase/xprompts/skills/sase_monitor.md`, plus ordinary affected
docs/help/tests. The memory's seven command examples become `sase tool run -- just ...`;
explain conditional queueing, dynamic check prices and force without retaining the
absolute "never queues" promise. Teach the monitor skill command-after-`--` syntax, tool
wrappers, the independent default-one monitor weight, explicit `--weight` with a reason,
and why it must not add a second monitor weight to a wrapped tool. Preserve sase-zl's
landed continuation guidance. Use `/sase_memory_write`, then `sase memory init`; never
hand-edit generated instruction shims or deployed skill copies. Preview skills with
`sase skill init --diff`/`--dry-run`. Deploy with `sase skill init --force` and chezmoi
only from the clean canonical landed revision through host completion.

Remove `%q(w=2.0)` from the lander source in `src/sase/default_config.yml`. Remove
`w=0.25` from all four segments in the research plugin's
`src/sase_research_artifacts/xprompts/research_swarm.md`, conditionally omitting an
empty queue directive while preserving optional runners/priority. Do not remove general
decimal-weight support or hand-edit historical artifacts/accepted decision records.

The acceptance worker produces an indexed report containing actual commands, revisions,
profile versions, table values, lifecycle/queue outcomes, PNG review, performance and
calibration evidence, rollout protocol checks and per-machine deployment state.
Offline/unavailable apollo or Mac activation remains an explicit outstanding
prerequisite; never claim a remotely verified rollout from a local config edit. Use the
approved host deployment mechanisms and tailnet memory if remote access is needed. No
implementation phase raises capacity or ships changed swarm defaults independently of
the complete admission system.
