---
tier: epic
title: Make failed agent sessions and clans unmistakable
goal: 'When an agent session or clan stops on a failure, every Agents-tab surface
  says so: the row, the BY_STATUS section, the count chips, the waves and jump strips,
  and the detail panels all read FAILED and agree with each other. The cause and the
  next step are one glance away. A staged handoff the host refused to adopt shows
  as NOT ADOPTED, never as a phantom RUNNING with a ticking clock.

  '
decisions:
  readopt_action:
    ask: Should the NEEDS ATTENTION callout offer a TUI key that runs `sase agent
      readopt`?
    choices:
      cli_hint: The callout prints the exact readopt command; no new key or proc flow
      tui_key: Adds a confirm-gated key that runs readopt as a durable proc from the
        panel
    default: cli_hint
    why: Readopt only succeeds after an out-of-TUI repair, so a one-key retry invites
      repeat failures
    answer: cli_hint
phases:
- id: core-held
  title: Project held staged monitors as a first-class state in sase-core
  depends_on: []
  size: medium
  description: 'core-held: in sase-core, project a staged monitor member whose `monitor_stage_held`
    flag is set as monitor turn state `held`. Add `NOT ADOPTED` to the status-bucket
    table as Failed, teach the staged-membership facts about `held`, expose a binding
    for Python fallback loaders, and make previously indexed held members re-project.'
- id: held-state
  title: Consume the held state in sase and keep the held flag truthful
  depends_on:
  - core-held
  size: medium
  description: 'held-state: bucket `held` as Failed but nonterminal and never in flight.
    Show the member as `NOT ADOPTED ‖` with no ticking clock, move it to the settled
    monitor lane, and route the filesystem fallback loaders through the binding. Make
    every not-adopted path set the durable held flag and every successful readopt
    clear it.'
- id: session-rollup
  title: One failure-aware session rollup that every surface consumes
  depends_on:
  - held-state
  size: medium
  description: 'session-rollup: not-adopted members never count as started work. A
    failed turn stays the session''s state until a real later turn supersedes it,
    for both plan and plain roots. Clan roster, waves, tribe summary, BY_STATUS, and
    count chips all read the one session rollup. A `FailureFocus` record carries the
    cause to presentation, and a cross-surface consistency test pins it all.'
- id: row-signals
  title: Failure signals on Agents-tab rows, waves, and jump strips
  depends_on:
  - session-rollup
  size: medium
  description: 'row-signals: render the failure clearly in the list. Add a clan-row
    cause chip naming the failed member and its cause, show the failure time instead
    of a running hourglass on Failed rollups, and draw a red ✗ wave pip and ▶→✗ in
    WAVES. Put the ‖ glyph in SESSION TURNS and add a visual golden for the incident
    shape.'
- id: attention-callout
  title: NEEDS ATTENTION callout in session and clan detail panels, plus docs
  depends_on:
  - row-signals
  size: medium
  description: 'attention-callout: a left-barred NEEDS ATTENTION callout at the top
    of the session and clan detail panels. It says which turn failed, the cause, any
    not-adopted handoff, the downstream members it blocks, and the exact next step.
    The tribe NEEDS ATTENTION entry reuses it. Update the status, glyph, and monitor
    docs, and add panel goldens.'
proposed_by: bbugyi200.athena.1f4
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-22 08:33:02
status: wip
bead_id: sase-1ux
---

- **PROMPT:** [prompts/202611/failed_clan_visibility.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/failed_clan_visibility.md)
- **BEAD:** [sase-1ux](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ux/README.md)

# Plan: Make failed agent sessions and clans unmistakable

## What happened (incident `tailnet-1e.6.1.8`, 2026-11-22)

The epic clan `tailnet-1e.6.1` looked healthy on the Agents tab for about 8 hours while
it was actually stuck:

1. At 23:53 the continuation turn `tailnet-1e.6.1.8--1` ran `sase monitor start`. That
   **staged** monitor member `tailnet-1e.6.1.8--mon-0`, a full sibling artifact dir with
   `monitor_state: "staged"`, a `workflow_state.json` of `running`, and `run_started_at`
   set to the staging time.
