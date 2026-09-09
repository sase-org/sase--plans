---
tier: epic
status: done
title: Close the actionable SASE task backlog
goal:
  Resolve, verify, and close every currently ready SASE task bead without racing owned
  work, violating deferrals, or retiring live feature flags
phases:
  - id: core_storage
    title: Repair core storage, wire, and concurrent bead identity defects
    depends_on: []
    size: large
    description:
      "core_storage: resolve Rust-core retention and wire gaps plus duplicate-ID
      reporting and workspace-claim parsing, then prove all four task contracts end to
      end."
  - id: completion_architecture
    title: Unify completion architecture and fast repository catalogs
    depends_on: []
    size: large
    description:
      "completion_architecture: move shared model filtering to core, remove eager facade
      imports, expose the full repository inventory, and add live snippet-trigger
      completion without violating fast-path budgets."
  - id: policy_publication
    title: Reconcile memory, plan publication, and flag policy contracts
    depends_on: []
    size: large
    description:
      "policy_publication: make generated-memory checks deterministic, choose one
      approval-time plan publisher, fix project-scoped flag inheritance and flag-triage
      evidence, and separate task-type refusal copy."
  - id: successor_publication
    title: Make research publication and family handoffs collision-safe
    depends_on:
      - policy_publication
    size: large
    description:
      "successor_publication: eliminate research-output and artifact-directory
      collisions, drain stalled publication requests, reuse the successor engine, and
      stabilize pipe-family allocation."
  - id: shell_distribution
    title: Finish shell completion measurement, inline references, and deployment
    depends_on:
      - completion_architecture
    size: large
    description:
      "shell_distribution: stabilize completion latency tests and live zsh verification,
      add embedded prompt-reference completion, measure fish, and deploy generated
      scripts through managed hosts and linked CI."
  - id: guardrail_cleanup
    title:
      Repair documentation, build guards, deleted imports, and plugin-isolated tests
    depends_on: []
    size: medium
    description:
      "guardrail_cleanup: close the confirmed documentation gaps and make build, scoped
      collection, and doctor tests independent of stale recipe lines, deleted modules,
      and ambient plugins."
  - id: ace_navigation
    title: Restore ACE actions, navigation, bulk launch, replay, and kill-edit behavior
    depends_on: []
    size: medium
    description:
      "ace_navigation: complete user-facing notification, palette, launch fan-out,
      replay, and kill-edit workflows with focused interaction coverage."
  - id: ace_layout
    title: Finish responsive ACE layout and selected-detail coverage
    depends_on:
      - ace_navigation
    size: medium
    description:
      "ace_layout: correct Procs hints, compact tab-strip reflow, split-pane
      selected-detail coverage, prompt-hint overflow, and Patch query-bar placement with
      scoped visuals."
  - id: snippet_async
    title: Eliminate snippet-name modal settle flakes
    depends_on: []
    size: medium
    description:
      "snippet_async: replace fixed-delay verdict assertions with semantic settling
      across the four tracked snippet-name modal nodes."
  - id: ace_async
    title: Stabilize remaining ACE lifecycle and interaction flakes
    depends_on:
      - ace_navigation
      - ace_layout
    size: large
    description:
      "ace_async: root-cause and fix the tracked Models, Vim, plugin-modal,
      startup-worker, cache-load, mount, undo, Commit-pane, update-pane, and Jump-All
      failures."
  - id: process_concurrency
    title: Stabilize monitor, process, runner-slot, suite-gate, and runner isolation
    depends_on: []
    size: large
    description:
      "process_concurrency: repair output draining, process-group tests, teardown races,
      fork seams, pid publication, heartbeat clocks, cache poisoning, and ambient
      workspace guards."
  - id: test_isolation
    title: Make cache, snapshot, and fixed-flake accounting deterministic
    depends_on: []
    size: medium
    description:
      "test_isolation: replace process-wide read counting, isolate the agent-page cache,
      exclude dirty structural snapshot drift, and retire the committed Projects
      navigation fix correctly."
  - id: visual_closeout
    title: Fix visual convergence and clear the standing PNG backlog
    depends_on:
      - ace_navigation
      - ace_layout
      - snippet_async
      - ace_async
      - test_isolation
    size: large
    description:
      "visual_closeout: fix every mechanism-owned visual failure first, inspect each
      remaining diff, and update only intentional goldens until the serial visual lane
      is green."
