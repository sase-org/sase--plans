---
tier: epic
title: Finish the weighted-capacity core pin, lifecycle acceptance, and released floors
goal:
  The pinned Rust core actually contains the lineage wire this repo consumes, the
  weight-2 monitor/gate lifecycle and one-snapshot runtime/CLI/TUI parity acceptance
  exist, the capacity-strip visual corpus is honest, and published floors are proven.
parent_bead: sase-z4.6.5
phases:
  - id: core-pin
    title: Ratchet the core revision pin to the commit that carries the lineage wire
    size: medium
    depends_on: []
    description:
      "core-pin: advance sase-core-revision.txt to a core commit containing the
      agent-scan runner_claim_owner_key wire field and index schema 27, then prove a
      clean provision from the pin builds and passes."
  - id: monitor-gate-acceptance
    title: Add the missing weight-2 monitor and gate lifecycle acceptance
    size: medium
    depends_on:
      - core-pin
    description:
      "monitor-gate-acceptance: exercise a weight-2 land-style family through real
      monitor creation, supervision, and --next handoff plus the gate routes, instead of
      hand-authoring a monitor record."
  - id: snapshot-parity
    title: Compare runtime, CLI, and TUI capacity from one captured snapshot
    size: medium
    depends_on:
      - core-pin
    description:
      "snapshot-parity: from one captured source snapshot, compare runtime admission,
      sase agent list -j, the ACE capacity header, queue ranks/blockers/details, local
      and remote badges, unknown usage, and filtering/folding."
  - id: capacity-visual-corpus
    title: Regenerate the capacity-strip visual corpus deliberately
    size: medium
    depends_on:
      - core-pin
      - snapshot-parity
    description:
      "capacity-visual-corpus: settle the capacity-strip text and regenerate the stale
      agents-pane PNG goldens by inspection, retiring the status-strip diff class that
      has kept just test-visual red since the prefix landed."
  - id: published-floors
    title: Prove actual released floors and retire the rollout flag
    size: medium
    depends_on:
      - core-pin
      - monitor-gate-acceptance
      - snapshot-parity
      - capacity-visual-corpus
    description:
      "published-floors: establish releases containing the repaired core and host work,
      verify the exact published wheels in a clean environment, ratchet floors and pins,
      and close the weighted_queue_capacity flag bead only after the release proof
      succeeds."
proposed_by: bbugyi200.athena.sase-z4.6.5.land
create_time: 2026-09-10 17:42:10
status: wip
---

- **PROMPT:**
  [prompts/202609/weighted_capacity_remaining_acceptance.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/weighted_capacity_remaining_acceptance.md)
