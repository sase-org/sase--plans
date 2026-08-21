---
tier: tale
title: Graduate artifact links and retire the beta flag
goal:
  Make the first-class artifact link graph unconditional and healthy end to end, migrate
  the last live v1 Referenced By truth, remove the artifact_links flag and deprecated
  Off branch, fix project-key resolution so link writes cannot partially land, and close
  the dedicated flag bead only after focused, live-sidecar, and whole-repository
  verification.
size: medium
proposed_by: bbugyi200.athena.099
create_time: 2026-08-21 09:18:27
status: done
---

# Plan: Graduate artifact links and retire the beta flag

## Outcome

The behavior introduced by `sase-r8` becomes the only artifact-link implementation:

- `sase artifact link add|list|rm|migrate-notes`, audited `sase artifact read` edges,
  prompt-ref `cites` rows, Markdown `Links` / `Referenced By` projections, artifact
  doctor checks, bead-event truth, and ACE relationship panes work without
  `-f artifact_links` or configuration.
- The deprecated flag-off path and v1-only Referenced By implementation are gone after
  every live sidecar index is proven to be schema v2.
- Current workspaces resolve the canonical ProjectSpec key before any artifact-link
  write. A malformed or unresolved key fails before a sidecar row or lock file can be
  created.
- The `artifact_links` registry/schema entry and all beta-era help, docs, tests, and
  agent-skill wording are removed.
- Flag task `sase-rc` closes as `done` only after the current tree and live SASE
  sidecars pass the acceptance checks below.

## Why this is a tale

This is substantial but bounded graduation work. The storage, CLI, presentation,
documentation, live-data migration, and bead closure must be validated as one coherent
transaction; splitting them between independent phase agents would create an unsafe
interval where v1 data outlives its reader or callers outlive the flag definition. One
follow-up coding agent can implement it directly, so the appropriate tier is `tale` and
the appropriate size is `medium`.

## Verified baseline

- The implementation epic `sase-r8` and all eleven descendants are closed. Its landing
  audit records passing focused artifact-link coverage and the published
  `sase-core-rs>=0.29.5` bead-link bindings.
- The dedicated removal bead is `sase-rc`, a small open task of type `flag`. Its removal
  contract says to delete the Off branch and make the On branch unconditional. The
  qualitative gate requires a clean doctor run and no v1-only caller.
- `artifact_links` remains a beta registry member with default Off, although this host
  currently enables it through `SASE_FEATURE_FLAGS`.
- The plans sidecar currently contains one schema-v2 link index and one live schema-v1
  index: `links/202608/monitor_followup_wait_release.md.json`. No aggregate exists at
  the canonical `~/.sase/projects/gh_sase-org__sase/artifact-links.json` path yet.
- A real audited read reproduced a blocking end-to-end bug. The workspace marker carries
  the provider slug `sase-org/sase`, while `_project_key_for_cwd()` passes it directly
  to `artifact_link_aggregate_path()`, which accepts only a single canonical project-key
  component. `sase artifact read` wrote the sidecar v2 row and lock first, then failed
  with `invalid project key for artifact-links index: 'sase-org/sase'`. The two files
  created by that probe were removed immediately, restoring the plans sidecar to a clean
  tree. `sase artifact doctor` currently hides the same defect as `skipped (no store)`
  because link-health resolution catches the exception.
- The deprecated branch is concrete, not just a registry toggle:
  `referenced_by_refresh.py` dispatches to `_referenced_by_refresh_legacy.py`, writers
  raise `ArtifactLinksDisabledError`, doctor and ACE return disabled snapshots, bead
  pages strip link tables, and v1 readers/migrators remain reachable.
- `docs/artifact_links.md`, `docs/artifact_references.md`, `docs/cli.md`, parser help,
  and the installed `sase_artifact_file` skill still tell agents to enable the beta
  flag. The skill's canonical managed copies are in the configured `chezmoi` repo; do
  not edit only the installed Codex copy.

## Constraints and sequencing

1. Run `just install` before tests or workspace-local CLI probes. Use the workspace's
   installed executable for code-under-test probes; use bare `sase` for SASE lifecycle
   commands required by skills.
2. Open every non-primary repo with `/sase_repo` before reading or changing it. This
   includes the plans, research, beads, agents, and `chezmoi` repositories. Use only the
   paths printed by `sase repo open`; do not encode an ephemeral workspace directory in
   source, docs, commits, or handoffs.