2. At 00:00:32 the turn's `commit` finalizer failed: the before-commit hook
   `sase_git_fix` exited 1 because local commands in an external repo's `sase.yml` were
   untrusted. The runner marked `--1` FAILED, wrote a `handoff_not_adopted` adoption
   receipt, and ran `cleanup_staged_monitor(held=True)`. That cleanup writes only
   `monitor_stage_held: true` on the member (`src/sase/monitor/start_stage_cleanup.py`).
   **Nothing reads that flag.**
3. The TUI then:
   - projected the never-started member as `(RUNNING)` with `⧗ 8h`
     (`MONITOR_STATE_BUCKETS["staged"] = "Running"` in `src/sase/monitor_state.py`; the
     elapsed clock ran from `run_start_time`);
   - mirrored that phantom onto the session root, because the newest-turn fallback in
     `apply_status_overrides` (`src/sase/ace/tui/models/_agent_status_apply.py`) copies
     a newest monitor turn;
   - so the clan read `RUNNING [R1 W1 D7] ●●◐`, sat in the BY_STATUS **Running**
     section, and its tribe header counted no failure (`[R6 W23 D41]`, no `F`).
4. The only hints were a small red `⊛✗ commit` chip on the expanded session row and a
   `.8 ✗` in the clan detail's JUMP strip. The WAVES strip on the same panel showed
   `.8 ▶`: three different rollups disagreed (see below). The failure notification was
   dismissed unread at 06:44, alongside other dismissals.

Two older held stages on this machine have the same shape right now
(`sase-1r2.6--mon-2`, staged Nov 7, and `1dt--mon`, staged Nov 18: `staged` plus the
held flag, with a `handoff_not_adopted` starter receipt). The same rule projects both as
RUNNING wherever their sessions are visible.

### Three rollups, three answers

| Surface                                                                     | Rollup today                                        | Incident answer      |
| --------------------------------------------------------------------------- | --------------------------------------------------- | -------------------- |
| Session row, clan row and counts, WAVES, BY_STATUS, tribe header, tab strip | Newest-in-flight / newest-turn mirror onto the root | RUNNING              |
| Clan detail CLAN MEMBERS entry, tribe-summary unit                          | Severity ladder over all of the session's turns     | FAILED               |
| Session detail SESSION TURNS                                                | Per turn                                            | `--1 ✗`, `--mon-0 ▶` |

The ladder rollup has the opposite bug: a failure that a later successful turn
superseded keeps showing `✗` forever.

## Design principles

1. **Truthful leaves.** A row reads RUNNING only while something executes. A staged
   handoff the host refused to adopt is **NOT ADOPTED**: not running, not finished, and
   waiting on a human.
2. **One rollup, every surface.** The session root's computed status and bucket are the
   only session rollup. Every surface that summarizes a session reads them, so the
   surfaces cannot disagree.
3. **Failure is sticky until superseded.** A failed turn is the session's state until a
   real later turn starts (a retry, a readopted monitor, a continuation). A NOT ADOPTED
   member never supersedes anything.
4. **Say why, say what next.** A failed rollup carries a compact cause on its row and a
   full explanation in the detail panel: which turn failed, which finalizer, which
   handoff is stuck, what it blocks, and the exact command to resume.
5. **Quiet when healthy.** No new glyphs or chips on healthy rows. Failure uses the
   existing red family and existing glyphs (`✗`, `⊛`); one new glyph (`‖`) marks
   not-adopted handoffs.

## Vocabulary

| Concept                | Durable fact                                         | Projected state | Row word      | Glyph | Color                   | Bucket |
| ---------------------- | ---------------------------------------------------- | --------------- | ------------- | ----- | ----------------------- | ------ |
| Not-adopted handoff    | `monitor_state: staged` + `monitor_stage_held: true` | `held`          | `NOT ADOPTED` | `‖`   | bold `#FF875F` (salmon) | Failed |
| Failed turn or session | `done.json` failed / finalizer failed                | —               | `FAILED`      | `✗`   | bold `#FF5F5F`          | Failed |
| Finalizer cause        | `agent_meta.json["finalizer_status"]`                | —               | chip          | `⊛✗`  | `#FF5F5F`               | —      |

