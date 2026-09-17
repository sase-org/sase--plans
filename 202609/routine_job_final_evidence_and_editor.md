---
tier: tale
title: Finish shared job identity evidence and the routine timeout editor
goal:
  Job identity resolution agrees across launch, assignment, wait, fork, and TUI, and
  canonical timeout editing preserves legacy source values.
size: medium
proposed_by: bbugyi200.athena.sase-11e.8.6.5.4.land
bead: sase-11e.8.6.5.4
status: done
---

- **PARENT:**
  [202609/routine_job_identity_diagnostic_residuals.md](https://github.com/sase-org/sase--plans/blob/main/202609/routine_job_identity_diagnostic_residuals.md)
- **BEAD:**
  [sase-11e.8.6.5.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11e/sase-11e.8.6.5.4.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-11e.8.6.5.4.land](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-11e.8.6.5.4.land.md)
- **COMMITS:**
  - [0e5ab63](https://github.com/sase-org/sase/commit/0e5ab634be4ca470a0d08c866bc393902f7a1e40)
    — feat(axe): unify routine job evidence and editor writes

# Finish shared job identity evidence and the routine timeout editor

## Outcome and boundaries

Repair only the remaining contracts reproduced while landing `sase-11e.8.6.5.4`: every
semantic `job` operation sees the same stored identity evidence, and the routine editor
shows and edits an existing timeout under its canonical label without losing the
authored key. This is one bounded tale, as the owner requested in the epic's note #1; do
not create another child epic.

Read `sase bead show sase-11e.8.6.5.4`, its latest landing note, and the audited
artifacts `plan:202609/routine_job_identity_diagnostic_residuals.md` and
`plan:202609/routine_job_identity_diagnostic_completion.md`. Audit artifact
`file:explicit:72ce5e88a1d23411d3bc4881` contains isolated reproduction scripts. Earlier
config, diagnostics, SDK, and plugin work is implemented; preserve it. This tale
continues the existing land-agent family through the normal plan approval and coder
handoff. Tale validation treats `parent_bead` as inert, so no new child epic or
purported child-epic handoff is authored. Retain the original landing objective: after
implementation and verification, resume its readiness checks before declaring the
assigned epic complete. Parent closure, post-close Symvision, and parent plan status
updates remain landing work, outside this repair plan.

All source paths here are relative to the named repository (SASE by default). Open
`sase-core` with `/sase_repo` before accessing it. Shared identity and config semantics
belong in Rust; Python may supply cached context, marshal snapshots, perform file I/O,
and render Textual fields. Read `tui_perf.md` and `src/sase/ace/AGENTS.md` before
changing UI code. Preserve AXE, scheduling/admission behavior, stored `chop` identities,
historical independent `job` identities, `.chop.` names, schema-v1 wire keys, state and
environment paths, exact user names and values, and the retained
`axe_routine_job_contract` flag (`sase-11f` owns its later removal). Use isolated
config/state and mocked launches; no live job execution or operator config change is
needed.

## Verified baseline and failures

The audit used SASE `4e9278048a779b08ddc972c260fe6fc6e9611fbe`, pinned core
`575f97a8de11461e20d4bd1a675444bc91585231` (v0.34.41), and the actual published
`sase-core-rs==0.34.41` wheel. The opened core was `e210d18` (v0.34.42).

All five epic phases are closed. Their Python commits `bb839b4ea8`, `c05aa3a94a`,
`897e69e1e1`, `4e9278048a`, and core `f04da63` are present. The live diagnostic changes
and core floor are implemented; doctor no longer rewrites embedded user values. The
following production counterexamples still fail:

1. With no assignment-file entries, put a waiter in project A at timestamp
   `20260917010000`, a completed `tribe: chop` agent in A at `20260917020000`, and a
   running `tribe: job` agent in project B at `20260917015000`. The runner fast path
   returns resolved=True using `chop`; the all-project wait check returns False using
   `job`; fork says no completed `@job` entity exists. Replace B's direct tribe with
   only `clan_tribe: job` on a clan member and the same mismatch occurs.
2. In either fixture, `resolve_agent_tribe_assignment(..., "job", layers=[])` returns
   `chop`. An actual mocked `%id(fresh, tribe=job)` launch writes metadata `chop`,
   although the global index recognizes an independent `job`. Writers in
   `ace/agent_tribes.py`, `axe/run_agent_directives.py`,
   `axe/chop_proposal_planning.py`, and `ops/commands/_agent_directive.py` still consult
   only `agent_tribes.json`. This does not meet the plan's metadata plus store plus
   effective clan evidence contract.
3. TUI `collect_agent_wait_status_maps` with a waiter, a running clan-only `job` member,
   and a completed `chop` row binds `@job` to the completed `chop` agent. Adding an
   unrelated direct `job` row older than the waiter flips it to the pending `job` clan.
   `ace/tui/_agent_completion_wait.py` gathers only `row.tribe` and omits
   `row.effective_clan_tribe` from the evidence used to interpret the reference.
4. For legacy config `axe.lumberjacks.checks.chop_timeout: 123s`,
   `compose_axe_config -> _build_axe_editor_seed -> build_axe_entry_form` displays
   `job_timeout` with `has_effective=False`, `has_target=False`, and no draft value.
   Both flag states reproduce with inherited flag transport removed. Composition
   inventory and contributions still contain `chop_timeout: 123s`; phase .3 only
   switched the visible field name and hid the old one.

The new acceptance file `tests/test_axe_cli_chop_run_contract_repairs.py` does not cover
these counterexamples. Its two diagnostic tests force only the On state. The existing
cross-project unit test in `tests/test_run_agent_wait_deps.py` seeds a redundant global
assignment entry, masking the metadata-only case.

## Repair the shared context and its consumers

Start with `core/agent_tribe.py`, `ace/agent_tribes.py`,
`core/wait_dependency_resolution/{_index,_index_queries}.py`,
`axe/run_agent_wait_deps.py`, `axe/run_agent_directives.py`,
`axe/chop_proposal_planning.py`, `ops/commands/_agent_directive.py`,
`scripts/_agent_chat_from_name_tribe.py`, `scripts/sase_chop_wait_checks.py`, and
`ace/tui/_agent_completion_wait.py`.

Define one explicit semantic context containing cached config layers, direct metadata
tribes, assignment-file identities, and precedence-resolved effective clan tribes across
projects. Keep `sase-core`'s existing `resolve_agent_tribe_identity` as the identity
decision point. Reuse the existing clan precedence resolver; do not union raw superseded
clan declarations. Python may gather and pass these already-defined inputs, but any new
shared resolution policy belongs in core with its binding/tests.

Make the runner's project-local index receive this global identity evidence without
changing ordinary project-local dependency lookup. For a cross-project `@job` that has
no local candidate it must remain waiting, never release against a local `chop`. The
wait-check and fork keep their existing all-project candidate behavior. Reuse an
existing indexed/snapshot source or a narrow invalidatable cache for global evidence; do
not introduce an uncached all-project artifact scan on every wait tick. The persistent
artifact index facade is in `core/agent_scan_facade.py`; any use must account for
stale/missing index data and historical metadata, not silently treat an incomplete
hot-row query as complete evidence. Cache keys must separate temporary
`SASE_HOME`/project roots and invalidate when metadata, clan assignments, or the
assignment store change. Record scan/read counts and before/after timing for repeated
wait checks.

Use the same context for `%id`, `%clan`, CLI assignment, TUI direct and clan assignment,
and proposal preparation. Resolve and reject before metadata writes, prompt rewrites,
name claims, or assignment persistence. Retain the resolved identity through persistence
rather than making conflicting decisions from independently loaded evidence. Include the
current historical metadata assignment when the store has no entry. Preserve
no-independent-identity `job -> chop`, same-name historical `job` updates, early
same-/cross-layer collision rejection, and raw launch prompt bytes. Keep the syntactic
`@tribe` recognition path cheap; supply context wherever a caller actually chooses an
identity. In particular, wait/fork/TUI parsing currently supplies no config layers;
ensure source-aware collision diagnostics from the ancestor contract survive the
consolidated context instead of silently choosing an alias.

For TUI wait binding, use the shared evidence projection over already-loaded direct and
effective-clan rows. Pass any required global/context snapshot from the existing worker
load path. Rendering, completion, and event handlers must not discover config, scan
artifacts, read assignment files, or take filesystem locks. Preserve the existing
binding-derived wait-lane colors and stored-panel-key neighbor/collapse fixes.

## Repair the routine timeout field projection

Start with `ace/tui/actions/axe_config_actions/_backend.py`,
`ace/tui/modals/axe_entry_editor_types.py`, `ace/tui/modals/axe_entry_editor_modal.py`,
`axe/config_backend.py`, and core `config/{axe,plan}.rs`.

The editor requires a coherent canonical field view of effective values, inherited
values, selected target contributions, per-scope contributions, and provenance. Reuse
the Rust-owned structural projection and source mapping; do not implement another alias
normalization dictionary in the Python config backend. If necessary expose a small
additive entry-editor projection through the existing config wire/binding while
retaining internal schema-v1 entry identities. Public human field labels remain
canonical in both rollout states even though general effective/JSON projections remain
flag-dependent.

Ensure `job_timeout` initially shows `123s`, its actual contributing scope, inherited
state, and provenance for legacy source. Scope switching must preserve that mapping.
Editing/resetting/unsetting this field must use the existing lossless edit planner to
modify/remove the authored `chop_timeout` contribution in place; new canonical entries
may use `job_timeout`. Preserve comments, siblings, exact routine names, and avoid
duplicate alias subtrees. Verify equivalent canonical and legacy authored forms, sparse
inherited overrides, and both rollout states through the production seed, form, preview,
apply, and recomposition paths.

## Acceptance and integration

Add regression coverage for the counterexamples above, using real temporary metadata
rather than patching each consumer to an independently chosen evidence tuple. Include:

- Metadata-only and effective-clan-only independent `job` in another project, with no
  assignment entry; both running and completed candidates; a completed local `chop` must
  not release a `job` wait. Wait-check/fork select the same identity and preserve their
  established candidate order.
- Launch and proposal identity, same-name metadata-only historical updates, direct and
  clan TUI mutation, and CLI assignment. Rejected same-/cross-layer collisions leave
  prompt/meta/store unchanged and do not claim a name.
- In-memory TUI clan-only binding stays stable when an unrelated older direct `job` row
  is added, and uses the correct pending/bound color. No new I/O in render or
  completion; unchanged repeated waits reuse global evidence, and changes invalidate it.
- The timeout form/preview/apply regression described above, including scope changes and
  reset/unset with comments and sibling values preserved.

Extend the existing acceptance fixture under both flag states, explicitly removing
inherited `SASE_FEATURE_FLAGS` and asserting the resolved state. Include the two new
Rust diagnostic probes in that matrix; keep the established raw-value doctor and
collision tests. Preserve the verified diagnostic wording and all existing upgrade
coverage rather than recreating the earlier epic phases.

Review post-audit drift before completion. Already-reviewed changes since this epic
started include service proc metadata, child-supervision extraction `dfb07cbbbd`, cgroup
escape `86458d2607`, launch-admission splits, hold preview/arming, monitor recovery, and
documentation/test splits. They do not require a routine/job redesign. Preserve their
behavior; mechanical test splits require using current file paths. The final fetch found
`origin/master` one commit ahead of the audited checkout at `96288aea4c` (prompt-history
modal extraction), which does not touch these repaired contracts. Preserve that split
when refreshing the coding tree.

Run focused identity, wait/fork, persistence/proposal, completion/display, editor and
upgrade suites. Run affected narrow/wide visual tests for any changed field or lane;
inspect actual/expected/diff before changing goldens. Run `just install` before the
required checks. If core changes are needed, run core `just check` with Python >=3.12
PyO3 coverage, land the change through host finalization, and wait for a complete
published release before ratcheting SASE's dependency floor and revision using the
supported tools. Keep published-floor evidence separate from local editable builds. Run
binding/version checks and the advisory floor probe after any ratchet.

Run `just fix` inline and SASE `just check`; before combined completion run
`just check-full` only through `/sase_monitor` with TESTING/TESTED and a mechanical
continuation. Record exact SASE/core revisions and complete gate results. Existing
acceptance monitor `m3mx0kg9fs18` had 42,448 passes, 15 skips, and one unrelated
failure; it was not a green full gate. If the independent stale-node repair below has
landed, integrate it before the final full run. Otherwise keep its failure explicit and
coordinate through its existing task, without hiding or skipping the test.

## Follow-up outcomes to carry into the landing note

All five phase proposals were triaged with `/sase_new_task`:

- Phase .1 note #1 and phase .2 note #1 (`hold` directive vocabulary), plus phase .4
  note #1 (missing hold Rust bindings), are resolved on the audited tree. Core `a685c07`
  supplies the bindings; Python commit `86458d2607` repairs the contract expectation.
  Four affected suites passed together: 50 tests on published 0.34.41. No new tasks were
  created for these three resolved proposals.
- Phase .4 note #2 is corroborated on existing flake task `sase-120` (+2), with the
  proposing phase identified; the causal fixture issue was also appended to active epic
  `sase-10w`. No new land-agent flake reproduction is claimed.
- Phase .5 note #1 is ready small CI task `sase-122`. Commit `99764a3fc7` moved
  `test_deep_merge_list_concatenation` to `tests/test_config_merge.py`, leaving
  `tests/test_proc_env_isolation.py::_SASE_ML_FILE_FAMILIES` stale. The land agent
  independently reproduced its deterministic missing-node failure. Do not duplicate this
  task.

The ancestor task dispositions remain `sase-11m` (missing chezmoi busted) and `sase-10y`
(artifact attachment/hidden-plan write trouble). The current epic has no `--epic-symbol`
entries at audit. Its closure and the directly parented plan-ancestor readiness checks
resume only after this tale's repaired contracts are verified.

The audit snapshot above is confirmed exact/live by `sase artifact show`. Its attachment
command returned a legacy/event overlap projection error even though the subsequent show
displayed the related bead edge. This independent recurrence was recorded on causal
active epic `sase-yy.8.6`; no duplicate task, migration, or hidden-clone cleanup was
attempted.
