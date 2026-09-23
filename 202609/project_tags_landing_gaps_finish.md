---
tier: epic
title: Finish the project tag (+sase) landing-gap fixes
goal: 'Fix the defects the sase-16n.11 landing audit found in its own work. sase-core:
  a macOS-red CI test, and accept joining lines. sase-nvim: tag colors never show,
  and palette overrides get lost. sase: a follow-up monitor regression, a warm-refresh
  that does nothing, pager tag styling, and a history-filter blocking load. Also integrate
  the new tribe PROMPTS chip, and close the remaining test and doc gaps.

  '
phases:
- id: core-accept
  title: sase-core macOS test fix, accept line-join fix, and tag cleanups
  depends_on: []
  size: medium
  description: 'core-accept: make the case-variant collision test pass on case-insensitive
    file systems; accept removal no longer deletes the newline after a line-end ref
    and counts refs hidden inside a rejected glued match; pre-v5 LSP catalogs fall
    back to enabled rows for accept; fix stale comments and dead checks.

    '
- id: nvim-tokens
  title: sase-nvim set-shaped token modifiers, full override tracking, picker errors
  depends_on: []
  size: small
  description: 'nvim-tokens: token_group reads the set-shaped modifiers Neovim passes
    so accent, sigil, unknown and disabled colors actually show; palette refresh keeps
    any user override, not just a different foreground; the +query picker reports
    failures and adds a trailing space; tests use real modifier shapes and cover the
    auto-mode fallback.

    '
- id: backend-regressions
  title: sase follow-up prefix regression, non-blocking history filter, test isolation
    and gaps, docs nits
  depends_on:
  - core-accept
  size: medium
  description: 'backend-regressions: ratchet the sase-core pin; frozen_intent_vcs_prefix
    counts only a real +tag token; the prompt-history project filter never builds
    the catalog on the event loop; the tag catalog cache resets between tests; add
    the missing assertions for completion order, fresh-process CLI output, alt fan-out
    and the MRU label; skip catalog loads for raw/json prompt show; add occupant_kind
    to doctor JSON; fix the docs nits.

    '
- id: tui-tag-surfaces
  title: TUI warm refresh that rebuilds surfaces, pager tag accents, tribe PROMPTS
    chip
  depends_on:
  - backend-regressions
  size: medium
  description: 'tui-tag-surfaces: ProjectTagCatalogWarmed rebuilds the surfaces that
    rendered cold (agent detail and panels, history, query accents), with a real-app
    test; the metadata pager styles only resolved tags, in each project''s D6 accent,
    and never inside fences; the tribe PROMPTS chip shows +name only for catalog-known
    projects; refresh the affected goldens.

    '
proposed_by: bbugyi200.athena.sase-16n.11.land
parent_bead: sase-16n.11
create_time: 2026-09-23 14:10:48
status: done
bead_id: sase-16n.11.7
---