Why `NOT ADOPTED` and not `HELD`: "held by …" already means reverse-wait admission holds
on WAITING rows, and `⏸` already means a deferred finalizer and admission holds. The
word "NOT ADOPTED" matches the notification text ("was not adopted") and names the
remedy (`sase agent readopt`). `‖` (U+2016) is unused on the Agents tab today; verify it
renders single-width in the bundled visual font stack and through `wcwidth`.

Salmon `#FF875F` sits between FAILED red and RETRYING orange. It reads as
"failure-family, recoverable by you" and stays distinct from the `#FFAF5F` monitor gear
and finalizer warning colors.

## Before / after

Agents list, `@epic` tribe, BY_STATUS, clan collapsed:

```text
BEFORE
▲ @epic · 70 [R6 W23 D41] ⚙1 ⚙25 ⋔12
▶ Running ───────────────────────────── 52 agents · 6 running
  🐾 🤖 tailnet (RUNNING) [R1 W1 D7] ●●◐ ∎ ⚙1 ⚙2 tailnet-1e.6.1           ⧗ 16h46m

AFTER
▲ @epic · 70 [R5 W23 F1 D41] ⚙26 ⋔12
✗ Failed ────────────────────────────── 1 agent
  🐾 🤖 tailnet (FAILED) · .8 ⊛✗ commit [W1 F1 D7] ●●✗ ∎ ⚙3 tailnet-1e.6.1   00:00:32 · 9h16m
```

Session expanded inside the clan:

```text
BEFORE
3 – ⚡ tailnet ⧉7 (RUNNING) ⊛✗ commit ×8 −3 ⚙1 ⚙1 ◆ tailnet-1e.6.1.8            ⧗ 8h41m
    └ ⚡ (DONE) ◆ tailnet-1e.6.1.8--plan                               Nov 21 23:22 · 4m46s
    └ ⚡ ⚙ (MEASURED ATHENA ✓) ◆ tailnet-1e.6.1.8--mon                  Nov 21 23:53 · 30m30s
    └ ⚡ (FAILED) ⊛✗ commit ◆ tailnet-1e.6.1.8--1                           00:00:32 · 6m39s
       └ ⚡ ⚙ (RUNNING) ◆ tailnet-1e.6.1.8--mon-0                                     ⧗ 8h

AFTER
3 – ⚡ tailnet ⧉7 (FAILED) ⊛✗ commit ×8 −3 ⚙2 ◆ tailnet-1e.6.1.8                 00:00:32 · 6m39s
    └ ⚡ (DONE) ◆ tailnet-1e.6.1.8--plan                               Nov 21 23:22 · 4m46s
    └ ⚡ ⚙ (MEASURED ATHENA ✓) ◆ tailnet-1e.6.1.8--mon                  Nov 21 23:53 · 30m30s
    └ ⚡ (FAILED) ⊛✗ commit ◆ tailnet-1e.6.1.8--1                           00:00:32 · 6m39s
       └ ⚡ ⚙ (NOT ADOPTED ‖) ◆ tailnet-1e.6.1.8--mon-0                 staged 23:59 · never ran
```

Clan detail panel (session detail gets the same callout, scoped to the session):

```text
CLAN
tailnet-1e.6.1 FAILED [W1 F1 D7] ●●✗ ∎ · tailnet · @epic · 13 agents · 2 sessions
┃ ✗ NEEDS ATTENTION · wave 3 stopped at .8
┃   .8  tailnet-1e.6.1.8--1 failed 00:00:32 · ⊛✗ commit
┃   ‖   monitor handoff not adopted · --mon-0 never ran
┃   ⤷   blocks .land
┃   next  repair the cause, then  sase agent readopt tailnet-1e.6.1.8--1
┃         jump 07 · R retry · x dismiss
◆ MAIN  Summary
▸ WAVES · 3 + land · ●●✗ ∎ · wave 3 failed
  1 ● .1 ✓ .2 ✓ .4 ✓ .5 ✓ .7 ✓
  2 ● .3 ✓ .6 ✓
  3 ✗ .8 ✗
  ∎ ○ .land ⧗
```

## Phase `core-held`: project held staged monitors in sase-core

Work in the linked `sase-core` repo (open it with `sase repo open sase-core` and use the
printed path). The boundary rule applies because CLI, gateway, and TUI frontends must
all agree that a held stage is not running.

