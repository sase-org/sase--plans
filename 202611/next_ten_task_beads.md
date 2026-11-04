---
tier: epic
title: 'Next-ten task beads: honest update-skew failures, reboot recovery, and the
  false-red test backlog'
goal: 'The ten task beads ranked by the 2026-11-04 next-ten research report are fixed,
  verified, and closed, together with the three beads the report attached to them
  (sase-1i6, sase-1jd, sase-12u). None of this work collides with the live Agent Hijack
  (sase-1qz), publication (sase-1r8), or v0.19.0 release (sase-1r2) epics.

  '
decisions:
  auto_relaunch:
    ask: Should reboot and zombie-epic recovery relaunch agents automatically, not
      just notify?
    default: true
    why: A manual step left 27 waiters stranded for 2.5 days after the 11-01 reboot
    answer: true
  next_epic:
    ask: After landing, should the lander propose the next backlog epic from the remaining
      task beads?
    default: true
    why: You are starting to work through the backlog, and the proposal still waits
      for your approval
    answer: true
phases:
- id: project_names
  title: -p accepts a project's display name, alias, or key (sase-zo)
  depends_on: []
  size: small
  description: 'project_names: canonicalize -p/--project through the project registry
    in resolve_project_context, list candidates on ambiguity, and add display-name-differs-from-key
    regressions (sase-zo).'
- id: editor_drafts
  title: A failed $EDITOR never deletes the typed draft (sase-1ce)
  depends_on: []
  size: small
  description: 'editor_drafts: on a non-zero editor exit or read failure, keep and
    recover changed prompt and workflow-YAML drafts instead of unlinking them (sase-1ce).'
- id: pytest_env
  title: 'Pytest isolation: scrub SASE_MODEL_OVERRIDE and O(1) per-test homes (sase-1q3,
    sase-1qe)'
  depends_on: []
  size: small
  description: 'pytest_env: add SASE_MODEL_OVERRIDE to the autouse env scrub and replace
    the per-test mktemp home allocation with a per-worker parent plus mkdtemp, both
    in tests/_conftest_environment.py (sase-1q3, sase-1qe).'
- id: token_fifo
  title: FIFO fairness for pytest worker-token waiters (sase-1pw)
  depends_on:
  - pytest_env
  size: medium
  description: 'token_fifo: give WorkerTokenLease waiters cross-process FIFO arrival
    tickets with bounded backfill and dead-waiter reclamation, proven by a fake-clock
    starvation test (sase-1pw).'
- id: false_red
  title: 'False-red batch: cold keymap test, MRU provider pruning, AF_UNIX path length
    (sase-1qu, sase-172, sase-1i6)'
  depends_on: []
  size: medium
  description: 'false_red: fix three tests that fail deterministically when run cold
    or alone, repairing product behavior where the root cause is a product bug (sase-1qu,
    sase-172, sase-1i6).'
- id: panel_pilot
  title: Command-line panel pilot trio passes on apollo (sase-1al)
  depends_on: []
  size: medium
  description: 'panel_pilot: remove the grammar-load race from the three test_panel_shell_pilot.py
    nodes without raising timeouts, and verify 20x serial plus 3x -n 8 on apollo (sase-1al).'
- id: skew_label
  title: Signature-skew TypeErrors are classified and explained as update skew (sase-1r4,
    Python half)
  depends_on: []
  size: medium
  description: 'skew_label: teach runner_failure_facts and code_swap_explanation the
    signature-mismatch TypeError shapes, gated on a proven code-identity move, without
    silencing failure notifications (sase-1r4 part 1).'
- id: staged_zombie
  title: Orphaned staged monitors settle and crashed members stay listed (sase-1r4,
    zombie half)
  depends_on:
  - skew_label
  size: medium
  description: 'staged_zombie: settle a staged monitor whose adopter died into a terminal
    held or failed state within one reconcile pass, and keep the crashed member row
    visible in sase agent list -a (sase-1r4 part 2).'
- id: skew_core
  title: sase-core auto-restart classifier gives signature skew an honest verdict
    (sase-1r4, core half)
  depends_on:
  - skew_label
  size: medium
  description: 'skew_core: after sase-1r2 closes, make the Rust agent_auto_restart
    catalog classify W1-witnessed signature TypeErrors as update skew and stop the
    bare provider marker from declining adoption tracebacks; move the core pin (sase-1r4
    part 3).'
- id: import_ratchet
  title: CI ratchet against post-provider first imports (sase-1jd)
  depends_on:
  - staged_zombie
  size: large
  description: 'import_ratchet: preload the staged-adoption import closure and add
    a subprocess import-recording test that fails on any new sase module first imported
    after provider_running, with a shrink-only allowlist (sase-1jd).'
- id: boot_recovery
  title: Host-owned boot reconciler relaunches reboot-stranded epics once per boot
    (sase-1qj)
  depends_on: []
  size: large
  description: 'boot_recovery: plan now, then implement after sase-1qz.6.1, sase-1qz.7,
    sase-1r2, and this epic''s skew_core close; a sase-core policy plus an AXE job
    finds prior-boot dead waiters and land agents, clears dead-owner claims, and recovers
    each epic once per boot (sase-1qj).'
- id: zombie_epics
  title: Zombie-epic report and host-owned land-agent liveness invariant (sase-12u)
  depends_on:
  - boot_recovery
  size: large
  description: 'zombie_epics: let sase-core see closed children, ship a report-only
    zombie-epic listing for owner triage, and add a watermarked, rate-limited invariant
    that recovers a parent''s land agent when every child is closed and none is live
    (sase-12u).'
