---
tier: epic
title: Supervisor ownership for every ACE proc
goal:
  Replace every ACE-owned durable callable proc with an argv-based supervisor-owned
  operation, make cross-instance exclusion authoritative in the shared proc store, turn
  ACE into a read-only proc observer, and retire detached as a user-selectable mode.
phases:
  - id: durable-operation-contracts
    title: Durable operation and result contracts
    depends_on: []
    size: large
    description:
      "durable-operation-contracts: inventory and classify every current
      _submit_tracked_proc and _submit_proc call site, define versioned mode-0600
      request sidecars and typed result envelopes, add focused domain-command entry
      points for durable patch, agent, bead, notification, plugin, workspace, and run
      operations, and extend the proc submission adapter so callers provide argv, stable
      fingerprints, namespaced concurrency keys, and result paths without accepting or
      serializing Python callables."
  - id: migrate-patch-and-agent-producers
    title: Migrate patch and agent proc producers
    depends_on:
      - durable-operation-contracts
    size: large
    description:
      "migrate-patch-and-agent-producers: move ACE patch/status/rebase/sync/rewind/mail
      and agent launch, approve, revert, cleanup, wait, rename, and tribe-assignment
      operations onto the durable domain commands; preserve optimistic UI behavior,
      reconstruct completion from result envelopes, and translate patch identity, dedup,
      exclusive scopes, workspace claims, AXE slots, and agent metadata locks into
      atomically reserved namespaced concurrency keys."
  - id: migrate-bead-plugin-and-utility-producers
    title: Migrate remaining durable ACE producers
    depends_on:
      - durable-operation-contracts
    size: large
    description:
      "migrate-bead-plugin-and-utility-producers: migrate bead and artifact mutations,
      notification actions, monitor control, AXE background commands, and plugin/update
      operations to explicit domain argv plus durable request/result files; classify
      prompt stashing and other short UI-only work as ordinary threaded Textual workers
      with no proc row, and remove all remaining duck-typed callable submission paths
      while keeping sensitive payloads out of argv and logs."
  - id: readonly-ace-proc-observer
    title: Read-only ACE proc observation
    depends_on:
      - migrate-patch-and-agent-producers
      - migrate-bead-plugin-and-utility-producers
    size: large
    description:
      "readonly-ace-proc-observer: replace ProcQueue ownership and ProcMirror writes
      with an off-event-loop, read-only observer of supervisor-owned proc ids, results,
      active counts, and logs; marshal selective/coalesced updates through the UI
      thread, restore authoritative state after ACE restart, preserve Procs-pane and
      optimistic completion behavior, and prove quitting or killing ACE cannot affect
      active commands."
  - id: detach-retirement-and-enforcement
    title: Detached-option retirement and invariants
    depends_on:
      - readonly-ace-proc-observer
    size: large
    description:
      "detach-retirement-and-enforcement: remove public -d/--detached from proc run,
      proc list, and the legacy task alias because every new proc is supervisor-owned;
      keep --session only for attribution, always include unattributed procs alongside
      an explicit session filter, retain hidden legacy-kind history filtering, add the
      one-release suppressed obsolete-token diagnostic, update help/completions/docs,
      and enforce statically and in tests that proc APIs accept only durable argv and no
      active proc is owned by the ACE pid."
proposed_by: bbugyi200.athena.sase-m9.3
parent_bead: sase-m9.3
bead_id: sase-m9.3.1
create_time: 2026-09-09 19:49:38
status: wip
---

- **PROMPT:**
  [prompts/202608/ace_proc_ownership.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/ace_proc_ownership.md)
- **PARENT:** [202608/supervised_proc_shells.md](supervised_proc_shells.md)
- **BEAD:**
  [sase-m9.3.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m9/sase-m9.3.1.md)

# Plan: Supervisor ownership for every ACE proc

## Outcome

