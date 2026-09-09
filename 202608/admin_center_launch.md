---
tier: epic
title: Move Launch Control into the SASE Admin Center
goal:
  Launch configuration becomes a polished, lazy Config sub-tab in the SASE Admin Center,
  and every Launch Control entry point—including the `,m` leader key—opens Config
  directly on Launch without losing current editing, override, navigation, refresh, or
  responsiveness guarantees.
phases:
  - id: launch_pane
    title: Extract reusable Launch content and lifecycle contracts
    depends_on: []
    size: medium
    description:
      "launch_pane: separate Launch Control content, state, and lifecycle behavior from
      its standalone modal host without changing the production route."
  - id: launch_integration
    title: Integrate and route the guarded Config Launch sub-tab
    depends_on:
      - launch_pane
    size: medium
    description:
      "launch_integration: add the lazy Launch child behind a temporary beta flag and
      route every Launch Control opener through the Admin Center Config host."
  - id: launch_cutover
    title: Polish, verify, and make Config Launch unconditional
    depends_on:
      - launch_integration
    size: medium
    description:
      "launch_cutover: finish responsive visual and interaction coverage, remove the
      standalone production route and temporary flag, and verify the combined epic."
proposed_by: bbugyi200.athena.sase-ri.land.w2
bead_id: sase-rp
create_time: 2026-09-09 19:49:44
status: wip
---

- **PROMPT:**
  [prompts/202608/admin_center_launch.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/admin_center_launch.md)
