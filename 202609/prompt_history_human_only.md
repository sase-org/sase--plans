---
tier: epic
title: Prompt history records human submissions only
goal: 'Prompt history holds one row per human submission: the canonical text a person
  submitted through the TUI prompt bar, `sase run` / `sase prompt run`, or mobile/Telegram.
  It holds nothing a machine launched: swarm members, routine jobs, bead work, approvals,
  restarts, relaunched member agents, monitor and gate-command launches. The machine
  rows already in the store can be pruned safely.

  '
phases:
- id: gate
  title: Write gate and sase run ingress provenance
  depends_on: []
  size: medium
  description: 'gate: make generated-origin writes a no-op in both history writers
    (no row, no placeholder, no Stash entry); classify sase run invocations from monitors
    and gate commands as generated; record the root prompt for direct typed admission
    and remote dispatch; enforce explicit launcher origins with an AST test.'
- id: canonical-text
  title: Record each submission's canonical text once
  depends_on:
  - gate
  size: medium
  description: 'canonical-text: add an ingress-owned history_text to the launcher
    so single-slot swarms, %r:N, force-reuse and launch_units launches record the
    submitted text exactly once; teach sase run the history_text and history_origin
    payload keys.'
- id: tui-provenance
  title: TUI submissions carry their history text and origin
  depends_on:
  - canonical-text
  size: medium
  description: 'tui-provenance: keep the pre-remodel prompt on PendingLaunch and send
    it as history_text; mark member-agent relaunches and mentor-apply launches generated
    across submit, cancel and failed-launch recovery.'
- id: prune
  title: Prune machine rows from the existing store
  depends_on:
  - gate
  size: medium
  description: 'prune: extend sase-core''s looks_generated classifier and expose it
    to Python; add sase prompt prune --generated/--legacy with preview, backup and
    typed-wins protection; report origin counts in sase prompt doctor.'
proposed_by: bbugyi200.athena.0ud
create_time: 2026-09-30 07:44:44
status: wip
bead_id: sase-1d8
---

- **PROMPT:** [prompts/202609/prompt_history_human_only.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/prompt_history_human_only.md)
- **BEAD:** [sase-1d8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1d8/README.md)

# Plan: Prompt history records human submissions only

## Context

Prompt history (`~/.sase/prompt_history/YYMM.json` shards, written only through
`add_or_update_prompt` and `record_failed_launch_prompt` in
`src/sase/history/prompt_store_mutations.py`) is supposed to hold user prompts. Each row
should correspond to one request a human made. Today it is mostly machine output:

- **The first History page is mostly machine rows.** Of the 100 newest rows, 79 are
  `origin=generated`.
- **The store as a whole is mostly machine rows.** About 71% of its ~11,600 rows are
  machine-shaped, and that share is rising month over month.
- **Bead work is the biggest source in the screenshot.** The highlighted row and every
  `+2`/`+3`/`+4` burst come from `sase bead work` epic fan-out, which writes one joined
  bundle row plus one row per phase segment.
- **Routine jobs add more.** The "Can you help me split … into multiple files?" rows
  come from `toobig` split jobs.

This matches the consolidated research report
`research:202609/prompt_history_human_submissions_only/prompt_history_human_submissions_only.md`.

`PromptEntry.origin` (`typed | generated`) is already passed by every in-tree launch
site, but only the next-word prediction corpus reads it. Both writers still persist
generated rows, learn `<placeholder>` tags from them, and stash failed generated
launches. Several paths also record the wrong text for a human submission:

- the TUI provider-guard remodel records the expanded swarm members;
- a swarm that expands to a single slot records that member;
- `%r:N` records its slots;
- force-reuse records the rewritten prompt;
- relaunching a member agent records the member's prompt as `typed`.

Two paths record nothing at all: direct typed `%if`/`%proc` admission and `%dispatch`
never record the human's root prompt.

## Invariant

> Prompt history records the canonical text a human submitted through a human entry
> point, once per submission. A machine-originated launch writes no history row, no
> Stash entry, and no placeholder.

