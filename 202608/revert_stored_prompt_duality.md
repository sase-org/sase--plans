---
tier: epic
title: Revert stored prompt duality and xprompt linkification
goal: 'Chat markdown and published prompt archive entries store exactly what they
  stored before sase-e6 — one `## Prompt` section in a chat, one verbatim body in
  an archive entry — every already-written file in the sase-e6 format is rewritten
  to the pre-sase-e6 format, and no code anywhere in sase or sase-core knows the sase-e6
  format exists.

  '
phases:
- id: chat
  title: Chat markdown returns to a single Prompt section
  depends_on: []
  size: medium
  description: 'chat: strip the sentinel-delimited XPrompt and rendered-prompt sections
    out of the chat writer, drop the two keyword arguments from every `save_chat_history`
    caller, remove the parser''s strip pass, and delete the `chat_history.rendered_prompt_max_bytes`
    config field.

    '
- id: archive
  title: Prompt archive publishes only the verbatim body
  depends_on: []
  size: medium
  description: 'archive: remove the appended rendered-prompt section and the xprompt
    link rewrite from prompt archive rendering and preparation, drop the sentinel/fence/link-target
    checks and the legacy counter from validation, and restore the sidecar agents
    README template.

    '
- id: surfaces
  title: Read surfaces and documentation
  depends_on:
  - chat
  - archive
  size: medium
  description: 'surfaces: revert the ACE Chats detail pane, the `sase chat show` and
    `sase agent prompts show` rendering selectors, and prompt search''s section stripping,
    then delete `chat_prompt_sections.py` and remove every documentation paragraph
    describing the two stored renderings.

    '
- id: provenance
  title: Launch-time provenance capture removal
  depends_on:
  - chat
  - archive
  - surfaces
  size: small
  description: 'provenance: stop writing `xprompt_sources.json` at launch, reduce
    the source-collection and hosted-URL modules to exactly the definition-resolution
    surface `sase xprompt show` calls, and delete the record loading and link rewriting
    helpers that only the reverted stores used.

    '
- id: core
  title: Rust prompt_xprompt module removal
  depends_on:
  - surfaces
  - provenance
  size: small
  description: 'core: delete the `prompt_xprompt` module and its three PyO3 bindings
    from the sibling Rust core repository while keeping the shared `prompt_rewrite`
    helper the artifact rewriter still uses.

    '
- id: migrate
  title: One-shot rewrite of stored files
  depends_on:
  - chat
  - archive
  size: medium
  description: 'migrate: rewrite every already-stored chat transcript and archived
    prompt entry back to the pre-sase-e6 format with a throwaway tool, commit and
    push the affected agents sidecars, delete the orphaned provenance artifacts, and
    then delete the tool itself.'
proposed_by: bbugyi200.athena.sase-ej.land.w2
create_time: 2026-08-03 14:48:17
status: wip
bead_id: sase-f2
---

- **PROMPT:** [prompts/202608/revert_stored_prompt_duality.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/revert_stored_prompt_duality.md)
- **BEAD:** [sase-f2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-f2/README.md)

# Plan: Revert stored prompt duality and xprompt linkification

## Background

Epic `sase-e6` ("Store both prompt renderings and linkify xprompt references") shipped in six phases on 2026-08-02. It
changed what SASE writes into two durable stores and added the machinery behind that change:

| Phase       | Commit           | What it added                                                               |
| ----------- | ---------------- | --------------------------------------------------------------------------- |
| `sase-e6.1` | `4d83afb` (core) | `prompt_xprompt.rs` + three PyO3 bindings; extracted `prompt_rewrite.rs`    |
| `sase-e6.2` | `cb90eaf00`      | launch-time `xprompt_sources.json` capture                                  |
| `sase-e6.3` | `e30935808`      | `src/sase/xprompt_links.py` hosted-URL resolution                           |
| `sase-e6.4` | `e6624e324`      | sentinel-delimited chat sections + `chat_history.rendered_prompt_max_bytes` |
| `sase-e6.5` | `f578c0aa4`      | appended rendered section and xprompt links in published prompt archives    |
| `sase-e6.6` | `e3ca2c11c`      | ACE / CLI read surfaces and documentation                                   |