- **BEAD:**
  [sase-rp](https://github.com/sase-org/sase--beads/blob/main/pages/sase-rp/README.md)

# Plan: Move Launch Control into the SASE Admin Center

## Outcome and interaction design

The Admin Center keeps its six top-level tabs. Config expands from five to six children
in this order:

1. **XPrompts** — reusable prompts and workflows.
2. **Snippets** — prompt snippets and their relationships.
3. **Glossary** — project terminology and relationships.
4. **Memory** — scoped memory notes and publishing.
5. **Launch** — launch models, aliases, effort, concurrency, provider routing, and tmux
   Agent controls currently exposed by Launch Control.
6. **Misc** — the layered schema-driven settings browser.

This preserves every existing Config child's relative position, keeps the general
catch-all last, and places the focused launch settings immediately before it. A normal
first visit to Config continues to open XPrompts, while later ordinary visits resume the
last Config child. The `,m` leader chord is an explicit direct-entry request: it opens
the Admin Center on Config and selects Launch even when another Admin Center or Config
selection was remembered. The successful direct entry then participates in the same
session history as mouse and bracket navigation.

Every current Launch Control entry point converges on that same route: `,m`, repeat-last
leader dispatch, the default-model/alias/provider status pills, and the
`OpenLaunchControl` notification action. Keep the configurable leader action ID
`models_panel` and its legacy `temporary_llm_override` alias working so existing user
keymaps do not require migration; only their destination and user-facing label change.
Help, command search, tooltips, comments, and configuration/schema prose should call the
destination **Config › Launch** or **Launch settings**, rather than implying that a
standalone Launch Control screen still opens. As a global action, it stays in the help
modal rather than the conditional footer.

Visually, Launch should feel native to the two-level Admin Center rather than like a
modal placed inside a modal:

- Reuse `PanelTabStrip` and Config's teal hierarchy for the new tab, with `Launch` at
  normal/compact widths and a concise `Run` micro label on very narrow terminals.
- Remove the standalone double border, centered `Launch Control` title, and duplicate
  close hint when embedded. Preserve the useful information from that title as a slim
  context line: the current bucket breadcrumb and provider-disable status remain visible
  without competing with the Admin Center heading and Config strip.
- Let the launch list use the available Admin Center width and height. Keep it as the
  flexible scroll region while the selected-row explanation and context-sensitive action
  legend remain readable. The embedded legend lists local actions; the Admin Center
  footer owns close and section-history guidance.
- Retain the existing semantic color language for launch settings, built-in aliases,
  user aliases, override state, ownership, effort, and providers. The surrounding teal
  frame supplies hierarchy without flattening Launch's cyan, violet, orange, green, and
  gold distinctions.
- At 120x40 the pane should breathe and use the available width; at 70x32 and other
  narrow sizes the tab strip, columns, descriptions, and action legend should compact or
  wrap deliberately, never clip controls or create a second horizontal rail.

`j`/`k`, arrows, `Ctrl+N`/`Ctrl+P`, apostrophe jump, bucket `h`/`l`,
override/edit/reset, history, effort, runner-limit, provider, and tmux Agent actions
retain their current meaning. `[`/`]` remain Config-child navigation and
`Tab`/`Shift+Tab` remain top-level Admin Center navigation because Launch has no
relationship traversal. `Escape` and `q` close the Admin Center when no launch write is
active. Nested pickers, edit previews, history, provider routing, and tmux Agent modals
remain focused overlays and return to the same Launch row and bucket.

## Architecture and invariants

- Extract a reusable `LaunchPane`-style `Widget` from the current `ModelsPanel`
  `ModalScreen`. Keep the existing focused `models_panel_*` mixins and data facades;
  move only composition, host/lifecycle boundaries, and shared initialization needed to
  let those behaviors run in either host. A thin `ModelsPanel` adapter preserves the
  existing constructor, result, monkeypatch seams, and focused tests while the guarded
  route exists. After cutover, no production entry point may push that adapter; retain
  the compatibility export only if removing it would break supported internal callers or
  materially weaken focused component tests.
- Define a small typed Launch host contract for close requests and a typed
  `LaunchPaneSessionState`. The session records a stable row identity plus the active
  bucket, not a fragile visual index. Restore after the asynchronous provider/alias
  snapshot resolves, and fall back to the first enabled row if an alias, bucket, or
  setting disappeared. Cache the mounted pane for the Admin Center modal's lifetime and
  preserve its cursor across other Config and top-level tabs; use the process-owned
  Admin Center session to preserve it across close/reopen.
- Extend `ConfigSubTab`, `ConfigHubEntry`, the immutable catalog, and the lazy factory
  with `launch`. Construct and load Launch only when requested by `,m`, a direct opener,
  a click, or `[`/`]`; opening Config on XPrompts must not import or start Launch's
  snapshot workers. A failed Launch mount leaves the prior child visible and permits a
  retry, exactly like the existing Config children.
- Give Launch the same visibility contract as other cached children. Hidden panes must
  not steal focus or repaint the visible child. Countdown refreshes should pause or
  become no-ops while hidden, then revalidate/repaint on activation. Snapshot and edit
  workers must remain off the event loop, reject stale completions where applicable, and
  be cancelled or safely ignored after unmount.
- Preserve the old modal's write-safety contract across the richer host. While an
  override, clear, persistent edit, effort, runner-limit, threshold, or provider write
  is active, reject Config-child changes, top-level tab changes, and Admin Center close
  with the existing warning instead of hiding or cancelling an in-flight mutation.
  Read-only loads may be hidden or cancelled normally. Route this through explicit
  `can_deactivate`/`can_close`-style host checks so parent priority bindings cannot
  bypass the guard, including when Launch was mounted and later hidden.
- Consolidate successful Launch mutations behind one `_mark_changed`-style seam. It
  updates the compatibility `ModelsPanelResult`, requests any existing Agents refresh,
  and tells the app to refresh the default-model, alias-override, and provider-disable
  indicators. Provider-routing changes must also invalidate the cached launch default.
  Refresh on successful mutation rather than relying only on standalone-modal dismissal
  so changes made after entering Launch through ordinary Admin Center navigation cannot
  leave stale top-bar state.
- Route all openers through the existing allocation-only `_open_config_center` path with
  `ConfigHubEntry(subtab="launch")`. Do not read config, resolve models, or probe
  providers in the `,m` key handler. Keep repeat-last leader bookkeeping, Admin Center
  current/alternate history, and non-blocking history persistence unchanged.
- This is Textual presentation, navigation, lifecycle, and Python glue. Continue using
  the existing launch/config/provider facades; no shared backend behavior should be
  reimplemented outside the Rust core boundary.
- Stage the user-reaching route with a beta flag created only through
  `sase flag new admin_center_launch_subtab`. Author its semantics as follows: enabled
  means Config exposes Launch and every Launch Control opener selects it; disabled means
  Config retains its current five children and every opener pushes the standalone
  `ModelsPanel`; remove it when behavioral parity, busy-write safety, status refresh,
  responsive visuals, navigation, and full verification pass. Test both states until the
  cutover deletes the disabled branch, removes the registry entry, and closes the
  generated flag bead.

## Phase: `launch_pane` — Extract reusable Launch content and lifecycle contracts

Refactor Launch Control into reusable content without changing what users open yet. This
phase owns the `models_panel*` family, the new pane/host/session contract, its styles,
exports, and focused component tests; it does not edit the Config catalog or leader
routing.

- Move the current screen's initialization, composition, bindings, option-list
  navigation, mutation flows, worker dispatch, and teardown onto a reusable Launch
  content widget. Keep modal-only dismissal and result delivery in a thin `ModelsPanel`
  adapter that hosts the pane and forwards focus/visibility/close behavior.
- Preserve every existing capability: launch-setting rows, default/built-in/custom
  aliases and buckets, temporary and persistent model changes, default effort, max
  runners, big-epic threshold, provider routing/disables, tmux Agent settings, alias
  history, jump mode, config commit offers, toasts, and Agents refresh requests.
- Introduce stable selection/bucket session restore and the explicit host lifecycle
  methods. Make first load and all secondary loads worker-backed as today, guard
  programmatic OptionList highlights, recapture the live selection after asynchronous
  completions, and ensure teardown cancels every owned worker and timer.
- Separate standalone and embedded presentation in the component contract. Preserve the
  current standalone appearance during this phase, while exposing an embedded class or
  mode that can omit outer chrome and size from its actual host rather than the current
  110x39 modal constants.
- Convert focused tests to exercise `LaunchPane` directly where that improves signal,
  retaining narrow adapter tests for result delivery, close refusal during writes,
  focus, and public compatibility. Keep coverage for all existing action, bucket,
  layout, effort, provider, runner-limit, threshold, history, jump, tmux Agent, failure,
  and unmount paths.

## Phase: `launch_integration` — Integrate and route the guarded Config Launch sub-tab

After the reusable pane is stable, create the beta flag with `sase flag new` using the
authored semantics above and implement both explicit branches.

- Add Launch before Misc in the typed Config catalog and session state. The enabled
  factory mounts `LaunchPane` with the Config hub as host, applies embedded styling, and
  participates in existing lazy/cached/failure-safe switching. Keep XPrompts as the
  default for an unseeded Config session.
- Extend Config hub and Admin Center navigation with the busy-write deactivation and
  close checks. Prove that `[`/`]`, top-level digits, `Tab`/`Shift+Tab`, clicks,
  alternate-section navigation, `q`, and `Escape` are deterministic both while idle and
  during a write; no parent binding may dismiss or hide a busy Launch pane.
- Replace direct standalone opens in leader dispatch, `action_open_models_panel`, all
  three status indicators, and `OpenLaunchControl` handling with the central Config
  direct-entry route when enabled. Preserve custom `models_panel` mappings, the legacy
  action alias, `,,` repeat behavior, and the notification action contract.
- Extract the app-level launch-indicator refresh helper and connect it to the pane's
  successful-mutation seam. Test ordinary Config navigation as well as direct entry so
  alias, default, provider, runner-limit, and effort changes are visible immediately and
  provider changes invalidate cached default resolution exactly once per completion.
- Update the Admin Center Config description/home orientation, help modal, command
  catalog, status-pill tooltips, default-config comments, schema descriptions, exports,
  and tests that enumerate five Config children or name a standalone destination. Do not
  add `,m` to a conditional footer and do not silently rename/remove the
  user-configurable `models_panel` keymap ID.
- Test enabled and disabled states for exact catalog order, direct target precedence,
  ordinary resume, lazy construction/load counts, cache reuse, mount failure/retry,
  row/bucket resume, focus restoration from nested overlays, indicator refresh, and
  notification/pill/leader entry. The disabled state must remain behaviorally equivalent
  to today's standalone route for straightforward flag removal.

## Phase: `launch_cutover` — Polish, verify, and make Config Launch unconditional

Treat the production interaction and pixels as acceptance criteria, then remove the
temporary branch rather than leaving a permanent preference.

- Finish the embedded layout and re-home representative production visual scenarios
  under the Admin Center frame: populated/default, active model and effort/runner
  overrides, provider disabled, custom/built-in bucket and drilled-in breadcrumb, empty
  custom section, long/wrapped pool description, jump hints, and a nested edit/provider
  overlay. Include at least 120x40 and 70x32 coverage, plus dark/light coverage where
  the visual harness supports it. Remove obsolete standalone root snapshots only after
  equivalent embedded coverage exists; keep focused nested-modal goldens that still
  exercise reachable overlays.
- Add end-to-end keyboard and mouse tests beginning at the ACE app: `,m` opens Admin
  Center → Config → Launch, a remapped `models_panel` action and `,,` do the same,
  indicator clicks and notifications converge there, bracket/main-tab navigation is
  correct, and closing returns focus to the originating ACE pane. Verify row/bucket
  selection across nested overlays, child/main-tab switches, Admin Center close/reopen,
  and stale/missing identities.
- Stress write safety and lifecycle: attempt sub-tab switches, top-level switches, and
  close during every mutation family; verify the write completes once, the pane remains
  visible/focused, indicators and Agents refresh once, hidden read-only work cannot move
  visible focus, and unmount leaves no worker/timer completion touching dead widgets.
- Add lazy/performance regression assertions proving `,m` performs no data-scaled work
  in its key handler, unseeded Config still constructs only XPrompts, Launch data loads
  once per cached pane, countdown refresh is inactive while hidden, and rapid
  Config/main-tab changes cannot apply stale selection. Run the documented TUI trace or
  `j`/`k` responsiveness bench and retain the p95 < 16 ms target.
- Delete the flag's Off branch, make Config Launch and all direct-entry routes
  unconditional, remove the registry definition, and close the generated flag bead in
  the same change. Remove the standalone production opener and stale user-facing Launch
  Control wording; keep only intentional compatibility symbols with tests explaining why
  they remain.
- Run `just install` before verification. Iterate with the focused Models/Launch,
  Config-hub, keymap, notification, indicator, session, and visual tests, then run
  `just test-visual`. Because this is an epic landing with broad TUI navigation and
  layout changes, run `just check-full` only through `/sase_monitor` with required
  `TESTING`/`TESTED` statuses and a follow-up action that inspects and resolves the
  result. Confirm `tools/check_feature_flags` reports neither an orphan definition nor a
  live removal bead before landing.

## Acceptance criteria

- Config visibly contains exactly XPrompts, Snippets, Glossary, Memory, Launch, and Misc
  in that order; an ordinary first Config visit still opens XPrompts.
- `,m`, a configured replacement for `models_panel`, repeat-last leader dispatch, Launch
  status pills, and `OpenLaunchControl` all open the SASE Admin Center on Config with
  Launch selected. No production entry point opens the standalone screen.
- Launch retains every current setting, alias/bucket, override, provider, history, jump,
  effort, runner-limit, threshold, edit/commit, and tmux Agent workflow, including
  success/error toasts and nested-modal focus restoration.
- Launch is lazy, cached, failure-safe, and identity-preserving across Config-child,
  top-level, close/reopen, data reload, and disappearing-row cases. Opening Config on a
  different child starts no Launch work.
- No active launch write can be hidden, cancelled by navigation, or bypassed by parent
  close bindings. Successful mutations refresh all affected top-bar indicators and
  Agents state without waiting for a special dismissal path or applying twice.
- The embedded pane has one clear visual hierarchy, no redundant modal frame/title or
  close legend, readable semantic accents, an obvious bucket/provider context, and no
  clipping or unusable rails at normal and narrow sizes.
- No synchronous disk, config, provider, history, or subprocess work enters a key
  handler, render path, or Textual pump callback; hidden/closed panes do not repaint,
  steal focus, or apply stale worker results, and measured navigation remains within the
  documented responsiveness target.
- The temporary beta flag and disabled standalone route are removed, focused tests,
  production PNG snapshots, `just test-visual`, and monitored `just check-full` pass,
  and help/config/schema wording describes the final Config › Launch destination.
