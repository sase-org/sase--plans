---
tier: epic
title:
  "Task-bead backlog epic 2: agent isolation, poisoned slots, close guards, and
  toolchain drift"
goal: "The five next-costliest open task beads (sase-163, sase-1p9, sase-o7, sase-1cs,
  sase-vr) are fixed on master and closed. The land agent then proposes the next
  five-bead backlog epic under the same rules.

  "
decisions:
  claude_tools:
    ask:
      Should SASE Claude agents lose the vendor async and peer-discovery tools as well
      as cross-session inbound messages?
    choices:
      inbound_and_async:
        Also deny ListAgents, Workflow, Monitor, Cron* and RemoteTrigger; SendMessage
        stays
      inbound_only:
        Only refuse peer messages; async tools stay and keep losing results at turn end
    default: inbound_and_async
    why:
      Their results die at SASE's declaration teardown; /sase_monitor and /sase_run do
      those jobs
    answer: inbound_and_async
  slot_refusal:
    ask:
      Should a workspace whose nested-preservation refusal failed a launch stop
      receiving new launches until it is clean?
    choices:
      quarantine:
        New launches skip the slot until its refusal clears; one notification names the
        repo
      notify_only:
        The slot stays assignable with one notification; later launches there still fail
    default: quarantine
    why:
      Fail-closed preservation is right, but failing every queued agent on it is pure
      waste
    answer: quarantine
  toolchain_source:
    ask:
      Should the dev extra pin Textual, ruff and mypy exactly, so CI and every workspace
      venv resolve the same versions?
    choices:
      pin_extras:
        Pin the three tools exactly in the dev extra and add a test that keeps those
        pins equal to uv.lock
      lock_install:
        Leave the extra floating and install the dev toolchain from uv.lock in just
        install and CI
    default: pin_extras
    why:
      One pyproject edit fixes both install paths, and a guard test keeps the lock in
      step
    answer: pin_extras
phases:
  - id: claude-isolation
    title: Refuse cross-session messages into SASE Claude agents (sase-163)
    depends_on: []
    size: small
    description:
      "claude-isolation: make ClaudeCodeProvider launches refuse cross-session inbound
      messages and apply the claude_tools decision to the vendor async tools, with a
      provider-args regression test."
  - id: slot-refusal
    title: Stop a preservation refusal from poisoning its workspace slot (sase-1p9)
    depends_on: []
    size: medium
    description:
      "slot-refusal: remember launch-time nested-preservation refusals per workspace,
      apply the slot_refusal decision, and raise one actionable notification instead of
      failing every later launch there."
  - id: close-guard
    title: Make the epic close guard see upstream --epic-symbol rows (sase-o7)
    depends_on: []
    size: medium
    description:
      "close-guard: have sase bead close and sase bead epic-symbols read the fetched
      default branch's Justfile as well as the local one, so sibling-phase whitelist
      rows cannot escape the close."
  - id: dev-toolchain
    title:
      Make the dev toolchain deterministic across CI and workspaces (sase-1cs, sase-vr)
    depends_on: []
    size: medium
    description:
      "dev-toolchain: apply the toolchain_source decision so Textual, ruff and mypy
      resolve to one declared version everywhere, refresh uv.lock, and fix any lint
      findings the chosen versions report."
proposed_by: bbugyi200.athena.sase-1pm.land
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-29 10:44:06
status: wip
---

- **PROMPT:**
  [prompts/202610/task_backlog_epic_2.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/task_backlog_epic_2.md)
