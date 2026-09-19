---
tier: epic
title: Named tools and the foreground ToolRun ledger
goal:
  Deliver project-owned named commands and a Rust-owned, machine-local ToolRun ledger
  through sase tool list/run/runs/show, preserving command behavior while recording
  stages, fingerprints, and host load, with bounded retention, adoption guidance, and
  reproducible black-box acceptance evidence.
phases:
  - id: core-ledger
    title: Establish the versioned ToolRun store and bindings
    size: medium
    depends_on: []
    description:
      "core-ledger: Implement section 1, including Rust records, event/projection
      transactions, reconciliation and retention policy, golden fixtures, PyO3 bindings,
      the initial disk owner, and a forward-only core pin update."
  - id: catalog
    title: Add the project-owned catalog and tool list command
    size: medium
    depends_on:
      - core-ledger
    description:
      "catalog: Implement section 2, including whole project-layer tool definitions,
      Rust validation, operational retention configuration, five repository tools,
      versioned list output, and parser/default-list coverage."
  - id: foreground-run
    title: Execute and inspect foreground runs reliably
    size: medium
    depends_on:
      - catalog
    description:
      "foreground-run: Implement section 3, including exact argv, two-stream output,
      signals, fail-open recording, ownership and bounded logs, lost-run recovery,
      runs/show queries, and real-process acceptance cases."
  - id: stage-timeline
    title: Record run_silent stages and render the timeline
    size: medium
    depends_on:
      - foreground-run
    description:
      "stage-timeline: Implement section 4, adding recoverable JSONL start/finish events
      to run_silent without changing Justfile recipes, parent ingestion, compact
      progress, monitor timing fields, and timeline recovery tests."
  - id: run-evidence
    title: Capture fingerprints, host samples, and recording metrics
    size: medium
    depends_on:
      - stage-timeline
    description:
      "run-evidence: Implement section 5, including complete-or-explicitly-incomplete
      pre/post repository fingerprints, bounded probes, PSI/loadavg samples,
      low-cardinality telemetry, and evidence/concurrency acceptance cases."
  - id: agent-adoption
    title: Teach the tool workflow and measure its adoption
    size: medium
    depends_on:
      - run-evidence
    description:
      "agent-adoption: Implement section 6, including docs/tool.md, the lint_and_test
      memory update, Tool Run and Tool Catalog glossary strands, monitor skill source
      guidance, compact root help, and the read-only adoption report."
  - id: acceptance
    title: Prove the combined product and publish rerunnable evidence
    size: medium
    depends_on:
      - agent-adoption
    description:
      "acceptance: Implement section 7 and the numbered Definition of Done. Run the real
      CLI harness and combined checks, dogfood check, arrange the final verify monitor
      handoff, and publish an evaluation artifact with exact commands, results, run
      identities, limitations, and ownership of unrelated failures."
proposed_by: bbugyi200.athena.0nm
create_time: 2026-09-18 22:19:14
status: wip
---

- **PROMPT:**
  [prompts/202609/tool_e1_named_tools.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/tool_e1_named_tools.md)

# E1: named tools and the foreground ToolRun ledger

## Outcome and scope

`sase tool run check` executes the repository's declared argv and records what happened.
`sase tool runs` and `sase tool show RUN` reconstruct that durable history;
`sase tool list` reports each definition's last result and observed typical duration.
The ledger is useful immediately, before any prediction or admission policy exists.

This is an epic because it crosses Rust contracts, Python execution, command/config
surfaces, stage producers, storage maintenance, and agent guidance. Seven explicit,
serial `medium` phases keep each worker's implementation bounded and avoid concurrent
edits to the executor, renderer, and core contract. No model override is requested.

Design inputs, read through `sase artifact read`:

- `research:202609/sase_tool_epic_roadmap/sase_tool_epic_roadmap.md`.
- `research:202609/sase_tool_e1_named_tools/sase_tool_e1_named_tools.md`.
- `decisions:record-before-admit`, `decisions:rust-core-required`, SASE size guidance,
  CLI rules, feature-flag policy, and verification/generation conventions.

The E1 report refines the roadmap: omit the legacy importer and speculative beta flag;
use seven serial phases and a checked-in acceptance harness. Current-tree checks govern
freshness; the report's embedded draft prompt does not select models.

Planning baseline: SASE HEAD `2ec00fe68`, core pin
`8d5341a4d5e98b3b22b52395fffd267128bbf06a`, declared wheel range `>=0.34.48,<0.35.0`.
There is no tool command or ToolRun core module. The LLM Calls rename and accepted
decision already exist; `sase-zm` is closed as superseded. Do not redo them. `sase-zw.8`
remains in progress with its listed phases closed; `sase-j0` remains the existing owner
of the known full-suite budget failures. Recheck these facts when implementing, without
changing unrelated task lifecycles.

