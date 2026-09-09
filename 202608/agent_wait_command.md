---
tier: epic
status: done
title: sase agent wait
goal: "`sase agent wait` blocks until named agents (or every agent running right now)
  reach a terminal state, reports each outcome with output good enough to act on, and
  exits non-zero when any target failed, is blocked on a human, or timed out.

  "
phases:
  - id: engine
    title: Wait target resolution and settle engine
    depends_on: []
    size: medium
    description: "engine: build the presentation-neutral wait engine — resolve names to
      wait units, classify each unit per tick from one artifact snapshot, and decide
      when the wait settles and with which outcome.

      "
  - id: cli
    title: sase agent wait command and exit-code contract
    depends_on:
      - engine
    size: medium
    description: "cli: register the subcommand, wire target selection and
      self-exclusion, implement the exit-code contract, and ship the non-TTY, JSON, and
      quiet output modes so the command is fully usable.

      "
  - id: live
    title: Live TTY display and settle summary
    depends_on:
      - cli
    size: medium
    description: 'live: add the refreshing TTY panel, the per-target "why it is not
      done" column, terminal-blocker warnings, the final settle summary with inspect
      pointers, and signal-safe teardown.

      '
  - id: docs
    title: Documentation, help polish, and integrated verification
    depends_on:
      - live
    size: small
    description:
      "docs: document the command and the monitor gate idiom, polish help text and
      examples, and verify the whole feature end to end with `just check-full`."
proposed_by: bbugyi200.athena.0bd
bead_id: sase-s8
create_time: 2026-09-09 19:49:48
---

- **PROMPT:**
  [prompts/202608/agent_wait_command.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/agent_wait_command.md)
- **BEAD:**
  [sase-s8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-s8/README.md)

# Plan: `sase agent wait`

## Goal

Add `sase agent wait`, a gate command that blocks until the agents you name (or every
agent running right now) reach a terminal state, then exits with a status code that says
what happened. It is the missing primitive for "run this only after those agents
finish".

```bash
sase agent wait sase-s7.2 && just check-full
sase agent wait -a -t 2h
sase monitor start -s WAITING -S WAITED -n 'agents finished; land the epic' -- sase agent wait -a
```

## Design Principles

Three properties drive every decision below.

**Intuitive** — the command answers exactly one question ("are they done yet?") and
answers it the same way the rest of SASE already answers it. Name resolution, completion
semantics, status labels, glyphs, and colors are reused from the existing `%wait` /
`sase agent list` machinery rather than reinvented, so `sase agent wait X` and a
`%wait X` directive can never disagree.

**Reliable** — a gate command that hangs is worse than no gate command. Every state in
which a target will not progress on its own is detected and terminates the wait with a
distinct, actionable exit code. The two ways this command could realistically hang
forever — waiting on itself, and waiting on an agent that has already stopped without
writing a completion marker — are both closed by design, not by timeout.

**Beautiful** — a live panel on a TTY that reads like `sase agent list`, a line-oriented
transition log when the output is captured (which is the common case, since agents run
this under `sase monitor`), and a settle summary that tells you what failed, why, and
the exact command to inspect it.

## Command Surface

```
sase agent wait [-h] [-a] [-i DURATION] [-j] [-p PROJECT] [-q] [-t DURATION]
                [-w] [NAME ...]
```

Options are alphabetical, every long option has a short alias, and none is required —
per `sase/memory/cli_rules.md`. The value the command needs to run is the `NAME`
positional; `-a/--all` is the modifier that makes it unnecessary.

| Option                    | Meaning                                                                         |
| ------------------------- | ------------------------------------------------------------------------------- |
| `-a, --all`               | Wait for every agent running when the command starts. Rejects `NAME` arguments. |
| `-i, --interval DURATION` | Fixed poll interval. Default: adaptive (see below).                             |
| `-j, --json`              | Emit one JSON envelope on settle; suppresses progress output.                   |
| `-p, --project NAME`      | Limit `--all` targets, and scope name resolution, to one project.               |
| `-q, --quiet`             | Suppress progress; print only the final summary line.                           |
| `-t, --timeout DURATION`  | Give up after DURATION. Default: no timeout.                                    |
| `-w, --wait-blocked`      | Keep waiting through pauses that need a human instead of exiting `3`.           |