1. **Scanner projection.** In `crates/sase_core/src/agent_scan/scanner.rs`
   (`agent_session_turn_from_object`, monitor branch, around the
   `state: coerce_str(data.get("monitor_state"))` line), project `state = "held"` when
   `monitor_state == "staged"` and `monitor_stage_held` is truthy. Apply the same rule
   to the nested-turn path (`nested_turn_object`) so both meta shapes agree. A stale
   `monitor_stage_held` on any other state (for example `running` after a readopt) is
   ignored. Put the rule in one small pure function, for example
   `project_monitor_turn_state(monitor_state, stage_held) -> Option<String>`, in
   `crates/sase_core/src/staged_handoff/monitor_lifecycle.rs`. That module owns
   staged-membership facts; extend its module doc to say a held stage is durable intent
   that the host refused to adopt.
2. **Membership facts.** `evaluate_staged_monitor_membership` accepts `held` with
   exactly the `staged` answers: nonterminal, blocks lane start, no supervisor, no
   runner capacity.
3. **Status table.** In `crates/sase_core/src/agent_archive/status.rs`,
   `status_bucket_for_status("NOT ADOPTED")` returns `"Failed"`, so archive outcome
   reads `failed`. Update the doc comment's claim that the table adds nothing beyond the
   live Python table: the live table now delegates here for this word.
4. **Binding.** Expose `project_monitor_turn_state` through the Python binding crate
   (`crates/sase_core_py`), next to the existing staged-handoff and status-bucket
   bindings. The sase filesystem fallback loaders call it in `held-state`.
5. **Re-projection.** Previously indexed held members must re-project without any file
   changing. Find how the artifact index caches scan projections (record JSON keyed by
   file signatures plus a projection or schema version) and bump the version the
   established way, so cached `staged` records are rebuilt. If the index already
   re-derives turn state on read, document that and skip the bump.
6. **Audit other consumers.** Check every Rust consumer of the monitor turn `state`
   (agent list projections, gateway or fleet wires, staged reconcile) and make sure
   `held` is handled like `staged` wherever it is not explicitly about display.

**Tests (Rust):**

- `staged` + flag → `held`; `staged` with no flag → `staged`; `running` + stale flag →
  `running`; terminal states are unchanged.
- The nested-turn shape projects the same way as the flat shape.
- Membership accepts `held`.
- `NOT ADOPTED` buckets as Failed and archives as `failed`.
- A binding smoke test.

Run the sase-core repo's own checks.

## Phase `held-state`: consume `held` in sase and keep the flag truthful

If the pinned `sase-core-revision.txt` does not yet contain the `core-held` commit, move
the pin with `just ratchet-core-revision` (see the CI source revision pin section of
`docs/contribute/rust-backend.md`). A source checkout builds `sase_core_rs` from the
linked `sase-core` checkout.

1. **State semantics** (`src/sase/monitor_state.py`, `src/sase/turns/state.py`):
   - Add `"held": "Failed"` to `MONITOR_STATE_BUCKETS`.
   - Keep `held` **nonterminal**. Today `turn_state_is_terminal` means "bucket is not
     Running", which would make held terminal. Add an explicit
     `nonterminal_states: frozenset[str]` to `TurnStateConfig` (monitor config:
     `{"held"}`) that `turn_state_is_terminal` honors, and update the
     `monitor_state_is_terminal` docstring.
   - Audit every caller of `monitor_state_is_terminal` / `turn_state_is_terminal`
     (about 19) and every `monitor_state == "staged"` check reached through projected
     `Agent.monitor_state`. `held` must behave like `staged` everywhere except display
     and rollups. Write-side code reads durable meta and still sees `staged`, so the
     lane-blocking, readopt, and stage-replay paths are untouched.
2. **Loader projection.**
   - `src/sase/ace/tui/models/_loaders/_meta_enrichment_monitor.py`: a `held` monitor
     member gets status `NOT ADOPTED` and bucket Failed.
   - The filesystem fallback loaders (`_meta_enrichment_filesystem_turn.py`,
     `_done_filesystem_loaders.py`, and any other `data.get("monitor_state")` reader in
     `src/sase/ace/tui/models/_loaders/`) call the `core-held` binding through one thin
     adapter, so both load paths agree.