Use `/sase_repo` to open `sase-core` from each implementing worker's checkout; read its
own `AGENTS.md` and use only the path returned. Paths below are relative to the named
repository, never a particular numbered checkout. Use host-owned finalizers for commits
and release sequencing. No implementation changes precede approval.

## Binding contracts

### Ownership, data, and compatibility

Rust owns catalog normalization, canonical hashing, record validation, state
transitions, event ingestion algebra, retention selection, and queries. Implement these
under `crates/sase_core/src/tool_run/` and expose them through `crates/sase_core_py`.
Python owns subprocesses, signals, observations, file/stream effects, CLI presentation,
and thin `require_rust_binding("tool_run_...")` adapters. No Python store or
deterministic-domain fallback; release the GIL during core work.

V1 records are `ToolDefinition`, `ToolRun`, `ToolAttempt`, `ToolRunEvent`, `ToolStage`,
and `ToolLoadSample`. Every public JSON envelope includes `schema_version: 1` and
diagnostics. Queries expose bounded results, truncation and cursor metadata, and stable
ordering. Missing observations are null plus a typed explanation. Never substitute zero
for missing duration, PSI, or fingerprint components.

Each native invocation has one run id and attempt 1. Store `source: native`,
`executor: inline`, definition digest, redacted display argv, extra-argument digest,
project/agent/workspace/bead attribution, owner, parent run id, lifecycle timestamps,
monotonic durations, exit/signal facts, evidence completeness, and log metadata. Do not
add rerun behavior or imported rows. Store the fields needed by later executors without
adding fields to monitor/proc wires. Reserve artifact kind `tool` in the core registry
with `offered_in_completion: false`; it remains unresolved, with a useful diagnostic,
and cannot be claimed by a document provider. No new public artifact projection.

### Store, recording failure, and retention

Python resolves `sase_home()/tools`; Rust receives an explicit path. Layout:

```text
tools/                         mode 0700
  runs.sqlite                  mode 0600, WAL and sidecars private
  logs/<run-id>/stdout.log      foreground retained stdout
  logs/<run-id>/stderr.log      foreground retained stderr
  logs/<run-id>/events.jsonl    stage facts, including for enclosing-owned runs
```

The event file contains metadata, not a second copy of output. Protected execution argv
may be stored only in private storage, excluded from normal query serialization and
artifacts; it is never reconstructed from display argv. Never persist an environment
dump. Retention also covers WAL companions, event files, private argv, and quarantined
stores, with deletion owned by this store alone.

SQLite tables: `meta`, `runs`, `attempts`, `events`, `stages`, `samples`. Append an
event and update its projection in the same `Immediate` transaction. Exact duplicate
event ids are idempotent; conflicting duplicates are integrity errors. Use bounded busy
handling (250 ms default, no indefinite retry). Recognized corruption can be quarantined
on the write path under exclusive coordination; never quarantine a busy, read-only, or
newer-schema store. Refuse newer schemas without modifying them. Read paths do not
create a missing database, mutate schema, or quarantine data.

Healthy-store sequence: create durable identity, commit `running`, then spawn once.
Persist wrapper identity before spawn and child pid/process-group facts afterward.
Recording failures are the explicit exception: warn once `run not recorded` (or
`recording incomplete` after a successful begin), execute the exact command once, and
preserve its result. Do not print a fabricated durable id or export one to the child. On
begin failure, clear inherited tool-run/event variables for this child so its stages
cannot accidentally attach to an enclosing run; preserve monitor/proc attribution and
output ownership. Never retry execution because a DB/log/sample write failed. Do not
convert missing Rust bindings into a Python backend; report the standard stale-wheel
error before entering this feature, with documented raw-command fallback.

Retention defaults: run/attempt/lifecycle summaries 180 days; detailed stage/sample
events and projections 60 days; output/event files 14 days after settlement. Configure
positive limits under `tool_runs:`: `summary_days`, `detail_days`, `log_days`,
`log_max_bytes` (2 GiB aggregate target), `run_log_max_bytes` (256 MiB per run), and
`event_max_bytes` (16 MiB per run). Reject unknown or inconsistent values. Retain
bounded output tails and explicit dropped-byte/truncation facts on overflow while
continuing to drain and stream the child. Bound event lines and samples per batch.
Protect unsettled runs, their stage events, and active output from age/aggregate
deletion; per-run caps still apply. Report protected bytes over the aggregate target
honestly. Lifecycle history survives detail pruning; identify pruned detail in queries.

Use indexed, bounded retention work on writes and the same core policy from the
`sase disk` owner. Rust returns authorized deletion candidates; Python performs
no-follow, root-confined deletion and reports failures and actual reclaimed bytes.
Revalidate candidates against current liveness/ownership before applying. Database row
deletion is not assumed to free equivalent filesystem bytes. Dry-run performs no writes.
Register the disk inventory and reap owner in phase 1, before any public writer; finish
output-cap integration with the executor in phase 3.

### Catalog and CLI

