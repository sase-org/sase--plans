---
tier: epic
title: Keep tribe panels mounted under BY_STATUS grouping and surface stale running-TUI
  code
goal: The Agents tab stops tearing down and remounting tribe panels (for example `@epic`)
  under BY_STATUS grouping with live agent churn, and a long-running TUI whose editable
  checkout has advanced past the code it imported detects that staleness, tells the
  user, and offers the existing restart-when-ready flow — so landed fixes actually
  reach the screen instead of silently sitting on disk.
phases:
- id: by-status-incremental
  title: Admit BY_STATUS grouping to the incremental Agents display path
  depends_on: []
  size: medium
  description: 'by-status-incremental: determine the BY_STATUS panel-safety invariants,
    allow the incremental display diff and row remove paths when the rendered widget
    tree and status-bucket membership are stable, retain a full rebuild (under a distinct
    fallback reason) for genuine bucket, hierarchy, or anchor changes, and cover both
    directions with focused and perf tests.'
- id: stale-process-restart
  title: Detect and surface a running TUI whose editable checkout has advanced
  depends_on: []
  size: medium
  description: 'stale-process-restart: record the editable checkouts'' imported git
    revisions at TUI startup, cheaply revalidate them on the existing update-status
    cadence, and when the deployed checkout has advanced past the running process
    surface a restart-to-load-new-code state in the Update panel and notifications
    wired to the existing restart-when-ready machinery, without adding render-path
    or keystroke cost.'
- id: verify-on-athena
  title: On-host verification of panel stability and stale-code surfacing
  size: medium
  depends_on:
  - by-status-incremental
  - stale-process-restart
  description: 'verify-on-athena: restart the athena TUI onto the fixed tree, soak
    under BY_STATUS grouping with live churn proving `@epic` stays mounted with no
    steady-state unsupported_grouping fallbacks, script a checkout-advance to prove
    the staleness indicator fires and clears through a restart, and add a regression
    guard against silent full-rebuild reintroduction.'
proposed_by: bbugyi200.athena.0mq
create_time: 2026-09-18 06:26:36
status: wip
bead_id: sase-12p
---

