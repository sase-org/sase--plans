---
tier: epic
title: Clear the five highest-impact open task beads
goal:
  Task beads sase-ll, sase-mv, sase-nk, sase-mw, and sase-mr are fixed, verified, noted,
  and closed, and the remaining "sase" task-bead backlog is either handed to a follow-up
  agent or reported to the user with every TASK NEEDS APPROVAL note consolidated.
phases:
  - id: monitor_lane
    title: Implicit lane resolution for in-agent `sase monitor start`
    description:
      "'Implicit lane resolution for in-agent sase monitor start' section: fix the
      implicit-lane derivation so an epic-phase agent can hand a command to
      /sase_monitor without an explicit --lane, closing task bead sase-ll."
    depends_on: []
    size: large
  - id: config_cache_flake
    title: The config-cache full-parallel-lane flake
    description:
      "'The config-cache full-parallel-lane flake' section: find the process-global
      config-state leak that reds
      test_owner_snapshot_reuses_parsed_overlay_until_token_changes only under the
      whole-suite parallel lane, closing task bead sase-mv."
    depends_on: []
    size: large
  - id: bead_stream_writes
    title: Per-stream bead event-store writes in sase-core
    description:
      "'Per-stream bead event-store writes in sase-core' section: make write_event_store
      rewrite only the streams whose events changed, closing task bead sase-mr."
    depends_on: []
    size: large
  - id: file_panel_tests
    title: File-panel assertions against the scroll-anchor seam
    description:
      "'File-panel assertions against the scroll-anchor seam' section: repoint six
      deterministically failing tests/test_file_panel.py assertions at the _update_body
      seam without weakening what they guard, closing task bead sase-nk."
    depends_on: []
    size: small
  - id: models_panel_png
    title: Models-panel jump PNG snapshot seam
    description:
      "'Models-panel jump PNG snapshot seam' section: repoint the stale
      build_alias_views monkeypatch at its owning module so the three Models-panel jump
      PNG snapshots render again, closing task bead sase-mw."
    depends_on: []
    size: small
proposed_by: bbugyi200.athena.04c
status: done
bead_id: sase-ns
create_time: 2026-09-09 19:51:57
---

- **PROMPT:**
  [prompts/202608/top_task_bead_sweep.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/top_task_bead_sweep.md)
- **BEAD:**
  [sase-ns](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ns/README.md)

# Plan

Five ready task beads on the `sase` project were selected as the highest-impact open
work after a full backlog triage. Each phase below owns exactly one bead. The phases are
independent — different subsystems, different files, no shared edits — so they may run
in parallel.

Triage that produced this selection also closed seven beads that no longer reproduced or
belonged elsewhere (`sase-n2`, `sase-mn`, `sase-nm`, `sase-n0`, `sase-ms`, `sase-no`,
`sase-ln`); do not re-open or re-litigate those.

## Rules Every Phase Must Follow

These apply to all five phases. They are not optional and they are not overridden by
anything a phase's own scope section says.

1. **Claim the bead first.** Before doing any implementation work, mark your bead
   in-progress:

   ```bash
   sase bead update <task-bead-id> -s in_progress
   ```

   Do this even though you were launched against a phase bead — the _task_ bead is a
   separate, standalone bead and the runner does not transition it for you.

2. **Never ask the user for approval directly.** If you conclude that your bead needs a
   decision from the project owner before the work can land, do **not** raise a question
   gate and do **not** stop and ask. Instead leave a note on the _task_ bead:

   ```bash
   sase bead note <task-bead-id> "TASK NEEDS APPROVAL: <what decision is needed, the concrete options with their tradeoffs, and your recommendation>"
   ```

   Then do every part of the bead that does not depend on that decision, and record what
   you left undone. Be lenient about this: an objectively correct fix — a stale test
   seam, a wrong lane lookup, a redundant file write — does **not** need approval.
   Reserve `TASK NEEDS APPROVAL` for genuine product or policy choices, for changes that
   would reverse a previously approved interface decision, and for cases where two
   defensible fixes would leave the codebase in materially different shapes.

3. **Leave a note on the bead explaining what you did.** Before closing, append a brief,
   concrete note:

   ```bash
   sase bead note <task-bead-id> "<what you changed, the commands you ran, and the evidence the defect is gone>"
   ```

   If you could **not** complete the work, the note must justify why — what you tried,
   what you found, and what the next agent should pick up. Say so plainly rather than
   overstating partial progress.

