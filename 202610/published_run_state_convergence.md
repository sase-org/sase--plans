---
tier: epic
title: Published agent run state convergence
goal: Make published agent runs converge to verified terminal state without regressing
  newer snapshots or treating live runs as finished.
parent_bead: sase-1pu.5
phases:
- id: state-policy
  title: Define shared terminal-state evidence and reconciliation rules
  size: medium
  depends_on: []
  description: 'state-policy: implement the Rust state-evidence, merge, and bounded
    selection contracts and Python bindings described in Phase state-policy. Cover
    identity, terminal precedence, conflicting evidence, and cursor progress with
    core and binding tests. Keep domain decisions in sase-core; report its host-created
    commit for the next phase''s sase-core-revision.txt update.'
- id: publication-safeguards
  title: Preserve terminal state in every existing hood publisher
  size: medium
  depends_on:
  - state-policy
  description: 'publication-safeguards: integrate the state-policy bindings into inventory
    projection and snapshot merging, update the core revision pin, and prove that
    targeted and full publication cannot replace verified terminal state with an older
    active or waiting observation. Keep state payloads, snapshot fields, digests,
    and pages consistent; preserve the publication-protection boundary.'
- id: bounded-repair
  title: Add bounded report-and-apply repair for already-published runs
  size: medium
  depends_on:
  - publication-safeguards
  description: 'bounded-repair: implement the bounded state-reconciliation service
    and sase agent sync --repair-states mode specified below. Resolve exact local
    evidence for already-published runs, update only their state and derived projections,
    re-evaluate after upstream changes, and expose a resumable report cursor. Test
    read-only preview, bounds, hidden-clone ownership, publication races, failure
    recovery, and idempotence with temporary repositories.'
- id: scheduled-convergence
  title: Schedule bounded reconciliation and document owner-local backfill
  size: medium
  depends_on:
  - bounded-repair
  description: 'scheduled-convergence: add the periodic agent_state_refresh job using
    the bounded repair service, durable fair cursors and retry backoff. Wire its entry
    point and default configuration, document the bounded owner-local backfill procedure,
    and verify active-to-completed and active-to-failed convergence through the scheduled
    path without another commit. Record evidence for the child lander to close sase-1pu.5;
    do not close ancestors or seed task sase-1cc yourself.'
proposed_by: bbugyi200.athena.sase-1pu.5
proposed_with: codex/gpt-6-astra
create_time: 2026-10-29 20:35:09
status: wip
bead_id: sase-1pu.5.1
---

