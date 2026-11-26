---
tier: tale
title: Fix busy lookup progress logging and finish sase-1vh
goal:
  Every busy lookup reports its first retry reliably, and the verified epic is closed.
size: small
proposed_by: bbugyi200.athena.sase-1vh.land
proposed_with: codex/gpt-6-astra
bead: sase-1vh
status: done
---

- **PARENT:**
  [202611/launch_lookup_busy_resilience.md](https://github.com/sase-org/sase--plans/blob/main/202611/launch_lookup_busy_resilience.md)
- **BEAD:**
  [sase-1vh](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1vh/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1vh.land](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1vh.land.md)
- **COMMITS:**
  - [47e48e4](https://github.com/sase-org/sase--plans/commit/47e48e4ccc22615a8f72c4e3c92392fd6ef04be1)
    — docs(plans): mark busy lookup plan complete

# Fix busy lookup progress logging and finish sase-1vh

This is the remaining work for epic `sase-1vh`, whose approved design is
`plan:202611/launch_lookup_busy_resilience.md`. Implement this tale directly and finish
that epic's landing in the same turn as the final code changes. No later land agent
resumes this tale. Closing must not depend on this work's commit SHA, push, or CI
result.

## Verified baseline

The lander read the epic and every child note and reviewed the actual code and commits:
sase `47605d5cb53a` (patience), `6574797e4031` (typed failures), and sase-core
`2efa9f6b97fe` (SQLite diagnostics). All three phases are closed. The third phase's note
about leaving changes uncommitted was superseded by its host-created commit. The
selected `patience_budget = s60` is implemented.

The retry helper releases the process-local index lock before sleeping, retries only
busy-incomplete results, preserves fail-closed results, and leaves ordinary callers at
one attempt. Runner policy covers bootstrap and the refreshed pass. Typed lookup
failures are excluded from provider retry and receive appropriate hints. Rust busy
diagnostics retain classification while adding extended codes. The original approved
plan explicitly needs no pin change for that diagnostic-only change to existing bindings
and wire shapes. The lean waiter uses marker polling and a separate wait-index probe,
not these targeted lookup facades.

Integration review covered unrelated commits since epic creation: sase `ebe2eb17c4`
(question action split), `758745cb40` (question-copy cues), and sase-core `12334fe3`
(release cadence). None needs the new lookup policy or duplicates it. Remote master tips
matched the reviewed sase and core commits. This is a master workflow, with no separate
PR base. Recheck subsequent drift before closing, concentrating on lookup, retry, and
runner changes.

The seven focused test files listed below passed together: **108 tests**. No
`--epic-symbol` entries were found for `sase-1vh`. Its current parent is null. The
lander recorded full audit evidence and follow-up triage on `sase-1vh`.

## Remaining defect

`src/sase/axe/run_agent_runner.py::_runner_lookup_retry_logger` stores one
`last_progress_log` in units of `event.seconds_waited`. That field starts over for each
lookup, but the logger closure lives for the runner's lifetime.

A deterministic reproduction calls the same logger with these events:

1. Artifact index: attempt 1 at 0 seconds, next delay 0.25.
2. Artifact index: progress at 10 seconds.
3. Artifact index: recovered at 10.25 seconds.
4. Dismissed-bundle index: attempt 1 at 0 seconds, next delay 0.25.
5. Dismissed-bundle index: recovered at 0.25 seconds.

The current output omits event 4 entirely. Reusing the artifact-index label for the
second lookup fails too. Later progress can be delayed by the previous lookup's elapsed
time. This violates the original requirement to announce each lookup's first busy retry.

## Implementation

1. Fix the reporter's clock and initial-event handling. Measure the progress throttle
   with a monotonically increasing clock rather than elapsed time belonging to different
   operations. Allow deterministic clock injection into the private reporter factory.
   Always announce a nonterminal `attempt == 1`; throttle subsequent progress to at most
   one line per ten seconds. Terminal recovery/give-up events must always emit. Preserve
   the existing concise formatting, diagnostic normalization, 60-second patience policy,
   and callback exception isolation. Keep this as runner presentation code; no event
   wire schema or new backend behavior is needed.
2. Extend `tests/axe/test_run_agent_runner_swap_start.py` (or a focused logger test
   module if clearer). Use an injected clock with no wall-clock sleeps. Cover
   consecutive retries for the same label and for different labels, after both recovery
   and give-up; every new lookup must print its initial busy line. Verify the ten-second
   progress boundary and unconditional terminal messages. Exercise interleaved events to
   ensure one lookup's elapsed time cannot silence another lookup's first line. Adapt
   the existing stable-output test to the clock injection. Preserve the original
   helper/facade behavior tests.

## Governing decisions and verification

- `decisions:rust-core-required`: lookup, ranking, and bounded recovery stay in Rust.
  This repair only changes Python runner log presentation.
- `decisions:check-full-is-explicit`: use `sase tool run check`, which runs
  `just check`. Do not run `just check-full`.

Run focused tests first:

```sh
.venv/bin/python -m pytest -q \
  tests/test_index_lookup_patience.py \
  tests/test_agent_names_indexed_lookup.py \
  tests/test_agent_names_group_indexed_lookup.py \
  tests/axe/test_run_agent_runner_swap_start.py \
  tests/test_llm_provider_retry_config.py \
  tests/axe/test_runner_failure_facts.py \
  tests/axe/test_run_agent_exec_attempts_integration.py
```

Then run formatting as needed, `sase tool run check`, and `git diff --check`. If the
named check escalates, follow `/sase_monitor` using the same run ID and give its
successor this closeout responsibility. Do not rerun or lose the run. No Rust edits or
screenshot changes are anticipated.

## Final step: finish the epic landing

After implementing and verifying the repair, complete all of these actions:

1. Re-read `sase-1vh` and its three children (`sase-1vh.1`, `.2`, `.3`) with
   `sase bead read <id> -r "Recheck readiness for final epic closeout"`. Review any new
   notes, confirm every descendant is complete, and read the linked plan through
   `sase artifact read plan:202611/launch_lookup_busy_resilience.md`. Supply a reason
   for that audited artifact read. Recheck post-audit commits for integration needs.
   Finish any newly discovered epic-caused defect first.
2. Carry forward the lander's triage: no `PROPOSED FOLLOW-UP:` entries existed; no
   distinct unrelated actionable follow-up was confirmed or filed. The historical
   unnamed phase-.2 test failures were not filed without identities or reproduction;
   phase .1 later reported a passing check, and the lander's 108 focused tests passed.
   The logger defect remains epic work. Use `/sase_new_task` only if new distinct
   unrelated work is actually discovered, and include every resulting outcome or
   declined proposal in the close note.
3. Run `sase bead epic-symbols sase-1vh`. Resolve every listed exemption by wiring,
   privatizing, a valid non-test pragma, or deleting according to the Symvision policy.
   Re-key only when a still-open later bead actually needs the exemption. Do not leave
   any entry keyed to this epic or its closed phases.
4. Close normally with
   `sase bead close sase-1vh --note "<verification and triage outcomes>"`. Describe the
   three reviewed phase commits, 60-second selection, fail-closed and provider-retry
   behavior, Rust diagnostics, unrelated-commit integration review, repaired logging,
   final checks, symbol cleanup, and triage outcomes. If stale symbols reject the close,
   clean them and retry. If named phases are incomplete, finish or reopen them. Never
   use `--force` just to obtain a close or advance a successful nested landing. A
   deliberately canceled/superseded scope alone may use
   `--force --reason ... --resolution canceled|superseded`.
5. Run `just symvision` after the close. Open the plans repository with `/sase_repo` and
   set `status: done` in the frontmatter of `202611/launch_lookup_busy_resilience.md`
   under the printed plans path. Preserve the rest of that plan and include this sidecar
   edit in finalization.
6. Run `sase bead read sase-1vh -r "Need the parent link"` after closing. The audited
   parent is null, so no ancestor close is expected. If the relationship changed,
   inspect the new parent: a phase can close normally only after its required work is
   verified, leaving its epic to its waiting land agent; a plan requires reviewing its
   prior landing note, all descendants/notes, linked plan, and post-child drift before
   normal close, symbol cleanup, `just symvision`, and marking its plan done. Repeat
   only through complete directly parented plans; note and report the first incomplete
   or ambiguous ancestor.
7. Finish through `/sase_final`, committing the code and changed plan sidecar. Do not
   wait for these commits to exist before closing the epic.
