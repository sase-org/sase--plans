---
tier: tale
title: Flip sase-core onto macro spellings and pin sase in the same turn
goal: "sase-core emits only macro spellings, the legacy Python binding names are gone,
  and the LSP crate is sase_macro_lsp. The same turn updates sase's mirrors and the
  chezmoi LSP installer. The host writes the new core SHA into the sase pin. Durable
  pre-rename data still loads.

  "
size: medium
proposed_by: bbugyi200.athena.sase-1eq.10
bead: sase-1eq.10
create_time: 2026-10-04 23:38:47
status: wip
---

- **PARENT:**
  [202610/xprompts_to_macros.md](https://github.com/sase-org/sase--plans/blob/main/202610/xprompts_to_macros.md)
- **BEAD:**
  [sase-1eq.10](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1eq/sase-1eq.10.md)

# Plan: Flip sase-core onto macro spellings and pin sase in the same turn

This is one tale, not a child epic. The parent design
(`plan:202610/xprompts_to_macros.md`, section "sase-core contract flip with same-turn
pin bump") requires sase-core, sase, and chezmoi to land in one declared turn. sase CI
builds `sase_core_rs` from `sase-core-revision.txt`, and the six-hourly ratchet will pin
sase-core's HEAD if this flip lands alone. Splitting the breaking commit from the sase
update recreates the sase-1ab failure this epic exists to prevent. The work is one
bounded checklist, so the tale size is medium.

Implement the parent section and its shared policy. Where this plan and the parent
disagree about a current constant, trust the tree: the index schema is 35, not 19. Do
not re-plan.

## Out of scope

Leave these for `audit-deploy` (sase-1eq.11) and the sunset flag:

- Deleting `src/sase/xprompt/` and `TEMP(xprompt->macro shim)` aliases.
- Widening `tests/test_macro_terminology.py` to the whole repo.
- `git mv` of chezmoi `home/sase/xprompts`, `.chezmoiremove`, `sase.yml` key renames,
  live machine migration, and `sase skill init --force`.
- Removing `/api/v1/xprompts/catalog`. It stays inside mobile API v1 as a deprecated
  alias. Record that removal as a `PROPOSED FOLLOW-UP:` note.
- Removing `legacy_xprompt_syntax`, `src/sase/legacy_xprompt_names.py`, or
  `src/sase/legacy_xprompt_syntax.py`.
- Flag-gated user spellings: `SASE_XPROMPT_LSP_CMD`, `SASE_DISABLE_PLUGIN_XPROMPTS`, the
  `sase xprompt` command, and a `sase-xprompt-lsp` binary still accepted on PATH while
  the flag is enabled.
- Jinja's `{% macro %}` keyword, `JinjaLocalKind` macro variants, and the LSP semantic
  token type `macro`. Do not reorder the token legend.
- Query-language shorthands are already renamed. Do not rename them back.
- Crate `version` fields and generated `CHANGELOG.md` files. release-plz owns those. Do
  bump `*_SCHEMA_VERSION` constants whose emitted shape changes.
- Closing epic sase-1eq, phase sase-1eq.11, or any ancestor. Do not create beads.
  Discovered work is a `PROPOSED FOLLOW-UP:` note on sase-1eq.10.

## Rules for every emitted spelling

`macro` is a Rust keyword. Never introduce a bare `macro` identifier. A JSON field that
must be `macro` uses `#[serde(rename = "macro")]` on another name.

Today the expand phase emits the legacy spelling and accepts the new one
(`rename = "<legacy>", alias = "<new>"`). Flip that:

- Emit the macro spelling.
- Keep `alias = "<legacy>"`, with a `// legacy xprompt spelling` comment, only where
  durable data can still carry the old spelling: proc rows, artifact file names and
  `record_json` keys, `%xprompts_enabled`, persisted artifact-ref target kind
  `xprompt_skill`, and any other value an earlier phase proved is stored. Writers emit
  only the new spelling.
- Drop aliases on in-process request wires. Jinja scope kind and the completion spacer
  are request values, not stored files.
- Python legacy spellings that must survive live only in
  `src/sase/legacy_xprompt_names.py`. Do not blind-replace that file, the flag module,
  `*legacy_xprompt*` fixtures, or `LEGACY_*` names.
- Case-aware replacement order, when a mechanical edit is appropriate: article forms,
  then `XPROMPT`, `XPrompt`, `Xprompt`, then `xprompt`. Review every legacy constant
  after the edit.

Search both repos for `rename = "xprompt` and for pyo3 names containing `xprompt` before
editing, and again before declaring. The list below is the contract, not a substitute
for that search.

## sase-core

Open the linked repo with `sase repo open sase-core` and read its `AGENTS.md` before
editing. Agents run `sase tool run check` there, not bare `just check`. `just fast` is
the inner loop. A features failure is fixed with
`cargo hakari generate && cargo hakari manage-deps`.

### Serialized output

Swap the legacy rename pins. Known sites:

- `crates/sase_core/src/procs/wire.rs`: `xprompt_proc` emits `prompt_proc`. Keep the
  alias. `procs.jsonl` is durable. Origin value becomes `prompt-proc`, still reading
  `xprompt-proc`.
- `crates/sase_core/src/agent_stats/wire.rs`: emit `macros`, `runs_with_macros`,
  `runs_without_macros`, `distinct_macros`, `macro_top_n`, `macro_breakdown_top_n`, and
  `macro_focus`. Stats responses are computed, not stored. Drop the legacy aliases once
  tests read only the new keys.
- `crates/sase_core/src/agent_scan/wire.rs`: `used_xprompts` emits `used_macros` on
  `UsedMacroWire`. Keep the alias. Stored `record_json` must still deserialize. Do not
  rewrite every index blob.
- `crates/sase_core/src/editor/wire.rs`: completion context kinds emit `macro` and
  `macro_argument_*`, plus `active_macro`. These are request and response wires. Drop
  legacy aliases unless a test proves the value is persisted.
- `crates/sase_core/src/agent_launch/wires.rs`: `local_xprompts_file` emits
  `local_macros_file`. Drop the alias if the field is request-only.
- `crates/sase_core/src/content_layout.rs`: stop emitting `xprompt_sources` (and any
  `xprompts` key). `macro_sources` stays, and `sase/macros` is the canonical directory.
  Keep discovery of user-authored legacy directories. That discovery is flag-gated
  input, not an emitted key.
- `crates/sase_core/src/query/profile.rs`: `CompiledQueryProfileWire` still deserializes
  a `macros` field with a `shorthands` alias. Emit `shorthands` only. Keep accepting a
  stored `macros` key on input.
- `crates/sase_core/src/artifact_ref/wire.rs`: emit `macro_skill`. Keep parsing
  `xprompt_skill`. Artifact refs are persisted.
- Workflow kind value `macro`.
- Jinja scope kinds `macro`, `macro_only`, and `macro_skill_only`. Accept only those on
  the request wire.
- Grep `axe_chop` for an xprompt migration key and emit the macro spelling.
- Bump `FLEET_CONTRACT_SCHEMA_VERSION` only if a fleet wire's emitted shape changed.
  Confirm with a search. Do not bump it speculatively.

### Schema versions

Bump each constant whose emitted shape changed, in Rust and in every sase mirror in the
same turn. Keep older versions in the existing supported set so stored payloads still
load. Do not hand-edit a crate version.

Current values to re-check in the tree:

- `CONTENT_LAYOUT_SCHEMA_VERSION` is 5. Losing `xprompt_sources` bumps it.
- `AGENT_SCAN_WIRE_SCHEMA_VERSION` is 11. The `used_macros` key bumps it. Python
  `SUPPORTED_AGENT_SCAN_WIRE_SCHEMA_VERSIONS` is `{10, 11}`. Add the new version. Do not
  drop versions that still appear in stored scans.
- `AGENT_STATS_WIRE_SCHEMA_VERSION` is 7. The stats key flip bumps it.
- `PROC_WIRE_SCHEMA_VERSION` is 4, with support for 1 through 4. The `prompt_proc` key
  bumps it and keeps 1 through 4 readable.
- `AGENT_ARTIFACT_INDEX_SCHEMA_VERSION` is 35. The column rename below bumps it to 36.
  Python mirrors currently publish 34 and accept `{33, 34, 35}` in
  `src/sase/core/agent_scan_wire_records.py`. Update every copy, including
  `tools/validate_sase_core_rs`.
- `MACRO_SKILL_DEFINITION_WIRE_SCHEMA_VERSION` is 1. Bump it only if the JSON shape
  changes. The binding rename alone does not. Do fix the sase facade error text that
  still says `xprompt-skill`.

### Agent-scan index

In `crates/sase_core/src/agent_scan/index/storage.rs`, add the next step in the existing
chain (`prior_version < 36`) and set `AGENT_ARTIFACT_INDEX_SCHEMA_VERSION` to 36. Rename
SQLite column `xprompts_sig` to `macros_sig` in place (`ALTER TABLE ... RENAME COLUMN`).
Update the SQL in `selection.rs`, `alias_history.rs`, `maintenance.rs`, and
`refresh.rs`. Do not rescan or rewrite the artifact archive on open. A no-op open of an
already-migrated index must stay a no-op.

### Bindings

Remove these Python names and assert them absent. The macro names are already
registered. Keep those:

| Remove                                         | Keep                                         |
| ---------------------------------------------- | -------------------------------------------- |
| `resolve_xprompt_skill_definition`             | `resolve_macro_skill_definition`             |
| `xprompt_skill_definition_wire_schema_version` | `macro_skill_definition_wire_schema_version` |
| `xprompt_argument_spans`                       | `macro_argument_spans`                       |
| `xprompt_proc_origin`                          | `prompt_proc_origin`                         |

The completion spacer has no macro binding yet. Rename
`xprompt_completion_spacer_to_parentheses_edit` to
`macro_completion_spacer_to_parentheses_edit`, including the Rust function
`plan_xprompt_completion_spacer_to_parentheses_edit` and the prelude alias. Register
only the new name. This is an in-process binding. Do not keep the legacy name.

`prompt_proc_origin()` must return `prompt-proc`.

### Env transport

Stop reading `SASE_XPROMPT_*` transport variables inside the LSP crate (`server/mod.rs`,
`catalog_cache.rs`). Read only the `SASE_MACRO_*` variables the sase launcher already
sets. Stop reading `SASE_AGENT_LOCAL_XPROMPTS`. Set and read only
`SASE_AGENT_LOCAL_MACROS`.

### Diagnostics and LSP copy

In `crates/sase_core/src/editor/diagnostics.rs` and the LSP tests, rename the xprompt
diagnostic codes to the matching `*macro*` codes (`unknown_macro`,
`invalid_macro_arg_type`, `missing_macro_memory_tag`, and the rest of that family).
Messages say "macro". Diagnostic source becomes `sase-macro`. Code-action titles become
"Open macro source" and "Refresh macro catalog". Hover text that means a Jinja macro
says "Jinja macro".

### LSP crate

`git mv crates/sase_xprompt_lsp crates/sase_macro_lsp`.

- Package name `sase_macro_lsp`. Its only binary is `sase-macro-lsp`. Remove the
  `sase-xprompt-lsp` bin target.
- `serverInfo.name`, `--version`, and the log filter use `sase-macro-lsp`. The server
  type stays `MacroLspServer`.
- Command ids are only `sase.macroLsp.refreshCatalog` and `sase.macroLsp.openSource`.
- Update the workspace `Cargo.toml` members and `Cargo.lock`.
- In `release-plz.toml`, replace the `sase_xprompt_lsp` package entry with
  `name = "sase_macro_lsp"`, `release = false`, and its own
  `git_tag_name = "sase_macro_lsp-v{{ version }}"`. Without that tag template, git_only
  release-pr cannot find the renamed crate and every release fails.
- Update `AGENTS.md`'s crate table row.

### Gateway

- Rename `MobileXprompt*Wire` to `MobileMacro*Wire`.
- Add `GET /api/v1/macros/catalog` as the canonical route. Keep
  `GET /api/v1/xprompts/catalog` as a deprecated alias that serves the same payload.
  Register both in `routes/router.rs` and declare both in `contract.rs`.
- `host_bridge.rs` calls `macro-catalog`, not `xprompt-catalog`.
- Regenerate `crates/sase_gateway/contracts/api_v1/mobile_api_v1.json` with
  `UPDATE_MOBILE_CONTRACT=1 just test -p sase_gateway committed_`.
- Update `crates/sase_gateway/README.md`.

### Fixtures and docs

- `git mv` `tests/fixtures/xprompt_args_corpus.json` to `macro_args_corpus.json` in
  sase-core and in sase.
- Update `crates/sase_core/tests/python_wire_parity.rs` and the prose in
  `tests/fixtures/note_attachment/at_bearing_notes.jsonl`.
- Regenerate `tests/fixtures/command_line/sase_spec.json` from the landed sase CLI,
  following that directory's README, after the sase CLI side is unchanged by this tale.
  This tale does not rename CLI commands. Regenerate only if a core-owned spec drifted.
  If it did not, leave it.
- Update `README.md`, `crates/sase_core_py/PYPI_README.md`, and `docs/` lines that name
  the old crate or binary.

The sase-core commit subject is `feat!:` and the body has a `BREAKING CHANGE:` footer
listing the removed binding names, the renamed binary and crate, the emitted key
changes, the diagnostic-code renames, and the schema bumps. The host writes that message
from the final declaration. Do not commit by hand.

## sase, same turn

Read `sase/memory/lint_and_test.md` with `sase memory read` before finishing sase edits.
Do not hand-edit `sase-core-revision.txt` and do not run `just ratchet-core-revision`. A
declaration that commits sase and sase-core makes the host commit sase-core first and
write that SHA into the pin (`docs/rust_backend.md`, `repos.linked[].revision_pin`).

### Build tooling

Drop crate-name detection and always build `sase_macro_lsp`:

- `Justfile` (`rust-lsp-install` and the second `lsp_pkg` block)
- `src/sase/dev_update/prebuild_cache.py` (`lsp_package_name`)
- `.github/workflows/build-core.yml`
- `.github/workflows/master-gate.yml`
- `.github/actions/setup-sase/action.yml`
- `tests/dev_update/test_prebuild.py`

Leave the flag-gated PATH fallback from `sase-macro-lsp` to `sase-xprompt-lsp` in place.
That fallback belongs to the sunset flag, not to crate detection.

### Mirrors and adapters

Update the Python schema mirrors and `tools/validate_sase_core_rs` to the bumped
constants. The validator still writes an `xprompts.json` fixture and asserts
`payload.get("xprompts")`. Point it at `macros.json` and `macros`.

`src/sase/core/health.py` must stay green against the new core. Update it if it still
probes a legacy key.

Flip the pre-flip adapters sase-1eq.5.1 left for this phase. sase-1eq.10 note 1 names
them:

- `src/sase/macro/jinja_assist.py`: `JinjaScopeKind` becomes
  `Literal["prompt", "macro"]`. Callers stop sending `xprompt`. The core request no
  longer accepts it.
- `src/sase/ace/tui/widgets/_argument_syntax_editing.py` calls
  `macro_completion_spacer_to_parentheses_edit` directly. Remove
  `require_legacy_xprompt_completion_spacer_binding` if nothing else needs it.
- `src/sase/macro/highlight.py`: argument source `macro` is recomputed, not a stored
  file. Drop `"xprompt"` from `_SOURCES` unless a persisted fixture still contains it.
  If one does, read it through `legacy_xprompt_names.py` and keep that legacy-input
  test.
- `tests/perf/bench_detail_header_summary.py` still lists the span suffix
  `xprompts_used`. The producer in
  `src/sase/ace/tui/widgets/prompt_panel/_agent_display_header_summary.py` already
  traces `macros_used`. Update the bench suffix and delete the pair row in
  `tests/_macro_terminology_string_pairs_b_early.py`.
- `src/sase/stats/_view_builders.py` still falls back to legacy stats keys. Read only
  the new keys.

Remove the `SASE_AGENT_LOCAL_XPROMPTS` fallback in
`src/sase/agent/multi_prompt_macros.py`. Remove any remaining
`SASE_LAUNCH_SWARM_XPROMPTS` dual-write in `src/sase/macro/used_macros.py`. Launch code
sets only `SASE_AGENT_LOCAL_MACROS` and `SASE_LAUNCH_SWARM_MACROS`.

Update doc lines that still name `crates/sase_xprompt_lsp` or `sase-xprompt-lsp` as the
crate sase builds. Do not rewrite history, changelogs, or decision records.

### Terminology guard

`tests/_macro_terminology_strings.py` allowlists files as "owned by sase-1eq.10". After
the wire flip, those files must not contain pre-flip wire vocabulary. Drop their rows,
and the identifier-file comments that point at this phase, once the hits are gone. Do
not widen the guard past that list.

Known owned files include the agent launch and swarm modules, artifact-ref wire modules,
`content_layout_wire.py`, `macro_skill_definition_facade.py`,
`integrations/macro_lsp.py`, `jinja_assist.py`, `used_macros.py`, `pager/link_scan.py`,
the snippet modules, `stats/_view_builders.py`, and the tests named beside those rows.
Re-read the dict. It is the list.

Keep every durable legacy reader and `tests/test_legacy_xprompt_names.py`.

## chezmoi, same turn

Open the linked repo with `sase repo open chezmoi` and read its `AGENTS.md` before
editing. Touch only the LSP installer:

- In `home/bin/executable_install_sase_github`, rename `install_xprompt_lsp` to
  `install_macro_lsp`.
- Probe with `command -v sase-macro-lsp` in `probe_uv_tool_health` too, not only in the
  install function.
- Install with
  `cargo install --path "$SASE_CORE_DIR/crates/sase_macro_lsp" --bin sase-macro-lsp --locked --force`.
- When `command -v sase-xprompt-lsp` succeeds, run `cargo uninstall sase_xprompt_lsp`
  before the new install.
- Update `tests/bash/install_sase_github_test.sh` to match.
- Run chezmoi's check (`just check`, or the narrower bash test if the full check cannot
  run). Follow that repo's `AGENTS.md` for how agents invoke it. The host commit is what
  `chezmoi update -a --force` applies. Do not apply uncommitted source as a substitute
  for the check.

Do not rename chezmoi macro directories or config keys in this tale.

## Verify, then declare

1. In sase-core, `sase tool run check`. Give it at least 10 minutes. A passing targeted
   `just test -p` does not replace it.
2. Build that dirty core into this sase workspace (`just install`) and run
   `sase tool run check`. Run `just fix` first so formatting does not fail the gate.
   Confirm `sase core health` is green.
3. Run the chezmoi check from the step above.
4. Declare all three repos in the turn finalizer. sase-core's message carries the
   `feat!:` subject and `BREAKING CHANGE:` footer. Do not commit by hand and do not
   write a guessed SHA into `sase-core-revision.txt`.
5. Before closing sase-1eq.10, run `sase bead epic-symbols sase-1eq.10`. If any
   `--epic-symbol` row still names this phase, resolve it or re-key the Justfile line to
   a still-open bead (sase-1eq or sase-1eq.11). `sase bead close` refuses while
   leftovers remain.
6. Close only sase-1eq.10, with a note that says what was verified.

A check failure that reproduces on the clean base tree does not keep the bead open.
Record it as `PROPOSED FOLLOW-UP:` on sase-1eq.10, cite any task bead that already
tracks it, and close. Do not set bead status by hand.

## Exit

sase-core emits macro spellings and rejects the four legacy binding names plus the
legacy spacer name. `sase-macro-lsp` is the only LSP binary in the renamed crate. sase's
mirrors, build scripts, and the listed adapters match that output. chezmoi installs
`sase-macro-lsp` from `crates/sase_macro_lsp`. Each repo's check passed, or a clean-base
failure is recorded as a follow-up. `sase core health` is green. The host pin move is
part of the three-repo declaration, not a later ratchet.
