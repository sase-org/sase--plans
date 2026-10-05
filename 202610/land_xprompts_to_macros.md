---
tier: epic
title: Finish the xprompt-to-macro core flip and land sase-1eq
goal: sase-core's check is green again after the contract flip, sase-core and sase
  no longer emit pre-flip xprompt wire spellings outside durable legacy readers and
  flag-gated sunset inputs, and the macro docs infographic shows macro names, so the
  sase-1eq land agent can close the rename epic.
parent_bead: sase-1eq
phases:
- id: core-green
  title: Make sase-core green after the macro contract flip
  size: medium
  depends_on: []
  description: 'core-green: Repair every sase-core test the 0279de6b contract flip
    left red, including the corrupted local-helper reader, the hanging stdio diagnostics
    test, and the env-racing catalog test. Pin the flipped macro output. Keep durable
    legacy-input tests. Pass sase tool run check in sase-core. Follow the core-green
    section.'
- id: key-flip
  title: Drop leftover pre-flip xprompt wire keys in sase-core and sase
  size: medium
  depends_on:
  - core-green
  description: 'key-flip: Stop emitting the remaining pre-flip xprompt keys and values
    (content layout, snippet entries, launch units, proc and highlight labels). Rename
    leftover core identifiers, bump the content-layout schema, and update the sase
    mirrors, validator, and terminology guard in the same declared turn so the host
    moves the pin. Follow the key-flip section.'
- id: infographic
  title: Relabel the macro-resolution infographic
  size: small
  depends_on: []
  description: 'infographic: Replace every retired xprompt label in docs/images/macro-resolution-infographic.png
    with its macro spelling through a deterministic ImageMagick overlay, then update
    its prompt record and final SHA. Follow the infographic section.'
proposed_by: bbugyi200.athena.sase-1eq.land
create_time: 2026-10-05 10:21:49
status: wip
bead_id: sase-1eq.12
---

- **PROMPT:** [prompts/202610/land_xprompts_to_macros.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/land_xprompts_to_macros.md)
- **BEAD:** [sase-1eq.12](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1eq/sase-1eq.12.md)

# Plan: Finish the xprompt-to-macro core flip and land sase-1eq

## Context

Epic `sase-1eq` ("Rename xprompts to macros", plan `plan:202610/xprompts_to_macros.md`)
has all 11 phases closed. Its land agent verified the work on 2026-10-05 at sase
`17c2907d3b` with the core pinned at `fe2ef0e6`. It recorded a `LANDING TRIAGE` note on
`sase-1eq`; read it with `sase bead read sase-1eq --no-links -r "<why>"` first. Use
`--no-links` because a link-resolution bug, `sase-1gs`, makes plain `bead read` fail.

sase itself is green:

- `sase tool run check` passed every lint gate.
- Its escalated full suite ran 52669 passed, with 1 KNOWN flake (now `sase-1gp`).
- `sase core health` is OK.

The flip phase `sase-1eq.10` (plan `plan:202610/core_flip.md`) did not finish its
sase-core side, though:

1. **sase-core is red at master.** The flip commit `0279de6b` changed emitted spellings
   and schemas, but the tests still assert pre-flip output. This was reproduced at
   `fe2ef0e6`:
   - `sase_core --lib`: 14-15 failures.
   - `sase_core --test python_wire_parity`: 1 failure.
   - `sase_core_py --lib`: 6 failures.
   - `sase_macro_lsp --lib`: 2 failures.
   - `sase_macro_lsp --test jsonrpc_stdio`: `stdio_jsonrpc_frontmatter_diagnostics`
     hangs forever.

   As a result, `sase tool run check` in sase-core never finishes. These are epic note
   #5 on `sase-1eq`.