proposed_by: bbugyi200.athena.18d
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-04 17:59:09
status: wip
bead_id: sase-1ra
---

- **PROMPT:** [prompts/202611/next_ten_task_beads.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/next_ten_task_beads.md)
- **BEAD:** [sase-1ra](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ra/README.md)

# Next-Ten Task Beads: Honest Update-Skew Failures, Reboot Recovery, and the False-Red Test Backlog

## Context

The research report
`research:202611/next_ten_task_beads_live_epic_screen/next_ten_task_beads_live_epic_screen.md`
(swarm `3a`, snapshot 2026-11-04 20:44 UTC, master `34b09fa219`) ranked the ten task
beads most worth working now. It screened them against the live epics on apollo and
athena. It also attached three beads to them:

- sase-1i6 as a ride-along in the false-red batch;
- sase-1jd as the follow-on to #1;
- sase-12u together with #4.

This epic implements all thirteen.

| Rank | Bead(s)                         | Phase                                                |
| ---- | ------------------------------- | ---------------------------------------------------- |
| 1    | sase-1r4 (+ sase-1jd)           | skew_label, staged_zombie, skew_core, import_ratchet |
| 2    | sase-1al                        | panel_pilot                                          |
| 3    | sase-zo                         | project_names                                        |
| 4    | sase-1qj (+ sase-12u)           | boot_recovery, zombie_epics                          |
| 5    | sase-1pw                        | token_fifo                                           |
| 6–7  | sase-1q3, sase-1qe              | pytest_env                                           |
| 8    | sase-1ce                        | editor_drafts                                        |
| 9–10 | sase-1qu, sase-172 (+ sase-1i6) | false_red                                            |

**Re-verified while planning (2026-11-04, athena, master `cdfbb82d2b`):**

- **Reproductions.** `sase repo list -p sase` still fails with "Project 'sase' has no
  WORKSPACE_DIR". Each of these still fails when run alone with a raw single-node
  pytest:
  - the sase-172 node: `assert ['#gh:sase'] == ['#gh:sase', '#git:otherproj']`;
  - the sase-1qu node: `AssertionError: keypress dispatch must stay in-memory`, raised
    from `src/sase/ace/tui/keymaps/defaults.py:18`, `_builtin_keymaps_config` →
    `read_text`;
  - `tests/test_dispatch_federation_ipc.py`: 2 failed, 5 passed (sase-1i6).
- **Live owners.**
  - sase-1qz: phases `.6.1.4` (planning), `.6.1.5`–`.7`, `.7`, `.9`, `.10`, `.11`, and
    `.land` are all still open.
  - sase-1r8: `.5` is running.
  - sase-1r2: `.5--mon` is WAITING PYPI; `.6`, `.7`, `.8`, and `.land` are open.
  - The only other live agents are bob-cli work. The report's `18a`/`18b` coders have
    finished.
- **Owner decisions are not part of this epic.** sase-j0 and sase-1ot belong to the
  owner.

**Capacity.** The report advises against running everything at once: athena already
carries about 32 agent rows, and pytest worker tokens are the shared bottleneck. Seven
phases have no in-epic dependency, so approve with a launch capacity of about 3–4.

## Rules for every phase

- **Read and reproduce.** Read each assigned task bead in full, including every note and
  +1, with `sase bead read <id> -r "<why>"`. Reproduce the failure before changing code
  wherever it is reproducible. Record the base commit, host, before/after results, and
  limits on the phase bead.
- **Stay out of live-epic files.** The Agent Hijack epic sase-1qz owns these; do not
  edit them:
  - `src/sase/core/wait_dependency_resolution/`
  - `src/sase/ace/scheduler/stale_running_cleanup.py`
  - `src/sase/agent/_restart_planning.py`, `src/sase/agent/_restart_execute.py`, and
    `_dismiss_persistence.py`
  - `src/sase/agent/hijack/`
  - `src/sase/ace/tui/actions/agent_workflow/_prompt_bar_stash.py`
  - every `_has_staged_handoff` check:
    - `axe/run_agent_runner_signals.py`
    - `axe/_run_agent_runner_shutdown_finalize.py`
    - `workspace_provider/vcs_release.py`
    - `finalizers/_failed_close_recovery_assess.py`
    - `agent/auto_restart/inputs.py`
  - `_staged_origin_retains_claim`.

  Reading these files is fine. If a fix seems to need one of them, record a
  `PROPOSED FOLLOW-UP:` instead, and append a coordination note to the owning sase-1qz
  bead. Before editing, re-run `sase agent list` to confirm that no newly launched agent
  owns your files.

- **Keep the guards.** Keep assertions, safety refusals, and existing budgets. Do not
  add skips, xfails, flake-baseline entries, timeout or elapsed-bound increases, or
  suppressions just to make a gate pass.
- **Rust core boundary.** Shared backend behavior belongs in sase-core. Open it with
  `/sase_repo` (`sase repo open sase-core`). Presentation, test-fixture, and Python I/O
  fixes stay in this repo. Let host completion move `sase-core-revision.txt`, or use
  `just ratchet-core-revision`. Never invent a SHA.
- **Core-pin gate.** Do not commit to sase-core or move the core pin until bead sase-1r2
  is closed. sase-1r2.5–.8 need master's source pin to stay inside the published
  sase-core-rs until v0.19.0 reaches PyPI.
