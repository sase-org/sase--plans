---
tier: epic
title: Store both prompt renderings and linkify xprompt references
goal: "Every SASE agent chat markdown file and every published prompt archive entry stores both the unrendered XPrompt
  prompt and the rendered agent prompt, and every resolvable xprompt reference inside an unrendered prompt renders as a
  Markdown hyperlink to the hosted file that defines it, including chezmoi-managed definitions when `use_chezmoi: true`.

  "
phases:
  - id: core
    title: Rust xprompt-source wire and reference link rewriting
    depends_on: []
    size: medium
    description: "core: add the `prompt_xprompt` module to sase_core with the launch-capture record wire,
      newest-per-reference selection, and a boundary-aware reference rewriter that shares the artifact rewriter's
      literal-zone and Markdown-link protection, plus the sase_core_py bindings and Rust tests.

      "
  - id: capture
    title: Launch-time capture of xprompt definition provenance
    depends_on:
      - core
    size: medium
    description: "capture: write the per-run `xprompt_sources.json` artifact recording each surviving reference's exact
      token and its owning repository, repo-relative path, chezmoi remapping, and definition line, resolved best-effort
      so a launch never fails because provenance was unavailable.

      "
  - id: links
    title: Hosted URL resolution for xprompt definitions
    depends_on:
      - core
      - capture
    size: small
    description: "links: add the resolver that turns one captured provenance record into a hosted blob URL with a line
      anchor, pinning the primary repository at the publication revision and returning nothing rather than guessing.

      "
  - id: chat
    title: Chat markdown stores both prompt renderings
    depends_on:
      - links
    size: medium
    description: "chat: extend the chat writer with sentinel-delimited XPrompt and rendered-prompt sections, harden turn
      parsing against them, update every `save_chat_history` caller to supply both renderings, and linkify references in
      the stored XPrompt section.

      "
  - id: archive
    title: Prompt archive stores both prompt renderings
    depends_on:
      - links
    size: medium
    description: "archive: linkify xprompt references in the published prompt body, append the rendered prompt as a
      collapsed verbatim section, and extend prompt archive validation and the sidecar README template to cover both.

      "
  - id: surfaces
    title: Read surfaces, docs, and end-to-end verification
    depends_on:
      - chat
      - archive
    size: small
    description:
      "surfaces: teach the ACE chat detail view and the `sase agent prompts show` / `sase chat show` commands about the
      two renderings, update the documentation, and verify the whole path end to end."
proposed_by: bbugyi200.athena.rs
create_time: 2026-08-02 09:22:19
status: wip
---

- **PROMPT:**
  [prompts/202608/stored_prompt_duality.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/stored_prompt_duality.md)

# Plan: Store both prompt renderings and linkify xprompt references

## Background

A SASE agent run produces three distinct prompt texts. Two of them are the ones the agent metadata panel shows on the
ACE agents tab:

| Panel section   | Artifact                                 | Meaning                                                                      |
| --------------- | ---------------------------------------- | ---------------------------------------------------------------------------- |
| `AGENT XPROMPT` | `<artifacts_dir>/raw_xprompt.md`         | Alias-resolved but **unexpanded**: `#foo` references are still literal text. |
| `AGENT PROMPT`  | `<artifacts_dir>/<agent_type>_prompt.md` | The **rendered** text actually handed to the model.                          |

The third is `LoopState.current_prompt`, an intermediate that exists only inside the execution loop.

Today's storage is lopsided, and each store keeps a different one:

- **Chat markdown** (`~/.sase/chats/<YYYYMM>/*.md`) has a single `## Prompt` section written by `write_chat_history()`
  in `src/sase/history/chat_storage.py`. `finalize_loop()` in `src/sase/axe/run_agent_exec_finalize.py` fills it with
  `state.current_prompt`, while `src/sase/llm_provider/postprocessing.py` fills it with the rendered prompt. The two
  producers therefore disagree about what `## Prompt` means, and neither store guarantees the panel's two renderings
  survive artifact retention.
