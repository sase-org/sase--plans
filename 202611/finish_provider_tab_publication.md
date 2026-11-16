---
tier: tale
title: Finish provider-tab publication and close its epic
goal:
  Repair the remaining snapshot and publication defects, verify the approved acceptance
  criteria, and close sase-1t3.7.1 and its parent phase sase-1t3.7.
size: medium
proposed_by: bbugyi200.athena.sase-1t3.7.1.land
proposed_with: codex/gpt-6-astra
bead: sase-1t3.7.1
create_time: 2026-11-16 18:01:23
status: wip
---

- **PARENT:**
  [202611/provider_tab_generation.md](https://github.com/sase-org/sase--plans/blob/main/202611/provider_tab_generation.md)
- **BEAD:**
  [sase-1t3.7.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1t3/sase-1t3.7.1.md)

# Finish provider-tab publication and close its epic

## Assignment and constraints

This is the remaining work from the landing audit of **sase-1t3.7.1** at `7c68d1860c`.
Implement it directly as one bounded TUI repair. This tale has no land agent: its coder
owns verification and the concrete closeout below in the same turn as the final
implementation. Closeout must not depend on this work's future commit SHA, push, or CI
result.

Read these through their audited commands before implementation:

- `sase bead read sase-1t3.7.1 -r "Need landing findings and follow-up outcomes"`
- Both children, `sase-1t3.7.1.1` and `sase-1t3.7.1.2`, including all notes.
- `sase bead read sase-1t3.7 -r "Need parent phase requirements and DECISIONS"`
- `sase artifact read plan:202611/provider_tab_generation.md "Need the original acceptance requirements"`
- Memory `ace.md`, `tui.md`, `tui_perf.md`, `lint_and_test.md`, and `tui_screenshot.md`
  through `/sase_memory_read`.

Preserve final decisions: `trust_ack_compat=cutover`, `tui_perf_memory=no`, and
`hotfix_reinstall=yes` only for the outer storm-hotfix phase. **Edit no memory files.**
This tale does not reinstall SASE, alter the user's running TUI, add an executor or
feature flag, or redesign identity/inventory policy. The repairs are presentation state
and Python glue; shared domain behavior remains in sase-core.

The existing bootstrap snapshot, pane-ID command executors, and post-first-paint
pump-free scheduling are useful and should remain. Retain both pane flags and standalone
synchronous discovery. Do not restart the implementation from scratch.

## What the audit established

Reviewed both phase commits, `1bd855256e` and `7c68d1860c`, all ten child notes, and the
implementation. The original epic has no separate DECISIONS block; its parent phase
supplies the decisions above. The core pin in the first phase includes
`a2abeb1e1b1905780bcd554f55989114c3995292`, whose source repairs archive-corpus schema
compatibility and Clippy warnings; it is not a provider-policy rewrite.

Five deterministic probes against the real methods, using small fake hosts where
appropriate, reproduced these failures:

| Scenario                                                            | Observed result at audit HEAD                                                         |
| ------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| Healthy provider becomes degraded with the same pane ID             | `_stage_artifacts_panes` stages zero widgets and removes none                         |
| Two added widgets; second mount raises                              | Publication returns false, but the first mounted widget is never removed              |
| Worker epoch 0, cache reset advances epoch to 1                     | `_prepare_and_commit_artifacts_snapshot` still publishes epoch 0                      |
| Mark an empty fixed pane with an explicit app snapshot              | `_toggle_artifacts_entry_mark` still calls `provider_source_token` synchronously      |
| Palette opened before a provider appears, then new command selected | Dismiss callback executes zero commands because its captured catalog lacks the new ID |

The existing `test_artifacts_descriptor_snapshot.py` and
`test_artifacts_post_paint_publication.py` suites passed together: **25 tests in 67.40
seconds**. That confirms the missing cases need regression coverage; it does not
establish the original epic's full acceptance.

Additional source findings are part of the same repair: discovery scheduling is only
connected to startup and an otherwise unconnected retry action; no normal refresh path
adopts source changes. Staging passes `display_name=None`, and its post-await check does
not validate source epoch or reapply current scope/display state. Pending jumps only
track tab names, so leaving and returning before completion can incorrectly revive a
superseded jump. The existing test named
`test_open_help_and_palette_refresh_on_publication` opens only help and inspects a fresh
catalog, not actual palette execution. The cold identity test starts on Artifacts and
can pass without any identity call; it does not prove a nonempty cold Agents load ran
off the UI pump.

Integration review: fetched origin/master matched audit HEAD. The three non-epic commits
after `1bd855256e` are `ab352aeeb6` (eager-annotation guard), `a2c9c3ecb5` (monitor
hijack custody), and `3a2b26855d` (workspace/finalizer fixes). They are already
integrated and do not supply a competing provider-publication path. The broader
since-creation review also found the per-load agent memoization and hijack overlay
compatibility fixes already present. Keep those fixes; rerun the drift review before
closing if HEAD/base has advanced.

## 1. Finish the snapshot boundary and palette dispatch

Audit mounted UI calls to descriptor, contract, label, noun, normalization, order, and
document-pane helpers. Pass the committed app snapshot or a captured contract through
each UI helper layer; never fall back to global discovery from a mounted action.
Confirmed missed seams include:

- `actions/artifacts_navigation.py::_toggle_artifacts_entry_mark`
- `actions/refresh_panel.py::_refresh_tab_label`
- `actions/clipboard/_artifacts.py::_document_copy_handlers`
- `_artifact_references.py`, `_artifact_reference_resolution.py`,
  `_artifact_target_support.py`, and `_palette_artifact_previews.py` in that clipboard
  package.

Distinguish genuine standalone/worker discovery from UI capture. Where a background copy
operation needs a contract from the selected generation, capture it before dispatch
instead of resolving a newer global generation. Preserve explicit standalone API
behavior and existing identity-rendering compatibility.

Fix `_base_commands.py::action_open_command_palette`: publication currently updates the
modal's specs but not the dismissal closure's `catalog_by_id`. Resolve a selected stable
ID against the current committed catalog or maintain one shared updated dispatch map.
Preserve filter text, draft, category ordering, and highlighted stable ID. A disappeared
pane must give the unavailable response; a newly published command must execute. Keep
execution by pane ID rather than a stale digit.

## 2. Make discovery refresh and publication transactional

Use `actions/_artifacts_publication.py`, `artifact_tabs.py`, the existing refresh paths,
and `widgets/artifacts/view.py`. Split the publication module into small private
collaborators if needed; keep a single owner for the state machine.

- Connect normal background source invalidation/revalidation and explicit refresh to
  publication. Keep all source-token I/O and provider enumeration in workers. Reuse the
  existing invalidation/token and refresh machinery, with one active pass and at most
  one pending pass per app; do not turn periodic ticks into full enumeration.
- Associate results with their source epoch/token and request lifetime. Reject stale
  results before staging and again after awaits, including the unchanged-descriptor fast
  path. If inputs change during discovery, coalesce a fresh pass without briefly
  advertising the stale result. Preserve authoritative completed-zero results,
  diagnostics, and last coherent state after failure. Make retry reachable through the
  existing refresh flow rather than adding a new keybinding.
- Diff descriptors beyond ID membership. Replace or safely update changed provider
  contracts/specs and healthy/degraded transitions; preserve genuinely unchanged widget
  instances, selection, filter, marks, scroll, and detail state. Stage replacements
  without duplicate DOM IDs and without exposing a target before its widget is ready.
  Commit snapshot, maps, strip, bindings, current contract, and consumer invalidation
  together without awaits.
- Own staged widgets throughout composition/mounting, including partial failure and
  cancellation. Currently the caller receives the staged list only on a successful
  return, so cleanup sees an empty list on failure. Roll back every staged widget and
  retain the old generation; a retry must succeed without duplicate-ID errors. Do not
  swallow a missing view/switcher or an essential commit failure as a successful partial
  publication.
- Re-read lifetime, active pane, navigation, project scope/display snapshot, and
  presentation state after mount awaits. Apply current scope and display names to all
  new/replacement panes before publication. Preserve split/description state. Clear
  pending provider intent on any later navigation, including leaving and returning to
  Artifacts, and on scope changes. Removed active panes fall back to stitches with
  exactly the appropriate deactivate/activate effects. Cancel tasks and discard late
  thread results on teardown.

## 3. Verify the missing behavior and startup evidence

Extend focused tests with deterministic worker/mount gates and the repository wait
helpers. Use real startup scheduling for integration tests, not only AcePage's fast
startup overrides. Test the five confirmed reproductions, then cover source change and
scope/navigation change during staging, failed partial mount followed by retry,
healthy/degraded transitions in both directions, unchanged-widget retention, actual
open-palette selection, real help contents, and teardown at a mount boundary. Assert one
coherent generation across snapshot, strip, mounted panes, bindings, help, and catalog
throughout. Cover both initial tabs, both values of the retired-Agent flag, Files
on/off, and digit overflow. Use representative parameterization without duplicating
every heavy scenario across the full Cartesian product.

Exercise empty mark/copy/refresh and provider copy helpers with discovery/token reads
made fatal on the UI thread, including a newer global-cache result than the committed
snapshot. A passing builder-only test is insufficient.

Prove a cold, nonempty Agents identity/provider load actually occurs after first paint
off the event loop and serial message pump. Do not replace identity with
`unconfigured()` or an empty inventory. Preserve the real loader memoization that landed
during this epic.

Complete the original measurement contract: at least five separate-process runs per
comparison revision with matching Python, Textual, core binding, query, viewport, flags,
and initial tab. Use existing startup telemetry and the narrow
`artifacts.provider_publication` event. Record host load, imported revision/source,
construction, first paint, visible readiness, publication, and first Agents provider
substage, distinguishing cold and warm identity. Use a pre-feature revision such as
`1bd855256e^` in an isolated checkout prepared through the SASE repo workflow; never
reset the working implementation or use another agent's checkout. Existing numbers (one
baseline construction sample 1.474s, provider 0.603s; five post-change construction
samples 0.76-0.80s and deferred worker 0.57-0.60s) are component evidence, not the
required matched end-to-end comparison. Do not claim total startup gains from those
alone.