**Human entry points:**

- the TUI prompt bar, including cancelled drafts and failed submits;
- `sase run` or `sase prompt run` from a terminal;
- the mobile gateway;
- Telegram.

**Machine sources (everything else):**

- agents and LaunchApproval;
- routine/job (chop) launches;
- bead work;
- plan/epic approval;
- restart and drain;
- monitor commands and gate commands;
- relaunches of member agents;
- mentor-apply launches.

The user's two rules follow from the invariant:

- A swarm member's prompt is machine text, so it is not recorded. The
  `#research_swarm(…)` invocation is the human's text, so it is recorded once.
- Every routine launch is `generated`, so none is recorded.

**Canonical** means after project-alias and project-tag canonicalization, which keeps
dedup and the `project:` filter working. It also means before any swarm, repeat, alt,
admission, provider-guard, or force-reuse rewriting.

## Decisions

These include three corrections to the research report.

1. **Gate on provenance at write time.** `generated` means "do not write". Text
   heuristics are used only for the one-time cleanup of legacy rows (phase `prune`).
2. **Correction: chop/job variables in the process env do not mean automation.** The
   Telegram inbound handler runs as a job (`sase_job_tg_inbound` /
   `sase_chop_tg_inbound`). Its `--once` mode launches human Telegram prompts from
   inside the job tick, where `SASE_CHOP_NAME`/`SASE_JOB_NAME` are set. So chop markers
   count only when they appear in the launch's own `extra_env`/`segment_extra_env`,
   which the chop launchers set explicitly (see `build_chop_launch_env`). The research
   report's advice to check `is_chop_launch_env(os.environ)` would have silently dropped
   Telegram history.
3. **Correction: the new automation markers apply only at the `sase run` ingress.** The
   markers are `SASE_MONITOR_ID` and a new `SASE_GATE_COMMAND`. Proc and daemon
   boundaries don't scrub either variable, so a daemon restarted from inside a monitor
   or gate command would inherit it. If the check lived in the central store helper,
   such a daemon would silently drop human mobile/Telegram rows. `sase run` is the only
   human entry point that automation can reach, so the check lives there. `SASE_AGENT`
   keeps its central rule, because it is scrubbed at every proc, monitor, and chop
   boundary. Every agent sets it, and any origin written under it becomes `generated`.
4. **Correction: the launcher `origin` default stays `None`, which still records.**
   - The `origin` keyword is not in any sase release yet (latest tag `v0.17.1`).
   - sase-telegram requires `sase>=0.17.0` and calls `launch_agents_from_cwd(prompt)`
     without an origin.
   - A `generated` default would therefore either silently drop Telegram history or
     force a compatibility shim in the plugin.
   - Instead, an AST test requires every in-tree launcher call to pass `origin=`
     explicitly. That is the fail-closed guarantee for future automation surfaces.
5. **Relaunching a member agent is not a new request.** A TUI retry, kill-and-edit, or
   wait relaunch whose source agent belongs to a clan, is bound to a bead, or is a
   routine agent is recorded as `generated`, even if the user edited it. Swarms, epics,
   and routine batches all launch as clans (`%clan(…, tribe=…)`). A relaunch of a
   standalone agent stays `typed`.
6. **User-typed `---` segment rows stay.** A typed multi-prompt still records the whole
   text plus each long-enough segment.
7. **LaunchApproval prompts stay `generated`, even when edited before approval.**
8. **Clean up by pruning, not by hiding rows on read.** No "show generated" toggle, no
   separate launch log, no third origin value, and no feature flag. This corrects
   documented intent; no caller migrates and no old branch has to stay reachable.

## Phase `gate`

Files: `src/sase/history/prompt_store_mutations.py`,
`src/sase/agent/launch_cwd_agents.py`, `src/sase/agent/launch_cwd_bead_work.py`,
`src/sase/main/query_handler/_launch.py`,
`src/sase/notification_gates/command_runner.py`, docs, and tests.