Three later commits by other agents built on top of it and are reverted here as well, because they exist only to serve
the sase-e6 format: `09bedcef0` (prompt search strips the stored sections), and the sase-e6 paragraphs added by the
documentation refresh commits `39ef28e01` and `fe0d71e09`.

The user wants all of this reverted. Chats go back to one `## Prompt` section; archived prompts go back to a single
verbatim body. Already-written files in the sase-e6 format must be rewritten, and no code that understands the sase-e6
format may survive anywhere.

### What the current stores look like

A chat written since `sase-e6.4` interleaves two HTML-comment-delimited regions between the metadata block and
`## Prompt`:

````markdown
- **MODEL:** codex/gpt-5.6-sol

<!-- sase:section:xprompt -->

## Agent XPrompt

#gh:gh_sase-org__sase %model:@epic_lander

<!-- /sase:section:xprompt -->

<!-- sase:section:rendered -->

<details>
<summary><b>Agent Prompt</b> — rendered, 2.4 KB</summary>

```markdown
…verbatim rendered prompt…
```

</details>

<!-- /sase:section:rendered -->

## Prompt
````

A published archive entry keeps its `PLAN` / `AGENTS` / `ARTIFACTS` header block and verbatim body, then appends only
the `<!-- sase:section:rendered -->` region, and its body may carry `[#ref](https://…)` links where the author typed a
bare `#ref`.

Survey of what exists today (counts taken while planning; re-derive them at migration time rather than trusting these
numbers):

- `~/.sase/chats/**/*.md`: 263 of 12,690 transcripts carry a sentinel (176 with an XPrompt section, 262 with a rendered
  section).
- `~/.sase/projects/gh_sase-org__sase/repos/agents/prompts/`: 39 of 2,984 entries carry the rendered section, and
  exactly one carries linkified `#` references.
- `~/.sase/projects/gh_bobs-org__bob-cli/repos/agents/prompts/`: 11 of 12 entries carry the rendered section.
- 109 orphan `xprompt_sources.json` artifacts under `~/.sase`.

### Scope boundary: `sase xprompt show` keeps its provenance link

The closed epic `sase-eb` (`sase xprompt show`) was built after sase-e6 and calls into two sase-e6 modules to render a
hosted URL for the definition it is showing. `src/sase/xprompt/cli_show_resolve.py` imports `definition_file_for_source`
and `definition_line_for` from `src/sase/xprompt/xprompt_sources.py`, calls `write_xprompt_sources(None, …)` to collect
one record without writing a file, and builds an `XpromptTargetResolver` from `src/sase/xprompt_links.py`.

This plan therefore reverts every sase-e6 behavior while keeping exactly the definition-resolution surface
`sase xprompt show` calls. It does **not** revert sase-eb, which the user did not ask for. Concretely, the `provenance`
phase keeps repository/chezmoi resolution, YAML line anchors, `XpromptSourceRecord`, and `XpromptTargetResolver`; it
deletes the launch-time artifact, the record loader, and the link rewriter. If the reviewer would rather remove the
provenance link from `sase xprompt show` too, say so at approval and the `provenance` phase collapses into deleting both
modules outright.

Nothing else outside sase-e6 depends on this machinery. `scan_xprompt_references()` in
`src/sase/xprompt/used_xprompts.py` was factored out by `sase-e6.2` and is now shared with sase-eb, so that refactor
stays. `crates/sase_core/src/prompt_rewrite.rs` was factored out by `sase-e6.1` and is still the shared helper behind
`prompt_artifact.rs`, so it stays too.

### Verification discipline