Read `tools:` only from the actual project layer (`sase/sase.yml`), as complete entries.
Do not obtain argv from the normal recursively merged config. Use the existing layer
provenance and core normalization boundary; diagnose and ignore `tools:` in
builtin/plugin/user/machine contributions, without splicing entries. Malformed project
YAML or catalog fields are actionable errors, not an empty catalog. `tool_runs:` is
independent operational policy using ordinary config precedence.

Definition fields are exactly: nonempty string-array `argv`; string `description`;
`stages: run_silent | none`; `inputs` (repository-relative glob patterns); `env`
(allow-listed variable names); `args: allow | deny` (default deny); `fingerprint.repos`
(configured repo identities, current repo by default); and `fingerprint.toolchain`
(mapping from probe name to nonempty string-array argv). Inputs, env names, and repo
identities are string arrays; fingerprint is an object containing only repos/toolchain.
Freeze these shapes in phase-1 Rust fixtures and phase-2 schema. Reject unknown fields,
invalid types, empty argv, invalid paths, and malformed probes before spawning. Do not
expand shell syntax or environment variables in argv. Run named tools at the project
root; ad-hoc commands use the invocation cwd. Extra arguments append verbatim only when
allowed. An explicit shell such as `sh -c` remains an explicit argv choice.

The SASE project's catalog defines:

| Name        | argv                  | Extra args | Stage producer |
| ----------- | --------------------- | ---------- | -------------- |
| check       | `[just, check]`       | deny       | run_silent     |
| check-full  | `[just, check-full]`  | deny       | run_silent     |
| install     | `[just, install]`     | deny       | none           |
| test        | `[just, test]`        | allow      | none           |
| test-visual | `[just, test-visual]` | deny       | none           |

Declare relevant config/lock/recipe inputs and bounded Python/just/core version probes.
Allow-list `SASE_PYTEST_WORKERS` and `SASE_TEST_GATE_DISABLED` as relevant observations.
Store only explicitly safe values; redact secret-like variables and arguments in
query/display fields. Treat arbitrary ad-hoc shell text and unknown argument payloads
conservatively, and test secret-bearing examples. Full retained child output is private
and read explicitly with `show -l`; do not publish it in default JSON or acceptance
artifacts.

Public command contract:

| Command                           | Controls and behavior                                                                                        |
| --------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| `sase tool`                       | Central default-list delegation, including the existing notice                                               |
| `sase tool list`                  | `-j/--json`; current project catalog, LAST and TYPICAL                                                       |
| `sase tool run TOOL [-- ARGS...]` | `-q/--quiet`, `-T/--tail-lines` (200), `-v/--verbose`; flags before `--`                                     |
| `sase tool run -- ARGV...`        | Mandatory separator for ad-hoc execution, `tool: null`                                                       |
| `sase tool runs`                  | `-a/--all`, `-A/--agent`, `-c/--cursor`, `-j/--json`, `-n/--limit` (50, max 1000), `-s/--state`, `-t/--tool` |
| `sase tool show RUN`              | Exact run id; `-j/--json` or `-l/--logs`, mutually exclusive                                                 |

All help/options/subcommands are alphabetized; required values are positional. Usage,
catalog, or missing-run errors exit 2. Query/store failures exit nonzero with
diagnostics; run returns the child's code, or 128+signal. No special code 75, `run -j`,
`-H`, `--follow`, `stop`, or inferred shell parsing. Preserve tokens after `--`,
including leading dashes, empty strings, whitespace, and metacharacters. Mutually
exclusive quiet/verbose and invalid numeric controls fail before executing anything.

LAST is the newest native result for this project and definition. TYPICAL is an observed
median with sample count: latest at most 30 normally exited native runs within 30 days,
same machine/project/definition digest, no appended args. Include ordinary successes and
failures with a visible status breakdown; exclude lost and signal-censored observations.
Show an em dash for no samples, never an ETA. Ad-hoc and different-definition runs
cannot contaminate a named tool's statistic.

### Execution, ownership, and lifecycle

Use the existing inline subprocess primitives and shared `supervision.pump_output` with
two concurrent pumps. Do not delegate foreground execution to detached `proc run -w`,
merge the two streams, or introduce a supervisor/daemon. Child stdin is inherited,
stdout/stderr are pipes; document that the child has no output TTY. Drain both pipes
throughout execution even if a display/log sink fails. Keep in-memory tail buffers
bounded. Wrapper headers, stage progress, and footer go to stderr.

Humans default to exact passthrough of stdout/stderr; direct agent execution
(`SASE_AGENT_NAME`) defaults to compact output. `-q` forces compact and `-v` forces
streaming. Compact output includes identity, one completion line per stage, result, and
last `-T` retained lines on failure with `sase tool show ID -l`. Do not duplicate
`run_silent` status lines. If a safe retained-output sink cannot be created, degrade to
passthrough with one warning so compact mode cannot silently discard evidence.

