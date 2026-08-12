---
tier: tale
title: Finish and land the artifact reference contract epic
goal:
  The three defects epic sase-js caused are fixed - sase.artifact_providers imports
  cleanly from a cold interpreter, the retired Artifacts Chats pane and its keymaps,
  commands, copy targets, and help-modal rows are deleted, and @chat is absent from
  artifact-ref completion - and the epic is then closed with its follow-ups filed, its
  symvision whitelist retired, and its plan file marked done.
size: medium
proposed_by: bbugyi200.athena.sase-js.land
bead: sase-js
create_time: 2026-08-12 10:34:23
status: done
---

- **PROMPT:**
  [prompts/202608/land_artifact_ref_contract.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/land_artifact_ref_contract.md)
- **PARENT:**
  [202608/artifact_ref_contract.md](https://github.com/sase-org/sase--plans/blob/main/202608/artifact_ref_contract.md)
- **BEAD:**
  [sase-js](https://github.com/sase-org/sase--beads/blob/main/pages/sase-js/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-js.land](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-js.land.md)
- **COMMITS:**
  - [ad11756](https://github.com/sase-org/sase/commit/ad11756e6ed919d83f30c69fdb82d3c37c59b955)
    — fix(ace): break artifact-provider import cycle, delete retired Chats pane, drop
    @chat completion

# Plan: Finish and land the artifact reference contract epic (sase-js)

## 1. Why

Epic `sase-js` closed all nine phases, and its feature work is genuinely delivered: the
ref contract wire types, the provider registry, the four builtin entry providers, the
`@file` object store, reference-style prompt links with `Referenced By` write-back, the
dynamic Artifacts sub-tabs, the `sase-research` plugin, and the adoption docs are all in
the tree and working. Landing verification found three defects the epic itself caused,
all of which must be fixed before the epic can close.

1. **`sase.artifact_providers` cannot be imported first.** Importing it (or
   `sase.artifact_providers.registry`) before any other `sase` submodule raises
   `ImportError: cannot import name 'ArtifactProviderRegistry' from partially initialized module 'sase.artifact_providers.registry'`.
   This is not theoretical: it is why `import sase_research` fails in the installed
   `sase` tool environment, and `sase-research`'s own `tests/conftest.py` carries a
   workaround import for it. Phase `sase-js.3` introduced the cycle; phase `sase-js.8`
   reported it.

2. **The retired Artifacts Chats pane was unmounted but never deleted.** The epic plan
   said to delete `chats_*`. The pane is gone from `resolve_artifacts_subtabs()`, but
   its nine widget modules and its actions mixin survive, wired into keymaps, the
   command palette, the copy-target registry, and the help modal. The help modal still
   renders a whole `Copy Mode · Chats` section whose first row is `Copy @chat reference`
   — advertising a pane that cannot be reached and a ref kind this same epic retired
   from live authoring. Two test modules were marked `pytest.mark.skip` instead of
   removed.

3. **`@chat` still has payload completion in ACE.** The epic requires `@chat` to be
   "absent from completion". The kind stage is correct — `kind_inventory()` builds from
   `completion_artifact_ref_kinds()`, which omits `chat` — but the payload stage still
   builds a `chat` index and the completion catalog eagerly scans chat transcript files
   on every catalog build, which also costs the completion hot path for a retired kind.

## 2. Verified current state

Read at the tip of `master` (`56d6bd772 docs: adopt artifact reference provider docs`).

**The import cycle is exact and small.** `sase/artifact_providers/__init__.py` imports
`.registry`; `registry.py:14` does `from sase.config.core import current_config_token`,
which executes `sase/config/__init__.py`; that file at line 63 imports
`sase.config.file_hooks`; and `file_hooks.py:12` does
`from sase.artifact_providers.registry import ArtifactProviderRegistry`, which is still
mid-initialization. The only use of that name in `file_hooks.py` is the
`registry: ArtifactProviderRegistry` parameter annotation of
`_resolve_file_hook_provider` at line 271, and the module already has
`from __future__ import annotations`, so the annotation never evaluates at runtime.

**The Chats pane is genuinely unreachable.** `ArtifactsView._compose_pane` in
`src/sase/ace/tui/widgets/artifacts/view.py` composes only `patches`, `stitches`,
`beads`, `files`, and provider document panes. `_chats_pane()` in
`src/sase/ace/tui/actions/artifacts_chats.py:43` does
`query_one("#artifacts-chats-pane", ...)` inside a `try/except` and now always returns
`None`, so every `action_chats_*` handler is a silent no-op.
`tests/ace/tui/test_artifacts_chats_loading.py` has a module-level
`pytestmark = pytest.mark.skip(reason="Artifacts Chats is no longer a mounted pane")`
and two tests in `tests/ace/tui/test_artifacts_chats_filtering.py` carry the same skip.

**Chat transcripts themselves are not going away.** `sase.history.chat_catalog`,
`chat_catalog_provenance/`, `chat_storage`, `chat_resume`, `chat_fork`, `chat_links`,
`chat_extras`, `sase.main.chat_handler`, and `sase.integrations.chat_install` all stay:
they back the `sase chat` CLI, agent detail, chat resume/fork, and archive rendering of
historical `@chat:` strings. Only the Artifacts _pane_ and its actions are retired.
`LEGACY_ARTIFACTS_SUBTABS["chats"]` in `src/sase/ace/tui/artifact_tabs.py` must also
stay: it is the fallback that keeps a persisted `chats` sub-tab selection from erroring.

**`@chat` completion splits across two stages.**
`src/sase/ace/tui/widgets/_artifact_ref_completion_menu.py:251-267` (`kind_inventory`)
already omits `chat`, because `completion_artifact_ref_kinds()` returns
`('stitch', 'patch', 'bead', 'agent', 'file')`. Line 322-338 (`payload_indexes`)
separately hardcodes `"chat"` into its kind list and skips only `{"commit", "bug"}`, and
`payload_rows` has a `folded == "chat"` arm at line 418.
`_artifact_ref_completion_catalog.py:76-83` calls `load_chat_candidate_catalog()`
unconditionally when building the catalog.

**Two landing observations that are deliberately not work here.** The stale
`sase-core-rs>=0.24.0,<0.25.0` floor in `pyproject.toml` is by design: `just install`,
`just symvision`, and `tools/probe_core_floor` all print that no action is needed
because `tools/ratchet_core_window` moves the published window on the release branch,
and a dry run confirms it resolves cleanly to `>=0.26.2,<0.27.0`. And the flake-baseline
gate failures every phase hit are already tracked on `sase-jq` and `sase-iu`.

## 3. Phases

Implement in this order. Phases 3.1-3.3 are code; 3.4 is the landing sequence and must
run last.

### 3.1 Break the artifact-provider import cycle

In `src/sase/config/file_hooks.py`, move
`from sase.artifact_providers.registry import ArtifactProviderRegistry` (line 12) under
a `if TYPE_CHECKING:` block. The module has no other runtime use of the name and already
carries `from __future__ import annotations`, so the `_resolve_file_hook_provider`
annotation keeps working and mypy still checks it.

Then confirm no sibling edge recreates the cycle: `src/sase/config/__init__.py` also
imports `sase.config.artifact_ref_files`, so check whether that module reaches
`sase.artifact_providers` at runtime and apply the same treatment if it does.

Add a regression test that runs in a **fresh interpreter** — an in-process `import` is
useless once `sase.config` is already in `sys.modules`. Use
`subprocess.run([sys.executable, "-c", "import sase.artifact_providers"])` and assert a
zero exit, plus the same for `sase.artifact_providers.registry` and
`sase.config.file_hooks`. Put it somewhere obvious, e.g.
`tests/artifact_providers/test_import_order.py`.

Verify by hand before moving on:

```bash
.venv/bin/python -c "import sase.artifact_providers"
.venv/bin/python -c "from sase.artifact_providers.registry import ArtifactProviderRegistry"
```

Both must exit 0. Today both raise `ImportError`.

### 3.2 Delete the retired Artifacts Chats pane

Delete these modules outright:

```text
src/sase/ace/tui/widgets/artifacts/chat_filter_bar.py
src/sase/ace/tui/widgets/artifacts/chats_data.py
src/sase/ace/tui/widgets/artifacts/chats_detail.py
src/sase/ace/tui/widgets/artifacts/chats_filter_session.py
src/sase/ace/tui/widgets/artifacts/chats_filtering.py
src/sase/ace/tui/widgets/artifacts/chats_list.py
src/sase/ace/tui/widgets/artifacts/chats_navigation.py
src/sase/ace/tui/widgets/artifacts/chats_pane.py
src/sase/ace/tui/widgets/artifacts/chats_rendering.py
src/sase/ace/tui/actions/artifacts_chats.py
```

Then unwire them. Each of these is a known call site, found by grepping the pane's
symbols; re-grep after editing to catch anything this list misses:

- `src/sase/ace/tui/widgets/__init__.py:27,123` and
  `src/sase/ace/tui/widgets/artifacts/__init__.py:18,56` — the lazy `ArtifactsChatsPane`
  export entries. Both packages have `.pyi` stubs; keep them in sync.
- `src/sase/ace/tui/actions/artifacts.py:18,41,183,347` — the
  `ArtifactsChatsActionsMixin` base and the `CHATS_ARTIFACT_ACTIONS` splat and
  re-export.
- `src/sase/ace/tui/_app_action_availability.py:83,150` — the `CHATS_ARTIFACT_ACTIONS`
  availability arm.
- `src/sase/ace/tui/bindings.py:151-159` — the nine `Binding(...)` rows.
- `src/sase/ace/tui/keymaps/app_keymaps.py:78-86`,
  `src/sase/ace/tui/keymaps/metadata.py:98-106`, and
  `src/sase/ace/tui/keymaps/mode_keymaps.py:101` — the keymap dataclass fields, the
  display metadata rows, and the `artifacts_chats` copy-mode block.
- `src/sase/default_config.yml:322-330` (the `# Chats sub-tab` keymap block) and the
  `artifacts_chats:` copy-mode block near line 524. Per the repo's default-keymap rule,
  this file must not keep bindings for actions that no longer exist.
- `src/sase/ace/tui/commands/_app_metadata.py:196-212`,
  `src/sase/ace/tui/commands/availability.py:149-157`, and
  `src/sase/ace/tui/commands/_mode_commands.py:165` — the command-palette entries.
- `src/sase/ace/tui/copy_targets.py:361-424` — the eight `artifacts_chats` copy targets.
- `src/sase/ace/tui/actions/clipboard/_palette_registry.py:65` — the `artifacts_chats`
  palette entry.
- `src/sase/ace/tui/actions/clipboard/_artifact_target_selected.py` —
  `_copy_chat_target` (line 147 onward) and its `read_full_chat` import.
- `src/sase/ace/tui/actions/clipboard/_artifact_target_marked.py:16,151` — the
  `chat_row_target` import and `_copy_marked_chat_targets`.
- `src/sase/ace/tui/modals/help_modal/patches_copy_bindings.py:16` and the whole
  `Copy Mode · Chats` section (~lines 158-194). This is the user-visible half of the
  bug.
- `src/sase/ace/tui/styles.tcss:196-206` — the `ChatFilterBar` rules.
- `src/sase/ace/tui/artifact_tabs.py` — drop the `"chats"` entry from
  `ARTIFACTS_ACCENTS`. **Keep** `LEGACY_ARTIFACTS_SUBTABS["chats"]`.

Delete the corresponding tests and goldens:

```text
tests/ace/tui/_artifacts_chats_helpers.py
tests/ace/tui/test_artifacts_chats_agent_link.py
tests/ace/tui/test_artifacts_chats_detail.py
tests/ace/tui/test_artifacts_chats_filtering.py
tests/ace/tui/test_artifacts_chats_loading.py
tests/ace/tui/test_artifacts_chats_rendering.py
tests/ace/tui/visual/test_ace_png_snapshots_artifacts_chats.py
tests/ace/tui/visual/test_ace_png_snapshots_artifacts_chats_empty.py
tests/ace/tui/visual/snapshots/png/artifacts_chats_empty_120x40.png
tests/ace/tui/visual/snapshots/png/artifacts_chats_populated_120x40.png
```

And surgically update the four test modules that reference the pane incidentally:
`tests/ace/tui/_artifacts_copy_helpers.py`, `tests/ace/tui/_copy_as_palette_helpers.py`,
`tests/ace/tui/test_artifacts_copy_marked.py`,
`tests/ace/tui/test_artifacts_scaffold.py`, plus
`tests/test_agent_artifact_marker_path_passing_audit.py` and
`tests/test_command_availability_changespecs.py`.

Do **not** touch `src/sase/history/chat_*`, `src/sase/main/chat_handler.py`, or
`src/sase/integrations/chat_install.py` — those back the `sase chat` CLI and agent
detail and are unrelated to the pane.

Run `just test-visual` afterwards. The Artifacts sub-tab strip loses no tab here (the
pane was already unmounted), so goldens for other panes should not move; if any do,
inspect `.pytest_cache/sase-visual/` before accepting with
`--sase-update-visual-snapshots`.

### 3.3 Drop `@chat` from artifact-ref payload completion

In `src/sase/ace/tui/widgets/_artifact_ref_completion_menu.py`, remove `"chat"` from the
hardcoded kind list at line 322-330 and fold it into the same exclusion the `commit` and
`bug` kinds already get at line 336, then delete the `folded == "chat"` payload arm at
line 418-430. In `_artifact_ref_completion_catalog.py`, stop calling
`load_chat_candidate_catalog()` when building the catalog and remove the now-unused
loader, the `max_chat_scan_rows` / `max_chat_rows` parameters, the `chats` catalog
field, and the `("chat", truncated)` truncation row. Drop the `chat` hint in
`_artifact_ref_completion_context.py:85-86`, the `chat` member of
`ArtifactRefPayloadSource` in `_artifact_ref_completion_models.py`, and the `"chat"`
badge in `_prompt_input_bar_completion_rows_artifacts.py:28`.

Keep parsing untouched: `parsable_artifact_ref_kinds()` must still return `chat` and
`bug` so archived prompts keep rendering. Add or extend a test asserting that neither
the kind stage nor the payload stage offers `chat`, alongside the existing coverage that
`commit` and `bug` are already excluded.

### 3.4 Land the epic

Run this only after 3.1-3.3 are committed and `just check-full` is green.

1. **File the deferred follow-ups.** Each of these was recorded by a phase agent as
   `PROPOSED FOLLOW-UP:` and is unbuilt scope rather than a defect this epic caused, so
   each goes through `/sase_new_task` (which will corroborate a duplicate, attach it to
   a causally related active epic, or create a sized task). Name the proposing bead in
   each one:
   - `sase-js.4` — migrate `@stitch` / `@patch` entry resolution from Python into
     `sase-core`, replacing the `unknown_kind` placeholder arms, so the LSP and future
     frontends share one resolver.
   - `sase-js.4` — extend the artifact-ref use wire to schema 2 with the epic's §3.7
     field set (`span`, `project`, `provider`, `origin`, `properties`,
     `captured_revision`, `captured_digest`); schema 1 only persists a subset today.
   - `sase-js.4` — add a cheap sha-to-Patch/stitch-number mapping so `@stitch` entries
     can carry the `patch` and `stitch_number` properties the epic's §3.4 lists.
   - `sase-js.5` — surface `@file` capture targets and sidecar visibility in the
     launch-approval preview, which needs a host-side resolve-only pass because the
     preview is built before the agent process starts.
   - `sase-js.6` — provider-declared `Referenced By` columns; the epic's §3.8 wants
     columns from the provider spec, but the spec wire has no column vocabulary yet, so
     this needs a wire change plus a coordinated `sase-research` release.
   - `sase-js.6` — publish `agents/<agent>/ref-uses.json` through the v2 publication
     payload contract; today the manifest is read locally and staged into referenced-by
     outbox rows.
   - `sase-js.9` — `sase repo open plans` fails applying commit `b10820e6` ("Add SDD
     files for split_patch_handler") with a non-bead conflict in
     `202608/split_patch_handler.md`, though the worktree is clean afterwards.

   Two other proposals are **not** filed, and the close note must say why:
   - `sase-js.2`'s six reproducible flakes (`tests/test_contract_manifest.py`,
     `tests/test_core_vcs_log.py`) are already tracked on `sase-jq` and `sase-iu`,
     corroborated there by phases 4, 5, and 6.
   - `sase-js.9`'s "core dependency floor needs ratcheting" is by design. `just install`
     and `just symvision` both print that no action is needed because
     `tools/ratchet_core_window` moves the published window on the release branch at
     release time, and a `--report-only` run confirms it resolves `0.24.0 -> 0.26.2`
     cleanly. Hand-editing `pyproject.toml` on `master` would fight release automation.

2. **Close the epic** with `sase bead close sase-js --note "<what was verified>"`. The
   note should record the per-phase verification, the integration review against the
   commits that landed since the epic started (`19f827332`, `63bb1a27e`, `62951abcb`,
   `09e5fc43e`, `35b469d81`, `46773f606` — the adoption phase is at `HEAD` and already
   reconciled both docs commits, and the xprompt properties band is a separate domain
   from ref provider properties), the three defects this plan fixed, and every follow-up
   outcome above. If the close is rejected, the named phases were never completed —
   finish or reopen them rather than forcing.

3. **Run `just symvision` after the close.** The epic-symbol whitelist entries expire at
   close, so five `--epic-symbol "sase-js(...)"` flags in the `Justfile`'s
   `_lint-symvision` recipe (lines 305-309) go stale: `ArtifactRefKindAlias`,
   `CanonicalArtifactRef`, `canonical_artifact_ref_kind`,
   `artifact_ref_expansion_validate`, and `ArtifactRefUseRecord`. Remove the flags, then
   remove whatever symvision reports as unused. Read `sase/memory/symvision.md` through
   `/sase_memory_read` first. Each of the five is either genuinely dead or missing its
   intended consumer — decide per symbol, and prefer wiring up the intended caller over
   deleting a contract the epic deliberately published.
   `artifact_ref_expansion_validate` in particular is exercised by
   `tests/artifact_providers/test_builtin_entry_stitch.py` and
   `test_builtin_entry_patch.py`, so it has real coverage even if `src/` has no caller.

4. **Mark the plan file done.** Set `status: done` in the frontmatter of
   `plans:202608/artifact_ref_contract.md` (the `PLAN` path from
   `sase bead show sase-js`), and set it in this plan's own archived file too.

## 4. Verification

- `.venv/bin/python -c "import sase.artifact_providers"` exits 0 from a cold
  interpreter, and the new subprocess regression test covers it.
- No `chats_*` action, keymap, binding, command, copy target, or help-modal row
  survives; `grep -rn 'artifacts_chats\|ArtifactsChatsPane\|chats_next' src/` is empty.
- No `pytest.mark.skip(reason="Artifacts Chats is no longer a mounted pane")` remains.
- ACE artifact-ref completion offers `chat` at neither the kind nor the payload stage,
  while `parsable_artifact_ref_kinds()` still includes `chat` and `bug`.
- `just check-full` passes every lint gate and the full test suite. This change touches
  keymaps and the ACE widget tree, so `check-full` is required rather than `just check`.
  The flake-baseline gate may still fail on the `sase-jq` / `sase-iu` flakes;
  corroborate rather than re-file, and do not treat it as a regression from this work.
- `just test-visual` passes, with any golden movement inspected before acceptance.

## 5. Risks

| Risk                                                                       | Mitigation                                                                                                                         |
| -------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| Deleting `chats_*` reaches into live `sase.history.chat_*` transcript code | The pane's modules are confined to `ace/tui/widgets/artifacts/` and one actions module; `sase chat` and agent detail are untouched |
| A persisted `current_artifacts_subtab: chats` starts erroring              | `LEGACY_ARTIFACTS_SUBTABS["chats"]` stays and keeps mapping to the default sub-tab                                                 |
| Removing a keymap field breaks the keymap contract or config schema        | Update `app_keymaps.py`, `metadata.py`, `mode_keymaps.py`, and `default_config.yml` together, then run the keymap tests            |
| The `TYPE_CHECKING` move hides a real runtime need for the registry type   | It is used only as an annotation and the module has `from __future__ import annotations`; mypy still type-checks it                |
| Symvision cleanup deletes a contract the epic deliberately published       | Decide per symbol after reading `sase/memory/symvision.md`; prefer wiring the intended consumer over deleting                      |

## 6. Out of scope

- Everything filed in 3.4 step 1 as a follow-up task.
- Ratcheting the `sase-core-rs` floor on `master`.
- The `sase-jq` / `sase-iu` flake investigation.
- Any further ACE sub-tab consolidation beyond removing the retired Chats pane.