Workspace virtual environments are ephemeral, so every phase runs `just install` first and `just check` before
finishing. Phases that change ACE rendering also run `just test-visual`. The `core` phase additionally runs the Rust
repository's own `cargo fmt --all -- --check`, `cargo clippy --workspace --all-targets -- -D warnings`, and
`cargo test --workspace`.

## Phase: Chat markdown returns to a single Prompt section

Restore `write_chat_history()` output to exactly the pre-sase-e6 byte sequence: title, metadata block, optional extra
sections, optional previous conversation, `## Prompt`, `## Response`.

In `src/sase/history/chat_storage.py`, drop the `xprompt_prompt` and `rendered_prompt` keyword-only parameters, the
`render_prompt_sections` import, the block that appends `prompt_sections`, and the `read_xprompt_prompt()` helper that
sase-e6.4 added. In `src/sase/history/chat.py`, drop the same two parameters from `save_chat_history()` and their
pass-through. In `src/sase/history/chat_resume.py`, remove the `strip_prompt_sections` import and both calls in
`parse_chat_turns()` and `extract_previous_conversation_turns()`.

Update every caller so no call site mentions either rendering:

- `src/sase/axe/run_agent_exec_finalize.py`: remove the `_load_prompt_renderings` and `rewrite_xprompt_links` imports,
  the `_prompt_renderings()` wrapper, and the two `save_chat_history` arguments.
- `src/sase/axe/run_agent_exec_finalize_chat.py`: delete `load_prompt_renderings()`, `rewrite_xprompt_links()`, the
  `XpromptLinker` alias, and the `_read_text()` helper that only they used.
- `src/sase/axe/run_agent_exec_plan.py` and `src/sase/axe/run_agent_exec_questions.py`: remove the `read_xprompt_prompt`
  imports and both arguments.
- `src/sase/llm_provider/postprocessing.py`, `src/sase/ace/handlers/workflow_handlers.py`,
  `src/sase/ace/workflows/crs.py`, `src/sase/workspace_provider/changespec.py`, and
  `src/sase/xprompt/workflow_executor_steps_prompt.py`: remove the arguments they pass.

Remove the config field sase-e6.4 added: `chat_history.rendered_prompt_max_bytes` in `src/sase/default_config.yml` and
`src/sase/config/sase.schema.json`, `get_chat_rendered_prompt_max_bytes()` in `src/sase/config/core.py`, and its
re-export in `src/sase/config/__init__.py`. The field is not set in any user configuration, so no migration is required,
but confirm that before deleting the schema entry. Its row in `docs/configuration.md` is removed by the `surfaces` phase
along with the rest of the documentation.

`src/sase/history/chat_prompt_sections.py` stays in place for now; the `archive` and `surfaces` phases are its remaining
importers and `surfaces` deletes it.

Tests: remove the sase-e6 cases from `tests/test_axe_run_agent_exec_finalize_metadata.py`,
`tests/test_chat_history_config.py`, `tests/test_llm_provider_postprocessing.py`, and
`tests/main/chat_handler_helpers.py`. Add or keep a test asserting that a freshly written chat contains no
`<!-- sase:section:` sentinel and that its turns round-trip through `parse_chat_turns()`.

## Phase: Prompt archive publishes only the verbatim body

Restore `render_prompt_document()` in `src/sase/agents_sync/prompt_archive/render.py` to producing the body with `@`
artifact links plus the `PLAN` / `AGENTS` / `ARTIFACTS` header sections and nothing else. Drop the `xprompt_records`,
`xprompt_target`, and `rendered_prompt` parameters, the `prompt_xprompt_rewrite_links` call, the
`linked_xprompt_records` field on `RenderedPromptArchive`, the `XpromptTargetResolver` type alias and its
`TYPE_CHECKING` import, and the `render_prompt_sections` import and call.

In `src/sase/agents_sync/prompt_archive/preparation.py`, drop the `XpromptTargetResolver` and
`load_xprompt_source_records` import, the resolver construction, the `xprompt_records` load, and
`_read_rendered_prompt()` together with its `ArtifactFileCache` import if nothing else in the module needs it.

