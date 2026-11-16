---
tier: epic
title: Publish provider tabs after first paint as one generation
goal:
  Fixed Artifacts panes paint without provider discovery, and every TUI consumer adopts
  each complete provider-tab generation together.
parent_bead: sase-1t3.7
phases:
  - id: snapshot-consumers
    title: Pin Artifacts consumers to an explicit descriptor snapshot
    depends_on: []
    size: medium
    description:
      "snapshot-consumers: introduce the TUI descriptor snapshot and pure lookup
      boundary; route bindings, Artifacts composition, help, command catalogs, and
      navigation through the app's committed snapshot. Keep discovery timing unchanged
      in this preparatory phase. Make command selections target stable pane IDs,
      preserve standalone synchronous discovery, and test isolation from a newer
      discovery-cache result."
  - id: post-paint-publication
    title: Discover after first paint and publish mounted generations coherently
    depends_on:
      - snapshot-consumers
    size: medium
    description:
      "post-paint-publication: bootstrap fixed descriptors without discovery; schedule
      coalesced provider discovery after first paint; prepare missing panes and publish
      one complete generation on the UI thread. Preserve navigation, pane state,
      diagnostics, retry, and teardown behavior. Add delayed-discovery integration
      coverage and startup measurements, including the cold Agents identity path, and
      leave closure evidence for the land agent."
proposed_by: bbugyi200.athena.sase-1t3.7
proposed_with: codex/gpt-6-astra
create_time: 2026-11-16 11:11:47
status: wip
---

- **PROMPT:**
  [prompts/202611/provider_tab_generation.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/provider_tab_generation.md)
