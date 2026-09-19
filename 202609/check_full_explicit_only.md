---
tier: tale
title: Run just check-full only when explicitly instructed
goal:
  SASE agents invoke `just check-full` only when the current prompt, user, or assigned
  bead explicitly instructs them to (typically to repair a CI failure). Epic landers
  follow the same rule and no longer author `%q(w=2.0)`. A `just check` pass with a
  `just check-full` failure is a test-infrastructure bug, out of scope for the current
  agent.
size: medium
proposed_by: bbugyi200.apollo.0w
create_time: 2026-09-19 10:43:32
status: wip
---

# Plan: Run `just check-full` only when explicitly instructed

## Outcome

After this tale:

1. The only verification recipe a SASE agent runs on its own is `just check`.
2. `just check-full` runs only when the current prompt, the user, or the assigned bead
   **explicitly** names that command (the intended case is repairing a CI failure).
3. Epic land agents follow the same rule. The bundled `bd/land_epic` xprompt no longer
   authors `%q(w=2.0)`, so a lander claims the default `1.0` runner-capacity unit and
   `sase bead work --capacity 1` no longer raises the land segment to
   `%queue(capacity=2)`.
4. If a change passes `just check` and would fail `just check-full`, that gap is a test
   infrastructure bug. The current agent files it through `/sase_new_task` and does not
   treat it as remaining product work.
5. `just check` may still escalate **internally** to the governed full test lane when
   `tools/select_tests` fires a broadening, serial-budget, or ratio rule. That is the
   recipe doing its job. It is not the agent running `just check-full`. Do not change
   those escalation rules in this tale.

## Why this is a medium tale

One follow-up coding agent can land the policy, the lander queue claim, the tests, and
the docs together. The pieces are coupled: dropping the lander's double-weight claim
without rewriting the instructions leaves landers still running `just check-full`, and
rewriting the instructions without dropping the weight leaves every epic lander claiming
two runner-capacity units for work it is no longer supposed to do.

This is not an epic. There is no independent phase a second agent could complete without
the rest of the change.

`medium` rather than `small` because the work spans an immutable decision-record
supersession, a bundled xprompt, several tests that pin the lander's weight-2 raise, and
the agent-facing docs/skills that currently teach `just check-full` as the landing gate.

## Current behavior

Agents run `just check-full` more than they should because several always-reachable
instruction surfaces still tell them to.

### Agent default vs landing gate

`sase/memory/decisions/two-speed-verification.md` (accepted 2026-08-05) claims:

> `just check` is the agent default. `just check-full` is reserved for landing,
> broadening changes, and CI.

That claim is inlined into every generated instruction file (`AGENTS.md`, `CLAUDE.md`,
and the other provider shims) as decision 13.

`sase/memory/lint_and_test.md` is the required-read when an agent changes a tracked file
in this repo. It currently says to run `just check-full` instead of `just check`:

- before landing an epic's combined tree
- when the change touches the broadening set
- whenever `just check`'s scoped run escalated or reported an unusual selection

The Justfile comments above the `check` and `check-full` recipes repeat the same three
triggers. `docs/development.md` does too.

### Why landers hog two capacity units

The bundled lander in `src/sase/default_config.yml` (`bd/land_epic`) begins with
`%q(w=2.0)`. That is a queue **weight** of `2.0`, not an authored `%queue(capacity=2)`.
The effect the user named ("default capacity of 2") is the combination of that weight
with the raise in `src/sase/bead/work_queue_capacity.py`:

```text
segment_capacity = max(N, ceil(authored_weight))
```

So `sase bead work --capacity 1` renders `%queue(capacity=1)` on phases and
`%queue(capacity=2)` on the lander, and a lander without `--capacity` still claims two
runner-capacity units once its waits resolve (`docs/troubleshooting/runner-slots.md`).

`plan:202609/bead_work_capacity_segment_weight.md` (sase-zu.5) introduced that raise
**because** the lander authored `%q(w=2.0)`. It explicitly rejected dropping the weight:

> The weight is a deliberate resource claim from the weighted-queue rollout, and
> removing it would change admission for every epic launch, not just `--capacity` ones.