- **How to wait on an external gate.** Check gate beads with `sase bead read`. If one is
  still open, finish every ungated step first (reading, design, tests that need no new
  binding). Then start a pure-wait `/sase_monitor` (`-w 0`) whose command polls the gate
  beads every 15 minutes until they are closed. Use the status pair `WAITING <bead-id>`
  → `WAITED <bead-id>`, and continue in the monitor's follow-up. Never poll inline, and
  never edit a gate bead.
- **Verification.** Read the `lint_and_test` reference memory before finishing. Also
  read `tui` and `tui_perf` before TUI changes, `cli_rules` before adding CLI
  subcommands or options, and `docs_writing` before editing `docs/`. Run focused
  regressions, then `sase tool run check` in each changed repository. Do not run
  `just check-full`.
- **Bead hygiene.** Phase workers never create beads; record out-of-scope findings as
  `PROPOSED FOLLOW-UP:` notes on the phase bead. Record
  `TASK RESOLVED: <task-id> — <evidence>` on the phase bead and close only the phase
  bead. The lander closes the task beads.

## Phase project_names: `-p` accepts a project's display name, alias, or key (sase-zo)

**Defect.** The `match_repo_record` half of sase-zo was fixed in `e57d76c79b`. The `-p`
half remains:

- `resolve_project_context` in `src/sase/main/workspace_handler_context.py:47` passes
  `-p` straight to `get_project_file_path`, which only understands the directory key.
- On athena, `-p sase` lands on a stray `~/.sase/projects/sase/sase.sase` with no
  `WORKSPACE_DIR` and exits 2.
- Under decision `project-labels-never-keys`, the display name is what every surface
  shows agents.

**Fix.**

- Canonicalize the `-p` value once, inside `resolve_project_context`, through the
  project registry:
  - accept an exact registered key, the display name (`PROJECT_NAME`), or an alias,
    case-insensitively for names and aliases;
  - reuse the resolver `sase project show` uses (`resolve_project_alias_ref` in
    `src/sase/project_aliases.py`, plus the `find_project_ref_owner` display-name
    matching);
  - do not write a new matcher.
- A registered project wins over an unregistered stray project directory with the same
  name.
- When a value matches more than one project, exit 2 and list the candidates with both
  key and display name.
- Audit every caller of the helper so all of them get the fix:
  - `repo_handler_list.py`, `repo_handler_open.py`, `repo_handler_path.py`,
    `repo_handler.py`
  - the `workspace_handler_*` modules
  - `patch_handler.py`

**Tests.**

- A registry fixture whose display name differs from its key. Cover `sase repo list`,
  `repo open`, `repo path`, and one `workspace` command with `-p <display>`,
  `-p <alias>`, and `-p <key>`.
- An ambiguity case.
- A stray-directory shadowing case.

**Done when** `sase repo list -p sase` works on athena and the regressions pass.

## Phase editor_drafts: a failed `$EDITOR` never deletes the typed draft (sase-1ce)

**Defect.** In `src/sase/ace/tui/actions/agent_workflow/_editor.py`, both editors unlink
the temp file in their `finally` block:

- `_open_editor_for_agent_prompt` → `run_editor` returns `None` on a non-zero exit;
- `_open_workflow_yaml_editor` → `run_editor` has the same shape.

So `:cq`, an editor crash, or a kill loses everything typed.

**Fix.**

- **Detect a changed draft.** On a non-zero exit, or when reading the file back fails,
  compare the file with the initial content. Unlink the temp file only on success, or
  when nothing changed.
- **Prompt editor.** Save a changed draft to the Prompt Stash, following the
  non-prompt-bar precedent in `src/sase/agent/failed_launch_prompt_stash.py`. Use
  `append_prompt_stash` from `src/sase/core/prompt_stash_facade.py`, with a distinct
  recovery `source` (for example `editor_recovered`) and the current project. Keep the
  temp file, and show a toast that names the stash entry and the file path. Never
  auto-submit the draft, and never route it through `_prompt_bar_stash.py`, which the
  hijack phase sase-1qz.10 owns.
- **Workflow-YAML editor.** Keep a changed temp file and toast its path. A workflow
  template is not a prompt, so do not stash it.
- **If the stash write fails,** still keep the file and toast its path; the draft must
  survive.

**Tests.**

- Cover a non-zero exit with a changed draft, a non-zero exit with an unchanged draft, a
  read failure, and a successful edit, for both editors. Fake the editor subprocess.
- Assert stash contents, file retention, and toast text.

Read `tui` first. If a toast's rendered output reaches a PNG golden, follow the snapshot
rules in `lint_and_test`.

## Phase pytest_env: pytest isolation for SASE_MODEL_OVERRIDE and O(1) per-test homes (sase-1q3, sase-1qe)

Both fixes edit `tests/_conftest_environment.py`, so one worker does both.

**sase-1q3.**

- _Defect._ `_clear_agent_env_vars` (around lines 437–500) scrubs
  `SASE_MODEL_ALIAS_OVERRIDES` but not `SASE_MODEL_OVERRIDE`. The runner sets that
  variable on fallback-model retries (`src/sase/axe/run_agent_exec_retry.py:494`). With
  it set, the bead's control selection gives 7 failed / 44 passed; on a clean
  environment it gives 51 passed.
- _Fix._ Add `SASE_MODEL_OVERRIDE` to the per-test scrub and the teardown pop list.
  Explicit per-test `monkeypatch.setenv` must keep working.
- _Audit._ Look at the other `os.environ[...] =` writes under `src/sase/axe/` for
  ambient runner variables that leak in the same way. Add only the ones you prove change
  a test outcome.
