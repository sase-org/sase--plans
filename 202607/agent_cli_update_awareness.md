---
tier: epic
title: Provider-aware comprehensive update experience
goal: 'ACE''s cached startup and periodic update checks include supported agent CLI
  providers without adding UI-thread or per-tick network cost, and the global ,U action
  safely updates exactly the provider CLIs identified by the latest completed background
  check while presenting a distinct, polished update state.

  '
phases:
- id: provider_update_snapshot
  title: Provider-aware background update snapshot
  depends_on: []
  size: medium
  description: '''Provider-aware background update snapshot'' section: extend the
    durable update snapshot and automatic checker with efficient, source-independent
    agent CLI discovery, caching, and candidate-only revalidation.'
- id: comprehensive_update_flow
  title: Snapshot-gated comprehensive update flow
  depends_on:
  - provider_update_snapshot
  size: medium
  description: '''Snapshot-gated comprehensive update flow'' section: make global
    ,U carry the last completed background candidate set into one safe preview, tracked
    execution flow, and coherent completion or restart result.'
- id: indicator_and_product_polish
  title: Distinct update surfaces and release validation
  depends_on:
  - provider_update_snapshot
  - comprehensive_update_flow
  size: small
  description: '''Distinct update surfaces and release validation'' section: finish
    the segmented agent-CLI badge, toast/help/docs language, visual snapshots, performance
    assertions, and end-to-end validation.'
create_time: 2026-07-20 10:22:41
status: done
bead_id: sase-83
---

# Plan: Provider-aware comprehensive update experience

## Context and product contract

ACE already schedules its first update check after first paint and then attempts another check every ten minutes. The
worker maintains a durable snapshot for SASE core and installed plugins, locally revalidates that snapshot on cheap
periodic ticks, and permits a full network-backed recompute only on the longer `ace.updates.recompute_interval_minutes`
cadence. Separately, the Agent CLIs sub-tab and `sase agent-cli` commands already share provider metadata, installation
detection, a six-hour npm latest-version cache, safe update planning, and sequential execution.

This feature should join those existing systems rather than add another timer, network oracle, or update-command
implementation. Its user-facing contract is:

- A completed automatic startup/periodic check reports SASE/core/plugin and supported agent CLI updates as one coherent
  snapshot, while retaining counts by domain.
- The global `,U` shortcut captures the provider-candidate names from the most recently _completed_ automatic background
  result at the instant the key is pressed. Only those names may enter that invocation's provider update plan. A
  provider update discovered later by the user-triggered Updates-pane load, or by a background check that finishes after
  the keypress, must not be added.
- Candidates are revalidated against the live provider inventory before the confirmation is shown. A candidate that was
  removed, updated externally, or is no longer safely actionable is dropped or shown as an explicit manual/ skipped
  item; stale data is never used to execute a guessed command.
- When no prior automatic result identified a provider update, `,U` preserves its current SASE/core/plugin behavior and
  performs no provider update work. The direct pane-wide `A` action remains the deliberate way to inspect/update current
  provider state regardless of the automatic snapshot, and pane-wide `u` remains the SASE/core/plugin-only action.
- Known outdated Homebrew, unknown-provenance, non-writable, and otherwise manual-only CLIs still count as available
  provider updates. They appear in the preview with the exact suggested command or vendor documentation, but SASE never
  escalates privileges or invents an unsafe mutation.
- Source failures degrade independently. A failed provider lookup must not hide valid SASE/plugin updates or erase the
  last known-good provider candidate state; a failed SASE/plugin source likewise must not suppress valid provider
  results. Freshness/error metadata must distinguish “checked and current” from “could not check.”

This is ACE session status, presentation, and orchestration over the existing Python agent-CLI service, so it stays on
the established Python/TUI side of the Rust core boundary. No new provider-specific update logic, CLI-provider special
case, or `sase update` CLI semantic change is part of this work.

## Design decisions

### Snapshot shape and freshness

Add a lightweight, serializable provider-update candidate record to the shared update snapshot. It should contain stable
provider identity and the installed / latest versions needed for truthful presentation and revalidation, but not treat a
cached mutation command as executable authority. Preserve separate derived counts for SASE/core/plugins and agent CLIs,
plus the existing `has_core_update` signal. Bump the snapshot schema; older snapshots should be a safe cache miss, not
partially decoded state.

Full recomputation should call the existing `collect_agent_cli_statuses()` service, filter to installed/outdated CLIs,
and retain source completeness and freshness. The Updates pane's own online load currently writes the shared SASE/plugin
snapshot, so it must write the same composite shape using the agent-CLI statuses it already collected; it must never
clobber provider candidates with an apparently empty legacy snapshot.

Cheap revalidation remains drop-only. With zero cached provider candidates it runs no provider subprocesses and no
provider network requests. With candidates it re-probes only those candidates (bounded by the existing provider command
timeouts), compares them with cached latest versions, and conservatively keeps a known candidate when a transient local
probe cannot prove it current. New provider updates are discovered only by a full recompute. A successful direct or
comprehensive update schedules an immediate off-thread revalidation so the badge does not wait for the next ten-minute
tick.

### Cadence and responsiveness

Reuse the one existing automatic-update worker, in-flight guard, timer, and configuration. Do not introduce a second
periodic callback or provider-specific cadence knob. Startup still occurs after first paint; timer callbacks remain thin
and synchronous; all config/disk parsing, provider registry inspection, subprocess probes, and remote latest-version
work remain in the worker thread.

The ten-minute tick reads and locally revalidates the composite snapshot. A network/full-inventory recompute remains
eligible only after the existing longer recompute interval. Provider latest-version resolution continues to use its
six-hour cache, so even an eligible full snapshot recompute does not imply an npm request. Independent sources should be
evaluated with bounded work and failure isolation so a slow/unavailable registry cannot delay first paint or remove
results from another source.

### `,U` snapshot ownership and races

Applying an automatic result on the UI thread should atomically update both the badge and an immutable in-memory
provider candidate set. `,U` copies that set before opening the Admin Center and passes it through the Config Center
into the Updates pane as a one-shot comprehensive-update request. The pane's normal background load may refresh the
displayed inventory, but the automatic action intersects live statuses with the captured names and cannot broaden them.

This definition gives “if and only if a previous background check” precise race semantics: an in-flight check is not
previous, a failed check does not become authority, and a later check cannot mutate a confirmation already being built.
Closing/cancelling consumes only the one-shot request, not the app's latest background state. Clicking the badge
continues to open Updates without starting a mutation.

### Comprehensive preview and execution

Build one typed comprehensive preview that can contain an independently actionable SASE leg and a provider leg. Reuse
the current managed/dev SASE planner and the agent-CLI planner with live statuses restricted to the captured names. The
confirmation should have clearly separated “SASE, core & plugins” and “Agent CLIs” sections, show every exact command
that can run, and show manual/skipped candidates with reasons and docs. Important edge states are:

- SASE current + runnable provider candidates: offer a provider-only confirm.
- SASE updates + provider candidates: offer one combined confirm.
- SASE unavailable/blocked + runnable provider candidates: explain the blocked SASE leg and still allow the independent
  provider leg.
- Only manual provider candidates and no SASE work: leave the Updates pane open on Agent CLIs with actionable guidance
  instead of offering a false no-op confirmation.
- Everything revalidated current: show one accurate all-current message.

After confirmation, submit one tracked comprehensive task. Execute safe agent CLI commands sequentially using the
existing executor, then run the selected SASE managed/dev leg; independent failures should be recorded without silently
skipping the remaining independent leg. Prevent overlap with the existing `sase-update` and `agent-cli-update` task
scopes. The aggregate result should distinguish updated, already-current, manual/skipped, and failed providers from the
SASE outcome.

Only a real SASE/core/plugin code change triggers the existing ACE/axe restart, and restart occurs after all provider
commands finish. Extend the one-shot restart receipt/completion rendering enough to preserve provider successes and
failures across that restart; partial success must remain visible. Provider-only changes refresh status in place without
restarting. No-op and fully failed flows do not restart.

### Visual language

Keep the current compact badge states for users with only SASE/plugin updates: purple for routine updates and amber with
`*` when a Rust core rebuild is needed. When agent CLIs are involved, render a two-tone segmented pill rather than
changing only an obscure glyph:

- SASE-only: the existing `↑ N` purple/amber segment.
- Agent-CLI-only: a cyan `CLI ↑ N` segment.
- Mixed: the existing SASE segment joined to a cyan `CLI ↑ N` segment.

The explicit `CLI` label makes the state understandable without memorizing an icon, while domain-specific counts prevent
a misleading aggregate number. The tooltip should spell out both counts, identify manual-only provider updates when
present, explain the core rebuild marker when present, and say that `,U` updates the eligible set. Startup toast rows
should include provider version transitions without attempting repository commit previews for them, and its shortcut
copy should describe the comprehensive behavior.

## Provider-aware background update snapshot

1. Introduce the provider candidate and source-freshness representation in the update-status model, expose separate
   domain counts/flags, and update the atomic snapshot serializer/deserializer with a schema bump and tolerant failure
   behavior.
2. Extend full status computation and the Updates-pane snapshot writer to use the shared agent-CLI collector, keeping
   installed/outdated candidates and preserving last-known-good source state when one source fails.
3. Add candidate-only, no-network revalidation and thread the composite result through the existing startup/periodic
   worker. Preserve the timer/in-flight guards, recompute interval, provider latest cache, and off-thread UI handoff.
4. Cover cache migration, separate counts, all-current versus unknown state, source failure isolation, no-candidate
   zero-work ticks, candidate-only local probes, long-cadence discovery, and pane-written snapshot coherence.

## Snapshot-gated comprehensive update flow

1. Store an immutable provider candidate projection whenever a completed automatic result is applied; capture it at `,U`
   dispatch and carry it through `ConfigCenterModal` into the one-shot Updates-pane auto-update request.
2. Replace the current auto-on-load SASE-only branch with a typed comprehensive planner that intersects captured names
   with live statuses and composes the existing SASE and agent-CLI plans without broadening the candidate set.
3. Present one domain-grouped confirmation, including exact runnable commands, blocked SASE state, and manual/skipped
   provider guidance. Preserve direct `u`, direct `A`, badge-click, cancel, load-error, and all-current behavior.
4. Execute the accepted work as one conflict-safe tracked task, aggregate partial outcomes, refresh the composite
   snapshot immediately, and preserve a complete provider/SASE summary through an eventual code-update restart.
5. Add interaction and task tests for the keypress race boundary, provider-only and mixed plans, no-prior-snapshot
   behavior, stale candidate removal, no newly discovered candidate broadening, manual-only handling, independent
   failures, task conflicts, and restart/no-restart semantics.

## Distinct update surfaces and release validation

1. Implement the cyan agent-CLI badge segment alongside the existing purple and amber states. Update tooltip, startup
   toast, all-current banner, global command/help/footer labels, and configuration descriptions so counts and `,U`
   behavior agree everywhere without renaming the configurable action id.
2. Add focused widget tests and PNG snapshots for CLI-only, mixed routine, and mixed core-rebuild states, retaining the
   existing SASE-only goldens. Add a combined-confirm visual snapshot if the preview layout changes materially.
3. Update `docs/ace.md`, `docs/configuration.md`, and `docs/plugins.md` to explain automatic provider checks, the two
   cadences, snapshot-gated `,U`, manual safety, the segmented badge, and the continued direct roles of `u` and `A`.
4. Run `just install` before verification, then targeted update-status, agent-CLI, automatic-check,
   comprehensive-action, receipt, indicator, and visual tests. Inspect intentional PNG diffs before accepting new
   goldens, run `just test-visual`, and finish with the repository-required `just check`. Include assertions that an
   ordinary ten-minute tick performs no network fetch/full provider inventory and that key dispatch/render paths perform
   no disk I/O or subprocess work.

## Acceptance criteria

- Startup and periodic automatic results can discover and durably represent provider CLI updates, and a transient
  failure in either update domain does not erase valid state from the other.
- Ten-minute ticks remain cache/revalidation work only; full discovery uses the existing longer cadence and npm
  latest-version TTL, and all slow work stays off Textual's event loop and serial message pump.
- `,U` includes provider CLIs exactly when its keypress-time completed background snapshot named candidates, never adds
  names discovered by its own foreground load, and never executes a stale cached command.
- Safe candidates update sequentially; unsafe/manual candidates receive clear instructions; mixed failures are truthful;
  and any ACE restart waits until provider work completes and retains the combined outcome.
- The top-right indicator is immediately distinguishable for CLI-only and mixed updates, remains compact, preserves the
  special core-rebuild signal, and is covered by pixel snapshots.
- Existing `u`, `A`, `sase update`, and `sase agent-cli update` behavior remains compatible, all documentation/help text
  agrees, and `just check` passes.
