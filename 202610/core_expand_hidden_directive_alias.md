---
tier: tale
size: small
title: Keep the legacy directive contract byte-identical and land sase-1eq.1.1
goal:
  Accept %macros_enabled as a hidden input-only directive alias in sase-core so
  unchanged sase passes against the combined core, then close epic sase-1eq.1.1 and its
  parent phase sase-1eq.1.
proposed_by: bbugyi200.athena.sase-1eq.1.1.land
bead: sase-1eq.1.1
status: done
---

- **PARENT:**
  [202610/finish_core_macro_expand.md](https://github.com/sase-org/sase--plans/blob/main/202610/finish_core_macro_expand.md)
- **BEAD:**
  [sase-1eq.1.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1eq/sase-1eq.1.1.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1eq.1.1.land](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1eq.1.1.land.md)
- **COMMITS:**
  - [34151dc](https://github.com/sase-org/sase--plans/commit/34151dcf6741964e799f91da126e0fdd10016ff7)
    — chore(plans): mark finish_core_macro_expand epic plan done

# Hidden `%macros_enabled` alias, then land sase-1eq.1.1

This tale finishes the landing of epic **sase-1eq.1.1** ("Finish the additive Rust macro
rename and close sase-1eq.1"). The epic's land agent verified all seven closed phases.
It found one defect the epic caused that is not yet fixed. This tale fixes that defect
and then performs the whole epic closeout. Nothing resumes the landing after this tale,
so the closeout steps below are part of the work.

Read the epic and its plan with
`sase bead read sase-1eq.1.1 -r "Need the epic scope and landing note"` (the PLAN path
it prints is the epic plan file). Its latest note, `LAND TRIAGE`, records the evidence
and the follow-up outcomes summarized here.

## The defect

Phase sase-1eq.1.1.4 (sase-core commit `e6a3452f`) made `%macros_enabled` an input alias
of `%xprompts_enabled`. It did so by changing the `xprompts_enabled` row of `DIRECTIVES`
in `crates/sase_core/src/editor/directive/metadata.rs` from `alias: None` to
`alias: Some("macros_enabled")`. That field is emitted:

- `directive_contract()` / `directive_contract_with_flags()`
  (`editor/directive/contract.rs`) serialize it as the row's `alias`. The
  `xprompts_enabled` row now emits `"alias": "macros_enabled"` instead of `null`.
- `build_directive_completion_candidates_with_flags`
  (`editor/directive/candidate_lists.rs`) matches partial tokens against the alias and
  emits `detail: "alias %macros_enabled"`.
- The LSP directive recipe filter
  (`crates/sase_xprompt_lsp/src/server/completion_items.rs`) matches partials against
  the contract alias.

Unchanged sase consumes that contract in its ACE completion code. With the combined core
rebuilt into unchanged sase, typing `%m` now matches both `%model` and
`%xprompts_enabled`, so Ctrl-T no longer completes `%model`. These four sase tests fail
deterministically. Phase sase-1eq.1.1.7 misclassified them as pre-existing: they are
not, because the cause is the core, and a clean sase tree still reproduces them.

- `tests/test_xprompt_directive_contract.py::test_runtime_directive_vocabulary_matches_core_contract`
  (extra contract alias `macros_enabled -> xprompts_enabled`)
- `tests/ace/tui/widgets/test_directive_completion_candidates.py::test_directive_completion_matches_aliases_to_canonical_insertions`
- `tests/ace/tui/widgets/test_directive_completion_interactions.py::test_ctrl_t_at_alias_partial_inserts_canonical_directive`
- `tests/ace/tui/widgets/test_directive_completion_interactions.py::test_percent_partial_auto_opens_directive_panel`

This violates the epic's contract: emitted directive metadata stays on the legacy
spelling, serialized output stays byte-identical, and a sase tree on the previous pin
must still pass. The permitted additive outputs do not include a directive alias row.

## Fix (sase-core only)

Open the linked core with
`sase repo open sase-core -r "Fix hidden macros_enabled directive alias for sase-1eq.1.1 landing"`,
read the printed checkout's `AGENTS.md`, and work only in that printed path. Do not
change any tracked file in the primary sase checkout: the point is that **unchanged**
sase passes.

1. In `editor/directive/metadata.rs`, restore `alias: None` on the `xprompts_enabled`
   row. Add a small input-only alias table beside `HIDDEN_COMPLETION_DIRECTIVES`, for
   example
   `pub(super) const HIDDEN_DIRECTIVE_ALIASES: &[(&str, &str)] = &[("macros_enabled", "xprompts_enabled")];`.
   Give it a short doc comment: these aliases are accepted and canonicalized but never
   emitted in the directive contract or offered by name completion. Mark the
   `macros_enabled` entry with the plan's `// legacy xprompt spelling` annotation if
   that reads naturally beside the legacy target name.
2. In `editor/directive/contract.rs`, make `canonical_directive_name` fall back to that
   table after the `DIRECTIVES` search, so the `alt` special case and every visible name
   and alias keep precedence. `directive_metadata[_with_flags]`,
   `agent_launch/directive_scan.rs::canonical_directive_name`,
   `editor/directive/context.rs`, and `editor/argument_spans.rs` all route through it,
   so `%macros_enabled` keeps resolving to `xprompts_enabled` in launch scanning,
   argument/diagnostic parsing, and the LSP. Do not iterate the hidden table anywhere in
   completion or contract code.
3. Leave the literal-zone regexes in `agent_launch/directive_scan.rs` and the
   `"macros_enabled"` arm of `directive_metadata_supports_colon` in `editor/wire.rs`
   alone. They accept the new input or only answer a direct call with the new name, so
   neither changes legacy output.
4. Tests in `editor/directive/tests.rs`:
   - Update `macros_enabled_alias_canonicalizes_to_legacy`: `canonical_directive_name`
     and `directive_metadata` still resolve `macros_enabled` to `xprompts_enabled`, and
     the `xprompts_enabled` contract entry's `alias` is now `None`. No contract row may
     have `name` or `alias` equal to `macros_enabled`, with and without enabled feature
     flags.
   - Add a regression test that `build_directive_completion_candidates("%m")` yields
     only `%model`, and that no `%ma…` partial offers `%xprompts_enabled`. Both
     behaviors match the pre-epic core.
   - Keep `new_family_marker_canonicalizes_to_legacy` and the mixed-marker tests in
     `directive_scan.rs` passing unchanged.
   - Search the core, PyO3, and LSP tests for any other assertion that depends on the
     contract alias being `macros_enabled`, and fix it to the legacy expectation. The
     land agent found none outside `editor/directive/tests.rs`.
5. Iterate with `just fmt`, `just fast`, and `just test -p sase_core directive`. Then
   run the complete gate `sase tool run check` from the linked checkout and give it a
   tool timeout of at least 15 minutes. Never run bare cargo, and do not run
   `just check-full`.

## Prove unchanged sase passes

From the primary sase checkout, read `lint_and_test.md` through `/sase_memory_read`,
then run `just rust-dev-install` (about 3 minutes; give it a timeout of at least 30
minutes). It builds the linked core into the workspace venv. Confirm
`sase_core_rs.__file__` resolves into the linked checkout and that the
`xprompts_enabled` row of `sase_core_rs.directive_contract()` has `alias` `None`. Then
run, with `just test` or the venv's pytest:

- `tests/test_xprompt_directive_contract.py`
- `tests/test_xprompt_directive_completion_parity.py`
- `tests/ace/tui/widgets/test_directive_completion_candidates.py`
- `tests/ace/tui/widgets/test_directive_completion_interactions.py`
- `tests/test_content_layout.py`, `tests/test_core_health.py`,
  `tests/test_editor_helper_xprompt_catalog.py`, and
  `tests/test_editor_helper_snippet_catalog.py`, to recheck the epic's other unchanged
  sase surfaces

All four previously failing nodes must pass, and nothing else may regress. Do not edit
sase tests or expected data to get a pass. Record the core revision and dirty state and
the sase revision you tested. Also confirm `git status` in the primary sase checkout
shows no tracked change.
`tests/ace/tui/command_line/test_panel_shell_pilot.py::test_empty_panel_escape_follows_the_hide_panel_binding`
is a known parallel-lane flake, now tracked by task sase-1ew and not caused by this
epic; it is not part of this work.

Record the evidence (fix summary, ToolRun ids, test counts, revisions) with
`sase bead note sase-1eq.1.1 "..."`.

## Epic closeout (the final step; do it in this same turn)

1. Run `sase bead epic-symbols sase-1eq.1.1`. The land agent found no entries. For any
   entry that has since appeared, resolve the symbol (wire it up, privatize it, add a
   non-test pragma, or delete it per the Symvision epic-whitelist policy), or re-key the
   Justfile line only to a still-open later bead that genuinely needs it.
2. Close the epic: `sase bead close sase-1eq.1.1 --note "<verification>"`. The note must
   state: all seven phases verified against their commits (`015ce7f6` base, then
   `c4444abb`, `421324bf`, `4f0bfd33`, `e6a3452f`, `926edfb8`, `be86aa9f`, and
   `29da6fb7` in sase-core); this tale's hidden-alias fix and its core gate ToolRun id;
   the rust-dev-install into unchanged sase with the revisions; the four repaired sase
   nodes passing plus the focused test counts; all four binding pairs resolving and
   agreeing; both LSP binaries building and answering `--version`; and the follow-up
   outcomes from the `LAND TRIAGE` note (4 failures fixed as epic work, flake task
   sase-1ew created, provider_priority flake already tracked by sase-yn). Never use
   `--force` to make the close succeed.
3. From the primary sase checkout, run `just symvision` and confirm the whitelist is
   clean.
4. Set `status: done` in the frontmatter of the epic plan file. Use the PLAN path that
   `sase bead read sase-1eq.1.1` prints (`plan:202610/finish_core_macro_expand.md`).
5. The epic's `parent_bead` is **phase** `sase-1eq.1` ("sase-core additive macro
   rename"). Read it with
   `sase bead read sase-1eq.1 -r "Need the parent phase scope before closing"` and run
   `sase bead epic-symbols sase-1eq.1` (planning found none; resolve any new entry as in
   step 1). Check that the child epic completed that phase's whole additive contract:
   internal renames with legacy serde pins, new input spellings (env vars, option keys,
   YAML keys, `%macros_enabled`, content layout), the four new binding names beside the
   legacy ones, the `sase-macro-lsp` binary, durable new-first readers, and
   unchanged-sase verification. Then close only that phase:
   `sase bead close sase-1eq.1 --note "<combined core gate; unchanged sase revision and rebuilt core identity; health and focused compatibility tests incl. the four repaired directive nodes; both bindings and binaries; policy/durable coverage; flake sase-1ew noted as independent>"`.
   Do **not** close `sase-1eq` or any other ancestor. The containing epic has its own
   land agent and downstream phases.
6. Finish with `/sase_final`. Declare the sase-core change and the truthful bead
   completions. No manual commit is needed, and no step may wait for this turn's own
   host-created commit, push, or CI.