1. **One origin resolver.** Replace the private `_effective_prompt_origin` with a public
   `effective_prompt_origin(origin, *, launch_envs=())`. It returns `"generated"` when
   any of these holds:
   - `origin == "generated"`;
   - any mapping in `launch_envs` satisfies `sase.axe.chop_agents.is_chop_launch_env`
     (these are launch parameters only, never `os.environ`);
   - `SASE_AGENT` is set in `os.environ`, whatever the origin.

   Otherwise it returns `origin` unchanged, so `None` stays `None`. Use it in
   `launch_agents_from_cwd_impl` and `launch_planned_bead_work_agents` with
   `launch_envs=(extra_env, *(segment_extra_env or ()))`. Then delete their inline
   `SASE_AGENT` checks and the failure-only `is_chop_launch_env` early return in their
   `record_failed_launch_prompt` closures; the gate now covers success and failure
   alike.

2. **Gate both writers.** `add_or_update_prompt` and `record_failed_launch_prompt`
   resolve the effective origin first. When it is `generated`, they return before
   `record_prompt_placeholders`, before `is_recordable_prompt`, before any shard
   mutation, and before `stash_failed_launch_prompt`.
   - As a result, a generated reuse of already-typed text no longer bumps that row's
     `last_used`.
   - `typed` and `None` behave exactly as today.
   - `merge_prompt_origin` and the reading of `generated` rows stay, because legacy rows
     remain until they are pruned.
3. **Ingress origin for `sase run`.** In `launch_query`, compute the origin once. It is
   `"generated"` when `SASE_MONITOR_ID` or `SASE_GATE_COMMAND` is non-empty in
   `os.environ`, and `"typed"` otherwise. Keep this in a small named helper with the
   variable names as constants. Replace every hard-coded `origin="typed"` in
   `_launch.py` with it: the project-tag failures, the force-reuse failures, and the
   three `launch_agents_from_cwd` calls. (`SASE_AGENT` never reaches these writes,
   because it diverts to LaunchApproval first.)
4. **Mark gate commands.** Define `GATE_COMMAND_ENV = "SASE_GATE_COMMAND"` in
   `command_runner.py`. Pass `env={**os.environ, GATE_COMMAND_ENV: "1"}` on both the
   `subprocess.run` path and the streaming `Popen` path of `run_owned_command`. The
   ingress helper imports the constant.
5. **Record the root prompt for paths that bypass the launcher.** This must land in the
   same phase. Today these launches leave only `generated` unit rows, and once those are
   gated they would vanish from history. Capture `history_query` in `launch_query`
   before the force-reuse rewrite.
   - **Direct typed admission** (`_dispatch_direct_typed_launch_if_active`). After a
     successful dispatch, call
     `add_or_update_prompt(history_query, allow_short=True, origin=<ingress origin>)`.
     On `LaunchRequestError`, call
     `record_failed_launch_prompt(history_query, origin=<ingress origin>)`.
   - **Remote-dispatch early return.** Apply the same success/failure recording to the
     query as forwarded. It is not yet project-tag-expanded, because remote dispatch
     forwards it verbatim. Recording on the target machine is unchanged.
6. **Enforcement test.** Add an AST test over `src/sase/**/*.py` that fails when a call
   omits the `origin=` keyword. It covers calls to these functions, whether made by bare
   name or by attribute (for example `launcher_mod.launch_agents_from_cwd`):
   - `launch_agents_from_cwd`
   - `launch_agent_from_cwd`
   - `launch_planned_bead_work_agents`
   - the injected `launch_agents_from_cwd_fn` and `launch_agent_from_cwd_fn` names