4. **Close the bead if you finished it.**

   ```bash
   sase bead close <task-bead-id> --note "<what you verified>"
   ```

   Only close a bead whose defect you actually verified gone. If you could not finish,
   leave the bead `in_progress` with the explanatory note from rule 3 and say so in your
   reply.

5. **Verify before you close.** Run `just install` first (workspaces are ephemeral and
   dependencies may have drifted), then `just check`. If your change touches the
   broadening set or `just check`'s scoped run escalates, hand `just check-full` to
   `/sase_monitor` rather than running it inline.

6. **Do not create new task beads for work you discover inside your own phase.** Append
   `PROPOSED FOLLOW-UP: <summary — detail>` to your phase bead instead and let the land
   agent route it.

7. **Cross-repo work goes through `/sase_repo`.** Do not clone, path-guess, or web-fetch
   another repository's contents.

## 'Implicit lane resolution for in-agent sase monitor start' section

Owns task bead **`sase-ll`** (large, 9 independent reporters, recurring the same day
this plan was written). This is the highest-impact bead in the backlog: it breaks the
documented `/sase_monitor` → `just check-full` handoff that every agent is told to use
before replying.

### What is wrong

Two related failure shapes, both reported by multiple independent agents:

- From an epic phase lane, `sase monitor start --command 'just check-full' …` with no
  explicit `--lane` fails before launch with
  `FamilyAttachError: Cannot create agent family 'sase-ku': resolved parent is named 'sase-ku.4'`
  — a _sibling_ phase's family member, not the caller's own lane. Passing
  `--lane sase-ku.10` explicitly gets past it.
- From other lanes the same implicit path fails with
  `no agent artifacts found for agent '<name>' in project 'gh_sase-org__sase'`, where
  the live agent list shows the artifact-bearing member under a suffixed name (for
  example the caller is `sase-m6.6.1.land` but the artifact-bearing member is
  `sase-m6.6.1.land--plan`). So the implicit resolution collapses to the family/base
  name rather than the exact calling member.

Read the bead's full `+1 EVIDENCE` section with `sase bead show sase-ll` — nine
reporters recorded exact commands and exact errors, and they are the acceptance
criteria.

### Scope

Start from the durable-lane derivation in `src/sase/monitor/start.py` (`resolve_lane()`
and the family-attach path that raises `FamilyAttachError`). Work out why, on an
epic-phase lane, the implicit lane resolves a family parent belonging to a sibling phase
rather than the caller's own lane, and why a suffixed member (`--plan`, `--code`,
`--mon`) is not found by base name.

Decide whether the implicit lane should come from the caller's own `SASE_AGENT_NAME` /
its own artifacts directory rather than from a lane-wide newest-member lookup, and
implement that. Cover the epic-phase-lane case _and_ the suffixed-member case with
regression tests.

If the correct answer turns out to be that `--lane` is genuinely required inside an epic
family, that is acceptable — but then it must be enforced with a clear, actionable error
that names the lane to pass, and `docs/monitors.md` plus the `sase_monitor` skill source
must be updated to match. Do not leave agents following documentation that cannot work.

Both failure shapes must be fixed or explicitly explained. If you fix only one, say
which one and why in the bead note.

## 'The config-cache full-parallel-lane flake' section

Owns task bead **`sase-mv`** (large, 14 independent reporters — the most-corroborated
bead in the store).

### What is wrong

`tests/test_config_cache.py::test_owner_snapshot_reuses_parsed_overlay_until_token_changes`
fails only when the _whole_ suite runs in parallel. It passes in isolation, and it
passes under `SASE_CONTENTION_REPEAT=3 just test-contention tests/test_config_cache.py`
(18 passed per repeat, zero red repeats). Narrow file-level contention is therefore
**not** sufficient evidence here — it is already green and proves nothing.

That evidence points at another test elsewhere in the suite poisoning process-global
config state. The node is fragile because it patches `sase.config.core.CONFIG_DIR`,
`Path.cwd`, and `_load_yaml_file`, then asserts _both_ identity reuse
(`first is second`) and an exact loader call count (`calls['count'] == 1`) against the
process-global memoized owner snapshot. Any other test that leaves the config cache
warm, invalidates it mid-test, or perturbs the change token (config mtime/size,
`machine_name_path`, selector stat) breaks one of those assertions.

### Scope

Find the poisoning test. The global-state leak detector built by closed phase bead
`sase-j7.2` is the natural instrument — locate it and use it rather than reinventing
one. Then either fix the leak at its source or isolate the global config cache around
this node.

Prefer fixing the leak over relaxing the assertions: the identity-reuse and call-count
assertions are what give this test its value. Do not change production caching behavior
to make a test pass.