- _Test._ Add a subprocess-level regression: run the bead's selection, or a small
  representative node set, with an inherited `SASE_MODEL_OVERRIDE=sonnet` and with a
  clean environment, and assert the results are identical.

**sase-1qe.**

- _Defect._ `_isolate_sase_home` calls `tmp_path_factory.mktemp("home")` for every test,
  around line 286. pytest's `make_numbered_dir` lists the whole basetemp on each call,
  so a worker's allocation cost grows quadratically. One worker had 27,816 entries
  during a two-hour check.
- _Fix._ Allocate one parent directory per worker, once and lazily, through
  `tmp_path_factory`. Create each test's home with `tempfile.mkdtemp(dir=parent)`.
  - Homes must stay unique per test and outside each test's own `tmp_path`.
  - They must still be cleaned up with basetemp retention.
  - Grep for tests that depend on the old `home<N>` naming or location, and fix them.
- _Evidence._ Record a before/after microbenchmark at 1k, 10k, and 30k existing entries
  on the phase bead. It is a microbenchmark, not a suite-wide speedup claim.

## Phase token_fifo: FIFO fairness for pytest worker-token waiters (sase-1pw)

**Defect.** `WorkerTokenLease.acquire` in `tests/_suite_gate_lease.py` (around lines
121–163) polls `_attempt` with no arrival order and grows greedily. Later runs that
happen to poll first take freed tokens. A four-token waiter burned its whole 2,700 s and
exited 4 without running a test (ToolRun `43ddd1bdeb9d756471872ee0a7e2bbe5`).

**Fix.** Waiters are separate processes, so the tickets must live in the shared pool
state, under the existing pool lock.

- **Ordering.**
  - Give each blocking waiter a FIFO arrival ticket.
  - The oldest feasible waiter reserves freed tokens up to its floor.
  - Later requests, including `try_acquire`, may backfill only tokens that do not delay
    that reservation, and only boundedly.
- **Non-blocking contract.** Keep `try_acquire` non-blocking: it returns at once, and
  never takes reserved tokens.
- **Dead waiters.** Reclaim the tickets of dead waiters, using process identity rather
  than a bare pid, so pid reuse cannot pin a ticket. Keep crash-safe holder ownership
  and the existing stale-holder reclaim.
- **Timeout.** Do not lengthen the 45-minute timeout.
- **Other users.** Check `tools/run_pytest`, `tests/_suite_gate.py`, and
  `tests/_test_selection_gear.py` for assumptions about lease order.

**Proof.**

- A fake-clock, fake-poll unit test replays the bead's sequence:
  1. Hold the pool at capacity.
  2. Waiter A (floor 4) waits.
  3. Release the holder; waiter B arrives before A's next poll.
  4. A gets the grant, and B either waits or backfills only within bounds.
- Add a dead-waiter reclamation test and a `try_acquire`-respects-reservation test.
- Extend `tests/test_suite_gate_lease.py` and `tests/test_suite_gate_reclaim.py` rather
  than creating parallel files.

## Phase false_red: cold-run and run-alone false reds (sase-1qu, sase-172, sase-1i6)

All three nodes fail deterministically when run cold or alone, so any scoped selection
that hits them gives an agent a false red.

**sase-1qu.**

- _Defect._
  `tests/ace/tui/test_log_panel_keymap.py::test_update_shortcut_dispatch_performs_no_disk_or_subprocess_work`
  patches `Path.read_text` before the `functools.cache`d `_builtin_keymaps_config` in
  `src/sase/ace/tui/keymaps/defaults.py` has been warmed.
- _Decide first._ Does the real app warm the keymap defaults before the first keypress,
  or does the first keypress parse `default_config.yml` on the UI thread?
  - **If it reads on the UI thread,** warm the defaults during startup, where keymaps
    already load. Read `tui_perf` first, and do not add UI-thread work at mount. This
    fixes the product and the test together.
  - **If the app always warms before input,** warm the cache in the test before it
    installs the patches.
- _Either way,_ keep the "no disk or subprocess work" assertion, and add a test that
  pins the warm-before-input guarantee.

**sase-172.**

- _Defect._
  `tests/test_vcs_macro_mru_pruning.py::test_load_launchable_prunes_provider_mismatched_prefix`
  prunes the genuine `#git:otherproj` entry when run alone and passes inside the full
  suite.
- _Suspect._ The bead narrows it to `_vcs_prefix_provider_mismatched` and
  `_workflow_and_ref_for_prefix` (`src/sase/history/vcs_macro_mru.py`, around lines
  450–540). The latter iterates `get_ref_patterns()` in plugin order.
- _Find the input that flips it._ Candidates are pattern order, the
  `detect_workflow_type` import path bypassing the monkeypatch, and a derived cache that
  an earlier test warms.
- _Fix._
  - **Product bug** (for example, an earlier provider's pattern claiming an explicit
    `#git:` sigil): match the explicit `#git:` / `#gh:` sigil before falling back to
    pattern iteration. On a host with sase-github installed, genuine bare-git MRU
    entries must survive.
  - **Test-only cause:** fix the isolation.
  - **In both cases,** make the test hermetic about which provider plugins are
    registered, and add a variant with sase-github's patterns registered first.

**sase-1i6.**

- _Defect._ `tests/test_dispatch_federation_ipc.py` binds AF_UNIX sockets under
  `tmp_path`. Under the agent temp root, that path exceeds the 108-byte `sun_path`
  limit.
