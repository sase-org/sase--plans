---
tier: tale
title: Validate and repair stored bead plan links
goal: Bead doctor reports broken, ambiguous, and mismatched plan links, an explicitly
  confirmed repair canonicalizes every safely recoverable legacy reference, and the
  live sase bead store is migrated and revalidated.
bead: sase-9z.5
create_time: 2026-07-27 10:02:51
status: done
---

- **PROMPT:** [202607/prompts/design_ref_doctor_repair.md](prompts/design_ref_doctor_repair.md)
- **PARENT:** [202607/durable_plan_refs.md](https://github.com/sase-org/sase--plans/blob/main/202607/durable_plan_refs.md)
- **BEAD:** [sase-9z.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-9z/sase-9z.5.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-9z.5](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-9z.5/README.md)
  - [bbugyi200.athena.sase-9z.5--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-9z.5.md#member-code)
- **COMMITS:**
  - [7ac5b91](https://github.com/sase-org/sase/commit/7ac5b917c08ebe10f847caacdcabf2a2fcc401a6) — feat(beads): repair legacy design references (sase-9z.5)

# Plan: Validate and repair stored bead plan links

## Context and outcome

Bead `sase-9z.5` is the final implementation phase of the durable plan-reference epic. The shared Rust `plans:`
parser/canonicalizer/resolver and Python root-resolution facade already exist, and every plan-reference reader now
routes through them. The remaining gap is operational: `sase bead doctor` still ignores non-empty `design` fields, there
is no safe migration command for legacy paths, and the live plans-sidecar store still contains machine- and
workspace-specific references.

This change will make plain doctor runs honestly report missing, ambiguous, and wrong-owner plan links without mutating
the store. An explicit `-F, --fix-design-refs` run will preview a deterministic repair set, require confirmation, write
each accepted replacement through the normal bead update/event path, and commit the aggregate store mutation through the
existing bead auto-commit path. The migration deliberately includes every repairable legacy reference, even one that
resolves in the current workspace, because those relative paths remain machine- and layout-dependent. Already resolved
canonical `plans:` references remain unchanged.

## Rust-core diagnostics

Extend the bead doctor implementation in the linked `sase-core` checkout without removing its existing compatibility
entry point:

- Add a plan-root-aware doctor path in `crates/sase_core/src/bead/read.rs`, retaining the current no-roots wrapper for
  callers that only want structural bead-store checks.
- For every issue with a non-empty `design`, resolve through `plan::resolve_plan_reference` against the ordered roots.
  Group diagnostics by missing/malformed, ambiguous, and resolved-target owner mismatch. Read `bead_id:` first and the
  legacy `bead:` field second from target frontmatter; only report an ownership mismatch when the plan declares a
  non-empty owner different from the bead.
- Emit category counts with bead-identifying details instead of one warning line per bad record. Preserve all existing
  store/orphan/projection diagnostics and the existing clean result. When roots are explicitly unavailable, skip only
  link validation and emit a clear note rather than failing the health check.
- Thread optional roots through the `bead_doctor` PyO3 binding while keeping the argument optional for older call
  patterns. Add Rust unit/parity and binding tests for exact/drifted success, missing and ambiguous links, legacy and
  canonical forms, owner matches/mismatches, malformed typed references, skipped checks, and unchanged structural
  diagnostics.

Do not edit Rust crate versions: release-plz owns them. Keep the Python package constraint consistent with the epic's
existing core-release strategy rather than naming an unpublished version.

## Repair planning and mutation

Add a focused Python repair planner near the bead administrative code, backed by the shared plan-reference facade:

- Discover plan documents under each ordered plan root, parse only the `bead_id`/legacy `bead` ownership fields, and
  construct deterministic per-root reverse and basename indexes.
- Consider every legacy `design` plus every canonical reference that does not resolve. Select the first unambiguous
  target using the approved order: authoritative owner match (store before local), unique basename in the store, then
  unique basename in the local archive. Stop at the first ambiguous tier. Reject a basename candidate whose declared
  owner names a different bead. Canonicalize the chosen absolute path through the shared Rust canonicalizer.
- Return an immutable preview containing old/new references and an explicit unrepaired reason (`no candidate`,
  `ambiguous`, or `frontmatter names another bead`). Deduplicate store/local copies by normalized path and keep output
  ordering stable by bead order.
- Add an explicit-from-roots canonicalization helper to `sase.sdd.plan_refs` so repair code does not recreate reference
  syntax or root containment behavior.

Update `src/sase/bead/cli_admin.py` so plain doctor resolves store-first roots and passes them through
`BeadProject.doctor`/`sase.core.bead_read_facade`. If SDD root discovery fails, pass an explicit unavailable state so
core doctor reports the skipped link check. For `--fix-design-refs`, print the complete action preview and every
unrepaired record, require an interactive default-no confirmation before mutation, then apply all replacements with
`BeadProject.update(..., design=...)` inside one `bead_store_mutation` scope and one normal auto-commit. Cancellation,
EOF, non-interactive stdin, an empty repair set, or a changed/stale preview must perform no writes.

Register `-F, --fix-design-refs` on the doctor parser with clear help text and keep option ordering/alias conventions.
Update the onboard quick-start text and the source template at `src/sase/xprompts/skills/sase_beads.md`; do not edit a
deployed skill copy by hand.

## Tests and documentation deployment

Add focused Python tests covering root forwarding/degradation, parser help and short alias, read-only doctor behavior,
preview-before-confirmation, cancellation, confirmed event-backed updates, one aggregate auto-commit, all candidate
priority/ambiguity/ownership branches, canonical no-op behavior, and migration of a legacy reference that already
resolves. Update existing facade mocks for the optional roots argument without weakening their structural assertions.

After editing the generated skill source, open the configured `chezmoi` linked repo through `sase repo open`, then run
`sase skill init --force` and `chezmoi apply` as required by the generated-skills workflow. Verify the generated
`sase_beads` skill documents the new doctor flag.

Run focused Rust tests and formatting while iterating, focused Python tests for the repair/CLI/facade paths, then the
repository-mandated sequence `just install` followed by `just check`; also run `just rust-check` because the linked core
changes. Review both repository diffs and statuses before touching live state.

## Live migration and completion

Capture a pre-migration plain `sase bead doctor` report and counts by reference/result class. Run
`sase bead doctor --fix-design-refs` against the live `sase` store, inspect the full preview, and explicitly confirm it.
This intentionally rewrites all repairable legacy references, including currently resolving `sase/repos/plans/...`
values, through ordinary bead update events and the normal bead sync/auto-commit path. Capture the resulting
commit/output, run plain doctor again, and report before/after totals plus every remaining unrepaired reason. Re-run the
relevant focused tests after migration if live data exposes an untested edge case.

Finally close only `sase-9z.5` with `sase bead close sase-9z.5` and verify it is closed while parent epic `sase-9z`
remains open/in progress. Do not create any beads and do not close the parent.

## Risks

- A basename fallback can silently bind the wrong plan. Root-priority, uniqueness, owner-field rejection, full preview,
  and default-no confirmation are the safety boundary.
- Store and local archives commonly contain copies of the same plan. Indexing is per root and store-first so those
  expected copies do not manufacture cross-root ambiguity.
- Holding a bead-store lock across an interactive prompt can block other agents; build and render the preview before
  acquiring the mutation lock, then recompute and compare it under the lock before writing.
- Repairing hundreds of records must remain one auditable CLI operation while still emitting one ordinary update event
  per bead. Tests must assert both properties and that the compatibility projection stays synchronized.
- Core/Python version skew must remain understandable: the binding's roots argument is optional, and released-package
  constraints must not be pointed at a version that release-plz has not produced.