3. Preserve unrelated dirty state. Before every sidecar mutation, record its status and
   exact candidate paths. Never use a broad cleanup command.
4. Migrate and verify live v1 data while the migration reader still exists. Delete v1
   compatibility only after all configured artifact sidecars report zero schema-v1
   indexes. The migration must preserve the original citation's agent, target, uses,
   timestamp/published evidence, and rendered `Referenced By` meaning.
5. Do not edit SASE memory files or generated provider instruction shims. The artifact
   relation registry remains unchanged, so `sase memory init` is not part of this work.
6. The Rust artifact-link row/relation and bead-event contracts are already published.
   Keep this graduation in Python, docs, tests, and sidecar data unless a focused test
   proves a genuine Rust defect; if one is found, open `sase-core` through `/sase_repo`
   and honor the Rust backend boundary.
7. A live end-to-end probe must use dedicated temporary artifacts/beads or a precisely
   reversible test fixture. Do not leave synthetic links, MIGRATED notes, companions, or
   audit rows in durable sidecars merely to prove the CLI.

## Implementation

### 1. Fix canonical project resolution before exercising writes

- Replace `_project_key_for_cwd()`'s direct trust in `CheckoutMarker.project_key` with
  the existing project alias/inventory resolution used elsewhere in SASE. Resolve marker
  `project_name`, canonical ProjectSpec key, aliases, and provider slugs such as
  `sase-org/sase` to the canonical key `gh_sase-org__sase`; do not invent a local
  slash-to-underscore transform.
- Validate the resolved key when constructing `ArtifactLinkStore`, before any method can
  enter `_upsert_sidecar`, bead mutation, or aggregate-only mutation. A bad key must
  fail before creating either `links/*.json` or `*.lock`.
- Use the same resolver in `sase doctor`'s `project.artifact_links_aggregate` check so
  the CLI and diagnostic cannot disagree about the current project.
- Add regression coverage using a realistic checkout marker whose legacy/provider
  `project_key` is `sase-org/sase` and whose registered project key is
  `gh_sase-org__sase`. Assert `resolve_artifact_link_store()` selects the canonical
  aggregate path and an audited read/link write succeeds. Add a failure-path assertion
  that an invalid/unresolvable project creates no sidecar JSON or lock file.
- Stop swallowing this class of configuration error as a healthy doctor skip. A repo
  that has an SDD store but cannot obtain a valid canonical project key must report an
  actionable error; a checkout with genuinely no configured artifact store may still
  degrade explicitly.

### 2. Prove the removal gate and migrate live v1 truth

- With the resolver fix in place and v1 support still present, inspect every configured
  document sidecar's `links/**/*.json`, including plans and research. Record counts by
  schema version and run artifact-link doctor without `--fix` first.
- Convert every schema-v1 index to the v2 row shape using the existing migration logic.
  For the known plans row, verify the resulting canonical edge is an agent `cites` edge
  to `plan:202608/monitor_followup_wait_release.md`, retains `uses: 1`, and renders the
  same citation semantics. Make the data change through the supported SDD/sidecar
  workflow and keep the plans sidecar cleanly attributable; never hand-edit generated
  Markdown tables.
- Rebuild the canonical project aggregate from v2 sidecar truth plus bead events. Run
  doctor again and inspect the rendered blocks, missing companion report, dangling rows,
  stale table report, and "index not in HEAD" report. Repair genuine findings found by
  these checks before continuing; do not weaken doctor to make the gate green.
- Re-scan all opened sidecars and assert there are zero schema-v1 index documents.
  Verify the migration and aggregate rebuild are idempotent on a second pass.

### 3. Make the v2 graph unconditional and delete the Off branch

Remove flag checks rather than replacing them with constants:

- In the store facade/support and implementation, delete `ArtifactLinksDisabledError`,
  `artifact_links_enabled()`, the disabled message, `require_artifact_links_enabled()`,
  and the guards on `upsert_row()` and `remove_rows()`.
- In bead mutation paths, delete both the public adapter guard and the duplicate guard
  in `core/bead_mutation_facade.py`. Preserve the write sandbox, Rust validation,
  reserved-relation errors, idempotency, and append-only bead events.
- Make RELATED-note application unconditional while keeping dry-run/apply semantics and
  its manual worklist behavior unchanged.