- _Fix._ Bind under a short `tempfile.mkdtemp(dir="/tmp")` directory and remove it in
  teardown.
- _Production check._ Check whether production federation socket paths can exceed the
  limit too. If they can, record a `PROPOSED FOLLOW-UP:` rather than widening this
  phase.

**Done when** each node passes alone, in a fresh process, and inside its module.

## Phase panel_pilot: the command-line panel pilot trio passes on apollo (sase-1al)

**The failing nodes.** These three nodes in
`tests/ace/tui/command_line/test_panel_shell_pilot.py` fail together on apollo. They
failed in every red apollo full run since 11-03 (9 of 9), including a clean-base run:

- `test_empty_panel_semicolon_hops_to_palette_and_back`
- `test_empty_panel_escape_follows_the_hide_panel_binding`
- `test_panel_escape_keeps_draft_across_reopen`

On athena they failed 1 time in 625.

**Unverified hypothesis.** Each test waits 5 s for `is_command_line_grammar_pending()`
to clear, while the grammar spec (about 0.5 MB) is built in a subprocess. That can
exceed 5 s on the slower, 16-CPU host.

**Steps.**

1. **Reproduce on apollo.** Never run tests inside an apollo SASE workspace, and remove
   every scratch path afterwards. Use the recipe proved in the sase-1qt epic plan
   (`plan:202611/task_backlog_epic_5.md`):
   1. `git archive HEAD | ssh apollo 'mkdir -p /tmp/<scratch> && tar -x -C /tmp/<scratch>'`.
   2. Create a scratch `~/.local/bin/uv venv -p 3.12` on apollo.
   3. Install `-e ".[dev]"`, overriding `sase-core-rs` with athena's cached abi3 wheel
      from `~/.sase/cache/sase-core-artifacts/` (copy it with `scp`).
   4. Run the trio under `-n 8` alongside `tests/ace/tui/`.

   If apollo is unreachable, emulate the slow host on athena, for example with `taskset`
   and a concurrent load, and say so in the phase notes.

2. **Fix.** If the hypothesis holds, give the trio a session-scoped, pre-built grammar
   fixture. Keep the async-load coverage in the loader-specific tests
   (`test_grammar_loader_notifies_every_pending_screen_callback`,
   `test_grammar_ready_refreshes_open_empty_panel_without_a_keystroke`, and
   `test_grammar_readiness_through_real_loader_with_in_process_grammar`). Do not raise
   timeouts.
3. **Product lag.** If `;` or escape genuinely cannot act while the grammar is pending,
   treat that as product lag on slow hosts: let the hop or hide proceed, and fill in
   completion later. Read `tui` and `tui_perf` first.

**Done when** the three tests pass 20 times serially, and 3 times under `-n 8` with
`tests/ace/tui/`, on apollo. If apollo was unreachable, record the exact recipe for the
lander.

## Phase skew_label: signature-skew TypeErrors are classified and explained as update skew (sase-1r4, Python half)

**Incident.** At 2026-11-03 10:56, a live `sase update` (`a1a07efff8` → `ced5901dd5`)
removed two keyword arguments:

- `require_capture` from `capture_monitor_handoff()`;
- `persist_chat` from `handle_gate_marker()`.

Three runners died at function-level lazy imports during staged-handoff adoption,
raising `TypeError: … unexpected keyword argument`. Nothing classified those deaths or
explained them. This phase is Python-only.

**Classify** (`src/sase/axe/runner_failure_facts.py`).

- Add the signature-mismatch shapes:
  - `unexpected keyword argument`
  - `missing N required (positional|keyword-only) argument`
  - `takes N positional arguments but M were given`
- These shapes count as skew **only** when the caller supplies a proven code-identity
  move. The module must stay stdlib-only, with no function-local imports; the AST test
  in `tests/axe/test_runner_failure_facts.py` enforces this. So pass a new keyword-only
  input, such as `code_moved: bool`, from the callers.
- A same-revision TypeError must stay an application failure.
- Sites that build facts:
  - `run_agent_runner_errors.py` around line 76;
  - `run_agent_exec_finalize.py` around lines 342 and 408;
  - `_run_agent_runner_shutdown_markers.py` around line 81.

**Compute `code_moved`** by comparing the runner's boot identity (`_BOOT_CODE_IDENTITY`
in `run_agent_runner.py`, persisted as `agent_meta.json["code_identity"]`) with a
**fresh** current identity. `managed_roots.current_code_identity()` is memoized per
process, so use a non-memoized read, for example
`run_agent_runner_refresh.runner_code_identity()` or
`source_skew._source_revision_changed()`.

**Explain** (`src/sase/axe/source_skew.py`, `code_swap_explanation()`).

- Today it labels only ImportError and AttributeError chains.
- Label signature-mismatch TypeErrors when the revision moved, with wording that says
  the sase checkout moved under the runner and a called signature changed. The current
  "deferred import" wording does not fit this case.
- Its callers are `run_agent_runner_errors.py:70` and
  `run_agent_exec_plan_accept_sdd.py:204-210`.

**Keep users notified.**

- Setting `skew_suspect=true` routes the runner's failure into pending recovery
  (`run_agent_runner_errors.py`, around lines 78–112) and may suppress its own failure
  notification. Verify that every newly-skew path still ends in a user-visible
  notification that carries the explanation.
- For the 11-03 shape, a held handoff receipt exists. The healer's skip rules decline
  that shape loudly with `needs_user_decision` before any classification.