This tale reopens that rejected alternative. The resource claim existed so a lander
could run the expensive `just check-full` landing gate without stacking it onto a fully
loaded host. Once landers stop running that command, the claim has no remaining job.

The lander prompt itself never names `just check-full`. It says "verify the epic is
truly complete". Combined with lint_and_test's "before landing" sentence, that is enough
for landers to start a 45-minute `just check-full` monitor on every epic.

### What `just check-full` does that even an escalated `just check` does not

Leave the recipes' stages alone. The policy change is which recipe agents invoke.

| Stage                                        | `just check`                                                       | `just check-full`                                           |
| -------------------------------------------- | ------------------------------------------------------------------ | ----------------------------------------------------------- |
| Whole-repo lint / validate / committed plans | yes                                                                | yes                                                         |
| Tests                                        | `just test-scoped` (may escalate to the governed `just test` lane) | `just test-cost` (always the full suite, with cost budgets) |
| Flake-baseline gate                          | no                                                                 | `just selection-health --fail-on-new-flake`                 |
| Local TUI screenshot update                  | no                                                                 | `just fix-tui-screenshots`                                  |

A scoped false negative, a test-cost budget miss, a new flake-baseline node, or a
screenshot golden that only the exhaustive lane would rewrite is therefore a test
infrastructure bug relative to the agent's assigned work. CI remains the backstop
(`decisions:ci-two-speed-split`): Master Gate plus scheduled Full CI.

## New policy (write this into the new decision and into lint_and_test)

**Claim.** `just check` is the only verification recipe a SASE agent runs unless the
current prompt, the user, or the assigned bead explicitly names `just check-full`. Epic
landers are not an exception. A `just check` pass with a `just check-full` failure is a
test-infrastructure bug, out of scope for the current agent; file it through
`/sase_new_task` and do not take it as remaining product or landing work. `just check`
may still escalate internally to the full test suite; that is rare by design and is not
the agent choosing `just check-full`.

**When `just check-full` is allowed.** Only an explicit instruction. The intended case
is "CI is red on a check-full-only gate; fix that failure." An agent that was told to
run it still uses `/sase_monitor` with the `verify` profile (`TESTING` / `TESTED`),
never inline.

**What does not count as explicit instruction.** Landing an epic. Touching the
broadening set. Seeing `just check` escalate or print an unusual selection. "Being
careful." A sibling agent's example. The monitor skill's old canonical snippet.

**Why.** The host-capacity measurements in `[[decisions/two-speed-verification]]` still
hold: the full suite can consume a quarter to a half of the machine continuously. Using
that lane as a default landing gate also converts test-infrastructure failures (flake
baseline, test-cost budgets, full-lane-only flakes, screenshot drift) into blocking epic
work. CI already runs the exhaustive non-visual suite and a check-only visual job.

**Cost.** A scoped false negative waits for CI rather than for the next lander. Flake
baseline, test-cost, and screenshot-update gates of `just check-full` are no longer an
agent default. Screenshot goldens that a change actually dirties are still the changing
agent's job via `just fix-tui-screenshots` when TUI output or snapshot coverage changed
(existing lint_and_test / tui_screenshot rule).

**Reopens when.** Selection-health shows the heuristic is materially wrong in practice,
or check-full-only CI failures become frequent enough that the detection lag is no
longer acceptable.

Keep the two-speed **mechanism**. This tale changes who is allowed to start the slow
lane, not whether the slow lane exists.

## Implementation

Do the memory edits first, then the lander xprompt and tests, then the remaining
agent-facing docs and the monitor skill source. Run `sase memory init` last among the
memory edits so the decisions roster, `AGENTS.md`, and provider shims pick up the new
record and the supersession marker together.

### 1. Supersede the landing-gate decision

Follow `sase/memory/decisions.md` and the convention in
`plan:202609/memory_supersession_annotation.md`. Do not rewrite the old record's
argument.

**New strand** `sase/memory/decisions/check-full-is-explicit.md`:

```yaml
---
keyword: Check-Full Is Explicit-Only
aliases: [agents run check not check-full, landing does not run check-full]
summary:
  just check is the only agent-initiated verification recipe; just check-full runs only
  when explicitly instructed, typically to repair a CI failure.
metadata:
  status: accepted
  decided: 2026-09-19
---
```