Record enclosing `SASE_MONITOR_ID` or `SASE_PROC_ID` on the ToolRun only; do not create
or modify that executor. With both present, choose the nearest identifiable owner
(monitor before proc) and retain the other id as attribution. An existing
`SASE_TOOL_RUN_ID` becomes `parent_run_id`; child runs get new ids and event paths.
Validate parent identity before linking. Nested foreground output is owned by the outer
ToolRun; enclosed output is owned by the monitor/proc. These runs create only their
stage-event file, never duplicate stdout/stderr logs.

Owner precedence resolves a conflict in the research: enclosed/nested wrappers must pass
through full output to the existing owner, even when an agent id is inherited. Explicit
`-q` with an enclosing log owner is a usage error explaining that the outer owner
controls presentation; `-v` is accepted. Thus `show -l` can delegate to the existing
retained-log reader without having lost output to an inner compact mode. Report
expired/missing/truncated owner logs honestly; do not copy or extend their retention. A
foreground `show -l` replays retained stdout to stdout and retained stderr to stderr
without claiming a total order between streams.

State machine:
`created -> running -> succeeded | failed | signaled | interrupted | lost`. Atomic begin
may record both created and running events in one transaction. A spawn error settles
failed with a typed diagnostic and exit 127 for executable not found, 126 for not
executable/other launch errors. A wrapper SIGINT produces interrupted/130; termination
by SIGTERM produces signaled/143. Forward signals once to the child's process group,
drain output, reap it, and use bounded escalation for children that ignore termination.
Do not signal unrelated processes. Distinguish actual child return/signal from the
wrapper interruption reason in evidence.

Store wrapper pid, boot identity, process start identity, and child/group identity.
Before run/list/runs/show, collect liveness facts for bounded candidate rows; Rust marks
definitively dead wrappers lost with reason `runner exited without settling`. Boot
changes and PID reuse count as dead; unavailable/permission-denied observations are
unknown, never proof of death. Reconciliation ingests surviving stage facts
idempotently, does not invent an exit time/duration, and never adopts or kills an
orphan. A killed wrapper's child may outlive it; fixture cleanup owns that child. On a
read-only DB, return a diagnostic that reconciliation could not persist instead of
claiming a durable transition. No polling loop or daemon.

### Stages, fingerprints, and samples

`tools/run_silent` remains an output producer. When `SASE_TOOL_RUN_EVENTS` is set,
append versioned `started` and `finished` records containing run/stage/event ids,
description, wall epochs, monotonic elapsed time, exit code, and output byte count. Use
a bounded record and an advisory lock around a complete append, not an assumption that
pipe atomicity applies to regular files. Bound lock acquisition and fail open. Repeated
descriptions have different stage ids. A missing finish remains incomplete.

The parent reads only newly appended bytes on its existing execution tick, validates
through core, updates progress, and commits bounded batches. Flush at settlement;
reconciliation rereads the file after a crash using event ids for deduplication.
Malformed/oversized lines and a torn final line produce diagnostics; valid records
remain recoverable. A helper never opens SQLite. Add start epoch and elapsed duration to
the existing monitor stage JSON without breaking its diagnostics. With no tool env,
`run_silent` behavior is unchanged. Do not edit Justfile recipes or their ordering.

Compute unattributed elapsed time from the union of completed observed stage intervals
within the child's execution interval, not the sum of potentially overlapping intervals.
Incomplete stage coverage is marked. Wrapper setup/finalization overhead is separate
from child duration. Live progress and final queries show the same stage identities and
values.

Observe fingerprints before and after execution. Canonical components: project identity;
tool definition and extra args digest; per-repo logical identity, HEAD, index tree,
sorted dirty/untracked relative paths with status/kind/mode/content hash; declared
inputs; explicitly observed safe environment values; bounded toolchain probes. Never
hash the physical checkout path. Reuse the repository-observation and dirty-path
fingerprint helpers behind `finalizers/prepare.py`, not the compact
`monitor/host_completion_state.py::_observation_fingerprint` tuple. Do not require an
agent artifact root for standalone execution or invoke finalization as an observer.
Observe only prepared/registered declared repositories; do not clone repos on run.

Bound observation work: no input mutation, symlink following outside the declared root,
indefinite git lock waits, or unbounded version probes. Bound each version probe to one
second and 4 KiB of retained output, all probes to two seconds, and each
repository/input observation pass to four seconds and 64 MiB of content hashing. These
are observation limits, not command timeouts; omissions produce typed incompleteness.
Describe those budgets in docs and goldens. Missing repo, non-Git cwd,
unreadable/too-large input, changing-during-read files, absent executable, and probe
timeout are visible facts. A before/after change sets `mutated_input`; incomplete
fingerprints cannot later support receipts. For stages added before evidence lands,
these fields remain explicitly unavailable rather than fabricated.