Finish the proc-shell migration by making the unified supervisor service the only owner
of durable work launched from ACE. ACE submits explicit commands, observes durable rows
and typed results, and applies presentation updates on its UI thread; it never executes
or mirrors a durable operation itself. Every new proc has the same detached lifetime, so
session becomes attribution only and `detached` survives solely as immutable legacy
history.

The current tree has 30 direct submissions through `_submit_tracked_proc` and
`_submit_proc` (including their adapter call): patch/status/rebase/sync and rewind
actions; agent launch, approval, cleanup, revert, wait, rename, and tribe assignment;
bead/artifact and notification actions; monitor control; AXE background commands; and
plugin/update workflows. Recount and classify the then-current tree at implementation
time. This inventory is a lower bound, not permission to overlook duck-typed callers.

## Invariants preserved by every phase

- A durable proc submission is explicit non-empty argv executed from its persisted
  request by the detached supervisor. No proc API accepts a Python callable, callback,
  import path, pickle, or generic serialized dispatcher.
- Durable input is reduced to stable identifiers. Large or sensitive values use a
  versioned mode-0600 request sidecar, and structured completion uses a typed, versioned
  mode-0600 result envelope at `result_path`. Combined stdout/stderr remains
  human-facing presentation and is never parsed as a result protocol.
- Shared/domain behavior belongs in the existing domain service or command and, when
  required by multiple frontends, in the Rust core. Do not create a `sase ace` namespace
  merely to host former TUI callables.
- Deduplication and exclusion are cross-process guarantees. Stable, namespaced
  `concurrency_keys` encode patch identity, exclusive scopes, workspace and AXE-slot
  claims, and agent metadata locks; reservation occurs atomically in the shared store.
- ACE may optimistically update its view and retain ephemeral callbacks as a
  convenience, but durable proc state and result envelopes are authoritative after
  restart. ACE shutdown, crash, or PID death cannot terminate an active proc.
- Proc observation performs no blocking store I/O, parsing, or subprocess work on the
  Textual event loop or serial message pump. Background results are coalesced and
  marshalled through `call_from_thread` for selective UI updates.
- Historical `command`, `tui`, and `detached` rows stay readable and controllable under
  compatibility rules. New writers do not emit semantic execution kinds, and terminal
  state continues to mean command death plus complete durable settlement.

## Phase details

### 1. Durable operation and result contracts

Start with a machine-checkable inventory of all definitions, direct calls, duck-typed
protocols, test doubles, and worker-completion paths involving `_submit_tracked_proc`,
`_submit_proc`, `ProcQueue`, and `ProcMirror`. Classify each operation as durable and
independently inspectable or short-lived UI-only work. Record its owning domain command,
minimal identifiers, result type, concurrency keys, optimistic UI effects, and recovery
behavior. The final inventory must account for the then-current count rather than
assuming the 30 calls observed while this plan was authored.

Define a small shared envelope header (schema version, operation, proc id, success,
message/error) and operation-specific typed payloads. Extend the proc request/service
adapter only as needed to create mode-0600 request and result paths atomically before
launch, validate ownership and schema, and let supervisor settlement publish results
before terminal state. A missing, malformed, mismatched, or partial result must produce
an explicit durable error and must never be inferred from stdout.

Expose focused noninteractive entry points in existing namespaces such as `patch`,
`agent`, `bead`, `notify`, `plugin`, `workspace`, and `run`, reusing domain services
rather than copying TUI handlers. Required identifiers are positionals; optional policy
uses sorted short/long options under the CLI rules. Add round-trip, permissions,
invalid-schema, partial-write, crash-boundary, and result-before-terminal tests.

### 2. Migrate patch and agent proc producers

Convert the high-volume patch family first: status transitions, submit/archive/restore,
accept/rebase, sync, rewind, reword/tag/mail, and related persistence. Then convert
agent launch/approval, cleanup/dismiss/kill, revert previews and execution, wait
directives, rename, and tribe assignment. Each ACE action submits the matching durable
argv with an operation-specific request/result path, stable fingerprint, project
attribution, and all required concurrency keys.