proposed_by: bbugyi200.athena.08u
bead_id: sase-rm
create_time: 2026-09-09 19:51:43
---

- **PROMPT:**
  [prompts/202608/task_backlog_closeout.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/task_backlog_closeout.md)
- **BEAD:**
  [sase-rm](https://github.com/sase-org/sase--beads/blob/main/pages/sase-rm/README.md)

# Close the actionable SASE task backlog

## Outcome and scope

This epic implements and closes the 71 task beads that were `ready` in the live
2026-08-20 audit. It deliberately does not commandeer task beads already owned by other
workers, override owner snoozes, or retire feature flags before their removal contracts.
The bead store is concurrent: every phase and the land agent must re-query current state
and treat the IDs below as an audited assignment snapshot, not authority to race a later
owner.

The prior obsolete-bead sweep is valid. The following nine beads remain closed and must
not be reopened absent contrary post-close evidence:

- `sase-rg` — superseded by byte-for-byte-equivalent canonical task `sase-rf`.
- `sase-od` — commit `3078146fe` corrected all four Admin Center tab-number statements.
- `sase-pj` — `eed1d9a67` added import-verifying required-plugin repair; both required
  plugins import and the doctor check passes.
- `sase-q1` — the two exact monitor PNG nodes pass after `91c432385` and `5df623a97`.
- `sase-q9` — `8437cfd9c` removed the broken `project_accent_map` dependency.
- `sase-r9` — `ed3416c38` aligned the parallel cheap-header assertion with the intended
  no-`Step` contract.
- `sase-rh` — `4f87eb4b2` removed both stale symvision waivers and the gate passes.
- `sase-q6` — `2b2c5edef` intentionally removed the obsolete default-field contract and
  its test.
- `sase-q4` — all six model/pool-alias repro nodes pass on a freshly installed current
  tree.

The refreshed task-only snapshot was 87 active tasks: 71 `ready`, 9 `open`, 3 `snoozed`,
and 4 `in_progress`; task types were 54 legacy untyped, 13 flake, 9 flag, 5 bug, 4
feature, and 2 CI. The whole bead store additionally had 18 active plan/phase beads, for
105 active beads overall. Those two totals answer different questions and must not be
conflated.

Current source checks confirm that the ready backlog is not obsolete: the Muse/Grok
wording and bundled-skill table still drift; all three fish latency cells are still
unmeasured; the stale `proc_queue` shim now masks imports across many test helpers and
modules, broader than the bead's original three-module title; the
`SASE_ALLOW_STALE_CORE` early return remains on its own `just` recipe line; and
`sase-qo` still has a live baseline row asking for a `fixed-at` replacement even though
`b6779c4d6` contains the product fix. In `sase-core`, `BeadUpdateFieldsWire.resolution`
still lacks the presence-preserving deserializer and the LSP still owns model-filtering
logic. The linked Telegram workflow still uses unpinned `extractions/setup-just@v2`, and
the linked chezmoi source contains no deployed SASE completion scripts. Finally,
`sase validate` and `sase init memory --check --diff` now agree with each other, but
both detect fresh generated README drift; `sase-n0` therefore remains open despite the
original checker-disagreement mechanism having been narrowed.

## Protected tasks

Do not close or assign these four task beads while their current workers own them:

- `sase-j0`, `sase-lb`, `sase-mk`, `sase-oh`.

Do not close these owner-deferred beads before the snooze/wake or corroboration contract
allows new work:

- `sase-nf`, `sase-o0`, `sase-po`.

Do not close these live flag-removal beads before their November 2026 / release 0.18.0
criteria are satisfied:

- `sase-qe`, `sase-qf`, `sase-qg`, `sase-qh`, `sase-qi`, `sase-qq`, `sase-qu`,
  `sase-rc`, `sase-rk`.

If any protected bead changes state while this epic runs, record the new state in the
landing audit. Do not silently expand this epic to own it.

## Implementation and closure protocol

Each phase must begin by reading every assigned bead's live record and history. If a
target is now `in_progress` under another assignee, snoozed, or otherwise no longer
available, do not race it; note the state on the phase bead for the residual report. If
concurrent work has already fixed a target, verify the current tree and treat that
verification as the phase's implementation contribution.

Phase workers must not create task beads. Record distinct discoveries as
`PROPOSED FOLLOW-UP:` notes on their phase bead. Use `/sase_repo` before reading or
modifying `sase-core`, `sase-telegram`, chezmoi, or any other repository outside the
primary checkout. Read each opened repository's own instructions. Shared backend and
domain behavior belongs in `sase-core`; Python should use its binding or a thin adapter.

Run `just install` before repository checks. Run focused tests while iterating, then
`just check` after primary-repository changes. Use `/sase_monitor` for `just check-full`
or any other long lane, with required start/stop statuses and a useful next prompt.
Visual work must inspect actual/expected/diff artifacts and update only named nodes;
never run the broad visual-update command. Cross-repository phases must run the native
checks of every repository they modify and leave durable, recoverable changes there.

Phase agents should record one close-ready evidence block per assigned task on their
phase bead. The land agent closes a target only after its phase change is integrated and
the combined-tree condition below passes. Every close requires:

1. an evidence-rich bead note naming the cause, relevant files/commit, and exact
   verification result;
2. a concise close reason explaining why no scope remains on that bead; and
3. the correct resolution: `done` for implemented or already-landed completion,
   `superseded` only for a named canonical replacement, or `canceled` only when current
   evidence proves the report inapplicable.

If a close condition is not met, leave the task open and append an honest progress note.
Never close a task merely because a bounded reproduction attempt stayed green.

## Phase assignments and exact close conditions

### 1. `core_storage`

Open `sase-core` with `/sase_repo`, honor its instructions, update the Rust wire/API and
bindings before Python callers, and close only after cross-language tests pass:

- `sase-kh`: close after a documented hidden-agent-row retention/archive policy runs
  safely, preserves required lookups/rebuild recovery, bounds index growth, and passes
  index migration/query tests against a representative large fixture.
- `sase-mu`: close after explicit JSON `null` has a deliberate tested meaning for
  `BeadUpdateFieldsWire.resolution`—presence-preserving clear or explicit rejection—and
  round-trip tests prevent silent no-op behavior.
- `sase-oi`: close after every bead-create surface returns the post-conflict relocated
  ID, uses it in commit/output and follow-up mutations, and a concurrent collision test
  proves neither agent can mutate the other's bead.
- `sase-qa`: close after malformed trailing fields cannot make a valid RUNNING workspace
  claim disappear, the allocator remains conservative, and Rust/parser plus allocator
  regression tests pass.

### 2. `completion_architecture`

Keep the shell-completion import and latency contracts explicit:

- `sase-m1`: close after one `sase-core` model-filter implementation serves both the
  Python/ACE binding and Rust LSP while preserving provider scoping, aliases, nested
  slash models, ordering, and version-skew behavior.
- `sase-ou`: close after importing a `sase.core.*` leaf no longer executes unrelated
  facades, import-contract tests pass, and the measured completion budget is tightened
  to the demonstrated improvement.
- `sase-ov`: close after repo-kind completion offers primary, linked, sidecar, and
  external repositories with kind descriptions without importing ACE/Rich/Textual or
  exceeding the fast-path budget.
- `sase-re`: close after `sase snippet show/delete` offer live trigger candidates from a
  completion-safe catalog projection with latency and no-heavy-import coverage.

### 3. `policy_publication`

Preserve all generated-memory and flag invariants:

- `sase-n0`: close only when the workspace-pinned validator and direct memory check
  agree and are both green on the same clean tree, project/home template precedence is
  covered hermetically, and any canonical memory edit has explicit user permission and
  is followed by mandatory `sase memory init`. Without that permission, implement all
  non-memory-file fixes and leave this task open with the exact remaining action.
- `sase-n3`: close after exactly one approval-time writer publishes the canonical plan,
  one commit/push policy remains, and repeated approval/recovery tests prove idempotent
  publication without add/add conflicts.
- `sase-o2`: close after project-scoped flags are resolved in the launched workspace or
  cross-project inherited snapshots are rejected explicitly, with global/project scope
  matrix tests.
- `sase-o3`: close after frozen call-site data is captured into the FlagTriage payload,
  rendered deterministically—including zero-call-site state—and strict byte-comparable
  preview validation still passes.
- `sase-qz`: close after task-type specs support explicit create-refusal copy,
  Rust/schema digest and Python resolution agree, fallback compatibility is tested, and
  the non-creatable `feature` message no longer contradicts itself.

### 4. `successor_publication`

Use collision tests with frozen clocks and genuine family transitions:

- `sase-m0`: close after parallel research dispatches allocate deterministic unique
  report targets or fail safely without overwrite, with workflow-level concurrency
  coverage.
- `sase-p0`: close after previously skipped research-hood publication requests are
  attempted or assigned a recorded terminal reason and no request can remain forever at
  `attempts=0`, verified by outbox/doctor lifecycle tests.
- `sase-pb`: close after two in-process handoffs in one wall-clock second receive
  distinct artifact directories and both `agent_meta.json` files survive.
- `sase-pc`: close after the feedback replan branch delegates to `continue_as_successor`
  while preserving named/unnamed suffixes, relationships, metadata order, and the
  full-feedback artifact.
- `sase-r2`: close after the exact default-pipe family/workspace node is stable under
  contention and the successor uniqueness/workspace-sharing contract is exercised
  repeatedly without hiding a real launch collision.

### 5. `shell_distribution`

Use `/sase_repo` for the Telegram and chezmoi repositories and keep local install,
managed deploy, compilation stamps, update hooks, and doctor checks consistent:

- `sase-ok`: close after `sase-telegram` CI installs a pinned/reliably authenticated
  `just`, the affected workflow checks pass on repeated runs, and the note identifies
  the linked-repository change.
- `sase-ow`: close after real fish measurements replace all three placeholder cells and
  estimate prose; record the host/version/method and add a skip-safe live fish smoke
  test when practical.
- `sase-ox`: close after zsh, bash, and fish complete embedded xprompts, directives, and
  artifact references inside `sase run` prompt text while retaining fast-path import and
  latency contracts.
- `sase-oy`: close after managed hosts receive current generated scripts through
  chezmoi, target-side zcompile/stamps are coherent, doctor and update refresh do not
  fight, and linked-repo tests pass.
- `sase-p9`: close after the real interactive-zsh registration test uses a deterministic
  or separately bounded probe and passes repeated contention runs without weakening the
  production latency contract.
- `sase-pg`: close after fast-path tests measure process CPU, IO/call counts, or a
  controlled serial wall-clock lane rather than scheduler luck, and every candidate kind
  still satisfies the product budget.

### 6. `guardrail_cleanup`

- `sase-m3`: close after `docs/getting_started.md` separates “never auto-detected” from
  alias-pool routing and matches the already-correct provider documentation.
- `sase-pf`: close after the bundled-skill table includes `sase_monitor` and
  `sase_new_task` in order with source-matching descriptions, preferably with an
  enumeration drift test.
- `sase-rb`: close after all three stale-core guards genuinely short-circuit their full
  recipe when the environment flag is set and recipe-level tests prove no fetch/build
  command runs.
- `sase-qb`: close after every stale test/helper import of deleted `proc_queue` types is
  migrated to the supported source, the conftest module shim is removed, and affected
  modules collect in scoped isolation.
- `sase-ql`: close after doctor repo tests own a deterministic artifact-provider
  registry/cache and pass identically with plugins enabled and disabled, including an
  audit of sibling direct-check tests.

### 7. `ace_navigation`

- `sase-nh`: close after usage-limit notifications expose a handled Open Launch Control
  action and sender, modal vocabulary, dispatch, and unsupported-action fallback tests
  pass.
- `sase-oa`: close after grouping-cycle palette actions appear on grouping-capable
  Artifacts and Agents panes, remain absent on unsupported panes, and keybindings stay
  unchanged.
- `sase-p6`: close after marked Patch submission fans out one durable launch per marked
  Patch with correct context and notification count, covered for two or more marks and
  partial failure.
- `sase-p7`: close after submit-time VCS selection refreshes Ctrl+Space replay while
  preserving implicit-home and non-launchable guards.
- `sase-qj`: close after kill-and-edit can reopen named prompts even when directive
  extraction fails, without weakening validation for normal launch paths, with the exact
  reproduction covered.

### 8. `ace_layout`

- `sase-of`: close after Procs hints degrade honestly at 120 and 90 columns and the
  named normal/monitor goldens show either every token or explicit truncation.
- `sase-oj`: close after the tab strip selects compact tier at width 20 under the
  supported Textual version and deterministic resize tests pass in isolation.
- `sase-os`: close after narrow/even/wide split snapshots cover selected detail (while
  retaining empty-state coverage if valuable), only intended goldens change, and the
  serial file passes.
- `sase-pi`: close after constrained prompt `g`-hint overflow is discoverable via
  pagination, scrolling, columns, or an explicit remainder indicator, with a
  bound-height interaction/visual test.
- `sase-ra`: close after the Patch filter moves to the shared pane-top query slot,
  filtering behavior stays intact, visual grammar docs agree, and only inspected scoped
  goldens are rebaselined.

### 9. `snippet_async`

Treat these as one mechanism family but preserve node-specific evidence. Replace fixed
pauses with semantic verdict waits or fix a proven underlying ordering defect:

- `sase-ke`: close after filter-order/tab-completion settles deterministically under
  repeated full-lane contention.
- `sase-og`: close after derived-only collision waits for the composed verdict and drops
  out of current flake evidence.
- `sase-r7`: close after new-trigger creation waits for `Create` rather than observing
  `Checking`, with the exact node passing repeated contention.
- `sase-rf`: close after elsewhere-collision selection waits for the other-template
  verdict, with the canonical task's exact node passing repeated contention.

For every node, update `tests/reproducible_flake_baseline.txt` through a `fixed-at`
directive when prior durable evidence requires retirement; never merely delete history.

### 10. `ace_async`

Capture the actual worker/modal/state mechanism before changing deadlines:

- `sase-n5`: close after Models bucket restore is isolated from ordering/global config
  and retains `bucket:worker` under contention.
- `sase-ni`: close after the full Vim normal-key containment family is deterministic
  under repeated full-lane contention without weakening key containment.
- `sase-oe`: close after the comprehensive update-confirm modal remains mounted on
  submit collision under load using semantic mount/message settling.
- `sase-oz`: close after `AcePage` teardown deterministically awaits or cancels every
  Textual worker and the structural-quiet node passes cross-file contention.
- `sase-pd`: close after cache-miss loading and worker-result delivery cannot fail on an
  event wait under full-lane scheduling pressure.
- `sase-pe`: close after raw frontmatter is queried only after its widget mounts and the
  exact enter/bind test passes repeated contention.
- `sase-q8`: close after Ctrl-J insertion remains one undo checkpoint under the full
  parallel lane with semantic state assertions.
- `sase-qm`: close after Commit-pane result/diff completion uses reliable lifecycle
  signals and its durable flake evidence is retired correctly.
- `sase-qr`: close after the update-confirm execute/refresh path awaits real completion,
  repeated contention is green, and selection health stays clean.
- `sase-r3`: close after Jump-All Ctrl-U scrolling waits for the actual offset/state and
  repeated full-lane contention preserves the modal.

### 11. `process_concurrency`

Use injected clocks/events and module-owned seams instead of broad wall-clock widening:

- `sase-lk`: close after monitor output draining cannot consume the whole no-hang budget
  or discard readable tail data, all three exact nodes pass under real contention, and
  the reproducible-flake set retires them correctly.
- `sase-n6`: close after child slot exemption/repeat-root capping is correct under load
  and the harness distinguishes scheduler delay from a slot-accounting deadlock.
- `sase-nc`: close after process-group timeout capture either reliably obtains the
  flushed PGID before kill or tests a deterministic seam without assuming 0.2-second
  child startup.
- `sase-nr`: close after the disappearing `pytest-current` teardown race is reproduced
  and tolerated narrowly; if bounded investigation cannot reproduce it, use a detailed
  canceled close only when current evidence justifies that resolution.
- `sase-or`: close after remaining multiprocessing/`os.fork` test sites use safe
  in-process seams or an explicitly justified process boundary and the affected suites
  retain their behavioral coverage.
- `sase-qk`: close after pid-file publication/read is atomic or waits for complete
  digits, with repeated process-group contention green.
- `sase-qp`: close after fresh-heartbeat safety is tested with an injected clock or
  explicit timestamps so host starvation cannot make a live holder look stale.
- `sase-qs`: close after research-swarm allocation caches are scoped/reset across
  temporary homes and both triggering file orders plus isolated runs pass repeatedly.
- `sase-r4`: close after linked-repo setup tests pass explicit scratch workspace data or
  a supported guard seam, no test reads ambient occupancy, and empty production paths
  are rejected if unsafe.

### 12. `test_isolation`

- `sase-o1`: close after prompt-artifact cache tests count reads through a module-owned
  seam/counter rather than `builtins.open`, while still proving repeated access does not
  reread content.
- `sase-nl`: close after agent-page URL TTL cache state is scoped or reset and the exact
  refresh node passes repeated parallel scheduling without weakening expiry semantics.
- `sase-pr`: close after dirty-tree structural snapshot failures no longer pollute
  reproducible-flake accounting while real clean-tree drift still fails its owning run;
  both snapshot nodes and selection-health regression tests must pass.
- `sase-qo`: close after the live baseline row is replaced by a `fixed-at` directive
  naming `b6779c4d6` (or its integrated successor), the cross-navigation node and
  current project-seed regression pass, and no post-fix record remains unaccounted for.

### 13. `visual_closeout`

Fix diagnosed convergence/fixture mechanisms before accepting any pixels:

- `sase-cx`: close after slow-tools footer/context loading has a deterministic semantic
  settle under full visual contention and no current post-fix recurrence remains.
- `sase-ol`: close after AXE layout snapshots wait for the populated summary/status
  block and repeated visual contention never captures the CHOPS-only transient.
- `sase-ph`: close after snapshot fixtures pin every launch-default reader/token so the
  two nodes render the fixture model independent of host config.
- `sase-pu`: close after the plan-family context snapshot waits until ARTIFACTS and
  MEMORY leave `resolving` and does not rebaseline a transient frame.
- `sase-r5`: close last, after every overlap above and `sase-os` is resolved, each
  remaining failing golden is attributed to an intentional rendering change, only
  inspected node-scoped updates are accepted, and
  `SASE_PYTEST_WORKERS=1 just test-visual` passes end to end on the combined tree.

## Landing audit and mandatory residual handoff

After integrating all phases, the land agent must refresh the task backlog and inspect
every non-closed task bead. It must close each assigned bead whose exact condition above
passes, preserve every protected or unresolved bead, and summarize changes in status or
ownership since this plan was authored. It must run the appropriate combined checks;
because this epic touches broadening paths and visual fixtures, run `just check-full`
only through `/sase_monitor` and run the serial visual lane after the mechanism fixes.

If no task bead remains open, report the nine validated prior closures, this epic's
exact closure list, and the final empty task-backlog query to the user.

If any task bead remains active—and protected flags, snoozes, or concurrent owners make
that likely—the land agent must finish its turn with `/sase_pipe` to a fresh successor.
The pipe prompt must include the improved audit, every implementation/verification and
closure outcome, and the fresh full list of remaining task beads. Instruct that
successor to:

1. produce a short per-bead report answering why it is still open, what was attempted,
   and what must happen to resolve and close it;
2. use `/sase_repo` before opening or writing the `sase--research` sidecar;
3. write a new Markdown report under its `202608/` directory;
4. use `/sase_artifact_file` to register the durable report and link it to relevant
   beads or the plan when applicable; and
5. relay the report path and key results to the user.

The land agent must not substitute its own final prose for that required pipe when any
task remains active.
