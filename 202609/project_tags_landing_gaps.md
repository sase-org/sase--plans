---
tier: epic
title: Close project tag (+sase) landing gaps
goal: "Finish the work the sase-16n landing audit found missing or broken in project
  tags. The cases are: `+home` on a fresh machine, CLI and cold-TUI tag rendering, the
  D7 accept parity, the LSP disabled/hover fields, the invalid rewrite-to-tag, the
  doctor case collisions, the red prompt-history tests, the metadata pager regression,
  the tag PNG goldens, the nvim picker/palette/sigil, and the docs.

  "
phases:
  - id: core-fixes
    title: sase-core accept parity, target wire fields, and LSP tag fixes
    depends_on: []
    size: medium
    description: "core-fixes: accept removes every workspace target the Python
      one-target guard counts; LSP accept uses all tag targets; ProjectTagTargetWire
      gains state and workspace_dir; LSP gains disabled diagnostics/modifier and a
      richer hover; rewrite-to-tag emits only valid tags; suggestions are deduped;
      collision warnings skip siblings and flag key case variants; the dead catalog wire
      is used or removed.

      "
  - id: backend-fixes
    title: Python project tag backend fixes and missing launch tests
    depends_on:
      - core-fixes
    size: medium
    description: "backend-fixes: `+home` resolves before home's spec exists;
      project_tag_for leaves unknown names alone; targets send state and workspace_dir
      over the wire; the completion cache follows current/MRU changes; doctor reports
      key case collisions with accurate wording; dead catalog code goes; the backend
      step-8 launch tests that were never written are added.

      "
  - id: display-fixes
    title: CLI and cold-TUI tag rendering, pager, MRU label, and red tests
    depends_on:
      - backend-fixes
    size: medium
    description: "display-fixes: fix the two red prompt-history label tests; CLI
      surfaces load the catalog so they tagify; the TUI warms the catalog at startup,
      refreshes cold renders, and re-highlights the editor; the metadata pager keeps
      Markdown highlighting around tags; MRU labels read the tag; clan triage tags are
      accent-colored.

      "
  - id: tag-goldens
    title: Deterministic tag PNG golden coverage
    depends_on:
      - display-fixes
    size: small
    description: "tag-goldens: pin a fixture tag catalog in the visual harness, add
      prompt-highlighting and AGENT XPROMPT tag cases, and capture and inspect their
      goldens plus any tag-surface drift with fix-tui-screenshots.

      "
  - id: nvim-fixes
    title: sase-nvim picker fallback, palette overrides, and dim sigil
    depends_on:
      - core-fixes
    size: small
    description: "nvim-fixes: the Ctrl+T +query picker works without native LSP
      completion; palette application keeps user overrides; the + sigil renders in dim
      accent; disabled tags use the new server modifier; README and tests match.

      "
  - id: docs-fixes
    title: Project tag docs accuracy pass
    depends_on:
      - core-fixes
      - backend-fixes
      - display-fixes
      - nvim-fixes
    size: small
    description:
      "docs-fixes: editor.md semantic-token legend and palette capability, stale
      #gh:sase and gh_sase alias examples, the -P help wording, getting_started
      tag-first wording, the sase-github #gh(sase) note, and a fresh-machine check of
      the +home first-run examples."
proposed_by: bbugyi200.athena.sase-16n.land
parent_bead: sase-16n
create_time: 2026-09-23 08:51:41
status: wip
---

- **PROMPT:**
  [prompts/202609/project_tags_landing_gaps.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/project_tags_landing_gaps.md)