- **PROMPT:** [prompts/202609/project_tags_landing_gaps_finish.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/project_tags_landing_gaps_finish.md)
- **PARENT:** [202609/project_tags_landing_gaps.md](https://github.com/sase-org/sase--plans/blob/main/202609/project_tags_landing_gaps.md)
- **BEAD:** [sase-16n.11.7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-16n/sase-16n.11.7.md)

# Plan: Finish the project tag (`+sase`) landing-gap fixes

## Why

Epic sase-16n added `+<project>` project tags. Its landing-gap epic sase-16n.11 (plan
`202609/project_tags_landing_gaps.md`) closed all six phases. The sase-16n.11 landing
audit then re-checked every phase against the code. The audit found defects that
sase-16n.11 itself caused, and integration gaps with work that landed while it was open.
This plan fixes only those. It changes no design decision: the parent design contract
D1–D10 in `202609/project_tags.md` still applies
(`sase plan show 202609/project_tags.md`). The rules this plan leans on:

- **D3:** unknown, unanchored tags are plain text.
- **D5:** Patch refs and `owner/repo` never tagify.
- **D6:** a tag uses its project's accent, with a `dim` sigil and a `bold` name. `home`
  and disabled projects use a neutral dim style.
- **D7:** accepting a tag leaves exactly one workspace target and collapses whitespace
  the way `_strip_trigger_token` does.

Phase agents open linked repos with `sase repo open <repo>` and work in the path it
prints.

Verification:

- sase: run `just install`, then `sase tool run check`.
- sase-core: run its wrapped `sase tool run check`.
- sase-nvim has no recipe. Run each headless unit suite under `tests/` the way the
  README describes.
- Do not run `just check-full`.
- The master symvision failure on `ClanSummaryDigest` belongs to active epic sase-170,
  not this plan. If `check` fails only on that, record it and move on.

## Phase core-accept (sase-core)

Follow the sase-core `AGENTS.md` conventions.

1. **The macOS CI failure.** sase-core master CI has been red on `macos-latest`
   since 4300166. Every push since fails, and so does the v0.34.74 release PR.
   - The failing test is
     `project_spec::tests::lifecycle_project_ref_collisions_flag_case_variant_keys_and_home`
     (`crates/sase_core/src/project_spec.rs`, around line 1692). It creates the
     directories `alpha` and `ALPHA`. macOS file systems ignore case by default, so the
     second create fails with `AlreadyExists` (`panicked at project_spec.rs:1700`).
   - Make the test independent of file-system case sensitivity. Keep asserting that
     case-variant keys are flagged. Either build the colliding records directly, or run
     the two-directory case only when a probe shows the file system is case-sensitive,
     and cover the flagging logic on every platform.
   - A Linux run cannot prove the macOS fix, so reason it through: after the change, no
     test on any platform may create two paths that differ only by case.
2. **Accept must not join lines.** In `project_tag/accept.rs`,
   `workspace_target_ref_regex` ends in `(?:\s|$)`, and `workspace_ref_deletions`
   deletes up to `whole.end()`. Now that refs in the middle of a line match, a ref at
   the end of a line also deletes its newline:
   - `"fix in #gh:foo\nmore stuff +sa"` becomes `"fix in more stuff +sase "`. Expected:
     `"fix in\nmore stuff +sase "`.
   - `"a #gh:foo\n\nb +sa"` loses the blank line between the paragraphs.

   Fix it:
   - End the deletion span at the ref itself, which is what the Python guard's span
     covers (`find_vcs_workflow_tag_span`).
   - Collapse only horizontal whitespace around it.
   - A ref alone on its line removes that line, without joining its neighbors.
   - Add vectors for the three inputs above, plus `"line one #gh:foo\n+sa"`.
   - Keep every existing vector passing.

3. **Glued-match parity (small).** The `regex` crate has no lookbehind, so the left
   boundary is checked after matching. A rejected glued match can therefore swallow a
   valid ref inside it: `"x#gh(a #gh:foo) now +sa"` keeps `#gh:foo`, but the Python
   guard counts it. When the boundary check fails, resume the search at the next byte
   (`find_at`) instead of skipping the whole match. Add that vector.
4. **Pre-v5 catalogs.** The LSP accept now builds targets from the catalog's
   `project_tags` only (`server/completion.rs` →
   `build_vcs_project_completion_candidates_with_targets`). A pre-v5 catalog has none,
   so accept deletes no other tags at all. When `project_tags` is empty, fall back to
   the enabled completion rows, which was the old behavior. Add a test.
5. **Cleanups:**
   - The doc comment on `apply_project_tag_selection` (`accept.rs`, around lines 64–66)
     still says "VCS workflow refs at line starts". Correct it.
   - In `add_project_ref_collision_warnings` (`project_spec.rs`, around lines
     1030–1044), the `sibling_names` check can never match, because `key_owners` already
     excludes siblings. Remove it.
   - The LSP hardcodes the supported catalog versions as `1..=5` (`server/catalogs.rs`).
     Derive the upper bound from `VCS_PROJECT_CATALOG_SCHEMA_VERSION`.
   - `build_vcs_project_completion_candidates` (`editor/completion/vcs_candidates.rs`,
     plus its `lib.rs` re-export) has only test callers now. Check sase_core_py and the
     sase Python callers. Delete it with its tests if nothing uses it; otherwise leave
     it.
6. Run the sase-core check.

## Phase nvim-tokens (sase-nvim)

1. **Set-shaped modifiers.** In `lua/sase/project_tag_highlight.lua`, `modifier_set`
   loops with `ipairs(token.modifiers)`.
   - Neovim 0.10 and later pass modifiers as a set, e.g.
     `{ sigil = true, accent3 = true }` (see `runtime/lua/vim/lsp/semantic_tokens.lua`).
     The loop finds nothing, so every project tag falls through to
     `SaseProjectTagDisabled`. Accent, sigil and unknown colors therefore never show.
   - Accept both shapes: `name = true` keys and list strings.
2. **Test the real shape.** The tests build list-shaped modifiers, which hides the bug.
   - Add cases with the set shape Neovim actually passes: sigil+accent, accent-only,
     unknown and disabled.
   - If practical, add one assertion that goes through Neovim's own semantic-token
     update path.
3. **Keep every override.** `track_palette_group` compares only the foreground color.
   - An override with the palette's foreground but other attributes (italic, no bold,
     underline, bg) is overwritten, and bold is forced back on.
   - Remember exactly what this module last set for each group. Refresh a group only
     when it is unset or still identical to that; leave anything else alone.
   - Test the italic-with-same-foreground case.
