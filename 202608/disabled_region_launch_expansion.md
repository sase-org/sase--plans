---
tier: tale
title: Honor %xprompts_enabled:false at the launch-expansion boundary
goal:
  "A workflow reference that appears as prose inside a %xprompts_enabled:false region is
  never expanded at launch, and a bare #fork can never resolve the run being launched as
  its own parent."
size: medium
proposed_by: bbugyi200.athena.0d6
---

- **AGENTS:**
  - [bbugyi200.athena.0d6](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0d6.md)
- **COMMITS:**
  - [9c7c10a](https://github.com/sase-org/sase/commit/9c7c10aa2a9f73445db7361a6dfaf0e0dbec9877)
    — feat(axe,xprompt): expand disabled-region handling to fork setup, embedded
    workflows, and VCS tag parsing

# Plan

## Motivation

Agent `sase-t8.1--1` never reached its model. It died during launch with:

```
WorkflowExecutionError: Python step 'resolve' failed:
RuntimeError: Invalid fork parents:
- parent 1 (<default>): No agent with chat history found for: sase-t8.1--1
```

Artifacts:
`/home/bryan/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202608/24/20260824191559`

The run was a monitor follow-up whose prompt began with `#fork:sase-t8.1--plan` and
whose body was correctly wrapped by the monitor in a `%xprompts_enabled:false` /
`%xprompts_enabled:true` region. Inside that region, the monitor's `--next` prose read:

> ... typed proc/monitor #fork sources in `src/sase/scripts/agent_chat_from_name.py` ...

That incidental `#fork` token was expanded as a real, argument-less workflow reference.
A bare `#fork` resolves its parent through `_resolve_default_agent_name()`, which
returned the launching run's own name, and the resolver correctly refused to fork an
agent with no chat history yet.

Two independent defects combined. Both are ours and both are worth fixing.

### Defect A (primary): launch expansion ignores disabled regions

`src/sase/main/query_handler/_embedded_workflows.py` computes its skip set with
`code_literal_ranges()`, whose contract is explicitly _"fenced and inline code ranges,
**excluding** disabled regions"_. The sibling helper `literal_zone_ranges()` is
documented as _"all launch-inert ranges: fenced, disabled, and inline code"_ — that is
the one a launch-boundary expander wants.

The inconsistency is provable on the failing prompt. `write_used_xprompts()` runs over
the same query at the top of `expand_embedded_workflows_in_query()` and reaches
`literal_zone_ranges()` through `xprompt_inspect`, so `xprompts.json` for the failed run
records exactly one fork reference:

```json
[
  {
    "name": "fork",
    "kind": "workflow",
    "positional": ["sase-t8.1--plan"],
    "named": {},
    "tags": []
  }
]
```

...while the expander in the same function found two. Reproduced against the persisted
`raw_xprompt.md`:

```
'#fork:sase-t8.1--plan'  offset 0      in code_literal=False  in literal_zone=False
'#fork'                  offset 26507  in code_literal=False  in literal_zone=True
```

The metadata collector and the expander disagree about the same text, which is also why
the ACE row for this run shows only the intended reference and hides the token that
actually killed it.

Because references are expanded last-to-first, the prose `#fork` was processed before
the real one, so the legitimate parent never got a chance to resolve. Reference ordering
is not the bug and needs no change.

### Defect B (enabling): bare `#fork` can resolve the run being launched

`_resolve_default_agent_name()` in `src/sase/scripts/agent_chat_from_name.py` calls
`get_most_recent_agent_name(exclude_artifacts_dir=os.environ.get("SASE_ARTIFACTS_DIR"))`.
Its docstring promises the current run is excluded "so an agent cannot accidentally
resume itself".

That promise does not hold in the launch-deferred path.
`expand_deferred_launch_xprompts()` is called from `_admit_and_launch()` in
`src/sase/axe/run_agent_runner.py`, but `SASE_ARTIFACTS_DIR` is not published until
`_publish_phase_env()` runs inside `run_execution_loop()`
(`src/sase/axe/run_agent_exec.py`), which is strictly later. At expansion time the
variable is unset — or, worse, still holds an inherited value from whatever launched
this runner. Either way `exclude_artifacts_dir` is wrong and the self-exclusion silently
no-ops.

Note the blast radius: `bootstrap_agent_run()` has already written this run's
`agent_meta.json`, and `get_most_recent_agent_name()` only considers metadata that
carries a name. So a bare `#fork` in a **named** run (family/clan agents, which is most
scheduled work) will usually resolve to itself and fail, while an unnamed ad-hoc
`sase run "#fork ..."` still works by accident. Fixing A alone would leave that trap
armed.

### Defect C (same class as A): launch VCS tag scanning

`find_vcs_workflow_tag_span()` appears twice — `src/sase/xprompt/_parsing.py` and
`src/sase/xprompt/_parsing_vcs_tags.py` — and both use `code_literal_ranges()`. A
`#gh:<repo>` written as prose inside a disabled region can therefore be picked up as the
run's VCS workflow tag. This is the identical failure shape as A (inert prose steering a
launch decision) and is cheap to fix alongside it. The docstrings on both functions
currently say only "fenced or inline code"; they should say disabled regions too.

## Scope note