2. **Pre-flip keys are still emitted.** sase-core and sase still emit some pre-flip
   xprompt wire keys and values that `core_flip.md` required to flip (sase-1eq.11
   PROPOSED FOLLOW-UP #3). The terminology guard still allowlists about 20 sase files as
   "pre-flip wire/transport vocabulary; owned by sase-1eq.10".
3. **The infographic is stale.** `docs/images/macro-resolution-infographic.png` still
   renders retired xprompt labels (sase-1eq.6 PROPOSED FOLLOW-UP #1).

Everything here was caused by this epic. Follow-ups not caused by it were already
triaged on `sase-1eq`, so do not file or fix them here: `sase-172`, `sase-18v`,
`sase-1ct`, `sase-1gn`, `sase-1go`, `sase-1gp`, `sase-1gq`, `sase-1gr`, `sase-1gs`, and
the `sase-z3` note.

## Shared rules (every phase)

- Shared behavior lives in the linked `sase-core` repo. Open it with
  `sase repo open sase-core -r "<specific reason>"`, use only the printed path, and read
  its `AGENTS.md` before editing. Paths marked **core** are relative to that checkout.
  Other paths are relative to the sase checkout.
- Before you finish sase edits, read `lint_and_test.md`, and `symvision.md` if Symvision
  complains, through `/sase_memory_read`.
- Verification:
  - Run `sase tool run check` in each repo you changed. In sase-core, give it at least
    10 minutes.
  - A targeted `just test -p` never replaces the full check.
  - Do not run `just check-full`.
- Do not hand-edit `sase-core-revision.txt` and do not run `just ratchet-core-revision`.
  When one declaration covers sase-core and sase, the host commits sase-core first and
  writes its SHA into the pin.
- `macro` is a Rust keyword. A JSON field that must be `macro` uses
  `#[serde(rename = "macro")]` on another identifier.
- **Legacy policy (from the parent plan, unchanged):**
  - Writers emit only macro spellings.
  - Keep a legacy alias or reader only where durable data can still carry the old
    spelling. That covers proc rows, artifact file names, `record_json` keys,
    `%xprompts_enabled`, persisted artifact-ref kind `xprompt_skill`, stored MRU and
    save-state files, and authored `xprompts:` config and frontmatter.
  - The flag-gated user spellings also stay, behind the `legacy_xprompt_syntax` sunset
    flag (`sase-1fj`).
  - Mark every surviving legacy literal in Rust with `// legacy xprompt spelling`.
  - In Python, route surviving legacy literals through
    `src/sase/legacy_xprompt_names.py` or `src/sase/legacy_xprompt_syntax.py`.
  - In-process request wires drop their legacy aliases.
- **Never rewrite a realistic legacy-input fixture just to make an assertion pass.** A
  test that proves stored legacy data still loads keeps its legacy input and gains a
  canonical-output assertion. A test that pinned pre-flip _output_ is updated to the
  flipped output.
- Never weaken an assertion to get a green run. A failure that reproduces on the clean
  base tree and is not listed here goes on your phase bead as `PROPOSED FOLLOW-UP:`. Do
  not create beads.
- Do not close `sase-1eq` or any phase of it. This epic's land agent resumes that
  landing.

## core-green

Work only in sase-core unless a fix proves a sase caller is wrong. Reproduce first:

```bash
just test -p sase_core --lib
just test -p sase_core --test python_wire_parity
just test -p sase_core_py --lib
just test -p sase_macro_lsp --lib
timeout 300 just test -p sase_macro_lsp --test jsonrpc_stdio   # hangs today
```

Known failures and the intended direction. Re-derive each one from the code; this list
is a map, not a substitute for reading the test.

### Real bugs

- **The local-helper reader was corrupted by the mechanical rename.** In **core**
  `crates/sase_core/src/editor/diagnostics.rs`, `local_macro_entries` loops over
  `["macros", "macros"]`. Its comment says it reads the canonical `macros:` section and
  the retired spelling, so the second key must be the retired `xprompts`, marked
  `// legacy xprompt spelling`. Restore it. Then confirm
  `editor::diagnostics::tests::canonical_local_section_wins_on_helper_name_conflict`
  passes with its legacy input intact.
  - Inspect
    `editor::frontmatter::authored_inputs_tests::duplicate_local_sections_are_an_error_naming_macros`
    (`editor/frontmatter.rs` near `validate_local_section_keys`) for the same kind of
    corruption.
  - Grep the crate for other `[..."macros", "macros"...]`, `"macros" | "macros"`, or
    identical-pair legacy readers and fix any you find.
- **`stdio_jsonrpc_frontmatter_diagnostics` hangs.** It is in **core**
  `crates/sase_macro_lsp/tests/jsonrpc_stdio*.rs`. It waits for the pre-flip
  `*_xprompt_frontmatter_*` diagnostic codes, which the flipped server never sends.
  - Update it to the macro codes.
  - Make the shared wait helper fail after a bounded timeout with the last messages
    received, so a future mismatch fails instead of hanging the whole check.
- **`macro_catalog::tests::catalog_sources::macro_env_precedence_is_new_first` races.**
  It sets and removes process-global `SASE_*_DIR` variables while sibling tests run in
  parallel threads, and it failed in 1 of 2 identical runs. Serialize it, and every
  other test that mutates those catalog variables, behind one shared test mutex, or pass
  the paths through `MacroCatalogLoadOptions` instead of the environment. Keep its
  new-first assertions.

### Stale pre-flip expectations

Update each of these to pin the flipped canonical output:

- `agent_launch::tests::wires::launch_request_local_macros_alias_matches_legacy_key`:
  `local_macros_file` is request-only, so the legacy alias is dropped. Assert only the
  canonical key, and assert that the legacy key is not accepted.
- `agent_stats::wire::tests::macro_request_keys_emit_legacy_xprompt_spellings` and
  `macro_stats_response_emits_legacy_xprompt_keys`: assert the macro keys (`macros`,
  `runs_with_macros`, `runs_without_macros`, `distinct_macros`, `macro_top_n`,
  `macro_breakdown_top_n`, `macro_focus`), and rename the tests to match. Keep the
  older-payload deserialization tests if their input is genuinely stored data.
- `agent_stats::run::tests::macros::aggregates_ranked_xprompt_usage_and_focused_breakdowns`
  and
  `runner_occupancy::runner_occupancy_handles_overlap_carry_in_waits_and_boundaries`:
  stats schema 8.
- `content_layout::tests::ref_directories_are_canonical`: content-layout schema 6.
  `key-flip` bumps it again.
- `editor::hover::tests::builds_frontmatter_field_hover` and **core** `sase_macro_lsp`
  `server::tests::surfaces::exposes_hover_diagnostics_code_actions_and_definition`: the
  fixture uses an `xprompts:` key. Hover the canonical `macros:` key. Add a legacy-key
  case only if the server is meant to hover retired keys.
- `editor::wire::tests::completion_context_macro_variants_pin_legacy_output` and
  `macro_argument_source_accepts_old_and_new_spellings`: legacy request variants are
  rejected by design. Pin the canonical output and assert that `xprompt` and
  `xprompt_argument_*` are rejected.
- `macro_catalog::tests::loading::loads_markdown_and_workflow_with_canonical_insertions`:
  kind `macro`.
- `procs::store::tests::prompt_proc_field_accepts_both_spellings_and_emits_legacy`: read
  both spellings and emit `prompt_proc`. Rename the test.
- `query::profile::tests::patch_profile_digest_matches_python_compiler`: sase
  `f0893af93b` made the Python compiler hash under `shorthands`. Recompute the expected
  digest from sase's Python compiler, not from the Rust output. If the two disagree, fix
  the side that does not match the canonical `shorthands` payload.
- `python_wire_parity::proc_snapshot_json_uses_canonical_proc_keys`: proc schema 5.
- `sase_core_py` pins:
  - `agent_scan` stats and activity stats: 8.
  - Scan snapshot: 12.
  - `editor_content::memory_xprompt_bindings_expose_the_shared_contract`: content
    layout 6. Rename the test to a macro name.
  - `procs::tests::reserve_proc_uses_proc_name_spelling` and
    `proc_store_bindings_round_trip_python_dicts_and_legacy_aliases`: fail with "reserve
    requires schema_version 5". Build v5 request dicts, and keep a legacy-read case for
    stored v1-v4 rows.
- `sase_macro_lsp` `server::tests::macro_lsp::metadata_env_prefers_macro_prefix`:
  `core_flip.md` removed `SASE_XPROMPT_*` reads from the LSP crate. Assert that the
  legacy variable is ignored, and rename the test.

### Verify

`sase tool run check` in sase-core passes. Then:

1. Build that core into the sase workspace with `just install`.
2. Run `sase tool run check` in sase.
3. Confirm `sase core health` is green.

If this phase changed no sase file, declare only sase-core. `key-flip` moves the pin.

## key-flip

Depends on `core-green`. Before editing, search both repos:

- `git grep -n -i xprompt -- crates` in sase-core.
- `git grep -n -i xprompt -- src tools` in sase.

Classify every hit as one of:

- (a) a durable legacy reader;
- (b) a flag-gated sunset input;
- (c) a test fixture that proves legacy input;
- (d) leftover pre-flip vocabulary to flip.

Flip every (d). Mark every (a) to (c) as the shared rules require.

### sase-core

- **Content layout** (**core** `crates/sase_core/src/content_layout.rs`):
  - Stop emitting the per-scope `xprompts` path fields on the project, home, and chezmoi
    layout wires, and the `xprompt_sources` list. Retired directories (`sase/xprompts`,
    `.xprompts`, `xprompts`, `dot_xprompts`, `~/sase/xprompts/...`) stay discoverable as
    `legacy` entries of the `macros` compatible path and as flag-gated entries in
    `macro_sources` (`accept_legacy_xprompt_names`).
  - Point canonical symbolic locators at the moved package data:
    - sase's packaged dirs are `src/sase/macros/` and `src/sase/default_macros/`;
    - check `package:xprompts`, `package:default_xprompts`, `package:xprompts/steps`,
      and `package:xprompts/skills` against how sase resolves them;
    - `entrypoint:sase_xprompts/*` is a flag-gated legacy plugin source.
  - Rename internal ids such as `project_xprompt_canonical` where they are canonical
    rather than legacy.
  - Bump `CONTENT_LAYOUT_SCHEMA_VERSION` to 7.
- **Snippet entry wire** (**core** `crates/sase_core/src/host_bridge.rs`):
  `EditorSnippetEntryWire.xprompt_name` becomes `macro_name`, and snippet source and
  kind value `xprompt` becomes `macro`. Update the mobile and gateway consumers and
  their contract snapshots, following the "Add a gateway route" regeneration steps in
  **core** `AGENTS.md`. Bump the owning wire schema if its shape changed.
- **Internal identifiers:**
  - `XPromptOccurrence` → `MacroOccurrence`;
  - `xprompt_occurrences` → `macro_occurrences`;
  - `xprompt_reference_re` → `macro_reference_re`;
  - `scan_xprompt_reference_end` → `scan_macro_reference_end`.

  The catalog option fields `package_xprompts_dir`, `default_xprompts_dir`, and
  `plugin_xprompt_dirs` get macro names. If they are request-wire keys, emit and accept
  only macro keys and update the sase callers.

- **Legacy reader constants** get explicit legacy names, so readers cannot mistake them
  for canonical names:
  - `XPROMPT_PROC_ORIGIN` → `LEGACY_XPROMPT_PROC_ORIGIN`;
  - scanner `USED_XPROMPTS_FILE` → `LEGACY_USED_XPROMPTS_FILE`;
  - scanner `RAW_PROMPT_FILE` (`raw_xprompt.md`) → `LEGACY_RAW_XPROMPT_FILE`.
- **Gateway:** `routes/mobile_handlers.rs` `macro_catalog` authenticates with the label
  `/api/v1/xprompts/catalog` for both routes. Use the canonical route label, unless the
  label is a stored scope that must stay. Keep the deprecated alias route itself.
- Update the tests, fixtures, `python_wire_parity`, and README/docs lines for every
  emitted change.

### sase (same declared turn)

- `src/sase/core/content_layout_wire.py`:
  - drop the `xprompts` fields and `xprompt_sources`;
  - require schema >= 7;
  - move consumers to `macros` and its legacy entries, for example
    `src/sase/ace/tui/modals/macro_browser_helpers.py`, which still reads
    `.xprompts.write_path`.
- Update `tools/validate_sase_core_rs` and its contract tests
  (`tests/test_validate_sase_core_rs_contracts_*.py`) to every bumped schema.
- Snippets: `src/sase/completion/candidates/catalog_snippets.py` reads only
  `macro_name`. `src/sase/snippet/models.py`, `catalog.py`, and `mutation.py` use kind
  `macro`. If snippet kind or `xprompt_name` is persisted in user files, read the legacy
  value through `legacy_xprompt_names.py` and keep a legacy-input test.
- Launch units: `src/sase/agent/launch_guard.py` `swarm_xprompts` and
  `src/sase/agent/multi_prompt_launch_execution.py` `segment_swarm_xprompts` become
  `swarm_macros` and `segment_swarm_macros`. If run-history ingress
  (`tests/main/test_sase_run_history_ingress.py`) or another stored path proves stored
  requests use the old key, read it through `legacy_xprompt_names.py` and keep that
  test.
- `src/sase/agent/_macro_swarm_rendering.py` fallback name `xprompt` and the
  `src/sase/agent/macro_swarm.py` group-id prefix `xprompt:` move to macro spellings.
  First check every reader of those ids.
- `src/sase/agent/launch_proc_runtime.py`: the reason code, messages, and
  `workflow=f"xprompt-proc:..."` move to the `prompt-proc` spelling. Stored rows keep
  loading through `prompt_proc_origin_matches`.
- `src/sase/macro/highlight.py`: drop `"xprompt"` from `_SOURCES` and from the value
  check, unless a persisted fixture still carries it.
- `src/sase/core/macro_skill_definition_facade.py`: the error text says `macro-skill`.
- `src/sase/macro/used_macros.py`: remove any remaining `SASE_LAUNCH_SWARM_XPROMPTS`
  write. Keep a pop or read only if a running pre-upgrade child can still set it.
- `src/sase/integrations/macro_lsp.py`: stop exporting `SASE_XPROMPT_*` transport
  variables the LSP no longer reads. Keep the flag-gated `SASE_XPROMPT_LSP_CMD` user
  spelling.
- Terminology guard: delete every row in `tests/_macro_terminology_strings.py`, every
  comment in `tests/_macro_terminology_identifiers.py` that names `sase-1eq.10`, and
  every pair-table row (`tests/_macro_terminology_string_pairs_*.py`) for the flipped
  literals. A remaining hit needs a durable-reader or `legacy_xprompt_syntax` reason;
  "owned by sase-1eq.10" is not a reason. Do not widen the guard.

### Verify and declare

1. `sase tool run check` in sase-core.
2. `just install` in sase, `just fix`, then `sase tool run check` in sase, and
   `sase core health`.
3. Re-run both `git grep -i xprompt` sweeps and confirm every hit is class (a), (b), or
   (c).
4. Declare sase-core and sase in one turn so the host commits core and moves the pin.
   The sase-core message is `feat!:` with a `BREAKING CHANGE:` footer listing the
   emitted key changes and schema bumps.

## infographic

`docs/images/macro-resolution-infographic.png` (1672×941) still renders these labels:

- "inline xprompt · embeddable workflow · xprompt swarm";
- "Iterative xprompt expansion · ≤100 passes";
- discovery rungs `<project>/sase/xprompts/`, `~/sase/xprompts/`,
  `~/sase/xprompts/{project}/`, "plugin packages (sase_xprompts EPs)",
  `<pkg>/default_xprompts/*.md`, and `<pkg>/xprompts/ + steps/`;
- "protected while xprompt references expand";
- "after full xprompt expansion";
- `sase xprompt graph` / `sase xprompt explain`.

Its record, `docs/images/macro-resolution-infographic.prompt.md`, says the labels are a
deterministic ImageMagick overlay on a text-free base. Only the composited PNG is
committed.

1. Take every corrected string from the current `docs/macros.md`, including its
   canonical discovery table and the `sase macro graph` / `sase macro explain` commands.
   Do not invent paths.
2. Cover each stale label with an opaque backing box that matches its card's local
   background color, sampled from the PNG. Render the corrected label in the same
   position, weight, and approximate size. Use ImageMagick 7 (`magick`), DejaVu Sans for
   prose, and a monospace font for paths (Fira Code if installed, else DejaVu Sans
   Mono). `rsvg-convert` is not installed, so draw with `-annotate`/`-draw` or let
   ImageMagick render the SVG itself. Keep the canvas at 1672×941 sRGB and strip
   metadata.
3. Open the result with your image viewer at full resolution:
   - No xprompt text remains.
   - No old glyph shows around a patch.
   - Labels are legible and inside their cards.
   - Arrows and panel borders are untouched.

   Iterate until it is clean.

4. Append the exact commands and label list to the prompt record's "Deterministic
   Post-Processing Record". Update its final SHA-256. Update
   `macro-resolution-infographic.critique.md` if it describes the labels.
5. Run `sase tool run check` in sase.

## Exit

- sase-core's `sase tool run check` passes with no hanging test.
- sase and sase-core emit only macro spellings outside classified legacy readers,
  fixtures, and flag-gated inputs.
- The pin names the key-flip core commit.
- No terminology-guard row cites `sase-1eq.10`.
- The infographic shows only macro vocabulary.
- The `sase-1eq` land agent then resumes and closes the rename epic.
