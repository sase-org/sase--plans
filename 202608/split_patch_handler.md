---
tier: tale
title: Split the patch command handler into focused modules
goal:
  Patch command code is organized by responsibility with every source file at or below
  500 lines and no CLI behavior regressions.
size: medium
proposed_by: bbugyi200.athena.toobig-2f.split_file.src.sase.main.patch_handler.0
create_time: 2026-08-11 11:49:02
status: wip
---

# Split the patch command handler into focused modules

Refactor `src/sase/main/patch_handler.py`, currently about 750 lines, into cohesive
command modules while preserving CLI behavior, compatibility imports, and test seams.
Keep every resulting Python source file at or below 500 lines.

## Implementation

1. Keep `sase.main.patch_handler` as the command-dispatch facade. Preserve
   `handle_patch_command`, the currently exported private handlers, and compatibility
   globals used by tests or callers. Its wrapper functions should delegate without
   changing exit codes, stdout/stderr text, JSON payloads, or the legacy
   `sase changespec` facade.
2. Move current-checkout resolution and rendering into a focused current-patch module.
   Keep branch/PR matching, project scoping, diagnostics, and markdown/plain/JSON
   rendering together. Where facade-level dependency injection is needed to preserve
   monkeypatch compatibility, pass dependencies explicitly rather than mutating module
   globals.
3. Move artifact-reference resolution, list rendering, add/remove persistence, and
   reference-context lookup into a focused ref-command module. Preserve the existing
   `_handle_ref` and `_artifact_reference_context` facade entry points and their
   monkeypatch behavior.
4. Move `sync-deltas` and `sync-external` behavior into a focused sync-command module,
   including external-project selection and reporting. Preserve provider injection via
   the facade where existing tests patch it.
5. Move `set-origin` and `.gp`-to-`.sase` migration behavior into a focused maintenance
   module. Keep common command-label and file-location helpers in the narrowest shared
   location that avoids circular imports.
6. Update or add focused tests only where necessary to enforce facade compatibility and
   delegated dispatch. Avoid broad behavior changes during the structural refactor.

## Verification

- Run `just install` before repository checks because the workspace may be stale.
- Run targeted patch/current/ref/sync/set-origin tests during development.
- Run `just check` after all file changes, as required by the repository instructions.
- Use `wc -l` on `src/sase/main/patch_handler.py` and each new patch-handler module to
  confirm every file is at most 500 lines.
