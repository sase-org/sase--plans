---
tier: epic
title: "E1.5: enforced sase tool adoption and monitor wrapping"
goal: "Every heavy verification run by a SASE agent is a recorded ToolRun by
  construction: guarded recipes refuse a raw agent invocation, monitors wrap what they
  run, the wrapper is at least as faithful as the raw command, and the linked repos
  carry their own catalogs and guards.

  "
phases:
  - id: ownership-roots
    title: Make an agent an ownership root and always export the wrapper marker
    depends_on: []
    size: medium
    description:
      "ownership-roots: scrub executor-ownership variables at every agent launch, ignore
      an inherited monitor/proc id under SASE_AGENT, always export SASE_TOOL_NAME and
      SASE_TOOL_PROJECT_ROOT, and carry a monitor's starter agent into its recorded
      runs."
  - id: core-child-facts
    title: Record child process facts and authorize reaping in sase-core
    depends_on:
      - ownership-roots
    size: medium
    description:
      "core-child-facts: add the sase-core wire, store write, and binding that persist a
      running run's child pid, pgid, and process identity, extend reconcile to return
      identity-matched reap candidates, and move sase's core revision pin past that
      commit."
  - id: wrapper-fidelity
    title: Make the wrapper as faithful as the raw command
    depends_on:
      - core-child-facts
    size: medium
    description:
      "wrapper-fidelity: keep the child in the wrapper's process group under a live
      owner, make an inline child die with its wrapper, reap identity-matched survivors
      of lost runs, and give the child one merged pipe whenever a single target receives
      both streams."
  - id: recipe-guard
    title: Refuse a raw agent invocation of a guarded recipe
    depends_on:
      - wrapper-fidelity
    size: medium
    description:
      "recipe-guard: add the dependency-free tools/require_tool_run script, wire it into
      check and check-full, document the environment contract, teach the adoption report
      bypasses and refusals, and record the decision that partly supersedes
      record-before-admit."
  - id: monitor-wrap
    title: Wrap a monitor's command in sase tool run
    depends_on:
      - recipe-guard
    size: medium
    description:
      "monitor-wrap: upgrade an exact catalog match to a named run and wrap other
      verify-profile commands ad-hoc, in the proc argv only, behind a monitor.tool_wrap
      config field, leaving monitor_command and host completion bindings untouched."
  - id: linked-repo-catalogs
    title: Give the linked repos catalogs and guards
    depends_on:
      - monitor-wrap
    size: medium
    description:
      "linked-repo-catalogs: add project tools catalogs and the recipe guard to
      sase-core, sase-telegram, sase-github, and sase-research-artifacts so enforcement
      and named upgrades reach the raw residual that lives outside the sase repo."
proposed_by: bbugyi200.athena.0pf
create_time: 2026-09-22 13:05:36
status: wip
---

<!-- sase:links:start -->

## Links

| Relation     | Artifact                                                                                                              | Why                                                                           |
| ------------ | --------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| derives-from | [research:202609/sase_tool_guarded_recipes_and_monitor_wrapping/sase_tool_guarded_recipes_and_monitor_wrapping.md][1] | Consolidated report this plan implements, including its adjusted requirements |
| related      | [plan:202609/tool_e1_named_tools.md][2]                                                                               | Landed E1 plan whose executor, catalog, and ownership contracts this extends  |
| related      | [research:202609/sase_tool_epic_roadmap/sase_tool_epic_roadmap.md][3]                                                 | Control-plane roadmap whose E2/E4/E6/E7 phases depend on guaranteed coverage  |

[1]:
  https://github.com/sase-org/sase--research/blob/main/202609/sase_tool_guarded_recipes_and_monitor_wrapping/sase_tool_guarded_recipes_and_monitor_wrapping.md
[2]: https://github.com/sase-org/sase--plans/blob/main/202609/tool_e1_named_tools.md
[3]:
  https://github.com/sase-org/sase--research/blob/main/202609/sase_tool_epic_roadmap/sase_tool_epic_roadmap.md

<!-- sase:links:end -->

# Plan: E1.5 — enforced `sase tool` adoption and monitor wrapping

## Outcome and scope

E1 made `sase tool run check` possible and taught agents to prefer it. Guidance worked:
about 90% of inline agent `just check` calls in the sase repo are wrapped. This epic
makes the remaining gap structural rather than behavioral, because E4 receipts, E6's
forecasting corpus, and E7's fail-closed admission all need "every heavy agent run is a
ToolRun" to hold by construction.

