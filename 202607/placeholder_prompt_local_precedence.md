---
tier: tale
title: Restore prompt-local precedence in placeholder completion
goal: "Manual `Ctrl+T` on a `<` context inserts a lone current-prompt placeholder match outright again even when saved
  common tags also match, and placeholder text written inside inline code, fenced blocks, or `%xprompts_enabled:false`
  regions is offered again as a current-prompt completion candidate ranked after live tags, while highlighting,
  submit-time collection, and the durable saved-tag store stay raw-only.

  "
create_time: 2026-07-29 08:35:44
status: done
---

- **PROMPT:** [202607/prompts/placeholder_prompt_local_precedence.md](prompts/placeholder_prompt_local_precedence.md)

# Plan: Restore prompt-local precedence in placeholder completion

## Context

Two independent changes narrowed what the `<placeholder>` completion menu does for the prompt the user is actually
editing. Neither was intended to take anything away.

**Regression 1 — `Ctrl+T` lost its one-key insert.** `_try_file_completion_tab`
(`src/sase/ace/tui/widgets/_file_completion_tab.py:69-76`) auto-accepts when the placeholder payload holds exactly one
candidate. Since the saved common-tag group was merged into that same payload, the count is rarely one any more, so a
prompt-local match that used to be inserted outright now opens a menu that must be confirmed with Enter. Reproduced
against the live store:

```
text:  "Rebind <ctrl+enter> and <ctrl+e"      saved: ["ctrl+e", "ctrl+e/y"]
today: [("ctrl+enter","prompt"), ("ctrl+e","common"), ("ctrl+e/y","common")]  -> 3 rows, no insert
before: [("ctrl+enter","prompt")]                                            -> inserted outright
```

The epic that added saved tags called this out (`sase/repos/plans/202607/common_placeholder_tags.md`, decision D5) but
only reasoned about the shortcut _firing_ for a lone saved tag; it did not intend for the shortcut to _stop_ firing for
the prompt's own tags.

**Regression 2 — literal-zone tags dropped out of the candidate list.** `sase-core` commit `61f4162`
(`feat(editor): add raw placeholder transforms`) added a `raw` flag to `PlaceholderSpan` and made
`build_placeholder_completion_candidates` skip every non-raw span (`crates/sase_core/src/editor/placeholder.rs`, the
`if !placeholder.span.raw { continue; }` guard). Excluding literal zones is right for highlighting, submit-time
collection, and substitution — those all mean "this text will be replaced". It is wrong for _completion_, whose job is
to let the user re-type a tag they have already written in this prompt. Reproduced:

```
text:  "Fix `<the widget>` and update <t"      saved: []
today: None (no menu at all)
before: [("the widget","prompt")]
```

This one is easy to mistake for regression 1 because keybinding-shaped prompts tend to put tag-like text in backticks.

## Governing rule

Adopt and state one invariant, because it is the thing the user actually asked for and it settles both changes:

> The saved common-tag group never changes what the current prompt's own tags would have done on their own.

Saved tags may only _add_ rows below the prompt-local group, and may only act on their own when the prompt-local group
is empty. Everything below follows from that.

## Change 1 — leading-group auto-accept (this repo, Python only)

Replace the "exactly one candidate" test in `_try_file_completion_tab`'s placeholder branch with "the leading non-empty
source group holds exactly one candidate". Candidates arrive ordered prompt-first, then common, so the leading group is
simply the run of candidates sharing `candidates[0]`'s source, and the row to accept is always index `0` — which
`_open_placeholder_completion` has already selected. Concretely:

- one prompt-local match plus any number of saved matches → insert the prompt-local match outright (the restored
  behavior);
- zero prompt-local matches and exactly one saved match → insert the saved match outright (keeps today's D5 behavior and
  the `test_manual_trigger_accepts_a_lone_saved_match_outright` contract);
- anything else → open the menu, as today.

Note that the rule is deliberately independent of whether the prefix after `<` is empty. A bare `<` with exactly one
prompt-local tag inserted that tag before saved tags existed, and under the governing rule it must again.

