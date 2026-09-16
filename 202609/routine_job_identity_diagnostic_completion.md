---
tier: epic
title: Finish contextual job identity and public diagnostic contracts
goal: "Every canonical job identity operation and live routine/job diagnostic honors the
  compatibility contract, and the complete upgrade passes published-floor and full
  landing verification.

  "
parent_bead: sase-11e.8.6
phases:
  - id: tribe_context
    title: Route every job tribe operation through contextual identity resolution
    size: medium
    depends_on: []
    description:
      "tribe_context: unify assignment, query, wait/fork, completion, and display
      resolution without changing stored identities."
  - id: public_diagnostics
    title: Finish canonical live diagnostics without rewriting user data
    size: medium
    depends_on: []
    description:
      "public_diagnostics: update remaining reachable SDK, context, CLI, TUI, and
      routine-log templates at their owners."
  - id: acceptance
    title: Prove the complete routine and job upgrade contract
    size: medium
    depends_on:
      - tribe_context
      - public_diagnostics
    description:
      "acceptance: exercise both-state production paths, published-floor compatibility,
      drift, and full repository gates."
proposed_by: bbugyi200.athena.sase-11e.8.6.land
create_time: 2026-09-16 09:55:14
status: wip
---

- **PROMPT:**
  [prompts/202609/routine_job_identity_diagnostic_completion.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/routine_job_identity_diagnostic_completion.md)