3. **Liveness and time.**
   - `agent_row_is_in_flight` already requires `monitor_state == "running"`; pin that
     with a test.
   - `monitor_row_is_settled` (`src/sase/ace/tui/models/agent_session_members.py`)
     treats `held` as settled for lane purposes, so `⚙N` counts it grey, never orange.
     Keep the invariant that every turn lands in exactly one lane.
   - `src/sase/ace/tui/models/agent_time.py` and `_agent_time_intervals.py`: a `held`
     row never ticks. Its time column renders `staged <time> · never ran` in dim style
     instead of `⧗ <elapsed>`.
4. **Row word.** Render `(NOT ADOPTED ‖)` in bold `#FF875F` in
   `src/sase/ace/tui/widgets/_agent_list_render_agent_status.py`, following the existing
   monitor status-pair conventions. Define the color and glyph once, next to the other
   monitor display constants, rather than inline.
5. **Write-side invariant.** Every path that settles a stage as not adopted marks the
   member held; every successful adoption clears the mark.
   - `retain_dead_staged_origin` (`src/sase/axe/run_agent_exec_staged_retention.py`)
     writes the starter's failed `done.json` with `handoff_not_adopted`. Verify whether
     the member then gets `monitor_stage_held`, through this path or through the
     `staged_handoff_reconcile` housekeeping recovery. If not, run held cleanup for a
     monitor-kind stage there.
   - `launch_monitor` (`src/sase/monitor/start_launch.py`, where `monitor_state` becomes
     `running` and the owner index is deleted) clears `monitor_stage_held`.
   - Add a test that enumerates the not-adopted entry points (runner dispatch, explicit
     readopt failure, automatic recovery, dead origin) and asserts the flag.
6. **Bounded audit, no fix.** The dead-pid gate in `_compact_workflow_loaders.py` and
   `_agent_loader_normalization.py` exempts session-turn members. Confirm or rule out
   that a session agent turn whose runner died stays RUNNING. Record the result as a
   `PROPOSED FOLLOW-UP:` note on this phase's bead; do not change that behavior here.

**Tests:**

- A loader fixture with the incident's member meta (`staged` + flag), which must come
  out NOT ADOPTED, Failed bucket, nonterminal, not in flight, no ticking time, and in
  the grey lane.
- `staged` without the flag stays Running, so the brief adoption window is unchanged.
- After a simulated readopt (`running`, flag cleared) the row is RUNNING again.
- Both the scan-wire and filesystem load paths give the same answer.

## Phase `session-rollup`: one failure-aware rollup that every surface reads

1. **Helper.** Add a single predicate, `row_is_not_adopted_handoff(row)` (`is_monitor`
   and `monitor_state == "held"`), in
   `src/sase/ace/tui/models/agent_session_members.py`. Use it everywhere below; no
   inline state checks.
2. **Root rollup** (`apply_status_overrides`, session-root branch in
   `src/sase/ace/tui/models/_agent_status_apply.py`):
   - Exclude not-adopted members from `candidates`, `descendant_turns`,
     settled-turn-time computation, and the `newest_pool` fallback. They are proposals
     attached to their starter, not turns.
   - **Failure frontier:** when nothing is in flight or waiting, and the newest
     _started_ member is a failed agent turn (bucket Failed, not superseded by a retry),
     the root shows that member's FAILED status and Failed bucket. This holds for
     **both** plan roots and plain-agent roots, which today keep their own terminal
     status. Write the plain-root test first to see the current behavior, and change
     only what that test proves wrong.
   - Audit `_override_live_handoff_done_status`, which can flip a DONE root back to
     RUNNING. A not-adopted handoff, whose staged record and receipt stay on disk for
     readopt, must never count as a live handoff.
   - Keep existing semantics: live retrying roots stay RETRYING; parallel sessions keep
     the severity ladder; a failed monitor with a launched or pending continuation still
     mirrors its lane bucket (the `test_nested_failed_monitor_*` cases).