`DURATION` reuses the syntax `sase monitor start -t` already documents: bare seconds or
`90s` / `45m` / `2h`. Share the parser rather than copying it (see `_parse_timeout` in
`src/sase/main/monitor_handler.py`); lifting it into a small shared helper is preferred
over a second implementation.

### Targets

A positional `NAME` is resolved in the same reference space `%wait` uses, in the same
precedence order: clan, then agent family, then workflow, then exact agent name. Waiting
on a family root therefore waits for the whole family including successors launched
after the wait began — which is what makes the command correct for piped and monitored
agents.

`-a/--all` snapshots the set of agents that are live at start and waits for exactly that
set. It deliberately does **not** absorb agents launched later: otherwise a chain of
agents that keep launching successors would make the command non-terminating. Each
snapshotted row is keyed to its clan or family when it has one, so a successor of a
snapshotted agent still counts as that unit continuing.

Targets are pinned at start. A clan/family/workflow unit re-evaluates membership each
tick (late members count). A single-agent unit pins its resolved `artifact_dir` so a
later, same-named agent cannot silently re-target the wait; a retry of the pinned
artifact follows the retry chain forward, because that is the same unit of work
continuing.

**Not supported in v1**, each rejected with a clear message rather than a hang:

- Tribe references (`@tribe`). `resolve_wait_dependency` already refuses them because a
  tribe wait needs the waiting artifact's launch cutoff, which a CLI invocation does not
  have. Error message points at `%wait`.
- Bead waits (`%wait` supports `wait_for_beads`). Out of scope.
- Running a follow-up command directly. That is `sase monitor start`'s job, and the docs
  phase shows the composition instead.

### Self-exclusion

An agent that runs `sase agent wait -a` is itself a running agent. Without exclusion,
the command waits for itself and never returns — the single most likely way this feature
would be used wrong.

Resolve the calling agent from `SASE_ARTIFACTS_DIR`, read its `agent_meta.json`, and
exclude that artifact plus the rest of its agent family (its monitor members and family
successors) from `--all` targets. This matters concretely because the expected usage is
`sase monitor start -- sase agent wait -a`: the monitor member running the wait, and the
starter that launched it, are both live agent rows.

Naming yourself (or a member of your own family) explicitly is refused with exit `2` and
an explanatory message, not accepted and then hung on.

## The Reliability Problem This Must Solve

`sase agent list` is not a sufficient source of truth for waiting, and building on it
naively would produce a command that hangs.

`list_running_agents()` filters to records whose process is alive (`_record_is_live`),
and `list_all_agents()` adds only records that have a `done.json` marker
(`_done_info_from_record` returns `None` without one). An agent whose process has exited
**without** writing `done.json` therefore appears in neither list — it silently
vanishes. That is not a rare corner: an agent that submits a plan for review exits with
`plan_path.json` written and no `done.json` (verified against live artifacts under
`~/.sase/projects/*/artifacts/ace-run/`), and a crashed or SIGKILLed agent looks the
same.

So the engine classifies targets directly from the artifact snapshot
(`scan_agent_artifacts`, which is Rust-backed and does no in-process caching, so each
poll tick sees fresh state). Every marker it needs is already on
`AgentArtifactRecordWire`: `agent_meta` (pid, `stopped_at`, family, clan, monitor
fields), `done`, `waiting`, `pending_question`, `plan_path`, `has_done_marker`.

## Phase `engine`: Wait Target Resolution and Settle Engine

New package `src/sase/agent/wait_watch/` — presentation-neutral, no `rich`, no
`argparse`, no `sys.exit`. Suggested split, following the module sizes already used
under `src/sase/agent/`:

- `_types.py` — `WaitTarget`, `WaitTargetState`, `WaitTick`, `WaitSettlement`.
- `_resolve.py` — name → wait unit resolution, `--all` snapshot, self-exclusion.
- `_classify.py` — one snapshot record set → per-target state.
- `_watch.py` — the tick loop as a generator, with injected snapshot provider, clock,
  and sleep so tests never wait on wall time.

### Per-target states

Classified from markers, in this order:

| State                  | Detection                                                          | Wait continues? |
| ---------------------- | ------------------------------------------------------------------ | --------------- |
| `succeeded`            | done marker, outcome in `WAIT_SUCCESS_OUTCOMES`                    | no              |
| `failed`               | done marker, outcome in `FAILURE_OUTCOMES`                         | no              |
| `terminal_other`       | done marker, known outcome that is neither (e.g. `plan_rejected`)  | no              |
| `queued`               | `waiting` marker with `slot_requested_at`, process live            | yes             |
| `waiting`              | `waiting` marker with `wait_for` / `wait_for_beads`                | yes             |
| `needs_input`          | `pending_question` marker, or process dead with a pending question | only with `-w`  |
| `needs_review`         | process dead, no done marker, `plan_path` marker present           | only with `-w`  |
| `stalled`              | process dead, no done marker, no pending-input marker              | only with `-w`  |
| `running` / `starting` | process live, no done marker                                       | yes             |

Reuse `WAIT_SUCCESS_OUTCOMES` / `FAILURE_OUTCOMES` / `KNOWN_DONE_OUTCOMES` from
`sase.core.dismissed_agent_completion`, and the liveness check from
`sase.agent.names.is_process_alive`, so the classification cannot drift from what the
wait-checks chop and the launch admission gate already believe.

A composite unit (clan, family, workflow) is `succeeded` only when every member is
terminal and every member succeeded; it is `failed` as soon as any member failed; it
stays pending while any member is live. This mirrors `is_agent_family_complete` and
`is_agent_clan_complete`; prefer calling those helpers over re-deriving the aggregation.

### Blocked states and the default

`needs_input`, `needs_review`, and `stalled` are the states where the target has stopped
and will not resume without a human. **By default the wait stops on them** and reports
`blocked` (exit `3`). The alternative — keep waiting — turns a question you have not
noticed into an hour of silence and then an opaque monitor timeout, which is exactly the
failure mode that makes a gate command untrustworthy.

`-w/--wait-blocked` opts into the other behavior for the case where you know you will
answer the question or approve the plan and want the gate to survive it.

### Settle policy and precedence

The wait settles when every target is in a stop state (given the `-w` setting), or the
timeout expires, or a signal arrives. When more than one outcome applies, the most
actionable wins: `failed` > `blocked` > `timeout`. A settlement carries the per-target
final states, elapsed seconds, and the derived exit code.

### Polling

One snapshot per tick, shared by every target — cost is independent of target count.
Adaptive interval by default: `1s` for the first `30s`, `2s` up to `5m`, then `5s`; `-i`
pins a fixed interval. The generator yields a `WaitTick` per poll so renderers can react
to transitions without owning the loop.

### Tests

Synthetic artifact trees (follow the existing helpers in
`tests/_agent_list_entries_helpers.py` and `tests/_agent_names_fixtures.py`) with an
injected clock, covering: success, failure, family with a late successor, clan
aggregation, a target already terminal at start (settles immediately, no sleep),
`plan_path` with no done marker classified `needs_review`, dead process with no markers
classified `stalled`, `-w` continuing through a blocked state that later completes,
timeout, and the outcome-precedence matrix.

## Phase `cli`: Command and Exit-Code Contract

- `src/sase/main/parser_agent.py` — register the `wait` subparser with the option table
  above, a description, and an epilog carrying the three examples from the top of this
  plan. Subcommand help is sorted centrally by `_sort_subcommand_help`, so registration
  order does not need to be alphabetical.
- `src/sase/main/agent_handler.py` — dispatch `wait` to
  `sase.agents.cli_wait.handle_agents_wait`, returning its exit code via `sys.exit`,
  matching how `restart` and `prompts` are dispatched.
- `src/sase/agents/cli_wait.py` — argument validation, target selection, self-exclusion,
  signal handling, renderer selection, exit code.
- `src/sase/agents/_wait_render_plain.py` — non-TTY and quiet output.
- `src/sase/agents/_wait_json.py` — the JSON envelope.

### Exit codes

| Code          | Meaning                                                                                                                                                             |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `0`           | Every target finished successfully.                                                                                                                                 |
| `1`           | At least one target finished unsuccessfully (`failed`, `killed`, `stopped`, `epic_launch_failed`, or another non-success terminal outcome such as `plan_rejected`). |
| `2`           | Usage error: no targets given, `-a` combined with `NAME`, an unresolvable name, a tribe reference, or naming your own family.                                       |
| `3`           | At least one target is blocked needing a human, without `-w/--wait-blocked`.                                                                                        |
| `4`           | The `-t/--timeout` expired with targets unfinished.                                                                                                                 |
| `130` / `143` | Interrupted by `SIGINT` / `SIGTERM`.                                                                                                                                |