Sample monotonic elapsed time, loadavg, logical CPU count, and available Linux
CPU/memory/IO PSI at start, approximately every 10 seconds, and finish. Collect raw
values with host identity, wall timestamp, interval and availability diagnostics; never
infer pressure from duration. On unsupported platforms keep fields null.
Collection/write failures cannot alter child behavior. No ETAs, demand weights,
threshold decisions, or fleet publication. Aggregate metrics describe recording
attempts/errors and outcomes using bounded labels; run ids/argv/agent ids are not metric
labels. The detailed corpus lives exclusively in the entity store.

## 1. core-ledger

Implement the V1 Rust module, PyO3 exports and thin Python wire/store adapter, using the
telemetry store's WAL/transaction conventions without sharing its DB. Include catalog
normalization and fingerprint canonicalization contracts now, with optional evidence
slots so later phases fill them without repeatedly changing the wire. Implement begin,
append stage/sample facts, finish, reconcile, list/show/summary, retention
preview/apply, and store stats. Enforce event ownership and attempt/run foreign keys.
Invalid transitions and conflicting replay are errors.

Add core golden fixtures and tests for every terminal state, unknown evidence, version
rejection, idempotent replay, concurrent transactions, bounded busy failure, corruption
recovery, retention protection, and query side-effect boundaries. Reserve the artifact
kind and test its resolver/completion behavior. Introduce the disk inventory row/reap
step using `core/disk_footprint_inventory.py` and the existing owner protocol, not a
second reaper. Read the current `sase-zw.8` acceptance contract before touching that
seam; record a focused coordination note during implementation.

Add `tools/smoke_sase_core_rs_tool_runs` for an isolated real-binding round trip and the
`tools/smoke_sase_tool_runs` harness foundation plus pytest twin. No mock store. At this
phase the core smoke is executable; unavailable public-command cases are explicitly
phase-pending, not reported as passed. Extend the installed-binding validation
inventories and tests as appropriate without blindly changing golden binding counts.
Update `sase-core-revision.txt` forward only after host finalization provides a
reachable core commit containing all existing pinned contracts and E1.

Verify core with `just check` (includes PyO3, Python >=3.12); verify affected SASE
adapters/owners and core smoke using the intended local wheel. Record core SHA, pin and
binding version. Do not hand-edit Rust release versions.

## 2. catalog

Wire a provenance-preserving project catalog through `config/loading.py` and its layer
APIs to Rust normalization. Add strict schema entries in `sase.schema.json` and
operational defaults in `src/sase/default_config.yml`; add the five definitions to
`sase/sase.yml`. Verify list concatenation and deep merge cannot alter execution
identity; ensure runtime validation covers what JSON Schema alone does not enforce.

Add lazy `main/parser_tool.py`, register dispatch in the existing command registry, and
implement list through core queries. Use `parser_root_defaults.py` for bare delegation.
Keep `tool` out of compact root help until phase 6. Only advertise implemented verbs; do
not expose half-wired run/runs/show stubs. This coherent incremental exposure is why no
beta flag is needed.

Extend harness/pytest with valid and malformed temporary projects, non-project overlays,
missing catalogs/stores, alphabetic help, versioned JSON, LAST/TYPICAL empty values, and
the real SASE catalog's five entries. Verify the existing parser bare-default and help
goldens as well as the schema/config tests.

## 3. foreground-run

Implement the executor, two output pumps, bounded log sink, signal forwarding, private
argv/redacted projection, begin-before-spawn/fail-open distinction, parent and enclosing
ownership, and core settlement/reconciliation adapters. Complete disk retention against
real log/event files before enabling public execution. Add run, runs, show and exact-id
errors; render versioned queries and named-tool statistics.

Extend the black-box harness with an isolated temp git project and deterministic child
scripts: ok, exit 3, dual-stream flooding, literal argv, nonexistent executable,
read-only/locked/corrupt store, mid-run write failure, SIGINT, SIGTERM, SIGKILL, nested
runs, output caps, and PID-identity mismatch. Use real processes and the real DB; fault
injection must target the filesystem/process boundary, not substitute a fake store.
Child marker files and a blocked-start handshake prove the durable running row is
visible before the child executes and execution happens exactly once. Clean all fixture
process groups even if assertions fail.

Exercise monitor/proc-owned passthrough in integration fixtures without creating a new
supervision path or mutating those wire schemas. Test expired owner logs and
compact-owner conflicts explicitly. Preserve separate streams even with binary
bytes/invalid UTF-8; human rendering may decode with replacement, execution/logging must
not rewrite bytes. Tests must not rely only on monkeypatched `Popen`.

## 4. stage-timeline

Implement the bounded locked JSONL protocol in `tools/run_silent`, with optional helper
code if needed to keep script size reasonable. Keep unwrapped behavior and Justfile
recipes unchanged. Add monitor timing fields compatibly. Extend the parent to tail
events on its existing tick, ingest through core, emit compact completion lines, and
flush/recover at finish or lost reconciliation.