In `src/sase/agents_sync/prompt_archive/validation.py`, remove `_validate_prompt_sections()`,
`_rendered_fence_is_balanced()`, `_validate_xprompt_link_targets()`, the `_RENDERED_FENCE_OPEN_RE` pattern, the four
sentinel imports, the `legacy_files` field and its JSON key, and their call sites. Restore
`_markdown_targets_outside_fences()` to the pre-sase-e6 single-function form that tracks fences by their first character
and returns targets only — the label capture and the fence-length comparison exist solely to tolerate the dynamic fences
sase-e6.5 introduced. In `src/sase/agents/cli_prompts.py`, remove the `legacy_summary` text from `_print_validation()`.

Restore the archive description in `src/sase/sdd/templates/sidecar-agents-README.md` to the single-body wording, then
regenerate the derived sidecar READMEs with `sase init repo` and verify the check mode is clean.

Tests: remove the sase-e6 cases from `tests/agents_sync/test_prompt_archive.py` and
`tests/agents_sync/test_prompt_archive_validation.py`. Keep or add a test asserting a published document contains no
`<!-- sase:section:` sentinel and that a body reference such as `#plan` is published verbatim.

## Phase: Read surfaces and documentation

Revert the ACE Chats detail pane in `src/sase/ace/tui/widgets/artifacts/chats_detail.py`: restore
`_read_transcript_preview()` to a plain bounded line read returning `(preview, truncated)`, remove the `xprompt_prompt`
and `rendered_prompt` fields from `ChatDetailData`, the `PROMPTS` heading block in `build_chat_detail()`,
`_prompt_rendering_section()`, `_PROMPT_RENDERING_PREVIEW_LINES`, and every `chat_prompt_sections` import.

Revert both CLI read surfaces:

- `src/sase/chat/cli_show.py` and `src/sase/main/parser_chat.py`: restore `--format` to `("raw", "resume", "response")`,
  delete the `-r/--rendered` and `-x/--xprompt` shortcuts, and delete `_print_prompt_rendering()`.
- `src/sase/agents/cli_prompts.py` and `src/sase/main/parser_agent.py`: delete the `-r/--rendered` option and restore
  `_handle_show()` to printing the whole document — `console.print(file.relpath)` followed by
  `Syntax(content, "markdown", word_wrap=True)`, with `payload["content"] = content` in JSON mode — and delete
  `_document_body()`, `_normalize_printable_prompt()`, and `_write_prompt_content()`. Note for the reviewer: this
  restores Rich-highlighted whole-document output and gives up the raw-to-stdout behavior sase-e6.6 introduced, which is
  the correct full revert but is a visible CLI change beyond the storage format.

Revert `src/sase/prompt/search/sources.py` to its state before `09bedcef0`: remove the `strip_prompt_sections` import
and call and the `_LINKED_XPROMPT_RE` normalization, and restore the digest-based cross-store dedup and its module
docstring. Both exist only because archives carried machine-generated sections and linkified references.

Delete `src/sase/history/chat_prompt_sections.py` and `tests/history/test_chat_prompt_sections.py`. Confirm with a
repository-wide search that no importer remains.

Documentation — remove every paragraph, table row, and list entry describing the two stored renderings, the
`xprompt_sources.json` artifact, the hosted xprompt links, or the CLI selectors:

- `docs/xprompt.md`: the whole "Stored Prompt Renderings" section.
- `docs/ace.md`: the Chats detail prompt-rendering paragraph.
- `docs/agent_images.md`: the `xprompt_sources.json` bullet and the linkified/rendered archive paragraph.
- `docs/agents_sidecar.md`: the rendered-prompt paragraph and the "become hosted links" clause.
- `docs/sdd.md`: the rendered-prompt paragraph, the `#...` link sentence, and the `--rendered` sentence.
- `docs/cli.md`: the `sase agent prompts` row and any `sase chat show` format list mentioning the selectors.
- `docs/prompt.md`: the `show --rendered` paragraph and any search-stripping wording from `09bedcef0`.
- `docs/configuration.md`: the `chat_history.rendered_prompt_max_bytes` row.

