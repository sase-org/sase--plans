---
tier: epic
title: Finish deck repaint and scroll convergence, then confirm sase 0.18.0
goal:
  The remaining deck state defects are repaired, and the landed repairs pass the guarded
  release path through an installable sase 0.18.0.
parent_bead: sase-1kx.5.3.3
phases:
  - id: deck-convergence
    title: Repair the remaining Tool Runs and Main deck state transitions
    size: medium
    depends_on: []
    description:
      "deck-convergence: repair loaded-detail repaint and Main card/layout convergence
      with behavioral regressions and clean visual proof."
  - id: publication-proof
    title: Observe the committed repairs through guarded publication
    size: small
    depends_on:
      - deck-convergence
    description:
      "publication-proof: make no edits, observe CI and publication on the repair
      commit, and always close with an honest release verdict."
proposed_by: bbugyi200.athena.sase-1kx.5.3.3.land
create_time: 2026-10-15 05:53:51
status: wip
---

- **PROMPT:**
  [prompts/202610/v0_18_0_deck_repaint_completion.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/v0_18_0_deck_repaint_completion.md)
- **PARENT:**
  [202610/v0_18_0_capture_convergence.md](https://github.com/sase-org/sase--plans/blob/main/202610/v0_18_0_capture_convergence.md)

# Finish the remaining deck and release acceptance

This is remaining work for **sase-1kx.5.3.3**, whose plan is
`plan:202610/v0_18_0_capture_convergence.md`. Read that artifact and the landing note on
that bead first. Preserve the inherited decisions **core_release=dispatch** and
**stale_epic_1jz=notes**. The original release criteria still require publication; a
canceled watcher is not success.

One medium coding agent can repair these bounded presentation problems. The separate
no-edit phase is necessary because release acceptance must observe that agent's future
landed commit in CI and PyPI. A tale cannot finish that proof before its own commit. The
coding phase must end normally after local verification; it must not wait for its own
SHA, push, or CI. This child epic's `parent_bead` returns control to the interrupted
landing. Ancestor closure is never an implementation phase.

## Verified starting point

The landing audit fetched origin. Local HEAD and `origin/master` were both
**be73b6021ac29d8fcad2cb897192bc6455a47e2b**, the only commit since the current epic
started. It is phase `.1`'s nine-file visual-fixture repair; no intervening base changes
require integration. Fetch again and inspect newer changes before editing.

Both children and every note were reviewed. Phase `.1`'s evidence is real:

- ToolRun **76cc0322368c97df8ee8485040deffd4** ran `just check`, exit 0, triage pass,
  **8m54s overall**. The note's 3m11s is the scoped-test stage duration.
- Monitor **4njkqnvs5b2p** ran the full UTC visual check, exit 0: **1366 passed, 1
  skipped**, `clean`, `created=0 updated=0 unchanged=1100 stale=0`.
- Phase `.2` closed **canceled**, `release not confirmed: FULL_CI_RED`, correctly. Leave
  its historical verdict intact.

The new semantic waits exposed remaining defects in CI instead of fixing them. Complete
evidence, rendered SVG text, exact node IDs, and an executable deterministic Tool Runs
probe are in **file:explicit:9dbb8a7a761400203c1c56f0**. Consume it through
`sase artifact read`. The audit independently retrieved raw job logs with:

```sh
gh api --allow-escape-sequences repos/sase-org/sase/actions/jobs/JOB_ID/logs
```

`gh run view --log-failed` truncated before the useful exceptions here. Do not infer the
full failure inventory from that shortened output.

Full CI **38683274240** on the repair SHA had three visual failures, 1363 passes:
renamed session Reply selection, New Project bare-git create, and dark codeblock
highlighting convergence. The next same-SHA run **38698824253** had two visual failures,
1364 passes: single Main paged and Tool Runs loaded detail. Its executed report status
is **failed**, not drift. Zero comparison counts mean comparison did not complete, not
that the images matched. Its nonvisual jobs all passed.

At audit, Master Gate **38680491171** is green at the repair SHA. PR **#299** is OPEN at
**735922478413edaa3e4d3276f0a2b17007d4a2ad**. Publish **38682622881** only completed
metadata work; build/publish/install jobs were skipped. Fresh PyPI JSON still reports
sase **0.17.1**, no **0.18.0**, and core **0.40.0** available. The pin is unchanged; do
not cut core again for the already satisfied minimum. Full CI **38713421385** was in
progress; inspect its result without dispatching a duplicate. Even a passing run does
not disprove the deterministic repaint probe below.

## Rules

- Read `lint_and_test.md`, `symvision.md`, `tui.md`, `tui_screenshot.md`, and
  `tui_perf.md` through `/sase_memory_read`. Read applicable `AGENTS.md` files.
- Preserve assertions, existing timeouts, pixel equality, release guards, and watcher
  close/verdict contracts. No fixed sleeps, assertion retries, skip/xfail, copied CI
  candidates, broad rebaselines, or artificial cache prewarming to hide a product bug.
- These are Textual rendering and scroll lifecycle repairs. Keep them in the existing
  presentation layer. If investigation requires shared domain behavior, open `sase-core`
  through `/sase_repo`, implement it there, and update bindings/pin.
- Run `just fix`, then `sase tool run check` (wrapped `just check`). **Do not run
  `just check-full`.** Unrelated check-full-only infrastructure is follow-up work.
- Use `/sase_monitor` for long visual runs and external waits; wait for the start
  command itself to exit. Use its escalation join if inline `sase tool run check`
  requests it. Never leave a provider-owned command running at turn end.
- Phase workers record new unrelated findings as `PROPOSED FOLLOW-UP:` notes on their
  own phase. Do not file task beads or duplicate the triage below.
- No manual release merge, version edits, global install, primary-checkout deploy, or
  killing the waiting sase-1jz agents. No forced successful nested landing.

## Phase deck-convergence

### 1. Make loaded Tool Runs detail actually repaint

The exact failed node is
`tests/ace/tui/visual/test_ace_png_snapshots_tool_runs_cards.py::test_tool_runs_card_png_snapshots`.
Job **116305071749** timed out after 60s waiting for `lint (symvision)`, even though the
new `loaded_run_ids` barrier succeeded. The SVG still shows
`run detail unavailable · summary retained` for `tr-c-failed`.

This has a deterministic production-path reproduction, not merely a timing guess: build
the placeholder and loaded document for the same fixture run and summary digest, then
apply both through the real `ToolRunsDeckView.show_tool_runs_document`. Only replace
`_apply_section_content` and the unmounted prepare seam for observation. The audit got
**one paint**, `_document is placeholder`, absence text still present, and the stage
text absent. The full probe is in the evidence artifact.

Trace these paths together:

- `widgets/decks/tool_runs/rows.py::tool_runs_cache_signature` includes selector, run
  count, live IDs, and truncation, but no detail/render content.
- `tool_runs/widget.py` and `loader.py` use that signature as `document.digest` for both
  placeholder and loaded bodies.
- `document_view.py::show_document` dedupes the same render key before replacing
  `_document`. Therefore a successful worker can leave a permanent placeholder.
- `ToolRunsDeckView.update_display` also returns early for the same painted signature
  when the cache has become complete, without proving that complete detail was painted.
  A canceled/stale worker populating the cache can trigger this path.

Separate summary freshness from rendered-content identity. Make the visual key reflect
all inputs needed for a real repaint, including detail availability/content and the
view's detail level; preserve cheap unchanged settled renders and live repaint behavior.
Review width/hints and store-token inputs at the same boundary. Do not disable dedupe
globally, invent a timer-based digest, or hash arbitrary rich renderable object
identity. Keep worker I/O off the pump and preserve stale-subject and generation
rejection, cursors, and the intentional pruned-detail case.

Add a behavioral regression that traverses the actual document/view paint path: cached
summary without detail -> worker delivers detail -> stages and tail are visible, with
the absence line gone. Also cover a stale worker populating the cache before the next
valid update, changing detail level without a summary change, and unchanged complete
state avoiding unnecessary repaint. The existing
`test_fast_path_still_loads_when_details_missing_across_generation_bump` mocks
`_paint_tool_runs_result`, so its worker-start assertions cannot prove this behavior.
Extend the focused split test modules; preserve their compatibility exports.

### 2. Converge Main selection, scroll, and automatic layout

In job **116275103654**, the renamed generic session fails the new visible-Reply wait.
The final diagnostic is:

```text
main_active='context'; view_active='reply'; spread_active='context'; pending=None;
scroll_y=0; max_scroll_y=9; anchor_rows=(('reply', 2),); anchor_generation=4/4
```

The visible header is Context 1/2 while the body begins with Reply. This is the exact
state the parent promised to repair. A longer wait cannot recover a dropped target.
Trace `document_view.py::scroll_to_card/_apply_spread_scroll`, section layout reserve,
deferred Main restoration, and `panel_interaction.py` scroll-derived chrome. The current
`_apply_spread_scroll` clears its pending card immediately after requesting a scroll,
without proving the eventual layout accepted it. Early clamping followed by range growth
is a hypothesis to prove with controlled layout timing.

Preserve a current selection target until the current document/layout can apply it;
stale generations must not override later user navigation. Reuse established
generation-aware scroll settling rather than adding ad hoc repeated user actions. Test
delayed anchor/range readiness and refresh-after-selection so the active card, view,
scroll position, and visible header agree. Keep the fixture's new identity and
document-generation barriers; do not remove the failing visible-header assertion.

Also investigate
`test_ace_png_snapshots_agents_decks.py::test_agents_decks_single_main_paged_png_snapshot`
from job **116305071749**. It times out after 15s waiting for the long Reply to exceed
the spread threshold; the SVG actually shows **MAIN spread · auto** with long Reply
text. Trace `_decide_document_mode`, measurement cache keys, viewport readiness,
`_view_generation`, and deferred apply. Confirm that complete content and final geometry
trigger the correct automatic mode. Keep this fixture automatic: do not force paged
mode. Add a regression for the demonstrated transition or fixture readiness defect, not
speculative production changes.

The existing overlap owner is **sase-1jz.3**; the land audit appends evidence to its
epic. **sase-1ii** owns the neighboring spread-sticky and committed-search nodes; do not
claim the new paged symptom proves the same root or close that task yet.

### 3. Verify the repairs and preserve completed capture work

Run targeted behavioral tests and the actual visual nodes, using the canonical visual
recipe. Recheck all Tool Runs card cases, renamed session, single Main paged,
`spread_deck_sticky`, `left_right_search_committed`, and callers of the shared Main
readiness helper (gate, weighted-capacity, neighbors). Repeat the targeted visual checks
in the default zone and UTC to expose late layout transitions. Retain the completed
top-bar density and Logs telemetry isolation fixes.

Then run full **`TZ=UTC just fix-tui-screenshots --check`** through a monitor. Require
no failed tests and `clean`, `created=0 updated=0 stale=0`; partial output or an
isolated rerun does not replace full proof. Inspect and attribute every new failure.
Only regenerate an intended visual change after proving its stable state and inspecting
the images; these convergence fixes normally need no golden changes. The unrelated
flakes below retain their task ownership; do not silently waive the release's green-CI
acceptance or rebaseline them.

Run `just fix` and `sase tool run check`. Record exact evidence and source changes,
retire this phase's epic-symbols, close the phase normally, and let host finalization
commit. Do not wait for this work's own commit or CI in this phase.

## Phase publication-proof

This phase makes no file changes and always closes with an honest verdict.

1. Read the repair phase's notes. Fetch and fast-forward a clean checkout, prove its
   repair commit and be73b6021a are on origin/master, and use the **new repair SHA** as
   FIXED_SHA. Historical reds must not immediately consume this watch's budget. Check
   newer core pins through `/sase_repo`; 0.40.0 already satisfies the audited pin. Only
   repeat the accepted urgent core cut if a newer pin actually needs it, without
   duplicating an equivalent running release.
2. Inspect current Full CI and PR #299. Do not duplicate an in-flight workflow. Resolve
   this phase's `sase bead epic-symbols <phase-id>` first. Preflight inline:

   ```sh
   NO_COLOR=1 GH_FORCE_TTY=0 tools/release_watch --repo sase-org/sase --version 0.18.0 --fixed-sha <FIXED_SHA> --await-core-version 0.40.0 --once --dry-run --close-bead <phase-id>
   ```

   Require exit 0 and a fully parsed state line. On failure close this phase canceled
   with reason `release not confirmed: WATCHER_BROKEN (preflight)` and the output, then
   finalize; do not repair code here.

3. Start one `/sase_monitor`, timeout 13h, statuses `WATCHING RELEASE` /
   `RELEASE WATCHED`, with:

   ```sh
   NO_COLOR=1 GH_FORCE_TTY=0 timeout -k 5m 760m tools/release_watch --repo sase-org/sase --version 0.18.0 --fixed-sha <FIXED_SHA> --await-core-version 0.40.0 --deadline 12h --close-bead <phase-id>
   ```

   The watcher owns bounded deduplicated dispatch; ci_watch owns guarded merge. The
   follow-up reads the verdict and phase, repairs a mechanical close failure only (done
   for RELEASED, canceled with the precise verdict otherwise), and finalizes. It must
   not edit, reopen the phase, or restart the watch. If monitor start fails twice, close
   canceled with WATCHER_NOT_STARTED and both errors. Wait for any yielded start-command
   session to exit.

## Landing and ancestor resumption

Closing both phases is not proof this epic is complete. Its lander verifies source,
notes, current-base integration, clean full visual evidence, fresh green Full CI and
Master Gate, PR #299's merge, GitHub v0.18.0, actual successful Publish build and
publish jobs, and PyPI 0.18.0. Require a fresh temporary environment outside the repo
with **unpinned `pip install sase`** resolving 0.18.0, working `sase --version`, and the
resolved core version recorded. The watcher's pinned smoke is insufficient.

If only external waiting remains, use `/sase_monitor` without epic `--close-bead`; if
implementation remains, plan only that work. Never force a successful close.

Once this child lands, resume **sase-1kx.5.3.3** through `parent_bead`. Re-read its
landing note, all descendants and notes, linked plan and post-child drift. Rerun
descendant and linked-plan readiness checks. Resolve or re-key every entry from
`sase bead epic-symbols sase-1kx.5.3.3`, then close normally with
`sase bead close sase-1kx.5.3.3 --note "<verification and follow-up outcomes>"`, run
`just symvision`, and set `status: done` in `plan:202610/v0_18_0_capture_convergence.md`
using the sanctioned plans checkout. These are landing actions, not child phases.

Repeat the full readiness, drift, symbol, normal-close, symvision and plan-done checks
for directly parented plan ancestors in this order:

| Bead         | Linked plan                                      |
| ------------ | ------------------------------------------------ |
| sase-1kx.5.3 | plan:202610/v0_18_0_visual_release_completion.md |
| sase-1kx.5   | plan:202610/v0_18_0_release_finish.md            |
| sase-1kx     | plan:202610/v0_18_0_release.md                   |

For sase-1kx, complete its explicit release closeout: evidence-based closure of
sase-16p, sase-1gh, sase-19d, sase-1kg and sase-1ii; honor **stale_epic_1jz=notes**
(verify before closing .3, then note .4/.5 with green evidence); report whether
ci_watch's heavy dispatcher is live. Do not deploy to the primary or kill its waiting
agents. Preserve all historical follow-up outcomes recorded on the ancestor beads.
Inspect the next parent link and continue only through fully ready plan ancestors; if a
parent is a phase, close only that phase normally. Stop at the first incomplete or
ambiguous parent and record its blocker. No `--force` to advance this landing.

## Follow-up disposition

The proposing lander completed unrelated follow-up triage and recorded every outcome on
**sase-1kx.5.3.3**. Neither current child has a `PROPOSED FOLLOW-UP:` entry. Preserve
earlier outcomes from sase-1kx.5.3, .5, and sase-1kx rather than re-filing them.

- **sase-1a0** received independent `+1` evidence for the exact bounded monitor-result
  wait; no duplicate task was created.
- **sase-1l7** owns the Ctrl+T-before-reveal failure. It is related to **sase-1g8**,
  whose different midword visibility failure does not establish a shared root.
- **sase-1l8** owns the New Project bare-git create/review capture that never settles
  its availability check.
- **sase-1l9** owns the dark codeblock capture blocked by the artifact-kind worker. It
  is related to **sase-hx**, whose older dropped-invalidation fix does not prove the
  same root. Do not reopen that old task based on this symptom alone.
- The active **sase-1jz** epic received a `DISCOVERED ISSUE` note with the deterministic
  Tool Runs probe, phase **sase-1jz.3** provenance, and the two Main-deck observations.
  This child plan retains the release's remaining repairs; no duplicate task was filed
  for them. The new paged node is not yet evidence to corroborate **sase-1ii**.

All three new tasks are ready and sized `large` because their CI-only roots are
unconfirmed, and each includes the originating child **sase-1kx.5.3.3.2**, note #2,
same-SHA fail/pass evidence, and the durable artifact reference. Topical overlap with
other active epics was not treated as a demonstrated causal link.

The four independently observed unrelated flakes are the bounded monitor-result wait,
Ctrl+T before peek reveal, New Project bare-git create, and the dark codeblock snapshot
waiting for `prompt-artifact-ref-kinds:0`. Each failed in 38683274240 and passed in the
next same-SHA run 38698824253. They are separate from the deterministic Tool Runs probe
and the two Main-deck states above. No omnibus FULL_CI_RED task is appropriate; no local
`just check-full` failure is being used to add release work.
