---
tier: epic
title: Finish supervisor ownership for the reclassified ACE producers
goal:
  Move the thirteen durable ACE producers that were reclassified as in-process session
  workers onto supervisor-owned argv operations, restore the Procs-pane, indicator, and
  restart-gating coverage those producers lost, and make the inventory prove that a
  producer with durable side effects can never be classified as UI-only again.
phases:
  - id: durable-bead-and-notification-producers
    title: Migrate bead, issue, and notification producers
    depends_on: []
    size: large
    description:
      "durable-bead-and-notification-producers: move the bead.mutate, bead.issue,
      notify.question, notify.plan_gate, and notify.legacy_epic producers off
      _submit_session_worker and onto _submit_durable_proc with their existing domain
      commands, private request sidecars, typed result envelopes, and namespaced
      concurrency keys; delete the synchronous event-loop fallback that answers a
      question notification inline; and keep pane refresh, conflict/already-handled, and
      optimistic UI behavior driven by the typed envelope."
  - id: durable-plugin-update-and-sync-producers
    title: Migrate plugin, update, and agents-sync producers
    depends_on: []
    size: large
    description:
      "durable-plugin-update-and-sync-producers: move plugin.install_many,
      plugin.mode_switch, plugin.sase_update, plugin.dev_update, plugin.combined_update,
      plugin.agent_cli_update, plugin.comprehensive, agents.sync, and agents.cached onto
      supervisor-owned argv through their owning plugin, update, agent-cli, and agents
      command paths; preserve the restored dedup keys and sase-update/agent-cli-update/
      agents-sync exclusive scopes as cross-process concurrency keys; and reconstruct
      browser state, progress history, restart notices, and indicator revalidation from
      typed result payloads so a self-update survives ACE exiting mid-flight."
  - id: session-visibility-and-classification-invariants
    title: Restore session-work visibility and enforce classification
    depends_on:
      - durable-bead-and-notification-producers
      - durable-plugin-update-and-sync-producers
    size: large
    description:
      "session-visibility-and-classification-invariants: make ACE's restart gate, proc
      indicator, and Procs pane account for whatever work legitimately remains
      session-local instead of silently ignoring it, and add inventory and static
      conformance rules so a producer with durable side effects cannot be classified as
      ui_only or submitted through _submit_session_worker; cover ACE quit and restart
      while each remaining session worker is active."
proposed_by: bbugyi200.athena.sase-m9.3.1.land
parent_bead: sase-m9.3.1
create_time: 2026-09-09 20:00:04
status: wip
---

- **PROMPT:**
  [prompts/202608/finish_supervisor_owned_ace_producers.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/finish_supervisor_owned_ace_producers.md)