Two mechanisms do that. A **recipe guard** refuses a raw agent invocation of a guarded
recipe and prints the wrapped form plus an override. **Monitor wrapping** covers the
detached half, which the guard cannot see because monitor supervisors scrub agent
identity on purpose.

Three wrapper defects land first. Today the wrapper is _less_ faithful than the raw
command in three ways, and both mechanisms push far more work through it:

- `sase-16c` — epic phase agents inherit the live epic-launch monitor's
  `SASE_MONITOR_ID`, so their runs record as monitor-owned with no compact output and no
  retained logs. A guard would force every phase agent onto that path.
- `sase-16b` — the child runs in a new session, so a SIGKILL of the caller's process
  group (a monitor timeout or stop, or a provider tool timeout) kills only the wrapper
  and orphans the `just check` tree; the run settles `lost` with no recorded pgid.
- `sase-16d` — two pipes pumped by two threads regroup interleaved output whenever
  stdout and stderr share a target, which covers monitor logs and `2>&1 | tail`.

Close each of those beads in the phase that fixes it.

Design inputs, read through `sase artifact read`:

- `research:202609/sase_tool_guarded_recipes_and_monitor_wrapping/sase_tool_guarded_recipes_and_monitor_wrapping.md`
  and, only if a phase needs the underlying detail, its three `__cld`, `__gem`, `__mus`
  siblings in the same directory.
- `research:202609/sase_tool_epic_roadmap/sase_tool_epic_roadmap.md`,
  `plan:202609/tool_e1_named_tools.md`.
- `decisions:record-before-admit`, `decisions:rust-core-required`, plus SASE size
  guidance, `sase/memory/lint_and_test.md`, `sase/memory/sase_flags.md`,
  `sase/memory/generated_skills.md`, and `sase/memory/symvision.md` through
  `/sase_memory_read`.

Planning baseline: sase master `d3002aba1`, core pin
`c5186cc4af853c86fb07485bcb3add69463984f3`. `sase-16b`, `sase-16c`, and `sase-16d` are
open, `ready`, and linked `related`. `sase-135` (E1) is closed. Recheck each of these
facts when implementing; do not redo landed work and do not change unrelated task
lifecycles.

Every phase that touches a repository other than the sase checkout opens it with
`/sase_repo` and uses only the printed path. Read that repo's own `AGENTS.md` first.
Paths below are repository-relative and never name a numbered checkout. Commits and
release sequencing stay with host-owned finalizers. No implementation change precedes
approval.

## Questions this plan settles

The report left four questions open. The user answered the third; the plan adopts the
report's own recommendation for the rest.

1. **Monitor wrap default: `verify`.** It covers every local verification monitor seen
   since E1 and skips CI watches and one-off scripts. `monitor.tool_wrap` makes widening
   to `all` a config edit, not a code change.
2. **Guard match: strict.** `SASE_TOOL_NAME` must equal the guarded recipe's tool name.
   Lenient matching would accept `sase tool run -- sh -c 'just install && just check'`
   and give up the named identity that LAST/TYPICAL, receipts, and fingerprints need.