Re-measure under the full parallel lane, and check the result against
`just selection-health` and `tests/reproducible_flake_baseline.txt`.

Nine other beads in the backlog describe the same "green alone, red under the full
parallel lane" class (`sase-md`, `sase-n5`, `sase-n6`, `sase-nc`, `sase-nd`, `sase-nf`,
`sase-ni`, `sase-nl`, `sase-nn`); `sase-nn` explicitly cites this bead's mechanism. If
your root cause plausibly explains any of them, record that as a `PROPOSED FOLLOW-UP:`
on your phase bead — do not silently close beads you did not verify, and do not expand
this phase's scope to chase them.

## 'Per-stream bead event-store writes in sase-core' section

Owns task bead **`sase-mr`** (large). Cross-repo: the change belongs in the sibling
`sase-core` repo, which you must open with `/sase_repo` before reading or editing it.

### What is wrong

`crates/sase_core/src/bead/jsonl.rs` — `write_event_store()` — re-serializes and
rewrites **every** stream file on every bead mutation, not just the streams whose events
changed. The store holds 850+ streams, so appending a single note to one bead rewrites
all of them. Two observed consequences:

1. **Blast radius.** A single event that is not round-trip stable wedges the _entire_
   store instead of one bead. That is what happened: one `resolution: null` event in
   `sase-mk.jsonl` failed to round-trip, and because every mutation re-emitted that
   stream, the append-only integrity guard refused every commit, push, and rollback
   store-wide, blocking epic launches for hours.
2. **Latency.** A failed `sase bead work` reported
   `slow_launch_stage phase_creation elapsed_ms=200901.7` and
   `dependency_creation elapsed_ms=170376.8` — each stage rewriting every file.

The writes are currently byte-identical, so git sees no spurious diff today. The cost
and the blast radius are the defect.

### Scope

Make `write_event_store` write only the streams whose events actually changed, keeping
`events/manifest.json` consistent and **without weakening the append-only guard**. Cover
it with a Rust test that mutates one stream in a multi-stream store and asserts only
that stream's file mtime/content changed.

This is an objective efficiency and blast-radius fix and does not need approval on its
own. **However**, leave a `TASK NEEDS APPROVAL` note on `sase-mr` if — and only if — you
find the fix requires changing the on-disk event-store format, the `manifest.json`
schema, or the semantics of the append-only integrity guard. Those are decisions about
critical shared infrastructure and belong to the project owner.

The bead store is live state that every agent depends on. Verify against a scratch copy
of a full multi-stream store before touching anything that runs against the real one,
and follow the Rust-core boundary rules: wire/API and tests land in `sase-core`, and any
Python caller or adapter here is updated to match.

## 'File-panel assertions against the scroll-anchor seam' section

Owns task bead **`sase-nk`** (small, 4 reporters). Verified failing on clean master
during triage.

### What is wrong

Six tests in `tests/test_file_panel.py` fail deterministically on clean master:
`test_render_static_file_result_renders_content`,
`test_display_linked_diff_renders_banner_and_raw_content`,
`test_live_diff_renders_all_lines_and_posts_line_count`,
`test_live_diff_timestamp_refresh_reuses_cached_body`,
`test_file_panel_pathological_cap_posts_explicit_range`, and
`test_linked_diff_full_rerender_keeps_banner`.

Reproduce with `.venv/bin/python -m pytest tests/test_file_panel.py -q -p no:randomly`.

Failure shape: `assert panel.update.called` → `AssertionError: assert False`, where
`panel` is the test's `MagicMock` body widget.

Root cause: commit `ce1ad41a1` (tale plan `202608/file_panel_scroll_anchor.md`) replaced
the ad-hoc save/restore calls with a scroll-anchor controller, so every body render now
funnels through the `_update_body` seam in
`src/sase/ace/tui/widgets/file_panel/_content.py` instead of calling `body.update()`
directly. The tests were never updated.

### Scope

Update the six assertions to observe the current `_update_body` seam, or make the mock
body satisfy it. Preserve what each test actually guards: rendered content, the
linked-diff banner, the live-diff line-count message, cached-body reuse on timestamp
refresh, and the pathological-cap explicit-range message.

Do **not** weaken the assertions to bare "was called" checks on a seam that no longer
proves content reached the widget, and do **not** change production behavior — the
scroll-anchor routing is the intended fix. This is a test-seam repair and does not need
approval.

## 'Models-panel jump PNG snapshot seam' section

Owns task bead **`sase-mw`** (small). Verified failing on clean master during triage.

