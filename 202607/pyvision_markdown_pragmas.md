---
create_time: 2026-07-08 22:14:14
bead_id: sase-5n
tier: epic
status: done
prompt: 202607/prompts/pyvision_markdown_pragmas.md
---
# Plan: Replace Markdown pyvision Pragmas With Real Visibility Boundaries

## Diagnosis

`actstat --repo sase-org/sase --limit 2` showed the latest settled SASE CI failures in the `lint` job, specifically the
`Lint` step. The failed GitHub Actions log for run `28983280362`, job `86006664127`, failed during `just lint` at:

```bash
BD_COMMAND=tools/sase_bead .venv/bin/python tools/pyvision-260708 src/sase
```

The reported failures were `# pyvision:` pragmas pointing at missing markdown files:

- `src/sase/project_aliases.py`: `sdd/tales/202606/project_name_field.md`
- `src/sase/project_display_names.py`: `sdd/tales/202607/telegram_project_display_names_1.md`
- `src/sase/main/qa_prompt.py`: `sdd/research/202606/with_q_and_a_xprompt_consolidated.md`
- `src/sase/xprompt/vcs_repo_completion.py`: `sdd/epics/202607/vcs_repo_slash_completion.md`
- `src/sase/plugins/installed.py`: `sdd/tales/202607/onboarding_install_plugins_step.md`
- `src/sase/ace/tui/task_queue.py`: `sdd/tales/202606/tasks_tab_live_output.md`
- `src/sase/ace/tui/task_subprocess.py`: `sdd/tales/202606/tasks_tab_live_output.md`
- `src/sase/ace/tui/widgets/xprompt_inline_expansion.py`: `sdd/epics/202606/xprompt_expand_keymap.md`

Current `master` has a later local vendored pyvision patch that resolves `sdd/...` references under `.sase/sdd/...`,
which gets past this exact missing-file symptom. That is not the desired end state. The deeper bug is that pyvision
accepts markdown design/research docs as evidence that a public Python symbol is externally required.

Running `just _lint-pyvision` at current `master` gets past the markdown pragma validation and exposes another pyvision
class of failures: private helpers/types imported or re-exported from non-test modules, mostly in the family attach
split modules and the legacy `multi_agent_xprompt` compatibility wrapper. Those should be fixed with real visibility
boundaries, not pragmas.

## Target Invariants

- Local `# pyvision: <path>` pragmas may point at code or configuration files only. Markdown paths such as `.md` and
  `.markdown` must be rejected by pyvision with a clear error.
- URI pragmas such as `https://github.com/sase-org/sase-core.git` remain valid, but only when pyvision can verify the
  consuming repo actually references the symbol.
- No local markdown `# pyvision:` pragmas remain in `src/sase`.
- Public symbols that are only used in their defining file and tests are made private or deleted with their tests
  updated.
- Private symbols that need to cross a non-test module boundary are either renamed to public names or moved so they no
  longer cross that boundary as underscore-prefixed symbols.
- The pyvision script is changed in the chezmoi source repo first and then re-vendored into this repo with `pyvendor`;
  do not hand-edit the vendored pyvision copy as the source of truth.

## Phase 1: Baseline And Inventory

Owner goal: produce a precise inventory of code symbols affected by the markdown pragma policy and the current
private-import pyvision failures.

Steps:

1. Run `actstat --repo sase-org/sase --limit 2` and record the current failing settled run, if any.
2. Run `just install` if the workspace has not already been prepared, then run `just _lint-pyvision`.
3. Inventory all local markdown pragmas:
   ```bash
   rg -n "# pyvision: .*\\.m(arkdown|d)\\b" src/sase
   ```
4. Inventory non-markdown pragmas separately, especially external repository URI pragmas:
   ```bash
   rg -n "# pyvision:" src/sase
   ```
5. For each markdown-pragma symbol, classify it as:
   - used by non-test Python code in this repo,
   - used only by tests and its defining file,
   - used by a real external repo and therefore needing a URI pragma,
   - dead code to delete.

Deliverable: a short notes artifact or implementation comment in the phase handoff listing every markdown-pragma symbol
and the chosen action.

## Phase 2: Harden pyvision In Chezmoi

Owner goal: update the source `pyvision` script so markdown paths can no longer satisfy pragma validation.

Steps:

1. Edit the pyvision source in the chezmoi repo, not the vendored SASE copy:
   - source path: `~/.local/share/chezmoi/home/bin/executable_pyvision`
2. In local pragma validation, reject markdown references before existence/content checks. Treat at least `.md` and
   `.markdown` as invalid.
3. Keep external repository URI pragmas working.
4. Do not preserve the SASE vendored-only `.sase/sdd` fallback as a way to validate markdown pragmas. If there is still
   a legitimate non-markdown SDD/config use case, implement it explicitly and test it; otherwise remove that behavior
   when re-vendoring.
5. Add or update chezmoi pyvision tests, likely in `~/.local/share/chezmoi/tests/bash/pyvision_test.sh`, covering:
   - `.md` local pragma target is rejected,
   - `.markdown` local pragma target is rejected,
   - a non-markdown config/code target still works when it references the symbol,
   - URI pragma behavior remains unchanged.
6. Run the targeted chezmoi test script for pyvision.

Do not run `git commit` directly. If a commit is required by the workflow, use the SASE commit flow for the chezmoi
repo.

## Phase 3: Remove Markdown Pragmas In SASE Code