3. **Linked-repo catalogs: yes** (the user's answer, phase `linked-repo-catalogs`). The
   report's precondition is already satisfied and must be re-verified rather than
   re-investigated: `content_layout._is_project_root` returns true for any `.git`
   directory, so every linked checkout is _already_ a project root and a config file
   changes nothing about root discovery; and `sase-telegram` already ships a root
   `sase.yml` (`commit_hooks`), so a linked repo demonstrably carries a project config
   layer today without becoming a registered SASE project.
4. **Epic placement: its own epic before E2.** E2's `sase tool run -H check` then
   becomes "start a verify monitor whose command is `check`", built on this epic's
   wrapping.

## Binding contracts

### The environment contract

Three variables decide everything, and the decision must be makeable without starting a
`sase` process, because the override exists precisely for when `sase` is broken.

| Variable                 | Set by                           | Meaning                                                                       |
| ------------------------ | -------------------------------- | ----------------------------------------------------------------------------- |
| `SASE_AGENT`             | the agent runner (already)       | this process tree is a SASE agent's own shell                                 |
| `SASE_TOOL_NAME`         | **new:** `sase tool run`, always | the tree is inside `sase tool run <name>`, or `ad-hoc`                        |
| `SASE_TOOL_PROJECT_ROOT` | **new:** `sase tool run`, always | the resolved project root the named run executes in; empty for an ad-hoc run  |
| `SASE_TOOL_BYPASS`       | **new:** an agent or human       | run raw on purpose; any non-empty value bypasses, and the value is the reason |

`SASE_TOOL_NAME` must be exported whether or not recording succeeded. E1 recording is
deliberately fail-open and `child_env(recorded=False)` _removes_ `SASE_TOOL_RUN_ID`, so
a guard keyed on the run id would refuse the child of a fail-open `sase tool run check`
and tell the agent to run the command it is already running.

`SASE_TOOL_PROJECT_ROOT` is this plan's one addition to the report's contract, and it
exists because the user put linked-repo catalogs in scope. Without it, an agent that
runs `sase tool run check` in the sase repo and then `cd`s into `sase-core` and types
`just check` would satisfy sase-core's `check` guard with sase's marker. The guard
therefore requires the name _and_ the root to match. Set it from the named run's
resolved project root (`ResolvedToolArgv.cwd`) and leave it empty for ad-hoc runs, whose
`SASE_TOOL_NAME=ad-hoc` can never equal a guarded recipe's name anyway.

`SASE_TOOL_BYPASS` is a permanent interface and `monitor.tool_wrap` is a permanent
config field, so per `sase/memory/sase_flags.md` this epic uses **no feature flag**:
every user-reaching piece ships complete inside its own phase.

### The guard rule

An agent may run a guarded recipe only from inside `sase tool run <that tool>` for
_that_ project root, or with an explicit bypass. The rule is deliberately not "the
caller is not an agent": inside `sase tool run check` the child is still an agent
process running `just check`. Name the guard after the rule (`require_tool_run`), never
`ensure-not-agent`.

The guard is a dependency-free POSIX `sh` script, wired as the **first** `just`
dependency so a refusal costs milliseconds rather than a dependency sync. It must never
start `sase`:

- a `sase` process on the refusal path fails in exactly the states the override exists
  for, and prints a traceback where the agent needs the bypass instructions;
- every human and CI `just check` would otherwise pay roughly 0.3 s measured (not the
  80–120 ms estimated) for logic that needs three environment variables.

It **refuses**, it does not redirect. A `just` dependency cannot replace its recipe, so
redirecting would require splitting each recipe into a public stub and a private body,
with double-run hazards; and once E4 receipts or E6 routing exist, a silent redirect
could return a cached receipt or hand off to a monitor and end the agent's turn without
the agent asking. Revisit after E6.

The guard is a guardrail against habit, not a security boundary. `env -u SASE_AGENT`
defeats it; the bypass is deliberately easier and more visible than that.

Guarded recipes in v1 are **`check` and `check-full` only**. `test` and `install` are
never guarded: guarding `test` mostly pushes agents to call pytest directly, which
cannot be guarded at all.

### Wrapper fidelity and ownership

- **An agent is a new ownership root.** Nothing an agent's own shell runs was captured
  by an ancestor monitor, proc, or tool run, so the agent-launch boundary must scrub
  `SASE_TOOL_*`, `SASE_MONITOR_*`, and `SASE_PROC_*`. Keep this separate from
  `scrub_agent_identity_env`, which also runs inside monitor and proc supervisors where
  those owner variables are set on purpose.
- **Process groups follow the process owner.** Under a live monitor/proc owner the child
  joins the wrapper's process group and the wrapper skips its own SIGKILL escalation, so
  the owner's `killpg` reaches the whole tree and stays the single process owner. For an
  inline run the child tree must die with the wrapper.
- **Child facts are recorded at spawn, not at settlement.** Today
  `child_pid`/`child_pgid` reach the ledger only through `finish_tool_run`, so a `lost`
  run records no group to reap.
- **Reaping is identity-matched and conservative.** E1's accepted contract said
  reconciliation "never adopts or kills an orphan"; this epic narrows that to a verified
  case and must say so in its tests. Rust returns _authorized_ reap candidates (the same
  split retention already uses), and Python signals only when the recorded child
  process-start identity still matches the live process-group leader. A PID-reuse
  mismatch, an unreadable identity, or a permission error is never proof, and never a
  reason to signal.
- **One target means one stream.** When an enclosing owner holds the output, or when fds
  1 and 2 name the same device and inode, give the child a single merged pipe and one
  pump. Inline compact mode, which retains separate logs, keeps two pipes.
- **Attribution survives the supervisor scrub.** 13 of 17 monitor-owned runs currently
  record `agent: null` because the supervisor scrubbed identity. The monitor already
  knows its starter (`reserved_by`); pass it in a tool-specific variable rather than by
  restoring `SASE_AGENT*`, which would re-enable compact mode inside an owner.

### Monitor wrapping policy

The wrap goes only into the argv the proc supervisor execs. `monitor_command` stays
exactly as the agent wrote it and `monitor_execution_argv` stays unset for ordinary
monitors, because `monitor/host_completion_state.py::_command_argv` reads
`monitor_execution_argv` or else `monitor_command`; rewriting either breaks every
prepared-completion `-f` binding.

| Monitor                                                                                                | Proc argv                                    |
| ------------------------------------------------------------------------------------------------------ | -------------------------------------------- |
| Host-owned `execution_argv` launch (an epic `sase bead work`)                                          | unchanged, **never** wrapped                 |
| Exactly one `sase tool run …` already                                                                  | unchanged                                    |
| A simple command equal to a catalog tool's argv, with the monitor's cwd at that catalog's project root | `<sase> tool run <name>` (**named upgrade**) |
| Anything else under `-p verify`                                                                        | `<sase> tool run -- /bin/sh -c CMD` (ad-hoc) |
| Anything else with no profile                                                                          | unchanged in v1                              |
| `SASE_TOOL_BYPASS` set, `monitor.tool_wrap: off`, or bindings unavailable                              | unchanged, plus one reason line in the log   |

Wrapping every monitor is wrong in five ways, and each exclusion above answers one:
wrapping an epic launch makes every phase agent's runs children of a long-lived ad-hoc
run (`sase-16c` one level up); re-wrapping `-- sase tool run check` produces two
ToolRuns for one semantic run; rewriting the recorded command breaks `-f`; the kill race
(`sase-16b`) would run on every monitor timeout; and sleeps, CI watches, and ad-hoc
scripts produce ToolRuns with no stable identity that cost retention and add nothing to
E6.

`<sase>` must be the supervisor's own installation (`sys.executable -m sase`), not
whatever `sase` is first on `PATH`. Extra arguments count as a catalog match only where
that tool's `args: allow` policy permits them. The monitor log remains the output of
record: owner-mode runs retain no duplicate stdout/stderr, so `sase tool show -l` is
empty for them by design.

### Core boundary

Shared backend behavior belongs in `sase-core`, per `decisions:rust-core-required` and
the repo's Rust-boundary memory. In this epic that is exactly one thing: the ToolRun
wire, store write, and reconcile algebra for child process facts and reap authorization
(phase `core-child-facts`). `ToolRunBeginRequestWire` is `deny_unknown_fields` and begin
happens _before_ spawn, so this needs a new observe call rather than extra begin fields.
Everything else — the guard script, process-group choice, signal handling, stream
merging, monitor policy, config, docs — is Python, shell, and presentation, and stays in
the sase repo.

Because `sase-core-revision.txt` pins the SHA that CI builds, and the `lint` job's
"Check pinned core bindings" step fails when sase calls a binding the pin does not
expose, the pin moves in `core-child-facts`, before `wrapper-fidelity` calls the
binding.

### Governance and memory

`decisions:record-before-admit` lists "adoption is instruction-led … so bypass is
measured, not enforced" as an accepted cost. A guard reverses that clause. Its stated
reopen condition (a measured concurrent-duplicate or queue-starvation rate) has _not_
been met, so this is a deliberate change of course, not a reopening: write a **new**
decision record and mark the old one `superseded-in-part`, never edit its accepted body.

Every memory or skill-source change below is named explicitly so plan approval carries
its authorization; each phase still routes the edit through `/sase_memory_write` and
republishes with `sase memory init`. The full set for this epic is:

- new `sase/memory/decisions/guarded-recipes.md` (phase `recipe-guard`);
- a `superseded-in-part` mark plus `superseded_by` and a back-link on
  `sase/memory/decisions/record-before-admit.md` (phase `recipe-guard`);
- edits to `sase/memory/lint_and_test.md` (phases `recipe-guard` and
  `linked-repo-catalogs`);
- edits to `src/sase/xprompts/skills/sase_monitor.md` (phase `monitor-wrap`).

Skill sources are generated artifacts: preview with `sase skill init --diff` only, and
leave deployment to the landed canonical revision, per
`sase/memory/generated_skills.md`.

### Acceptance cannot depend on a green master

The ledger shows `check` failing 76% of the time on athena. Every exit criterion in this
plan is therefore stated as observable behavior — a refusal, an argv, an owner, a
recorded field, a surviving or absent process — never as "`just check` passes". Run
`just check` as the repo requires it, but do not make a phase's acceptance contingent on
master being green.

### Out of scope

Wrapping `sase proc run` (E2's standalone leg); redirect instead of refusal (revisit
after E6); guarding `test` or `install`; a `guard:` field on catalog entries, because
`ToolDefinitionWire` is `deny_unknown_fields` and a new field would change every
definition digest and break LAST/TYPICAL continuity for no proportionate gain; any
fail-closed admission (E7); and any new `sase tool` subcommand.

## 1. ownership-roots

Give the executor a marker a guard can trust and stop an agent inheriting an ancestor's
ownership.

Add a `scrub_executor_ownership_env` helper beside the existing scrubbers in
`src/sase/agent/env_hygiene.py` that removes `SASE_TOOL_*`, `SASE_MONITOR_*`, and
`SASE_PROC_*`, and call it from every agent-launch boundary: `agent/launch_spawn.py`,
`agent/launch_admission_coordinator.py`, and `agent/launch_condition_runtime.py`. Do not
widen `scrub_agent_identity_env`, which also runs in `procs/supervisor.py` and
`procs/spawn.py` where those owner variables are deliberate. Verify that the epic
launcher reads `SASE_MONITOR_ARTIFACTS_DIR` in its own process and not in the agents it
spawns before removing it from the child env.

As a second safeguard, have `tool/ownership.py::resolve_ownership` ignore a monitor or
proc id when `SASE_AGENT` is set. Monitor and proc supervisors scrub `SASE_AGENT*`, so
the two can only coexist through an agent launch. Keep the existing settled-monitor and
terminal-proc checks; this is an additional reason to drop an id, not a replacement.
`_parent_exists` checks existence rather than liveness, so an inherited
`SASE_TOOL_RUN_ID` gets the same treatment.

In `tool/executor_process.py::child_env`, always export `SASE_TOOL_NAME` (the resolved
tool name, or `ad-hoc`) and `SASE_TOOL_PROJECT_ROOT` (the named run's resolved project
root, empty for ad-hoc), independently of `recorded`. Keep the existing behavior that
`SASE_TOOL_RUN_ID` and `SASE_TOOL_RUN_EVENTS` are removed when recording failed, so a
child's stages cannot attach to a run that does not exist. `child_env` needs the
resolved argv to do this, so thread `ResolvedToolArgv` (or just the two values) through
from `executor._execute_resolved`.

Restore monitor attribution: have `monitor/start.py` add `SASE_TOOL_RUN_AGENT` to the
proc request's env overlay from `lane_start.starter_agent`, and have
`tool/executor_recording.py::begin_tool_run` fall back to it when `SASE_AGENT_NAME` is
absent. Do not restore `SASE_AGENT*` inside a supervisor: that would flip compact output
back on inside an owner. `SASE_TOOL_RUN_AGENT` is scrubbed at agent launch like every
other `SASE_TOOL_*` variable, which is correct — an agent is a new root.

Tests: a fake agent launch proves the child env carries no `SASE_TOOL_*`,
`SASE_MONITOR_*`, or `SASE_PROC_*`; `resolve_ownership` with `SASE_AGENT` plus a live
monitor id resolves to no owner and `owns_output=True`; `child_env` exports the marker
on both the recorded and unrecorded paths, and exports `ad-hoc` with an empty root for
`run -- ARGV`; a monitor-started run records the starter agent. Close `sase-16c` with
the live evidence it asks for: a phase agent's `sase tool run check` gets compact output
again and `sase tool show RUN -l` replays it.

## 2. core-child-facts

Open `sase-core` with `/sase_repo`, read its `AGENTS.md`, and work only in the printed
path. Never run bare `cargo`; `just check` there takes about five minutes, so give it a
generous explicit timeout.

Under `crates/sase_core/src/tool_run/`, add the wire, store write, and normalization for
observing a running run's child: `run_id`, `child_pid`, `child_pgid`, and the child's
process-start identity, persisted onto the running row rather than waiting for finish.
`ToolRunBeginRequestWire` is `deny_unknown_fields` and begin runs before the spawn, so
this is a new request/result pair (an observe call), not an extra begin field; keep the
existing `child_pid`/`child_pgid` columns as the single home for these facts and make a
later `finish` that repeats them idempotent rather than conflicting. Expose it through
`crates/sase_core_py` next to the other `tool_run_*` bindings.

Extend reconciliation so a run it marks `lost` can report an **authorized reap
candidate**: the recorded pgid plus the recorded child identity, emitted only when the
run's own wrapper is definitively dead and the child facts were recorded. Rust
authorizes; it never signals. Unknown liveness stays unknown. Add golden fixtures and
tests for observe-then-finish, observe-then-lost, a missing observation, a replayed
observation, and a reconcile result with and without candidates.

On the sase side, add the thin adapter in `src/sase/core/tool_run.py` beside the
existing wire-only facades, extend the installed-binding validation inventories
(`tools/check_sase_core_rs_bindings`, `tools/validate_sase_core_rs`) deliberately rather
than by editing golden counts to whatever the run prints, and extend
`tools/smoke_sase_core_rs_tool_runs` with a real-binding round trip of the new call.

Move `sase-core-revision.txt` past the new `sase-core` commit once that commit is pushed
and reachable, so CI builds a core that exposes the binding before phase 3 calls it.
Nothing in `src/sase/` calls the new adapter yet, so symvision will report it as unused:
add it to the Justfile symvision command's
`--epic-symbol <this epic's bead id>(<symbol>)` whitelist, and require phase 3 to remove
that entry when the real caller lands. Read `sase/memory/symvision.md` before doing so.

## 3. wrapper-fidelity

Fix `sase-16b` and `sase-16d` in `src/sase/tool/executor_process.py` and
`src/sase/tool/executor.py`, and turn both reproductions into regression tests.

Process groups. `spawn_child` currently always passes `start_new_session=True`. Make the
choice explicit and ownership-driven:

- with a live enclosing monitor or proc owner, spawn the child into the wrapper's own
  process group and skip the wrapper's `TERM_ESCALATE_SECONDS` SIGKILL escalation
  entirely, so the owner's `killpg` reaches the whole tree and the owner stays the
  single process owner;
- for an inline run, make the child tree die with the wrapper. Choose between sharing
  the caller's process group and a parent-death signal plus a bounded reap, and state
  the choice and its portability cost in the code; whichever is chosen must survive a
  SIGKILL of the caller's group, which no in-wrapper handler can catch.

Call the new observe binding immediately after a successful spawn, before the pumps
start, so a run killed seconds later still records a reapable group. Recording stays
fail-open: an observe failure warns at most once and never changes the child's result.

Reaping. In `src/sase/tool/liveness.py`, apply the candidates reconcile returns: verify
the live process-group leader's process-start identity still matches the recorded one,
then signal the group (TERM, then KILL after a bounded grace). A mismatch, an unreadable
identity, a permission error, or a missing process is a diagnostic, never a signal.
Never signal a group that is not authorized, and never signal from a read-only store
path.

Stream fidelity. Give the child a single merged pipe and one pump when an enclosing
owner holds the output, or when `os.fstat(1)` and `os.fstat(2)` report the same device
and inode. Inline compact mode, which retains separate stdout and stderr logs, keeps two
pipes; document in `docs/tool.md` that only the merged mode preserves interleaving and
that `show -l` still claims no total order between two retained streams.

Tests, all with isolated `SASE_HOME`s and full fixture-process-group cleanup even when
an assertion fails: a TERM-ignoring child under a simulated supervisor that SIGTERMs
then SIGKILLs the wrapper's group at 5.0 s leaves no surviving `sh`/`sleep`; an inline
run whose caller's group is SIGKILLed leaves nothing behind; a merged-target run
interleaves `out1 err1 out2 err2 …` instead of regrouping; a reconcile with a stale pgid
whose identity no longer matches signals nothing and says why. Remove the phase-2
symvision epic-symbol whitelist entry in the same change. Close `sase-16b` and
`sase-16d`.

## 4. recipe-guard

Add `tools/require_tool_run`, a POSIX `sh` script with no `sase` dependency, mode 0755.
It takes the tool name; exits 0 when `SASE_AGENT` is unset (humans, CI, finalizers,
monitors, procs); exits 0 when `SASE_TOOL_NAME` equals the tool _and_
`SASE_TOOL_PROJECT_ROOT` resolves to the recipe's own root (compare with `pwd -P`, a
builtin, so the guard still forks nothing); exits 0 with one stderr line naming the
reason when `SASE_TOOL_BYPASS` is non-empty; exits 0 with a stderr line when `sase` is
not on `PATH` at all; and otherwise prints the wrapped form and the bypass form on
stderr and exits 2. It uses a `#!/bin/sh` shebang, so
`tools/typecheck_extensionless_tools` correctly skips it, as it already skips
`tools/run_silent`.