Do **not** edit anything under `sase/memory/` or any generated agent instruction file. A survey during planning found no
sase-e6 references there, but if one turns up, record it with `/sase_new_task` instead of editing.

Tests: update `tests/ace/tui/test_artifacts_chats_detail.py`, `tests/main/test_agent_prompts_handler.py`,
`tests/main/test_chat_handler.py`, and `tests/prompt_command/test_search_sources.py`. Run `just test-visual` and accept
a snapshot change only if it is an intended consequence of removing the `PROMPTS` block.

## Phase: Launch-time provenance capture removal

Stop capturing provenance at launch. In `src/sase/axe/run_agent_runner_setup.py`, remove the `write_xprompt_sources`
import and call from `preprocess_prompt_xprompts()`, leaving the existing `write_used_xprompts()` call and its
best-effort `try` discipline untouched.

Reduce `src/sase/xprompt/xprompt_sources.py` to the collection surface `sase xprompt show` needs. Replace
`write_xprompt_sources()` with a `collect_xprompt_sources()` that returns records and writes nothing, keep
`definition_file_for_source()`, `definition_line_for()`, the repository/chezmoi resolution, and the record shape, drop
the now-unused `json` and `os` imports, retire the private `_resolve_definition_line` and `_definition_file_for_source`
aliases if nothing references them, and update `__all__` and the module docstring so it no longer claims to capture
launch-time provenance. Update the single call site in `src/sase/xprompt/cli_show_resolve.py`.

In `src/sase/xprompt_links.py`, delete `load_xprompt_source_records()` and `rewrite_xprompt_source_links()`, the
`require_rust_binding` import, and the `Mapping`/`Sequence`/`Any` imports they used. Keep `XpromptSourceRecord` and
`XpromptTargetResolver`, and reword the module and class docstrings so they describe resolving one xprompt definition
record rather than launch-captured provenance.

Confirm no `require_rust_binding("prompt_xprompt_…")` call site remains anywhere in `src/sase`; the `core` phase depends
on that being true.

Tests: rework `tests/test_xprompt_sources.py` around the collector (dropping the artifact-serialization case), remove
the provenance cases from `tests/test_run_agent_runner_setup.py`, and trim `tests/test_xprompt_links.py` to the
resolver. Keep `tests/xprompt/test_cli_show_resolve.py` green. Verify by hand that `sase xprompt show` still prints a
hosted definition URL for a project-defined and a chezmoi-defined xprompt.

## Phase: Rust prompt_xprompt module removal

Work in the sibling Rust core repository. Open it with the `/sase_repo` skill and use only the path that skill prints.

Delete `crates/sase_core/src/prompt_xprompt.rs` and its `pub mod prompt_xprompt;` line in `crates/sase_core/src/lib.rs`.
In `crates/sase_core_py/src/lib.rs`, delete the `py_prompt_xprompt_records_parse`, `py_prompt_xprompt_records_select`,
and `py_prompt_xprompt_rewrite_links` functions, the `prompt_xprompt_records_from_py_list` helper, their three
`wrap_pyfunction!` registrations, their three module-doc binding lines, the `sase_core::prompt_xprompt` import, and the
`prompt_xprompt_bindings_round_trip_record_shapes` test.

Keep `crates/sase_core/src/prompt_rewrite.rs`. `sase-e6.1` extracted it from `prompt_artifact.rs`, and
`prompt_artifact.rs` still imports `rewrite_prompt_links` and `PromptLinkCandidate` from it; removing it would undo a
refactor of the surviving artifact rewriter.

Run the Rust repository's own gates before finishing: `cargo fmt --all -- --check`,
`cargo clippy --workspace --all-targets -- -D warnings`, and `cargo test --workspace`.