3. **Make the other rollups read the root:**
   - **Clan roster** (`clan_roster_entries`, session branch,
     `src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan_roster.py`): use the
     session root's status and bucket instead of laddering
     `agent_session_lane_status_entries(rows)` plus the running monitor. A failure that
     a later successful turn superseded then stops showing `✗`.
   - **Tribe summary** (`_unit_snapshot` in
     `src/sase/ace/tui/models/agent_tribe_summary.py`): a session unit uses its root's
     bucket. Check that the NEEDS ATTENTION entries, the tribe-level status, and the
     header count chip now agree.
   - **Unchanged but pinned by tests:** clan container status
     (`apply_clan_container_status`; clan-local priority already puts Failed first), the
     wave fold (`_wave_state` in `src/sase/ace/tui/models/_agent_clan_waves.py` already
     folds only anchor rows), BY_STATUS placement (`status_bucket_for`), and the panel
     header and lane counts (`sase_agent_status_counts`). With a truthful root, these
     yield FAILED, `[W1 F1 D7]`, a Failed wave 3, the Failed section, and `F1`.
4. **FailureFocus.** Add a small frozen dataclass, computed in the model layer right
   after the rollup and attached to Failed session roots and Failed clan containers. The
   renderers in `row-signals` and `attention-callout` only format it; render paths never
   read files (see the `tui_perf.md` memory, rule 8). Fields:
   - `member_label` (clan short label such as `.8`; `None` for a standalone session);
   - `failed_row` (the frontier failed turn: name and stop time);
   - `finalizer_chip` (reuse `glance_finalizer_state` output, for example `⊛✗ commit`);
   - `not_adopted` (the NOT ADOPTED member's name and kind, plus the starter name for
     readopt);
   - `blocked` (clan member labels that wait on this member, from the clan wave edges
     already computed);
   - `extra_failed` (a count, when several members failed).
5. **Cross-surface consistency test** (new, for example
   `tests/ace/tui/models/test_failure_rollup_consistency.py`). Run it over a fixture
   matrix:
   - the incident shape;
   - a failed turn later retried and done;
   - a failed monitor with a launched continuation;
   - a plain-root session whose last turn failed;
   - a parallel session with one failed member;
   - a healthy running clan.

   For each, assert that the session row bucket, clan roster bucket, wave state, tribe
   unit bucket, BY_STATUS section, and count-chip lane all agree. This test is the
   contract that keeps the surfaces from drifting again.

## Phase `row-signals`: failure signals on rows, waves, and jump strips

1. **Clan cause chip.** On Failed clan rows
   (`src/sase/ace/tui/widgets/_agent_list_render_agent.py`), right after the status
   parentheses, render `· <member_label> <finalizer_chip>` from `FailureFocus`:
   - with no finalizer cause, `· .8 ✗`;
   - with only a not-adopted cause, `· .8 ‖`;
   - with more failures, `· .8 ⊛✗ commit +1`.

   The label is dim red and the chip keeps its existing finalizer style. Under the
   existing row-width budget, the chip drops before the name or the count chip does.
   Healthy rows are unchanged.

2. **Time column.** Failed session and clan rollups never show the running `⧗` elapsed
   form. They show the failed turn's stop time and duration, in the format FAILED agent
   rows already use.
3. **Waves.** The WAVES strip in `_agent_display_clan_waves.py` derives its per-member
   marker from the rollup bucket, so a failed member draws `✗` instead of `▶`. The
   header phrase reads `wave N failed` instead of `wave N in flight` when the wave state
   is Failed. The wave pip vocabulary (`_clan_wave_presentation.py`) is unchanged; the
   input is now truthful.
4. **SESSION TURNS strip.** In `_agent_display_agent_session.py`, a NOT ADOPTED member
   shows `‖` in the NOT ADOPTED color instead of `▶`.
5. **Goldens.** Add an incident-shaped fixture to the clan and agent-session PNG fixture
   modules (`tests/ace/tui/visual/_ace_agents_png_snapshot_clan_fixtures.py`,
   `_ace_agents_png_snapshot_agent_session_fixtures.py`). Cover at least BY_STATUS with
   the clan collapsed and the session expanded, at 120 columns plus one narrow width so
   the chip-dropping path is covered. Regenerate with `just fix-tui-screenshots`
   (targeted selectors). Follow the `tui_screenshot.md` memory's golden rules: pin time,
   zone, hostname, and HOME; run the check twice, once with `TZ=UTC`; inspect every
   changed golden.