- **PARENT:**
  [202609/project_tags.md](https://github.com/sase-org/sase--plans/blob/main/202609/project_tags.md)

# Plan: Close project tag (`+sase`) landing gaps

## Why

Epic sase-16n (plan `202609/project_tags.md`) added `+<project>` project tags. All ten
of its phases closed. The landing audit then compared every plan bullet against the code
and found gaps and regressions that the epic caused. This plan finishes that work.

The parent plan's design contract (D1–D10) still applies unchanged. Read it first:
`sase plan show 202609/project_tags.md`, or the PLAN path printed by
`sase bead read sase-16n`. This plan changes no design decision. It only makes the code
match it.

Phase agents open linked repos with `sase repo open <repo>` and work in the path it
prints. Verification:

- sase: `just install`, then `sase tool run check`.
- sase-core, sase-nvim, sase-github: each repo's own wrapped `check`/test recipe.

Do not run `just check-full`.

The known master red `symvision: ExpandedLaunchSegments` belongs to task sase-16u, not
this epic. If `check` still fails only on that, record it and move on.

## Phase core-fixes (sase-core)

Follow the sase-core `AGENTS.md` conventions.

1. **Accept parity (D7 "always leaves exactly one").**
   - `project_tag/accept.rs::workspace_target_ref_regex` only matches refs at line start
     (`(?m)^((?:%\S+\s+)*)#…`). The Python one-workspace-target guard also counts
     mid-line refs: `find_vcs_workflow_tag_span("fix in #gh:foo now")` returns
     `(7, 14)`. Accepting `+sase` there leaves two targets, and the launch then fails.
   - Make the removal set match what the Python guard counts. Keep literal-zone
     skipping, the per-`---`-segment scope, and whitespace collapsing.
   - When `workflow_names` is empty, match nothing. Today the empty alternation matches
     `# Heading` and deletes it.
   - Add vectors for: a mid-line ref, the empty-workflow-names case, and a heading line.
2. **LSP accept uses every tag target.**
   - `editor/completion/vcs_candidates.rs` (around the two `apply_project_tag_selection`
     call sites) builds its targets from enabled completion rows only.
   - Build them from the catalog's `project_tags` instead, which include disabled
     projects and `home`. The LSP and TUI accept then remove the same resolved tags.
3. **Target wire fields (D2).**
   - `ProjectTagTargetWire` (`project_tag/wire.rs`) gains optional serde-default fields
     `state` (`enabled` | `disabled` | `system`) and `workspace_dir`.
   - Older catalogs without these fields keep working.
   - Update any Python binding round-trip tests that construct targets.
4. **LSP disabled handling and hover (D3 editor column, lsp steps 1, 3, 4).**
   - In `crates/sase_xprompt_lsp/src/project_tags.rs` and `semantic_tokens.rs`, a
     resolved target whose `state` is `disabled` gets:
     - a warning diagnostic: `+x is disabled — sase project enable x`
     - the `disabled` token modifier
   - Today only provider-less targets get either.
   - Hover and completion documentation show the state and workspace dir when present.
   - Tests use v5 catalogs with and without the new fields.
5. **Rewrite-to-tag emits only valid tags (lsp step 5).**
   - Offer "Use project tag +x" only when the rewritten text is a standalone tag under
     D1: a valid left boundary before it, and end/whitespace/`|`/`}` after it.
   - Today `#gh:sase.`, `#gh:sase,` and `(#gh:sase)` become `+sase.`, `+sase,` and
     `(+sase)`. Those never expand, so the prompt silently loses its project.
   - Add negative tests.
6. **Suggestions.** `project_tag/resolve.rs` dedupes only adjacent entries (the
   `dedup_by` near line 103). Currently `resolve("xome", [home, zome])` yields
   `+home, +zome, +home`. Dedupe fully while keeping rank order, and add that test.
7. **Collision warnings (D8).** `project_spec.rs::add_project_ref_collision_warnings`:
   - exclude sibling specs
   - flag two directory keys that differ only by case
   - flag a non-system key that case-folds to `home`
   - keep folding consistent with Python `str.casefold`, or document the ASCII-only
     difference in a comment
   - add tests
8. **Dead wire.**
   - `editor/wire.rs` exports `VcsProjectCatalogWire` and
     `VCS_PROJECT_CATALOG_SCHEMA_VERSION`, but only their own tests use them. The LSP
     parses the catalog by hand.
   - Either parse through them or delete them with their tests.
9. **Directive scanning (optional, only if cheap).** `project_tag/scan.rs` hand-rolls
   leading-directive stripping and frontmatter detection. Reuse `directive_scan` and the
   `editor/exclusion.rs` frontmatter helper if that is a mechanical swap with identical
   test results; otherwise leave it.
10. Run the sase-core check.

## Phase backend-fixes (sase)

1. **Pin.** If core-fixes changed anything a Python-called binding consumes, run
   `just ratchet-core-revision` (see `docs/rust_backend.md`) once core-fixes is on
   sase-core master.
2. **`+home` on a fresh machine (D2: `home` is always a target).**
   - The tag catalog (`src/sase/project_tags/catalog.py`, built from
     `iter_patch_project_files(..., include_home=True)`) contains `home` only once its
     ProjectSpec exists.
   - `launch_query` validates tags (`main/query_handler/_launch.py`, around lines
     111–121) before `#git:home` would bootstrap home. So on a fresh `SASE_HOME`,
     `+home …` fails with `Unknown project tag +home`, even though README,
     getting_started, and `sase run --help` present it as the first-run example.
   - Always include a synthetic system `home` target: key `home`, workflow type `git`,
     state `system`, no accent. `+home` then expands to `#git:home` and bootstraps
     exactly as `#git:home` does today.
   - Add a test with an empty projects dir.
3. **`project_tag_for` leaves unknown names alone.**
   - `project_tags/tags.py::project_tag_for` returns `+<name>` for any in-grammar name
     that the catalog does not know. Examples: `sase-core` and other siblings. Those
     tags then fail launch validation.
   - This breaks its docstring and the `#wf:` fallback promised by
     `bead/work.py::_vcs_launch_prefix`.
   - Return the input unchanged for unknown names. Check the other two callers
     (`workspace_provider/plugins/bare_git_ref.py`, `monitor/followup_continuation.py`)
     still behave, and add tests.
4. **Wire fields.**
   - `ProjectTagTarget.to_wire()` also sends `state` and `workspace_dir`.
   - `vcs_project_catalog_payload()`'s `project_tags` entries carry them.
   - Update the schema tests and `tools/validate_sase_core_rs` as needed.
5. **Completion cache freshness (D7).**
   - `xprompt/vcs_project_completion.py` caches entries, including the `current` flag
     and MRU order, keyed only on ProjectSpec mtimes.
   - `sase project set-current` writes only the MRU, so the TUI `+` menu keeps showing a
     stale `current` badge and order.
   - Key the cache on the current-project and MRU signature too, or compute
     current/order outside the cached part.
   - Add a test.
6. **Doctor (unique-names step 2).**
   - `project_alias_records.py` (the conflict scan around lines 236–285) checks only
     `PROJECT_NAME`/alias claims. It never reports two directory keys that differ only
     by case, or a non-system directory keyed like `Home`. Report both.
   - `doctor/checks_project.py` (around line 259) says `directory {occupant}` even for
     alias-vs-alias conflicts. Word the message by conflict kind.
   - Add tests.
7. **Dead code.**
   - Delete `catalog_cache_signature` (no caller) from `project_tags/catalog.py` and its
     `__all__`.
   - `load_project_tag_catalog(use_cache=False)` has identical if/else branches, so it
     still overwrites the shared cache. Make it leave the cache alone.
8. **Missing backend tests (parent plan backend step 8):**
   - `launch_query` ordering: validation and expansion run only after the local-dispatch
     decision, and before force-reuse, typed dispatch, MRU, and spawn.
   - Remote dispatch forwards `+tags` verbatim.
   - MRU still records `#gh:<key>`.
   - The unit guard (`agent/launch_cwd_guards.py::guard_project_tags_for_launch_units`)
     rejects: two workspace targets in one unit, ambiguous tags, provider-less tags, and
     disabled tags. All are rejected before any spawn.
   - `%{+sase | +bob-cli}` alt fan-out and a swarm.
   - LaunchApproval preview (`build_preview_plan`) surfaces tag errors.
   - A `#git:Sase` init is refused with a `+sase` hint.

## Phase display-fixes (sase)

Obey tui_perf rules: use `peek()` on render and keystroke paths, and do builds off the
UI thread. Read the `tui.md` memory before editing TUI code.

1. **Red tests.**
   - Two tests in `tests/ace/tui/modals/test_prompt_history_modal_label.py` have failed
     since 3bf3b998a: `test_prompt_history_preview_metadata_includes_prompt_metadata`
     and `test_prompt_history_preview_uses_display_text`.
   - The cause: `_prompt_history_interactions.py` now passes a Rich `Text` to
     `preview.update()`, while the tests still compare against a `str`.
   - Update the tests to assert on the plain text and the tag styling.
2. **CLI surfaces tagify (tag-surfaces step 4, D5).**
   - The humanizer and `stylize_project_tags` only read the in-memory catalog snapshot,
     and no CLI path loads it. As a result, `sase agent show`, `sase prompt show` (never
     wired: `prompt/cli_show.py`), `sase prompt list|search`, and `sase xprompt show`
     print `#gh:sase` in a fresh process.
   - Load the catalog (blocking is fine in the CLI; degrade silently on failure) before
     rendering.
   - Tests must start from a cleared catalog cache.
3. **TUI warm-up and cold refresh.**
   - The catalog warms only when a prompt bar mounts (`_prompt_input_bar_lifecycle.py`),
     so agent panels, history, and query accents render `#gh:` until then.
   - Warm it off-thread at app startup.
   - When the snapshot first warms, invalidate or refresh the surfaces that rendered
     cold. Their caches already key on the catalog signature, so a refresh message
     should be enough.
4. **Editor re-highlight (tag-display step 4).** Nothing re-highlights the prompt editor
   when the catalog warms, because no handler reacts to the catalog worker finishing
   (`_file_completion_workers.py`, `_xprompt_syntax_highlight.py`). Tags therefore stay
   unstyled until the next edit. Add that re-highlight.
5. **Metadata pager.**
   - `actions/agents/_metadata_pager_conversation.py::_tag_styled_xprompt_body` adds tag
     spans as producer styling. That makes `pager/syntax.py` (around line 203) return
     PRESERVED, so an AGENT XPROMPT body containing a tag loses all Markdown
     highlighting.
   - Emit tags through the pager highlighter via the existing but unused
     `SyntaxRole.PROJECT_TAG`, so Markdown and tag highlighting coexist.
   - Add a test.
6. **MRU label.**
   - `_dedupe_mru_pairs` now tagifies display prefixes.
   - `actions/agent_workflow/_entry_custom.py` (around line 30) and
     `_entry_prompt_history.py` (around line 44) read the project name with
     `extract_project_from_vcs_tag`, which returns `None` for `+sase`. The bar label
     then shows `+sase` instead of `sase`.
   - Use a tag-aware lookup and add tests.
7. **Clan triage lines.** `prompt_panel/_agent_display_clan_sections_text.py` (the
   triage line near line 182) tagifies but does not accent-color. Apply the shared tag
   overlay.

## Phase tag-goldens (sase)

1. The catalog snapshot is process-global. Give the visual harness a fixture tag catalog
   with fixed projects and accents, and reset it between tests. Goldens then never
   depend on the host's projects or on test order.
2. Add a tag case to `tests/ace/tui/visual/_ace_prompt_png_snapshot_prompts.py`, and an
   AGENT XPROMPT tag case to the `agents_xprompt_panel_highlighting*` family.
3. Capture with `just fix-tui-screenshots -- <selectors>` through `/sase_monitor`, using
   the TESTING/TESTED pair. Then refresh any drift in the prompt-history, stash,
   launch-context, and Projects-pane goldens caused by tag rendering. Inspect every
   creation and update in the report before finalizing (`lint_and_test.md` memory).

## Phase nvim-fixes (sase-nvim)

1. **Picker fallback (parent nvim step 2).**
   - With `native_completion = false` (the user's nvim-cmp setup), `<C-t>` on `+query`
     calls `sase.lsp.complete()`, which returns false and does nothing.
   - Implement the planned fallback in `complete.lua`: offer projects from
     `sase project list --json` (its `tag` field) and insert `+name` in place.
   - Fix the README claim that this works "in every completion backend".
2. **Palette overrides.** `project_tag_highlight.lua` applies the palette with
   `{force = true}` when the LSP starts. That clobbers group overrides the README tells
   users to set after setup. Keep `default = true` semantics while still refreshing on
   `ColorScheme` and palette changes.
3. **Dim sigil (D6).**
   - The server's `sigil` modifier is ignored, so `+` renders in bold accent.
   - Add `SaseProjectTagSigil<N>` groups (dim accent fg) and map sigil tokens to them.
   - Map the core-fixes `disabled` modifier to the existing disabled group.
4. Update the README and tests (highlight unit test and the picker fallback), then run
   the repo's test recipe.

## Phase docs-fixes (sase, sase-github)

1. **`docs/editor.md`.**
   - Add `saseProjectTag` and its modifiers (`sigil`, `unknown`, `disabled`,
     `accent0`–`accent17`) to the semantic-token legend.
   - Correct the "Every semantic token uses a standard LSP token type" sentence.
   - Mention tags in the Semantic highlighting row.
   - Document `experimental.sase.projectTagPalette` and the disabled diagnostic.
2. **`docs/xprompt.md`.** Change `sase run '#gh:sase #!sync'` (around line 418) to
   `+sase`.
3. **Alias example.**
   - The xprompt-alias example `gh_sase: "gh:sase"` appears in `docs/xprompt.md` (around
     line 3220) and `docs/configuration.md` (around line 3720). The schema description
     in `src/sase/config/sase.schema.json` (around line 4558) has
     `{"#gh_sase": "#gh:sase"}`.
   - All three teach the shortcut the parent epic removed as redundant. Replace them
     with a non-project example, and change the schema at its source if it is generated.
4. **`-P` help.** Make the `-P/--vcs-prefix` example in `main/parser_prompt.py` agree
   with the `-P` limitation documented in `docs/prompt.md`, and update its help test.
5. **`docs/getting_started.md`.** Around lines 166–172, present `+<name>` as the way to
   target an existing managed project. `#git:<name>` stays for creating one.
6. **sase-github `docs/xprompts.md`.** The "by shorthand name" `#gh(sase)` note (around
   lines 48–49) should say that plain targeting uses `+sase`, and that the paren form is
   for arguments.
7. **Fresh-machine check.** After backend-fixes, confirm on an empty `SASE_HOME` that
   the `+home` first-run examples pass tag validation. The examples are in `README.md`,
   `docs/getting_started.md`, `sase run --help` (`main/parser_commands.py`,
   `main/parser_root_help.py`), and `smoke/pypi/README.md`. Use
   `validate_project_tags_for_launch` or a LaunchApproval preview; do not launch an
   agent.

## Verification

- **Each phase** runs its repo's wrapped check recipe.
- **sase phases:** `just install` first. PNG goldens run only through targeted
  `fix-tui-screenshots` under `/sase_monitor`.
- **End-to-end** (backend-fixes or later):
  - `+home` validates on a fresh `SASE_HOME`.
  - `sase prompt show` on a `#gh:<key>` prompt prints `+sase` in a fresh process.
  - Accepting `+sase` into `fix in #gh:foo now` leaves exactly one workspace target.
  - The LSP warns on a disabled `+x`.