- Make `refresh_referenced_by()` always call the v2 artifact-link refresh under the
  artifact-link lock/cause. Delete `_referenced_by_refresh_legacy.py` and its flag-only
  dispatch.
- Make `sase artifact read` record its `read` graph edge whenever it is inside an agent
  run with an identity. Outside an agent run, keep the audited read and consumption
  event but make the informational message accurately say that no agent identity/run
  exists; it must not mention enabling a removed flag.
- Always load and show link neighborhoods in `sase artifact show`, always run link
  health in `sase artifact doctor`, always render bead Links/Referenced By projections,
  and always load the artifact-link snapshot used by ACE. Remove redundant `enabled`
  fields/disabled sentinels where they have no remaining semantic value; retain a
  distinct missing-store/error state where callers genuinely need one.
- Keep `link list` readable and all writers idempotent. Removing the flag must not
  accidentally turn a former disabled path into a second implementation.

### 4. Retire v1-only code after the data gate is green

- Remove `artifact_link_migrate.py`, schema-v1 auto-migration on ordinary reads,
  schema-v1 iteration in aggregate rebuilds, the v1 `--fix` migration result fields, and
  fixtures/tests whose only purpose was preserving the deprecated Off branch.
- Refactor `referenced_by_index.py` rather than deleting useful neutral primitives
  blindly. Keep or rename only path construction, managed-block detection, and other
  helpers still required by v2 projection/doctor code; delete schema-v1 read/write and
  normalization APIs with no remaining callers.
- A schema-v1 JSON encountered after graduation must produce an explicit unsupported
  schema/upgrade diagnostic, not silently disappear from the aggregate and not be
  rewritten by a hidden compatibility path.
- Use `rg` plus import/static checks to prove there are no remaining references to the
  deleted flag helpers, disabled error, legacy refresh module, v1 migration functions,
  or v1-only fixtures.

### 5. Remove the flag contract and update every user/agent surface

- Delete `FeatureFlag.artifact_links` and its definition/bead pointer from
  `src/sase/feature_flags/registry.py`; regenerate or update the shipped config schema
  through the repository's feature-flag tooling so its generated-integrity test stays
  authoritative.
- Update feature-flag consumer/integrity tests so `artifact_links` is no longer a
  registered key. Add a focused CLI assertion that `sase flag list` omits it and a
  retired `-f artifact_links` override cannot silently control behavior.
- Remove beta/enable/disabled wording from artifact CLI parser help,
  `docs/artifact_links.md`, `docs/artifact_references.md`, and `docs/cli.md`. Document
  the unconditional behavior, the outside-agent `read` exception, v2 truth, and the
  unsupported-v1 diagnostic.
- Update the canonical `sase_artifact_file` skill copies in the opened `chezmoi` source
  so every runtime says writes are unconditional and reads create edges inside agent
  runs. Keep all provider copies byte-equivalent where they are intended to be the same,
  and apply/deploy only through the canonical chezmoi workflow after the SASE source
  change is committed and landed; never patch only `~/.codex/skills`.
- Finish with `rg artifact_links` across the primary tree and relevant skill source. The
  only acceptable residue is historical artifact/plan/bead content that must not be
  rewritten; no live source, config schema, help, docs, test name, or skill instruction
  may advertise the retired flag.

### 6. Replace flag-state tests with unconditional behavior and regression coverage

Keep the existing positive contract tests, remove flag wrappers, and strengthen them so
the graduated path is tested without an override:

- Store: directed and undirected dedup, aliases, reserved relations, sidecar truth,
  aggregate-only rows, bead endpoints, removals, protection, and atomic/early project
  key validation.
- Beads and migration: `LinkAdded`/`LinkRemoved`, bead-to-bead and bead-to-document
  rows, page projections, RELATED-note dry run/apply, original-note preservation, and
  idempotent `MIGRATED:` behavior.
- Read/show: Markdown/frontmatter/managed-block stripping, binary and stitch cards,
  audit plus consumption ordering, `read` edge creation inside an agent run, no graph
  edge outside one, stable JSON `links`, and surfaced write failures.
- Refresh/projection: prompt-ref `cites`, plan-header placement, automatic versus
  curated table routing, binary companions/collision protection, commits with the system
  projection cause, and no content outside managed blocks changing.
- Doctor: canonical project resolution, aggregate rebuild comparison, dangling rows,
  stale Links tables, missing companions, indexes absent from HEAD, unsupported v1, and
  repair idempotency. There must be no "flag disabled" healthy skip test.