- **PROMPT:** [prompts/202610/published_run_state_convergence.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/published_run_state_convergence.md)
- **PARENT:** [202610/task_backlog_epic_3.md](https://github.com/sase-org/sase--plans/blob/main/202610/task_backlog_epic_3.md)
- **BEAD:** [sase-1pu.5.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1pu/sase-1pu.5.1.md)

# Published Agent Run State Convergence

## Scope and evidence

This child epic implements only phase `published-run-state-freshness` of
`plan:202610/task_backlog_epic_3.md`, tracked by `sase-1pu.5`, for seed bug `sase-1cc`.
The parent decision `optional_scope = keep_five_seeds` is final. No memory changes,
unrelated features, historical secret-purge work, or TUI changes belong here.

The phase was in progress with no previous work notes when inspected. The reported
September measurement found 8,659 of 15,457 published runs still active or waiting. This
is historical evidence, not a new measurement of the current sidecar. Current source
still has the causal gap:

- `workflows/commit/workflow_publication.py` calls `publish_committed_agent_hood` before
  the execution finalizer writes `done.json`.
- `axe/run_agent_exec_markers.py:write_done_marker_and_update_index` refreshes the local
  artifact index and pulses but does not refresh published state.
- Explicit full sync reconciles eligible hoods; no periodic publisher does so.
  `sase_chop_sidecar_auto_sync.py` fetches primary clones and is not a publisher.
- `agents_sync/inventory_sources.py:_artifact_state` currently prefers waiting metadata
  to a done marker. The new policy must distinguish stale waiting metadata from a
  genuinely unfinished run using the exact run's evidence.
- `publication_snapshot.py:_merge_richer_run_files` preserves richer content but has no
  state precedence rule. A stale inventory can regress terminal state.
- The existing publication outbox is keyed by agent and commit revision; its fulfillment
  checks page/revision presence, not terminal-state convergence.

Use periodic, bounded reconciliation of **already-published runs**. This repairs old
rows and covers new completions even when a completion callback is missed. Keep network
publication out of marker writers. Do not enqueue fake commit revisions or change the
existing commit outbox's acknowledgment semantics.

Four medium phases separate the shared contract, existing publishers, the repair
transaction, and scheduled operation. Each has a testable deliverable; implementation is
serial because the later stages consume the preceding contracts.

## Required behavior

1. An active/waiting published run with verified local completion or failure evidence
   eventually reaches that terminal state while the scheduler and remote are healthy. A
   commit-less sibling already included in a published hood is eligible too.
2. Match project, owner, canonical global name, and stable source-run identity. Never
   infer a terminal outcome from age, lack of a PID, an absent local record, another
   owner's files, a commit-only placeholder, or a session container.
3. Read authoritative current markers or dismissed records for selected identities. The
   index locates evidence; an old cached classification alone is insufficient. Unknown,
   unreadable, ambiguous, or conflicting evidence leaves state unchanged and produces a
   bounded diagnostic. Do not invent missing finish timestamps.
4. For the same immutable run, preserve an already-published terminal outcome against
   active/waiting or older observations. Preserve verified dismissal/stopped state.
   Contradictory terminal outcomes are diagnosed and left unchanged, except for a
   verified later dismissal of that same run or filling missing fields from consistent
   evidence. A reused name with a different run identity never borrows terminal state.
5. Canonical `state.json`, hood snapshot state and file references, owner-manifest
   digests, and affected rendered pages agree after every successful update. Unrelated
   runs, owners, prompts, transcripts, commits, and relationships survive.
6. A retry reads the latest upstream snapshot and local evidence before computing a new
   payload. It never replays a stale precomputed state patch over newer data.
7. Writes use the managed hidden agents clone, its existing lock, publication
   protection, normal commits, and bounded push recovery. No raw primary-sidecar
   mutation, history rewrite, or manual outbox surgery is part of the repair.

## Phase state-policy

Open `sase-core` through `/sase_repo`, read its instructions, and add a focused shared
domain module next to `agent_publication_recovery`. Do not extend unrelated recovery
wire versions just to house the new feature. Provide versioned request/response wire
types and registered Python bindings for:

- Classifying local state evidence and resolving it against a published observation.
  Inputs must distinguish a validated terminal marker, a dismissed record, an unfinished
  record, missing evidence, and an identity conflict. Outputs include the selected
  state/timestamps, change/no-change, and a stable diagnostic reason.
- Selecting a deterministic bounded window of published runs using an opaque cursor
  bound to project and owner, ordered by `(local_hood, source_run_id)`. The cursor
  represents the last examined row, including live and unresolvable rows; it must not
  repeatedly select only the first irreparable entries. Report end-of-pass explicitly.
- Choosing the next eligible project and its checkpoint transition from supplied time,
  project identities, prior cursors, and retry outcomes. Use fair rotation and bounded
  exponential backoff; this gives the scheduled phase its shared policy without
  requiring new backend contracts during integration.

Core owns classification, merge precedence, cursor and limit validation, selection, and
scheduling transitions. Python supplies filesystem observations and writes. Keep the
public snapshot schema at v2 unless a demonstrated incompatibility requires more; the
reconciliation API and local checkpoint can have their own schemas.

Tests cover completed, failed, stopped, dismissed, live/waiting, done plus leftover
waiting metadata, missing/malformed evidence, same-name different-run identity,
conflicting terminal evidence, timestamp formats already supported by SASE, stale
nonterminal candidates, unchanged repeats, deterministic pagination, inserted/deleted
rows between pages, project fairness/backoff, owner mismatch, and invalid
limits/cursors. Include a real binding round-trip test, not only Rust unit tests. Use
the linked repo's required checks.

## Phase publication-safeguards

Once the preceding core commit lands, move `sase-core-revision.txt` past it and add a
thin typed adapter under `src/sase/core/`. Follow `docs/rust_backend.md`; never add a
Python fallback for the new domain rules.

Apply the evidence policy at the existing inventory boundary, with exact-run marker
validation, and apply the merge rule while building a hood snapshot. Relevant files are
`agents_sync/inventory_sources.py`, `inventory_models.py`, `publication_snapshot.py`,
and `v2_run_io.py`. Keep the fresh and preserved state bytes inside the current
protection scan/withhold path. Resolve the state before computing file references and
rendering so a preserved terminal snapshot cannot point at an active `state.json`
payload.

Cover both `publish_agent_hood` and `reconcile_agent_hoods`. Extend
`tests/agents_sync/test_inventory.py`, `test_publication_targeted.py`, and
`test_publication_reconciliation.py` or focused neighboring modules. Reproduce stale
waiting metadata plus a same-run done marker and stale active inventory after a terminal
publication. Verify live records stay live, different-run identities do not inherit
terminal state, foreign owners survive, and repeating unchanged input writes identical
bytes. Preserve content-protection tests.

## Phase bounded-repair

Add a service under `src/sase/agents_sync/` shared by the CLI and later scheduled job.
Its scope is a window of existing runs in the current owner's validated published hoods,
including hoods originally published through linked-repository commits.

The window defaults to at most **100 examined runs across at most 5 hoods**. A public
run limit accepts 1 through 500; the hood cap remains fixed. Count examined rows rather
than only successful repairs, and enforce the existing payload byte bounds plus a
90-second work budget. This is not a promise that a hood snapshot is constant-sized:
reject/diagnose an oversized or corrupt unit safely and allow other units to progress.
Avoid loading all transcripts or building a full project inventory to locate one window.
Use the artifact index to locate selected identities and read their current marker
files; resolve dismissed evidence through existing core-backed APIs. Tests must observe
the scoped reads, not merely inspect a configured limit.

Preview reads cached publication state and local evidence without network, sidecar
writes, audit writes, or checkpoint advancement. Apply acquires the existing agents
lock, refreshes upstream, and rebuilds the window/evidence against that snapshot.
Operate on the validated published snapshot plus authorized state changes: preserve
unselected run records and content references, update changed state files and their
digests, and regenerate affected browsing pages using the existing targeted renderer. Do
not run an ordinary full sync or drain unrelated outboxes as a side effect.

Reuse `git_sync_transaction.py`, the lock helpers, and publication recovery where their
semantics fit. Its current transaction also publishes pending prompt objects; introduce
a narrow internal entry point or explicit mode if needed so state repair stays bounded
and does not publish unrelated pending content. Share the established protection and
push recovery instead of writing a second Git protocol. On non-fast-forward recovery,
discard stale planning caches and recompute from new upstream. Failed/uncertain push
does not report the window as applied; repeating it is safe. Distinguish successful
no-change from withheld or failed publication.

Expose these alphabetically documented arguments in `parser_agent_sync.py` and route
them through `agents/cli_sync.py`:

- `-s, --repair-states`: report a bounded state window, requiring exactly one project.
- Existing `-a, --apply`: apply this repair, with help updated for both repair modes.
- `-L, --state-limit N`: examined-run limit, only valid with `--repair-states`.
- `-A, --state-after CURSOR`: resume after a report's opaque cursor, also only valid
  with `--repair-states`; an omitted cursor starts a pass.

Reject combinations with status/refresh, other repairs, and retry/drop modes. Preserve
`--json` and display project names. Report examined, changed, already-current, live,
unresolved/conflicting, withheld, and failed counts; affected identities; the examined
window; `next_cursor`; and `pass_complete`. An apply response advances the usable cursor
only after successful publication or a verified no-write window. Preview is advisory:
apply always revalidates and may report different counts.

Use temporary bare remotes and two clones to test report purity, completed/failed
repair, live runs, unresolvable historical rows, no new hood enrollment, foreign-owner
preservation, same-owner newer state, a concurrent non-fast-forward push, lock busy,
push failure then retry, withheld content, byte/batch/time limits, and a second no-op
pass. Existing fixtures in `tests/agents_sync/git_sync_fixtures.py` and
`commit_publication_fixtures.py` provide the transaction scaffolding.

## Phase scheduled-convergence

Register an `agent_state_refresh` built-in job and console entry point using the
existing job SDK. Add it on the five-minute `checks` cadence in `default_config.yml`
with a two-minute timeout; it calls the bounded service with the 90-second internal
budget. Keep the existing `sidecar_auto_sync` routine's fetch-only responsibility.

Each tick selects one enabled project with a configured agents sidecar, rotates fairly
through projects, and processes one default window for the current owner. Persist a
versioned per-project/per-owner cursor and bounded retry backoff atomically under a
local job-state lock. Keep scheduling policy in core where it defines shared behavior;
Python supplies time, project enumeration, storage, and calls to the service. On a
successful end-of-pass, reset for a future pass so newly finished runs are revisited.
Live, missing, or contradictory rows must not starve later rows. A transient project
failure backs off that project and allows other projects on subsequent ticks; it does
not discard its retry position. Crash after push but before cursor persistence safely
repeats the window. Unsupported/corrupt checkpoint state resets safely without inferring
any run outcome.

Add SDK/config/entry-point tests and a deterministic scheduled integration test: publish
a mixed live hood; then write success and failure evidence with one sibling still live;
advance scheduled windows; inspect the remote state files, snapshots, manifests, and
pages without another commit or manual full sync. Include scheduler restart, cursor
wrap, failures, and enough irreparable early rows to prove fairness. Record the
convergence cadence as bounded work per tick, not a guaranteed five-minute fleet-wide
completion time.

Read `docs_writing.md` and regenerate CLI/config documentation with `just fix`. Update
the existing feature behavior in `docs/reference/agents/hood-sync.md` and add the repair
procedure to `docs/contribute/agents-sidecar-sync.md`, within their budgets. The
procedure is:

1. On each machine that owns affected runs, preview one project with
   `sase agent sync --repair-states -p PROJECT --state-limit 100 --json`.
2. Apply the same starting window with `--apply`; inspect counts and diagnostics.
   Continue from the successful apply response's `next_cursor` via `--state-after`.
3. Stop when `pass_complete` is true, then allow scheduled passes to catch later
   completions. Stop on publication failure and retry that window after remediation.
4. Repeat owner-locally on other machines. Rows whose original evidence is unavailable
   remain explicitly unresolved. Never impersonate another owner or mark historical rows
   terminal based solely on their age.

This phase provides and tests the runbook; it does not run an unbounded fleet backfill
or dispatch work to other machines.

## Verification and landing

Each implementation phase runs focused meaningful tests and `sase tool run check` in
every repository it modifies. Read the applicable lint/test memory first, use the
monitor skill for commands needing handoff, and never run bare cargo or a global
install. No `just check-full` is required by this plan. Keep core and binding tests,
Python policy-adapter coverage, and real transaction tests as separate evidence.

The child lander verifies all four phases, the actual pinned core revision, the
scheduled convergence test, and the repair runbook. Before closing its own child epic,
check its phase symbol obligations. Then run `sase bead epic-symbols sase-1pu.5` and
resolve or correctly re-key any entries before closing **only the original assigned
phase** with `sase bead close sase-1pu.5 --note "<actual verification>"`. This is the
explicitly requested child-land continuation for that phase. Do not close `sase-1pu`,
any higher ancestor, or seed task `sase-1cc`; their land agent owns those decisions.
Record `TASK RESOLVED: sase-1cc — <actual evidence>` on `sase-1pu.5` for that lander.

Workers create no task beads. Record discoveries as
`sase bead note sase-1pu.5 'PROPOSED FOLLOW-UP: <summary — detail>'`. A failure that
reproduces identically on the clean base is recorded with its evidence and any existing
tracking bead, and does not leave this completed phase open. Completion requires actual
convergence and concurrency evidence, not merely the existence of the scheduler job or a
successful preview.