Use a staged fixture with repeated names, a failure stopping later stages, an
interrupted stage, nested runs, overlapping producers, and malformed/torn events. Assert
event idempotency, no cross-run attribution, interval-union accounting, monotonic
durations, and preserved unwrapped output. Extend the existing monitor diagnostic tests
and Justfile regression coverage. Compare `show -j` timelines to the same values
rendered by `show` and the compact execution footer.

## 5. run-evidence

Implement bounded repository/git/input/toolchain observations and their Rust
fingerprints; add the 10-second sampling tick, initial/final observations, and bounded
telemetry. Sampling shares the executor lifecycle and stops at settlement. Respect
private outputs and config provenance. Add metric definitions, initialization,
documentation and tests; update the current metric-count expectation intentionally.

Extend harness/pytest for clean, staged, unstaged, untracked, deleted and executable
mode changes; identical contents in two checkout paths; declared linked-repo change;
input mutation during execution; missing inputs; probe timeout; unavailable PSI;
concurrent runs and independent nested stage files. Assert explicit incompleteness for
unsupported observations. A controlled 21-second child proves initial, periodic and
final samples; platform-specific assertions require PSI only when available. No
receipt/predictor logic is added to consume these facts.

## 6. agent-adoption

Add `docs/tool.md` with copyable Try it examples, catalog provenance, run vs retained
output, quiet/verbose/owner precedence, history/retention, incomplete evidence, failure
semantics and the harness command. Add compact root help and its goldens. Documentation
must distinguish ToolRuns from the already-renamed LLM Calls.

Apply `/sase_memory_write` before these adoption changes from E1's requested scope:

- Update `sase/memory/lint_and_test.md`: teach `sase tool run check`, automatic agent
  compact output plus `-q/-T/-v`, and wrapping check-full inside verify monitors.
  Preserve the `just check` verification semantics, formatting-before-monitor rule,
  exhaustive landing check, screenshot-report inspection, and stale-tool fallback.
- Add concise glossary strands `sase/memory/glossary/tool-run.md` and
  `sase/memory/glossary/tool-catalog.md`, with authored links to existing relevant
  terms; regenerate descriptors/instruction shims with `sase memory init`.
- Update `src/sase/xprompts/skills/sase_monitor.md` as the canonical skill source:
  ordinary verification examples use `-- sase tool run check[-full]`; prepared
  completion `-f` examples remain raw `just check` / `just check-full`, because their
  core exact-command contract is outside E1. Keep monitor handoff/continuation rules.
  Preview only with `sase skill init --diff`; deployment follows landing from a clean
  canonical revision through the established owner workflow.

When `sase tool` or its required wheel is unavailable, guidance uses the original raw
command and records `sase update` as the installation remedy. Never recommend bypassing
binding validation or silently re-running a child after an uncertain result.

Add `tools/tool_adoption_report` (`-d/--days` default 7, `-j/--json`) as a read-only
report, not a legacy importer. Python discovers normalized LLM-call files and reads
records; Rust owns the reusable pairing/classification/aggregation logic. Pair within
each file using `tool_use_id` (never across files), unwrap recognized shell launch argv
conservatively without executing it, and classify direct named-wrapper vs raw catalog
commands. Unknown pipelines/multicommands remain ambiguous with counts. Use positive
duration_ms or valid event-time delta; expose unpaired, negative, over-six-hour,
truncated and ambiguous records. Heavy means >=20 seconds.

Report count and wall-time shares, especially wrapped heavy `just check` wall / all
classifiable heavy `just check` wall, with denominator and coverage caveats. Do not add
independently measured ToolRun durations to overlapping LLM durations; do not count an
outer monitor-start call as the detached child's runtime. Ledger counts and recording
errors are separate coverage signals. Fixture tests cover Codex `/usr/bin/zsh -lc`, Bash
wrappers, reused ids across files, nested commands, and zero denominators. Record the
host's actual baseline, not research numbers. The target is >=70% within seven days
after landing; no calendar wait in this epic.

## 7. acceptance

Finish `tools/smoke_sase_tool_runs` and its CI pytest twin; rerun them against the
combined tree after any integration repair. The script supports `-s/--sase PATH`
(default PATH lookup), `-l/--live`, `-k/--keep`, and `-j/--json`. It uses an isolated
`SASE_HOME`, temp git project and known fixture catalog; inspect the real CLI/store
through versioned queries. Isolate user config/plugin effects without changing the
user's HOME. Never execute the expensive SASE catalog just to test basic contracts.
Default mode runs all hermetic cases and explicitly labels live cases not run; the
pytest twin shares fixtures/helpers without calling a fake application.

Each phase adds its relevant acceptance cases, runs them, and writes
`DEMO: <exact command> -> <observed result>` in its close note. Final harness results
must have no unexplained skips or unresolved phase-pending cases. A live fixture mode
exercises stage/load/owner behavior without invoking check-full as an automatic default.
Leave production retention mutations out of acceptance: deletion tests use the isolated
store only.