`2` matches argparse's own usage-error code and the refusal code `sase agent restart`
already uses. Document the precedence (`1` > `3` > `4`) in help and in `docs/cli.md`.

### Stream discipline

Progress goes to **stderr**; the settle summary and the `-j` envelope go to **stdout**.
That keeps both `sase agent wait X && next-command` and `payload=$(sase agent wait -aj)`
correct.

### Unresolvable names fail fast

An unknown name exits `2` immediately with `difflib.get_close_matches` suggestions drawn
from live and recently-completed agent names — the same did-you-mean pattern used in
`src/sase/agent_clis/operations.py`. A typo must never become an infinite wait.
Symmetrically, a name that resolves to an already-finished agent settles immediately
with that recorded outcome rather than erroring or hanging.

### Non-TTY output

No `Live`, no spinner. One line per state transition, plus a heartbeat no more often
than every 60s, plus a settle summary:

```
waiting on 3 agents: 0bd, sase-s7.2, sase-s7.3
[+00:02:01] sase-s7.2--mon-1  TESTED   (2m1s)
[+00:11:52] 0bd               DONE     (11m52s)
[+00:12:04] sase-s7.3         FAILED   (3m18s)  provider error: context window exceeded
settled: 2 succeeded, 1 failed, 0 blocked in 12m4s (exit 1)
```

This is what `sase monitor` captures and what a follow-up agent reads, so it is
greppable and stable rather than decorative.

### JSON envelope

One object on settle, stable schema, shaped like the existing `sase agent list -j` rows
so the two are easy to join:

```json
{
  "settled": "failed",
  "exit_code": 1,
  "waited_seconds": 724.1,
  "timed_out": false,
  "targets": [
    {
      "name": "sase-s7.3",
      "kind": "agent",
      "project": "sase",
      "state": "failed",
      "status": "FAILED",
      "outcome": "failed",
      "duration_seconds": 198,
      "artifacts_dir": "/home/…/20260823110206",
      "error": "provider error: context window exceeded",
      "blocked_reason": null
    }
  ]
}
```

`kind` is one of `agent`, `family`, `clan`, `workflow`. Composite units report their
members under a `members` list with the same row shape.

### Tests

Exit code per scenario; `-a` self-exclusion (including the monitor-member case); `-a`
with zero eligible targets exiting `0` with a "nothing to wait for" message; `-a`
combined with `NAME` rejected; unknown name suggestions; tribe reference refusal;
stdout/stderr split; JSON schema; `-q` output; already-finished target settling without
a poll.

## Phase `live`: TTY Display and Settle Summary

`src/sase/agents/_wait_render_live.py`, selected only when stdout is a TTY and neither
`-j` nor `-q` is given.

A `rich.Live` panel refreshed each tick, deliberately shaped like `sase agent list` so
the two read as one surface — reuse `agent_status_text` from
`src/sase/agents/status_style.py` and `AGENT_STATUS_BUCKET_GLYPHS` from
`src/sase/agent/status_buckets.py` for glyphs and colors, and `project_display_name_for`
for the project column (never the ProjectSpec key):

```
╭─ Waiting on 3 agents · 04:12 elapsed ─────────────────────────────────╮
│ ▶ 0bd         sase      12  opus    RUNNING   4m12s  #gh:sase add …   │
│ ⏳ sase-s7.2   sase      15  grok    WAITING   4m12s  waits on 0bd     │
│ … sase-s7.3   bob-cli    4  sonnet  QUEUED    4m10s  slot 2 of 3      │
│ ✓ s7.2--mon-1 sase      15  grok    TESTED    2m01s  exit 0           │
╰─ 3 pending · 1 done · 0 failed ───────────────────────────────────────╯
```

The last column is what makes this practically useful rather than a spinner: it explains
_why_ a target is not done yet, from data `sase.integrations.agent_list_entries` already
projects — `wait.wait_for` for a `WAITING` target, `wait.runner_slot_queue_position` /
`wait.runner_slots_in_use` for a `QUEUED` one, `monitor_command` for a monitor member,
the prompt snippet otherwise. Unfinished rows sort above finished ones.

### Terminal-blocker warning

