---
tier: tale
title: Finish the in-house telemetry release and landing
goal: 'Published SASE installs receive the Rust telemetry bindings, the Telemetry
  pane''s new keys participate in the configured keymap system, and epic sase-6k is
  closed only after clean-install, integration, and post-close Symvision verification
  succeeds.

  '
create_time: 2026-07-17 15:36:48
status: wip
prompt: 202607/prompts/telemetry_release_and_landing.md
---

# Plan: Finish the in-house telemetry release and landing

## Context and verified audit findings

Epic `sase-6k` implemented its seven advertised product slices, and every child bead is closed. The source audit found
the expected SQLite store, Python bindings, in-house accumulators, Rich render toolkit, local-store CLI, lazy
worker-backed Admin Center tab, external monitoring-stack removal, and responsiveness coverage. Targeted verification is
green: the Rust store's six behavioral tests and Python binding round trip pass; 166 Python telemetry/Admin Center tests
pass; and all three Telemetry PNG snapshots match exactly. The central CLI list-default mechanism also correctly maps
bare `sase telemetry` to `sase telemetry list`.

Two requirements are not yet landable:

1. Core commit `646cb0c` added the telemetry store and bindings after `sase-core-rs` 0.5.0 was released. SASE still
   declares `sase-core-rs>=0.5.0,<0.6.0`, and `uv.lock` resolves that older published wheel. Local development passes
   because `just install` builds the linked core checkout, but a normal published install cannot import the new
   `telemetry_record_batch`, `telemetry_query_instant`, `telemetry_query_range`, `telemetry_prune`, and
   `telemetry_store_stats` functions. Release-plz has an open v0.6.0 release PR and a release run queued from the
   telemetry commit; the release must finish before SASE can safely raise its floor.
2. `TelemetryPane` hard-codes `s`, `t`, and `r` in pane-local `BINDINGS`. The epic plan explicitly required every new
   keymap entry to land in `src/sase/default_config.yml`. The effective-config schema and keymap registry currently
   expose only application and mode mappings, so the Telemetry actions, hints, help text, and tests need a scoped
   configurable path that does not create globally active shortcuts.

The integration window from the first main-repository epic commit through the current base was reviewed. Execution
provider overrides already label LLM telemetry with the executing provider; refreshed phase metadata preserves the agent
runner's exit flush; the later retry, effective-agent, phase-bead, explicit-panel, collapsed-panel, and demo work does
not duplicate or conflict with local telemetry. The latest base commit was fast-forwarded and the Telemetry visual tests
were rerun successfully. No further integration changes are indicated outside the two gaps above.

## Publish and verify the telemetry-capable Rust core

Open `sase-core` through `sase repo open` before reading or changing it, and follow its release-plz-owned versioning
rules. Do not manually edit release-owned Cargo versions. Inspect the Release-plz run triggered by core commit `646cb0c`
and the open v0.6.0 release PR. Ensure the generated release commit is based on a master revision containing the
telemetry commit and that both core and PyO3 changelogs describe the telemetry API. If automation has not updated the
release PR, repair or rerun the documented release-plz flow rather than hand-authoring version changes.

Land the generated release through the repository's normal release workflow after its checks pass. Verify the v0.6.0
tag/release and the PyPI wheel matrix complete. Install a published wheel into a temporary clean environment with no
linked-checkout or editable override, then assert that all five telemetry functions import and perform a minimal
record/query/stats/prune round trip. Treat the release as incomplete until the published artifact, not merely a local
build, passes this smoke test.

## Integrate the released core and configurable Telemetry keys

Update SASE's dependency to the telemetry-capable release line (`sase-core-rs>=0.6.0,<0.7.0`, unless the completed
release establishes a different compatible version) and regenerate `uv.lock` from published artifacts. Sweep tests,
fixtures, validation tools, and documentation for assumptions tied to the old range. Add a regression that proves the
declared minimum core distribution exposes all five required telemetry bindings, so a stale but semver-satisfying wheel
cannot recur.

Complete the keymap requirement without making `s`, `t`, or `r` globally active. Extend the effective keymap config,
schema, and registry with a scoped Telemetry-pane action mapping (cycle subsystem, cycle range, refresh), seeded by the
same defaults now hard-coded in `TelemetryPane`. Build the pane's runtime bindings from the effective registry and
derive its on-screen hints/help descriptions from the effective keys. Preserve the pane-local focus semantics and lazy
loading behavior. Cover defaults, user overrides, schema validation, binding dispatch, hint/help rendering, and conflict
behavior. Read the required TUI performance memory before changing this path, and preserve the no-startup-I/O and
off-thread query guarantees.

Run `just install` before repository checks. Verify a clean published-core install as well as the editable linked-core
development path. Run focused Rust/Python telemetry tests, the Telemetry pane and keymap suites, `just test-visual`, and
the full `just check` gate. Re-audit commits added to the main and core bases while these phases were in progress and
integrate any new code that should use the local telemetry store or scoped Telemetry keymap.

## Close and clean up sase-6k

This is the final phase and must run only after the published-core and SASE integration phases are complete. Re-show
`sase-6k` and all seven children, confirm they remain closed and the current base contains every implementation commit,
then close the original epic with `sase bead close sase-6k`.

After closing, read `symvision.md` through `sase memory read` with an audit reason and run `just symvision`. Remove the
expired `sase-6k` epic-symbol allowances from the Justfile and remove or integrate any code that becomes unused once
those allowances expire; do not replace them with another blanket whitelist. Run `just symvision` again, then run the
full `just check` gate because the SASE repository changed.

Finally, open the plans sidecar through `sase repo open plans`, resolve the linked original plan through
`SASE_SDD_PLANS_DIR`, and change only its frontmatter `status` from `wip` to `done`. Confirm `sase bead show sase-6k`
reports closed and the linked plan reports `status: done`. Preserve the audit evidence in the final handoff: published
core version, clean-wheel smoke result, dependency floor, keymap coverage, post-close Symvision result, and full check
result.

## Risks and safeguards

- Release-plz may update or replace its existing release PR while this plan runs. Always re-read live PR/run state and
  verify ancestry from the telemetry commit before merging; do not assume the current PR snapshot is final.
- Editable installs mask missing published bindings. At least one acceptance test must isolate itself from all local
  core overrides and install the exact minimum published version accepted by SASE.
- A globally registered `s`, `t`, or `r` would steal common ACE actions. Keep Telemetry bindings scoped to the focused
  pane and test inactive-tab behavior.
- Closing the bead changes Symvision's active whitelist immediately. Close first, then run Symvision and remove every
  expired allowance before marking the original plan done.