- **Prompt archive markdown** (`<project>--agents` sidecar, `prompts/<YYYYMM>/<name>.md`) stores exactly
  `raw_xprompt.md` and nothing else. `prepare_prompt_archive()` in `src/sase/agents_sync/prompt_archive/preparation.py`
  reads that file directly when no explicit `prompt_content` is supplied, which is what the commit-time publication path
  in `src/sase/workflows/commit/workflow_publication.py` always does.

Neither store linkifies xprompt references, even though the published archive already linkifies `@` artifact references.
A survey of the sase project's published prompt archive shows references survive publication in volume (`#sshot`,
`#plan`, `#commit`, `#chop`, `#blog`, `#read`, `#research`, `#epic`, `#m_codex`, `#sase/pysplit`, …), so every one of
those is a dead end for a reader who wants to know what the agent was actually told.

The `@` reference pipeline is the model to copy. It captures provenance at launch (`stage_prompt_artifact()` in
`src/sase/core/prompt_artifact_staging.py` writes `prompt-artifacts.jsonl`), resolves targets at publication
(`_ArtifactTargetResolver` in `preparation.py`), and rewrites the text with a Rust-owned scanner
(`rewrite_prompt_artifact_links` in `crates/sase_core/src/prompt_artifact.rs`, bound as
`prompt_artifact_rewrite_links`). This plan builds the xprompt equivalent alongside it and reuses the scanner's
protection logic rather than reimplementing it.

## Design

### Naming

Use these two names everywhere — code, sections, help text, docs:

- **XPrompt prompt** — the unrendered text, `raw_xprompt.md`, the `AGENT XPROMPT` panel content.
- **rendered prompt** — the final text sent to the model, the `AGENT PROMPT` panel content, selected with
  `ArtifactFilesCache.select_prompt_file()` in `src/sase/agent/artifact_files_cache.py` (which already excludes commit
  finalizer follow-up prompts and honors workflow-child step names).

### Governing principle for links

A reference is linkified only when a hosted URL can be resolved for its definition. Anything unresolvable is left
exactly as the user typed it. There is never a broken link, never an invented revision, and never a placeholder. This
mirrors `_ArtifactTargetResolver`, which returns `None` to leave a reference alone.

### Where provenance comes from

Definitions must be resolved on the machine and at the moment the run launches, because publication happens later (at
commit time) and may happen after the user has edited or moved a definition. `XPrompt.source_path` and
`Workflow.source_path` (see `src/sase/xprompt/models.py`) already carry the resolved definition location, and
`collect_used_xprompts()` in `src/sase/xprompt/used_xprompts.py` already scans the raw prompt with the shared lexical
parser, applying alias normalization and skipping literal zones. Capture extends that existing scan.

The existing `xprompts.json` artifact is deliberately **not** overloaded: its records are parsed by the Rust agent
scanner (`crates/sase_core/src/agent_scan/scanner.rs`) and projected into agent rows, so a new sibling artifact keeps
that hot path untouched.

### Repository and chezmoi mapping

Given a definition path, the owning repository is found by remapping first and matching second:

1. If `get_use_chezmoi()` is true and the path is under the user's home, remap it with `chezmoi_source_path()` from
   `src/sase/content_layout.py` (`~/.config/sase/sase.yml` → `<chezmoi source>/home/dot_config/sase/sase.yml`,
   `~/sase/xprompts/foo.md` → `<chezmoi source>/home/sase/xprompts/foo.md`). Record `chezmoi: true`.
2. Match the resulting path against the roots from `collect_repo_inventory()` in `src/sase/repo_inventory.py`, choosing
   the deepest containing root. This covers the project's primary repo, its sidecars, every configured linked repo
   (including `chezmoi`, `sase-github`, `sase-telegram`, and `sase-nvim`), and other registered projects. Record the
   repo name, the repo root, and the POSIX repo-relative path.
3. If nothing owns the path, record the reason and no repository. That reference will never be linkified.

