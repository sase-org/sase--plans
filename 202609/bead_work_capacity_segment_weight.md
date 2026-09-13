---
tier: tale
title: Make sase bead work --capacity admissible for every epic segment
goal: '`sase bead work --capacity N` (including the advertised run-alone value 1)
  launches every epic phase and land agent with an admissible queue budget, and any
  remaining queue-directive conflict is rejected before the command changes beads
  or spawns agents.'
size: medium
proposed_by: bbugyi200.athena.0ke
status: done
---

# Plan: Make `sase bead work --capacity` admissible for every epic segment

## Problem

`sase bead work -c 1 <epic>` — the value its own `--help` advertises as "use 1 to run
alone", and that the epic approval modal renders as `1 (run alone)` — produces an epic
whose land agent can never start.

Reproduction (installed `sase-core-rs` 0.34.23, `queue_capacity_budget` on by default):

```text
seg = "%model:@large\n%auto\n%queue(capacity=1)\n#bd/land_epic:sase-demo"
expanded = process_xprompt_references(seg, defer_xprompt_names=LAUNCH_DEFERRED_XPROMPT_NAMES)
extract_prompt_directives(expanded)
-> DirectiveError: %queue weight exceeds this launch's capacity budget; the launch
   could never be admitted. Increase capacity or lower weight.
```

The same probe with `capacity=2` succeeds (`capacity 2, weight 2.0`).

## Root cause

Three facts combine:

1. `render_multi_prompt()` in `src/sase/bead/work.py` stamps the **same**
   `%queue(capacity=N)` line (`_queue_capacity_lines`) into every phase segment and the
   land segment.
2. The segments are not uniform. The built-in `bd/land_epic` xprompt in
   `src/sase/default_config.yml` authors `%q(w=2.0)` (pinned by
   `tests/test_bead_xprompt_tags.py::test_builtin_land_prompt_requests_double_capacity`),
   so the land segment is weight 2 while phases are weight 1.
3. Epic `sase-zt` (phases `.1`/`.2`, landed 2026-09-12) turned capacity into a
   per-launch _budget_ and, with `queue_capacity_budget` on, makes the Rust queue
   contract reject `weight > capacity` as unsatisfiable. The rejection spans separate
   `%queue` occurrences in one launch unit, so the CLI-authored `capacity=1` and the
   xprompt-authored `w=2.0` collide.

Before `sase-zt`, capacity was a pre-admission threshold and `capacity=1, w=2` was
legal, which is why `--capacity 1` used to work.

The failure is also detected far too late. `--capacity` is validated alone
(`_capacity_from_args`, argparse `_queue_capacity_arg`). The composed queue fields are
first checked inside the spawned runner, where `src/sase/axe/run_agent_directives.py`
expands xprompts and then calls `extract_prompt_directives`. The launch-time guard in
`src/sase/agent/launch_guard.py` extracts directives from the _unexpanded_ segment and
never sees `%q(w=2.0)`. So `sase bead work -c 1` marks the epic ready, preclaims beads,
publishes the graph, and spawns phase agents, and only then does the land agent's runner
die. The same happens for an approved epic plan launched with capacity 1 from ACE or
Telegram, because `build_epic_launch_argv` forwards `--capacity` to the same command.

A second latent conflict has the same shape. A user-overridden `land_epic` or
`work_phase_bead` xprompt that authors its own `capacity` would collide with the
CLI-rendered one as a duplicate queue field, again only inside the runner.

## Design

### One rule for the rendered budget

With `queue_capacity_budget` **on**, `--capacity N` renders each segment's budget as:

```text
segment_capacity = max(N, ceil(segment_authored_weight))
```

`segment_authored_weight` is the queue weight authored by the segment's expanded xprompt
content. Without an authored weight it is 1, so the rendered value is plain `N`.

This is a translation, not a silent override. `sase-zt` defined capacity as "this
launch's budget replaces `max_running_agents`", and it already translates a persisted
`capacity: 0` to `admission_limit = effective weight`, meaning "drain, then run". The
smallest budget a weight-`w` agent can run in is `w`, and admission then requires zero
other occupied load. So `-c 1` keeps exactly its documented meaning, "run alone", for
the weight-2 land agent, and any `N >= w` is rendered unchanged. Capacity stays an
integer, hence the `ceil`, so a fractional authored weight such as `2.5` rounds up to 3.