7. **Tests.**
   - **Rework `tests/history/test_prompt_origin.py`:**
     - `test_agent_context_forces_generated`: now nothing is written;
     - `test_failed_launch_records_origin`: a generated launch writes no row and no
       Stash entry;
     - `test_segments_inherit_origin`: segment rows are typed.
   - **Add writer tests:**
     - a generated add or failure leaves the shard, the common placeholder store, and
       the Stash untouched;
     - a generated add does not bump an existing typed row;
     - `None` still records.
   - **Add launch tests:**
     - a bead-work launch with N segments records 0 rows;
     - chop markers in `extra_env` record nothing, on both success and failure;
     - `SASE_MONITOR_ID` or `SASE_GATE_COMMAND` in the env makes `sase run` record
       nothing, while a plain terminal `sase run` records a typed row.
   - **Add gate and root-recording tests:**
     - `run_owned_command` sets the marker on both paths;
     - direct typed admission and remote dispatch each record the root once (cancelled
       on failure).
   - **Add a regression guard for decision 2:**
     `launch_agents_from_cwd(prompt, origin=None)` with `SASE_JOB_NAME` set in
     `os.environ` still records.
8. **Docs.**
   - Rewrite the origin paragraph in `docs/prompt.md` as a write policy: which entry
     points record; generated launches are not recorded; legacy generated rows remain
     until they are pruned.
   - In `docs/xprompt.md` (the multi-agent segment passage), say that segment recording
     applies only to multi-prompts a user submitted.

## Phase `canonical-text`

Files: `src/sase/agent/launch_cwd.py`, `launch_cwd_agents.py`, `launch_cwd_fanout.py`,
`launch_cwd_single.py`, `launch_cwd_guards.py`,
`src/sase/main/query_handler/_launch.py`, docs, and tests.

1. **`history_text` keyword.** Add `history_text: str | None = None` to
   `launch_agents_from_cwd`, `launch_agent_from_cwd`, and `launch_agents_from_cwd_impl`.
   The impl resolves the recorded text once: `history_text` if given, otherwise the
   query. Either goes through the same `canonicalize_project_aliases_in_prompt` step
   that produces today's `submitted_query`.
2. **One recorder per launch.** Build a small recorder in the impl (two closures or a
   frozen dataclass) exposing `record_submitted()` and `record_failed()`. Both always
   write the recorded text with the effective origin.
   - Pass the recorder to the guards and the fan-out/single branches in place of today's
     `record_failed_launch_prompt: Callable[[str], None]` parameter plus direct
     `add_or_update_prompt(...)` calls. No branch chooses its own text any more.
   - Keep each branch's timing: write after name validation and before spawn, and record
     a failure when an exception is raised.
   - Use `allow_short=True` when the launch fanned out through `---` segments or xprompt
     swarm expansion, as the multi-prompt branch does today, since a bare swarm trigger
     is short. Other launches keep the five-word threshold.
3. **Branch fixes.**
   - **Single-slot swarm:** record the invocation, not `expanded_segments[0]`.
   - **`%r:N`:** `launch_repeat_branch_if_applicable` records the text once, after
     validation, and recurses into the slots with `origin="generated"`. Slot rows
     (`%id:<base>.k`, `%wait:`) are therefore never written.
   - **Alt and multi-prompt:** record the recorded text. A multi-prompt still gets its
     own user-authored `---` segments; a swarm invocation has no `---`, so it produces
     one row.
4. **`sase run` payload keys** (in `launch_query`).
   - **`history_text`** (non-empty str) is the canonical text when the submitter rewrote
     the prompt before `sase run` (the TUI provider guard, next phase).
     - Apply the same project-tag expansion that `query` gets. This is best-effort; on
       error, keep the literal text.
     - Use it for every history write in `launch_query`, including the phase `gate` root
       recording, and pass it to the launcher.
   - **`history_origin`** is downgrade-only. Only `"generated"` is honored, and it
     overrides the ingress origin. Other values are ignored.
   - **Force-reuse:** pass the pre-rewrite query as `history_text`, so the row holds
     what the user submitted rather than `rewritten_prompt`.
