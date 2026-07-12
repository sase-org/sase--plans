---
create_time: 2026-07-09 10:53:56
status: done
prompt: 202607/prompts/github_actions_publish_fix.md
tier: tale
---
# Fix sase-github Publish Workflow Failures

## Context

`actstat` reports `sase-org/sase-github` failing on the `Publish` workflow for commit `c76814d` in run `28999834965`.
The failed `release` job never reached a runner: GitHub marked it cancelled with the annotation that the hosted runner
was not acquired after multiple attempts. That latest failure is an infrastructure retry case, not a code or
workflow-command failure.

There is also an earlier failed `Publish` run, `28886444321`, on the `chore(master): release 0.1.7 (#7)` release commit.
That run reached the workflow commands and failed in the `install-smoke` job while installing the built `sase-github`
wheel. The wheel declares `sase>=0.11.0`, but the smoke job first installs SASE from the checked-out `master` source
tree, whose package version is still `0.10.2`. The second install command then resolves dependencies against PyPI and
the installed environment, finds no installable `sase>=0.11.0`, and rejects the wheel as unsatisfiable.

The dependency bump commit already solved this for local development by using a uv overrides file in the `Justfile` when
`SASE_CORE_PATH` points at an editable SASE checkout. The publish workflow did not receive the same override, so release
smoke tests do not match local validation.

## Plan

1. Patch `.github/workflows/publish.yml` so the release smoke install resolves the `sase` dependency through the
   checked-out SASE source tree using a uv overrides file, mirroring the local `Justfile` behavior.
2. Keep the published wheel metadata unchanged. The package should still declare `sase>=0.11.0`; the override is only
   for validating against the source checkout used by CI before the corresponding SASE package version is available on
   PyPI.
3. Simplify the smoke install flow if appropriate so the wheel install is the operation that pulls in SASE via the
   override, avoiding a misleading pre-install of `sase==0.10.2`.
4. Validate locally with:
   - `just check`
   - `uv build`
   - a temporary fresh-venv smoke install that uses the same override pattern as the workflow and checks the expected
     entry points.
5. Re-run the latest failed `Publish` workflow after the workflow fix is in place or, if no commit has been pushed yet,
   document that run `28999834965` is retryable infrastructure noise. Monitor with `actstat` or `gh run view`.

## Expected Outcome

Future release publishes should pass the install smoke step even while SASE source `master` carries a package version
lower than the next published dependency floor. The current latest failure should clear on rerun because it was caused
by GitHub hosted runner acquisition, while the real release smoke failure is addressed by the workflow change.