- **PARENT:**
  [202609/routine_job_final_contract_repairs.md](https://github.com/sase-org/sase--plans/blob/main/202609/routine_job_final_contract_repairs.md)

# Finish the remaining routine/job landing contracts

## Scope and handoff

This plan contains only work still incomplete during the landing of `sase-11e.8.6`. Read
`plan:202609/routine_job_final_contract_repairs.md`, its parent plans
`plan:202609/axe_routine_job_landing_repairs.md` and `plan:202609/axe_routines_jobs.md`,
and the landing notes on `sase-11e.8` and `sase-11e.8.6` through audited artifact and
bead commands. Their compatibility decisions remain in force.

The `parent_bead: sase-11e.8.6` relationship is the mechanical handoff back to the
interrupted landing. Do not add closing this parent, its Symvision pass, or its plan
status update as implementation phases. The eventual land agent must recheck this epic
and then directly parented plan ancestors normally; never force a successful landing.

Keep AXE, `sase axe`, the AXE tab, `sase tui`, scheduling and admission semantics,
private wire fields, `.chop.` agent names, state directories, graph identities,
historical assignments, accepted legacy inputs, and opaque user values intact. Keep the
sunset flag `axe_routine_job_contract` and its independent removal bead `sase-11f`.
Shared backend/domain behavior belongs in Rust; Python owns file I/O, cached adapters,
and presentation. Use `/sase_repo` before accessing `sase-core` or another non-primary
repository. Before TUI changes read `tui_perf.md` and `src/sase/ace/AGENTS.md`; never
put configuration discovery or artifact scans on a render, completion, or event-loop
path.

No live AXE start/stop, real job execution, Telegram send, deployment, or operator
configuration application is required. Tests use temporary config/state and mocked
launches.

## Verified baseline and remaining failures

The landing audit used SASE `8c9d04759bb441d99211d0be1f4afb7698032012` and `sase-core`
`2d7fa287fba3482d5482b1ebfe71536caec730e6`; the SASE pin is
`51c7c38d6d1192fad0a3c807cc233b7ccdcb1acf`. It reviewed the epic and all four children,
every note, the linked plan, phase commits in both repositories, current source and
tests, and all post-start commits. All child phases are closed and contain one
completion note each. Neither the epic nor its children contain a `PROPOSED FOLLOW-UP:`
entry.

Source-preserving missing-leaf and list-form edits are implemented in `config/plan.rs`
plus the Python apply adapter, and the explicit config/diagnostic counterexamples from
the prior landing note now pass. The core pin contains the repair bindings. Post-start
core commits only repaired release workflow concurrency and cut v0.34.36; they do not
conflict with the feature. The canonical dependency-window tool still reports 0.34.35 as
the newest complete PyPI release, so do not manually advance the floor until the
supported workflow observes a complete release.

Two required contracts remain incomplete:

1. **Context-aware tribe resolution is not used by the operations that mutate or target
   identity.** After rebuilding the current binding, a layer containing distinct
   `ace.tribes.chop` and `ace.tribes.job` records is correctly rejected by
   `canonicalize_public_tribe_name("job", layers=...)`, but public
   `ace.agent_tribes.set_tribe(..., "job")` has no layer context and still stores
   `chop`. Plain `parse_tribe_reference("@job")` also returns `chop`; only the new call
   with stored-tribe evidence returns the independent historical `job` identity.
   `scripts/_agent_chat_from_name_sources.py` uses the plain parse before
   `resolve_tribe_fork_source`, so a wait may bind historical `@job` correctly while the
   following fork selects `chop`. Assignment CLI/TUI paths likewise do not surface
   config collisions. Existing focused inventory, historical-assignment, wait, and
   display tests pass, proving the production paths are missing from coverage.

2. **Live human diagnostics still expose old vocabulary on canonical surfaces.** For
   example, `ChopNotFoundError` and `AmbiguousChopError` render `chop`, `lumberjacks`,
   and `--lumberjack`; the TUI forwards those strings directly from
   `ace/tui/actions/axe_chop_run.py`. Public job context/env failures still say
   `conflicting chop context aliases` and `could not resolve chop env`, and routine logs
   in `axe/lumberjack.py` announce `Lumberjack ... chops`. The prior diagnostic phase
   fixed the specifically reported Rust validation and status templates, but it did not
   complete the required reachable-message audit across SDK/context/editor/CLI and logs.
   Canonical wording must not change exact names, paths, executable values, raw keys,
   internal codes, or immutable historical output.

The current integration fixture covers canonical list, doctor, one harmless run,
state/linkage records, and status, but it does not exercise the promised both-state
matrix, contextual assignment/fork collision, edit paths, TUI diagnostic route, timeout,
env scrubbing, or job-link navigation together. No full `just check-full` landing gate
has been recorded. `sase bead epic-symbols sase-11e.8.6` currently has no entries;
recheck only at actual closure.

## Phase `tribe_context`

Make the Rust identity resolver the single decision point for assignment, query,
wait/fork target resolution, completion, and display. Start with
`sase-core/crates/sase_core/src/agent_tribe.rs`, the PyO3 binding,
`src/sase/core/agent_tribe.py`, `src/sase/ace/agent_tribes.py`,
`src/sase/agents/cli_tribe.py`, `src/sase/ace/agent_query/evaluator.py`, the wait index,
and the fork/name/completion callers found by auditing every
`parse_tribe_reference`/`canonicalize_public_tribe_name` use.

Provide one explicit context object or adapter that carries config-layer provenance,
stored identities, and the current assignment when an operation needs semantic
resolution. Syntactic recognition may stay cheap, but it must not irreversibly
canonicalize `@job` before the indexed/store-aware operation runs. Assignment and
targeting must reject a same-layer or cross-layer collision with the resolver's
source-aware diagnostic and leave stores unchanged. A wait followed by a fork must use
the same resolved identity. Preserve ordinary built-in `job -> chop` writes when no
independent identity exists, preserve a current historical `job` assignment on a
same-name update, and keep metadata/history reads byte-faithful.

Reuse config tokens, cached layer inputs, and existing wait indexes. Do not discover
configuration, scan artifacts, or read assignment files from a TUI render/completion
path. If a UI action needs uncached resolution, run it in the established worker path
and report the diagnostic without optimistic persistence. Keep reserved-tribe behavior,
clan aggregation, earliest-complete wait ordering, glyph/color/grouping/collapse
settings, and exact `.chop.` agent names.

Add Rust/PyO3 tests plus public Python tests for built-in customization, same-layer and
cross-layer collision, independent stored `job`, assignment CLI and TUI mutation, query
filtering, wait followed by fork, chat/resume/fan-out references, and completion. Assert
rejected mutations do not change temporary stores. Cover both the cheap recognition path
and the contextual semantic path so another caller cannot silently fall back to plain
`job -> chop` canonicalization.

## Phase `public_diagnostics`

Complete the reachable-message audit promised by the parent plans. Start with
`src/sase/axe/chop_runner_types.py`, `chop_script_context.py`, `chop_env.py`,
`chop_runner_script.py`, `lumberjack.py`, the CLI and TUI callers that display their
errors/logs, and remaining live strings in Rust `axe_chop`, `config`, and `axe_status`
owners. Classify residual old-word strings as private identifiers, compatibility
keys/paths, immutable stored output, or live human text. Change only the live human
text, at its owner when possible.

Canonical job/routine surfaces use job/routine language in both rollout states and
through hidden legacy command aliases. Public errors may quote exact compatibility field
names such as `lumberjack_name` and `verbose_lumberjack_diagnostics`, but the
surrounding explanation must remain canonical and actionable. TUI notifications and
routine aggregate logs must not leak old labels merely because their private backing
types remain named `Chop`/`Lumberjack`. Keep internal diagnostic codes, schema-v1 wire
keys, state paths, environment variable names, exact configured executable paths,
arbitrary descriptions/values, and historical log bytes unchanged. Do not reintroduce
whole-string replacement over rendered values.

Add production-path tests for CLI and TUI not-found/ambiguous job errors, SDK/context
alias conflicts, env-secret failures without secret leakage, scheduled/manual run
preflight errors, routine startup/action logs, generated-entry errors, and the existing
duplicate/missing-description/status cases. Include names and paths containing
`chop`/`lumberjack` to prove only owned templates change. Exercise canonical and hidden
legacy invocations in both flag states while retaining v1/v2 envelope rules.

## Phase `acceptance`

Extend or split the isolated upgrade fixture so it proves the repaired production
contract rather than only a canonical happy path. Exercise equivalent legacy and
canonical configuration through general and AXE source edits, routine/job list, doctor,
a harmless fixture-script run, history/public JSON, timeout/preflight behavior, env
alias propagation and scrubbing, job/agent link navigation, and TUI-visible diagnostics.
Run both rollout states with inherited `SASE_FEATURE_FLAGS` removed and assert the
resolved state. Include map/list forms, exact dotted names, duplicate job names across
routines, expanded targets, old-only plugin entrypoints, equal/conflicting aliases, and
the independent `job` tribe assignment/wait/fork case. Assert exact source bytes, stored
identities, opaque values, and stable graph identities as well as canonical envelopes.

Recheck drift from the first `sase-11e.8.6` commits on SASE and core, excluding this
epic's commits. Preserve the post-start core release-concurrency fix and v0.34.36
release metadata. Re-run `tools/ratchet_core_window --report-only` and
`tools/probe_core_floor --advisory`; when v0.34.36 is a complete published wheel, use
the supported dependency-window ratchet and validate the published floor. Until then,
the missing published bindings are a real landing blocker, not permission to claim a
local editable build as release proof. Keep `sase-core-revision.txt` on an actually
verified revision and use the supported pin ratchet for any later core drift.

Run focused suites, full core `just check` with Python >=3.12 PyO3 coverage, SASE
`just check`, binding/version/floor checks, schema/default/completion validation, and
affected narrow/wide visual tests (inspect actual/expected/diff before accepting any
golden). Before combined landing, run `just check-full` only through `/sase_monitor`
with TESTING/TESTED evidence and a mechanical continuation. Record exact SASE/core
revisions and published-floor evidence. Do not claim success while the floor or any
epic-caused failure remains.

## Follow-up disposition

No child of `sase-11e.8.6` proposed a follow-up, so there is no new independent task to
file. The assignment/fork and diagnostic gaps are caused by this epic and remain epic
work in this plan. Preserve the ancestor dispositions: object-sharing wire alignment is
resolved; missing chezmoi `busted` remains tracked by `sase-11m`; hidden-plan artifact
attachment trouble remains corroborated on `sase-10y`. Do not create duplicates or claim
chezmoi's full check passed.