- ACE: links/linked-by relationships load from the aggregate without a flag snapshot,
  keep undirected dedup, and do no synchronous disk I/O in a rendering/event-loop path.
- CLI/help/docs contracts: writers work without `-f`, former flag-off failure tests are
  deleted or rewritten as success tests, and help no longer mentions a beta.

## End-to-end verification

After focused unit/integration tests pass, execute one isolated CLI journey against a
temporary project with plans, beads, and a binary artifact:

1. Add a plan-to-plan `implements` link twice and prove `added` then `unchanged` with a
   single stored row.
2. Add and remove a bead-to-plan relation; verify bead events, per-artifact sidecar
   truth, aggregate truth, `sase bead show`, generated bead page,
   `artifact link list -d both`, and `artifact show -j` all agree in both directions.
3. Run `sase artifact read` inside a synthetic agent environment and verify the audit
   row, consumption row, agent `read` edge, aggregate, and Referenced By projection; run
   it outside an agent and verify audit/consumption still occur without a graph edge or
   flag advice.
4. Exercise prompt-ref refresh into a `cites` row, a curated Links row, and a binary
   companion. Confirm managed blocks are placed correctly, repeated refresh is a no-op,
   and the document's semantic body is unchanged.
5. Run `migrate-notes` dry-run and apply against a disposable RELATED note; verify the
   worklist, preserved original note, typed edge, and idempotency.
6. Run `sase artifact doctor`, its repair mode, and doctor again. The final run must be
   healthy and non-skipped.

Then verify the real SASE project read-only after its deliberate v1 migration:

- all configured artifact sidecars are clean and contain only schema-v2 link indexes;
- the canonical `artifact-links.json` exists under the registered ProjectSpec key;
- live `artifact link list`, `artifact show`, and `artifact doctor` resolve the current
  project instead of returning `invalid project key` or `skipped (no store)`;
- no dangling/stale/missing/not-in-HEAD doctor finding is ignored;
- `sase flag list` no longer contains `artifact_links`.

Run the focused artifact-link suites covering the modules above, then `just check`.
Because this graduation deletes a cross-cutting compatibility path and the request asks
for thorough end-to-end assurance, run `just check-full` only through `/sase_monitor`
with `TESTING` / `TESTED` statuses and a continuation that inspects the result. If
scoped selection escalates or any failure is unrelated/flaky, follow the repository's
bead rules rather than suppressing it. No visual snapshot run is required unless an
actual ACE rendering change is made.

## Completion and bead closure

Close `sase-rc` only after all of the following are true:

- the feature is unconditional and the Off branch is absent;
- the canonical-key and no-partial-write regressions pass;
- every opened sidecar is schema v2 and artifact doctor is healthy/non-skipped;
- focused tests, the isolated CLI journey, `just check`, and monitored `just check-full`
  pass;
- source/config/docs/help/skills have no live `artifact_links` flag wording;
- primary and sidecar repository status contains only intentional, attributable work.

Use:

```bash
sase bead close sase-rc --note "Artifact links are unconditional; migrated remaining v1 truth, fixed canonical project resolution and partial-write regression, removed the beta/legacy paths, and passed focused E2E, doctor, just check, and check-full verification."
```

Re-read `sase bead show sase-rc` and confirm resolution `done`. Do not close `sase-r8`;
it is already closed. Submit the turn's required `/sase_final` declaration last, after
all repository and bead changes, and make no further file changes after a successful
submission.

## Acceptance criteria

- No registered/configurable `artifact_links` feature flag remains.
- No code path implements or describes the former disabled behavior.
- No live schema-v1 Referenced By index or v1-only reader/migrator remains.
- Current workspaces map provider slugs/marker names to canonical ProjectSpec keys
  before link writes, and failure cannot leave a partial sidecar row.
- Link add/list/remove, note migration, read/cites edges, projections, companions,
  doctor, show, bead pages, and ACE relationships work without flag overrides.
- Live and isolated doctor runs are healthy and non-skipped; repeat writes, refreshes,
  migration, and repair are idempotent.
- User docs, CLI help, and all managed runtime skill instructions match the graduated
  behavior.
- Focused tests, `just check`, and monitored `just check-full` pass.
- `sase-rc` is closed with a verification note and all touched repositories are left in
  an intentional state.