Put the predicate in `src/sase/ace/tui/widgets/placeholder_completion.py` as a small pure function over
`PlaceholderCompletionResult` (a name such as `placeholder_lone_leading_match` reads well) so it is unit-testable
without a running app, and read the source defensively the way `_visible_placeholder_sources` already does rather than
assuming the metadata type. Phrase the predicate in terms of "the source of the first row", not a hard-coded `"prompt"`
literal, so a future third source stays correct by construction. Update the `_try_file_completion_tab` docstring, which
currently asserts the old "a single match is inserted directly" rule.

## Change 2 — literal-zone tags rejoin the candidate list (sase-core)

Open the Rust core with the `/sase_repo` skill (`sase repo open sase-core -r ...`) and use the printed path for every
read and write.

In `crates/sase_core/src/editor/placeholder.rs`, `build_placeholder_completion_candidates` should stop discarding
non-raw spans and instead collect them in a second pass, so the prompt-local group is ordered "raw spans in document
order, then literal-zone spans in document order". Both passes keep the existing behavior in every other respect: the
span under the cursor is still excluded, the case-insensitive prefix filter is unchanged, and the shared `seen` set
still collapses duplicates — so a tag written both live and in backticks appears once, at its live position, and a saved
tag that duplicates either one still collapses onto the prompt-local row. Common candidates continue to be appended
last.

Both literal-zone and live spans stay `PlaceholderCandidateSource::Prompt`. Do not mint a third source variant: the
menu's story is "this prompt" versus "saved", the row badge and the `<> prompt   ◆ saved` legend already encode exactly
that, and a third badge would add wire surface, panel states, and docs for a distinction the user does not need at the
moment of choosing a completion. Ranking literal-zone rows after live rows is the only signal needed, and it is the one
that matters because it decides what Change 1 auto-accepts. Update the doc comments on the function and on
`PlaceholderCandidateSource::Prompt` to say that document candidates include literal-zone text, ranked last within the
group.

No wire, binding, or signature change follows from this: `PlaceholderCompletion` and its serde shape are untouched, so
`crates/sase_core_py/src/lib.rs` and the LSP call site in `crates/sase_xprompt_lsp/src/server.rs` need no edits. The
xprompt LSP picks the behavior up for free, which is correct — it should agree with the TUI.

Commit it in `sase-core` as a `fix(editor): ...` Conventional Commit so release-plz cuts a patch release inside the
existing `<0.13.0` window. Per `sase-core`'s `AGENTS.md`, do **not** hand-edit `[workspace.package].version`, crate
versions, or path-dependency pins.

## What deliberately does not change

State these in the commit message; they are the boundary that makes Change 2 safe.

- **Highlighting stays raw-only.** `_build_highlight_map` in `_placeholder_highlight.py` keeps its `if not span.raw`
  skip, because the cyan overlay means "this will be substituted". The `placeholder_raw_only_highlight_120x40.png`
  snapshot must not move.
- **Submit-time collection and substitution stay raw-only.** `raw_placeholder_fields` and `substitute_raw_placeholders`
  are untouched; backticked tags still survive literally through launch.
- **The durable saved-tag store stays raw-only.** `_placeholder_texts` in `src/sase/history/prompt_placeholders.py`
  keeps its `span.raw` filter. A code fence full of `<div>` should not pollute a store that outlives the prompt; inside
  one prompt the same noise is bounded and prefix-filtered, which is why the completion path can afford it and the store
  cannot.
- **Out of scope: saved tags shadowing prompt-word completion.** With no prompt-local tags, typing `<xy` now opens a
  saved-tag menu where prose/history word completion used to run, and `_structured_completion_claims_cursor`
  (`_file_completion_refresh.py`) reports the cursor claimed for a bare `<` backed only by saved tags. That is D5
  behaving as designed and is a separate judgement call from the two regressions here; leave both paths alone and do not
  widen this tale to cover them.

## Cross-repo sequencing and the published-core floor