- **PARENT:**
  [202610/task_backlog_epic_1.md](https://github.com/sase-org/sase--plans/blob/main/202610/task_backlog_epic_1.md)

# Task-Bead Backlog Epic 2: Agent Isolation, Poisoned Slots, Close Guards, and Toolchain Drift

## Context

This epic continues the backlog chain started by epic `sase-1pm`
(`202610/task_backlog_epic_1.md`). Its land agent landed all five epic-1 beads
(sase-1nh, sase-1ll, sase-1p5, sase-1ic, sase-1cb) at master `cddea47cc6` on 2026-10-29.
Then it re-listed the 384 non-closed task beads and re-verified epic 1's seed ranking.

- **Closed none.** No seed lead marked "likely closable" could be proved:
  - sase-1nr only flaked in the broad lane. Nothing has recurred since `79fa96dfc7`, but
    nothing proves the fix either.
  - sase-1ms and the hint/prompt-render cluster (sase-1ia, sase-1l3, sase-1hy, sase-1i9)
    need apollo.
  - sase-v4 asks for a creator nobody has identified.
- **Not owned by an active epic:** none of the five beads below. The contract-manifest
  failure is owned by `sase-1pi` and `sase-1pj.2.1`. The monitor-start supervisor tests
  are owned by `sase-1pg`.

This epic works the next five beads by fleet-wide cost:

| Bead               | Problem                                                                                                                                                        | Phase              |
| ------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------ |
| sase-163           | Security: any bypass-mode Claude session can inject unaudited messages into every running SASE Claude agent, and agents expose async tools that cannot finish  | `claude-isolation` |
| sase-1p9           | One unpreservable nested repo poisons a workspace slot: every later launch there fails about 5 s after admission, after waiting in the queue (53 min observed) | `slot-refusal`     |
| sase-o7 (+24)      | Epic-symbol whitelist rows outlive their epic, so an unrelated agent's `just check` discovers them; the close guard reads only the local Justfile              | `close-guard`      |
| sase-1cs + sase-vr | `.[dev]` floats Textual, ruff and mypy: CI-only reds blocked the 0.18.1 release for about 7 hours, and lint verdicts depend on install date                    | `dev-toolchain`    |

The four phases touch disjoint files and have no ordering constraints, so they may run
in parallel. Each phase leaves a `TASK RESOLVED: <task-id> — <evidence>` note on its own
phase bead. Phase workers do not close task beads and do not create beads. Follow-ups go
in `PROPOSED FOLLOW-UP:` notes on the phase bead.

## Phase `claude-isolation`: refuse cross-session messages into SASE Claude agents (sase-163)

Read sase-163 in full, including both correction notes. The provider class is
`ClaudeCodeProvider` in `src/sase/llm_provider/claude.py`. It launches every agent as
`claude -p ... --dangerously-skip-permissions --disallowedTools ScheduleWakeup`. The
only deny is `ScheduleWakeup`, and no `crossSessionInbound` setting is passed. The
problem was re-confirmed live on 2026-10-29: a SASE-launched Claude agent had
`CLAUDE_CODE_MESSAGING_SOCKET` set, and its tool list exposed `ListAgents`,
`SendMessage`, `CronCreate`/`CronDelete`, `Monitor`, `Workflow` and `RemoteTrigger`.

- Refuse cross-session inbound messages on every SASE Claude launch, for example by
  passing `--settings '{"crossSessionInbound":"refuse"}'`. Check how that interacts with
  any `--settings` value SASE already passes, and merge rather than clobber. Per the
  critique note, `refuse` also refuses the session's own hook and Bash socket posts.
  Confirm SASE does not rely on those, by grepping for the messaging socket and the hook
  paths. If it does, report that as a `PROPOSED FOLLOW-UP` and do not break it.
- Keep `SendMessage` allowed: denying it also removes in-session subagent messaging.
- Under either decision, record peer-origin messages as an audit tripwire if the
  provider's stream parser can see them (`origin.kind == "peer"` on stream-json output),
  so any message that still gets through is visible in the run artifacts.

> [!decision] claude_tools = inbound_and_async
>
> Also add `ListAgents`, `Workflow`, `Monitor`, `CronCreate`, `CronDelete` and
> `RemoteTrigger` to `--disallowedTools`. The dedicated switches
> `CLAUDE_CODE_DISABLE_WORKFLOWS=1` and `CLAUDE_CODE_DISABLE_CRON=1` are acceptable
> where they cover the same tools. Keep the existing `ScheduleWakeup` deny and
> `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1`.

> [!decision] claude_tools = inbound_only
>
> Change only the inbound setting. Leave the tool list as it is.

Acceptance: a provider-args unit test asserts the inbound refusal and, under
`inbound_and_async`, the denied tool set (including that `SendMessage` is not denied).
The existing Claude provider tests pass.

## Phase `slot-refusal`: stop a preservation refusal from poisoning its workspace slot (sase-1p9)

Read sase-1p9 in full. Launch-time nested preservation
(`src/sase/axe/runner_workspace_protection.py:prepare_launch_workspace_repos` →
`preserve_nested_repositories`) refuses with a `PreservationError` when a dirty nested
repo cannot be snapshotted. That raises
`WorkspacePreparationError(step="sidecar-protection", reclone_eligible=False)`. The
runner then releases the workspace as an ordinary setup failure
(`_is_workspace_preparation_failure` in `src/sase/axe/run_agent_runner_launch.py`). The
next launch is assigned the same slot and fails the same way.

Keep fail-closed preservation exactly as it is. The defect is that nothing remembers the
refusal. The work:

- When this refusal happens, durably record it against the workspace slot. Record the
  project, the workspace number, the nested repo path, a bounded list of dirty paths,
  the refusal reason and a timestamp. Workspace assignment is shared backend behavior.
  If the allocator's choice lives in the Rust core (`sase_core`, linked repo
  `sase-core`, opened with `/sase_repo`), put the skip rule there and keep Python as a
  thin caller. Follow `docs/rust_backend.md`, including moving the
  `sase-core-revision.txt` pin.