5. **Tests.**
   - `#research_swarm(…)` records exactly one row.
   - A swarm that reduces to one slot (for example through static conditional segments)
     records the invocation.
   - `%r:3` records the parent once and no slot rows.
   - Force-reuse records the pre-rewrite text.
   - `launch_units` plus `history_text` records only `history_text`: no joined members,
     no member segments.
   - `history_origin="generated"` records nothing; an unknown `history_origin` value is
     ignored.
   - A typed `---` multi-prompt still records the whole text plus its segments.
   - Mobile launches are unchanged.
6. **Docs.** In `docs/prompt.md`, define the recorded text as the canonical submitted
   text, stated precisely.

## Phase `tui-provenance`

Read the `tui` reference memory first. The files are under `src/sase/ace/tui/actions/`:

- `agent_workflow/`: `_pending_launch.py`, `_launch_provider_guard.py`,
  `_launch_procs.py`, `_launch_submission.py`, `_prompt_bar_stash_store.py`,
  `_types.py`, `_entry_relaunch.py`, `_mentor_review.py`, `_prompt_bar_mount.py`,
  `_launch_prompt_inputs.py`
- `agents/`: `_marking_kill.py`, `_wait_actions.py`
- `agent_durable.py`

The sase-side contract is already in place: `sase run` accepts `history_text` and
`history_origin`.

1. **Pre-remodel text.**
   - Add `history_prompt: str | None = None` to `PendingLaunch`.
   - `_finish_provider_guard_launch` sets it to the current `launch.prompt` (only if it
     is unset) before either remodel overwrite: the single-unit case and the joined
     case.
   - When it is set and differs from `launch.prompt`, the submission payload includes
     `"history_text": launch.history_prompt`.
   - Failed-launch history and Stash writes that can run after a remodel use
     `launch.history_prompt or launch.prompt`. These are:
     - dead-worker recovery in `_launch_procs.py`;
     - the `restore_pending_launch_prompt` stash fallback;
     - `flush_pending_launch_stashes` at quit.
   - Bar restores keep today's text. Guard aborts already restore the original.
2. **Session origin.**
   - Add `prompt_origin: PromptOrigin = "typed"` to the prompt session (`_PromptSession`
     and `begin_prompt_session`). Copy it onto `PendingLaunch` in
     `begin_pending_launch`, and let `_finish_agent_launch` accept it for launches that
     have no bar.
   - When the origin is `"generated"`:
     - the submission payload carries `"history_origin": "generated"`;
     - `_save_text_as_cancelled` and the empty-editor cancel skip their history writes
       (file-reference recording may stay);
     - failed-launch recovery passes `origin="generated"`, which the gate turns into a
       no-op.
3. **What becomes `generated`.**
   - **Member relaunches:**
     - `_retry_edit_agent`;
     - `_kill_and_edit_agent` / `_finish_kill_and_edit_agent`;
     - marked bulk kill-and-edit (`_bulk_kill_marked_agents_and_edit` →
       `_edit_and_relaunch_agents_bulk`), generated if any source agent is a member;
     - wait relaunch (`_apply_wait_relaunch`).
   - **The member test:** a source agent is a member when any of these holds:
     - it has `agent_clan` set;
     - it is bead-bound: `epic_bead_id`, `phase_bead_id`, or a name-derived bead id;
     - its `tribe`/`clan_tribe` is the routine tribe (`chop`, public name `job`; see
       `src/sase/core/agent_tribe.py`).

     A standalone agent with a hand-assigned non-routine tribe stays `typed`. Implement
     the test as one pure, unit-tested predicate next to the TUI `Agent` model.

   - **Mentor apply** (`_launch_mentor_apply_agent`, which renders
     `#make_mentor_changes`) is always `generated`.
   - **Unchanged (`typed`):** history picks, the Ctrl+Y workflow editor, bulk Patch
     launch, and fleet dispatch.

4. **Tests** (TUI unit tests; live screenshots are not needed).
   - A provider-guard remodel of a swarm sends `history_text` equal to the original
     invocation.
   - Dead-worker and quit recovery stash the original text.
   - Kill-and-edit of a clan member submits `history_origin="generated"`, and cancelling
     that bar records nothing.
   - Kill-and-edit of a standalone agent stays `typed`.
   - Mentor apply is `generated`.
   - The member predicate is tested across its matrix of cases.

