---
tier: epic
title: 'Backlog epic 1: cmdsub safety, honest retries, durable chop launches, one
  beads.db owner, triage owners'
goal: 'The five most important unowned task beads are fixed and closed: model output
  can never run host $(cmd), host finalizer failures never re-run an agent, a scheduler
  stop never silently drops a chop''s launch batch, beads.db has one SQLite library
  owner per process, and check-run triage attaches the owning task bead. The lander
  then continues the backlog chain with the next five beads.

  '
phases:
- id: cmdsub
  title: Model output never reaches host $(cmd) (sase-1sn)
  depends_on: []
  size: large
  description: 'cmdsub: close every model-output and agent-writable channel into host
    command substitution for human-launched workflows, keeping macro expansion and
    Jinja semantics.'
- id: retry_scope
  title: Provider retry never re-runs an agent for a host finalizer failure (sase-1he)
  depends_on:
  - cmdsub
  size: medium
  description: 'retry_scope: type host finalizer and publish failures so provider
    retry patterns never match them, while real provider 500s still retry.'
- id: chop_launch
  title: A scheduler stop never silently drops a chop's launch batch (sase-1c2)
  depends_on:
  - retry_scope
  size: large
  description: 'chop_launch: journal in-flight chop launch batches and resume them
    idempotently after a scheduler or service-host stop, with visible notices.'
- id: beads_db_owner
  title: beads.db has one SQLite library owner per process (sase-1kp)
  depends_on:
  - chop_launch
  size: large
  description: 'beads_db_owner: remove the Rust lock-file and Python WAL-mirror collision
    on beads.db per the mirror_fix decision, with a same-process repro test.'
- id: triage_owners
  title: Failure triage attaches the owning task bead (sase-1ha)
  depends_on:
  - beads_db_owner
  size: large
  description: 'triage_owners: root-cause and fix empty possible_owners in check-run
    triage so exact node and location matches attach the open task bead.'
decisions:
  mirror_fix:
    ask: Fix sase-1kp by retiring Python's WAL mirror of beads.db, so the file has
      one library owner?
    choices:
      retire_mirror: Python never opens beads.db; its two mirror reads move to the
        Rust store
      dedicated_lock: Rust locks a new dedicated file; needs a dual-lock rollout across
        sase versions
    default: retire_mirror
    why: Only two Python call sites remain, and no cross-version lock protocol has
      to change
    answer: retire_mirror
proposed_by: bbugyi200.athena.1dg
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-17 11:19:24
status: wip
bead_id: sase-1tc
---

- **PROMPT:** [prompts/202611/backlog_epic_cmdsub_retries_launches.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/backlog_epic_cmdsub_retries_launches.md)
- **BEAD:** [sase-1tc](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1tc/README.md)

# Backlog Epic 1: Model Output Never Runs Host Commands, Honest Retries, Durable Chop Launches, One beads.db Owner, and Triage Owners

## Context

On 2026-11-17 the project had **443 open task beads** (master `df06821acc`). A read-only
triage checked every one of them against code, git history, the linked repos, and
memory:

- 35 were closed. 34 were already fixed at HEAD, or their test, skill, or plan premise
  was gone. Each close note cites the commit or check that proved it.
- One duplicate (sase-1s8) was folded into the older bead sase-1pd.
- 408 remain open.

This epic works the five most important beads that **no other active epic owns**. Each
was re-verified as still reproducing at `df06821acc`.

| Rank | Bead     | Phase            | Why it ranks here                                                                                                                                                                                                             |
| ---- | -------- | ---------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1    | sase-1sn | `cmdsub`         | Security. Text written by a model (or an agent-writable branch name) can run arbitrary host `$(cmd)` substitutions, outside every permission gate, when a human launches a workflow.                                          |
| 2    | sase-1he | `retry_scope`    | A host finalizer failure (git push 500) matches a generic provider retry pattern and re-runs the whole agent turn. That costs a full LLM turn and gives the agent a misleading nudge.                                         |
| 3    | sase-1c2 | `chop_launch`    | A scheduler or service-host stop during a chop's launch loop silently lost 59 of 61 clan launches. The half-launched clan then blocked its own chop.                                                                          |
| 4    | sase-1kp | `beads_db_owner` | Rust uses `beads.db` as its mutation lock file while Python keeps a WAL SQLite connection on the same file. Under POSIX lock semantics, Rust's close drops SQLite's locks, which risks mirror corruption or SIGBUS.           |
| 5    | sase-1ha | `triage_owners`  | Check-run failure triage attaches no possible owners: 0 of 500 recent failure groups have one. Known failures look NEW, so agents treat them as their own regressions and keep filing duplicates (107 flake beads were open). |

**Excluded because another active epic owns them.**