When a pending target is itself `WAITING` on a dependency that has already reached a
terminal failure, that target can never start. Detect it with the same resolution the
wait-checks chop uses (`dependency_resolution_status` plus the candidate's `is_failed`)
and surface it inline:

```
⚠ sase-s7.3 waits on sase-s7.1, which FAILED — it will not start
```

This is a warning, not a state change: the wait keeps its declared semantics, but it
stops being a silent hang.

### Settle summary

The `Live` is torn down and replaced by a summary panel that answers "what now?":

```
╭─ Waited 12m04s · 3 agents ────────────────────────────────────────────╮
│ ✓ 0bd          DONE      11m52s   sase · ws12                         │
│ ✓ sase-s7.2    DONE      12m01s   sase · ws15                         │
│ ✗ sase-s7.3    FAILED     3m18s   bob-cli · ws4                       │
│     provider error: context window exceeded                           │
│     sase agent show sase-s7.3 · sase chat sase-s7.3                   │
╰─ 2 succeeded · 1 failed → exit 1 ─────────────────────────────────────╯
```

Failures print the `error` field from `done.json` and the exact inspect commands.
Blocked targets print their reason and the command that unblocks them (the pending
question, or the plan awaiting review).

### Signal-safe teardown

The `Live` is entered and exited under `try/finally`, and `SIGINT`/`SIGTERM` print the
current state before exiting `130`/`143`. Ctrl-C must never leave the terminal in a
corrupted state, and a monitor timeout that SIGTERMs the process group must still
produce a readable last word.

### Tests

Render against a fixed-width `rich.Console` with `record=True` and assert on the
exported text — no PNG snapshots, so `just test-visual` is untouched. Cover row
ordering, the why-column for each pending state, the terminal-blocker warning, the
summary for mixed outcomes, and teardown on simulated interrupt.

## Phase `docs`: Documentation, Help Polish, and Verification

- `docs/cli.md` — add the `sase agent wait` row in the `sase agent` table and a short
  subsection covering targets, the exit-code table with its precedence rule, the
  blocked-state default and `-w`, and the stdout/stderr split. Note alongside the
  existing `-n` disambiguation paragraph that `sase agent wait -a` is `--all` in the
  same sense as `sase agent list -a`.
- `docs/monitors.md` — the gate idiom, since running this inline inside an agent turn is
  wrong and the monitor composition is the intended usage:
  ```bash
  sase monitor start -s WAITING -S WAITED -n 'agents finished; land the epic' \
    -- sase agent wait -a
  ```
- `docs/agent_families.md` — one cross-reference noting that waiting on a family root
  waits for the family including successors, matching `%wait`.
- Re-read `-h` output for `sase agent wait` end to end against
  `sase/memory/cli_rules.md`: alphabetical options, short alias on every long option,
  nothing required, colored output where it aids reading.
- Exercise the real command against live agents (launch a trivial agent, wait on it by
  name; run `-a` from a monitor) and confirm the exit codes.
- Final verification through a monitor, never inline:
  ```bash
  sase monitor start --command 'just check-full' \
    --start-status TESTING --stop-status TESTED \
    --next 'sase agent wait epic: report or fix check-full failures'
  ```

## Decisions Recorded

**No Rust core change.** The `rust_core_backend_boundary` rule was checked against
`../sase-core`: the artifact scan is Rust (`scan_agent_artifacts`), but wait-dependency
resolution, outcome classification, and family/clan aggregation are Python in this repo
(`src/sase/core/wait_dependency_resolution/`, `src/sase/agent/names/`,
`src/sase/core/dismissed_agent_completion.py`), and `grep` for `wait_dependency` /
`is_resolved` across `crates/sase_core/src` finds no counterpart. This feature composes
those existing shared helpers instead of adding new domain logic, so it stays on the
correct side of the boundary — and if that resolution later moves to Rust,
`sase agent wait` inherits it for free because it never re-derives the rules.

**No feature flag.** Per `sase/memory/sase_flags.md`, a flag routes behavior that
reaches users before it is ready. The phases are ordered so every landed commit is
coherent: `engine` adds no user-reaching surface, and `cli` lands a complete, correct
command that `live` only makes prettier. Nothing needs a disabled branch kept reachable.

**Multiple `NAME` positionals.** The request was for a single agent name, and `--all`
already forces a multi-target engine, so accepting `NAME ...` is nearly free and covers
the obvious "wait for the three agents I just launched" case. Flagged here as a
deliberate extension of the ask rather than an assumption.

**Blocked stops by default.** The most consequential semantic choice, argued above: a
gate that silently waits through a question you have not seen is the failure mode that
would make this command untrustworthy. `-w/--wait-blocked` is there for when you want
the other behavior.