Wire it as the first dependency of both guarded recipes in `Justfile`, ahead of
`_setup`:

```just
# Agents must run guarded tools through `sase tool run` (docs/tool.md).
_require-tool-run name:
    @tools/require_tool_run {{ name }}

check: (_require-tool-run "check") _setup
check-full: (_require-tool-run "check-full") _setup
```

Add a test that asserts this wiring, so a later edit cannot silently drop the guard:
parse the `Justfile` and require each guarded recipe's first dependency to be
`_require-tool-run` with that recipe's own name. This replaces the report's optional
`guard:` catalog field, which is out of scope above.

Run the full behavior matrix as tests, each with an explicit environment: human; CI;
finalizer (allow-listed env with no `SASE_AGENT`); monitor and proc supervisor env;
agent raw (refused, exit 2, all three message lines); agent wrapped as this tool; agent
wrapped as a _different_ tool (refused); agent wrapped for a _different project root_
(refused — the case linked-repo catalogs create); recording-failure wrapped (allowed,
because `SASE_TOOL_NAME` is exported regardless); bypass with a reason; and no `sase` on
`PATH`.

Extend `tools/tool_adoption_report` with a `bypassed` class
(`SASE_TOOL_BYPASS=… just check`) and a refusal count (an exit-2 raw call followed by a
wrapped one), and make it filter by each call's own timestamp rather than the file's
modification time, so a window cannot mix in older calls. Update the "Measuring
adoption" section of `docs/tool.md` accordingly.