### What is wrong

All three nodes in `tests/ace/tui/visual/test_ace_png_snapshots_models_panel_jump.py`
fail before rendering:

```
AttributeError: <module 'sase.ace.tui.modals.models_panel_providers'> has no attribute 'build_alias_views'
```

at `tests/ace/tui/visual/test_ace_png_snapshots_models_panel_jump.py:35`.
`_patch_alias_views` monkeypatches `models_panel_providers.build_alias_views`, but
commit `de83c802d` moved provider state to `models_panel_provider_state.py`; that module
imports `build_alias_views` and calls it, so the seam the test needs now lives there.

Run this suite through `just test-visual` (the default lanes deselect PNG snapshots).

### Scope

Repoint the stale test seam at its owning module, preserve the deterministic `AliasView`
fixtures, and verify the three PNG snapshots plus the Models-panel visual suite. This is
fallout from a concurrent module split, unrelated to the alias-migration work that
proposed it. It is a test-seam repair and does not need approval.

**Adjacent, in scope if trivial:**
`tests/ace/tui/visual/test_ace_png_snapshots_artifacts_files.py::test_artifacts_files_populated_png_snapshot`
fails the same way, now on `files_options` having no attribute `local_now` (its
originally reported `files_pane.load_file_detail` seam was already repointed to
`files_detail_panel`). That is tracked separately as task bead `sase-my`. If repairing
it is a mechanical repeat of your own fix, do it and close `sase-my` too, following the
same claim/note/close rules. If it is not mechanical, leave `sase-my` alone — do not
expand this phase.

## Landing

After every phase has finished, the land agent must do the following, in order.

1. **Confirm each of the five task beads is closed** (`sase-ll`, `sase-mv`, `sase-mr`,
   `sase-nk`, `sase-mw`) and that each carries a note describing the work done or
   justifying why it could not be completed. Repair any phase that skipped its bead
   bookkeeping.

2. **Land the combined tree** and verify it with `just check-full` handed to
   `/sase_monitor` — never inline.

3. **Then check whether any ready task beads remain for the `sase` project:**

   ```bash
   sase bead list --type task --status ready --limit 0
   ```

   **If any remain**, start a pseudo monitor over `sleep 1` whose next action tells the
   follow-up agent to run this exact prompt again, verbatim, with these same steps:

   ```bash
   sase monitor start \
     -r 'Hand the remaining sase task-bead backlog to a fresh agent shell' \
     -t 5m --next-output none \
     -n 'Complete the work recommended by previous agents by following these steps: (1) Review all of my current open (not in-progress) sase task beads for the "sase" sase project. (2) Close any task beads that are no longer relevant with a good reason. (3) Select the 5 task beads that would have the most impact if worked to completion. (4) Use your /sase_plan skill to fix the issues / make the improvements that correspond with these 5 task beads. Make sure the plan file you propose tells the agent(s) to: (a) if you think any of these 5 beads need approval from the user before working (be lenient here and do not ask for approval for objective improvements), do not ask directly, but instead leave a `TASK NEEDS APPROVAL` note on the bead; (b) mark the bead(s) you intend to work as in-progress by changing their status with the `sase bead update` command; (c) leave a brief note on the task bead(s) explaining the work that was done to fix the reported issue / make the requested improvement or, if the agent was unable to complete the work, justifying why they were unable to do so; (d) close each of the 5 task beads that it was able to finish; (e) if there are more task beads associated with the "sase" project, the agent should then start a pseudo monitor using the `sleep 1` command with a next action that instructs the next agent to follow this exact prompt (with these exact same steps); (f) if there are no more task beads to work, the agent should move on to the next numbered step in this prompt. (5) Review all `TASK NEEDS APPROVAL` notes left by prior agent shells and consolidate them into a single report for the user with suggested next actions. (6) Terminate.' \
     -- sleep 1
   ```

   Note that `sase monitor start` is the last thing that agent shell produces — the
   runner adopts the handoff and continues in the follow-up agent.

   **If none remain**, skip the handoff and go straight to step 4.

4. **Consolidate the approval notes.** Search every task bead for notes left by this
   epic's phases and by any earlier agent shell in this chain:

   ```bash
   sase bead search "TASK NEEDS APPROVAL"
   ```

   Write a single report for the user containing, for each note: the bead ID and title,
   the decision being asked for, the concrete options and their tradeoffs, the reporting
   agent's recommendation, and what is blocked until the user decides. If there are no
   such notes, say so explicitly rather than omitting the section. Then terminate.
