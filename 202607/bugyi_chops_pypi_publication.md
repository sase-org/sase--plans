---
create_time: 2026-07-18 23:10:25
status: wip
tier: tale
---
# Publish bugyi-chops to PyPI

## Context

The sase-6v epic created the `bbugyi200/bugyi-chops` package (personal chop scripts as a SASE plugin) with a working
test suite, green CI on master, and a tag-triggered `Publish` workflow. The actual first publication to PyPI was
deliberately deferred and is still outstanding, for two reasons:

1. The package declares `sase>=0.12.0`, but the newest sase release on PyPI is 0.11.1. Until sase 0.12.0 is published,
   an uploaded bugyi-chops wheel would be uninstallable from PyPI.
2. The publish workflow uses PyPI trusted publishing through a `pypi` GitHub environment. Both the environment and the
   PyPI trusted-publisher registration for `bbugyi200/bugyi-chops` are one-time manual setup that only Bryan can do.

## Plan

1. Wait for / confirm the blockers are cleared: sase 0.12.0 available on PyPI, and the `pypi` GitHub environment plus
   PyPI trusted-publisher registration created for the repo.
2. Push the first release tag (e.g. `v0.1.0`) on `bbugyi200/bugyi-chops` master.
3. Verify the `Publish` workflow run succeeds end to end and the package appears on PyPI.
4. Smoke-test a fresh install: `uv pip install bugyi-chops` in a clean venv, then confirm the chop scripts register with
   `sase` (e.g. via the chop doctor) on a machine using the PyPI package instead of the source checkout.

## Expected Outcome

`bugyi-chops` is installable from PyPI and the chezmoi-managed hosts can consume it as a normal plugin dependency
rather than a local checkout.
