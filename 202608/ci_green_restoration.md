---
tier: epic
title: Restore green CI on master
goal: "A master CI run finishes with every job green: the published-core minimum smoke passes, `just validate` passes on
  a clean CI host, the bead ANSI snapshot matches in every environment, and all three test-matrix legs finish inside
  their timeout.

  "
phases:
  - id: core-floor
    title: Raise the published sase-core-rs floor to 0.17.8
    depends_on: []
    size: small
    description: "core-floor: bump the pyproject dependency window so the published-core smoke lane installs a
      sase-core-rs release that actually exposes every binding this repo calls, and refresh the recorded lock specifier
      to match.

      "
  - id: bead-color
    title: Make bead prose highlighting ignore ambient NO_COLOR
    depends_on: []
    size: small
    description: "bead-color: force color on the internal rendering console so `--color always` wins over the ambient
      NO_COLOR environment variable, restore the highlighted ANSI golden that was overwritten with degraded output, and
      add a regression test that renders under NO_COLOR.

      "
  - id: ci-budget
    title: Fit the test matrix inside its job timeout
    depends_on: []
    size: small
    description: "ci-budget: raise the test job timeout and stop running coverage on the matrix legs that never upload
      it, so the slowest interpreter leg can finish instead of being cancelled at the limit.

      "
  - id: validate-skip
    title: Skip the prompt-archive check when its context is unavailable
    depends_on: []
    size: medium
    description: "validate-skip: teach the prompt-archive validation an explicit unavailable-context outcome and have
      `sase validate` report it as skipped rather than failed, so a clean host without a project registry or
      agents-sidecar clone can still run the aggregate validation.

      "
  - id: publish-migration
    title: Publish the plans-sidecar prompt migration
    depends_on: []
    size: medium
    description: "publish-migration: finish and publish the historical prompt migration so the plans sidecar remote no
      longer carries prompt Markdown or dangling relative prompt links, coordinating with the in-progress phase bead
      that already owns this work instead of redoing it.

      "
  - id: verify-green
    title: Confirm a fully green master run
    depends_on:
      - core-floor
      - bead-color
      - ci-budget
      - validate-skip
      - publish-migration
    size: small
    description:
      "verify-green: watch a full master run after the other phases land, confirm every job is green, and re-tune the
      test timeout if the slowest leg still lands close to the limit."
proposed_by: bbugyi200.athena.rm
create_time: 2026-08-02 06:45:37
status: wip
---

- **PROMPT:**
  [prompts/202608/ci_green_restoration.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/ci_green_restoration.md)

# Plan: Restore green CI on master

## Background