- **PARENT:**
  [202608/ace_proc_ownership.md](https://github.com/sase-org/sase--plans/blob/main/202608/ace_proc_ownership.md)

# Plan: Finish supervisor ownership for the reclassified ACE producers

## Outcome

Epic `sase-m9.3.1` set out to replace every ACE-owned durable callable proc with an
argv-based supervisor-owned operation. Its first three phases did that for the patch,
agent, notification-gate, monitor, AXE, and xprompt/config families. Its fourth phase,
`readonly-ace-proc-observer`, then had to remove `ProcQueue`, `ProcMirror`, and
`_submit_tracked_proc` while thirteen producers were still routed through the callable
adapter. Its approved tale offered two ways out — "Complete their already-designed
domain-command cutover, or classify truly session-local work as `_submit_session_worker`
… Do not leave an ACE-owned durable body behind the observer facade" — and the
implementation took the second option for all thirteen.

The result is that ACE still executes durable work in its own process, but now with no
proc row at all. This plan finishes the cutover the epic promised and repairs the
observability the reclassification removed.

## Evidence

Comparing the machine-checked producer inventory at the end of phase
`migrate-bead-plugin-and-utility-producers` with the current inventory shows thirteen
sites whose `kind` changed from `direct_submit_tracked` or `duck_submit` and whose
`classification` changed from `durable` to `ui_only`:

| site                      | source                                           |
| ------------------------- | ------------------------------------------------ |
| `bead.mutate`             | `actions/_artifacts_beads_common.py`             |
| `bead.issue`              | `actions/_artifacts_beads_issue_mutations.py`    |
| `notify.question`         | `actions/agents/_notification_question_modal.py` |
| `notify.plan_gate`        | `actions/agents/_notification_plan_gate.py`      |
| `notify.legacy_epic`      | `actions/agents/_notification_modals.py`         |
| `agents.sync`             | `actions/agents_sync.py`                         |
| `agents.cached`           | `actions/agents_sync.py`                         |
| `plugin.install_many`     | `modals/plugins_browser_install.py`              |
| `plugin.mode_switch`      | `modals/plugins_browser_mode_switch.py`          |
| `plugin.sase_update`      | `modals/plugins_browser_sase_update_procs.py`    |
| `plugin.dev_update`       | `modals/plugins_browser_sase_update_procs.py`    |
| `plugin.combined_update`  | `modals/plugins_browser_sase_update_procs.py`    |
| `plugin.agent_cli_update` | `modals/plugins_browser_agent_clis_actions.py`   |
| `plugin.comprehensive`    | `modals/plugins_browser_comprehensive_update.py` |

`_submit_sase_update_proc` is representative: it runs the `uv tool install` self-update
engine inside the Textual worker pool, so quitting ACE mid-update kills a package
mutation that used to be tracked. `answer_question_notification` is worse — when
`_submit_session_worker` is unavailable it falls back to running the response body
inline on the event loop.

The reclassification also silently dropped three behaviors that the retired `ProcQueue`
provided for exactly these producers:

- `running_background_procs` (`modals/plugins_browser_sase_update_procs.py`) now reads
  only `app._proc_projection.rows`, so the post-update restart gate no longer waits for
  any in-process work.
- `_update_proc_indicator` (`actions/proc_actions.py`) counts only
  `projection.active_count`, so the top-bar count omits session workers.
- Session workers build `ObservedProc` rows but keep them in
  `_session_completion_callbacks`; they are never registered with `ProcObserver` or
  `ProcProjection`, and `_apply_proc_observer_snapshot` replaces the projection
  wholesale, so the Procs pane cannot show or count them.

A follow-on tale (`202608/ace_session_worker_submit_kwargs.md`) already restored
`dedup_key`, `exclusive_scopes`, and `duplicate_message` for session workers and added
submit-signature conformance to the inventory test, and it recorded the remaining
visibility and gating gap as a discovered issue on `sase-m9.3.1`. That gap is this
plan's third phase.

## Invariants preserved by every phase

- Inherit every invariant from `202608/ace_proc_ownership.md`. In particular: a durable
  submission is explicit non-empty argv executed from its persisted request by the
  detached supervisor; no proc API accepts a Python callable; large or sensitive values
  travel in a versioned mode-0600 request sidecar with a typed mode-0600 result
  envelope; and combined output is never parsed as a result protocol.
- Reuse the existing domain commands and services under `bead`, `notify`, `gate`,
  `launch`, `plugin`, `agent-cli`, `update`, `agent`, and `run`. Extend an existing
  namespace where its durable result support is incomplete; do not add an ACE-only
  dispatcher.
- Cross-process exclusion is preserved, not merely re-created in memory. The dedup keys
  and exclusive scopes restored by the session-worker tale become namespaced
  `concurrency_keys` reserved atomically in the shared store.
- Optimistic UI, targeted refreshes, toasts, selection revalidation, conflict and
  already-handled semantics, and failure rollback stay covered by tests, and completion
  reads the typed envelope.
- No production path may run a durable body on the Textual event loop or the serial
  message pump, including as a fallback when a submit helper is missing.
- Work that stays session-local must have no durable side effect outside the live ACE
  session.

## Phase details

### 1. Migrate bead, issue, and notification producers

Route `bead.mutate` and `bead.issue` through the durable bead and external-issue
operations, keyed by project, bead or issue identity, and operation, with titles,
bodies, labels, refs, and notes carried in the request sidecar rather than argv. Keep
the Beads pane refresh and the notification-count refresh driven by the typed envelope.

Route `notify.question`, `notify.plan_gate`, and `notify.legacy_epic` through their
durable notification, gate, and launch command paths. Response payloads stay private.
Preserve `conflict_already_handled` behavior and the modal's post-response callback, and
delete the synchronous fallback in `answer_question_notification` rather than porting
it: with a durable submission there is no reason to execute the body inline, and the
current fallback violates the event-loop rule whenever the submit helper is absent.

Update the inventory entries for these five sites to `direct_submit_durable` /
`durable`, with their domain command, identifiers, result kind, fingerprint inputs, and
concurrency keys recorded, and extend the focused tests for argv privacy, typed
success/failure completion, and cross-instance collision.

### 2. Migrate plugin, update, and agents-sync producers

Submit the plugin install batch, uninstall/mode switch, SASE update, dev update,
combined update, agent-CLI update, and comprehensive update through their owning command
paths with stable namespaced concurrency keys derived from the scopes those flows
already declare (`sase-update`, `agent-cli-update`, `comprehensive-update`,
`agents-sync`, and the per-plugin keys). Reconstruct the browser's installed/latest
version refresh, dev-update journal handling, progress and history surfaces, and the
post-update restart notice from typed result payloads instead of from a closure over
live modal state.

Do the same for `agents.sync` and `agents.cached`, whose bodies run agents-repository
git synchronization and cached-hood integration in the ACE process today. Keep the
agents-sync indicator revalidation and the post-sync agents refresh, but drive them from
the observed durable result.

Because a self-update rewrites the running installation, cover the case where ACE exits
or is killed between submission and settlement: the update must complete under its
supervisor and a newly started ACE must reconstruct the outcome from the durable row and
result envelope.

### 3. Restore session-work visibility and enforce classification

With phases 1 and 2 landed, the surviving `_submit_session_worker` callers are the
revert previews and the ordinary UI workers. Decide and implement one coherent model for
them rather than leaving them invisible: either project session rows into the Procs pane
and the indicator through a read-only session lane that cannot write the durable store,
or state explicitly in the inventory and the ACE docs that they are not proc-visible and
make the restart gate consult the live session-worker registry directly. Either way,
`running_background_procs`, `_update_proc_indicator`, and the Procs pane must agree on
what counts as active work, and ACE must not restart out from under an active session
worker.

Extend the producer inventory and the static invariants so this regression cannot recur:
a site may be `ui_only` only when it performs no durable mutation, and the static check
should fail when a `session_worker` body reaches a domain service, the proc store, the
bead store, the plugin installer, or a VCS operation. Add coverage for ACE quit and
restart while each remaining session worker is active.

## Verification

1. Run the focused ACE producer, durable-op, proc-observer, Procs-pane, plugin browser,
   notification, bead pane, and agents-sync suites while iterating, plus
   `tests/ace/tui/test_proc_producer_inventory.py` and
   `tests/test_proc_submission_static_invariants.py`.
2. Exercise a real durable submission per family through its CLI entry point, including
   a self-update dry run, and confirm the typed result envelope is written before the
   proc reaches a terminal state.
3. Run `just install`, then `just check`. Run `just check-full` through `/sase_monitor`
   before the epic's combined tree lands.
4. Run `just test-visual` only if a rendered surface changes; the Procs pane and
   indicator work in phase 3 is expected to change rendered output, so that phase must
   run it.

## Acceptance criteria

- No production ACE producer with a durable side effect is submitted through
  `_submit_session_worker`, and the inventory classifies every remaining session worker
  as side-effect-free.
- Every migrated producer submits explicit argv with a versioned request/result
  contract, namespaced cross-process concurrency keys, and typed completion; none passes
  a Python callable and none runs its body on the event loop, including as a fallback.
- Quitting or killing ACE during a plugin install, a SASE self-update, an agents-repo
  sync, a bead mutation, or a notification response does not terminate that work, and a
  newly started ACE reconstructs its outcome from the durable row and result envelope.
- ACE's restart gate, proc indicator, and Procs pane agree on the set of active work,
  and ACE does not restart while a session worker is still running.
- The static and inventory guards fail if a durable body is reclassified as `ui_only` or
  reintroduced behind `_submit_session_worker`.