Body uses the Claim / Why / Cost / Reopens-when shape of the other decision strands.
Link `[[decisions/two-speed-verification]]` for the host-capacity measurements. State
that epic landers follow the same rule and therefore no longer author `%q(w=2.0)`.

**Old strand** `sase/memory/decisions/two-speed-verification.md`:

- Set `metadata.status: superseded` and
  `metadata.superseded_by: decisions/check-full-is-explicit`.
- Keep `decided: 2026-08-05`.
- Add a short superseded mark at the top of the body with a
  `[[decisions/check-full-is-explicit]]` back-link. Do not reword, delete, or soften the
  rest of the accepted body.

Do not hand-edit the managed `<!-- sase:strands -->` roster in
`sase/memory/decisions.md`. `sase memory init` regenerates it. The old bullet must keep
appearing, with the `*[superseded by check-full-is-explicit]*` marker, plus a new roster
bullet for the new record.

Leave `[[decisions/two-speed-verification]]` links in
`sase/memory/decisions/ci-two-speed-split.md` and
`sase/memory/decisions/record-before-admit.md` as they are. Those records cite the
host-capacity rationale, which the old strand still holds.

### 2. Rewrite `sase/memory/lint_and_test.md`

This is the note every agent must read before finishing a turn that changed files. It is
the highest-leverage instruction change.

Keep the command cheat-sheet, the `just check` description, `just fix` before a verify
monitor, the ephemeral-workspace `just install` warning, the PNG snapshot section, and
the `[[symvision.md]]` pointer.

Replace the "Two-Speed Verification" section so that it:

- Requires `just check` after file changes in this repo (unchanged).
- Forbids `just check-full` unless the current prompt, user, or assigned bead explicitly
  names that command. Give the CI-failure example.
- States that landing, the broadening set, and a scoped escalation are **not** reasons
  to run `just check-full`. `just check` already escalates internally when the selector
  cannot trust the closure.
- States that a `just check` pass / `just check-full` failure is a test-infrastructure
  bug, out of scope; file it through `/sase_new_task`.
- Still requires `/sase_monitor` with `TESTING` / `TESTED` **when** `just check-full` is
  explicitly requested, because it routinely outruns a turn.
- Points at `[[decisions/check-full-is-explicit]]` instead of
  `[[decisions/two-speed-verification]]`.

In the PNG section, keep the factual sentence that local `just check-full` runs the
update form. Do not tell agents to use that as their default golden-update path; the
existing "when your work changes rendered TUI output, run `just fix-tui-screenshots`
explicitly" rule stays the agent path.

Update `sase/memory/tui_screenshot.md` so it no longer says "read lint_and_test for when
agents must run `just check` versus `just check-full`" as if those were two default
lanes. Point at the new rule.

### 3. Drop the lander's `%q(w=2.0)`

In `src/sase/default_config.yml`, delete the `%q(w=2.0)` line from `bd/land_epic`. Leave
the rest of the lander body.

Add one short sentence to the lander prompt, near the top after the role line, so
"verify the epic is truly complete" cannot be read as "run the exhaustive suite":

> Do not run `just check-full` unless this prompt or the user explicitly tells you to.
> File-change verification is `just check`. A `just check` pass with a `just check-full`
> failure is a test-infrastructure bug, not remaining epic work.

Do not add `%queue` / `%q` of any kind to the bundled lander. Phase and task workers
already author neither a priority nor a weight; the lander joins them.

Keep the raise-to-`ceil(weight)` machinery in `src/sase/bead/work_queue_capacity.py`. It
must still lift `--capacity N` for **any** xprompt that authors a weight greater than
`N` (overrides, future prompts, the existing `%q(w=2.5)` test). Rewrite the module
docstring so it no longer says built-in land xprompts author `%q(w=2.0)`.

### 4. Tests for the lander claim

The builtin body is pinned and the raise is pinned. Update both, and keep coverage that
an **override** with weight `2.0` still raises.

