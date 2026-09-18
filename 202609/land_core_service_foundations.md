---
tier: tale
title: Finish landing sase-core service foundations
goal:
  Epic sase-11y.2.1 and its parent phase sase-11y.2 are closed on fresh landing
  verification (core pin, drift, core just check, sase just check-full), the epic's plan
  is marked done, and the sase-11y.2 close note hands the service-foundation decisions
  and API names to sase-11y.4 and sase-11y.7.
size: small
proposed_by: bbugyi200.athena.0mm
create_time: 2026-09-18 03:25:11
status: wip
---

# Plan: Finish landing sase-11y.2.1 and close phase sase-11y.2

## Context

Epic **sase-11y.2.1** ("sase-core service foundations", plan
`plan:202609/core_service_foundations.md`) implements phase bead **sase-11y.2**
(`core-service`) of the top-level epic **sase-11y** ("Service host and Services tab").

Its first land agent (`sase-11y.2.1.land`) found two integration gaps and handed them to
a nested child epic, **sase-11y.2.1.5** ("Complete service-foundation landing
integration", plan `plan:202609/complete_service_foundations_landing.md`). That child
epic is now closed, and its plan is marked `status: done`. Its land agent was supposed
to resume the parent landing after a `just check-full` monitor, but it never did.
**sase-11y.2.1** and **sase-11y.2** are still `in_progress`, and no live agent owns
them. **sase-11y.4** (service host) and **sase-11y.7** (Services tab) are blocked on
sase-11y.2.

The user asked for this landing to be finished and for both **sase-11y.2.1** and
**sase-11y.2** to be closed. Do not close **sase-11y**. Its own land agent
(`sase-11y.land`) is waiting and owns that step.

### Readiness evidence gathered while planning (re-confirm; do not trust blindly)

- **Children are closed.** All four original phases are closed: `sase-11y.2.1.1`
  (proc-service-block), `.2` (service-config), `.3` (restart-state), and `.4`
  (status-wire). The nested epic `sase-11y.2.1.5` is closed too, along with its phases
  `.5.1` (boot-scoped status stops, deduplicated orphans) and `.5.2` (Python supervision
  delegates restart accounting to the Rust core, plus the core pin ratchet).
- **Follow-ups are resolved.** Each of the four original phases left the same proposal:
  `PROPOSED FOLLOW-UP: ratchet sase-core-revision.txt past <phase> core commit`.
  `sase-11y.2.1.5.2` completed that ratchet. Phase `.2` also noted an unrelated stale
  test-node-id reference, which it fixed itself. No other proposals exist.
- **The core pin is current.** `sase-core-revision.txt` is `bd5946c1…`, which is the
  linked sase-core remote `master` head. It contains all five service-foundation core
  commits: `4cee31a` (proc service wire), `51ae484` (config composer), `a756136`
  (restart + state), `fe7c4a0` (status wire), and `3c75d2e` (boot-scoped status stops).
- **No epic symbols remain.** `sase bead epic-symbols sase-11y.2.1` and
  `sase bead epic-symbols sase-11y.2` both print "No --epic-symbol entries". The 28
  service-facade Justfile entries are keyed to the still-open `sase-11y.4`. That is
  correct, so leave them alone.
- **No drift since the last landing commit.** In sase, no commit after `6e06a3e24c` (the
  last integration commit,
  `fix(supervision): align restart and gate decisions with core`) touches
  `src/sase/service`, `src/sase/supervision`, `src/sase/procs`, `src/sase/axe`,
  `src/sase/config`, `tests/service`, the procs query profile or query adapter, the
  `Justfile`, or `sase-core-revision.txt`. Later commits `e91fa138b0` and `cc6d51d2db`
  harden gate-failure transitions in `src/sase/notification_gates/` and
  `tools/validate_sase_core_rs`. They do not consume or duplicate service-foundation
  behavior.
- **The deliverables exist.**
  - Python modules: `src/sase/service/{config,paths,boot,restart,state,status}.py`,
    `src/sase/procs/service_meta.py`, and `src/sase/supervision/restart.py`, which calls
    `decide_service_restart`.
  - Procs query fields: `service` and `svc` in the procs query profile.
  - Config default: `ace.procs.default_query: "-service"` in `default_config.yml`.
  - Core modules: `crates/sase_core/src/service/{mod,config,restart,state,status}.rs`.
  - Bindings: all eight are called through literal `require_rust_binding(...)`:
    `service_config_compose`, `service_enablement_resolve`, `service_restart_decide`,
    `service_state_mutate`, `service_state_read`, `service_status_build`,
    `service_status_read`, and `service_status_write`.
- **Only the landing checks are missing.** The first plan's "Landing" section requires a
  sase `just check-full` (through a verify monitor) and a sase-core `just check`. No
  passing `just check-full` result for this landing has been recorded yet.
- **The plan file still says `wip`.** The frontmatter of
  `202609/core_service_foundations.md` in the `plans` sidecar still reads `status: wip`.

No source changes are expected. The deliverables are verification, bead closes, and a
one-line plan-status update in the plans sidecar.

## Steps

### 1. Re-confirm readiness (fresh evidence)

1. Run `sase bead show sase-11y.2.1`, `sase bead show sase-11y.2.1.5`, and
   `sase bead show sase-11y.2`. Confirm every descendant of sase-11y.2.1 is closed and
   sase-11y.2 has no other children. Read every child note again and collect any
   `PROPOSED FOLLOW-UP:` entries that were added after planning.
2. Run `sase bead epic-symbols sase-11y.2.1` and `sase bead epic-symbols sase-11y.2`.
   Both must print nothing. If an entry appears, resolve or re-key it according to the
   Symvision epic-whitelist policy (read `symvision.md` through `/sase_memory_read`
   first). Never key an entry to sase-11y.2 or any descendant.
3. Open the linked core repo with `sase repo open sase-core -r "<reason>"` and use the
   printed path. Run `git fetch origin` there. For each of the five commits listed
   above, confirm that
   `git merge-base --is-ancestor <sha> $(cat sase-core-revision.txt)` succeeds. If the
   core remote has since gained service commits under `sase-11y.2` that the pin lacks,
   run `just ratchet-core-revision` in sase, then `just install`.
4. Rerun the drift query from the evidence section against the current `origin/master`
   (`git log 6e06a3e24c..origin/master -- <those paths>`), and do the same for the core
   repo's service/procs/binding paths since `3c75d2e`. If any new commit duplicates,
   conflicts with, or should consume the service foundations, integrate it and record
   what you did. Integration is part of this landing.

### 2. Verify

1. In sase, run `just install` (it rebuilds `sase_core_rs` from the linked core), then
   `just fix`. `just fix` should produce no diff. If it does, inspect the diff; keep it
   only if it is a legitimate formatting correction.
2. Run the focused regression batch inline:
   `.venv/bin/pytest tests/service tests/test_supervision.py tests/test_procs_facade_models.py tests/test_procs_facade_retention.py tests/test_query_profile_procs.py tests/ace/tui/test_proc_query.py tests/test_config_schema.py -q`.
3. Run `just check` in the linked sase-core checkout. Its `AGENTS.md` requires this
   instead of bare `cargo test`. If the PyO3 tests cannot find libpython, set
   `PYO3_PYTHON` and `LD_LIBRARY_PATH` to the uv-managed interpreter, as earlier phases
   did.
4. Run sase `just check-full` only through the `/sase_monitor` skill, using the
   `TESTING` / `TESTED` status pair (per the `lint_and_test.md` memory: never inline).
   Write a self-sufficient follow-up prompt for the continuation that carries all the
   evidence gathered so far and says to finish steps 3–4 of this plan.
5. If `just check-full` fails, triage each failure before closing anything.
   - Failures in service, supervision, procs, config, or query code are landing work.
     Fix them in the correct repo (Rust behavior belongs in sase-core per the
     `rust_core_backend_boundary` memory) and rerun.
   - For a failure that plainly has nothing to do with this epic, first reproduce it on
     an unchanged tree (for example, rerun the failing test by node id). Examples: a
     visual snapshot owned by another active epic, or a known flaky test. Then handle it
     through `/sase_new_task` (`flake` or `ci` type, which may corroborate an existing
     task), cite that outcome in the close note, and treat the gate as passed for this
     landing.
   - Never close with an unexplained failure.

### 3. Close sase-11y.2.1 and mark its plan done

1. Run `sase bead close sase-11y.2.1 --note "<verification summary>"` with the default
   `done` resolution and no `--force`. The note must cover:
   - the four original phases plus nested epic sase-11y.2.1.5, all verified;
   - the ratchet proposals resolved by sase-11y.2.1.5.2, with the pin SHA, its five
     contained core commits, and the fact that no other proposals were declined or
     filed;
   - the drift review result;
   - the actual verification results (focused batch, core `just check`,
     `just check-full`);
   - empty epic symbols.
2. Run `just symvision` in sase to confirm the whitelist is still clean after the close.
3. Open the plans sidecar with `sase repo open plans -r "<reason>"` before writing any
   file in it, because opening it cleans untracked files. In
   `202609/core_service_foundations.md`, change only the frontmatter line `status: wip`
   to `status: done`. Leave `202609/service_host_1.md` (sase-11y) at `wip`.

### 4. Close phase sase-11y.2

Confirm that the child plan delivered everything the parent plan's `core-service`
section requires, with the deliberate refinements recorded in the epic plan. Then run
`sase bead close sase-11y.2 --note "<summary>"` (resolution `done`, no `--force`). The
first plan says this note is how the sase-11y.4 and sase-11y.7 workers learn the
decisions, so it must state:

- **Proc wire: no proc wire schema bump.** The `service` block `{name, mode, source}` is
  an optional additive field, validated only when a row is created (`append_proc` /
  `reserve_proc`) and immutable afterwards.
- **Retention.** Named, non-transient service rows keep their newest 20 terminal rows
  per name, exempt from the generic history cap. Transient oneshots stay under the
  generic cap.
- **Default query.** The Procs default query lives at `ace.procs.default_query`
  (`"-service"`), not `tui.procs.default_query`. sase-11y.7 wires the seed.
- **Invalid entries.** A bad `service.procs.<name>` entry is returned as unavailable,
  with `available: false` and its reasons. Only section-level errors are fatal, and
  `load_service_config()` raises `ServiceConfigError` on them.
- **Restart semantics.**
  - Policies are `always | on-failure | never`; the default is `on-failure`.
  - Clean exits are code 0, any listed `success_exit_codes`, and SIGHUP, SIGINT,
    SIGPIPE, or SIGTERM. A spawn error is a failure.
  - Backoff and crash-loop accounting match the orchestrator exactly, with no permanent
    give-up.
  - `src/sase/supervision/restart.py` now delegates to `decide_service_restart`.
- **Timestamps, overrides, and stops.**
  - All wires use epoch-second `f64` timestamps.
  - Enablement overrides live in the machine-local `~/.sase/service/state.json`.
  - A stop is active only while its boot id matches (two `None` values match). Status
    derivation applies the same rule.
- **Names consumers use.**
  - The eight bindings listed above.
  - The facade modules: `sase.service.config` (`compose_service_config`,
    `load_service_config`), `sase.service.state` (`read_service_state` plus the
    set/clear/record mutators), `sase.service.restart` (`decide_service_restart`),
    `sase.service.status` (`resolve_service_enablement`, `build_service_status`,
    `write_service_status`, `read_service_status`), `sase.service.paths` (`service_dir`,
    `service_state_path`, `service_status_path`), `sase.service.boot`
    (`current_boot_id`), and `sase.procs.service_meta.ProcServiceBlock`.
  - The Procs query fields `service` and `svc:`.
- **Epic symbols.** The Justfile service-facade epic-symbol entries remain keyed to
  sase-11y.4 and should be retired as it wires them up.

Afterwards, run `sase bead show sase-11y.4` and `sase bead show sase-11y.7` and confirm
that sase-11y.2 no longer blocks them.

## Commit and finalization

- **plans sidecar.** Commit the one-line frontmatter change through the final
  declaration's `commit` decision for that repo. The message must be a conventional
  commit, for example `docs(plans): mark core service foundations done`. The plans
  sidecar rejects non-conventional subjects.
- **sase and sase-core.** No commit is expected unless step 1.4 or 2.5 required a real
  fix. If one did, commit each changed repo with a conventional message that references
  the bead. If sase files changed, run `just check` again before finishing.
- **Out of scope.**
  - Do not close sase-11y, re-key the sase-11y.4 epic-symbol entries, or edit
    `service_host_1.md`.
  - Never edit crate versions or `CHANGELOG.md` in sase-core.