All three fixes are Python glue at the launch boundary. `disabled_region_ranges` is pure
Python in this repo and `literal_zone_ranges` already exists, so nothing here crosses
into `sase-core`. Do not add a Rust change for this.

## Implementation

### 1. Expand only outside launch-inert zones

In `src/sase/main/query_handler/_embedded_workflows.py`:

- Swap the `code_literal_ranges` import for `literal_zone_ranges` and build
  `literal_ranges` from it.
- Update the comment above that line — it currently says "Compute code-literal ranges so
  references inside them remain inert"; it should name disabled regions as inert too.
- Leave the
  `default_with_feedback_parent_from_family_attach(..., fenced_ranges=literal_ranges)`
  call reading from the same variable. That argument is used by
  `_prompt_segment_at_offset()` to skip `---` separators inside literal zones, and a
  `---` inside a disabled region is inert prose that should be skipped for the same
  reason. Confirm this while implementing rather than assuming it.

### 2. Publish the run's own artifacts dir before deferred expansion

In `src/sase/axe/run_agent_runner_setup.py`, `expand_deferred_launch_xprompts()` already
receives `artifacts_dir`. Set `SASE_ARTIFACTS_DIR` to that value for the duration of the
expansion and restore the previous value (including "was unset") afterward, so
bare-`#fork` self-exclusion sees the correct directory.

Prefer this narrow, scoped approach over calling `publish_phase_env()` early.
`publish_phase_env()` also sets `SASE_AGENT_TIMESTAMP`, and `run_execution_loop()`
snapshots that variable into `LoopState.original_agent_timestamp` _before_ the loop
publishes it; setting it early would change that snapshot from unset/inherited to this
run's own timestamp. That is a behavior change unrelated to this bug and must not be
smuggled in here.

Document in the function docstring why the variable is scoped there.

### 3. Make the VCS tag scanners disabled-region aware

Change `find_vcs_workflow_tag_span()` in both `src/sase/xprompt/_parsing.py` and
`src/sase/xprompt/_parsing_vcs_tags.py` to use `literal_zone_ranges()`, and update both
docstrings to mention disabled regions.

While there, check whether the sibling non-span extractors in those modules (the plain
`search()`-based tag lookups that ignore literal ranges entirely) feed any launch
decision. If they do, bring them under the same rule; if they do not, leave them alone
and say so in the commit message. Do not expand this step into a general audit of every
`code_literal_ranges` caller — the TUI highlight and semantic-overlay callers are
presentation code and are intentionally out of scope.

## Testing

Add regression tests. Do not settle for unit tests that only assert the range helpers
directly — the defect was a wrong _choice_ of helper at a call site, so the tests must
exercise the call sites.

1. **Embedded workflow expansion (defect A).** Build a query with a real `#<workflow>`
   reference outside a disabled region and an argument-less mention of the same workflow
   name as prose inside a `%xprompts_enabled:false` / `%xprompts_enabled:true` region.
   Assert the real reference expands and the prose mention survives verbatim.
   `tests/ test_embedded_workflows_per_step.py` and `tests/test_disabled_regions.py` are
   the nearest homes; pick one and keep the new case with its neighbors.

2. **End-to-end shape of the actual failure.** Assert that a prompt shaped like the
   monitor follow-up — leading `#fork:<parent>` directive line, body wrapped in a
   disabled region, a bare `#fork` token in the body prose — yields exactly one resolved
   fork parent, and that it is the named one. `tests/ test_fork_workflow.py` is the
   natural home. This is the test that would have caught the reported failure.

3. **Self-exclusion (defect B).** Cover `expand_deferred_launch_xprompts()` with a bare
   `#fork`, a named `agent_meta.json` already written for the run being launched, and at
   least one older named agent that does have chat history. Assert the older agent is
   chosen, not the run itself. Include a case where `SASE_ARTIFACTS_DIR` is pre-set to
   an unrelated directory to prove the inherited-value path is also corrected, and
   assert the variable is restored to its prior state after the call.
   `tests/test_run_agent_runner_setup.py` is the natural home.

4. **VCS tag scanning (defect C).** Assert a `#gh:<repo>`-shaped token inside a disabled
   region is not returned as the workflow tag, while one outside still is. Cover both
   module copies of `find_vcs_workflow_tag_span()`.

Then run verification. `just check` is the floor; this change touches prompt parsing and
the agent launch path, which are broad, so run `just check-full` through `/sase_monitor`
before landing:

```bash
sase monitor start --command 'just check-full' \
  --start-status TESTING --stop-status TESTED --next '...'
```

Both call sites in step 3 and step 1 are reached by a large number of existing tests;
expect some to need updating if they asserted the old (buggy) behavior. If any existing
test explicitly asserted that a reference inside a disabled region _does_ expand, stop
and report it rather than deleting it — that would mean the current behavior is
load-bearing somewhere and this diagnosis needs revisiting.

## Out of scope

- Reference expansion order (`reversed(refs)`) — correct as-is.
- The 2026-08-15 `02i--7` failure, which shares the "Invalid fork parents" message but
  has a different cause (a monitor-shell parent with no chat history) and was already
  addressed by the sase-t8.1 work.
- Any change to how the monitor composes its follow-up prompt. The monitor already wraps
  its body in a disabled region correctly; it was let down by the expander, not the
  other way around.
- Broadening `literal_zone_ranges` adoption to presentation-only callers.