- Clear the record automatically when the slot no longer refuses: the nested repo is
  clean or gone, or a later preservation succeeds. Also give it an explicit manual clear
  path through an existing workspace command. Read the `cli_rules.md` reference memory
  before adding any CLI option.
- Raise exactly one actionable notification per refusal record, not one per failed
  launch. It names the slot, the nested repo, its dirty paths and the clear path.

> [!decision] slot_refusal = quarantine
>
> While a refusal record stands, new launches skip that workspace slot. They take
> another free slot, or wait as if the slot were busy, but they are never failed because
> of it. A launch that explicitly targets that workspace still fails fast with the
> actionable message.

> [!decision] slot_refusal = notify_only
>
> Keep assigning the slot. Only add the durable record and the single deduplicated
> notification, and make each later failure's message point at it.

Acceptance: reproduce the bead's scenario in a test. Leave a dirty nested repo that
`snapshot_git_repo` refuses (for example, monkeypatch `SNAPSHOT_MAX_BYTES`), then launch
two agents in sequence. The first fails with the actionable message. Under `quarantine`,
the second is not assigned the poisoned slot. There is exactly one notification, and the
record clears once the nested repo is clean.

## Phase `close-guard`: make the epic close guard see upstream `--epic-symbol` rows (sase-o7)

Read sase-o7 in full, including its recent notes. `sase bead close` already refuses
while `--epic-symbol` rows keyed to the closing bead or its descendants remain. The same
is true of `sase bead epic-symbols`. Both are wired through
`src/sase/bead/epic_symbols.py:raise_if_leftover_epic_symbols` and
`load_epic_symbol_entries`. But `load_epic_symbol_entries` reads only the Justfile
nearest the caller's directory. A row that a sibling phase landed on the default branch
after the closing agent's checkout escapes the guard. So does a row in a checkout the
closer never synced. Each escape still turns some unrelated agent's `just check` red.
Recurrences were recorded on 2026-10-27 and 2026-10-28.

- Have both commands also read the Justfile at the project's default branch
  (`origin/<default>` from a bounded fetch, or the freshest remote-tracking ref when
  offline). Union those rows with the local ones, and report which source each row came
  from.
- A fetch or read failure must not wedge closes. Warn loudly and fall back to the local
  Justfile.
- Keep the per-symbol resolution as human judgment: do not auto-delete rows. The goal is
  that the close cannot succeed while rows keyed to the closing epic remain on either
  side.
- Check the other close paths: `--force`, `-p` phase-only closes, and batch
  `sase bead update ... --status closed`. Confirm they reach the same guard, or that a
  documented reason excuses them.

Acceptance: an integration test in a temporary git repo with a remote. The remote's
default-branch Justfile carries a row keyed to the epic, and the local Justfile does
not. `sase bead close <epic>` must refuse and name the row, and `sase bead epic-symbols`
must list it. A fetch failure falls back to the local Justfile with a warning. Existing
epic-symbol tests pass.

## Phase `dev-toolchain`: make the dev toolchain deterministic (sase-1cs, sase-vr)

Read both beads in full. Today `pyproject.toml`'s `dev` extra lists unconstrained `ruff`
and `mypy`. Textual comes only through the base `textual[syntax]>=0.45.0` (the `visual`
extra pins `textual==8.0.1`). `just install` and CI install `.[dev]` with
`uv pip install ... -e ".[dev]"`, so each install resolves whatever the index publishes
that day. On 2026-10-29 the drift was visible inside one day:

| Source                      | ruff    | mypy   | textual |
| --------------------------- | ------- | ------ | ------- |
| `uv.lock`                   | 0.15.4  | 1.19.1 | 8.0.1   |
| workspace `sase_20` venv    | 0.16.6  | 2.4.0  | 8.0.1   |
| workspace `sase_14` ToolRun | 0.16.10 | 2.4.0  | —       |
| CI fresh resolve (sase-1cs) | —       | —      | 8.2.8   |

> [!decision] toolchain_source = pin_extras
>
> Pin `ruff`, `mypy` and Textual exactly in the `dev` extra. Choose the versions
> deliberately: the newest ones the current tree passes `just lint` with, keeping
> Textual equal to the `visual` extra's renderer pin unless the goldens are regenerated.
> Refresh `uv.lock` to match. Add a fast contract test that fails when a `dev` pin and
> the `uv.lock` version disagree.

> [!decision] toolchain_source = lock_install
>
> Keep the extra unconstrained. Make `just install`, `just install-venv` and every CI
> install of `.[dev]` constrain the dev toolchain to `uv.lock` (for example with
> `uv export --frozen` constraints). Refresh the lock to the chosen versions, and add a
> test that the install recipes and CI pass those constraints.