- sase-1r4, sase-1qj, sase-12u, and sase-1jd belong to epic sase-1ra (phases .7–.12).
  That epic has had no live agent since the 2026-11-07 athena reboot.
- sase-1f3 and sase-1nc belong to sase-1nf and sase-1t3.

**Deferred.** The stranded-work bugs sase-1sh, sase-1si, and sase-1sk overlap sase-1ra.8
(staged-monitor settlement), .11 (boot reconciler), and .12 (land-agent liveness). They
wait until sase-1ra lands or is canceled. Fixing them here would build a second recovery
mechanism beside the one sase-1ra already plans.

All phases run strictly in sequence, one at a time.

## Rules for every phase

- **Read and reproduce.** Read your task bead in full, including every note and +1, with
  `sase bead read <id> -r "<why>"`. Reproduce the defect before changing code wherever
  that is possible. Record the base commit, before/after evidence, and any limits on the
  phase bead.
- **Stay out of live-epic files.** Before editing, run `sase agent list` and check
  `sase bead list -t plan` for in-progress epics whose live agents touch the same files.
  If a fix needs a file a live agent owns, record a `PROPOSED FOLLOW-UP:` instead.
  Append a coordination note to the owning epic bead.
- **Keep the guards.** Keep assertions, safety refusals, and existing budgets. Do not
  add skips, xfails, flake-baseline entries, timeout or elapsed-bound increases, or
  suppressions just to make a gate pass.
- **Rust core boundary.** Shared backend behavior belongs in sase-core. Open it with
  `/sase_repo` (`sase repo open sase-core`). Python I/O, presentation, and adapters stay
  in this repo. Let host completion move `sase-core-revision.txt`, or use
  `just ratchet-core-revision`. Never invent a SHA.
- **Verification.** Read the `lint_and_test` reference memory before finishing. Also
  read `ace` before touching `src/sase/ace/`, `tui` before TUI changes, `cli_rules`
  before adding CLI options, and `docs_writing` before editing `docs/`. Run focused
  regressions, then `sase tool run check` in each changed repository. Do not run
  `just check-full`.
- **Bead hygiene.** Phase workers never create beads. Record out-of-scope findings as
  `PROPOSED FOLLOW-UP:` notes on the phase bead. Record
  `TASK RESOLVED: <task-id> — <evidence>` on the phase bead and close only the phase
  bead. The lander closes the task beads.

## Phase cmdsub: model output never reaches host `$(cmd)` (sase-1sn)

**Defect.** sase-1sb.1 (`d0dc942c16`) protects workflow model output from host-side
`$(cmd)` only under the `_raw` and `_response` keys (`_wrap_model_output_values` in
`src/sase/macro/workflow_executor_utils.py`). For a human-launched workflow
(`SASE_PROMPT_AUTHORSHIP=human`), `preprocess_prompt_late` still runs
`process_command_substitution` over every other interpolated value:

- structured `output:` fields (`workflow_executor_steps_prompt.py` stores
  `output = data`);
- `_data`;
- loop `join: text` (`workflow_executor_loops.py`);
- bash/python step outputs that echo model text or agent-writable repo state, such as
  `{{ setup.branch_name }}` in `src/sase/macros/sync.yml`.

The bead's side-effect-free repro shows each of these channels executing
`$(echo EXECUTED_X)`.

**Fix.** Make it impossible for any value that originates in model output or agent-
writable state to be command-substituted on the host, through every
`render_prompt_template` caller:

- prompt prepare, parallel, embedded, and embedded_expand;
- workflow_runner;
- mentor.

Follow the sase-1sb decision `agent_cmdsub = provenance`. The bead lists two candidate
mechanisms:

- run late preprocessing with `agent_authored=True` when the template references prior
  step output;
- perform command substitution on the authored template before interpolation.

Choose between them in this phase's plan. Honor the bead's design constraints:

- Do not wrap every step output in a disabled region. `{{ render.combined }}` in
  `with_q_and_a.yml` and `with_feedback.yml` must still expand macros.
- Jinja conditionals and filters over model output must see real values. For example,
  `{% if review.verdict == "approve" %}` must still work.
- Human-authored `$(cmd)` in a human-launched workflow's own template text keeps
  working.

**Tests.**

- One marker-file or echo-substitution regression per channel: structured field,
  `_data`, loop join text, and a step output carrying a branch name.
- A Jinja conditional over model output that still renders correctly.
- A human-authored `$(cmd)` that still runs.

**Done when** none of the four channels can execute a substitution, the existing
cmdsub-provenance tests still pass, and `sase tool run check` passes.

## Phase retry_scope: provider retry never re-runs an agent for a host finalizer failure (sase-1he)

