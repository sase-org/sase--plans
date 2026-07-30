---
tier: tale
title: Protect consumed artifacts and land sase-b9
goal: Recorded artifact consumption protects lifecycle operations, and the verified sase-b9 epic is cleanly closed.
bead: sase-b9
create_time: 2026-07-30 13:01:42
status: done
---

- **PROMPT:** [202607/prompts/integrate_consumption_protection.md](prompts/integrate_consumption_protection.md)
- **PARENT:**
  [202607/artifact_consumption_ledger.md](https://github.com/sase-org/sase--plans/blob/main/202607/artifact_consumption_ledger.md)
- **BEAD:** [sase-b9](https://github.com/sase-org/sase--beads/blob/main/pages/sase-b9/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-b9.land](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-b9.land.md)
- **COMMITS:**
  - [6edbb71](https://github.com/sase-org/sase--plans/commit/6edbb71ef268d7569b8570ab64aac2f68ba41732) — chore(plans):
    mark artifact consumption epic done

# Integrate artifact consumption with lifecycle protection and land `sase-b9`

## Goal

Finish the `sase-b9` epic by integrating its consumption ledger with the artifact-store lifecycle that landed after the
epic began. An artifact with a recorded successful launch-time consumption must be protected anywhere the lifecycle
plans or applies destructive work, even when no ProjectSpec, ChangeSpec, bead, plan, or research document persistently
contains its `file:` reference. Then close `sase-b9`, run the post-close Symvision cleanup, and mark the original epic
plan done.

## Verified starting point

The land audit already established the following on `master`:

- Every child bead is closed with resolution `done`: `sase-b9.1` through `sase-b9.4`.
- Rust core commit `1bd3670` implements tolerant ledger parsing, fragment-free aggregation, the consumed-file-ref set,
  the query-wire-v3 unused filter before row limiting, the renderer binding, and the Python bindings.
- SASE commits `3a0a92d84`, `a4880ce32`, and `0d01edb91` implement best-effort write-side recording, read surfaces, CLI
  filtering, tests, docs, and the generated-skill source.
- Focused verification passed against the locally built linked core: 93 Python tests spanning the ledger, prompt
  expansion, query facade, CLI, end-to-end behavior, lifecycle, and protection scan; Rust ledger, binding, and pre-limit
  unused-filter tests also passed.
- The gap comes from later commits `95f8440` in `sase-core` and `18c01a152` / `be4c19969` in `sase`: lifecycle planning
  accepts an opaque protected-ID set, but `collect_protected_artifact_ids()` currently populates it only by scanning
  persistent text references. Consequently `sase artifact prune` can select a consumed-only artifact. The original
  consumption plan and lifecycle plan both identify recorded consumption as a future hard protection.
- Later lifecycle phases `sase-ba.4`, `sase-ba.5`, and `sase-ba.6` were assigned while this audit ran. They can add
  reclaim, automatic retention, and overlapping documentation before this plan is executed.

## Phase 1: Reconcile the current lifecycle surface

Start from current canonical `master`, not from the commit snapshot above. Inspect all commits since the verified
snapshot and re-read the implementations of:

- `src/sase/core/artifact_file_protection.py`
- `src/sase/core/artifact_consumption_query.py`
- `src/sase/artifact_cli/stats.py`
- `src/sase/artifact_cli/prune.py`
- any newly landed reclaim or automatic-retention caller
- the artifact lifecycle and consumption tests, docs, and `src/sase/xprompts/skills/sase_artifact_file.md`

Confirm whether the concurrent `sase-ba` phases already integrated consumption. Preserve any correct landed solution and
fill only the remaining gaps. Every destructive lifecycle entry point must obtain its protected IDs from one shared
collector; do not add one-off consumption checks only to `prune`.

## Phase 2: Make recorded consumption a fail-safe protection source

Extend the shared protection collector so its returned `ids` are the union of:

1. artifact IDs discovered in persistent ProjectSpec/SDD references, and
2. artifact IDs represented by admissible fragment-free `file:` keys in the consumption ledger.

Use the existing Rust-backed consumption query rather than parsing JSONL independently in Python. Accept only canonical
artifact-file keys and normalize them to the same bare `default:<digest>` / `explicit:<digest>` IDs the retention wire
already consumes. Ignore non-file consumption keys, deduplicate overlap with persistent references, and do not change
the retention planner wire merely to carry source provenance.

Preserve the lifecycle's fail-safe behavior:

- A missing ledger means no consumption has been recorded and is an empty optional source.
- A present, readable ledger contributes its consumed IDs.
- A present but unreadable or unqueryable ledger is reported through protection-source evidence so an apply/enforcement
  pass refuses to remove artifacts. Dry-run/reporting surfaces may still render the gap.

Keep protection reporting truthful. If `ProtectedArtifactIds`, stats JSON, or pretty output currently labels the union
as only "referenced", extend the data model and output additively so durable-reference, consumed, overlap, and total
protection counts are not conflated. Preserve existing stable JSON keys unless an intentional schema-version change is
required, and cover that contract in tests.

Audit every caller after the concurrent lifecycle phases land. At minimum, manual prune, reclaim (if it can change an
artifact ID or remove bytes), the default-policy projection in stats, and opt-in automatic retention must all pass the
union to the same core planner. A consumed-only row must never be selected or applied by any of them.

## Phase 3: Regression coverage and documentation

Add focused coverage that proves:

- a consumed-only file ID enters the shared protected set;
- a non-file ledger key does not;
- a persistently referenced and consumed ID is deduplicated while retaining truthful source evidence;
- a missing ledger is harmless;
- a present unreadable/query-failing ledger is surfaced as unavailable and blocks destructive apply/enforcement;
- manual prune dry-run and apply both exclude a consumed-only row;
- any newly landed reclaim and automatic-retention paths also exclude it;
- stats JSON and pretty output describe the expanded protection contract accurately.

Update the artifact documentation to replace the stale statement that consumption-based pruning protection is not
implemented. Reconcile with the concurrently landing lifecycle documentation rather than duplicating it. Update the
`sase_artifact_file` source template if its lifecycle guidance needs the new guarantee. Follow the `generated_skills.md`
workflow: preview generated output while iterating, commit and land the source change first, and deploy generated skills
only from the clean canonical commit.

Run `just install` before verification. Run the focused protection/consumption/lifecycle tests while iterating, then run
`just check`. Also run the linked `sase-core` checks only if this integration genuinely requires a core change; the
existing summary binding and opaque protected-ID planner should normally make the SASE-side integration sufficient.

## Phase 4: Close and clean up the original epic

This is the final phase and must be done only after the integration above is committed, landed on canonical `master`,
and verified.

1. Re-run `sase bead show sase-b9` and every child bead, and inspect the final integrated commits.
2. Close the epic without force:

   `sase bead close sase-b9 --note "<verified child commits, post-start lifecycle integration, and checks run>"`

   If close is rejected, address or deliberately reopen the named unfinished work. Never force a `done` close.

3. After the close succeeds, run `just symvision`. Closing expires any `sase-b9(...)` epic-symbol allowances. If
   Symvision reports stale entries or unused public code, read the `symvision.md` long-term memory, remove the expired
   allowances, make genuinely internal symbols private or remove unused code as directed, and rerun Symvision until
   clean. Run `just check` after any source change.
4. Finally, open the plans sidecar through `sase repo open plans`, change only the original plan's frontmatter status in
   `202607/artifact_consumption_ledger.md` from `wip` to `done`, validate the plan/link state, and leave the epic and
   plan consistently landed.