Back in the sase repository, no `sase-core-rs` version floor change is needed: removing bindings adds no requirement,
and `tools/check_sase_core_rs_bindings` passes because it scans `require_rust_binding` call sites and none name a
`prompt_xprompt_*` binding after the `provenance` phase. Confirm that gate is green through `just check`.

## Phase: One-shot rewrite of stored files

Rewrite every file already written in the sase-e6 format, then delete the tool that did it so no format-aware code
survives.

Add a throwaway `tools/revert_stored_prompt_sections` script with focused tests, supporting `--dry-run` (default) and
`--write`, and reporting per-store counts.

**Chat transcripts.** For every `*.md` under `~/.sase/chats` (all shard, legacy, and imported layouts — reuse
`iter_chat_files()` rather than reimplementing discovery), remove the `<!-- sase:section:xprompt -->` and
`<!-- sase:section:rendered -->` regions including their sentinels. `write_chat_history()` inserted them as
`"\n" + sections` immediately before `"\n## Prompt\n\n"`, so removing that exact inserted run restores the pre-sase-e6
bytes; normalize the surviving separator to exactly one blank line before `## Prompt` and assert the result parses to
the same turns as the original did through `parse_chat_turns()`. Files without sentinels are left byte-identical.

**Archived prompts.** For every registered project (enumerate with `sase project list --json`; at least the `sase` and
`bob-cli` sidecars are affected), for every `prompts/<YYYYMM>/*.md` in its agents sidecar:

1. Remove the `<!-- sase:section:rendered -->` region and the blank-line padding that preceded it.
2. In the document body only — never inside the `PLAN` / `AGENTS` / `ARTIFACTS` header block — rewrite
   `[#ref](https://…)` back to the bare `#ref` the author typed. Only one entry in the survey needed this; diff it by
   hand before writing.
3. Re-normalize with `format_with_prettier()` so the result matches what the reverted renderer would emit.

Commit the rewritten prompts in each agents sidecar under `store_git_write_lock` and push, so the published copies
match. Then delete the 109 orphaned `xprompt_sources.json` artifacts under `~/.sase`, which nothing reads after the
`provenance` phase.

**Verification.** A repository-wide search for `sase:section:` returns nothing under `~/.sase/chats` or any agents
sidecar; `sase agent prompts validate` passes for every affected project; `sase chat show -f resume` and
`sase chat show -f response` succeed on a migrated transcript; and one migrated archive entry renders correctly on
GitHub.

**Cleanup.** Delete `tools/revert_stored_prompt_sections` and its tests in this phase's final commit. Recovery is
available from git history if a re-run is ever needed, and nothing that understands the sase-e6 format remains at HEAD.

## Non-goals

- `sase xprompt show` keeps its hosted definition link. Reverting sase-eb was not requested; see the scope boundary
  above.
- `scan_xprompt_references()` in `src/sase/xprompt/used_xprompts.py` and `crates/sase_core/src/prompt_rewrite.rs` stay:
  both are shared refactors that survive their sase-e6 callers.
- `xprompts.json`, `used_xprompts`, and `prompt-artifacts.jsonl` are untouched. Only `xprompt_sources.json` goes away.
- `@` artifact reference linkification in published prompts is unchanged; it predates sase-e6.
- `## Prompt` / `## Response` semantics, resume, fork, and chat catalog behavior are unchanged — they were already
  unchanged by sase-e6 and stay that way.
- No file under `sase/memory/` and no generated agent instruction file is edited.

## Verification

Every phase runs `just install` before anything else, then `just check` before finishing. `surfaces` additionally runs
`just test-visual`. `core` additionally runs the Rust repository's format, clippy, and test gates. `migrate`
additionally runs `sase agent prompts validate` for every affected project and confirms a zero-hit search for
`sase:section:` across `~/.sase/chats` and every agents sidecar.