`use_chezmoi: false` therefore yields no link for a home-config definition, which is correct: those bytes are not
published anywhere.

### Definition line anchors

A `.md` xprompt file is the definition, so a file-level link is complete. A YAML-defined xprompt (the common case for
`~/.config/sase/sase.yml` entries such as `#plan`) is one key inside a large file, so a file-level link is close to
useless. Capture therefore records a 1-based `definition_line` for YAML-sourced definitions, and the resolver appends
`#L<line>` to the blob URL. When the line cannot be determined the link degrades to a file-level link rather than
guessing.

### Rust ownership

Reference scanning and link rewriting are shared backend behavior — a web view of an agent page would need identical
output — so they belong in the sibling Rust core repository per this repo's Rust core boundary rule. The Python side
keeps only filesystem work, repo inventory lookups, and URL assembly.

### Chat markdown format

The new sections are written **before** `## Prompt`, because `parse_chat_turns()` in `src/sase/history/chat_resume.py`
pairs each `Prompt` heading with the next `Response` heading and treats everything between them as the prompt body.
Nothing may be inserted between those two headings, and nothing may be appended after `## Response` (the response body
runs to the next turn heading or to end of file).

Both new sections are wrapped in HTML-comment sentinels so the parser can excise them before it looks for turn headings.
This matters because an unfenced verbatim prompt could itself contain a line that reads exactly `## Prompt` — a
realistic case for any prompt that quotes a chat transcript.

````markdown
# Chat History - ace-run (my.agent)

- **TIMESTAMP:** 2026-08-02 09:14:03 EDT
- **MODEL:** claude/claude-opus-5
- **AGENT:** my.agent

<!-- sase:section:xprompt -->

## Agent XPrompt

