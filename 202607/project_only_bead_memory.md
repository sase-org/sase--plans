---
tier: tale
title: Generate the Tier 2 bead memory note for SASE-managed project repos only
goal:
  sase/memory/sase_beads.md is generated and listed in Tier 2 only for SASE-managed project repositories, and one `sase
  memory init` pass retires the copy that was already generated under the home root so home AGENTS.md and its provider
  shims no longer reference it.
proposed_by: bbugyi200.athena.qr
create_time: 2026-07-31 17:47:48
status: wip
---

- **PROMPT:** [202607/prompts/project_only_bead_memory.md](prompts/project_only_bead_memory.md)

# Plan: Scope the generated bead memory note to SASE-managed project repos

## Background

Commit `d6a2cce1f` ("feat(memory): generate Tier 2 bead workflow note") packaged the shared bead guidance as a generated
long-term memory note and wired it into `sase memory init`. `sase memory init` initializes two roots per run
(`src/sase/main/init_memory_handler.py::_memory_root_plans` and `run_init_memory`):

- the project root, planned with `manage_memory=inputs.is_sase_managed`, `derive_project_title=inputs.is_sase_managed`,
  `include_project_agent_docs=True`;
- the home root (`~`, or `~/.local/share/chezmoi/home` when `use_chezmoi: true`), planned with the defaults
  (`manage_memory=True`, `derive_project_title=False`).

Both roots currently render `sase/memory/sase_beads.md`, so the note is also generated under the home root and listed in
Tier 2 of the home `AGENTS.md` plus its byte-identical provider shims (`CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`,
`QWEN.md`). That is wrong: bead workflow guidance is project work context, not global home context.

Desired end state:

- A SASE-managed project repository generates `sase/memory/sase_beads.md` and lists it in Tier 2 (unchanged behavior).
- The home root never generates it, never lists it in Tier 2, and never counts it in the home `sase/memory/README.md`.
- A home root that already holds the generated copy converges in a single `sase memory init` pass: the file is deleted,
  and `AGENTS.md`, the provider shims, and `sase/memory/README.md` are rendered in that same pass as if it were already
  gone (no intermediate state where `AGENTS.md` still references a file the same run deletes, and no "unreferenced
  memory file" blocker).

This matters on the live host: `~/.local/share/chezmoi/home/sase/memory/sase_beads.md` exists today and is
byte-identical to the packaged render, and `~/AGENTS.md` lists it in Tier 2.

## Key code map

- `src/sase/main/init_memory_handler.py` — `_memory_root_plans` (plan path) and `run_init_memory` (apply path) call the
  project and home roots.
- `src/sase/main/init_memory/root_planning.py` — `memory_root_context` / `plan_memory_root`; owns `_MemoryRootContext`,
  the legacy-tree migration deletes, the AMD sync plan call, and the unreferenced-file validation overlay.
- `src/sase/main/init_memory/root_application.py` — `initialize_memory_root`; writes expected files, applies shim plans,
  deletes migrated legacy files, then computes `unreferenced`.
- `src/sase/main/init_memory/root_rendering.py` — `render_generated_beads_memory_content`, `generated_long_notes`,
  `render_expected_memory_files`, `_render_memory_readme` / `_discover_memory_readme_notes`.
- `src/sase/amd/_memory.py` — `plan_amd_memory_sync`, `plan_minimal_agents_sync`, `_render_managed_agents`,
  `_long_memory_descriptions`, `_long_memory_description_updates`.
- `src/sase/amd/_template.py`, `src/sase/amd/templates/AGENTS.minimal.template.md` — minimal agent-instruction template.
- `src/sase/memory/inventory_reachability.py` — `unreferenced_memory_files_for_init`.
- `src/sase/main/init_memory/templates/memory-README.template.md`,
  `src/sase/main/init_memory/templates/memory-sase-beads.template.md` — packaged templates.

## Step 1 — Gate generation on the root kind

Add an explicit keyword argument rather than inferring the root kind from `project_name` or `derive_project_title`.

1. `src/sase/main/init_memory/root_planning.py`
   - Add `include_bead_memory: bool = False` to `memory_root_context` and `plan_memory_root`; document it as "generate
     the shared bead workflow note; SASE-managed project repositories only, never home roots".
   - Render `render_generated_beads_memory_content()` (and keep its blocker path) only when `include_bead_memory` is
     true. When it is false, pass `generated_long_notes={}` to `_amd_sync_plan`.
   - Forward the flag to `render_expected_memory_files`.
2. `src/sase/main/init_memory/root_application.py` — add the same keyword to `initialize_memory_root` and forward it to
   `memory_root_context`.
3. `src/sase/main/init_memory/root_rendering.py` — add `include_bead_memory: bool = False` to
   `render_expected_memory_files`. Only add the `sase/memory/sase_beads.md` `MemoryExpectedFile` and its README overlay
   entry when the flag is true. Keep the existing "render lazily when `generated_beads_content is None`" behavior, but
   only under the flag. Leave `render_generated_beads_memory_content` and `generated_long_notes` themselves unchanged.
4. `src/sase/main/init_memory_handler.py` — pass `include_bead_memory=True` on the two project-root calls
   (`_memory_root_plans` and `run_init_memory`); leave both home-root calls on the default. `manage_memory` already
   short-circuits `memory_root_context` for a repo without `is_sase_managed: true`, so no extra gate is needed there.
5. `src/sase/main/init_memory/project_deploy.py` — no change. Staging `sase/memory/sase_beads.md` stays correct because
   `deploy_to_project_repo` only runs for the project root and only stages it when `manage_memory` is true.

## Step 2 — Retire an already-generated copy at roots that no longer manage it

Without this step the goal is not reachable by code alone: `_render_managed_agents` and `_discover_memory_readme_notes`
discover notes from disk, so an existing home `sase/memory/sase_beads.md` would keep appearing in Tier 2 and in the home
README even after Step 1.

Ownership guard: treat the file as SASE-owned only when its bytes are exactly the current packaged render. A file that a
human wrote or edited is left untouched and keeps behaving as an ordinary long note (it stays discovered, described, and
listed in Tier 2). Never delete on a mismatch and never raise a blocker for one.

1. `root_planning.py`
   - Add a helper that returns the retired paths for a root: empty when `include_bead_memory` is true; otherwise
     `(memory_write_root(root) / "sase_beads.md",)` when that file exists and its text equals
     `render_generated_beads_memory_content()`'s output, else empty. Read failures return empty.
   - Add `retired_note_paths: tuple[Path, ...] = ()` to `_MemoryRootContext` and populate it (only when `manage_memory`
     is true — the `not manage_memory` early return keeps unmanaged repos untouched).
   - Thread the retired paths into the AMD sync plan, `render_expected_memory_files`, and the unreferenced check so the
     whole pass renders as if the file were already gone.
   - In `plan_memory_root`, emit one
     `MemoryFileChange(path=..., operation="delete", detail="remove retired generated memory note")` per retired path,
     keeping the existing legacy-migration delete changes separate and unchanged.
2. `root_application.py` — unlink the retired paths (tolerating `FileNotFoundError`) before `MemoryRootResult` is built,
   and include them in `deleted_paths`. This is what makes the home deletion reach `git add` in
   `_deploy_to_chezmoi`/`deploy_to_project_repo`. Keep `_delete_migrated_memory_paths` untouched; add a separate small
   helper so the legacy empty-directory pruning does not get entangled with note retirement.
3. `src/sase/amd/_memory.py` — add `excluded_note_paths: frozenset[str] = frozenset()` (canonical root-relative POSIX
   paths, e.g. `"sase/memory/sase_beads.md"`) to `plan_amd_memory_sync`. Filter discovered notes by
   `note.relative_path not in excluded_note_paths` in `_long_memory_descriptions`, `_long_memory_description_updates`,
   and `_render_managed_agents` — a single shared discovery helper is preferable to three copies of the filter.
   Excluding it from `_long_memory_description_updates` also prevents planning a frontmatter rewrite of a file the same
   run deletes.
4. `root_rendering.py` — add the same `excluded_note_paths` keyword to `render_expected_memory_files`,
   `_render_memory_readme`, and `_discover_memory_readme_notes`, and drop excluded notes before stats are computed so
   the README note list and the Statistics counts both ignore them.
5. `src/sase/memory/inventory_reachability.py` — add `ignored_paths: Iterable[Path] = ()` to
   `unreferenced_memory_files_for_init` and subtract the resolved paths from the discovered memory files, then thread
   the same keyword through `src/sase/main/init_memory/inventory.py::unreferenced_memory_files`. Pass the retired paths
   from both `plan_memory_root` and `initialize_memory_root` (in the apply path the files are already gone, but passing
   them keeps plan and apply symmetric). Do not change `build_memory_inventory` or any `sase memory list` behavior.

Convergence requirement to verify: with a retired copy present, one `plan_init_memory()` reports the delete plus the
`AGENTS.md`/shim/README rewrites and no blockers, and a second `plan_init_memory()` after applying reports no actions at
all for that root.

## Step 3 — Revert the minimal agent-instruction template's Tier 2 section

`plan_minimal_agents_sync` is reached only when `resolve_amd_h1_title` yields no title, which in `sase memory init`
happens only for a home root without a configured `amd_h1_title` (a managed project root always passes
`derive_project_title=True` and therefore always derives a title). Its Tier 2 block was fed exclusively by generated
long notes, so after Step 1 it can only ever render empty.

- `src/sase/amd/templates/AGENTS.minimal.template.md` — remove the `## Tier 2 (long-term) Memory` heading, its blurb,
  and `{{ tier2_entries }}`, restoring `# {{ title }}` + `{{ tier1_sections }}`.
- `src/sase/amd/_template.py` — set `_MINIMAL_TEMPLATE_VARS` back to `{"title", "tier1_sections"}`. Note
  `render_markdown_template` rejects unknown placeholders, so a user minimal template that still references
  `{{ tier2_entries }}` will now fail with a clear "unknown template placeholder" error; that is intended.
- `src/sase/amd/_memory.py` — drop the `generated_long_notes` parameter and the `render_memory_note_references(...)` /
  `tier2_entries` wiring from `plan_minimal_agents_sync`, and stop passing it from `root_planning._amd_sync_plan`.
- `docs/configuration.md` — restore the `amd_agents_minimal_template` row to require `title`, `tier1_sections`.

Do not remove `MemoryNote`/`render_memory_note_references` imports that are still used elsewhere in `_memory.py`; run
the symvision lint stage to catch anything left unused.

## Step 4 — Template and documentation wording

- `src/sase/main/init_memory/templates/memory-README.template.md` — this one template renders both project and home
  READMEs, so the wording must be true at both roots. Rewrite the generated-files bullet so `sase/memory/sase.md` is
  described as always generated, and `sase/memory/sase_beads.md` as generated only for SASE-managed project
  repositories. Update the `{% else %}` "No memory notes exist yet" branch to stop promising the bead note
  unconditionally.
- `docs/init.md` — update the two places that describe the note (the "fixed packaged asset and has no override key"
  sentence near the template-override paragraph, and the "provides shared bead workflow guidance and is listed in Tier 2
  of managed agent instructions" sentence). State that it is generated for SASE-managed project repositories only, never
  for home or chezmoi-home roots, and add one sentence documenting the retirement behavior from Step 2 (an unmodified
  previously generated copy at a root that no longer manages it is deleted on the next initialization pass; an edited
  copy is left alone).
- `docs/memory.md` — update the "Initialization generates both the short `sase/memory/sase.md` workspace note and the
  long `sase/memory/sase_beads.md` bead reference" sentence to reflect the project-only scope.

## Step 5 — Tests

Update the tests that encode the current (home-inclusive) contract, and add coverage for the new behavior. Test helpers
live in `tests/main/init_memory_handler_helpers.py` (`patch_standard_paths`, `run_handler`, `plan_memory`, `write`,
`short_note`, `long_note`).

Existing tests to update:

- `tests/main/test_init_memory_plan.py` — the home-root test must now assert `home_root/sase/memory/sase_beads.md` is
  **not** in the planned action paths; the project-root assertions stay.
- `tests/main/test_init_memory_formatting.py` — drop `home_root/sase/memory/sase_beads.md` from the generated-file lists
  (both the formatting-stability test and the prettier test), keeping the project entries.
- `tests/main/test_init_memory_agents_templates.py` — revert the minimal-template test to a custom template without a
  Tier 2 section, drop its assertion that the rendered file contains a bold `sase/memory/sase_beads.md` Tier 2 entry,
  and revert the invalid-minimal-template blocker message back to `template must contain {{ tier1_sections }}`. Keep the
  project-template override test's expectation that Tier 2 lists `sase/memory/sase_beads.md`.
- `tests/main/test_init_memory_handler_outputs.py` — verify which root each README assertion targets; the project README
  assertions (ordering, `- Total notes: 4`, `- Long notes: 2`) stay as-is.
- `tests/main/test_init_memory_commit.py`, `tests/main/test_init_memory_markdown_templates.py`,
  `tests/test_bead/test_cli_memory_asset.py` — project-scoped or template-scoped; expected to need no change, but re-run
  them.

New coverage (a dedicated `tests/main/test_init_memory_bead_note.py` keeps the existing modules under the toobig
thresholds):

1. Home root omits the note: after a successful `sase memory init` for a managed project plus a home root,
   `home_root/sase/memory/sase_beads.md` does not exist, the home `AGENTS.md` has no `sase/memory/sase_beads.md` Tier 2
   entry, and the home `sase/memory/README.md` does not list it — while the project root has all three.
2. Retirement converges in one pass: seed the home root with a byte-identical copy of
   `render_generated_beads_memory_content()` and a home `AGENTS.md` that references it, then assert the plan contains a
   `delete` action for that path with no blockers, that applying it removes the file and rewrites the home `AGENTS.md`
   and README without the entry, and that a follow-up `plan_memory()` reports `actions == ()`.
3. Retirement guard: seed a home copy whose bytes differ from the packaged render, then assert it is not deleted, is
   still listed in Tier 2, and produces no blockers.
4. No unreferenced-file blocker is reported for a retired path in the same pass that deletes it (assert on
   `plan.blockers` and on the handler exit code, which is 1 when unreferenced files are found).
5. `tests/test_memory_inventory.py` — unit coverage that `unreferenced_memory_files_for_init(..., ignored_paths=...)`
   drops the ignored path from its result and leaves other unreferenced files reported.

## Step 6 — Verification and live cleanup

Never hand-edit any file under `sase/memory/`, `AGENTS.md`, or a provider shim in this repo or under the home root:
every memory change in this plan must come from editing code/templates and then running the generator. The user
requested this change directly, so running `sase memory init` as the mandatory follow-through — including the home-root
retirement it performs — needs no further permission; hand-editing generated output would still be wrong.

```bash
just install          # ephemeral workspace: dependencies may be stale
just check            # fmt, ruff, mypy, symvision, toobig, sase validation, committed plans, tests
```

Then regenerate and confirm the live outcome from this workspace (the workspace is a SASE-managed project directory, so
the project root keeps the note while the home root retires it):

```bash
.venv/bin/sase memory init --no-commit --diff
.venv/bin/sase memory init --no-commit
```

`--no-commit` skips only the project commit path; home deployment still follows `use_chezmoi`, so the chezmoi source
deletion is staged, committed, and `chezmoi apply --force` runs. Verify afterwards:

- `sase/memory/sase_beads.md` still exists in this repo and is still listed in Tier 2 of the repo `AGENTS.md`;
  `sase/memory/README.md` reflects the new template wording.
- `~/.local/share/chezmoi/home/sase/memory/sase_beads.md` is gone, and `~/.local/share/chezmoi/home/AGENTS.md` plus its
  `CLAUDE.md` / `GEMINI.md` / `OPENCODE.md` / `QWEN.md` shims no longer mention `sase_beads`.
- `~/AGENTS.md` and `~/CLAUDE.md` no longer mention `sase_beads` after the apply.
- `~/sase/memory/sase_beads.md`: chezmoi does not remove a target whose source file disappeared, so this applied copy
  can survive as an orphan. Remove it with `rm` if it is still present, and confirm `~/sase/memory/README.md` no longer
  lists it.
- Re-run `.venv/bin/sase memory init --check` and confirm it reports no drift for either root.

If the chezmoi repository has unrelated uncommitted changes, do not force anything: `sase memory init` stages only the
paths it changed. Use the `/sase_repo` skill if the chezmoi checkout needs to be inspected or committed directly.

Finalize through the normal SASE commit workflow (do not commit unless the finalizer or the user asks).

## Out of scope

- Retiring the note in project repositories that are not SASE-managed. `manage_memory=False` short-circuits memory
  planning entirely, so those roots stay untouched by design.
- Changing the bead guidance content itself (`templates/memory-sase-beads.template.md`) or reviving the retired
  `sase_beads` xprompt skill removed in `642b4f490`.
- Any change to `sase memory list` / `build_memory_inventory` output.