| File                                            | What changes                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| ----------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `tests/test_bead_xprompt_tags.py`               | Replace `test_builtin_land_prompt_requests_double_capacity`. The builtin body must not contain `%q(w=2.0)` / `%queue(weight=`. After `extract_prompt_directives`, `queue_weight` is unset (default `1.0` at admission) and `queue_weight_explicit` is false. Keep asserting the cleaned body still starts with the lander role line.                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| `tests/test_bead/conftest.py`                   | Change `cli_work_xprompt_catalog`'s default `land_content` from `"%q(w=2.0)\nLand the epic."` to a body with no queue directive, matching the new builtin. Tests that need a heavy lander pass `land_content` explicitly.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| `tests/test_bead/test_work_queue_capacity.py`   | `test_flag_on_capacity_1_raises_only_land` currently expects land `2` and a raised entry. With the builtin at weight 1, `--capacity 1` must leave both phase and land at `1` with `raised == ()`. Rename it. Keep `test_fractional_authored_weight_rounds_up` and the conflict test (those use extra xprompts). Add or keep a case that a land override of `%q(w=2.0)` with `--capacity 1` still raises land to `2`. `test_large_phase_probe_includes_plan_and_still_floors_land` should still prove a `#plan` large-phase probe works; it must no longer expect land to be floored to `2`. `test_runner_accepts_every_capacity_1_segment_after_floor` becomes a plain `--capacity 1` admissibility check: every segment, including land, expands and extracts with `capacity=1`. |
| `tests/test_bead/test_cli_work_epic_launch.py`  | The `capacity=1` launch that currently asserts `land.count("%queue(capacity=2)") == 1` must assert `capacity=1` on the land segment too.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| `tests/test_bead/test_cli_work_epic_dry_run.py` | Same for the dry-run render, and drop the assertion that the summary prints `raised to 2 (queue weight 2.0)` for the builtin lander. A dry-run with an explicit heavy land override may still show that line.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| `tests/test_bead/test_work_rendering.py`        | Leave tests that pass an explicit `segment_capacity` map including land `2`, and the test that appends `%q(w=2.0)` after render to check composition. Those are mapping/composition tests, not builtin-body tests.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |

Search for remaining `%q(w=2.0)` and `queue(capacity=2)` assertions that assume the
**builtin** lander before finishing. Fixture strings in ACE PNG snapshots that say
"Full-suite verification before landing" are unrelated display fixtures; do not touch
them.

### 5. Agent-facing docs, Justfile comments, and the monitor skill

Update every surface that currently tells an **agent** to run `just check-full` as a
default landing / broadening / escalation step. Do not delete the recipe, the tool
catalog entry, or human contributor guidance that the exhaustive lane exists.

**Must change (agents read these as instructions):**

- `Justfile` comments above `check` and `check-full`. Today they say "Run
  `just check-full` instead before landing an epic's combined tree, when the change
  touches the broadening set, or whenever the scoped run escalated". Replace with: the
  agent default is `just check`; `just check-full` is exhaustive and is not an agent
  default; agents run it only when explicitly instructed (typically a CI failure). Keep
  the factual description of what each recipe runs. `tests/test_justfile_lint.py` pins
  recipe stages, not these comments, but do not drift the non-test gates.
- `docs/development.md` — the paragraph starting "Run `just check-full` — every lint
  gate..." that lists landing / broadening / unusual selection. Restate the new agent
  rule. Keep the factual description of what the exhaustive lane includes (test-cost,
  flake baseline, screenshot update) and that CI is the backstop. The flake-baseline
  section can stay as documentation of that gate; it must not say landers have to run
  it.
- `src/sase/xprompts/skills/sase_monitor.md` — the canonical example currently is
  `just check-full` with `--next 'Fix anything just check-full reported...'`. Change the
  canonical example to `just check`. Keep a sentence that **when** `just check-full` is
  explicitly requested, the same `verify` profile and `TESTING` / `TESTED` pair apply,
  never inline. Update `tests/main/test_init_skills_sources.py`, which pins
  `"-- just check-full"` and `"--next 'Fix anything just check-full reported"`.
- `docs/monitors.md` — align the first worked example with the skill (`just check`).
  `just check-full` may remain later as an example of a long command, labeled as
  explicit-only.