Document the environment contract in `docs/tool.md` as a table any repo can implement
from — that table is what phase 6 copies — together with the guard rule, the strict name
and root match, the bypass, and the fail-open-without-`sase` behavior.

Through `/sase_memory_write`: add `sase/memory/decisions/guarded-recipes.md` stating the
claim (an agent runs a guarded recipe only inside `sase tool run` for that root, or with
an explicit bypass), why it beat a `sase` subcommand and a redirect, its costs, and its
own reopen condition (sustained bypass above a stated rate, or any false refusal); mark
`sase/memory/decisions/record-before-admit.md` `superseded-in-part` with `superseded_by`
and a short `[[...]]` back-link naming only the bypass clause, leaving the rest of its
accepted body untouched; and note in the new record that _recording_ stays fail-open,
because the guard enforces routing, not recording success. Update
`sase/memory/lint_and_test.md` so the stale-install fallback becomes
`SASE_TOOL_BYPASS='<why>' just check` plus a recorded `sase update`, rather than a bare
raw `just check`. Regenerate with `sase memory init`.

## 5. monitor-wrap

Apply the §"Monitor wrapping policy" table where `monitor/start.py` builds the proc
request's `argv`, next to the existing `monitor_proc_argv` / `compile_monitor_argv`
branch. Put the decision itself in a small, separately testable helper (a new module
beside `monitor/proc_adapter.py`) that takes the command string, the execution argv, the
profile, the monitor's cwd, and the config value, and returns the argv plus an optional
"unwrapped because …" reason. `monitor_command` and `monitor_execution_argv` must be
byte-identical to today for every input.