Run focused suites, then `sase tool run check` (the guarded `just check`). Do not run
`just check-full`. Read screenshot memory, run targeted
`just fix-tui-screenshots -- <affected Artifacts/help/startup selectors>`, and inspect
the report and every changed golden. Use `/sase_monitor` for long verification or
captures with an explicit continuation that still performs this tale's closeout. Keep
infrastructure failures separate from epic defects using independent evidence and the
required follow-up policy; do not expand this into unrelated test repair.

## 4. Close the epic and its parent phase in this implementation turn

The lander already completed follow-up triage and recorded it on sase-1t3.7.1:

- The sole literal `PROPOSED FOLLOW-UP`, from sase-1t3.7.1.1 note #7, was routed as a
  `DISCOVERED ISSUE` to active causal epic **sase-1ah**, whose phase sase-1ah.6 owns
  no-new completion. `sase/sase.yml` omits sibling fingerprint coverage, while
  `monitor/no_new_receipt.py` correctly requires it. No duplicate task was created;
  closed task sase-182 concerns a different project-identity bug.
- The informal cold-identity optimization suggestion from sase-1t3.7.1.2 note #1 was
  declined as a new task: existing measurements establish a pre-existing cold cost, not
  a budget breach or independent regression. Retain its evidence; do not redesign
  identity here. The missing matched startup acceptance above remains epic work.