Land the `sase-core` commit first. Local dev installs and CI both build `sase_core_rs` from the checkout (`Justfile`'s
`sase_core_dir`; `.github/workflows/ci.yml` checks `sase-org/sase-core` out and builds a wheel), so the new Python tests
pass before any release exists. The `published-core-minimum-smoke` lane only smokes binding presence and telemetry round
trips, so it does not exercise this behavior against the older published minimum.

Raising `sase-core-rs>=0.12.5,<0.13.0` in `pyproject.toml` is a separate `build(deps):` commit once release-plz has
published the release containing the fix — matching how `702f1aece` and `ab6f07a68` handled earlier core bumps. If that
release has not been published by the time the rest of the work is ready, land everything else and report the pending
floor bump explicitly in the final summary rather than blocking on it.

## Tests

`sase-core`, in `placeholder.rs`'s test module:

- Rewrite `completion_candidates_exclude_literal_spans` into its replacement: the existing fixture
  ``"`<alpha>`\n```\n<alpine>\n```\n<apricot> use <a|>"`` must now yield `["apricot", "alpha", "alpine"]`, proving both
  inclusion and the raw-before-literal ranking across inline and fenced zones.
- Add a case where the same text appears live and in backticks and is also present in `common`, asserting one row at the
  live position with source `Prompt`.
- Leave `extracts_strict_single_line_spans_including_code`, `summarizes_exact_raw_fields_with_bounded_context`, and
  `substitutes_only_mapped_raw_spans_without_rescanning_values` untouched and passing — they are the guard that Change 2
  did not leak into highlighting or substitution.

This repo, in `tests/ace/tui/widgets/test_placeholder_completion.py`:

- Replace `test_builder_excludes_literal_placeholders_from_prompt_candidates` with a positive test over text that mixes
  a live tag, a backticked tag, and a saved tag, asserting order and sources `["prompt", "prompt", "common"]`.
- Add `Ctrl+T` coverage for each arm of the leading-group rule: one prompt-local match plus several saved matches
  inserts the prompt-local one and closes the menu (use the `Rebind <ctrl+enter> …` shape from the repro); two
  prompt-local matches plus a saved match leaves the menu open with nothing inserted; zero prompt-local matches plus two
  saved matches leaves the menu open.
- Add a case where the only prompt-local candidate comes from a literal zone and `Ctrl+T` inserts it outright — the
  single test that proves both regressions are fixed together.
- Keep `test_manual_trigger_accepts_a_lone_saved_match_outright`,
  `test_manual_trigger_shows_the_full_saved_list_at_a_bare_bracket`, and
  `test_disabled_feature_reproduces_todays_placeholder_menu` passing unchanged.
- Add or extend a test asserting the automatic trigger still never auto-accepts.

`tests/history/test_prompt_placeholders.py` and the highlighting tests must pass unchanged.

## Documentation

- `docs/ace.md`, "Raw Placeholder Inputs": the sentence claiming literal-zone tags "are not offered as common
  placeholder completions" now needs to distinguish the two facts — they are still not highlighted, not recorded as
  saved tags, and not collected on submit, but their text is offered as a current-prompt completion candidate ranked
  after live tags.
- `docs/ace.md`, the "Placeholder completion" bullet and the automatic-completion paragraph that follows it: document
  the literal-zone ranking and restate the `Ctrl+T` rule as "a lone match in the highest-priority group is inserted
  outright; saved tags never suppress that for a current-prompt match".
- `docs/configuration.md`'s common-placeholder paragraph: confirm it still reads correctly given that recording stays
  raw-only, and adjust only if it now overclaims.
- Per `src/sase/ace/CLAUDE.md`, check the `?` help popup. `binding_common.py`'s entry is the generic
  `("<...>", "Complete raw placeholders")`, so no change is expected; if one is made, keep the 57-character box rules.

## Validation

Run `just install` first — the workspace may be stale — then `just rust-check` for the core change and `just check`
here. Before finishing, re-run the two reproductions from the Context section through
`build_placeholder_completion_result` against the real saved store and confirm the outputs flipped to the "before"
shapes, and drive the `Ctrl+T` paths in the TUI at least once so the menu-versus-insert behavior is confirmed in the app
and not only in tests.