Resolve the catalog against the **monitor's** cwd, not the starting agent's, since the
two differ whenever an agent starts a monitor for another checkout. Match a simple
command to a catalog entry by comparing shell-split argv with the entry's argv, allowing
extra arguments only when that entry's `args: allow` policy permits them, and requiring
the monitor's cwd to be that catalog's project root. Detect an already-wrapped command
by argv shape (`… tool run …` for any path to `sase`), not by string prefix. Build the
wrapper invocation from `sys.executable -m sase`.

Add `monitor.tool_wrap` with values `off | verify | all`, default `verify`, to
`src/sase/default_config.yml` _and_ to the `monitor` block of
`src/sase/config/sase.schema.json`, which is `additionalProperties: false`. Read it
through a `get_monitor_tool_wrap` accessor in `src/sase/config/_settings.py` that fails
open to `verify`, matching `get_monitor_evidence_limits`. Also update the keymap/default
config expectations the repo's gotchas memory calls out, if the new field touches them.

Honor `SASE_TOOL_BYPASS` in the starting agent's environment as "do not wrap", and write
exactly one `sase: running unwrapped (<reason>)` line into the monitor log for every
unwrapped-by-policy case, so the log explains itself.

Tests: an epic launch with `execution_argv` produces byte-identical argv and creates no
ToolRun; `-- sase tool run check` is untouched; `-p verify -- just check` in the sase
project root becomes one **named** `check` run with `owner_kind=monitor` and the starter
agent recorded; `-p verify -- 'just install && just check'` becomes one ad-hoc run whose
inner wrapped `check` records as its child; a no-profile CI-watch command is untouched
under `verify` and wrapped under `all`; `off` wraps nothing; a `-f` monitor's host
completion still resolves its command and succeeds; and `--next-output auto` evidence
extraction still selects the right tail with the wrapper's `sase tool run <id>` header
and completion footer in the log.