Under either choice:

- Fix every new ruff or mypy finding the chosen versions report. Do not lower rule
  coverage to make them pass.
- Re-run the tests sase-1cs names under the chosen Textual
  (`tests/ace/tui/command_line/test_chrome_layout.py`,
  `tests/ace/tui/test_custom_gate_modal_sizing.py`).
- Note in the phase's `TASK RESOLVED` notes that existing workspace venvs pick up the
  pins only on their next `just install-venv`. Never run a global install
  (`just install`/`just install-dev`) from an agent.

Acceptance: a fresh `.[dev]` resolve, a lock-faithful venv, and the workspace venv after
`just install-venv` all report the same ruff, mypy and Textual versions. `just lint`
passes under them, and the new guard test passes.

## Landing

The land agent verifies that each phase landed and then closes the resolved task beads.

- For each phase bead's `TASK RESOLVED` note: confirm the fix is on master and the named
  tests pass at the landing SHA. Then run
  `sase bead close <task-id> --note "<commit SHAs and verification>"`. If a phase did
  not resolve its task bead, leave that bead open with a note explaining why.
- Triage the phases' `PROPOSED FOLLOW-UP` notes through `/sase_new_task` as usual.
- Carry-forward from epic 1: confirm `status: done` in the frontmatter of the plans
  `202610/task_backlog_epic_1.md`, `202610/rebased_stitch_gate.md` and
  `202610/finish_rebased_stitch_gate.md`. Set it where it is still `wip`. Epic 1's land
  agent set it in an uncommitted tree, and then its plan proposal ended the turn through
  the legacy handoff, which skips host finalization.

### Mandatory: continue the backlog chain

After closing this epic's beads, the land agent MUST author and propose the next backlog
epic. It must follow these exact rules, and must pass them on verbatim to the next
epic's land agent:

1. Re-list every non-closed task bead (`sase bead list -t task -n 0`). Close only beads
   you can verify are duplicates (resolution `superseded`, naming the canonical bead,
   with a note on the canonical) or obsolete (resolution `done`, citing the fixing
   commit or the grep that proves absence, or a passing focused test). Never close a
   bead an active epic owns. Never close a snoozed bead early. Beads that are merely
   dormant stay open.
2. Pick the 5 most important remaining task beads. Rank them by fleet-wide cost first
   (this means: breaks every agent's check or CI, breaks or wastes every landing,
   security or data-loss risk, heavy +1 corroboration), then by cost to fix.
3. Write an epic plan with at most 5 phases and at most 5 task beads. One phase may work
   several beads. Use plan decisions sparingly, but always give the owner a way to opt
   out of feature work and memory changes that might not be wanted.
4. That epic's Landing section must carry this same "continue the backlog chain" rule,
   until no non-closed task beads remain.
5. Propose it with `/sase_plan`. Proposing ends the land agent's turn, so do it last.

Seed ranking for the next epic. These are triage leads from 2026-10-29, not commitments,
and each must be re-verified before it is planned:

- **Every agent's check on athena:** sase-1pg (+3). A deterministic monitor-start
  supervisor env-scrub failure, plus the ack-marker kill test. It is labeled KNOWN in
  every full-lane ToolRun.
- **Top flakes by gate impact:**
  - sase-120 (+17, 180s timeouts)
  - sase-13a (+14, zsh PTY)
  - sase-1gz (+11, sase-core federation socket)
  - sase-xo (+7, recurred after its "fix")
  - sase-1fi (49 ToolRun failures from 43 agents)
- **Every sidecar user:** sase-zo (+10). `sase repo open` rejects configured sidecars
  when inventory uses the project display name.
- **Safety/data:**
  - sase-1kp: `beads.db` is both the Rust lock file and Python's WAL mirror (needs a
    design decision).
  - sase-1nm: sudo re-approval reruns privileged commands.
  - sase-1p0: athena root-NVMe writes of 2–4 TB/day.
- **Stale or needs a decision:**
  - sase-j0: `in_progress` since August with no live holder; check-full test-cost
    budgets are still red.
  - sase-1ot: owner rejected the purge gate on 2026-10-28; candidate to close as
    won't-do with owner consent.
- **Likely closable on re-verification:**
  - sase-1nr: no recurrence since `79fa96dfc7` pinned the Files-pane flag in its test
    module.
  - sase-1ms: apollo-only stale skill.
  - sase-v4: shards gone, creator unidentified.
  - The hint/prompt-render cluster (sase-1ia, sase-1l3, sase-1hy, sase-1i9) passes on
    athena but reproduced on apollo 2026-10-27, so re-check it on apollo.
