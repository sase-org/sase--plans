---
create_time: 2026-07-13 07:05:19
status: done
prompt: 202607/prompts/finish_sase_5v_epic.md
tier: tale
---
# Finish the sase-5v epic: pyvendor doc references + bead closure

## Context

The `sase-5v` epic ("Factor pyvendor + bugyi.sh into basher and migrate sase + chezmoi", plan:
`sase/repos/plans/202607/basher_extraction.md`) was audited end-to-end:

- **Phases 1–5 (basher repo) — verified complete.** `~/projects/github/bbugyi200/basher` contains the full planned CLI
  (vendor/lib/update/status/cat/path/export, layered config, dry-run, legacy provenance parsing), `just check` passes
  right now (56 tests, 95.86% coverage, 95% gate), CI + Publish workflows are green, v0.2.0 is released on GitHub and
  PyPI (wheel + sdist).
- **Phase 6 (sase repo) — mostly complete.** Commit `d643af684` removed `lib/bugyi-260221.sh`; `tools/pyscripts-260619`
  was correctly left vendored with its legacy provenance line. However, **step 3 (memory-file doc updates) was skipped**
  because the phase agent lacked the required explicit user permission, and the follow-up was never flagged on the bead.
- **Phase 7 (chezmoi) — verified complete.** Commit `835e8f19` added `run_onchange_install_basher.tmpl` (monthly
  `uv tool install --upgrade basher` + `basher export ~/lib`), deleted `home/lib/bugyi.sh` / `executable_pyvendor` /
  both bash test suites, and updated the two referencing docs. On this machine `~/lib/bugyi.sh` carries the basher
  v0.2.0 provenance line and `BUGYI_VERSION=0.2.0`, and `source ~/lib/bugyi.sh && log::info smoke` works.

## Remaining work

The only unfinished epic work is the Phase 6, step 3–4 documentation sweep inside the sase repo. All target files are
memory files, so **editing them requires explicit user permission granted in the current conversation** (a
`sase_questions` permission request must precede the edits; if permission is denied, stop and leave the bead open with
the follow-up recorded on the bead).

1. **Update `tools/AGENTS.md`** (first paragraph only):
   - Remove the parenthetical claiming `../lib/bugyi-260221.sh` was vendored via `pyvendor` (that file was deleted by
     commit `d643af684`).
   - Replace the pyvendor re-vendoring instruction: vendored `tools/*-YYmmdd` scripts are now vendored / refreshed with
     `basher vendor` / `basher update` from the `basher` PyPI package instead of `pyvendor`.
2. **Regenerate the provider shims** `tools/CLAUDE.md`, `tools/GEMINI.md`, `tools/OPENCODE.md`, `tools/QWEN.md` — they
   are byte-identical copies of `tools/AGENTS.md`, so copy the updated file over each of them.
3. **Update `memory/pyvision.md`**: in the "Never edit the vendored tool" paragraph, replace "re-vendor with `pyvendor`"
   with the basher equivalent (`basher vendor` / `basher update`). No other changes to this file (its broader
   pyvision→symvision staleness belongs to the sase-5t epic, not this one).
4. **Verify**: `just install` (ephemeral workspace) followed by full `just check`.
5. **Close the epic bead**: `sase bead close sase-5v` (all seven children are already closed).

## Post-close wrap-up (mandated by the launching user, outside the epic plan itself)

- `just pyvision` no longer exists (pyvision was retired by sase-5t); run its successor `just symvision` after the bead
  is closed to confirm no unused code was left behind by the epic.
- Update the epic plan file `sase/repos/plans/202607/basher_extraction.md` frontmatter `status` field from `wip` to
  `done`.

## Out of scope (flag only)

- `tools/AGENTS.md` / `memory/pyvision.md` still document the retired `pyvision-260708` tool (sase-5t leftovers) —
  report to the user; do not fix under this epic.