## Phase `attention-callout`: NEEDS ATTENTION callout and docs

1. **Callout widget.** Add one shared renderer under
   `src/sase/ace/tui/widgets/prompt_panel/` that formats a `FailureFocus` as a
   left-barred block (`┃` in `#FF5F5F`, matching the bordered RAW PROMPT block style):
   - a header line `✗ NEEDS ATTENTION · <where it stopped>`;
   - one line per cause (`⊛✗ <instance>`, `‖ … not adopted · <member> never ran`);
   - `⤷ blocks <labels>` when anything waits on the failure;
   - a `next` line.

   Keep it short: at most 6 lines and no free-text error parsing. The full error stays
   in the existing ERROR section, which the callout points to.

2. **Placement.**
   - Top of the session detail panel, under the SESSION header.
   - Top of the clan detail panel, under the CLAN header and before MAIN Summary.
   - The tribe detail's existing NEEDS ATTENTION entries reuse the same cause line, so
     wording matches across all three panels.
3. **Next steps.** With a not-adopted handoff, print
   `repair the cause, then  sase agent readopt <starter>`. Always list the existing jump
   key for the failed member (clan panel) and the existing `R retry` and `x dismiss`
   bindings when the row offers them.

> [!decision] readopt_action = cli_hint No new key: the callout prints the exact command
> with the code style used elsewhere in detail panels.

> [!decision] readopt_action = tui_key Add a session/clan detail action, shown only when
> a not-adopted handoff exists, that opens a confirm dialog and then runs
> `sase agent readopt <starter>` as a durable proc through the existing proc-submission
> path (the `tui_perf.md` memory, rule 3). A refused readopt toasts the held reason.
> Pick a free key, add it to `src/sase/default_config.yml` and the footer hints, and
> document it in `docs/reference/tui/keymaps.md`.

4. **Docs.** Read the `docs_writing.md` memory before editing.
   - `docs/reference/automation/monitors.md` ("Status and bucket" table): add the `held`
     row (Failed bucket, NOT ADOPTED, never ran, resumes through `sase agent readopt`).
   - `docs/reference/tui/agent-statuses.md`: add NOT ADOPTED, the failure-frontier rule
     for sessions, and the NEEDS ATTENTION callout.
   - `docs/reference/tui/agent-row-glyphs.md`: add `‖`, the clan cause chip, and `‖` in
     the row-token inventory.
   - `docs/reference/agents/agent-sessions.md` (readopt section): add how a not-adopted
     handoff looks in the TUI.
5. **Goldens.** Session-panel and clan-panel PNG goldens for the incident fixture
   (`test_ace_png_snapshots_agents_agent_session_panel_monitor.py`,
   `test_ace_png_snapshots_agents_clan_panel.py`), with the same golden discipline as
   `row-signals`.

## Verification (every phase)

- `just check` from the repo root, per the `lint_and_test.md` memory. `just check-full`
  runs only when explicitly instructed.
- Live confirmation after `held-state` and again after `attention-callout`: capture the
  running TUI with `sase screenshot`, then check that the two live held stages on athena
  (`sase-1r2.6--mon-2`, `1dt--mon`) render NOT ADOPTED and that their sessions read
  FAILED. Read-only; do not readopt or dismiss them.
- Perf: all new derivation happens at load or rollup time, never in render or keystroke
  paths. Incremental refresh stays incremental; a status change on an existing row
  patches the row rather than rebuilding the list (the `tui_perf.md` memory, rules 5, 6,
  and 8).

## Out of scope (record as `PROPOSED FOLLOW-UP:` notes if still relevant)

- Porting the session/clan rollup itself to sase-core. It is Python today;
  `FailureFocus` plus the consistency test are the seam a later port would move.
- Owner-index leaks: `~/.sase/staged-monitor-owners/*.json` outlive dismissed members
  (four orphans exist today).
- WAITING members blocked by a failed dependency get no row marker. The callout's
  `⤷ blocks` line covers the clan view.
- Failure notifications that are bulk-dismissed unread.