After repairs and verification, re-read the epic, its children and notes, and the linked
plan; verify no new descendant or untriaged note appeared. Recheck intervening
source/base commits. Record final tests, actual startup numbers, decisions honored,
drift integration, and both follow-up outcomes on **sase-1t3.7** and in the epic's close
note. Then perform this exact closeout:

1. Run `sase bead epic-symbols sase-1t3.7.1`. Resolve every listed symbol by wiring,
   privatizing, a justified non-test pragma, or deletion under Symvision policy; re-key
   only when a concrete still-open later bead needs the exemption. Audit found none, but
   recheck now.
2. Close normally with
   `sase bead close sase-1t3.7.1 --note "<what was verified, repaired, integrated, measured, and how each follow-up was handled>"`.
   If refused, resolve the actual descendant or symbol blocker and retry. Never force a
   successful nested landing or force merely to bypass validation.
3. Run `just symvision` and confirm the whitelist is clean. Open the plans repository
   with `/sase_repo` and set `status: done` in the frontmatter of
   `202611/provider_tab_generation.md` (the linked artifact
   `plan:202611/provider_tab_generation.md`). Read sidecar context only via
   `sase artifact read`; use the opened path for the edit and include its changes in
   finalization.
4. Re-read `sase-1t3.7.1` for the parent link and `sase-1t3.7` for requirements. The
   concrete parent is a **phase**, not a plan. Verify the completed child covers its
   provider-discovery-deferral scope, run `sase bead epic-symbols sase-1t3.7` and
   resolve/re-key any entries, then close only that phase normally with
   `sase bead close sase-1t3.7 --note "<verified child completion and startup evidence>"`.
   Confirm `just symvision` again if the parent cleanup changed the whitelist. Leave
   outer epic **sase-1t3** and its plan status for its already-waiting land agent. If
   the parent is incomplete or ambiguous, note the concrete blocker there and report it
   instead of forcing closure.
5. Finish through `/sase_final` with the completed code and plan-status edits. No close
   step waits for this turn's commit, SHA, publication, or CI.