[#gh:sase](https://github.com/sase-org/sase/blob/<sha>/src/sase/default_xprompts/gh.md) Can you help me …
[#plan](https://github.com/<owner>/<dotfiles>/blob/<sha>/home/dot_config/sase/sase.yml#L1029)

<!-- /sase:section:xprompt -->
<!-- sase:section:rendered -->

<details>
<summary><b>Agent Prompt</b> — rendered, 14.2 KB</summary>

```markdown
…verbatim rendered prompt…
```

</details>

<!-- /sase:section:rendered -->

## Prompt

…unchanged…

## Response

…unchanged…
````

Reliability rules for the format:

- The XPrompt section is **unfenced** so its hyperlinks are live on GitHub.
- The rendered section is **fenced**, using a backtick run one longer than the longest run found in its content (minimum
  three), so embedded fences and headings can never leak. It carries no links, which is exactly what the requirement
  asks for: linkification applies to unrendered prompts.
- Both payloads are escaped on write so a literal sentinel string inside a prompt cannot terminate its own section.
- `## Prompt` and `## Response` keep their current contents and meaning. Resume, fork, chat catalog, and search paths
  are untouched, so `sanitize_resume_prompt()` never sees a Markdown link where it expects a bare reference.
- Files written before this change contain no sentinels and parse byte-for-byte as they do today.

Two economy rules keep files honest and small:

- When the rendered prompt is byte-identical to the XPrompt prompt, the rendered section is replaced by a single
  metadata row `- **RENDERED:** identical to the XPrompt`.
- A new config field caps the stored rendered prompt; content beyond the cap is truncated with an explicit
  `… truncated (<N> bytes omitted)` marker inside the fence, never silently.

### Prompt archive format

The published prompt document keeps its existing `PLAN` / `AGENTS` / `ARTIFACTS` header block and its verbatim body. The
body gains xprompt links in the same rewrite pass that already produces `@` artifact links, and the rendered prompt is
appended as one collapsed, fenced section using the same escaping rules as the chat format.

## Phase: Rust xprompt-source wire and reference link rewriting

Work in the sibling Rust core repository. Open it with the `/sase_repo` skill and use only the path that skill prints.

Add `crates/sase_core/src/prompt_xprompt.rs`:

- `PromptXpromptRecord`: `schema_version`, `raw_ref`, `name`, `kind`, `source_kind`, `source_path`, `definition_line`,
  `repo`, `repo_root`, `repo_relpath`, `chezmoi`, `skipped_reason`. Deserialization tolerates unknown and missing
  optional fields so a newer writer never breaks an older reader.
- `parse_xprompt_records(bytes) -> Vec<PromptXpromptRecord>` and
  `select_xprompt_records(records) -> Vec<PromptXpromptRecord>`, the latter keeping the newest row per `raw_ref` in
  first-appearance order, mirroring `select_manifest_records`.
- `rewrite_prompt_xprompt_links(prompt, records, resolver) -> PromptXpromptRewrite`, returning the rewritten prompt and
  the subset of records that were actually linked.

Factor the protection and cursor-walk logic that `rewrite_prompt_artifact_links` already implements
(`prompt_literal_zone_ranges`, `markdown_link_ranges`, `merge_ranges`, longest-token-wins matching) into one private
helper used by both rewriters. Do not fork that logic.

The xprompt rewriter adds one rule the artifact rewriter does not need: a candidate token only matches at a valid
reference boundary — start of input, or immediately after whitespace or one of `(`, `[`, `{`, `"`, `'` — matching the
documented invoke grammar. This is what keeps `#D7D7FF` hex colors, `issue#42`, and Markdown headings from being
rewritten. Records are matched by their exact captured `raw_ref`, so argument forms (`#a:b`, `#a(b, c)`, `#a+`,
`` #a:`quoted arg` ``) are rewritten as complete tokens, and longest-token-wins keeps `#plan_more` from being shadowed
by `#plan`.

Expose bindings from `crates/sase_core_py/src/lib.rs` as `prompt_xprompt_records_parse`,
`prompt_xprompt_records_select`, and `prompt_xprompt_rewrite_links`, following the argument and return shapes of the
existing `prompt_artifact_*` bindings so the Python side can call them through `require_rust_binding`.

Rust tests must cover: fenced-code and `%xprompts_enabled:false` literal zones, already-linked references, hex color
literals, `#gh:` and `#git:` workspace refs, every argument shorthand, overlapping name prefixes, a resolver that
returns `None` for some records, and multibyte content around a reference.

Run the Rust repository's own verification gates (format, clippy with warnings denied, and its test suite) before
finishing.

## Phase: Launch-time capture of xprompt definition provenance

Add `src/sase/xprompt/xprompt_sources.py` in this repository.

- `collect_xprompt_sources(raw_prompt, *, extra_xprompts=None, swarm_xprompts=None)` reuses `collect_used_xprompts()`'s
  reference scan so alias normalization, VCS-underscore normalization, and literal zone skipping stay identical and are
  not duplicated. For each resolved reference it records the exact source substring as `raw_ref` and resolves provenance
  from the matched `XPrompt` or `Workflow` `source_path`.
- `resolve_definition_repo(path)` performs the chezmoi remap and repo-inventory match described in the Design section,
  returning repo name, repo root, POSIX repo-relative path, and the chezmoi flag.
- `resolve_definition_line(path, name)` returns the 1-based line of the xprompt's key for YAML sources, and `None` for
  Markdown sources and anything it cannot determine. Resolve it by locating the mapping key in the YAML source text; do
  not add a YAML parser dependency and do not guess when the key appears more than once ambiguously.
- `write_xprompt_sources(artifacts_dir, raw_prompt, ...)` writes `<artifacts_dir>/xprompt_sources.json` as a JSON array
  and returns the records.

Call it from `preprocess_prompt_xprompts()` in `src/sase/axe/run_agent_runner_setup.py`, immediately after the existing
`write_used_xprompts()` call and inside the same best-effort `try` discipline: provenance capture must never take down a
detached agent launch, and a failure prints a warning to stderr and continues. Records for references that resolve to no
definition, or to a definition outside any known repository, are still written with a `skipped_reason` so later
diagnosis does not require re-running the launch.

Tests: a reference defined in the project's own `sase/xprompts/`, one defined in a home Markdown xprompt with
`use_chezmoi` on and off, one defined in a user `sase.yml` with a line anchor, one defined in a package default, one
unknown name, references inside fenced blocks, and a swarm launch. Assert the artifact is valid JSON, that
`xprompts.json` is unchanged in shape, and that a raised exception inside resolution still leaves the launch path
intact.

## Phase: Hosted URL resolution for xprompt definitions

Add `src/sase/xprompt_links.py` with an `XpromptTargetResolver` shaped like `_ArtifactTargetResolver`:

- Construct it with the primary repository root, the publication revision, a `HostedLinkResolver`, a `GitRunner`, and
  the repo roots map from `repository_roots()`.
- For one record: skip it when it has no repository; choose the revision (the supplied `primary_revision` when the
  record's root is the primary root, otherwise that repository's resolved `HEAD`); call
  `HostedLinkResolver.blob_url_for_repository(root, revision, repo_relpath)`; append `#L<definition_line>` when a line
  was captured. Return `None` on any failure.
- Prefer a live repository root over a materialized per-workspace clone when both are known for the same repo name, so a
  chezmoi link resolves against the source tree the definition actually came from. When the recorded root no longer
  exists, fall back to the inventory root for the same repo name.

Also add a small helper that loads and selects the records for one agent run
(`load_xprompt_source_records(artifacts_dir)`), using the Rust parse and select bindings.

Tests: a GitHub-hosted primary repo, a chezmoi repo whose origin is GitHub, a non-GitHub remote (no link), a repo whose
`HEAD` cannot be resolved (no link), a record with and without a line anchor, and a record whose repository root has
disappeared.

## Phase: Chat markdown stores both prompt renderings

### Writer

Extend `write_chat_history()` and `save_chat_history()` in `src/sase/history/chat_storage.py` and
`src/sase/history/chat.py` with keyword-only `xprompt_prompt: str | None` and `rendered_prompt: str | None`.

Add `src/sase/history/chat_prompt_sections.py` owning the format:

- `render_prompt_sections(xprompt_prompt, rendered_prompt, *, xprompt_links=None)` returns the sentinel-delimited block
  described in the Design section, or an empty string when neither rendering is available.
- `strip_prompt_sections(content)` removes sentinel-delimited regions from a document.
- Sentinel escaping on write and fence-width selection for the rendered payload live here and are unit-tested directly.

Add a config field for the rendered-prompt byte cap, defaulting to a value large enough for ordinary runs, wired through
`src/sase/default_config.yml` with an accessor beside the other chat settings. Truncation is always marked in the
output.

### Parser hardening

In `src/sase/history/chat_resume.py`, run `strip_prompt_sections()` over the document (replacing removed regions with
equal-length whitespace so existing offsets stay valid, or restructuring the scan to work on the stripped copy) before
`parse_chat_turns()` and `extract_previous_conversation_turns()` match headings. Files without sentinels take an
unchanged path.

### Callers

Every `save_chat_history()` caller supplies what it genuinely has, and omits what it does not:

- `finalize_loop()` in `src/sase/axe/run_agent_exec_finalize.py`: read `raw_xprompt.md` and the `select_prompt_file()`
  selection from `state.current_artifacts_dir`, falling back to `ctx.artifacts_dir`. Linkify the XPrompt rendering using
  `load_xprompt_source_records()` and `XpromptTargetResolver`, resolved against the agent's workspace; when no hosted
  resolver is available the text is stored unlinked.
- `src/sase/llm_provider/postprocessing.py`: it already holds the rendered prompt; read the XPrompt rendering from
  `context.artifacts_dir`.
- `src/sase/xprompt/workflow_executor_steps_prompt.py`, `src/sase/ace/workflows/crs.py`,
  `src/sase/ace/handlers/workflow_handlers.py`, and `src/sase/workspace_provider/changespec.py`: pass through what is
  available for that call site. Never synthesize a rendering that was not produced.

### Tests

Round-trip tests that a written file re-parses to the same turns with and without the new sections; a test that an
XPrompt body containing a literal `## Prompt` line does not create a phantom turn; a test that a rendered prompt
containing fenced blocks and headings is preserved byte-for-byte inside its fence; identical-rendering economy;
truncation marking; a legacy file with no sentinels; and end-to-end coverage that `finalize_loop()` stores both
renderings with links.

## Phase: Prompt archive stores both prompt renderings

In `src/sase/agents_sync/prompt_archive/render.py`, extend `render_prompt_document()`:

- Accept the xprompt source records and an `XpromptTargetResolver`, and run the xprompt rewrite over the body in the
  same pass that produces `@` artifact links. Order the two rewrites so neither can rewrite inside the other's output —
  the Rust rewriters already protect existing Markdown links, so running the artifact rewrite first and the xprompt
  rewrite second is safe and must be asserted by a test.
- Accept the rendered prompt and append the collapsed, fenced section, reusing `chat_prompt_sections` so both stores
  produce identical formatting from one implementation.

In `src/sase/agents_sync/prompt_archive/preparation.py`, resolve the rendered prompt from the agent artifacts directory
with `select_prompt_file()`, load the xprompt source records, and build the resolver with the already available `hosted`
resolver, `git_runner`, `primary_revision`, and `repository_roots()`. A missing rendered prompt is not an error: publish
the body as before.

Verify that `format_with_prettier()` leaves the fenced rendered section and the `<details>` wrapper intact, and add a
test asserting the published document is stable across a second render of the same inputs (publication must stay
deterministic and idempotent).

In `src/sase/agents_sync/prompt_archive/validation.py`, add checks that the rendered section's fence is balanced, that
sentinels are paired, and that xprompt link targets are absolute `https://` URLs; report entries that predate this
change as an informational count rather than a warning per file.

Update `src/sase/sdd/templates/sidecar-agents-README.md` to describe both renderings and the xprompt links.

## Phase: Read surfaces, docs, and end-to-end verification

- ACE Artifacts tab chat detail (`src/sase/ace/tui/widgets/artifacts/chats_detail.py`) renders the two new sections as
  fold-aware sections, stripping the sentinel comments and the `<details>` / `<summary>` wrapper rather than showing raw
  HTML.
- `sase agent prompts show` (`src/sase/agents/cli_prompts.py`, parser in `src/sase/main/parser_agent.py`) gains
  `-r/--rendered` to print the rendered prompt instead of the default XPrompt body. `sase chat show`
  (`src/sase/chat/cli_show.py`) gains matching flags. Follow the CLI rules: alphabetical options, a short alias for
  every public long option, excellent `-h` output, and color where it improves readability.
- Update `docs/ace.md`, `docs/xprompt.md`, and `docs/agent_images.md` where they describe the prompt artifacts, and add
  a short section describing the stored chat format and its two renderings.
- Do **not** edit any file under `sase/memory/` or any generated agent instruction file: that requires the user's
  explicit permission. If `sase/memory/xprompts.md` should mention the new artifact, record it with `/sase_new_task`
  instead.
- Verify end to end: launch a throwaway agent whose prompt uses a project-defined reference, a chezmoi-defined
  reference, and an unknown reference; confirm the chat file carries both renderings with exactly the resolvable
  references linked; run `sase agent prompts validate`; and confirm a published prompt entry renders correctly.

## Non-goals

- Historical chat files and previously published prompt entries are not rewritten. `sase agent prompts validate` reports
  how many entries predate the change; a migration, if ever wanted, is separate work.
- `## Prompt` and `## Response` semantics, resume, and fork behavior are unchanged.
- Rendered prompts are not linkified. Linkification applies to unrendered prompts only.
- `%` directives are not linkified.

## Verification

Every phase runs `just install` first because workspace virtual environments are ephemeral, then `just check` before
finishing. The Rust phase additionally runs the core repository's own format, clippy, and test gates. Phases that change
ACE rendering run `just test-visual` and only accept snapshot changes that are intentional.