- **PARENT:**
  [202611/tui_storm_trust_heap_recovery.md](https://github.com/sase-org/sase--plans/blob/main/202611/tui_storm_trust_heap_recovery.md)

# Publish provider tabs after first paint as one generation

## Scope and evidence

This is a child epic of **sase-1t3.7**, the `provider-discovery-deferral` phase of
`plan:202611/tui_storm_trust_heap_recovery.md`. Read that artifact and this phase's bead
before implementing. The phase was in progress with no prior implementation notes when
inspected. Trust memoization (`157ae99cf0`) and the trust-root repair (`ac19f907ae`)
have landed; the provider-deferral change has not.

Read the consolidated research and its Codex report through `sase artifact read`:

- `research:202611/tui_hijack_storm_trust_root_and_heap/tui_hijack_storm_trust_root_and_heap.md`
- `research:202611/tui_hijack_storm_trust_root_and_heap/tui_hijack_storm_trust_root_and_heap__cdx.md`

A fresh-process component measurement at `ac19f907ae`, explicitly importing this
checkout with `PYTHONPATH=src`, found:

| Measurement                                                  | Seconds |
| ------------------------------------------------------------ | ------: |
| App module import                                            |   2.020 |
| `AceApp(auto_start_axe=False)` construction                  |   1.474 |
| Provider discovery inside that construction                  |   0.603 |
| First `AgentIdentitySnapshot.current()` after construction   |   0.720 |
| Cold identity in a separate process without app construction |   1.140 |

Environment: Python 3.14.7, Textual 8.0.1, sase-core-rs 0.44.2. Discovery returned 18
records and one issue. The measurement wrapped `load_project_provider_records` to
count/time calls; only `register_ace_session` and `current_toast_session` were stubbed
to avoid constructor-only session records. These are single component samples, not
live-TUI latency percentiles or an end-to-end improvement claim. The second identity
call in its separate process took less than 0.001 seconds; retain the distinction
between cold discovery and the existing warm cache.

The trust repair has removed most of the original multi-second regression. Limit this
work to the presentation boundary that protects first paint against provider count
growth. Two medium phases separate consumer migration from asynchronous widget
publication; neither requires a backend redesign.

## Constraints

- Honor the outer epic's final decisions: `trust_ack_compat=cutover`,
  `hotfix_reinstall=yes` for its storm phase only, and `tui_perf_memory=no`. **No memory
  note may be edited.** This child epic needs no reinstall gate.
- Read `ace.md`, `tui.md`, `tui_perf.md`, and `lint_and_test.md` through
  `/sase_memory_read`. Follow pump-free scheduling, activity gates, stale-result checks,
  and teardown cancellation.
- This snapshot is TUI presentation state. Keep provider-policy semantics, trust
  identities, agent-name resolution, and shared inventories unchanged. If a shared
  backend defect is discovered, record a `PROPOSED FOLLOW-UP:` on sase-1t3.7; shared
  domain changes belong in sase-core under the established repo workflow.
- No new feature flag, persistent snapshot store, executor expansion, archive scan, GC
  tuning, or new profiling framework. Preserve the existing
  `artifacts_agent_pane_retired` and `artifacts_files_pane` combinations.
- No agent restarts or alters the user's live TUI. Timing verification uses a separate
  instance. Use `/sase_monitor` for long commands and waits.
- Child phase workers close only their assigned child phase. They do not create task
  beads or close sase-1t3.7 or sase-1t3; follow-ups and closure evidence go in bead
  notes. The child epic's land agent owns completion of sase-1t3.7 after the child work
  settles, and must leave the outer epic sase-1t3 open.

## Current seams and failure modes

1. `actions/_state_init_late.py` calls `build_app_bindings`, which calls
   `resolve_artifacts_subtabs`. The cold resolver computes a provider source token and
   walks projects synchronously. Moving only `_artifact_subtab_bindings` is
   insufficient: `_app_layout.py` constructs a hidden `ArtifactsView`, whose constructor
   resolves the same list.
2. `artifact_tabs.py` has a process discovery cache with background refresh and an epoch
   guard. Its latest result is not an app publication boundary.
   `ArtifactsView._descriptors` remains the original tuple while help, catalogs,
   normalization, and cycling can read a newer tuple.
3. `_artifact_tab_descriptors.build_artifacts_strip(())` already builds fixed panes,
   including flag-dependent panes and their positional digit assignments. Provider tabs
   are inserted before Files, so a later generation can renumber Files. The digits and
   visible labels must change together.
4. `commands/catalog.py` currently records provider commands as
   `show_artifacts_digit(N)`. A palette opened before a generation change can execute
   that old digit against a new list and select another pane.
5. `provider_source_token()` also has synchronous cold I/O. A descriptor getter that
   still calls it is not a safe bootstrap or render-path getter.
6. `AgentIdentitySnapshot.current()` discovers configured agent-sidecar owner roots
   through `resolve_sync_targets`. Status normalization calls it in
   `models/_agent_status_apply.py`. Removing construction's config warmup must not hide
   a large regression in the first Agents load's `provider` substage.

## Phase snapshot-consumers

Introduce a small immutable snapshot type for the descriptor tuple, generation identity,
discovery status, and the source token/epoch needed for validation. Distinguish
**bootstrap/pending**, **completed with zero providers**, and **failed/retryable
discovery**. Reuse `ArtifactsTabDescriptor` and its compiled contracts; do not invent
another provider model. Add pure operations for exact lookup, canonical/legacy
normalization, ordered IDs, contracts, and digits.

Keep the synchronous discovery API for CLI, diagnostics, and deliberate standalone
callers. Add an explicit fixed-only bootstrap factory that calls no project enumeration
or provider-token computation and does not populate the authoritative discovery cache.
Separate discovery results from the app's current presentation snapshot; background
cache changes alone must never alter UI reads.

In this phase, initialize the app's committed snapshot from real synchronous discovery
once, preserving startup behavior until the next phase. Pass that snapshot explicitly
through the following consumers and their helper layers:

| Surface                        | Primary seams                                                                                                                      |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------- |
| App bindings and layout        | `keymaps/bindings.py`, `actions/_state_init_late.py`, `_app_layout.py`                                                             |
| Artifacts panes and tab strip  | `widgets/artifacts/view.py`, `widgets/panel_tab_strip.py`                                                                          |
| Navigation and availability    | `actions/artifacts_navigation.py`, `artifact_tabs.py`, `commands/context.py`, `commands/_availability_artifacts.py`                |
| Help                           | `modals/help_modal/patches_artifact_bindings.py`, `patches_bindings.py`, `modal.py`, and help-opening actions                      |
| Commands                       | `commands/catalog.py`, `commands/types.py`, `commands/execute.py`, `_base_commands.py`, command-line and clipboard catalog callers |
| Other visible labels/contracts | `widgets/tab_quickstart.py`, `widgets/patch_onboarding.py`, clipboard/query helpers reached by current-pane actions                |

Audit indirect calls to `normalize_artifacts_subtab`, `artifacts_pane_key`,
`artifacts_subtab_order`, descriptor/contract lookup, and `resolve_artifacts_subtabs`.
An app-owned path must use the app snapshot, including the duplicate
`current_artifacts_pane_key` implementations in `app.py` and the navigation mixin.
Explicit snapshot arguments or narrow app helpers are preferable to global "active app"
state or an ambient context variable. Two app instances must not overwrite each other's
published generation. Standalone adapters may retain explicit synchronous resolution;
never silently fall back to it from mounted UI.

Generate bindings and help digit descriptions from the same tuple. Preserve custom app
bindings and existing priority/unbound semantics. Catalog entries retain their stable
`artifacts.<pane-id>` identity and execute by pane ID, with the digit kept only as the
displayed shortcut. If a pane disappears before an old catalog selection executes,
report that it is unavailable; do not reinterpret its digit. Refresh already-open
help/catalog surfaces when publication arrives in the next phase, preserving
filter/draft and selection by stable ID.

Tests for this phase must prove that a changing discovery cache cannot change the
snapshot used by any of these consumers, app instances remain isolated, standalone
resolution still returns configured providers, and a stale command spec cannot jump to a
different pane. Preserve existing digit overflow, degraded provider, pane-flag, keymap
override, legacy ID, and command-availability tests. Adapt
`src/sase/ace/testing/_startup.py` and `ace_page.py` to provide explicit fixture
snapshots where necessary rather than relying on five patched resolver imports. Keep
production-path startup tests available for the next phase.

## Phase post-paint-publication

Replace synchronous app discovery with the bootstrap snapshot before binding
construction. `ArtifactsView` must receive it directly and compose fixed panes only.
Normalization and pane-constructor helper calls must not rediscover providers. Represent
pending discovery in the existing Artifacts status/diagnostic area; fixed panes remain
usable and pending is not shown as "no providers configured."

Launch discovery from the existing post-first-paint startup path
(`actions/_startup_loads_core.py` / `_startup_mount.py`). The callback must be thin and
synchronous: spawn an app-owned pump-free task, with provider enumeration and
source-token reads in a worker. Do not await slow work on a Textual message or timer
callback. Use the existing worker facilities; bound to one in-flight discovery and one
coalesced pending refresh per app, without adding an executor to increase throughput.
Keep the visible Agents load and fixed Artifacts ready milestones independent of
provider completion.

Use the existing invalidation/token machinery from a background refresh path. Do not add
disk probes to key presses, help rendering, or catalog building. Make completion
notification explicit: a cache refresh alone cannot update the app, and returning the
last cached tuple is not proof that a requested refresh has completed. Reject superseded
source epochs and results after app teardown; coalesce a new pass when inputs change
during discovery. Preserve diagnostics for missing/broken providers and the existing
retry behavior when token discovery fails. Retain the last coherent generation on an
unexpected refresh failure.

Publish in two steps so asynchronous mounts cannot expose a half-built generation:

1. Prepare new or replacement pane widgets on the UI thread without advertising them in
   the current tab strip, bindings, navigation, or catalogs. Await mount completion from
   the pump-free task. Retain unchanged pane widgets, selections, filters, marks,
   scroll, and detail state. Propagate the current keymap registry, project
   scope/display snapshot, split mode, and description mode to new panes.
2. After every await, re-check generation, app lifetime, current active pane, project
   scope, and user navigation. In a synchronous commit with no await, replace the app
   snapshot, view descriptor maps, tab strip, digit bindings, active pane/contract, and
   help/catalog invalidation state. Preserve selection by stable pane ID; if it was
   removed, select the existing fixed default with correct deactivate/activate hooks.
   Remove obsolete widgets after they are no longer reachable. On preparation failure,
   clean up staged widgets and retain the previous generation; a retry must not create
   duplicate DOM IDs.

During bootstrap and preparation, digit presses resolve only against the currently
displayed committed generation. Digits with no displayed target do nothing and are never
replayed against a later ordering. An explicit provider pane-ID request received while
discovery is pending may be retained as a single pending target; apply it only if no
later user navigation superseded it and the completed generation contains that ID. After
completion, a missing ID gets an unavailable response rather than silent fallback. Keep
intended legacy fallback behavior for genuinely stale persisted IDs distinct from
pending discovery.

Handle provider additions, removals, descriptor changes, and healthy/degraded
transitions through the same publication path. Keep completed zero-provider results
authoritative for their valid token, without treating the bootstrap tuple as a completed
discovery. Cancel app tasks at teardown and discard late worker results; thread
cancellation is not a guarantee that the computation stops.

## Verification and acceptance

Use deterministic worker gates/events and the repository's test wait helpers, not
arbitrary sleeps or tight wall-clock assertions. Add focused coverage that uses the real
startup scheduling path rather than the AcePage fast-startup resolver overrides:

- Parameterize initially visible Agents and Artifacts, including hidden `ArtifactsView`
  construction. While a fake provider is blocked, first paint, fixed-pane navigation,
  and help/command opening succeed. Neither constructor nor compose/render/key handlers
  call provider discovery or its cold token read.
- Capture generation, pane IDs, shortcuts, and command targets from all five consumers
  before, during, and after a gated publication. No advertised target lacks a mounted
  pane, and a digit always matches its displayed label. Cover Files enabled/disabled,
  retired Agent pane on/off, and more than nine panes.
- Exercise navigation and project-scope changes during preparation, a pending
  provider-ID jump, an already-open help/palette, provider removal while selected, retry
  after failure, source invalidation during a worker, zero providers, and shutdown
  before completion. No duplicate discovery storm, stale commit, state loss in retained
  panes, or late widget mutation is allowed.
- Gate cold identity/Agents normalization separately and verify that it executes after
  first paint off the UI pump. Compare cold and warm identity/provider timings with and
  without construction-time provider warming. Do not substitute
  `AgentIdentitySnapshot.unconfigured()` or an empty inventory to make it fast.

Relevant suites include `test_artifact_tab_discovery.py`, `test_artifact_tab_digits.py`,
`test_artifact_pane_tab_navigation.py`, `test_artifacts_files_subtabs.py`,
keymap/help/command suites, and startup loading and observability tests. Keep focused
tests with their implementation phase. Run `sase tool run check` after source changes.
Run targeted `just fix-tui-screenshots -- <selectors>` for affected
Artifacts/help/startup rendering, inspect every resulting golden change, and read
`tui_screenshot.md` first. Do not run `check-full` without an explicit separate
instruction.

For final evidence, compare at least five separate-process startup runs per revision
using the existing startup capture/telemetry tooling, matching Python, Textual, core
binding, query, viewport, flags, and initial tab. Explicitly set `PYTHONPATH=src` for
ad-hoc component probes and verify the imported source SHA. Record host load and
distinguish construction, first paint, visible readiness, provider publication, and the
first Agents `provider` substage. Reuse existing trace/startup fields; if publication
timing is needed, add one narrow trace event rather than a new telemetry system. Run
long captures with `/sase_monitor`.

The structural requirement is zero provider enumeration before first paint and one
coherent UI generation throughout discovery. Expect construction to lose the measured
provider cost; do not claim an overall startup gain if useful content simply became
slower to appear. If the post-trust identity cost remains material, report its
independent measurement as follow-up instead of expanding this change into shared
inventory or identity semantics.

The land agent records final before/after numbers, tests, and any identical clean-base
failures on sase-1t3.7. Such baseline failures become `PROPOSED FOLLOW-UP:` notes and do
not leave an otherwise completed phase open. Before closing, run
`sase bead epic-symbols sase-1t3.7`, resolve every entry or re-key it to a still-open
owner, and close only sase-1t3.7 with a verification note after this child epic's work
is complete. Do not close outer epic sase-1t3.