With `queue_capacity_budget` **off**, render `N` unchanged in every segment. The Off
branch keeps threshold semantics, where `weight > capacity` is legal and `0` is a drain.
Keep that branch explicit so the flag's removal can delete it.

Rejected alternatives:

- **Fail fast only** (refuse `-c 1` whenever any segment is heavier). This is honest but
  makes the advertised "run alone" value unusable for every epic, because every epic has
  a weight-2 land agent the user did not author and cannot lower.
- **Drop `%q(w=2.0)` from `bd/land_epic`.** The weight is a deliberate resource claim
  from the weighted-queue rollout, and removing it would change admission for every epic
  launch, not just `--capacity` ones.
- **Move the clamp into Rust.** The weight comes from Python-side xprompt expansion, and
  epic prompt rendering already lives in `src/sase/bead/work.py`. The Rust contract
  stays the single validator (next section), so no admission rule is duplicated. Reopen
  this if epic multi-prompt rendering moves into `sase-core`.

### Pre-flight the composed queue fields before any side effect

Validate each segment's composed queue directives on the host, before
`launch_epic_bead_work()` renders its first multi-prompt, and therefore before the
dry-run preview, destructive-cleanup preview, mark-ready, preclaim, graph publication,
and spawn. Use the same expand-then-extract order the runner uses. Any remaining
conflict becomes a `BeadWorkError` that names the segment's agent and the xprompt, and
nothing changes. The known remaining case is an overriding xprompt that authors its own
`capacity` while `--capacity` is given. In `sase bead work --json` mode it surfaces
through the existing error payload.

Probe only the queue-relevant text of a segment, never the whole segment. The probe is
the rendered `%queue(capacity=…)` line when capacity is present, plus the segment's
xprompt reference lines: `#<work_phase_xprompt.name>:<bead_id>`, the `#plan` line when
`phase_requires_plan(size)` holds, and `#<land_epic_xprompt.name>:<epic_id>`. Leave out
the `%id`, `%clan`, `%w`, `%model`, and VCS-prefix lines, so the probe cannot trigger
wait resolution, name allocation, or workspace resolution. Expand with
`process_xprompt_references(..., defer_xprompt_names=LAUNCH_DEFERRED_XPROMPT_NAMES)`,
then run `extract_prompt_directives` and read `queue_weight` and `wait_runners` (the
authored capacity). Cache probe results by expanded-input text, so a many-phase epic
expands each distinct xprompt shape once rather than once per phase.

Run the pre-flight with the flag in either state. With the flag off, the clamp is
skipped but the duplicate-field check still protects the runner.

### Visibility

When any segment's budget was raised, print one concise line per raised segment in the
existing work-plan summary, for both dry run and real launch, before any confirmation
prompt, for example:

```text
  Capacity: requested 1 · sase-zp.land raised to 2 (queue weight 2.0)
```

The dry-run multi-prompt already prints each segment's rendered `%queue(capacity=…)`, so
the effective value is visible there too. Keep the _requested_ `N` as-is in the `--json`
success payload's `capacity` field, in `_resume_command()`, in plan-file resume
commands, and in `build_epic_launch_argv()`, so re-running any printed resume command
reproduces the same request.

## Changes

### `src/sase/bead/` — capacity resolution and rendering

- Add a small module, e.g. `src/sase/bead/work_queue_capacity.py`, exposing one
  resolver. It takes `plan`, both resolved xprompt workflows, and the requested
  `capacity` and returns a frozen result: a per-agent-name effective capacity mapping
  (land included), a tuple of raised entries
  `(agent_name, requested, effective, weight)`, and nothing else. The resolver performs
  the probe and pre-flight described above and raises a dedicated error on conflicts.
  For `capacity is None` it returns an empty mapping and performs no expansion, so
  default launches pay no new cost. Read the flag through
  `sase.feature_flags.snapshot.current_flags().enabled(FeatureFlag.queue_capacity_budget)`,
  the same check `launch_feature_flag_keys()` uses.
- `render_multi_prompt()` in `src/sase/bead/work.py`: keep it pure. Replace the single
  `capacity: int | None` input with the per-agent effective-capacity mapping, or add it
  alongside, so each phase and land segment renders its own `_queue_capacity_lines(...)`
  value. Update the docstring accordingly.