**Defect.** `handle_workflow_error` (`src/sase/axe/run_agent_exec_retry.py`) passes the
whole step error string to `is_retryable_error` / `find_retry_config_for_error`
(`src/sase/llm_provider/retry_config.py`). Those do case-insensitive substring matching,
and the string includes stderr from host-owned finalizers such as
`sase stitch create --resume` and git push. GitHub's `remote: Internal Server Error`
matched the shipped claude pattern `"Internal server error"`
(`llm_provider.retry.claude.error_patterns` in `src/sase/default_config.yml`). The
runner then re-ran the entire agent turn and told the agent it had hit a provider
failure. Generic HTTP phrases such as "rate limit" and "Too Many Requests" can match git
or gh stderr the same way.

**Fix.**

- Host finalizer and commit-publish failures surface as a typed exception, or carry a
  typed marker, that provider retry classification never matches against.
- Provider retry matching only sees provider-invocation failure text.
- The finalizer's own durable retry (`--resume`) stays the recovery path for publish
  failures.
- Real provider "Internal server error" responses must still retry.

**Tests.**

- `handle_workflow_error` with the bead's finalizer push-500 text returns `"raise"`.
  There is no retry wait and no attempt snapshot.
- A genuine provider 500 still retries.
- A finalizer failure whose stderr contains "rate limit" is not retried.

**Done when** those regressions pass and `sase tool run check` passes.

## Phase chop_launch: a scheduler stop never silently drops a chop's launch batch (sase-1c2)

**Defect.**

- The routine's SIGTERM handling only stops the loop. The service host SIGKILLs the
  scheduler process group about 10 s later.
- Unlaunched proposals in `launch_chop_proposals`
  (`src/sase/axe/chop_proposal_launch.py` and its launch-execution loop) are neither
  persisted nor resumed.
- The real incident: `toobig_split[sase]` launched 2 of 61 `toobig-68` members, and then
  `systemctl --user restart sase.service` killed it. The half-launched clan inhibited
  the chop until someone killed it by hand.

The bead names the paths. Re-locate them at HEAD first, because the routine and signal
code has moved (`src/sase/axe/lumberjack.py`, `orchestrator.py`, and `runner_signals.py`
now own the SIGTERM handlers).

**Fix.**

- Make an in-flight launch batch crash-safe. Before the first launch, durably journal
  the remaining proposals. Mark each member as launched when it launches.
- On the next scheduler or routine start, resume an unfinished journal idempotently:
  skip members that already exist by name, and launch only the rest.
- A clan whose batch is being resumed must not inhibit its own resume.
- A SIGTERM while a batch is active finishes the current launch and then stops cleanly.
  The journal carries the rest; do not rely on a longer grace period.
- Every interrupted or resumed batch is visible: log the run and send one notification
  naming the clan and the counts.
- Keep any shared domain logic, such as journal format and resume decisions, behind the
  Rust boundary if it is something other frontends must match.

**Tests.**

- Simulate a stop after k of n launches, then restart. All n − k remaining members
  launch exactly once.
- Restarting twice launches nothing more.
- A batch journal whose clan was killed by the user is discarded, not resumed.

**Done when** the regressions pass, `sase tool run check` passes, and the phase bead
records a manual stop-and-resume demonstration on a disposable chop (not on live athena
routines).

## Phase beads_db_owner: beads.db has one library owner per process (sase-1kp)

**Defect.**

- sase-core `crates/sase_core/src/bead/mutation/store.rs` sets
  `BEAD_MUTATION_LOCK_FILENAME = "beads.db"`, and `with_bead_mutation_lock` opens,
  locks, and closes that file on every mutation.
- Python's `BeadProjectStoreMixin._conn` (`src/sase/bead/_project_store.py`) caches a
  stdlib `sqlite3` WAL connection on the same file.
- Closing any fd releases every fcntl lock the process holds on that file. A Rust
  mutation in a process that holds the mirror open therefore drops SQLite's locks on
  `beads.db`, `-wal`, and `-shm`.

This is the hazard class sase-1kk fixed for the TUI. It is CLI-only today. Python now
reads the mirror at only two sites: `_max_local_child_counter` and `_export`'s
`export_to_jsonl(self._conn, …)`.

> [!decision] mirror_fix = retire_mirror Remove every Python open of `beads.db`.
>
> - Compute the local child counter from the Rust bead store.
> - Produce the on-demand `issues.jsonl` export through the Rust store or event
>   projection.
> - Delete `_conn`, `_conn_cache`, and the mirror rebuild path once no caller remains.
>
> Older sase processes that still open the mirror are separate processes, so the
> per-process lock hazard does not span versions and no lock protocol changes. Add a
> guard test that fails if any `src/sase` module opens `beads.db` through `sqlite3`.

> [!decision] mirror_fix = dedicated_lock Move Rust's mutation lock to a new dedicated
> lock file in sase-core.
>
> - During rollout, new processes must take both the legacy `beads.db` lock and the new
>   lock, in a fixed order, so that mixed-version processes still exclude each other.
> - Document when the legacy lock can be dropped.
> - Add a test proving that a Python mirror connection keeps its SQLite locks across a
>   Rust mutation in the same process.