4. **Picker polish** (`lua/sase/complete/project_tag.lua`):
   - When `sase project list --json` fails, prints nothing or prints invalid JSON, show
     a WARN `vim.notify` and call `on_cancel`, so the fallback never fails silently.
   - Insert `+name ` with a trailing space, like LSP completion.
   - The picker does not remove other workspace targets the way LSP accept does. Say so
     in the README.
5. **Fallback test.** Test the case that was actually reported: in `auto` mode,
   `sase.lsp.complete()` returns false, and `+query` reaches the picker. Also exercise
   the real in-place text replacement, not a stubbed `pick()`.
6. Update the README as needed and run every headless unit suite.

## Phase backend-regressions (sase)

1. **Pin.** Once core-accept is on sase-core master, run `just ratchet-core-revision`
   (see `docs/rust_backend.md`). The TUI accept goes through the core binding.
2. **Follow-up monitor prefix regression.** `frozen_intent_vcs_prefix`
   (`src/sase/monitor/followup_continuation.py`) treats "a tag naming the recorded
   project" as already prefixed.
   - It does this with `tag not in mutable_next_action`, a substring test.
   - Since `project_tag_for` now returns unknown names unchanged, a plain prose mention
     counts. Example: meta `vcs_ref: ['git', 'sase-core']`,
     `monitor_next_action: 'Check the sase-core CI run and report.'`,
     `frozen_next_action: 'Report the result.'`. This now returns `'#git:sase-core\n'`;
     it used to return `''`.
   - Substrings also match: `+bob` counts inside `+bobby`.
   - Fix: only a resolved tag (`tag.startswith("+")`) can count, and only as a real tag
     token. Use the tag scanner, or match with D1 boundaries on both sides.
   - Add regression tests for all three cases.
3. **Prompt-history filter blocks the event loop.**
   - `PromptHistoryModal._append_page` calls `prepare_prompt_history_row_facts` on the
     UI thread (`src/sase/ace/tui/modals/prompt_history_modal.py`).
   - For every history row containing `+`, `_segment_active_ref`
     (`src/sase/history/prompt_history_project_filter.py`) calls
     `effective_vcs_workflow_tag` → `expand_project_tags` → `load_project_tag_catalog`.
     That walks and stats every project spec, and can build the whole catalog.
   - Make this path peek-only. Resolve against `peek_project_tag_catalog()`, and fall
     back to the unexpanded text when it is cold. Alternatively, compute the facts off
     the UI thread, as the tui_perf rules require.
   - Keep the `project:` filter results identical when the catalog is warm, and add a
     test that the modal path never calls `load_project_tag_catalog`.
   - Read the `tui.md` memory before editing TUI code.
4. **Reset the catalog cache between tests.**
   - The tag catalog snapshot (`_CATALOG_CACHE` in `src/sase/project_tags/catalog.py`)
     is process-global.
   - Every full-app test now warms a catalog at startup. `test_project_tag_launch.py`
     (the empty-projects `+home` test) also leaves a temp-dir catalog in the cache.
     Later tests in the same worker can then render `#git:home` as `+home`, depending on
     test order.
   - Add a function-scoped autouse fixture in `tests/conftest.py` that calls
     `_clear_project_tag_catalog_cache()` after each test. The visual conftest's fixture
     pin must keep working.
5. **Missing assertions:**
   - Completion cache (`tests/test_xprompt_vcs_project_completion.py`): after
     `sase project set-current`, the `+` menu's `current` badge and MRU order actually
     change. Today the test only proves the cache rebuilds.
   - CLI in a fresh process, starting from a cleared cache: `sase prompt show` on a
     stored `#gh:<key>` prompt prints `+<name>`. This is the parent plan's end-to-end
     check. Do the same for `sase agent show` and one of `sase prompt list|search` /
     `sase xprompt show`.
   - Alt fan-out with the literal `%{+sase | +bob-cli}` (the test uses `+bob` today).
   - The prompt-history MRU entry path (`_entry_prompt_history.py`) with a `+tag`
     prefix.
6. **Small fixes:**
   - `prompt show` with raw or JSON output (`src/sase/prompt/cli_show.py`) should skip
     `ensure_project_tag_catalog()`, since that output never tagifies.
   - Add `occupant_kind` to the doctor collision JSON (`_collision_data` in
     `src/sase/doctor/checks_project.py`).