- `docs/troubleshooting/runner-slots.md` — "The bundled epic lander authors `%q(w=2.0)`,
  so it claims two capacity units..." is false after step 3. State that bundled task,
  phase, and lander workers author no non-default weight. Overrides may still author
  one, and the `--capacity` raise still applies to those.
- `docs/xprompt.md` — the bundled-workers paragraph already says they do not author a
  priority wait. Add that they also do not author a non-default queue weight.

**Light touch (humans, not the agent default):**

- `CONTRIBUTING.md` and `README.md` currently say run `just check-full` before
  submitting. Keep that as **human** contributor guidance for the exhaustive local lane
  (it updates screenshot goldens). Add one sentence that SASE agents use `just check`
  unless explicitly instructed to run `just check-full`. Do not turn the human docs into
  a second copy of lint_and_test.

**Leave alone unless a sentence is now false:**

- `sase/sase.yml` `check-full` tool entry (description can stay "exhaustive repository
  check").
- CLI `--help` examples in `src/sase/main/parser_monitor.py` that use `just check-full`
  as a long command.
- `docs/cli.md`, `docs/getting_started.md`, `docs/rust_backend.md`,
  `docs/perf_runbook.md` mentions that describe the recipe or a long wait.
- `src/sase/xprompts/skills/sase_final.md`, which already uses `just check`.
- Epic `sase-135` (named tools / ToolRun ledger). It records `just check-full` runs; it
  does not instruct agents to start them.
- Selection, flake-baseline, and test-cost **machinery**. Out of scope.

Do **not** run `sase skill init` from this workspace. Per
`sase/memory/generated_skills.md`, a chezmoi deploy from an unlanded tree is refused or
reverts other agents' deployments. Edit the skill **source**; deployment is a post-land
step from a clean merged tree.

### 6. Regenerate memory

After the memory files in steps 1–2 are saved, run `sase memory init`. That regenerates
`AGENTS.md`, the provider shims, `sase/memory/decisions.md`'s roster, and
`sase/memory/README.md`. Do not hand-edit `AGENTS.md` or `CLAUDE.md`.

**Hazard:** `sase memory init` also writes the home root. Capture
`sase memory init --check --diff` before and after. Home-root changes beyond the
pre-existing baseline are unexpected; investigate them and keep unrelated chezmoi drift
out of this change.

`sase memory init --check` must be clean for the project root when you are done. The new
decision must appear in the generated DECISIONS roster, and the old
`two-speed-verification` bullet must carry the superseded marker.

## Out of scope

- Changing `just check`'s escalation rules, worker gears, or selector heuristic.
- Changing what `just check-full` runs.
- Making `just check` run test-cost, flake baseline, or screenshot update.
- Dropping the `--capacity` raise-to-`ceil(weight)` translation.
- Forbidding humans or CI from running `just check-full`.
- Filing new task beads for historical check-full-only flakes this policy will stop
  converting into lander work. Existing flake/ci tasks stay where they are.
- Deploying generated skills to chezmoi.

## Verification

- `just install` if this workspace venv is stale, then `just check`. Do **not** run
  `just check-full` for this tale unless the user or a CI failure explicitly asks.
- `sase memory init --check` is clean for the project root.
- `sase memory read decisions:check-full-is-explicit decisions:two-speed-verification -r "confirm supersession and new claim"`
  shows the new claim and the old record's superseded marker plus back-link.
- Generated `AGENTS.md` decision 13 (or the new numbering after init) no longer tells
  agents that `just check-full` gates landing.
- `python -c` expanding `#bd/land_epic:sase-demo` (or
  `sase xprompt expand --trace '#bd/land_epic:sase-demo'`) yields no `%q(w=2.0)` and no
  `%queue(weight=`.
- `sase bead work <open epic> -c 1 --dry-run` renders `%queue(capacity=1)` on **both**
  phases and the land segment, with no `raised to 2` summary line for the builtin
  lander.
- Targeted pytest: `tests/test_bead_xprompt_tags.py`,
  `tests/test_bead/test_work_queue_capacity.py`,
  `tests/test_bead/test_cli_work_epic_launch.py`,
  `tests/test_bead/test_cli_work_epic_dry_run.py`,
  `tests/main/test_init_skills_sources.py`. )