- **PROMPT:** [prompts/202609/by_status_panels_and_stale_tui.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/by_status_panels_and_stale_tui.md)
- **BEAD:** [sase-12p](https://github.com/sase-org/sase--beads/blob/main/pages/sase-12p/README.md)

# Keep Tribe Panels Mounted Under BY_STATUS Grouping And Surface Stale Running-TUI Code

## Problem

The `@epic` agent tribe panel on athena still visibly disappears and reloads frequently,
after epic sase-127 ("Fix Agents-tab flicker and disappearing tribe panels") closed with
a verified 30-minute athena soak showing `@epic` continuously mounted. Live diagnosis on
2026-09-18 established two compounding root causes, one explaining why the fix never
reached the user and one that will keep the symptom alive even after it does.

### Root cause 1: the running TUI imported pre-fix code and nothing says so

- The live TUI process (`python3 -m sase tui --restart-axe` from the uv tool install,
  editable against `~/projects/github/sase-org/sase`) started 2026-09-16 14:38 and has
  been running since. Every sase-127 fix commit (155aeee2e stable-query incremental
  display, 7058f16ce bounded-load convergence, 5b7c4553c no-op fleet projection skip,
  677ed7d8e perf guard) landed on 2026-09-17 — after the process imported its modules.
- The deployed checkout is at c331faace (2026-09-18 05:46) and contains all of those
  commits with a clean tree, so the fixes sit on disk while the process keeps executing
  the pre-fix code it imported.
- The update surface cannot see this. `detect_dev_latest`
  (`src/sase/dev_update/detect.py`) classifies the checkout against its git upstream
  only (behind/ahead/diverged via `classify_git_upstream`). The checkout is pulled
  regularly outside the TUI's own dev-update flow, so the Update panel reports "current"
  the moment a pull lands. No component compares the revision the running process
  imported against the checkout's current HEAD, and `restart_after_update` /
  `restart_after_update_when_ready` (`src/sase/ace/tui/update_restart.py`) only run at
  the end of a TUI-initiated dev-update proc.
- Consequence: on an editable-install host, every fix landed while the TUI is running is
  silently invisible until the user happens to restart, and the user re-reports
  already-fixed bugs. This report is itself the reproduction.

### Root cause 2: BY_STATUS grouping never gets the incremental display path

- This host's persisted Agents grouping is `by_status` (`~/.sase/grouping_mode.txt`),
  and the persisted filter query is active (`~/.sase/ace_agents_last_query.json`).
- The incremental display diff added by sase-127 admits only `GroupingMode.STANDARD` and
  `GroupingMode.BY_MACHINE`: `_agent_display_diff_panels`
  (`src/sase/ace/tui/actions/agents/_display.py`, the grouping-mode guard just after the
  `search_query_changed` check) records `display_fallback: unsupported_grouping` and
  falls back to the full `_refresh_panel_widgets` rebuild for every other mode. The
  optimistic row remove path (`_try_remove_agent_rows` in
  `src/sase/ace/tui/actions/agents/_display_panel_patches.py`) bails the same way, and
  the row patch path only survives BY_STATUS for non-structural changes via
  `_status_row_patch_is_safe`.
- This host runs live agent churn nearly continuously, so the agents surface reloads on
  most auto-refresh ticks; under BY_STATUS every finalized list replacement tears down
  and remounts every panel, which the user perceives as the `@epic` panel disappearing
  and reloading.
- This is exactly READY feature task sase-128 ("Extend incremental Agents refresh to
  Status grouping"), filed by sase-127.land as a pre-existing limitation outside that
  epic's acceptance. It was corroborated with a `+1` carrying this live evidence on
  2026-09-18. This epic is its remediation; the land agent should dispose sase-128
  against this epic's outcome rather than leaving it for independent triage.

Restarting the TUI today would stop the extra pre-fix rebuild sources (active-search
fallback, no-op fleet repaints, load-tier oscillation) but the BY_STATUS full-rebuild
path alone still remounts panels on effectively every refresh under churn, so both root
causes need fixing to close the symptom.

All changes are Textual presentation state, TUI loader glue, and the existing Python
update/dev-update stack in this repo. Per the Rust core boundary litmus test no
`sase-core` change is required: staleness detection concerns the running Python
process's own imported modules and is consumed only by this TUI's update surface.

Relevant tui_perf rules (read this turn): prefer selective updates over full rebuilds
(rule 6), route refreshes through the existing fast path (rule 5), periodic ticks
revalidate while recomputes get a longer cadence (rule 10), keystroke paths are
read-only and prompt-free (rule 11), cache disk reads keyed by mtime and keep render
paths free of stat/glob (rule 8), and keep startup off data-scaled work (rule 9).

## Phases

### Phase by-status-incremental: Admit BY_STATUS grouping to the incremental Agents display path

Implements the remediation of task sase-128.

1. Reproduce first. Add a failing test that renders a BY_STATUS-grouped Agents tab,
   applies a finalized list replacement whose rows all keep their status bucket,
   name-root/name-prefix subgroup, and launch anchor, and asserts the incremental path
   is taken: zero `update_list` full-panel calls and no `unsupported_grouping` fallback
   trace. Model it on the existing incremental-display coverage around
   `_agent_display_diff_panels` and `_refresh_affected_panel_widgets` from sase-127.
2. Derive the BY_STATUS safety invariants from the existing `_status_row_patch_is_safe`
   contract (same status bucket, same name-root/name-prefix subgroup, same launch anchor
   — a bucket, hierarchy, or recency change invalidates cached visual positions). Admit
   `GroupingMode.BY_STATUS` to `_agent_display_diff_panels` when the rendered widget
   tree matches the grouping mode and the recomputed panel key set and per-panel
   membership are unchanged; reuse the same membership computation the full rebuild
   would use so the diff and rebuild cannot disagree.
3. Extend `_try_remove_agent_rows` to BY_STATUS under the same invariants (a removal
   that empties a status bucket or collapses a subgroup falls back to rebuild).
4. Keep the full rebuild for genuine status-bucket membership changes and record it
   under a distinct fallback reason (for example `status_membership_change`) so traces
   distinguish it from the residual `unsupported_grouping` reason, which should remain
   only for modes still outside the path (for example BY_DATE, which stays out of scope
   here).
5. Tests both directions: stable membership patches incrementally (zero full-panel
   rebuilds); a row moving between status buckets, a bucket appearing/emptying, or a
   subgroup/anchor change triggers exactly one full rebuild with the new reason; row
   patch and row remove behavior under BY_STATUS is covered. Extend the sase-127 perf
   guard so a finalize pass with unchanged membership under BY_STATUS cannot silently
   regress to a full rebuild.

### Phase stale-process-restart: Detect and surface a running TUI whose editable checkout has advanced

1. At TUI startup, record the imported-code identity for each editable core package the
   update stack already knows about (host `sase`, editable plugins, `sase-core` when
   editable): the package's `git_root` plus its HEAD SHA at import time. Capture it off
   the first-paint path (background worker after the startup stopwatch ends, per
   tui_perf rule 9); a missing or non-git root degrades to "unknown" and never errors.
2. Revalidate cheaply on the existing update-status cadence (rule 10): gate re-reading
   each root's HEAD behind stats of `.git/HEAD` and `.git/packed-refs` mtimes so quiet
   ticks cost two stats per root and zero subprocesses; only a drifted stat re-resolves
   the SHA. No keystroke or render path may trigger this (rules 8 and 11).
3. When a root's current HEAD differs from the recorded imported SHA, surface a "running
   code is stale — restart to load new code" state:
   - an Update panel row/chip alongside the existing rows built by
     `build_update_panel_state` (`src/sase/ace/tui/update_panel_state.py`), with the
     incoming-commit preview reusing the existing `sase.updates.incoming_commits`
     machinery over `imported_sha..HEAD` so the user sees exactly which fixes they are
     missing;
   - a one-time notification per drift generation (no per-tick nagging), with an action
     that invokes the existing `restart_after_update_when_ready` flow (defer while
     tracked background procs run, the current wait/expiry semantics unchanged).
4. Restarting stays user-initiated: no silent auto-restart in this phase. The detection
   state must reset correctly across the restart (the relaunched process records fresh
   imported SHAs and the indicator clears).
5. Do not build a competing process supervisor: epic sase-11y ("Service host and
   Services tab", in progress) owns service supervision and restart accounting. This
   phase only extends the existing TUI update surface and its restart helper. Epic
   sase-12o ("Keep installed shell completion fresh across SASE upgrades", in progress)
   touches install/refresh/update diagnostics for shell completion; rebase on master and
   reconcile update surface conflicts if its work lands concurrently, without absorbing
   its scope.
6. Tests: startup capture (including unknown-root degradation); stat-gated revalidation
   performs no subprocess work on unchanged mtimes; drift produces the panel row, the
   notification exactly once per generation, and the restart wiring; a
   pulled-then-reverted checkout (HEAD returns to the imported SHA) clears the state;
   update-panel projection stays pure (extend the `update_panel_state` unit tests).

### Phase verify-on-athena: On-host verification of panel stability and stale-code surfacing

Prove both mechanisms with the instruments that diagnosed them.

1. Restart the athena TUI onto the integrated tree (this also delivers the
   already-landed sase-127 fixes to the user for the first time). Soak with
   `SASE_TUI_TRACE=1` for at least 30 minutes in BY_STATUS grouping with the user's
   persisted filter query and live agent churn, and verify:
   - tribe panels (for example `@epic`) remain continuously mounted;
   - steady-state finalized refreshes show no `display_fallback: unsupported_grouping`
     and no `display_fallback: active_search`;
   - full rebuilds occur only with a genuine membership-change reason;
   - quiet auto-refresh ticks still reload no surfaces (`refresh.auto_tick` counters,
     per tui_perf rule 14).
2. Script the staleness acceptance: with the TUI running, advance the deployed checkout
   by at least one commit (a pull or a scripted fast-forward in a controlled clone
   configured as the editable root for a disposable TUI session if mutating the live
   checkout is not acceptable at verification time), and verify the stale-code row and
   notification appear within one update-status cadence, the incoming-commit preview
   names the new commits, the restart-when-ready flow relaunches, and the indicator is
   clear in the relaunched process.
3. Keep this phase panel-stability and staleness-surface acceptance only: the active
   sase-124.8.4 line owns the Agents freshness latency/marker/trace target matrix on
   this host — do not absorb, rerun, or modify its verification artifacts.
4. Record before/after evidence in the phase notes (trace excerpts, mount longevity,
   rebuild counts and reasons), and record a `PROPOSED FOLLOW-UP:` note proposing a
   `tui_perf.md` memory addition documenting the grouping-mode incremental-path
   invariant and the editable-install stale-process gotcha (memory edits route through
   the memory-write procedure, not this epic; ready memory task sase-109 already covers
   adjacent refresh invariants and may be the natural home).

## Acceptance

- Under BY_STATUS grouping with live churn and a stable filter query, a finalized Agents
  refresh with unchanged panel membership performs zero full panel rebuilds, and `@epic`
  stays mounted across a 30-minute athena soak.
- Genuine status-bucket membership changes still rebuild correctly, under a distinct
  trace reason, with test coverage in both directions.
- A long-running TUI whose editable checkout advances shows the stale-code state and
  restart affordance within one update-status cadence, at zero steady-state subprocess
  cost, and the state clears after restart.
- The sase-127 perf guard family fails if either the BY_STATUS incremental path or the
  staleness surfacing silently regresses.
