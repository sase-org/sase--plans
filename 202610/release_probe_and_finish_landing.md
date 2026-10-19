---
tier: tale
title: Repair the release install probe and finish the sase-1mk landing
goal:
  The release watcher verifies the real sase version command, sase 0.18.0 has audited
  installation and release evidence, and sase-1mk.9.7 plus every fully complete directly
  parented plan ancestor are closed normally.
size: medium
proposed_by: bbugyi200.athena.sase-1mk.9.7.land
proposed_with: codex/gpt-6-astra
bead: sase-1mk.9.7
create_time: 2026-10-19 07:54:20
status: wip
---

- **PARENT:**
  [202610/ship_v0_18_0_marker_audit_repair.md](https://github.com/sase-org/sase--plans/blob/main/202610/ship_v0_18_0_marker_audit_repair.md)
- **BEAD:** sase-1mk.9.7

# Repair the release install probe and finish the sase-1mk landing

## Scope and verified starting point

This is the remaining work from landing **sase-1mk.9.7**, not another release epic. One
coder owns the bounded watcher repair and the entire closeout below. There is no later
land agent for this tale. Close the completed epic in the same turn as the final code;
the host commits afterward. Nothing below depends on this tale's own future commit SHA,
push, or CI result.

Read these with `sase artifact read` and the bead records with `sase bead read`:

| Bead         | Linked plan                                     | Direct parent |
| ------------ | ----------------------------------------------- | ------------- |
| sase-1mk.9.7 | plan:202610/ship_v0_18_0_marker_audit_repair.md | sase-1mk.9    |
| sase-1mk.9   | plan:202610/ship_v0_18_0_post_repair_reds.md    | sase-1mk      |
| sase-1mk     | plan:202610/ship_v0_18_0_owned_blockers.md      | none          |

The lander has already reviewed both children and every note of sase-1mk.9.7:

- **sase-1mk.9.7.1**, done: commit `850b9fdfcb8e691ecf9c19926535439e84e003d8` adds
  `write_json_marker_atomic` to the mutation scanner, changes both done-marker mutation
  tuples to that call while preserving index lifecycle coverage, and reviews the inline
  error-marker path. The source still writes atomically and immediately refreshes the
  index. JSON readers consume named fields, not key order.
- Fresh lander verification at `4b7b98153227a355e7d183ea25df4d56694e2998` passed all 15
  tests in the mutation/path-passing audit modules,
  `tests/test_run_agent_runner_completion_helpers.py`, and
  `tests/fakey/test_provider_drain_e2e.py` in 45.68 seconds.
- **sase-1mk.9.7.2**, canceled: the watcher correctly closed on its failure outcome, but
  reported `INSTALL_FAILED` because its fresh venv executed unsupported
  `sase --version`, which exits 2. Preserve that historical resolution; it records what
  the original watch actually did.
- The only concurrent code commit since the repair is `c2cbd80f1c`, which splits
  `tests/tool/test_triage_inputs.py` into focused modules. It does not conflict with the
  marker repair. Release commits `cde1dff6fb`, `97f5e75f47`, and `4b7b981532` set sase
  0.18.0 and the core dependency window `>=0.42.0,<0.43.0`. No repair was reverted.
  Fetch and inspect any newer drift before acting.
- `sase bead epic-symbols sase-1mk.9.7` was empty during this review; recheck before
  closing.

Fresh GitHub reads confirmed:

- PR [#299](https://github.com/sase-org/sase/pull/299) merged at `2026-10-19T11:01:45Z`
  by `bbugyi200`, merge commit `4b7b98153227a355e7d183ea25df4d56694e2998`.
- GitHub release [v0.18.0](https://github.com/sase-org/sase/releases/tag/v0.18.0) was
  published at `2026-10-19T11:07:33Z`.
- Publish [39242329155](https://github.com/sase-org/sase/actions/runs/39242329155) at
  that merge commit succeeded in `release`, `build`, `install-smoke`,
  `install-smoke-core-floor`, and `publish`; none of these jobs was skipped.
- Full CI [39229885153](https://github.com/sase-org/sase/actions/runs/39229885153)
  succeeded at `850b9fdfcb`, updated `2026-10-19T10:58:47Z`. Master Gate
  [39237716543](https://github.com/sase-org/sase/actions/runs/39237716543) succeeded at
  pre-merge tip `c2cbd80f1c31b8d3c6a3432b8ec4e6c418e75f58`, updated
  `2026-10-19T10:40:33Z`.
- The watch's chosen `CORE` was **0.42.1**, and its note reports that version on PyPI.
  Independently confirm core files, tag ancestry, and macOS evidence below.

The inherited decisions are final: **core_cut=yes**, **stale_epics=supersede**. Do not
change ci_watch logic/config, merge a PR manually, edit release versions, changelogs, or
dependency windows, or weaken CI/watcher gates. No new CLI option is needed. The current
user explicitly prefers a tale for a single-coder repair, so this tale replaces the old
linked plan's automatic nested-epic prescription for INSTALL_FAILED. Its literal
`sase --version` proof instructions are corrected below to the existing supported
command.

## 1. Repair the install proof and its regression coverage

Read the tools and lint/test memory before editing. The defect is local orchestration in
`tools/_release_watch_engine.py:ReleaseWatcher.install_attempt`:

- Both the uv and Python/pip branches currently execute `bin/sase --version`.
- The real parser is `src/sase/main/parser_version.py`: `sase version --json` is
  supported. `src/sase/main/version_handler.py` returns the runtime inventory JSON.
- `tests/test_release_watch_lifecycle.py::test_install_attempt_uses_fresh_venv_and_records_core_version`
  currently mocks the invalid flag as successful, concealing the defect.

Run the installed venv's absolute `bin/sase` with `version --json` in both branches.
Read the stable inventory schema in `src/sase/version/render.py` and validate the
`packages` row named `sase`: its `distribution_version` must exactly equal the requested
release. Do not accept a substring match anywhere in the output, a version from a plugin
row, missing host metadata, or malformed JSON. Preserve the existing subprocess
failure/timeout diagnostics and return a useful failed-proof detail. Update the
success/failure wording so it names the supported command.

The current code also only records the installed core version, without comparing it to
the configured minimum; its existing happy-path test even uses 0.40.1. Keep recording
the installed `sase-core-rs` distribution version. When `await_core_version` is
configured, the installed core must satisfy it; reuse `core_version_meets` from
`_release_watch_api.py`, already imported by the engine. This verifies the existing
release requirement, rather than merely checking that a new core happens to exist on
PyPI. Keep the existing pinned watcher install behavior, retry window, cleanup, and
verdict/close lifecycle unchanged.

Update the lifecycle test with representative inventory JSON and exercise both uv and
fallback paths. Cover an exact matching host, a wrong host version even when a plugin or
another field contains the target string, malformed/missing host data, the configured
installed-core minimum, and subprocess failure. Tests must reject the old invalid argv.
Do not make tests contact PyPI; the real install proof is step 3.

## 2. Verify the code before closeout

Run the release-watch test modules, then `just fix` and `sase tool run check` (the
recorded wrapper for `just check`). Never run `just check-full`. The marker suite
already passed at the reviewed revision; repeat it if intervening changes affect its
writers, scanner, or reviewed contexts.

Read ToolRun results accurately: a classified known/flaky nonzero result is not a pass.
Repair new failures caused by this change. Do not manufacture a reason to skip
verification, weaken assertions, or rerun an unchanged whole suite without new evidence.
For a long command use `/sase_monitor` with a successor explicitly instructed to resume
this tale's unfinished proof and closeout steps. Do not use a completion-only monitor
that would finalize before those steps. Do not wait for CI on this tale's future commit;
verification here covers the local fix.

## 3. Confirm the already-published release and create its evidence artifact

Refresh the release evidence without dispatching another release watch merely to repeat
the already-successful publication. Reuse the historical watcher note on sase-1mk.9.7.2
(including its verdict content) and append the corrected proof. The release is confirmed
only after all of the following:

1. Open `sase-core` through `/sase_repo`, fetch, and prove tag `v0.42.1` contains the
   core-fixture repair `80dc704a` and `mobile_pairing_issue`. Inspect released commit CI
   and require its macOS job to have passed. Verify every distribution expected by the
   core publish workflow exists on PyPI; retain filenames and sha256.
2. Read the matching #299 row in
   `~/.sase/axe/lumberjacks/ci_watch/ci_watch_releases.json` and associate it with the
   merge commit, `mergedBy`, exact pre-merge green Master Gate, and green Full CI less
   than six hours old at merge. The run IDs above are leads, not a substitute for the
   ci_watch receipt. Read-only runtime evidence is sufficient; do not edit it.
3. Reconfirm GitHub release v0.18.0, required Publish jobs, and PyPI's sase 0.18.0
   files. Record URLs, run/job IDs, SHAs, times, and file sha256 values.
4. In a fresh temporary directory outside every repository, create a venv and run
   unpinned `uv pip install --refresh --python <venv>/bin/python sase`. Invoke that
   venv's absolute `sase version --json` from the temporary directory. Require the
   installed host distribution to be exactly 0.18.0 and installed core to be at least
   0.42.1; record install output and inventory. Ensure no workspace `PYTHONPATH` or
   editable installation contaminates the probe. Do not change the global installation.
5. Exercise the repaired `install_attempt` against a separate fresh temporary venv with
   target 0.18.0 and core minimum 0.42.1; preserve its successful diagnostic. This needs
   no merge, workflow dispatch, or new long watcher. Use a monitor if the install cannot
   complete within the current command/turn constraints.

Read the artifact memory. Assemble `sase_v0_18_0_release_evidence.md` in a temporary
directory outside repositories, containing the proofs, original INSTALL_FAILED verdict,
corrected proof, verification results, and the task dispositions from step 4. Register
one final artifact with
`sase artifact create -p <file> -l "sase v0.18.0 PyPI release evidence" --bead sase-1mk.9.7`.
Link that artifact to `bead:sase-1mk.9` and `bead:sase-1mk` using relation `related` and
reason `v0.18.0 release evidence`. Reuse it for ancestor closes; do not create a second
release evidence artifact. If one already exists when this tale runs, audit and reuse
it, adding new proof as a clearly linked supplement only if needed.

## 4. Finish the release closeout obligations

The current epic's sole `PROPOSED FOLLOW-UP:` is already triaged. Phase **sase-1mk.9.7.1
note #2** reported a 180-second nested pytest timeout in
`tests/sdd/test_git_identity_fixture.py::test_sdd_git_identity_survives_empty_home_subprocess`.
The lander used `/sase_new_task`, read the retained ToolRun
`e996bbfb6e8d632139c7686cc9268d44` (FLAKY/no_new_failures), searched existing and recent
tasks, swept active epics, and recorded **+1 on existing sase-120** (+17 at this
review). This relays independent phase evidence; it claims no new lander repro. No
duplicate task was created and no proposal was declined. No causal link to an active
epic was established. The probe failure remains this release epic's work. These outcomes
are recorded in the landing audit note on sase-1mk.9.7; repeat them in its final close
note, without another +1 for the same observation.

Complete the linked plan's remaining landing obligations on evidence:

- Read the repair phase notes in the ancestor chain and the candidate task records.
  Close only tasks whose fixes are supported by phase stress evidence plus the release's
  green Full CI/Master Gate. Candidates named by the release plans are sase-1a7,
  sase-1mj, sase-1k9, sase-1j4, sase-1ht, sase-1a2, sase-1lr, sase-1mi, sase-1m9,
  sase-1l7, sase-1l8, sase-1ky, sase-1mm, sase-1k2, and this epic's marker-drift task
  sase-1mt. Record why each closes or remains open. Preserve prior closes and do not
  convert one green suite into unsupported claims of flake repair.
- Record the release outcome on sase-1m3.7.1. Close sase-10d only if its actual scope is
  proven satisfied. Triage the existing Install smoke issue and the 3.12 coverage time
  margin (sase-v8) from the run evidence. Use `/sase_new_task` for genuinely new
  independent issues, respecting duplicates and causally related active epics.
- Honor `stale_epics=supersede` only after release proof. Inspect current descendants of
  the stale sase-1lh chain (previous deepest plan sase-1lh.7.5.4.3.3.4.4), sase-1kx
  chain (previous deepest plan sase-1kx.5.3.3.3), and sase-1c1. Confirm the release work
  is superseded and no new independent/live scope needs preserving. Retire their
  epic-symbol entries before closing. Deliberately close stale plans deepest-first with
  resolution `superseded`, explicit reason
  `superseded by sase-1mk.9.7: sase v0.18.0 released`, and the evidence artifact. The
  inherited decision authorizes `--force` only for deliberately superseding unfinished
  descendants in those stale chains, never for successful completion. Give sase-1i5.9.1
  and sase-1jz evidence notes only; they retain plugin-floor and golden work outside
  this release closeout.

## 5. Close sase-1mk.9.7 and recheck each direct plan ancestor

This is the tale's final implementation step and is mandatory. A code-only result leaves
the task incomplete. Keep the original canceled watch history intact; the epic closes on
the newly established proof.

1. Re-read sase-1mk.9.7 and its descendants/notes and linked plan. Confirm the remaining
   repair and release obligations above are complete. Run
   `sase bead epic-symbols sase-1mk.9.7`. Resolve every entry by wiring it, making it
   private, adding a justified non-test pragma, or deleting dead code per Symvision
   policy. Re-key only to a still-open later bead that actually needs the exemption.
   Verify any resulting code changes before closing.
2. Close normally:
   `sase bead close sase-1mk.9.7 --note "<marker repair and drift verification; corrected install proof; release artifact; test results; every follow-up disposition>"`.
   If refused, fix the identified symbol/descendant/linked-plan readiness problem and
   retry. Never use `--force` merely to get a successful close, and never use it to
   advance this successful nested landing.
3. Run `just symvision` (via the tool wrapper if guarded). Open the plans repo with
   `/sase_repo`, use its printed path, audit-read
   `plan:202610/ship_v0_18_0_marker_audit_repair.md`, then set `status: done` in that
   file's frontmatter. Preserve its history and other metadata.
4. Re-read `sase bead read sase-1mk.9.7 -r "Need the parent link"`. Its expected direct
   parent is plan bead **sase-1mk.9**. Review that parent's previous landing notes,
   **all descendants and every note**, linked plan
   `plan:202610/ship_v0_18_0_post_repair_reds.md`, and post-child source drift. Recheck
   descendant and linked-plan readiness before deciding to close. Reuse the release
   artifact and prior follow-up dispositions; investigate any new notes. If fully
   complete, retire every entry from `sase bead epic-symbols sase-1mk.9`, then
   `sase bead close sase-1mk.9 --note "<what was rechecked; release artifact>"`, run
   `just symvision`, and set its linked plan's frontmatter `status: done`.
5. Repeat that full readiness review for direct parent plan **sase-1mk**, including all
   descendants/notes, accepted decisions, post-child drift, and
   `plan:202610/ship_v0_18_0_owned_blockers.md`. If fully complete, retire
   `sase bead epic-symbols sase-1mk` entries, close it normally with the verification
   note and same artifact, run `just symvision`, and mark that plan done. It has no
   expected parent; verify that from the current bead record.
6. Stop at the first incomplete or ambiguous ancestor, record a concrete blocker note
   there, and report it. If a parent link has changed to a phase, verify and close only
   that phase normally, leaving its containing epic to its land agent. Never force a
   successful ancestor landing. If the tale itself has a host-created linked bead that a
   readiness check identifies, satisfy its local work and close it normally before the
   containing epic; no dependency on its future commit is required.

Finish through `/sase_final`, including every repository actually edited (the main
checkout and plans repo). Report the release evidence, verification outcome, closed
beads, and any ancestor intentionally left open. Never claim a close or pass that the
commands did not confirm.