- `launch_epic_bead_work()` in `src/sase/bead/cli_work_handler.py`: resolve capacities
  once, in a new timer stage such as `queue_capacity_preflight`, after
  `work_plan_build`/`vcs_context` and before the first `render_multi_prompt` call. Pass
  the mapping to all three renders (initial, dry-run, post-revalidation). Convert the
  resolver's error to `BeadWorkError`. Print the raised-segment lines alongside
  `print_work_plan_summary`. The plan-file path (`cli_work_from_plan.py`,
  `cli_work_from_plan_resume.py`) reaches this function, so it needs no separate logic.
  Confirm that a plan-file launch failing the pre-flight rolls back like any other
  pre-launch `BeadWorkError`.

### CLI help and docs

- `src/sase/main/parser_bead_lifecycle.py` `--capacity` help: keep "use 1 to run alone"
  and state that a segment whose xprompt claims more weight gets a budget equal to its
  own weight. Keep the help concise and in the existing style.
- `docs/beads.md` (the `-c/--capacity N` paragraph near the `sase bead work` section and
  the option table row) and the `-c, --capacity` row in `docs/configuration.md` still
  describe the retired threshold semantics ("max already-running weighted load", "`0`
  waits for a drain"). Restate them as the per-launch budget, `N` at least 1, add the
  per-segment weight floor, and name `1` as run-alone. Epic `sase-zt`'s in-progress docs
  phase (`sase-zt.4`) sweeps `docs/xprompt.md`, `docs/ace.md`, `docs/configuration.md`,
  and `docs/troubleshooting/runner-slots.md`. Touch only the `sase bead work` capacity
  rows and paragraph, and rebase over that phase's edits if they have landed.
- No memory file changes: `sase-zt.4` owns the `sase/memory/xprompts.md` correction.

### Tests

Add or extend, covering **both** `queue_capacity_budget` states via
`from sase.feature_flags import override_flags`:

- `tests/test_bead/test_work_epic_plan.py` (or the existing render tests): with a
  per-agent mapping, phase segments render the requested `N` and the land segment
  renders its own value. `None` renders no `%queue` line.
- New resolver tests (e.g. `tests/test_bead/test_work_queue_capacity.py`):
  - Flag on, `capacity=1`, built-in xprompts: land is raised to 2, phases stay 1, and
    exactly one raised entry exists.
  - Flag on, `capacity=3`: nothing is raised.
  - Flag off, `capacity=1`: nothing is raised and land renders 1.
  - A fractional authored weight (`%q(w=2.5)` via a test-local land xprompt) rounds up
    to 3.
  - A test-local land xprompt authoring `%q:4` with `capacity=2` raises the dedicated
    conflict error in both flag states.
  - `capacity=None` performs no xprompt expansion (monkeypatch
    `process_xprompt_references` to fail).
- `tests/test_bead/test_cli_work_epic_launch.py`:
  - Extend `test_work_launch_threads_capacity_into_rendered_multi_prompt`, or add a
    sibling: with `capacity=1` and the flag on, the land segment carries
    `%queue(capacity=2)` and the phases carry `%queue(capacity=1)`.
  - A pre-flight conflict raises `BeadWorkError` before `mark_ready_to_work`,
    `preclaim_epic_work`, graph publication, or `launch_bead_work_agents` are called
    (assert those fakes were never invoked), including under `--dry-run`.
  - `--json` success payload and resume commands still report the requested capacity.
- Runner-path regression, the reproduction above as a test: for every segment of a
  `capacity=1` render with the flag on, expanding with
  `process_xprompt_references(..., defer_xprompt_names=LAUNCH_DEFERRED_XPROMPT_NAMES)`
  and calling `extract_prompt_directives` succeeds. Strip or neutralize any line that
  needs live state, or reuse the resolver's probe helper.

## Verification

- `just install` if the workspace virtualenv is stale, then `just check`. The change
  touches bead launch orchestration broadly enough that the worker should also run
  `just check-full` through the `/sase_monitor` skill with the `TESTING` / `TESTED`
  status pair before finishing.
- Manual, no side effects: `sase bead work <open epic> -c 1 --dry-run` shows
  `%queue(capacity=1)` on phases, `%queue(capacity=2)` on the land segment, and the
  raised-segment summary line. `sase bead work <plan.md> -c 1 --dry-run` still validates
  cleanly.
