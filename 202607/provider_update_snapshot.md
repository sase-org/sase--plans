---
tier: tale
title: Provider-aware background update snapshot
goal: 'ACE''s existing automatic update worker durably discovers supported agent CLI
  update candidates, revalidates only cached candidates between full checks, and preserves
  truthful per-source freshness without adding UI-thread, per-tick network, or zero-candidate
  subprocess work.

  '
create_time: 2026-07-20 10:26:48
status: wip
prompt: 202607/prompts/provider_update_snapshot.md
---

# Plan: Provider-aware background update snapshot

## Context and boundaries

ACE already has a schema-versioned update snapshot, a post-paint worker, a ten-minute revalidation tick, and a longer
full-recompute cadence for SASE core and plugins. The Updates pane separately obtains the same provider CLI inventory
used by `sase agent-cli`. This change will join those paths so one durable snapshot represents both domains while
preserving the existing timer, in-flight guard, worker-to-UI handoff, provider metadata, six-hour npm latest-version
cache, and safe cache-miss behavior.

This phase stops at background discovery and snapshot coherence. The later parent-epic phases own keypress-time
candidate capture, comprehensive execution, the segmented indicator, final toast/help language, and release
documentation. No provider-specific update commands, new cadence controls, or Rust-core behavior will be introduced.

## Composite status and source truth

- Add a small immutable provider-candidate record containing stable provider identity plus the installed/latest version
  evidence needed for display and later live revalidation. Keep executable commands and other mutation authority out of
  the durable snapshot.
- Represent successful-check freshness and latest failure information for the independently fallible core, plugin, and
  agent-CLI sources. Extend the shared status with derived SASE/component, agent-CLI, aggregate, and core-rebuild
  counts/flags so an empty successful source is distinguishable from an unknown or failed source.
- Bump the snapshot schema and round-trip every new typed field through the existing atomic writer. Treat prior-schema,
  malformed, partially invalid, and future snapshots as cache misses instead of partially trusting them.
- Merge a newly computed snapshot with the previous durable snapshot by source: successful sources replace their prior
  rows (including with a proven empty set), while failed sources retain their last-known-good rows/freshness and expose
  the current failure. This prevents one source failure from suppressing valid new results or erasing another source's
  known candidates.

## Discovery and drop-only revalidation

- Extend full update-status computation with the shared `collect_agent_cli_statuses()` service. Filter its result to
  installed, outdated providers and project those rows into durable candidates; continue to isolate core, plugin,
  registry, local-probe, and latest-version failures.
- Add a source-independent local detection path that can restrict provider metadata and probes to a supplied set of
  stable provider names. Candidate revalidation will compare live installed versions against the cached latest evidence
  without consulting npm/latest caches or making network requests.
- Make candidate revalidation conservative and drop-only: remove providers proven absent or current, retain providers
  whose local version probe fails transiently, and never discover a provider absent from the cached candidate set.
  Return immediately before provider registry inspection, disk reads, or subprocess calls when that set is empty.
- Thread the composite revalidation and per-source merge through `get_cached_update_status()` so fresh-cache reads,
  stale periodic reads, recompute failures, and explicit indicator refreshes share the same rules.

## Existing ACE worker and Updates-pane integration

- Carry the composite status through the existing startup/periodic worker and UI-thread completion callback. Keep the
  timer callback synchronous and thin, keep all config/disk/JSON/registry/subprocess/network work on the worker, and
  preserve the current overlap guard and long-cadence recompute decision.
- Feed the current aggregate/core state through the existing indicator API so the model is immediately coherent without
  taking on the later segmented visual redesign. Keep repository commit-section generation scoped to SASE and plugin
  components; provider toast presentation remains a later phase.
- When the Updates pane completes an online inventory load, build and persist the same composite shape from the
  core/plugin inventory and the provider statuses it already collected. A provider collection failure must merge with
  prior provider state instead of overwriting it with an apparently successful empty set; offline pane loads continue
  not to replace the shared snapshot.

## Verification

- Expand update-model/cache tests for schema migration and malformed data, provider round trips, independent source
  freshness/errors, separate and aggregate counts, successful-empty versus unknown state, and last-known-good merge
  behavior.
- Add provider discovery/revalidation tests proving full recomputes use the shared collector, installed/outdated
  filtering is correct, zero candidates do zero provider work, only named candidates are locally probed, transient probe
  errors retain candidates, and proven current/removed candidates are dropped without latest-version or network access.
- Extend automatic-check tests to prove ordinary ticks remain revalidation-only, long-cadence checks alone grow the
  candidate set, UI application receives the composite aggregate, and timer/key/render paths do no slow work.
- Extend Updates-pane loader tests for composite snapshot writes, reuse of the already-collected provider inventory,
  provider-error preservation, source isolation, and offline non-clobber behavior.
- Run `just install` before executing focused update-status, agent-CLI, automatic-check, and pane-loading tests. Finish
  with the repository-required `just check` after all implementation changes.

## Completion criteria

- A completed full automatic check durably reports SASE/core/plugin and installed outdated provider CLIs with truthful
  per-source freshness.
- Failure of any one source leaves valid results from all other sources visible and retains last-known-good data for the
  failed source without presenting it as newly checked.
- Ordinary periodic ticks cannot grow the provider set, perform a latest-version lookup, or probe anything when no
  provider candidate exists; candidate probes are bounded and remain off the Textual event loop and serial message pump.
- The Updates pane and automatic worker write/read the same schema, and all targeted tests plus `just check` pass.
