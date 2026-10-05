---
tier: tale
size: small
title: Finish and land the macro choice-wire LSP epic (sase-1g4.2.1)
goal:
  Fix the remaining epic-caused issues (core clippy, the xprompt-spelled choice
  diagnostic code, orphaned swept-in files, stale terminology pairs), then close epic
  sase-1g4.2.1 and its parent phase sase-1g4.2 with recorded verification.
proposed_by: bbugyi200.athena.sase-1g4.2.1.land
bead: sase-1g4.2.1
create_time: 2026-10-05 08:23:58
status: wip
---

- **PARENT:**
  [202610/macro_choice_wires_lsp.md](https://github.com/sase-org/sase--plans/blob/main/202610/macro_choice_wires_lsp.md)
- **BEAD:** sase-1g4.2.1

# Finish and land epic sase-1g4.2.1 (macro choice wires and LSP assistance)

## Context

Epic `sase-1g4.2.1` ("Carry macro input metadata and finish enum assistance in the LSP")
has all five phases closed. Its land agent verified the implementation and recorded the
evidence in the epic's `LANDING TRIAGE` note (`sase bead read sase-1g4.2.1 -r "..."`).

- **Core commits:** `0d27dada`, `57fee306`, `ecd2e074`.
- **sase commits:** `4fd4039a5a`, `399b13f3ef`, `a6df140bcc`, `6fde796604`,
  `8c8c47f720`.
- **Pin:** `sase-core-revision.txt` is `ecd2e074`, which is core HEAD.
- **Passing at HEAD:** 287 epic Python tests, `sase_gateway` (224), and the
  `sase_macro_lsp` choice-completion and choice-diagnostics JSON-RPC suites.

Four small problems caused by the epic remain. This tale fixes them and then performs
the whole closeout. The epic has no other landing agent: **this tale's final steps close
the epic and its parent phase in the same turn as the code.** Do not wait for, or order
anything after, this work's own commit, SHA, push, or CI.

Out of scope (already triaged, do not fix here):

- **Pre-existing core test failures.** These come from the in-progress rename epic
  `sase-1eq` (core flip `0279de6b`, phase `sase-1eq.10`) and are recorded as a
  DISCOVERED ISSUE on `sase-1eq`:
  - 14 `sase_core --lib` failures;
  - `python_wire_parity::proc_snapshot_json_uses_canonical_proc_keys`;
  - 6 `sase_core_py --lib` schema-pin failures;
  - 2 `sase_macro_lsp --lib` failures (`metadata_env_prefers_macro_prefix`,
    `surfaces::exposes_hover_diagnostics_code_actions_and_definition`);
  - `jsonrpc_stdio::stdio_jsonrpc_frontmatter_diagnostics`, which **hangs forever**.
- **Host setup blocker.** The host `_setup` probe fails with "sase_content_layout probe
  returned stale schema: got 6, expected 5". This is `sase-1eq` note #4, item 4.
- **Known mypy errors.** These are recorded on `sase-1eq` and `sase-1g4`:
  - `InputType` in `input_item_modal.py`;
  - `LEGACY_XPROMPT_JINJA_SCOPE_KIND` callers;
  - `LOCAL_XPROMPTS_ENV`.

Shared behavior lives in the linked `sase-core` repo. Open it with
`sase repo open sase-core -r "<specific reason>"`, use only the printed path, and read
its `AGENTS.md`. Paths marked **core** are relative to that checkout; other paths are
relative to the sase checkout. Before verifying, read `lint_and_test.md` and
`symvision.md` through `/sase_memory_read`. Never run bare `cargo`. Never run
`just check-full`.

## 1. Fix the core clippy failure this epic introduced

The contracts phase (`0d27dada`) left `clippy -D warnings` red, so core
`sase tool run check` fails at clippy (ToolRun `b4bae47dd49109560393bbb48f1e2bd2`). Make
these refactors with no behavior change:

- **core** `crates/sase_core/src/editor/macro_arg_choices.rs`: `type_complexity` fires
  on the row tuple `(String, Option<String>, Option<String>, usize, bool)` at about
  lines 94, 126, and 152, where the fuzzy-scored vector wraps it in `((u8, i32), row)`.
  Replace the tuple with one small private named struct, or a private type alias, used
  by all three sites.
- **Same file, in the test module:** `cloned_ref_to_slice_refs` fires at about line 373,
  `classify_completion_context(&doc, pos, &[entry.clone()])`. Use
  `std::slice::from_ref(&entry)`.
- **core** `crates/sase_core/src/macro_catalog/parsing.rs`: `type_complexity` fires on
  the return tuple of `parse_short_input_value`, at about line 651. Return a small
  private struct instead and update its callers.

Verify with `just fmt` and then `just clippy` in core. Clippy must be clean.

## 2. Rename the new choice diagnostic code to the macro spelling

The epic introduced the externally visible code `invalid_xprompt_arg_choice`
(`INVALID_XPROMPT_ARG_CHOICE`). It came from the pre-rename design. The core flip
`0279de6b` landed before this epic began and already renamed every sibling code:

- `unknown_macro_arg`
- `duplicate_macro_arg`
- `invalid_macro_arg_type`
- `*_macro_frontmatter_*`

Because of this one code, the parity phase had to add a terminology allowlist entry. No
consumer in sase, sase-nvim, or sase-telegram keys on the code yet, so renaming it now
is free. Later it would become a compatibility burden.

- **core:** rename the constant to `INVALID_MACRO_ARG_CHOICE` and its value to
  `"invalid_macro_arg_choice"` in `crates/sase_core/src/editor/diagnostics.rs`, covering
  the definition, the use at about line 541, and its tests. Update the literals in
  `crates/sase_macro_lsp/src/server/tests/choice_diagnostics.rs` and
  `crates/sase_macro_lsp/tests/jsonrpc_stdio_choice_diagnostics.rs`. Afterwards,
  `grep -rn invalid_xprompt_arg_choice` over core `crates/` must return nothing.
- **sase:** update the diagnostics row in `docs/editor.md`. In
  `tests/_macro_terminology_docs.py`, remove the
  `("docs/editor.md", "| Diagnostics ...")` allowlist tuple and the `"docs/editor.md"`
  entry in `_MACRO_DOCS_REASONS`, both added by `8c8c47f720`. Remove them only if
  `docs/editor.md` contains no other allowlisted retired term;
  `tests/test_macro_terminology.py` decides this. Afterwards,
  `grep -rn invalid_xprompt_arg_choice src tests docs` must return nothing.
- **Note the sibling phase.** After the rename, run:
  `sase bead note sase-1g4.4 "sase-1g4.2.1 landing renamed the closed-set argument diagnostic to invalid_macro_arg_choice to match the post-flip *_macro_arg* codes; emit the model warning as invalid_macro_arg_model, not the design's invalid_xprompt_arg_model."`

## 3. Remove the orphaned files swept into the parity commit

`8c8c47f720` accidentally committed two untracked files from the sase-146
adoption-report draft:

- `src/sase/core/tool_adoption.py`
- `src/sase/tool/adoption.py`

Nothing imports either file. The binding they call (`tool_adoption_report`) exists in no
sase-core revision. `sase-146` already has a note saying the draft is recoverable from
`8c8c47f720`. First confirm with grep that they are still unreferenced. Then delete both
files. Leave `tools/tool_adoption_report` and the `tool-adoption` Justfile recipe
untouched; they are the working implementation.

## 4. Drop the terminology pairs this epic made stale

`8c8c47f720` rewrote the parity tests, so these exact allowlist pairs no longer match
their files. Remove them where they are defined. The aggregate sets in
`tests/_macro_terminology_string_pairs_a.py`, `_b.py`, and
`_macro_terminology_strings.py` derive from these definitions.

- `tests/_macro_terminology_string_pairs_a_late.py`:
  - `("tests/_macro_directive_completion_parity_helpers.py", 'elif operation == "xprompt-catalog":')`;
  - the five `tests/macro/test_argument_surface_parity.py` pairs
    `'"duplicate_xprompt_arg"'`, `'"invalid_xprompt_arg_type"'`,
    `'"unknown_xprompt_arg"'`, `'"xprompt"'`, and `'"xprompt_argument_spans"'`.
- `tests/_macro_terminology_string_pairs_b_late.py`: the pair
  `("tests/test_macro_jinja_lsp_parity.py", '"xprompt"')`. Keep the `'"xprompts"'` pair,
  which still matches line 246.
- `tests/_macro_terminology_strings.py`: the file-reason entry for
  `tests/macro/test_argument_surface_parity.py`, at about line 143. That file no longer
  contains any retired term. Keep this entry if the guard still requires it.

Only remove pairs whose quoted string is absent from the named file. Re-check each one
with grep before deleting it.

## 5. Verify

**sase checkout:**

1. Run `just fix`.
2. Run `sase tool run check` once and record its run id. It is expected to die in
   `_setup` on the pre-existing content-layout probe. If it fails anywhere else, that
   failure is yours.
3. Because of the `_setup` block, run these gates directly:
   - `.venv/bin/python -m pytest -q -n 8` over:
     - `tests/test_macro_terminology.py`
     - `tests/macro/test_argument_surface_parity.py`
     - `tests/macro/test_macro_choice_projection_parity.py`
     - `tests/test_macro_directive_completion_parity.py`
     - `tests/test_macro_finalizer_completion_parity.py`
     - `tests/test_macro_input_type_parity.py`
     - `tests/test_macro_jinja_lsp_parity.py`
     - `tests/test_macro_model_alias_shortcut_parity.py`
     - `tests/macro/test_cli_show_render.py`
     - `tests/macro/test_cli_show_resolve.py`
     - `tests/macro/test_highlight.py`
     - `tests/macro/test_macro_properties.py`
     - `tests/test_macro_catalog_structured.py`
     - `tests/test_mobile_helpers.py`
     - `tests/test_validate_sase_core_rs_contracts_tool.py`
   - `just _lint-ruff`;
   - Symvision, invoked exactly as `_lint-symvision` does but without its `_setup`
     prerequisite:
     `SASE_SYMVISION_BEAD_STATUS_ONLY=1 BD_COMMAND=tools/sase_bead .venv/bin/symvision src/sase --exclude-decorator gate_command_entrypoint --exclude-decorator builtin_chop`.
     It must print "All public/private classes/functions are used properly!".
   - `.venv/bin/mypy`: its only errors may be the known pre-existing ones listed above.

   Rebuild the venv's core first so the Python tests run against the renamed code
   (`just rust-install` or `just install`).

**core checkout:**

1. Run `just fmt` and `just clippy`; both must be clean.
2. Run
   `just test -p sase_core -p sase_core_py -p sase_macro_lsp -p sase_gateway --no-fail-fast -- --skip stdio_jsonrpc_frontmatter_diagnostics`.
   Always pass that skip: the test hangs forever. The only failures allowed are the
   pre-existing set listed under Context. Any other failure is yours.
3. Run `sase tool run check`. It now passes fmt, features, and clippy, then stops at the
   14 pre-existing `sase_core --lib` failures. Cargo stops at the first failing test
   binary, so it never reaches the hanging test. Record the run id.

**Declaring both repositories:** declare both through `/sase_final`. The host commits
core first and writes the pushed SHA into `sase-core-revision.txt`. Do not create
commits, branches, or PRs, and do not guess a SHA.

## 6. Close the epic (final step, same turn)

1. Run `sase bead epic-symbols sase-1g4.2.1`. Resolve or re-key every entry; the land
   agent saw none.
2. Close the epic:

   ```bash
   sase bead close sase-1g4.2.1 --note "<what you verified>"
   ```

   The note must summarize:
   - the five closed phases and the commits above;
   - the clippy fix, the `invalid_macro_arg_choice` rename, the orphan removal, and the
     stale-pair cleanup;
   - the test, clippy, and symvision results with ToolRun ids;
   - the pre-existing blockers attributed to `sase-1eq`;
   - the follow-up outcomes from the epic's `LANDING TRIAGE` note.

   Never use `--force`.

3. Confirm the whitelist with the direct Symvision invocation from step 5.
   `just symvision` also dies in `_setup`; say so in your report.
4. Set `status: done` in the frontmatter of the epic's plan file, the PLAN path shown by
   `sase bead read sase-1g4.2.1 -r "..."` (`plan:202610/macro_choice_wires_lsp.md`).

## 7. Close the parent phase sase-1g4.2

`sase-1g4.2.1`'s `parent_bead` is phase `sase-1g4.2`, whose own parent is epic
`sase-1g4`. Close only `sase-1g4.2`:

1. Re-read it with `sase bead read sase-1g4.2 -r "..."`. Confirm that this epic
   delivered the phase's scope:
   - choices, `named_type`, and `value_role` on every hint, catalog, mobile, and CLI
     projection;
   - the shared Rust candidate builder and type label;
   - LSP enum completion;
   - invocation diagnostics with "Replace with" quick fixes;
   - rich hover;
   - frontmatter type completion;
   - the shared golden fixtures.
2. Run `sase bead epic-symbols sase-1g4.2` and resolve or re-key any entries.
3. Close the phase:

   ```bash
   sase bead close sase-1g4.2 --note "<what you verified>"
   ```

   Do not use `--force`.

Never close `sase-1g4`; its own land agent owns that. If `sase-1g4.2` turns out to be
incomplete or ambiguous, do not close it. Instead, record a note on it describing the
blocker and report that in your final response.
