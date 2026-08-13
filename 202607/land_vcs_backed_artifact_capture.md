---
tier: tale
title: Finish landing VCS-backed artifact capture
goal:
  Every artifact-file reader tolerates a byte-free row, the regenerated `sase_artifact_file` skill is deployed so `just
  check` is green again, and epic bead sase-b7 is closed with its plan file marked done.
bead: sase-b7
create_time: 2026-07-30 10:51:56
status: wip
---

- **PROMPT:** [prompts/202607/land_vcs_backed_artifact_capture.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/land_vcs_backed_artifact_capture.md)
- **PARENT:** [202607/vcs_backed_artifact_capture.md](https://github.com/sase-org/sase--plans/blob/main/202607/vcs_backed_artifact_capture.md)
- **BEAD:** [sase-b7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-b7/README.md)

# Plan: Finish landing VCS-backed artifact capture

Epic `sase-b7` ("Make artifact capture mean authorship and stop copying what version control stores") shipped all five
of its phases. Landing verification confirmed the substance is real: the Rust core admits byte-free rows and
materializes them with digest verification (`sase-core` `ee287b0`, released as 0.13.0), the capture policy implements
the decision matrix with its fail-safe invariants, finalization routes candidates through it, and the CLI, prompt
`@`-expansion, and Files pane all materialize on demand. `just fmt`, `just lint` (ruff, mypy, symvision, toobig), and
the 143 artifact-focused tests all pass on `master`.

Three things are still outstanding, and they are what this plan finishes.

1. The deployed skill was never regenerated, so `just check` fails for everyone.
2. Two ACE surfaces were never taught that `ArtifactFile.path` is now optional. One raises `TypeError`; the other
   silently drops rows.
3. The epic bead is still open and its plan file still says `status: wip`.

## 1. Deploy the regenerated `sase_artifact_file` skill

`just check` currently fails at the SASE validation stage:

```
fail   init skills --check
  run  init skills  overwrite 5 provider skill files
       ~ overwrite  .../dot_claude/skills/sase_artifact_file/SKILL.md  +27  claude/sase_artifact_file
       (and agy, codex, opencode, qwen)
```

The `+27` lines are exactly the "VCS-Backed Artifacts" section that phase `sase-b7.5` added to
`src/sase/xprompts/skills/sase_artifact_file.md`. That phase could not deploy, because a chezmoi deploy is refused from
an uncommitted tree; the source is now committed and `HEAD` is an ancestor of the canonical branch, so the deploy can
run.

Read the `generated_skills` long-term memory first (`sase memory read generated_skills.md -r "..."`) and follow exactly
what it prescribes: preview with `sase skill init --diff`, then deploy from the clean, merged tree. Use `--force` only
if the provenance-manifest guard refuses and the recorded source commit is an ancestor of `HEAD` — that is the
documented escape hatch for a stale destination, not a way to make the command succeed. Do not hand-edit any generated
`SKILL.md`.

Done when `sase validate` no longer reports `init skills --check`. The two remaining `plan links validate` errors
(`202607/family_scoped_agent_provenance.md` and `202607/prompts/artifact_consumption_ledger.md`) belong to other plans,
predate this epic, and are explicitly out of scope — leave them alone and say so in the closing note.

## 2. Teach the remaining ACE surfaces about byte-free rows

The epic changed `ArtifactFile.path` from `str` to `str | None` and updated the read surfaces its plan enumerated: the
`artifact` CLI, prompt `@file:` expansion, and the Files pane detail/actions. Two surfaces outside that list still
assume a stored path. Both are only reachable once a VCS-backed row exists, which is why the phase test suites did not
catch them — the live index still has zero such rows (`sase artifact doctor` reports `VCS reference rows 0`), so these
are latent, not yet observed.

### 2a. The Copy-as palette raises `TypeError` on a VCS-backed row

`src/sase/ace/tui/actions/clipboard/_artifact_targets.py:455`, in `_copy_marked_file_targets`:

```python
view_mode = artifact_file_view_mode(entry.path, kind=entry.kind)
```

`artifact_file_view_mode()` starts with `Path(path).suffix`, so a `None` path raises
`TypeError: argument should be a str or an os.PathLike object ... not 'NoneType'` before any `kind` branch runs
(verified directly against the installed build). Marking a VCS-backed artifact file and copying contents therefore
crashes the action rather than degrading.

The single-entry path in the same file already does the right thing: it reads `pane.selected_view_mode`, which is served
from `FilesSnapshot.view_mode_for()` — a per-row classification computed once at load time and already made VCS-aware in
`files_data.py:59` (`row.path or row.vcs_relpath or ""`). Make the marked path use the same snapshot accessor instead of
reclassifying from `entry.path`. That removes the crash and removes a duplicate classification rule at the same time.

### 2b. The Copy-as palette and picker modal never materialize VCS-backed content

Still in `_artifact_targets.py`, and in `src/sase/ace/tui/modals/artifact_files_modal_copying.py` via
`artifact_files_modal_rendering.py`, every path/contents target reads the row through the duck-typed helpers in
`src/sase/ace/tui/models/artifact_file_clipboard.py`:

- `artifact_file_clipboard_path()` → `artifact_file_preferred_path_text()` reads `getattr(row, "path", "") or ""`, so a
  VCS-backed row silently falls back to `source_path` — the workspace path the file had at capture time. Numbered
  workspaces are cleaned and reset on open, so that path is frequently dead, and the user gets a stale path reported as
  a successful copy.
- `artifact_file_resolved_stored_path()` returns `None`, so `_artifact_file_contents()` raises "`<label>` has no stored
  path" and `artifact_file_stored_clipboard_path()` reports the row as having no path.

Meanwhile the Files-pane verbs in `src/sase/ace/tui/actions/artifacts_files.py` materialize first (see
`_materialize_artifact_file_entry`, and the `Y` handler at line 339). The same user-facing verb therefore behaves
differently depending on which surface invoked it.

Route these through `materialize_artifact_file()` the way `artifacts_files.py` already does, so a VCS-backed row yields
its cache path and a materialization failure surfaces as a copy failure rather than a stale path.

Threading is already safe for this: `_schedule_artifacts_copy()`/`_schedule_marked_copy()` hand the value callable to
`deliver_copy()`, which resolves it with `asyncio.to_thread()` and turns an exception into a notification
(`src/sase/ace/tui/actions/clipboard/_delivery.py:98`). Do the git I/O inside those closures, never on the UI thread —
read the `tui_perf` long-term memory before touching this path. Keep `_materialize_artifact_file_entries()` (or a shared
extraction of it) as the single Python entry point; do not add a second materialization call site.

### 2c. The agent prompt panel silently drops VCS-backed rows

`src/sase/ace/tui/widgets/prompt_panel/_artifact_files.py:105` and `:120`:

```python
for artifact_file in artifact_files
if artifact_file.path and artifact_file.kind not in {"chat", "pdf"}
```

Before this epic every persisted media row had a stored path, so the filter was a no-op. Now every tracked-media capture
becomes a byte-free row, so the agent header's `ARTIFACTS` list silently loses exactly the files it exists to surface —
the visual-snapshot goldens, demo GIFs, and docs images the research measured as 97.8% of automatic captures.

Keep VCS-backed rows in the list. The display already prefers `source_path` for non-explicit rows, which is the
workspace-relative label users recognize, so a byte-free row has a good display string without materializing anything.
Decide deliberately what `actual_path` (the hint-jump target) should be and what `exists` should report for a row whose
bytes are not on disk — this list renders synchronously in the header, so it must not materialize inline. Prefer a
label-only entry over blocking the pump.

### 2d. Parity and hardening (small, same blast radius)

- `src/sase/ace/tui/widgets/artifacts/files_filtering.py:285` builds its text haystack from
  `(row.label, row.path, row.source_path)`. The Rust query needle was taught to match `vcs_relpath` in this epic
  (`artifact_file.rs`, `query_artifact_files_at`), so `sase artifact list -q foo.png` finds a VCS-backed row that the
  ACE Files filter cannot. Add `vcs_relpath` to the haystack.
- `artifact_file_dedupe_key()` in `src/sase/core/artifact_file_helpers.py:133` delegates to `_vcs_identity()`, which
  raises `ValueError` when a byte-free row is missing any of repo/relpath/`sha256`. `dedupe_artifact_files()` sits on
  the display path for `list_artifact_files()`, so one malformed index row would take down the Files pane and
  `sase artifact` listing instead of being skipped. `artifact_file_doctor.py` already has a
  `vcs_provenance_incomplete_ids` bucket for exactly this shape, so make the dedupe key fall back (to the row id) rather
  than raise.

### Tests

Add coverage that fails before the fix and passes after, for each of: the marked copy-contents path over a VCS-backed
row (2a), a copy-as path/contents target that materializes (2b), the prompt-panel `ARTIFACTS` list retaining a byte-free
row (2c), the Files-pane filter matching on `vcs_relpath`, and `dedupe_artifact_files()` tolerating a byte-free row with
incomplete provenance. Follow the existing fixtures: `tests/artifact_file_facade/test_vcs.py` builds a temp git repo and
a byte-free row, and `tests/test_artifact_file_e2e.py` shows the end-to-end shape.

## 3. Land the epic

Do this last, after `just install` and a green `just check` (`init skills --check` clean; the two unrelated
`plan links validate` errors above are expected to remain).

1. `sase bead close sase-b7 --note "<what was verified and finished>"`. The note should record that all five phases were
   verified against the source and the epic's commits, that the deployed skill was regenerated, and which read-surface
   gaps this plan closed. Do not `--force`: every phase is already closed, so a rejection would mean something genuinely
   changed and should be investigated, not overridden.
2. Run `just symvision` after the close — epic-symbol whitelist entries expire at close — and remove whatever stale
   entries or unused code it reports. Note that phase `sase-b7.4` already removed the epic's `--epic-symbol` entries
   from the `Justfile`, and symvision passes today, so no findings are expected; read the `symvision` long-term memory
   before acting on any that appear.
3. Set `status: done` in the frontmatter of `@plan:202607/vcs_backed_artifact_capture.md` (the epic's `PLAN` path, in
   the plans sidecar — reach it with `/sase_repo`).

## Out of scope

- The two pre-existing `plan links validate` errors for `family_scoped_agent_provenance` and
  `artifact_consumption_ledger`. They belong to other plans and predate this epic.
- Everything epic `sase-b7`'s own plan listed as out of scope, notably recommendation 1 (`sase artifact create`
  hardcoding `move=True` at `src/sase/artifact_cli/create.py:36`) and recommendation 3 (retention plus the retroactive
  migration of the 4,296 rows already in the store). Both remain worth doing; neither is this plan.
- Rendering `vcs_repo` through `repo_display_name()` (added by `63d0ca504`). For the primary repo — the source of
  essentially every VCS-backed row — `slug` is unset and the display name equals the identity, so there is nothing to
  fix in practice.
