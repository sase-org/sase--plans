---
create_time: 2026-07-12 18:52:19
status: done
prompt: 202607/prompts/sase_5t_closeout.md
tier: tale
---
# Close out epic sase-5t (pyvision → symvision)

## Context

Epic `sase-5t` ("Factor pyvision into symvision and migrate sase + chezmoi", plan
`sase/repos/plans/202607/symvision_extraction_1.md`) is nearly complete. Verification already performed:

- **Phases 1–3 (closed, verified)**: `~/projects/github/bbugyi200/symvision/` exists with commits `34bf0fa` (port),
  `d02a139` (test suite, fail_under=94), `9c8d1e3` (CI/release). Source is split into focused modules
  (cli/scanner/discovery/external/pragmas/models/git/api), epic-symbol + `BD_COMMAND` support present, CI green on
  master.
- **Phase 4 (`sase-5t.4`, in_progress)**: was blocked on the PyPI trusted-publisher user checkpoint. The publisher has
  since been registered: rerunning the failed jobs of Publish run `29211186259` now **succeeds**. Independently
  verified: PyPI JSON API reports `symvision 0.1.0`; a clean `uv tool install symvision` yields a working CLI. → All
  bead-note follow-ups are addressed; the bead can be closed.
- **Phase 5 (`sase-5t.5`, in_progress)**: migration commit `39451d036` verified (Justfile `_lint-symvision`/`symvision`
  recipes with the preserved `sase-5u(get_max_running_agents)` epic allowance, `symvision>=0.1.0,<0.2.0` dev dep,
  `tools/pyvision-260708` deleted, pragmas/docs/tests renamed; `grep -rn pyvision` outside memory files is clean). The
  only outstanding note: rerun **registry-based** `just install` (previously unsatisfiable while PyPI 404'd) plus
  `just symvision` and `just check`.
- **Phase 6 (`sase-5t.6`, closed, verified)**: chezmoi commit `6ac1be8f` deletes `home/bin/executable_pyvision` and
  `tests/bash/pyvision_test.sh`; no live pyvision references remain in chezmoi outside archived plans.

**Out of scope (requires explicit user permission not granted):** the Phase-5 step-5 memory-file updates —
`memory/pyvision.md` (rewrite for symvision), its Tier 2 listing in `AGENTS.md`/provider shims + `memory/README.md`, and
the `pyvision-260708` section in `tools/AGENTS.md`. These remain flagged as a follow-up for the user.

## Remaining steps

1. **Close `sase-5t.4`** with a note recording: publisher registered, Publish run `29211186259` rerun succeeded, PyPI
   reports symvision 0.1.0, clean `uv tool install symvision` verified.
2. **Registry-based verification in the sase workspace**: run `just install` (must now resolve `symvision` 0.1.x from
   PyPI), then `just symvision` (expect a clean run against `src/sase`), then `just check`. Judge any `just check`
   failures against known pre-existing issues (init-memory freshness gate, sandbox exit-144 test kills, llm_provider
   `default_effort` failures) before treating them as regressions; fall back to static gates + targeted pytest subsets
   if the full suite is killed by the sandbox.
3. **Close `sase-5t.5`** with a note recording the registry-based verification results.
4. **Close epic `sase-5t`** via `sase bead close`.
5. **Post-close lint sweep**: run `just symvision` (the renamed `just pyvision`) after the epic is closed to confirm no
   unused code was left behind (the preserved `sase-5u(...)` allowance belongs to a different, still-open epic and is
   unaffected).
6. **Mark the epic plan done**: update the frontmatter of `sase/repos/plans/202607/symvision_extraction_1.md`, setting
   `status: wip` → `status: done`.
7. **Report the memory-file follow-up** to the user (not performed — no permission granted in this conversation).