**Tests.** Cover both branches with:

- the bead's repro: an open mirror connection, then a Rust mutation in the same process,
  then a concurrent writer;
- existing bead CLI and store regressions.

Update the SQLite-ownership section of `docs/contribute/rust-backend.md` to match the
chosen branch.

**Done when** the repro can no longer lose locks, the bead store suites pass, and
`sase tool run check` passes in each changed repository.

## Phase triage_owners: failure triage attaches the owning task bead (sase-1ha)

**Defect.**

- Across 117 check runs (2026-10-05..07), all 897 triage items had empty
  `label.possible_owners`. That includes failures whose open task bead's `node_id` names
  exactly the failing test file.
- 29 of those runs timed out gathering owner candidates, and 5 had the bead store
  unavailable.
- `gather_owner_candidates(Path('.'))` alone returns the right candidates in about 0.75
  s. The loss is therefore in settle/matching (the sase-core matcher behind
  `tool_run_triage_settle`) or in the timeout path.
- Re-checked on 2026-11-17: 0 of 500 recent failure groups have owners.

**Fix.**

- Root-cause the matching loss end to end: candidate gathering, then the wire, then the
  core matcher, then the persisted label.
- Make exact `node_id` and test-file location matches attach the owning bead.
- Bound candidate gathering so it does not time out on a large bead store.
- Matching semantics belong in sase-core; the Python side stays an adapter.
- Do not reclassify verdicts: triage annotates and never changes exit codes (decision
  `triage-annotates-does-not-change-exit-codes`).

**Tests.**

- A fixture failure whose node is named by an open task bead gets that bead in
  `possible_owners`.
- A stage-label locator (the symvision case) either matches or is explicitly documented
  as unsupported.
- Timeout-path coverage.

**Done when** a real `sase tool run check` with a known failing node shows its owning
bead in `sase tool show -j`, `sase tool failures` reports non-empty `newest_owners` for
it, and `sase tool run check` passes.

## Landing and the next epic

The lander follows the normal land procedure. Then:

1. **Close what was fixed.** For each phase's `TASK RESOLVED:` note, verify the
   evidence. Close the task bead with `sase bead close <id> -n "<evidence>"`. If a fix
   did not fully land, leave the bead open with a note saying what remains.
2. **Triage follow-ups.** Each `PROPOSED FOLLOW-UP:` goes through `/sase_new_task`
   (duplicate check first).
3. **Continue the backlog chain.** This is mandatory: it is the owner's standing
   instruction. If open task beads remain (`sase bead list -t task -n 0`), do the
   following before finishing:
   1. **Re-triage.** Close task beads that are verifiably duplicate or obsolete.
      - The evidence must be concrete: a fixing commit after the bead's `created_at`, a
        `file:line` showing the behavior now exists, a deleted or renamed test or
        symbol, or a memory note that already says what the bead asks.
      - Flakes are never closed because a test "passes now".
      - When in doubt, keep the bead.
      - Read-only helpers may verify slices of the backlog in parallel. Only the lander
        performs the closes, each with an evidence note.
   2. **Exclude owned or deferred beads.** Exclude beads owned by another active epic:
      named in an in-progress plan or phase bead's title or description, or clearly
      inside its scope. Exclude beads the owner snoozed, and flag beads whose
      `remove_by` date or release has not been reached.
      - sase-1r4, sase-1qj, sase-12u, and sase-1jd stay with sase-1ra.
      - sase-1sh, sase-1si, and sase-1sk become eligible only after sase-1ra closes or
        is canceled.
   3. **Pick the next five.** Choose the five most important remaining beads, in this
      order:
      1. data loss, lost work, security, or stranded agents and epics;
      2. silent correctness bugs;
      3. anything that keeps CI or landing red or misleads other agents;
      4. user-visible breakage;
      5. performance;
      6. features;
      7. docs and memory.

      Boost beads with higher `+N` corroboration and those still reproducing at HEAD.
      Discount "decide/evaluate/measure later" beads and single flaky tests that block
      nothing.

   4. **Plan the next epic.** Author it with `/sase_plan` and the same rules:
      - at most 5 phases, strictly sequential, each depending on the previous one;
      - responsible for at most 5 task beads (one phase may work more than one);
      - a minimal set of plan decisions, used to let the owner opt out of feature work,
        memory changes, or work that might not be needed;
      - this whole "Landing and the next epic" section copied in, with its exclusions
        updated, so its lander continues the chain.
   5. **Propose it.** Run `sase plan propose`.

   The chain ends when no open task beads remain. If the only remaining open task beads
   are excluded under step 2, do not create an epic. Instead, record a note on this epic
   bead that lists them and why each is excluded.