Run the harness against `.venv/bin/sase` and the installed `sase`, first recording which
Python/core module each actually loads. A stale deployed wheel is an explicit owner
action, not a falsely passing test or a reason to add compatibility code. The workspace
harness and binding smoke remain mandatory. Measure paired cold runs of a trivial
wrapped child versus `sase proc list` on the same host; record samples and median
overhead, with a target of <=0.5 seconds above the baseline. Investigate an E1-caused
regression instead of repeatedly running expensive verification commands.

Perform one live `sase tool run check` after formatting; if it exceeds the inline
budget, use `/sase_monitor` with a concrete follow-up that resumes these remaining
acceptance steps. Do not run three full checks solely to populate statistics: use three
short native fixture runs for that demonstration. Review any failing check against the
unchanged baseline and fix E1 regressions.

The final exhaustive verification uses `/sase_monitor`, verify profile, wrapped
`sase tool run check-full`, and an explicit continuation. Creating the monitor ends the
current agent turn: finish other work first and do not poll or promise to keep working
in that turn. The follow-up inspects owner linkage, result, screenshot report and golden
diff, and finishes the evaluation. The harness proves enclosing-owner linkage promptly
with a bounded real monitor fixture; the long check-full start is additional production
evidence. No prepared host-completion intent may skip review of screenshot updates. Use
at most one exhaustive check for this combined tree unless a later repair invalidates
it.

Starting check-full satisfies the E1 ownership demo, not the repository's exhaustive
verification requirement. Honor the current lint_and_test landing policy: complete the
required check through its monitor follow-up, or report the check/landing as incomplete.
Do not call an unfinished verification green. Known baseline failures can be
dispositioned with exact signatures and existing ownership; they cannot waive any E1
acceptance case. Do not wait for a globally green master, a package release, or a week
of adoption data.

## Definition of Done (land agent: judge against these items only)

Use `tools/smoke_sase_tool_runs --sase .venv/bin/sase -j` for the numbered fixture
cases, and the explicit live commands where stated. The harness report maps its
individual cases to the following IDs; all fixture commands run inside its isolated
project with direct-human output unless testing agent behavior.

1. **DoD-1 — Catalog:** In SASE, `sase tool` prints the central delegation notice and
   the five named tools with LAST/TYPICAL. `sase tool list -j` is versioned. In a
   malformed fixture, list/run exit 2 naming the bad entry/field and never spawn.
2. **DoD-2 — Exact execution:**
   `sase tool run -- sh -c 'printf out; printf err >&2; exit 3'` has exactly `out` on
   stdout, `err` plus wrapper metadata on stderr, and exit 3. `sase tool show RUN -j`
   reports failed/3 and retained-log metadata. The literal-argv fixture proves spaces,
   metacharacters and leading dashes survive.
3. **DoD-3 — Signals:** Harness SIGTERM and SIGINT cases return 143/signaled and
   130/interrupted with no unintended process group surviving their cleanup.
4. **DoD-4 — Lost:** Harness kills only its wrapper with SIGKILL. The next
   `sase tool runs -j` durably reports lost and the specified reason, not a guessed
   duration/exit. Boot/PID-reuse and unknown-liveness fixtures behave distinctly.
5. **DoD-5 — Fail open:** Under an unwritable store, `sase tool run -- echo hi` prints
   hi, exits 0, warns once and claims no durable id. Lock/mid-write/log-failure fixtures
   preserve exact-once execution and exit; healthy child handshakes prove running is
   committed before spawn.
6. **DoD-6 — Timeline/evidence:** `sase tool run check` followed by `show RUN`
   faithfully lists all stages actually reached, durations, unattributed time,
   before/after fingerprints, dirty counts, toolchain and available samples. Three short
   fixture invocations populate TYPICAL with the correct count; definition/arg changes
   separate cohorts. A 21-second live fixture demonstrates ~10-second samples.
7. **DoD-7 — Agent output:** `sase tool run -q -T 5 test -- <failing-fixture-path>`
   produces compact metadata plus five retained failure lines and a show pointer.
   `show RUN -l` retrieves retained full output up to the documented cap; truncation is
   explicit. Agent default, `-v`, no-sink fallback and enclosing-owner precedence have
   real-path tests.
8. **DoD-8 — Existing owners:** Harness live monitor/proc cases show owner ids and no
   duplicate ToolRun stdout/stderr logs; nested runs have correct parent ids and
   separate events. The final verify monitor starts `sase tool run check-full` and its
   follow-up records the matching monitor-owned ToolRun. Monitor startup is the E1
   feature check; full-check disposition follows the landing rule above.
9. **DoD-9 — Disk:** `sase disk list` includes the tool-run owner; `sase disk reap`
   previews retention. Isolated apply tests prove horizons/caps, unsettled protection,
   no-follow deletion, missing-store safety, honest physical-byte accounting, and
   external-owner log protection. No real user corpus is pruned for this demo.