## Phase `prune`

This phase spans two repos: open `sase-core` with `sase repo open sase-core` and follow
its `AGENTS.md`. One declaration commits both repos. The host commits `sase-core` first
and moves `sase-core-revision.txt` automatically.

1. **sase-core classifier** (`crates/sase_core/src/prompt_prediction/origin.rs`,
   `looks_generated`).
   - Add the routine markers found in the live store: `tribe=chop`, `%tribe:chop`, and
     `%group:chop`. Add the `job` spellings too if the tribe grammar accepts that alias.
     Do not broaden the match to every `tribe=`, because humans assign tribes by hand.
   - Fix the doc comment so it matches the rules; it currently claims
     `%id(worker, tribe=quality)` matches.
   - Update the unit tests, and any prediction corpus or replay fixtures the new markers
     shift.
   - Expose the classifier through the `sase_core_py` prompt-prediction module as a
     boolean function over a text, plus its Python stub.
2. **sase adapter.** Add a thin wrapper under `src/sase/core/` that calls the binding.
   There is no Python fallback (`decisions:rust-core-required`).
3. **`sase prompt prune`** (`src/sase/main/parser_prompt.py`,
   `src/sase/prompt/cli_maintenance.py`, `src/sase/history/prompt_maintenance.py`).
   - **`-g/--generated`** selects rows whose merged origin is `generated`. "Merged"
     means after `load_prompt_history_for_write`'s cross-shard dedup, where typed wins.
     So a text that is typed anywhere is never removed, because removal works by exact
     text across all shards.
   - **`-l/--legacy`** requires `--generated`. It also selects origin-less rows that the
     core classifier flags.
   - Both intersect with `--before`, `--cancelled`, and the `--keep` floor, and keep the
     existing preview, confirm, `--yes`, and `--dry-run` flow. Help text is complete and
     options are sorted alphabetically (see the `cli_rules` memory).
   - The preview shows counts per tier (explicit vs. legacy heuristic) and a few
     truncated samples of each.
   - Every prune apply first writes a timestamped backup of the shard files. The backup
     must not match the `*.json` shard glob; the `.bak` convention from migration works.
     The command prints the backup path.
4. **`sase prompt doctor`.** Add origin counts (typed / generated / none) and the
   legacy-heuristic count to both the text and `-j` output.
5. **Do not apply a prune to the live store.** A dry run that reports counts in the
   phase notes is fine. The expected result on this machine is roughly 79 explicit rows
   plus about 8,100 legacy rows, out of about 11,600.
6. **Docs.** Update the prune and doctor sections of `docs/prompt.md`. Describe the
   one-time cleanup: run `sase prompt prune --generated --legacy --dry-run`, then apply.
7. **Tests.**
   - Extend `tests/history/test_prompt_maintenance.py` and
     `tests/prompt_command/test_maintenance.py`.
   - A typed copy in one shard protects a generated duplicate in another.
   - `--legacy` without `--generated` is rejected.
   - A backup is written before apply.
   - Dry run writes nothing.
   - Doctor reports the counts.
   - Add sase-core unit tests for the new markers and the binding.

## Not doing

- A read-side show/hide toggle, a separate launch log, a feature flag, a third origin
  value, text heuristics for new writes, or moving the store into Rust.
- Flipping the launcher `origin` default, or changing sase-telegram (decision 4).
  Telegram rows keep recording with no origin.
- Changing what the remote-dispatch target machine records. Its mobile bridge receives
  the forwarded prompt, and the Rust wire cannot tell a fleet dispatch from a phone
  launch.
- Applying the member-relaunch rule to mobile/Telegram retries. It covers TUI relaunches
  only.
- Scrubbing `<placeholder>` tags that generated templates already contributed to the
  common placeholder store.