Map the old in-memory patch key, `dedup_key`, and `exclusive_scopes` into explicit
namespaces whose overlap preserves current exclusion while extending it across ACE
instances. Transfer workspace claims and AXE-slot ownership through the existing proc
settlement policy, and use agent/family identity keys for metadata mutations. Preserve
optimistic row changes and notifications, but have completion read the typed envelope
and revalidate current selection/state before applying UI effects. Tests must exercise
two-process collisions, idempotent replay, ACE restart between submit and completion,
claim/slot release exactly once, stale callback suppression, and failure rollback.

### 3. Migrate remaining durable ACE producers

Move bead and artifact open/mutate/close flows, notification gate actions, monitor stop,
AXE background commands, plugin install/update/uninstall/mode switching, and every other
remaining durable producer to explicit domain commands. Keep payload-bearing requests
private and avoid leaking prompts, tokens, or large configuration data into argv,
fingerprints, proc labels, or combined logs.

Short local work whose value is only immediate UI state—prompt stashing is the canonical
example—uses a normal thread-backed Textual worker and creates no durable proc row. It
must still obey the event-loop and message-pump rules. Delete callable parameters and
duck-typed fallback hooks only after all production callers and representative tests use
command requests. Add a static source test that finds callable proc APIs or unclassified
producers, with an explicit narrow allowlist only for legacy read/control code that
cannot execute new work.

### 4. Read-only ACE proc observation

Replace the in-memory execution queue and write-owning `ProcMirror` with an observer
that reads proc rows, stored logs, result envelopes, and active counts off the event
loop. Track durable proc ids returned at submission, and reconcile unseen work by
session, project, shell name, and tags so restarting ACE reconstructs authoritative
running and terminal state without callback memory. Coalesce polling/watch notifications
and send small immutable snapshots to the UI thread for indicator, Procs-pane,
notification, refresh, and optimistic-state reconciliation.

Retain legacy TUI-row display and control compatibility, but never append or terminalize
a row from ACE for new work. Remove ACE ownership fields, worker-error terminal writes,
and quit-time coupling once the observer covers all surfaces. Verify with store-I/O
thread assertions, pump-stall tests, selective refresh tests, restart reconstruction,
external-proc discovery, log growth, malformed result handling, and end-to-end tests
that close ACE while a command runs and observe successful settlement from a new ACE.
Run the ACE performance benches and visual snapshots if any rendered output changes.

### 5. Detached-option retirement and invariants

All new procs are already detached from their starter, so make attribution the only
scope control. Remove `-d/--detached` from public `sase proc run`, `sase proc list`, and
the `sase task` alias; `--session none` creates an unattributed/global proc. An explicit
session filter still includes unattributed procs. Preserve legacy-kind filtering only as
a suppressed compatibility surface for historical rows and stop selecting submit APIs or
presentation behavior by semantic kind.

For one release, preprocess the obsolete token outside public argparse help so any use
fails with exactly:
`all procs are detached; remove --detached (use --session none for no attribution).`
Update examples, completion snapshots, JSON scope language, empty states, docs, and
tests together. Add final static invariants that no proc submission accepts or invokes a
callable, no new writer emits `tui` or `detached` as execution semantics, and no active
new proc records the ACE pid as owner. Preserve fixtures and compatibility tests proving
historical rows remain readable and controllable.

## Verification and landing

Every phase runs focused unit, process, CLI, and ACE tests while developing. Any phase
that changes repository files runs `just install` before `just check`. Before the child
epic lands, run `just install`, then `just check-full` only through `/sase_monitor` with
an explicit next action. Run `just test-visual` and inspect PNG artifacts for any ACE
rendering change; otherwise record why the visual suite was not required.

Acceptance requires every inventoried durable ACE producer to execute under a detached
supervisor using persisted argv; cross-instance concurrency collisions execute once; ACE
restart reconstructs state and typed results; ACE quit cannot kill active work; proc
observation stays off the event loop and message pump; public help contains no detached
option; explicit session filters retain unattributed visibility; and legacy proc history
remains intact.