`master` CI is red. The most recent run is [30717806922](https://github.com/sase-org/sase/actions/runs/30717806922) on
commit `ddbe622a9`, and it fails in four independent ways. There is no shared root cause: packaging, cross-repo data
state, CLI environment assumptions, and a CI resource budget each broke separately.

Verified failure inventory for that run:

| Job                            | Failing step                                                 | Root cause                                  |
| ------------------------------ | ------------------------------------------------------------ | ------------------------------------------- |
| `published-core-minimum-smoke` | Check every required binding exists in the published minimum | Stale `sase-core-rs` floor                  |
| `lint`                         | SASE validation → `plan links validate`                      | Unpublished plans-sidecar prompt migration  |
| `lint`                         | SASE validation → `agent prompts validate`                   | Check cannot run without a project registry |
| `test (3.14)`                  | Run tests                                                    | ANSI golden captured under `NO_COLOR`       |
| `test (3.12)`, `test (3.13)`   | Run tests                                                    | Cancelled at the 60-minute job timeout      |

`test (3.12)` has never completed in the last 200 master runs (spanning 2026-07-30 through 2026-08-01); it was cancelled
at exactly 60 minutes in all six runs that reached it. So fixing only the test failure would still leave the matrix red.

Every diagnosis below was reproduced locally before this plan was written.

## Raise the published sase-core-rs floor to 0.17.8

`pyproject.toml` declares `sase-core-rs>=0.17.5,<0.18.0`. The `published-core-minimum-smoke` job derives the version to
install from that floor (`tools/smoke_sase_core_rs_telemetry --print-minimum pyproject.toml`), installs exactly that
release from PyPI with no source override, and then runs `tools/check_sase_core_rs_bindings`. That check now fails:

```
sase_core_rs 0.17.5 is missing 9 of 245 required binding(s):
  bead_needs_plus_one_evidence_migration
  bead_plus_one
  bead_plus_one_evidence_migration_sql
  prompt_artifact_manifest_parse
  prompt_artifact_manifest_render_record
  prompt_artifact_manifest_select
  prompt_artifact_pool_filename
  prompt_artifact_rewrite_links
  prompt_artifact_wire_schema_version
```

The bead plus-one and prompt-artifact work landed in this repo without moving the floor. Local dev never noticed because
`just install` builds `sase_core_rs` from the linked `sase-core` source checkout, which is already at `0.17.8`.

Binding coverage was measured against each published release by installing it into a throwaway venv and running
`tools/check_sase_core_rs_bindings`:

- `0.17.6` — missing the same 9 bindings.
- `0.17.7` — missing the same 9 bindings.
- `0.17.8` — `exposes all 245 bindings required by src/sase`.

`0.17.8` is the newest published release and the first one that satisfies the repo, so it is the correct floor.

Work:

1. Change the `sase-core-rs` requirement in `pyproject.toml` to `>=0.17.8,<0.18.0`.
2. Refresh `uv.lock` so the recorded specifier and the locked package version agree with the new floor. The lock
   currently records `specifier = ">=0.17.5,<0.18.0"` and resolves `sase-core-rs 0.17.7`.
3. Confirm `tools/validate_sase_core_rs_version --pyproject pyproject.toml --published-minimum` still passes.

Note for whoever implements this: `just _setup` already emits
`WARNING: bump the published sase-core-rs window in pyproject.toml` when the floor goes stale. That warning fired and
was not acted on. Consider whether the warning should be promoted to an error in a follow-up; do not change it as part
of this phase.

## Make bead prose highlighting ignore ambient NO_COLOR

`tests/test_bead/test_cli_show_style.py::test_show_closed_phase_with_markdown_rich_ansi_snapshot` fails against
`tests/test_bead/golden/cli/show_style_closed_phase.ansi`. The rendered output carries Pygments highlighting and
explicit default-background codes (`ESC[1;49m`); the committed golden is unhighlighted (`ESC[1m`, plain list bullets,
plain code fence).

This is not renderer drift. `src/sase/bead/cli_detail_prose.py` builds its own `Console` in `_render_text` with
`force_terminal=True` and `color_system="256"`, but does not pass `no_color=False`. Rich still honours the ambient
`NO_COLOR` environment variable, which drops every foreground and background attribute while keeping bold. That is
exactly the shape of the committed golden.

Reproduced directly:

```
$ NO_COLOR=1 python -c "...highlight_prose('# Summary\n\nFixed the bug:\n...')"
'\x1b[1m# Summary\x1b[0m\x1b[0m\n\x1b[0m\nFixed the bug:\x1b[0m\n\x1b[0m\n- root caused by X\x1b'
```

That is byte-for-byte the committed golden. Without `NO_COLOR`, and with `no_color=False` passed explicitly even when
`NO_COLOR` is set, the output is the highlighted `ESC[1;49m` form that CI produces.

Ruled out while diagnosing: `rich` version (14.3.3, 14.3.4 and 15.0.0 all produce the highlighted form), `TERM=dumb`,
and the Python version. `src/sase/bead/cli_detail_prose.py` has not changed since the feature commit that introduced it,
so no source change explains the divergence.

This is a real product bug, not only a test problem: `sase bead show --color always` is documented to force color, and
an explicit CLI flag must beat an environment variable. It is also self-concealing, because `highlight_prose` wraps its
whole body in `except Exception: return text` and silently falls back to unhighlighted output.

The history explains itself once the mechanism is known. Task bead `sase-df` reported this test failing with the
_opposite_ polarity — golden expecting `ESC[1;49m`, renderer emitting `ESC[1m`. Commit `c1efe9f93` then "fixed" it by
rewriting the golden to the degraded bytes, which is what CI now rejects. Both the report and the bad fix came from an
environment with `NO_COLOR` set.

Work:

1. Pass `no_color=False` to the `Console` constructed in `_render_text` so `--color always` is authoritative.
2. Restore the golden to its pre-`c1efe9f93` contents (the highlighted `ESC[1;49m` form). Recovering it from that
   commit's parent is exact; do not hand-edit it.
3. Add a regression test that renders the rich style with `NO_COLOR=1` set in the environment and asserts the output
   still matches the golden. Without this, the same wrong golden can be committed again.
4. Review the other goldens `c1efe9f93` touched in `tests/test_bead/golden/cli/` for the same degradation. The JSON
   goldens in that commit are unaffected (those tests pass in CI), but confirm rather than assume.
5. Close `sase-df` as resolved by this phase, referencing the `NO_COLOR` mechanism.

Verified: restoring the pre-`c1efe9f93` golden makes all 38 tests in `tests/test_bead/test_cli_show_style.py` pass, and
takes the full local suite to zero failures (25356 passed, 7 skipped).

Consider, but do not necessarily do here: narrowing the blanket `except Exception` in `highlight_prose` so a
highlighting failure is observable in tests instead of silently degrading. Raise it as a follow-up if it grows beyond a
small change.

## Fit the test matrix inside its job timeout

The `test` job sets `timeout-minutes: 60`. Measured job durations on recent master runs:

- `test (3.14)` — 33 to 45 minutes, completes.
- `test (3.13)` — 50 to 58 minutes, completes some runs, cancelled at 60 in the latest.
- `test (3.12)` — cancelled at exactly 60 minutes in every run that reached it.

The `test (3.12)` log ends with `##[error]The operation was canceled.` one hour after start, with no pytest summary.
There is no successful `test (3.12)` job in the last 200 master runs. This is a resource budget problem, not a test
failure: the suite is ~25k tests, the runners have 4 vCPUs, and 3.12 is the slowest supported interpreter. For scale,
the same suite runs in 3.5 minutes locally without coverage on a much wider machine.

The matrix also wastes work. All three legs run `just test-cov`, but the workflow already comments that "Coverage is
collected from the 3.12 leg" and only uploads coverage when `matrix.python-version == '3.12'`. The 3.13 and 3.14 legs
pay the coverage tax and discard the result.

Work:

1. Raise `timeout-minutes` for the `test` job to 90.
2. Run `just test-cov` only on the coverage leg (3.12) and `just test` on the other legs. Both recipes select the same
   tests through `tools/run_pytest`; only coverage instrumentation and the 50% gate differ.
3. Leave `fail-fast: false` alone — the existing comment explains why it is set, and that reasoning still holds.

Sizing caveat for the implementer: 90 minutes is a headroom estimate, not a measurement. The true 3.12 runtime is
unknown because the leg has never finished. The `verify-green` phase must read the actual 3.12 duration from the first
complete run and raise the timeout again if the margin is thin. If 3.12 turns out to need substantially more than 90
minutes, prefer sharding the suite across parallel jobs over an ever-growing timeout, and say so rather than raising it
a third time.

## Skip the prompt-archive check when its context is unavailable

`just validate` runs `sase validate`, which runs five checks as subprocesses. On CI, `agent prompts validate` fails:

```
  fail   agent prompts validate

agent prompts validate failed (exit 1)
stderr:
Error: project 'sase' was not found
```

`src/sase/agents/cli_prompts.py::_resolve_context` needs a registered SASE project and a cloned agents sidecar. A clean
CI host has neither. There is no project registry under the runner's home directory, and `tools/ci_bootstrap_sidecars`
deliberately skips the hidden `agents` sidecar — its module docstring and `HIDDEN_SIDECAR_ROLES` both state that the
store record schema cannot represent that role and that `sase init repo --check` only warns when it is missing.

Reproduced by running the check with an empty home directory, which yields the sibling error
`no project with an available agents sidecar was found`.

So the check is not detecting a defect; it cannot run at all in that environment. `sase validate` already takes this
exact position for a different check — the comment above `_CHECKS` in `src/sase/main/validate_handler.py` explains that
machine identity is intentionally local and may be absent on a clean CI host — but the prompt-archive check was added
without the same accommodation.

Work:

1. Give the prompt-archive validation an explicit unavailable-context outcome, distinct from a validation failure.
   `_resolve_context` currently collapses three different conditions into `ValueError` plus exit 1: unresolvable
   project, no project with an available agents sidecar, and a resolvable project whose sidecar checkout is not a
   directory. All three mean "cannot run here"; none mean "the archive is invalid".
2. Have `sase validate` render that outcome as `skip` rather than `fail`, and not count it toward the aggregate exit
   code. Keep the reason visible in the output so a skip on a developer machine, where the context should exist, is
   still noticeable.
3. Keep genuine archive validation failures failing. A resolvable context with invalid contents must still exit
   non-zero.
4. Update `tests/main/test_validate_handler.py`, which asserts on the rendered `ok` / `fail` lines and on
   `agent prompts validate failed (exit 4)`, and add coverage for the new skip path.

Deliberately not chosen: cloning the agents sidecar in CI. It is a private sidecar that CI has no credentials for by
design, and the bootstrap tool documents the omission as intentional. Making a hidden, optional sidecar into a CI
requirement would be a larger policy change than restoring green CI warrants.

## Publish the plans-sidecar prompt migration

`sase validate` also fails its `plan links validate` check, with 5766 errors on CI and 5764 locally, in two codes:

- `prompt-in-plans-store` (2893 locally) — prompt Markdown still present under `<month>/prompts/` in the plans sidecar.
- `link-missing-target` (2871 locally) — plan files whose `PROMPT` bullet still points at a relative `prompts/<name>.md`
  that is no longer a link-graph node.

Both codes are the same unfinished migration seen from two sides. The prompt archive moved to the agents sidecar;
`src/sase/sdd/_legacy_prompt_files.py` documents that plans-sidecar prompts are migration inputs rather than live nodes,
so every unmigrated prompt produces one error of each kind.

The migration is not unwritten — it is unpublished. State as observed:

- The agents sidecar is fully published: `sase agent prompts validate` passes against it with 2894 archived prompts, and
  it has no unpushed commits.
- The plans sidecar has six completed local migration commits (`Migrate 202603..202608 prompts to agents sidecar`) that
  were never pushed. `sase--plans` `origin/main` still carries all 2893 prompt files.
- After migration, plan links become absolute hosted URLs, which `src/sase/sdd/_link_support.py::resolve_link_path`
  skips, so the `link-missing-target` errors resolve themselves once the migration is published.
- Two prompts created after the migration ran still need migrating: `202608/prompts/finish_dh.md` and
  `202608/prompts/land_sase_dr.md`. `sase agent prompts migrate` reports exactly these two.

Verified: validating the already-migrated checkout reports only 4 errors, all from those two stragglers. Publishing the
migration and migrating the stragglers therefore takes this check to zero errors.

Work:

1. Coordinate with phase bead `sase-dh.8.3` ("Publish and validate the historical migration", in progress under epic
   `sase-dh.8`), whose description is exactly this task: recover and publish the six completed plans-sidecar migration
   commits on top of current remote work. Do not redo that work in parallel. If it has already published by the time
   this phase starts, verify and record that instead.
2. Migrate the two remaining `202608` prompts with `sase agent prompts migrate --write`.
3. Publish the plans sidecar so `origin/main` carries the migration.
4. Prove the result from the remote tip, not from a local checkout: re-clone or hard-refresh the plans sidecar and
   confirm `sase plan links validate` reports zero errors against it.

Two things the implementer must surface to the project owner before publishing, rather than deciding alone:

- Publishing pushes to `sase--plans` `main`, which is outward-facing and not reversible by a local reset.
- The unpushed range also contains two commits unrelated to the migration (`Archive approved plan finish_dh` and
  `Link approved epic plan to its bead: finish_dh`). They look like ordinary SDD plan commits, but they ship with the
  push and should be acknowledged rather than swept along silently.

Follow-up worth filing, but out of scope here: the SDD store is configured to push after commit asynchronously
(`push_after_commit: async`), and six migration commits sat unpushed with nothing surfacing that. Whatever silently
dropped those pushes is a separate reliability bug, and it is the actual reason CI broke. File it through the task-bead
workflow rather than fixing it in this epic.

## Confirm a fully green master run

The preceding phases were each verified against a local reproduction, but three of them can only be proven on CI: the
published-core smoke installs from PyPI, the SASE validation runs on a host with no project registry, and the
test-matrix timing depends on runner hardware.

Work:

1. After the other phases land, watch a complete master run and confirm every job is green: `build-core`,
   `published-core-minimum-smoke`, `lint`, `visual-test`, `perf-floors`, and all three `test` legs.
2. Read the actual `test (3.12)` duration from that run. If it lands within ten minutes of the 90-minute timeout, raise
   the timeout or propose sharding rather than leaving CI one slow run away from red again.
3. If `lint` still fails, distinguish which of its two checks failed — `plan links validate` points back at the
   migration publish, `agent prompts validate` at the skip semantics.
4. Report the final run URL and per-job durations so the timing baseline is recorded somewhere durable.