- If any path would go silent, keep the runner's own notification for it.
- Do not touch the `_has_staged_handoff` checks; `agent/auto_restart/inputs.py` is
  hijack-adjacent.

**Tests.**

- Predicate cases: the three shapes, each with `code_moved` true and false.
- Explanation cases in `tests/axe/test_source_skew.py`.
- An 11-03 incident replay, using `tests/test_agent_auto_restart_incident_replay.py` as
  the template. It asserts:
  - the signature change gives `skew_suspect=true` and the healer admits it as a
    candidate;
  - a same-revision TypeError stays ineligible;
  - a post-provider death is never relaunched, so no completed provider work reruns;
  - a notification is emitted.

Do not change sase-core here; the skew_core phase owns the Rust verdict.

## Phase staged_zombie: orphaned staged monitors settle, and crashed members stay listed (sase-1r4, zombie half)

**Zombie monitor.**

- _What happened._ In the 11-03 incident, monitor `vdgpvm` (`17a--mon-2`) stayed
  `monitor_state=staged`, `pid=null`, bucket Running for more than six hours.
- _Why._ After a failed adoption, `dispatch_completed_staged_handoff` records a held
  receipt, and `cleanup_staged_monitor_handoff(held=True)` only sets
  `monitor_stage_held` (`start_stage_cleanup.py`, around lines 138–151). `staged` still
  buckets as Running (`monitor_state.py`, around lines 22–23). The held cleanup also
  lazily imports `sase.monitor.start_stage` and swallows its own failure
  (`run_agent_exec_staged_monitor.py:41`), so it is itself skew-vulnerable.
- _Gap._ No sweeper covers this:
  - dead-supervisor reconcile skips rows that are not running or have `pid=None`
    (`monitor/reconcile.py`, around lines 64–73);
  - `staged_handoff_reconcile` receives a Rust `Hold` for held handoffs.

**Fix.**

- **Stop the Running bucket.** A staged monitor whose handoff is held, or whose adopter
  is provably dead, must not bucket as Running.
- **Backstop pass.** In an existing reconcile pass, settle such a monitor into a
  terminal held or failed state within one pass. Good places are the hold branch of
  `reconcile_staged_handoffs` (`staged_handoff_reconcile.py`, around lines 125–139) or
  `sweep.run_job_tick`, next to `_settle_launched_records`. Use the existing
  `sase.core.staged_handoff.evaluate_staged_owner_liveness` policy to decide adopter
  death.
- **Preserve:**
  - custody, the accepted declaration, and dirty-repo obligations;
  - the Rust "held handoff requires explicit re-adoption" decision.

  Never launch the monitor's command, and never delete the record.

- **If Rust changes prove necessary,** apply the core-pin gate, or record a
  `PROPOSED FOLLOW-UP:`.

**Missing list row.**

- _Symptom._ The crashed `17a--3` member vanished from `sase agent list -a`.
- _Hypothesis to verify first._ `state.current_artifacts_dir` is updated only after the
  execution loop returns (`run_agent_runner_launch.py`, around line 421). So the error
  `done.json` lands in the launch root (`run_agent_runner.py`, around line 377) instead
  of the member directory. A member with `parent_timestamp` and no `done.json` is hidden
  (`_running_listing_done.py`, around lines 51–52).
- _Fix._ The failed member must get its own terminal `done.json` and show in `-a`.
- _If the hypothesis is wrong,_ fix the actual cause, or record a `PROPOSED FOLLOW-UP:`
  with the evidence.

**Tests.** Extend:

- `tests/monitor/test_monitor_start_stage_cleanup.py`
- `tests/test_staged_handoff_reconcile.py`
- `tests/test_staged_failure_boundaries.py`
- `tests/test_agent_list_entries.py`

Add an orphaned-monitor case that settles in one pass, and a crashed-member-listed case.

**Coordination.** If staged-adoption semantics change, append a note describing the
change to sase-1qz.6.1 for its lander.

## Phase skew_core: the sase-core auto-restart classifier gives signature skew an honest verdict (sase-1r4, core half)

**Gate.** Bead sase-1r2 must be closed before any sase-core commit or pin move (see
Rules). Do the reading and design first, then wait if needed.

**Defects** (sase-core `crates/sase_core/src/agent_auto_restart/`, verified at
`542448a0`):

- The `catalog.rs` TypeError mapping (around lines 359–366) recognizes only messages
  containing `signature`. An `unexpected keyword argument` therefore gets the "real bug"
  verdict (`no_update_signature`), even when the W1 code-identity witness fires
  (`classify.rs`, around lines 391–396).
- `never_restart_reason` (`catalog.rs`, around lines 46–62 and 115–175) matches a bare
  `provider` marker over the error, traceback, and log tail. Adoption tracebacks that
  name `provider_response` or `llm_provider` paths are declined as `provider_error`.

**Fix.**

- **Signature TypeErrors.** Classify the three signature-mismatch shapes as update skew
  only when W1 fires. Without W1, a TypeError stays `tier4_real_bug`.
- **The provider marker.** Narrow it to genuine provider-failure shapes, so it stops
  matching incidental identifiers and paths.
- **Phase gating.** Keep it intact: post-provider deaths stay `notify_post_provider`,
  and only `pre_provider` deaths may relaunch. Resume-in-place belongs to sase-1jf and
  is out of scope.
- **Tests and docs.** Update the Rust tests (`agent_auto_restart/tests.rs`), the PyO3
  tests (`crates/sase_core_py/src/agent_auto_restart/tests.rs`), the parity test
  `tests/test_core_agent_auto_restart.py`, and `docs/reference/agent-auto-restart.md`.
