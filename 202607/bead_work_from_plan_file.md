---
tier: epic
status: done
title: Host-owned epic launch via sase bead work <plan_file>
goal: "Approving an epic plan no longer runs bead creation and agent launch inside the
  dying planner agent. `sase bead work` accepts an epic plan file, creates the epic and
  phase beads with their dependencies, and launches the epic with excellent CLI output;
  approval surfaces run that command host-side (a visible background task in the TUI),
  and the planner agent finishes cleanly instead of crashing.

  "
phases:
  - id: work-from-plan
    title: sase bead work <plan_file> creates beads and launches from a plan file
    depends_on: []
  - id: agent-loop
    title: Crash-proof epic approval in the agent exec loop
    depends_on:
      - work-from-plan
  - id: approval-surfaces
    title: Host-owned epic launch from TUI, CLI, and headless approvals
    depends_on:
      - agent-loop
  - id: docs-and-verify
    title: Documentation and end-to-end verification
    depends_on:
      - approval-surfaces
bead_id: sase-64
create_time: 2026-09-09 19:49:33
---

- **PROMPT:**
  [prompts/202607/bead_work_from_plan_file.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/bead_work_from_plan_file.md)
- **BEAD:**
  [sase-64](https://github.com/sase-org/sase--beads/blob/main/pages/sase-64/README.md)

# Plan: Host-owned epic launch via `sase bead work <plan_file>`

## Context and diagnosis

Epic plan approval (sase-61.5, commit `9ef9688c8`) currently materializes the epic
_inside the blocked planner agent's execution loop_: after the user approves, the agent
process commits the SDD files, creates the epic and phase beads, wires dependencies, and
launches all phase agents — all from `handle_accepted_plan` →
`_create_and_launch_approved_epic` (`src/sase/axe/run_agent_exec_plan_accept.py`).

This design has three concrete problems, all observed in the failed "8u" agent run that
created the sase-62 epic:

1. **Misleading agent crash.** When `sase plan propose` interrupts the planner, the
   workflow result is `None` (`src/sase/axe/run_agent_exec.py:337`). The epic branch of
   `handle_accepted_plan` returns the loop outcome `"completed"`
   (`run_agent_exec_plan_accept.py:443`) — the only killed-iteration path that does — so
   `finalize_loop` hits `assert result is not None`
   (`src/sase/axe/run_agent_exec_finalize.py:473`) and the agent dies with
   `AssertionError` _after_ the epic launched successfully. The epic worked; the agent
   shows FAILED.
2. **No user replication.** The create-beads-and-launch sequence exists only as
   in-process calls inside the agent runner. A user cannot re-run it, resume it after a
   partial failure, or preview it.
3. **No transparency.** The launch runs invisibly inside a dying agent process. There is
   no way to watch its progress from the TUI, and its output is lost.

There is also a latent race today: the epic approval choice has
`archive_side_effect=True`, so the host archives the plan into the SDD store
(`_archive_plan_for_approval` in `src/sase/plan_approval_actions.py`) concurrently with
the agent writing the same plan file via `write_sdd_files`. Two writers, one path.

## Design principles

- **One canonical command.** `sase bead work <plan_file>` is the single entry point that
  takes a validated epic plan from file to running agents: archive into the SDD store →
  create epic bead → create phase beads → wire dependencies → link `bead_id` into the
  plan → commit → launch waves. Every surface (TUI, CLI, headless transports,
  auto-approval) invokes this same command, so users can replicate, preview
  (`--dry-run`), and resume any step by re-running it.
- **Host owns the launch; the agent owns only its own artifacts.** For epics, the
  approving surface runs the command. The planner agent records metadata, writes its SDD
  spec (prompt) file only — never the plan file — and finishes cleanly.
- **Explicit ownership handshake.** The plan-approval response JSON gains an
  `epic_launch_owner: "host"` field. A surface writes it only when it has taken
  responsibility for running `sase bead work`. If the field is absent (auto approvals,
  older/third-party transports, or a host that failed to submit its task), the agent
  falls back to running the same command itself as a subprocess — never as in-process
  calls. No approval can be orphaned, and no approval can be double-launched.
- **Idempotent and resumable.** Re-running `sase bead work <plan_file>` after any
  failure resumes instead of duplicating: a plan already linked to a `bead_id` skips
  creation and proceeds to launch the existing epic (which already supports retrying
  remaining non-closed phases).

## The command: `sase bead work <target>`

The positional argument (today `id`, "Epic plan bead ID") becomes `target`:

- **Bead-ID mode** (unchanged): `sase bead work sase-62` — the argument neither ends in
  `.md`, contains a path separator, nor names an existing file.
- **Plan-file mode** (new): `sase bead work sdd/plans/202607/sase_plan_foo.md`.

Plan-file mode runs these stages, each rendered as a distinct step in the output:

1. **Validate** — `validate_plan_file(path, "epic")`; on failure render the full
   diagnostics with the existing human renderer
   (`src/sase/main/plan_validate_render.py`) and exit 1.
2. **Resolve store** — resolve the SDD store and bead store from cwd/primary workspace
   via the existing `resolve_beads_location`/`get_project` machinery
   (`src/sase/bead/cli_common.py`), auto-initializing beads if needed (parity with
   `ensure_beads_initialized` in the current agent-side path).
3. **Archive** — ensure the plan lives in the SDD store at
   `<plans-root>/{YYYYMM}/{name}.md`. Extract the core of `_archive_plan_for_approval`
   (prettier formatting, create-time frontmatter, tier normalization,
   `validate_plan_for_commit`) into a shared helper (e.g.
   `src/sase/sdd/plan_archive.py`) used by both this command and the tale/commit
   approval side effects. A plan already inside the plans root is a no-op here.
4. **Resume check** — if the archived plan already carries `bead_id` and that bead
   exists, skip creation and jump to launch ("resuming epic sase-63"). If the bead is
   missing, fail with a clear remedy (remove the stale `bead_id` or restore the bead
   store).
5. **Create + link** — `create_and_launch_epic_from_plan`
   (`src/sase/bead/epic_from_plan.py`) with a host-side plan ref and commit function.
   The plan-ref computation (in-tree `sdd/plans/...`, separate-repo
   `.sase/sdd/plans/...`, sidecar workspace-relative) moves from
   `src/sase/axe/run_agent_exec_plan_sdd.py` into a store-derived helper the command can
   call without agent context. For in-tree stores the commit stage must also push
   (phase-agent checkouts wipe uncommitted files), reusing the
   `commit_sdd_files_for_exec_plan` / `commit_sdd_store_files` paths.
6. **Launch** — the existing `launch_epic_bead_work`
   (`src/sase/bead/cli_work_handler.py`): wave plan, mark-ready, preclaim, agent launch,
   bead-state commit, with its existing rollback semantics.

Transactionality is unchanged: any failure before agents launch rolls back the created
beads and restores the plan file (existing `_rollback_epic_creation`); post-launch
commit failures preserve the live agents and surface an actionable error. In every
failure case the command prints the exact command to resume.

`--dry-run` in plan-file mode validates, resolves, and previews the beads and wave plan
that _would_ be created (waves computed from the frontmatter `phases`/ `depends_on`
graph) without mutating anything.

New/changed CLI surface (follow `memory/cli_rules.md`: alphabetized options, short
aliases, excellent colored `--help`):

- `target` positional: "Epic bead ID, or path to a validated epic plan file".
- `-j/--json`: machine-readable result (epic id, phase bead ids, archived plan path,
  launched agent names) for scripting and tests.
- Existing `-n/--dry-run`, `-P/--no-push`, `-y/--yes` apply to both modes.

### Output design

Rich-rendered staged output (degrades cleanly to plain text when piped — which is
exactly what the TUI Tasks tab shows):

```
Epic plan  sdd/plans/202607/sase_plan_workspace_gc.md
✓ Validated       tier: epic · 3 phases · 3 dependency edges
✓ Store           sidecar sase--plans · beads at .sase/sdd/beads
✓ Archived        sdd/plans/202607/sase_plan_workspace_gc.md (committed)
✓ Epic bead       sase-63 — Workspace GC rewrite
✓ Phase beads     sase-63.1 core · sase-63.2 cli · sase-63.3 smoke
✓ Dependencies    core → cli → smoke
✓ Plan linked     bead_id: sase-63 (committed, pushed)

  Wave 1  sase-63.1 core
  Wave 2  sase-63.2 cli
  Wave 3  sase-63.3 smoke
  Land    sase-63 (@epic_lander)

✓ Launched 4 agents for epic sase-63 (workspace 3)

Epic sase-63 is underway — track it on the Agents tab, or run:
  sase bead show sase-63
```

The final summary always contains a stable `Epic: <id>` line (plain, grep-able) as the
output contract for hosts that need the epic id from captured output.

## Ownership protocol and approval flow

Response JSON contract (`plan_response.json`): for `action: "epic"`, an approving
surface that has committed to running `sase bead work` includes
`"epic_launch_owner": "host"` in the _initial_ write (the file is write-once and
consumed quickly by the agent; no late mutation).

Agent behavior on an accepted epic (`handle_accepted_plan`):

- Record meta (`plan_approved`, `plan_action: epic`) and write + commit the SDD **spec
  file only** (a spec-only variant of `write_sdd_files`; the plan file now has exactly
  one owner — the command).
- If `epic_launch_owner == "host"`: finish with the new loop outcome `"epic_approved"`.
  No bead creation, no launch, no plan write.
- Otherwise (auto-approved epics via `%auto`, older/headless transports, or a host that
  could not claim ownership): run `sase bead work <plan_file> --yes` as a streamed
  subprocess. Success → outcome `"epic_approved"` with `epic_bead_id` meta; failure → a
  distinct non-crashing outcome (e.g. `"epic_launch_failed"`) with the subprocess output
  tail in the agent log and an error notification. `_create_and_launch_approved_epic`
  and its in-process orchestration are deleted.

Surfaces:

- **TUI modal** (`src/sase/ace/tui/actions/agents/_notification_modals.py`): on an epic
  choice, after tier validation, _first_ submit an `epic-launch` tracked task
  (`_submit_tracked_task`, `dedup_key` per plan file, display name like "Epic launch:
  workspace*gc"), _then* write the response with `epic_launch_owner: "host"`. If task
  submission fails, write the response without the owner field (agent fallback covers
  it). The task body uses `TaskReporter.run` (`src/sase/ace/tui/task_subprocess.py`) to
  stream `sase bead work <plan_file> --yes` with cwd set to the project's primary
  workspace (derived from the notification's `project_dir`, exactly as
  `_archive_plan_for_approval` does). Live output, phase markers, and kill support all
  come free from the Tasks tab machinery. On success, back-fill
  `epic_bead_id`/`epic_started_at`/`sdd_plan_path` into the planner agent's
  `agent_meta.json` (artifacts dir via `resolve_plan_agent_artifacts_dir`) and notify
  "Epic sase-63 launched — N phase agents + land agent"; on failure, error notification
  pointing at the Tasks tab and the resume command.
- **CLI** (`sase plan approve -k epic`, `src/sase/main/plan_approve_handler.py`):
  `execute_plan_approval_response` writes the response with the owner field, then the
  handler runs `sase bead work <plan_file> --yes` in the foreground so the user watches
  the full staged output directly.
- **Headless transports** (anything else calling `execute_plan_approval_response`, e.g.
  the Telegram plugin): `execute_plan_approval_response` gains an epic-launch mode
  parameter — foreground (CLI), detached spawn with logged output plus a completion
  notification (default for headless callers), or skip (a caller that owns the launch
  itself). A shared `src/sase/bead/epic_launch.py` provides the single argv builder and
  the detached spawner.
- `plan_approval_choices.py`: the epic record drops `archive_side_effect=True`
  (archiving is now stage 3 of the command) and its `consequence_text` becomes accurate,
  e.g. "Commit to sdd/plans (tier: epic); launch beads via `sase bead work` (background
  task)". Update the approval modal text accordingly.

## Crash fix details

- Epic approval returns `"epic_approved"` (never `"completed"`) from the killed
  iteration, so the planner finalizes like `plan_committed` does today: chat saved, done
  marker written with the real outcome, agent shows as done — not FAILED.
- Wire the new outcome everywhere `plan_committed` appears as an outcome:
  `_NON_HOLD_FAILURE_OUTCOMES` in `src/sase/axe/run_agent_runner_lifecycle.py`,
  done-marker/status derivation, and TUI status display (label it to match the existing
  "EPIC APPROVED" override from `approval_choice_status_label`).
- Defense in depth: replace the bare `assert result is not None` in `finalize_loop` with
  graceful handling (`response_text` only when a result exists) so no future
  outcome/result combination can crash an agent at finalize.

## Phase details

### Phase `work-from-plan` — the command

Everything under "The command" above: target disambiguation, the six stages, the shared
archive helper, store-derived plan refs, resume semantics, `--dry-run` preview,
`--json`, the staged Rich output with the `Epic: <id>` contract line, and
rollback/resume messaging. Respect the Rust core boundary: all bead mutations go through
the existing `BeadProject` facade; no new domain logic beyond orchestration and
rendering. Tests: target parsing, plan-file mode against a temporary SDD store (in-tree
and sidecar), resume/idempotency, missing-bead `bead_id`, rollback on launch failure,
dry-run purity, JSON output shape, and help text. This phase must not change approval or
agent-loop behavior.

### Phase `agent-loop` — crash-proof approval handling

The agent behavior under "Ownership protocol" plus "Crash fix details":
`PlanApprovalResult` learns `epic_launch_owner`; the epic branch of
`handle_accepted_plan` is reduced to meta + spec-only SDD write + (fallback) subprocess
launch; `_create_and_launch_approved_epic` is deleted; new outcomes are wired through
finalize, lifecycle, and TUI status display; the finalize assert is removed. After this
phase lands (before `approval-surfaces`), behavior is intentionally intermediate-safe:
no surface writes the owner field yet, so every interactive epic approval takes the
agent-side subprocess fallback — same user experience as today, but crash-proof and
running the canonical command. Tests: exec-loop unit tests for owner-present (no launch,
`epic_approved`), owner-absent/auto (subprocess invoked — mocked), subprocess failure
(graceful outcome, no exception), and a regression test reproducing the sase-62 crash
shape (killed iteration + epic approval + `result=None` finalizes without raising).

### Phase `approval-surfaces` — host-owned launches

Everything under "Surfaces" above: the TUI tracked task (submission-before-response
ordering, cwd resolution, live streaming, meta back-fill, notifications, dedup), the CLI
foreground launch, the `execute_plan_approval_response` epic-launch mode with the
detached spawner for headless callers, and the choice-record/consequence text updates.
Tests: TUI task submission and response-field ordering (existing task-action test
patterns), duplicate-approval dedup, CLI handler launch invocation, headless detached
spawn, and response JSON contents per surface.

### Phase `docs-and-verify` — docs and end-to-end verification

Update `docs/sdd.md` (epic approval flow, `bead_id` linking, the command as the
canonical entry point) and `docs/beads.md`. Verify the full loop end-to-end in a scratch
project: author a small epic plan, run `sase bead work <plan_file> --dry-run` and the
real launch, kill mid-way and resume, and exercise a TUI epic approval confirming the
Tasks tab shows the live launch and the planner agent ends as done (not FAILED). Fix
anything the sweep uncovers; run the full `just check`.

## Risks and mitigations

- **Two processes committing to one SDD store** (agent spec commit vs. host plan commit,
  or concurrent manual + TUI runs): bead-store and SDD commits already run concurrently
  across agents; the command must use the same robust commit paths
  (`commit_sdd_store_files`, `commit_successful_work_launch`) and the resume semantics
  make retries safe. TUI-side dedup prevents double tasks per plan.
- **Host dies between claiming ownership and launching**: the window is small, the
  failure is visible (task error or missing task), and recovery is one documented,
  idempotent command re-run.
- **Month rollover between archive and re-run**: resume identity is the archived path
  recorded in the plan's own `bead_id` frontmatter, not a recomputed YYYYMM path, so
  rollover only ever creates a fresh archive for a plan that was never linked.
- **Uniform runtimes**: the launch path is runtime-agnostic (`sase bead work` launches
  whatever models/providers the beads specify); no runtime-specific branching is
  introduced.