Through `/sase_memory_write`, update `src/sase/xprompts/skills/sase_monitor.md`: plain
`-- just check` is now equivalent to `-- sase tool run check`, so agents need not
remember the wrapper; drop the "prepared-completion `-f` monitors still use raw
`just check`" caveat, because those are recorded now too; and replace the stale-install
fallback with `SASE_TOOL_BYPASS='<why>'`. Preview with `sase skill init --diff` only.
Document the policy table in `docs/monitors.md` and cross-reference it from
`docs/tool.md`.

## 6. linked-repo-catalogs

Close the largest remaining raw residual. About 80% of the raw heavy calls since E1 are
in linked repos, mostly `sase-core`, and neither the guard nor the named upgrade can
reach a repo with no catalog.

Before editing, re-verify the two facts that make this safe, and record what you
observed in the phase bead: `content_layout._is_project_root` already returns true on
`.git`, so adding a config file does not change project-root discovery for any linked
checkout; and `sase-telegram`'s existing root `sase.yml` shows a linked repo carries a
project config layer without being registered as a SASE project. Then confirm by
inspection that adding only a `tools:` key changes no merged-config value, since
`tools:` is read from the project layer alone and never merged.

For each of `sase-core`, `sase-telegram`, `sase-github`, and `sase-research-artifacts`,
opened with `/sase_repo`:

- add a `tools:` catalog with at least a `check` entry whose argv is that repo's real
  verification command, `args: deny`, declared `inputs` for its own lock and recipe
  files, and `fingerprint.toolchain` probes that match that repo's toolchain — cargo and
  rustc for `sase-core`, the venv Python for the three Python plugins. Do not copy
  sase's probes verbatim. Use `stages: none` unless that repo actually emits the
  `run_silent` stage protocol.
- put the catalog in the **canonical** `sase/sase.yml`, except in `sase-telegram`, where
  it must extend the existing root `sase.yml`: `resolve_project_layout` raises
  `LayoutCollisionError` when both locations exist.
- add a copy of the guard script, derived from the `docs/tool.md` contract table rather
  than reinvented, in whatever directory that repo already uses for dev scripts
  (`scripts/` in `sase-core`), and wire it as the first dependency of that repo's
  `check` recipe. In the three plugin repos `check` is a dependency-only recipe
  (`check: lint test`), so the wiring is `check: (_require-tool-run "check") lint test`.
- add a short section to that repo's `AGENTS.md` (or `CLAUDE.md` where it has no
  `AGENTS.md`) stating that agents run `sase tool run check` there and how to bypass.

Skip `sase-nvim`: it has no Justfile and no verification recipe of its own, so it has
nothing to guard.

Verify per repo, from inside that checkout: `sase tool list` shows the new entry;
`sase tool run check` resolves the catalog, runs the repo's real command, and records a
run; a raw agent `just check` is refused with the three-line message;
`SASE_TOOL_NAME=check` inherited from a sase-repo wrapper does **not** satisfy the
guard, because `SASE_TOOL_PROJECT_ROOT` names a different root; and a human `just check`
is unaffected. Also confirm the new entry's definition digest differs from the sase
repo's `check`, so LAST and TYPICAL cannot contaminate each other under the shared
`SASE_PROJECT` identity that agents export; if two digests do collide, differentiate the
definitions rather than changing `tool_project_identity`, which would re-attribute every
existing run.

Each repo is a separate commit through host-owned finalizers, and each carries its own
CI; treat a red gate in one as that repo's problem, not a reason to skip the others.
Finally, through `/sase_memory_write`, add a short paragraph to
`sase/memory/lint_and_test.md` stating that `sase tool run check` is now the wrapped
form in the linked repos too, and rerun `tools/tool_adoption_report` to record the
post-epic wrapped/raw split for both the sase repo and the linked repos in the phase
bead.