10. **DoD-10 — Contracts/concurrency:** `runs -j`, `show -j`, core golden fixtures, and
    concurrent short-run/stage tests agree on schema and identity. Torn/replayed events
    recover without cross-linking, livelock, or fabricated evidence.
11. **DoD-11 — Rerunnable harness:** Workspace harness and real-binding smoke exit 0;
    the pytest twin is included in CI. Run the deployed-binary harness too, recording
    either PASS or a precise stale-deployment owner action. Re-run after every landing
    repair and cite only final results as acceptance evidence.
12. **DoD-12 — Adoption:** `tools/tool_adoption_report -d 7 -j` emits counts, wall
    shares and coverage; baseline is recorded. An audited read of lint_and_test shows
    the tool guidance, fallback and unchanged prepared-completion rule;
    `sase skill init --diff` previews the correct source-generated guidance.
13. **DoD-13 — Combined verification:** SASE `just check`, core `just check` including
    bindings, relevant full-check disposition and `sase bead epic-symbols` are recorded
    for the combined tree. E1 tests pass, cold-start overhead meets the measured target,
    and no temporary epic symbol exemptions remain. Any unrelated failure has a
    demonstrated baseline and owner, not a blanket known-red waiver.

Four disqualifying outcomes, even with green unit tests:

1. On a healthy recording path, the child starts before durable running; on a failed
   recording path, the wrapper lies about a durable identity.
2. Named or ad-hoc execution inserts an inferred shell.
3. A recording/evidence failure changes the executed command, launches it twice,
   prevents an otherwise valid child from running, or changes its exit result.
4. Queries present guessed load, fingerprints, completion times or durations as facts.

## Evaluation artifact

The acceptance worker creates a redacted Markdown report and companion JSON through
`sase artifact create`; the report is evidence, not a replacement for the harness.
Schema for its JSON evidence:

```text
schema_version: 1
revision: {sase_sha, core_sha, core_pin, installed_versions, loaded_module_paths}
commands: [{argv, cwd_role, exit_code, elapsed_seconds, evidence_ref}]
dod: [{id, status, case_ids, observed, diagnostic, owner_action}]
harness: {workspace_report_ref, deployed_report_ref, pytest_result}
runs: [{run_id, fixture_or_live, owner, purpose}]
snapshots: {catalog, runs, show, disk}              # redacted query envelopes
retention: {isolated_before_bytes, isolated_after_bytes, preview, applied_result}
overhead: {baseline_samples, wrapper_samples, median_delta_seconds}
adoption: {window_days, counts, wall_shares, exclusions, coverage}
verification: {scoped, core, exhaustive_monitor_id, exhaustive_result, visual_review}
limitations: [{description, existing_owner, disposition}]
```

Statuses distinguish pass, fail, not-run, and owner-action. Include exact rerun
commands, the fixture catalog, and `--keep` recovery instructions. Do not publish
private argv, unredacted child logs, environment values or arbitrary command summaries.
Source research dates and measurements are context only; evidence records the actual
host and revision tested. Register a post-monitor final version when verification
settles.

## No calendar or release waits

Phase 1 moves the core pin forward; do not rewind another epic's contracts. Add the new
binding smoke now, verify against the local pinned build, and use the established
release ratchet to move the published wheel floor when the bindings are published. Do
not claim an old minimum wheel supports new exports, enable a failing floor smoke
against it, manually edit release versions, or release a SASE package with an
incompatible declared floor. The release workflow must run the new smoke once its floor
contains the contract. Track this explicitly in the acceptance owner actions. Landing
the implementation does not require waiting for PyPI. Installed-binary staleness and
seven-day adoption are similarly explicit post-landing owner checks.

## Out of scope — successor epics, not remediation

E2 owns handoffs, stop/follow and executor settlement coordination. E3 owns failure
triage and reruns. E4 owns receipts, reuse and cheap-stage scheduling. E5 owns ToolRun
TUI surfaces. E6 owns prediction/automatic routing and any historical importer. E7 owns
capacity admission/queues; E8 owns fleet publication and rollout. Also omit PTY mode,
user-level tool catalogs, Mac live verification, prepared-completion contract changes,
remote execution, new supervisors, provider enforcement hooks, Justfile recipe rewrites,
and resolving `tool:` artifacts. Never add a speculative feature flag; if implementation
exposes an unfinished branch contrary to this plan, restructure that phase or follow the
actual flag policy with removal before landing.

Known-red handling is specific: `sase-j0` covers the previously documented suite-cost
summary and ACE/Textual cause-budget failures, not arbitrary test failures. Compare
actual failures before assigning that owner. Phase workers record unrelated work as
`PROPOSED FOLLOW-UP:` on their own phase; the lander uses `/sase_new_task` for semantic
deduplication and proper ownership. Scope creep must not replace a failed E1 acceptance
case with a new task, nor expand E1 into any successor epic.