- **PARENT:**
  [202609/weighted_capacity_final_acceptance.md](https://github.com/sase-org/sase--plans/blob/main/202609/weighted_capacity_final_acceptance.md)

# Finish the weighted-capacity core pin, lifecycle acceptance, and released floors

This child epic contains only the work the `sase-z4.6.5` land audit found still missing
or newly broken. Its `parent_bead` link is the handoff back to that interrupted land
agent. Do not close `sase-z4.6.5`, `sase-z4.6`, or `sase-z4`, do not run their
post-close Symvision passes, and do not mark their linked plan files done from a phase
here.

Read the governing contract and the parent epic's own text first:

```sh
sase artifact read plan:202609/weighted_capacity_final_acceptance.md "Implement only the acceptance work the sase-z4.6.5 land audit found missing"
sase artifact read plan:202609/weighted_capacity_landing_repairs.md "Retain the combined weighted-capacity contract this epic must still satisfy"
sase bead show sase-z4.6.5
sase bead show sase-z5
```

Open `sase-core` with `/sase_repo` before reading or changing it, use only the path that
skill prints, and read its `AGENTS.md`. Recheck drift before each phase lands.

## What the audit established

Verified complete, do not redo:

- `admission-authority` (SASE `3260f6a42`) really did make Rust authoritative.
  `_serial_family_owner_key` and `_active_serial_claim` are gone from the tree, the
  candidate travels as the Rust request's own `candidate` field,
  `_require_candidate_decision` fails closed, `_publish_claim_ownership` persists
  `queue_weight` and `runner_claim_owner_key` under the runner-slot lock before
  `claim()` exposes work, and `capacity_only=True` is live in
  `_RUNNER_SLOT_SCAN_OPTIONS`.
- `integrated-acceptance` items 1 and 3 (SASE `788c63e28`): fractional fill with live
  cap reload, explicit zero runner/priority through real parking, and the installed
  `research_swarm` quarter-weight expansion all exist in
  `tests/fakey/test_runner_slots_e2e.py`.
- `published-floors` item 1 (SASE `0444bac58`): `tools/validate_sase_core_rs` now
  behaviorally validates the weighted-capacity surface, and the phase proved a clean
  venv holding the real PyPI `sase-core-rs==0.33.0` fails all three new contract checks.

Everything below is what is left.

## core-pin

`sase-core-revision.txt` is stale in a way that breaks a clean provision, and this epic
caused it. `admission-authority` landed SASE code that consumes the agent-scan
`runner_claim_owner_key` wire field and asserts
`AGENT_ARTIFACT_INDEX_SCHEMA_VERSION == 27`, but it never advanced the pin. The pinned
revision predates both: at that commit `runner_claim_owner_key` exists only in
`crates/sase_core/src/runner_capacity.rs`, and
`crates/sase_core/src/agent_scan/index.rs` still declares schema `26`. The core commit
that adds the wire field, the scanner lineage lookup, and schema `27` landed after the
pin was last moved by an unrelated commit. The phase agent worked around this locally by
rebuilding from its checkout and said so in its note; nothing durable followed.

1. Advance `sase-core-revision.txt` through `just ratchet-core-revision` — the supported
   tooling — not by hand. `--report-only` already agrees the pin should move. Do not
   edit release-plz-owned crate versions.
2. Before accepting whatever revision the tool proposes, confirm it actually contains
   the agent-scan `runner_claim_owner_key` wire field, the scanner lineage lookup, and
   index schema `27`. If the tool proposes core HEAD, check that the extra commits it
   sweeps in are ones this repo's current tree is ready for; if any is not, pin the
   first revision that carries the lineage wire instead and say why in the phase note.
3. Prove the pin from a genuinely cold build, not an already-warm venv: the audit found
   the repo `.venv` resolving `sase_core_rs` to an editable path with no compiled
   extension present, so every `just` gate on this host failed at import with
   `sase_core_rs is not importable in this environment`. A clean `just install` must
   produce a working extension and `AGENT_ARTIFACT_INDEX_SCHEMA_VERSION` must read `27`
   from the built binding, not from the Python constant alone.
4. Run `just check`. Two gates are red for reasons this epic does not own and must not
   be "fixed" here: `lint (feature flags)` fails on live flag bead `sase-z0` having no
   registry definition for key `link_events` (recorded on active epic `sase-yy.8`), and
   `tests/test_pooled_alias_single_consumption.py` plus two
   `tests/test_workflow_executor.py::TestShouldHitl` nodes fail deterministically on
   clean master from a model-resolution defect (task `sase-q4`, reopened with fresh
   evidence). Name any other failure in the phase note and diagnose it rather than
   assuming it belongs to that set.

## monitor-gate-acceptance

The parent plan's second `integrated-acceptance` item was not delivered. The only
monitor case in `tests/fakey/test_runner_slots_e2e.py` is the pre-existing
`test_fakey_monitor_holds_capacity_across_handoff_and_followup`, which the parent plan
had already named as inadequate — its own comment still says it simulates the monitor's
shape with a bare live pid "instead of driving the full monitor subsystem", and it runs
at default weight 1.0, not weight 2.

1. Drive a weight-2 land-style family through actual monitor creation, supervision, and
   `--next` handoff. Verify the starter, the monitor, and a serial successor retain one
   owner key and exactly one 2.0 claim across the whole handoff.
2. Verify an independently weighted parallel member and its own serial successor keep
   their own lineage and their own claim, and never inherit the land family's owner.
3. Cover the gate routes: a pending human gate holds zero capacity; approved automatic
   and detached gate routes transfer or reacquire before the gated command executes.
4. Cover the failure routes: cancellation, failed startup, timeout, and crash must not
   release another owner's claim.

Assert against the persisted `runner_claim_owner_key` and effective weight, since that
is now the durable lineage fact. Do not replace lifecycle coverage with parser-only
tests.

## snapshot-parity

The parent plan's fourth `integrated-acceptance` item was not delivered either; the
audit found no test that compares these surfaces against one shared snapshot.

1. Capture one source snapshot and, from it alone, compare runtime admission,
   `sase agent list -j`, the ACE capacity header, queue ranks, blockers, and details,
   local and remote non-default badges, unknown usage, and filtering/folding behavior.
2. Preserve remote ownership: remote weights are presentation metadata and must never
   charge the local snapshot. Assert that explicitly rather than leaving it implied.

## capacity-visual-corpus

`just test-visual` has been red on clean master since the weighted capacity strip
landed, and the parent epic carries a note asking whoever settles the final
capacity-strip text to regenerate the corpus deliberately in the same change rather than
leaving a bulk `--sase-update-visual-snapshots` for an unrelated agent.

The audit corroborated the cause. The commit that added `_append_capacity_prefix` to
`src/sase/ace/tui/widgets/agent_info_panel.py` changed 28 files and zero PNGs. A later
capacity commit refreshed only three agents-pane goldens and added two, against a corpus
of roughly 134 `agents*` PNGs. The reported signature is a fixed 12-file agents-pane
subset failing 44 to 12, with every diff confined to the status-strip row where goldens
still expect `[0/10 running` and the render now emits `0.0/10.0 [0 running`.

1. Settle the capacity-strip text first, informed by `snapshot-parity`. Regenerating
   against text that is about to change again wastes the corpus.
2. Establish the real current failure set on a clean tree at this epic's HEAD rather
   than trusting the reported count — several capacity and usage-badge commits landed
   after that observation.
3. Regenerate deliberately and inspect the changed PNGs. Confirm each diff is the
   expected status-strip change and nothing else; a golden whose diff is not explained
   by the capacity strip belongs to a different defect and must be named in the phase
   note, not swept into the rebaseline.
4. Related but out of scope: task `sase-x5` tracks a separate standing backlog of stale
   goldens with small scattered pixel ratios. Do not absorb it. If regenerating this
   corpus resolves goldens `sase-x5` also names, say so in the phase note so its owner
   can narrow that bead.

## published-floors

Items 2 through 5 of the parent plan's `published-floors` phase are unfinished. The
phase bead recorded that clearly and asked to stay open; it was then auto-closed by the
commit finalizer, which is why the parent epic looked complete when it was not.

The blocker is real and externally owned. `sase-core-rs` has had no release since
`v0.33.0`, and PyPI still serves only `0.33.0`. Every release-plz run on core master
since the weighted-capacity repairs landed has failed identically: `cargo package`
cannot verify `crates/sase_core_py/Cargo.toml` because its workspace-inherited
`sase_gateway` dependency resolves without a version requirement, even though the
workspace manifest now declares one. Bead `sase-xe.16.11.7.14.3` owns that fix, is still
in progress, and its first attempt did not clear the failure; no release-plz PR is open.
Do not duplicate that work here.

1. Recheck the blocker before assuming it: `sase bead show sase-xe.16.11.7.14.3`, plus
   release-plz run and PR state on `sase-core`, plus whether PyPI now serves a version
   above `0.33.0`.
2. If it is still blocked, keep this phase open with concrete tag and workflow evidence.
   Do not weaken the checks, do not treat a source checkout or a revision pin as a
   release proof, and do not close `sase-z5`.
3. If it has cleared, establish the actual releases through each repository's own
   automation, verify ancestry by tag, and verify the package index serves the exact
   versions. Then ratchet this repo's core dependency floor and source revision to the
   first containing release, replacing the provisional `sase-core-rs>=0.33.0,<0.34.0`
   assertion with the real first compatible version and keeping upper bounds coherent.
4. Run a genuinely clean, wheel-only minimum-version smoke: no editable checkout, no
   source override, no `maturin develop`, no local package path. Exercise the repaired
   candidate decision, the scan projection, the weighted fleet summary, and the research
   swarm segments. Keep a negative check that the last incompatible floor still fails,
   and make the ratchet depend on the positive smoke.
5. Only after the release proof succeeds, re-read `sase-z5`, confirm the
   `weighted_queue_capacity` registry entry and Off branch are still absent after all
   drift, and close that flag bead normally with a note naming the completed lifecycle,
   presentation, package, and rollout proof. This resolves `sase-z4.6.4`'s sole proposed
   follow-up; do not create a duplicate task bead.

Run the floor probe against the installed published wheel, every changed repository's
checks, and the combined `just check-full` only through `/sase_monitor` with `TESTING`
and `TESTED`. Record exact tags, package versions, smoke commands, and outcomes in the
phase note so this epic's land agent can revalidate them before handing control back.