- **The pin.** Commit both repos in one turn so host completion moves the pin.

**Done when** the 11-03 replay, run through the real binding, yields an honest
update-skew verdict with no relaunch of post-provider work, and a same-revision
TypeError still yields the real-bug verdict.

## Phase import_ratchet: a CI ratchet against post-provider first imports (sase-1jd)

**Why.** Update-skew deaths after the model turn come from modules first imported after
the provider starts. The 11-03 adopters' lazy imports are all post-provider and absent
from `preload_post_gate_modules`:

- in `run_agent_exec_staged_adopters.py`: `run_agent_exec_monitor`,
  `run_agent_exec_staged_gate` → `run_agent_exec_gate`, `run_agent_exec_plan`,
  `run_agent_exec_questions`, and `run_agent_exec_pipe`;
- in `sase.monitor`: `start_stage` and `staged_owner`;
- `plan_chain`.

`preload_post_gate_modules` is defined in `src/sase/axe/source_skew.py`, around lines
196–223, and has one call site, `run_agent_runner_launch.py:392`.

**Design.** This phase is large, so plan first.

- **Preload.** Extend `preload_post_gate_modules` to cover the staged-adoption import
  closure. Measure the boot-time cost per runner and keep it bounded.
- **The test.** Add a CI test that runs a representative runner turn in a subprocess and
  records sase module imports with a `MetaPathFinder`:
  - template: `tests/test_run_agent_runner_refresh_macros.py`, around lines 222–300;
  - harness: the fakey harness in `tests/fakey/harness.py`;
  - boundary: `mark_lifecycle_phase("provider_running")`.

  It fails on any sase module first imported after that boundary, unless the module is
  in an explicit, shrink-only allowlist with a reason for each entry.

- **Coverage.** Exercise at least the plan, question, monitor, gate, and pipe staged
  handoffs.

**Done when** the test fails when a new post-provider first import is added in a scratch
check, and the allowlist only ever shrinks.

## Phase boot_recovery: a host-owned boot reconciler relaunches reboot-stranded epics once per boot (sase-1qj)

**Plan now; implement later.** The planner may start immediately. Before editing any
code, the implementer must confirm that all of these are closed, and wait otherwise:

- sase-1qz.6.1 (keep-work recovery and crash repair);
- sase-1qz.7 (restart rewrite);
- sase-1r2 (the core-pin gate);
- this epic's skew_core phase bead, so that two core-pin moves never race.

**Evidence.**

- The 11-01 athena reboot left 27 waiters FAILED (16 land, 11 phase), with dead PIDs and
  no `done.json`. They were still unrecovered 2.5 days later.
- 7 of 8 `.f0` relaunches died with "already in_progress and assigned". That came from
  ACE's stranded Continue fork path, not from `sase bead work`:
  - `issue_is_in_progress_for_another_agent` in `src/sase/bead/force_reuse.py` (around
    lines 36–45) compares agent names only, never liveness;
  - the `.f@` fork name never matches the dead assignee.
- The only mitigation was a per-epic crontab watchdog. It exists only as plan prose in
  `plan:202611/v0_19_0_green_full_ci_release.md` (decision `reboot_watchdog` and "Phase
  watchdog"), plus one installed instance under
  `~/.local/state/sase-epic-watchdog/sase-1r2/`. That prose is a good spec for the
  checks: same boot, uptime, innermost epic, liveness, and a `bead work -n` preview
  refusal.

**Design constraints.**

- **Policy in sase-core.** Add a new pure module, for example `boot_recovery`. Its input
  is the previous and current boot ids, the in-progress epics with their phase, land,
  and `%w` rows (pid, boot id, `done.json` presence), hijack-custody flags, and durable
  markers. Its output is a per-epic action: recover, skip with a reason, or already
  handled.
- **I/O in Python.**
  - New modules such as `src/sase/agent/boot_recovery/` and
    `scripts/sase_chop_boot_recovery.py`.
  - Add a job in `src/sase/default_config.yml`. Each lumberjack runs its first tick on
    start (`axe/lumberjack.py`, around lines 541–542), so a cheap boot-id check makes
    the job effectively run once per AXE start.
  - Copy the durable-proc pattern of `agent_auto_restart`
    (`agent/auto_restart/sweep.py`, around lines 245–290).
- **Proving death in an earlier boot.**
  - Every agent row's `agent_meta.json` records
    `process_identity = "<boot_id>:<start_ticks>"`.
  - Prior art: `_is_pre_reboot_monitor` in `monitor/reconcile.py`, around lines 302–338.
  - Reuse `axe/wait_marker_scan.py`'s dead/alive/unknown classification.
  - Unknown is never treated as dead.
- **Stale claims.** Make the runner's claim check boot- and liveness-aware: a claim held
  by an owner provably dead in an earlier boot must not block a relaunch. Keep
  live-owner protection exactly as it is.
- **Hijack custody.** Read it read-only (`agent/hijack/custody.py`,
  `matching_active_record`). Never release, relaunch, or notify about a bead or agent
  under hijack custody; hijack crash repair owns those.
- **Once per boot.** Recover each epic once per boot, behind a durable marker. A second
  scheduler start must do nothing. Honor admission and capacity so a reboot does not
  stampede the host.
- **Notices.** Post one inbox notice per epic through `notifications/store.py`
  `upsert_notification` with a dedup key, following `auto_restart/_notifications.py`.
- **Retire the watchdog.** Document the reconciler in
  `docs/reference/automation/scheduler-default-routines.md` and
  `docs/reference/agent-auto-restart.md` as the replacement for per-epic crontab
  watchdogs. Removing an installed cron entry is a human action, so propose it through
  `/sase_gate` if one is still installed.

> [!decision] auto_relaunch
>
> **Automatic recovery.** Relaunch each stranded epic through the existing epic launch
> path (`launch_epic_bead_work`), never through `sase agent restart`. Run it
> non-interactively with the cleanup confirmations it needs. The epic's notice says what
> was relaunched.

> [!decision] auto_relaunch = no
>
> **Notify only.** Do not relaunch automatically. Each stranded epic gets one inbox
> notice that names the stranded rows and the exact `sase bead work <epic>` command. The
> stale-claim fix still ships, so a manual or ACE Continue relaunch no longer dies on a
> dead owner's claim.

**Done when:**

- a simulated reboot fixture recovers each stranded epic exactly once;
- a second scheduler start does nothing;
- no relaunch dies on a stale claim;
- hijacked beads are untouched;
- unknown liveness is never released.

## Phase zombie_epics: a zombie-epic report and a host-owned land-agent liveness invariant (sase-12u)

**Defect.**

- `close_one_and_delegated_parent` (sase-core
  `crates/sase_core/src/bead/mutation/close_remove.rs`, around lines 549–623) cascades a
  close only to a parent phase.
- When a follow-up epic's parent is a plan bead, the parent stays `in_progress` if the
  land agent's ancestor walk (`bd/land_epic` in `src/sase/default_config.yml`, around
  lines 2389–2405) is skipped or lost.