Owner goal: remove all markdown local pyvision pragmas from `src/sase` and replace them with real code cleanup.

Known markdown pragma sites to audit:

- `src/sase/integrations/agent_status_groups.py`
- `src/sase/integrations/changespec_tags.py`
- `src/sase/notifications/pending_actions.py`
- `src/sase/project_aliases.py`
- `src/sase/project_display_names.py`
- `src/sase/main/qa_prompt.py`
- `src/sase/plugins/installed.py`
- `src/sase/ace/tui/task_queue.py`
- `src/sase/ace/tui/task_subprocess.py`
- `src/sase/xprompt/vcs_repo_completion.py`
- `src/sase/ace/tui/widgets/xprompt_inline_expansion.py`

Decision rules:

- If a symbol is only referenced by tests plus its defining file, make it private and update tests to import the private
  name only when the test is intentionally unit-testing the helper.
- If a symbol is no longer used by product code and tests only preserve the old surface, delete the symbol and delete or
  rewrite those tests around the public behavior.
- If a symbol is consumed by `sase-core`, `sase-telegram`, or another real external repo, use a URI pragma and verify
  pyvision can find an actual reference in the consuming repo.
- If a symbol is used by non-test code in this repo, remove the stale pragma; pyvision should consider it used without
  help.

Likely cleanup candidates based on initial search:

- `allocate_project_name`, `set_project_name_locked`, and `ensure_project_name_locked` look test-heavy and should be
  audited for private conversion or deletion.
- `humanize_safe_stem` may be an external integration surface; verify against `sase-telegram` before deciding between a
  URI pragma and private conversion.
- `qa_rounds_from_payload`, `installed_plugin_distributions`, `VcsRepoCompletionConfig`, `capture_output`,
  `stream_subprocess`, `InlineExpansionReason`, and `InlineExpansionResult` appear to need careful public-vs-private
  review because several are currently used mainly by tests or same-file call paths.
- Existing `docs/integrations.md` pragmas must also be removed or replaced; markdown docs are not valid consumers under
  the new rule.

Validation for this phase:

```bash
rg -n "# pyvision: .*\\.m(arkdown|d)\\b" src/sase
just _lint-pyvision
```

Expect `just _lint-pyvision` may still fail on private-import issues until Phase 4 is complete.

## Phase 4: Fix Private Cross-Module Boundaries

Owner goal: make pyvision's private-symbol failures reflect real module boundaries instead of underscore-prefixed
compatibility exports.

Known areas from the current local `just _lint-pyvision` failure:

- `src/sase/agent/family_attach.py` re-exports many underscore-prefixed names from `_family_attach_*` modules and
  includes them in `__all__`.
- `src/sase/agent/multi_agent_xprompt.py` re-exports underscore-prefixed aliases from `xprompt_swarm.py` for legacy
  compatibility.
- Some reported duplicate helper names, such as `_str_or_none` and `_int_or_none`, are triggered by one cross-module
  underscore access and then reported for every same-named private definition. Fix the cross-module access first before
  broad renames.
- `src/sase/doctor/checks_resources.py` exposes underscore helpers from split resource check modules for monkeypatching;
  replace with public adapter names or update tests to patch the proper public seam.

Decision rules:

- If a helper/type must be imported from another non-test module, rename it to a public name in the implementation
  module and update all imports/aliases.
- If a helper/type is only needed by tests, keep it private and update tests to import the private module directly.
- If a compatibility wrapper exists only for old private names, remove those private aliases unless a real non-test
  caller still needs them.
- Avoid introducing runtime-specific behavior; keep shared agent runtime logic uniform.

Validation for this phase:

```bash
just _lint-pyvision
```

It should pass before moving to final validation.

## Phase 5: Re-Vendor pyvision Into SASE

Owner goal: replace the vendored SASE pyvision script with the updated chezmoi version using `pyvendor`.

Steps:

1. From the SASE repo root, run:
   ```bash
   pyvendor ~/.local/share/chezmoi/home/bin/executable_pyvision .
   ```
2. Confirm the old `tools/pyvision-260708` was replaced by a new date-stamped `tools/pyvision-YYMMDD`.
3. Confirm `Justfile` references the new vendored filename.
4. Review pyvendor's automatic reference updates. It may update filename references in `tools/AGENTS.md` and generated
   provider shims; do not make unrelated policy text edits to those files unless explicitly approved.
5. Confirm the vendored script matches the chezmoi source apart from pyvendor's provenance comment and expected
   vendoring transformations.

Validation:

```bash
just _lint-pyvision
```

## Phase 6: Full Repo Validation

Owner goal: prove the combined changes fix CI rather than only the isolated pyvision stage.

Steps:

1. Run `just install` if needed.
2. Run:
   ```bash
   just check
   ```
3. If `just check` fails on unrelated pre-existing failures, capture the failing command and a concise explanation
   before handing off.
4. Re-run:
   ```bash
   actstat --repo sase-org/sase --limit 2
   ```
   after changes are pushed/CI starts, if the workflow has reached a settled state.

Final acceptance criteria:

- `rg -n "# pyvision: .*\\.m(arkdown|d)\\b" src/sase` returns no source-code markdown pragmas.
- The vendored SASE pyvision rejects markdown local pragma targets.
- `just _lint-pyvision` passes.
- `just check` passes, or any remaining failure is documented as unrelated with evidence.