7. **Docs nits:**
   - `docs/editor.md` (around line 76) says unknown, ambiguous and provider-less tags
     all warn. In the LSP, `ambiguous_project_tag` is an Error, and unknown tags only
     get a diagnostic when anchored. Match the severities.
   - `docs/getting_started.md` (around line 136) still says "After the `#git:home` run
     above". That run now uses `+home`.
   - The `resolve_xprompt_aliases` docstring (`src/sase/xprompt/processor.py`) still
     uses the retired `#gh_sase → #gh:sase` alias example. Use a non-project example.

## Phase tui-tag-surfaces (sase)

Obey the tui_perf rules: use `peek()` on render and keystroke paths, and do builds off
the UI thread. Read the `tui.md` memory first.

1. **The warmed refresh must rebuild.** `on_project_tag_catalog_warmed`
   (`src/sase/ace/tui/actions/_startup_loads.py`) only calls `App.refresh()`.
   - That repaints the Screen but never re-renders child widgets. The agent detail is
     prebuilt `Text` set through `update_display`, so a cold `#gh:sase` stays until the
     selection or the data changes.
   - On a fresh signature, rebuild the surfaces that tagify:
     - the agent detail and panels
       (`_refresh_agent_focus_detail(render_immediate=False)`, the `watch_theme` pattern
       in `actions/agents/_display_detail_render.py`)
     - any open prompt-history or stash modal
     - query accents
     - tribe and clan sections, if their caches key on the catalog signature
   - Keep the one-refresh-per-signature guard.
   - Replace the fake-object test in `tests/ace/tui/test_project_tag_catalog_warm.py`
     with a real-app (pilot) test: render with a cold catalog, warm it, and assert the
     visible detail text changes from `#gh:sase` to `+sase`.
2. **Pager tag styling.** `_project_tag_spans` (`src/sase/pager/_markdown_syntax.py`)
   has three problems:
   - It also emits `project_tag_unknown` spans, so a typo like `+no-such-project` looks
     like a valid tag. Under D3 that is plain text. Emit resolved tags only.
   - Every tag uses the theme accent (`SyntaxRole.PROJECT_TAG`). D6 requires each
     project's own accent (`dim` sigil, `bold` name) and a neutral dim style for `home`
     and disabled projects. That is how the AGENT XPROMPT body looked before b924b0350.
     Carry the per-project accent through the pager spans, for example with an optional
     style override on the span, or per-accent roles. Keep the span-budget behavior.
   - `_fence_spans` lexes a fenced ` ```md ` block as nested Markdown, so a `+sase`
     inside it gets styled. The docstring says fences stay tag-free. Emit tag spans only
     for top-level prose.
   - Add tests for all three, and keep the existing Markdown-plus-tag coexistence test
     green.
3. **Tribe PROMPTS chip (integration with c6c70befc).**
   - `_append_entry_tags`
     (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_tribe_prompts.py`) renders
     `Text(f"+{digest.project}")` for any project ref.
   - As a result, Patch refs and `owner/repo` refs get a made-up tag:
     `#gh:sase_fix_parser` shows as `+sase_fix_parser`. `multi_project` also counts
     `sase` and `sase_fix_parser` as two projects.
   - In `_build_digest` (`_agent_tribe_prompts.py`, a worker path whose cache already
     keys on the catalog signature), compute the tag spelling with
     `known_project_tag_for(peek_project_tag_catalog(), ref)`.
   - Render `+name` only when that returns a `+` tag, and the plain ref otherwise. This
     is the pattern in `widgets/_vcs_mru_cycling.py` and
     `modals/project_management_rendering.py`.
   - Base `multi_project` on resolved projects.
   - Update `tests/ace/tui/widgets/test_agent_display_tribe_prompts.py`: its current
     test expects `+alpha`/`+beta` for projects the catalog doesn't know.
4. **Goldens.**
   - Use `just fix-tui-screenshots -- <selectors>` through `/sase_monitor`, with the
     TESTING/TESTED pair, for the goldens these changes touch: the AGENT XPROMPT tag
     cases, any pager tag goldens, and the tribe PROMPTS goldens.
   - Inspect every creation and update in the report before finalizing (the
     `lint_and_test.md` memory).
   - Pre-existing drift owned by other tasks stays untouched: sase-16w, sase-16o,
     sase-173.

## Verification

- **Each phase** runs its repo's wrapped check.
- **End to end**, after all phases:
  - sase-core master CI is green on macOS.
  - Accepting `+sase` into `"fix in #gh:foo\nmore"` leaves one target and both lines.
  - In Neovim 0.10 or later, `+sase` renders a dim accent sigil and a bold accent name.
  - A follow-up whose next action only mentions `sase-core` in prose still gets its
    prefix.
  - ACE started on the Agents tab shows `+sase` once the startup warm lands, without any
    selection change.