- 39 of 120 `in_progress` epics have every child closed; examples are sase-zn, sase-126,
  sase-j7, sase-kp, and sase-1q4.
- sase-core's stranded-work reducer (`stranded_work/`) cannot compute "every child
  closed": the Python collector (`bead/stranded_facts.py`, around lines 122–159) loads
  only open and in-progress candidates.

**Fix.** The core-pin gate is already satisfied once boot_recovery has closed; re-check
it anyway.

1. **Let sase-core see closed children.** Extend the stranded-work wire, or add a
   sibling reducer, so sase-core can compute "all children closed and no live land
   agent".
2. **Report.** Ship a report-only listing of zombie epics. For each one, show the last
   land outcome and whether a land agent is live. Read `cli_rules` first if you add a
   CLI surface; a doctor check is another option. Store a one-time report as an artifact
   for the owner. Do not close or relaunch historical zombies, and do not add a blind
   plan-to-plan cascade close.
3. **Invariant.** Add it to the boot_recovery job or a sibling. Apply it only to epics
   whose last child closed after the invariant was enabled. That activation watermark
   stops the roughly 39 historical zombies from being mass-relaunched. Rate-limit it,
   act at most once per closure event behind a durable marker, honor capacity, and skip
   hijack custody.

> [!decision] auto_relaunch
>
> **Automatic recovery.** When the invariant holds, relaunch the parent epic's land
> agent through the epic launch path, and post one notice.

> [!decision] auto_relaunch = no
>
> **Notify only.** When the invariant holds, post one inbox notice naming the epic and
> the exact relaunch command. Nothing launches automatically.

**Done when:**

- the report lists the current zombies;
- an invariant fixture recovers exactly one newly zombied epic once;
- a pre-watermark zombie is ignored;
- a second pass does nothing.

## Landing

1. **Verify.** Re-read the epic and every phase, task, and nested bead, with all notes.
   Check each `TASK RESOLVED` claim against the source and the commits, and integrate
   any drift that landed while the epic ran.
2. **apollo, if needed.** If panel_pilot could not reach apollo, run its recipe against
   master on apollo now, in a scratch directory, never inside an apollo SASE workspace.
   Record the 20x serial and 3x `-n 8` results.
3. **Close the task beads normally,** each with its own commit or test evidence:
   sase-zo, sase-1ce, sase-1q3, sase-1qe, sase-1pw, sase-1qu, sase-172, sase-1i6,
   sase-1al, sase-1r4, sase-1jd, sase-1qj, and sase-12u.
4. **Triage follow-ups.** Run every `PROPOSED FOLLOW-UP:` through `/sase_new_task`.
5. **Hand off the zombie report.** Link the zombie_epics report in the epic's close
   note, and leave each historical zombie's relaunch-or-close decision to the owner.
6. **The watchdog cron.** If sase-1r2's crontab watchdog is still installed after
   sase-1r2 closes, propose removing it through `/sase_gate`.
7. **Close the epic** normally, with the actual follow-up triage in its note, then mark
   this plan file done. Do not use `--force` to advance a successful landing.

> [!decision] next_epic
>
> **Continue the backlog.**
>
> 1. Re-list every non-closed task bead with `sase bead list -t task -n 0`.
> 2. Screen the list against the live agents, both locally and through `ssh athena` or
>    `ssh apollo`, and against the in-progress epics.
> 3. Start from the report's ranks 11–16: sase-1r7 with sase-167, sase-1cg (after
>    sase-1qz.10 lands), sase-1db, sase-1kp, and sase-15o. Re-verify each one.
> 4. Propose an epic of the most important remaining beads through `/sase_plan`. Its
>    Landing section must carry this same continue-the-backlog rule. Proposing ends the
>    turn, so do it last.

> [!decision] next_epic = no
>
> **Stop here.** Stop after closing the epic, and list the report's ranks 11–16 in the
> close note as suggested next work.
