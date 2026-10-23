---
title: Prove proc-backed monitor output delivery and finish sase-1nk
tier: tale
goal:
  A proc settlement regression proves the real final output reaches the follow-up
  prompt, and sase-1nk is verified and closed with its original plan marked done.
size: small
proposed_by: bbugyi200.athena.sase-1nk.land
proposed_with: codex/gpt-6-astra
bead: sase-1nk
status: done
---

- **PARENT:**
  [202610/monitor_followup_output_and_host_path.md](https://github.com/sase-org/sase--plans/blob/main/202610/monitor_followup_output_and_host_path.md)
- **BEAD:**
  [sase-1nk](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1nk/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1nk.land](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1nk.land.md)
- **COMMITS:**
  - [7552ebe](https://github.com/sase-org/sase/commit/7552ebe590c07d4f9bf5fd62bf583a30d79c31b3)
    — test(monitor): prove proc-backed follow-up prompts show the output tail

# Prove proc-backed monitor output delivery and finish sase-1nk

## Remaining work

Epic `sase-1nk` is implemented but lacks one acceptance test explicitly required by its
approved plan, `plan:202610/monitor_followup_output_and_host_path.md`: an end-to-end
regression through proc-backed settlement that checks the actual composed follow-up
prompt for the final command output. Add that regression, fix any epic-owned defect it
exposes, and finish the epic's landing in this coding turn. This tale has no land agent
or later landing continuation.

The landing audit read the epic, both children and every note, the approved plan, the
implementation and its callers, and both epic commits:

- `def4c66ce3` (`sase-1nk.1`) assigns retained ranges without a retention snapshot,
  supports legacy null segment ranges, and selects retained output from the tail.
- `ed321c5434` (`sase-1nk.2`) stamps the runner PATH and restores it before nested agent
  launch, preserving explicit `extra_env` PATH overrides.

Both children are closed. The approved decisions remain final:
`path_merge_order = host_first` and `monitored_command_path = no`. Preserve these;
monitored commands keep the spawning shell's PATH. No additional PATH feature is needed.
The runner boot, env scrubbers, provider resolution, missing snapshot, and explicit PATH
override already have focused tests.

At the audit's refreshed `origin/master` (`ed321c5434`), the two epic commits were the
only commits since epic creation (2026-10-23 09:29:10 EDT) and since the first epic
commit. There was no intervening integration work. `sase bead epic-symbols sase-1nk`
listed no entries. The epic has no `parent_bead` link.

## Add the missing regression

1. Read `sase bead read sase-1nk -r "Need the landing audit and remaining scope"` and
   use
   `sase artifact read plan:202610/monitor_followup_output_and_host_path.md "Need the approved monitor acceptance criteria"`.
   Read the testing memory and prepare this checkout's environment with
   `just install-venv` if needed. The lander's focused pytest invocation stopped during
   collection because its local venv could not import the required `sase_core_rs`; it
   produced no test verdict. Do not bypass the binding requirement. Use `/sase_monitor`
   if setup is long.
2. Add a focused regression under `tests/monitor/`. Existing references are
   `test_monitor_proc_settlement.py`, `_followup_fixtures.py`, and
   `test_monitor_followup_rollout.py`. Use the real proc-backed
   `settle_monitor_artifacts` path, persisted monitor result, evidence selection,
   `launch_followup_agent`, and `compose_followup_prompt`. Capture the prompt at the
   final agent process-spawn boundary; do not mock the functions that freeze, read,
   select, or render the output. Stub only external process launch/acknowledgment and
   unrelated lifecycle effects as needed. No real LLM launch is required.
3. Give the monitor output longer than `TOTAL_RAW_EXCERPT_MAX_BYTES`, with an early
   sentinel, many ordinary lines, and unmistakable final failure/summary lines. Exercise
   the failed command outcome with the ordinary raw-tail selection. Assert that the
   resulting `Last N lines of output` section contains the final lines, omits the early
   sentinel, and contains no `retained output gap` notice for this complete log. Ensure
   a real frozen result is loaded so an in-memory capture fallback cannot accidentally
   make the test pass. Keep the test deterministic, isolated from the enclosing agent,
   and bounded in runtime.
4. Run the regression and the existing diagnostics/tail, launch-env, env-hygiene, and
   runner-bootstrap tests. The regression must detect the original null-range and
   head-excerpt behavior, not merely assert a successful launch. If it exposes an
   epic-owned defect, repair it and retain the behavioral regression. Shared
   backend/domain changes, if actually required, belong in `sase-core` through
   `/sase_repo`; do not introduce a Python core fallback.

## Verify and account for drift

Refresh `origin/master` and inspect any newly landed commits since `ed321c5434` for
changes that should use or conflict with this epic. Integrate relevant changes. Run
formatting as needed and `sase tool run check` (the recorded `just check` recipe). Do
not run `just check-full`. If verification needs a monitor, its successor must finish
the closeout below; do not use a prepared completion that would skip these explicit
remaining steps.

Follow-up triage is already complete and recorded in the epic's landing note:

- `sase-1nk.1` note #1 and `sase-1nk.2` note #1 propose the same TUI import-budget
  failure (`3529` modules versus strict `3527` cap). The second phase reproduced it on
  clean base. `/sase_new_task` searches, recent-task sweep, and active-epic review found
  existing ready task `sase-1ic`; the lander recorded corroboration there (+25). No new
  task was created, because both proposals duplicate that node and remediation. No
  credible active-epic causal link was established.
- The phase completion notes report that their other transient failures passed focused
  reruns and the wording/assertion issues were fixed. They contain no other
  `PROPOSED FOLLOW-UP:` entries.

Do not raise the TUI import cap as part of this tale. Classify verification failures
from evidence: epic-owned failures must be fixed; the independently established
unrelated import-budget issue remains with `sase-1ic`. Record actual check outcomes
honestly. Route any genuinely new unrelated issue through `/sase_new_task`, avoiding
duplicate reports when ToolRun triage already names an owner.

## Final step: complete this epic's closeout

Perform this step in the same turn as the final code. Do not order it after this tale's
future commit SHA, push, or CI result; the host commits after the turn.

1. Re-read the epic and both children to confirm all descendants remain complete and
   account for any new notes. Run `sase bead epic-symbols sase-1nk`. Resolve every entry
   by wiring up, privatizing, adding a justified non-test pragma, or deleting the symbol
   under the Symvision policy. Re-key an exemption only when a concrete still-open later
   bead genuinely needs it. Do not leave stale entries.
2. Close normally with `sase bead close sase-1nk --note "<verification>"`. Write a
   concrete note naming the two reviewed implementation commits, the new proc
   settlement-to-prompt regression and its result, both preserved decisions, the fresh
   drift review, actual focused/check results, symbol cleanup, and both import-budget
   proposals' disposition on `sase-1ic` including why new tasks were declined. Never use
   `--force` to make a successful landing advance. If a close is rejected for symbols or
   incomplete descendants, fix the stated issue and close normally; canceled/superseded
   work requires deliberate evidence and the corresponding resolution, not a workaround
   for successful work.
3. After closing, run `just symvision` and resolve any stale-whitelist failure.
4. Open the plan sidecar using `/sase_repo`
   (`sase repo open plans -r "Mark the verified monitor epic plan done"`), obey its
   instructions, and set `status: done` in the frontmatter of the audited original plan
   at `202610/monitor_followup_output_and_host_path.md` under that printed repo root.
   Include that sidecar edit in the host final declaration.
5. Run `sase bead read sase-1nk -r "Need the parent link"`. The lander confirmed there
   is no parent, so normal completion stops here. If that relationship has unexpectedly
   changed, inspect it before advancing: close a parent phase only after verifying its
   scope, leaving its epic to its lander; for a direct parent plan re-audit its prior
   landing note, every descendant and note, linked plan, readiness and post-child drift
   before any normal close/symbol check/plan status update. Stop at the first incomplete
   or ambiguous parent, note the blocker, and report it. Never force a successful nested
   landing.

Finish with `/sase_final` only after all commands have exited. Report the epic's actual
completion status and any remaining independently tracked limitation.
