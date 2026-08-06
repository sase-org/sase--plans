---
tier: tale
title: Wrap agent prompts at the repo-wide 120-column width
goal:
  Agent prompts wrap at 120 columns on every surface — explicit gf / Ctrl+G f formatting in the ACE prompt input,
  launch-time preprocessing, and the published prompt document — retiring the 80-column agent-prompt special case so
  prompt text matches the width already used for plans, SDD files, generated skills, and repo Markdown.
proposed_by: bbugyi200.athena.ua
create_time: 2026-08-06 15:13:05
status: done
---

- **PROMPT:**
  [prompts/202608/agent_prompt_wrap_width_120.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/agent_prompt_wrap_width_120.md)

# Plan: Wrap agent prompts at the repo-wide 120-column width

## Context

Prettier prose wrapping for agent prompts currently has two policies, and the split does not fall where it appears to.

Measured behavior at `HEAD` (verified by calling both helpers with prettier installed):

| Surface                                                          | Entry point                                          | Width |
| ---------------------------------------------------------------- | ---------------------------------------------------- | ----- |
| `gf` / `Ctrl+G f` in the ACE prompt input                        | `PromptFormatMixin.format_prompt_markdown()`         | 80    |
| Launch-time prompt preprocessing (what the provider receives)    | `preprocess_prompt_late()`                           | 80    |
| Archived prompt document published to the `agents` sidecar       | `render_prompt_document()`                           | 120   |
| Plans, SDD writes, generated skills, commit hooks, notifications | `format_with_prettier()` default                     | 120   |
| Repo Markdown (`just fmt-md`), bead prose wrap                   | `Justfile`, `markdown_wrap.DEFAULT_PROSE_WRAP_WIDTH` | 120   |

The two surfaces named in the request already share one width. `src/sase/ace/tui/widgets/_prompt_format.py` and
`src/sase/llm_provider/preprocessing.py` both call `sase.file_references.format_agent_prompt_markdown()`, which is a
single-line delegation to `format_with_prettier(text, print_width=AGENT_PROMPT_WRAP_WIDTH)` with
`AGENT_PROMPT_WRAP_WIDTH = 80`. That seam was introduced deliberately (`b09728a8c`, plan
`202606/agent_prompt_wrap_width.md`) and the `gf` keymap was then built to reuse it (`fac33c7a2`, plan
`202607/prompt_input_format_keymap.md`).

The longer width that is visible on agent prompts comes from a third path: `prepare_prompt_archive()` reads the raw
prompt from `raw_xprompt.md` and re-renders the published prompt document through `format_with_prettier(document)` at
the 120-column default. So the observable inconsistency is real — an 80-column editor buffer and an 80-column launched
prompt against a 120-column published prompt artifact — but it is not a disagreement between `gf` and launch-time
preparation.

That distinction matters for scope. Changing only the `gf` keymap to 120 would introduce a _new_ inconsistency: the user
would format the buffer at 120, press submit, and the launch pipeline would silently reflow the same text back to 80
before the provider ever saw it. The only change that satisfies "use the longer width, and make these agree" is to
retire the 80-column agent-prompt special case entirely, so every agent-prompt surface lands on
`DEFAULT_MARKDOWN_WRAP_WIDTH` (120) alongside the rest of the repo's Markdown.

## Goal

Agent prompts wrap at 120 columns everywhere: explicit `gf` / `Ctrl+G f` formatting in the prompt input, launch-time
preprocessing, and the published prompt document all produce identical prose wrapping, matching the width already used
for plans, SDD files, generated skills, notifications, and repo Markdown.

## Non-goals

- No change to `format_with_prettier()`'s Markdown parser, `--prose-wrap=always` behavior, underscore unescaping,
  `SASE_DISABLE_PRETTIER` opt-out, timeout, or failure fallback.
- No change to fenced-code-block or `%xprompts_enabled:false/true` disabled-region protection, and no change to where
  formatting sits in the preprocessing pipeline.
- No change to the `gf` / `Ctrl+G f` binding table, hint labels, help modal text, worker/staleness lifecycle, or
  cursor-mapping logic in `_prompt_format.py`.
- No relocation of prompt formatting into the Rust core. This is an existing Python-side Prettier subprocess policy and
  `crates/sase_core` has no prompt-wrapping code today; moving the boundary is separate work.
- No change to `markdown_wrap.DEFAULT_PROSE_WRAP_WIDTH` (bead prose wrapping), which is already 120 and is asserted
  equal to `DEFAULT_MARKDOWN_WRAP_WIDTH` by `tests/test_bead/test_markdown_wrap.py`.

## Design

### 1. Retire the 80-column agent-prompt constant

In `src/sase/file_references.py`:

- Delete `AGENT_PROMPT_WRAP_WIDTH = 80` and the two-line comment above it. Nothing in `src/` imports it besides
  `format_agent_prompt_markdown()`; the only other importer is `tests/test_format_with_prettier.py`.
- Update the comment above `DEFAULT_MARKDOWN_WRAP_WIDTH = 120` so it reads as the single Markdown prose wrap width for
  both saved artifacts and agent prompts, rather than "for saved Markdown artifacts".
- Change `format_agent_prompt_markdown()` to `return format_with_prettier(text)` and rewrite its docstring: it is no
  longer a narrower policy, it is the named seam that keeps the editor-side and launch-side prompt formatting provably
  identical and pins them to the repo-wide width.
- Fix the `format_with_prettier()` docstring's `print_width` argument description, which currently claims "Agent prompt
  preprocessing passes `AGENT_PROMPT_WRAP_WIDTH`; other callers keep the default."

**Keep `format_agent_prompt_markdown()` rather than inlining it.** After this change it is a passthrough, but it is the
documented coupling point between `_prompt_format.py` and `preprocess_prompt_late()`, it is what
`tests/ace/tui/widgets/test_prompt_format.py` patches (10 call sites), and it keeps a single place to change if prompt
width is ever revisited. Deleting it would make the coupling implicit and would churn two unrelated modules and a widget
test module.

**Keep the keyword-only `print_width` parameter on `format_with_prettier()`.** No production caller will override it
after this change, but it stays exercised by the fallback tests, it is symmetric with the identical parameter on
`format_markdown_files_with_prettier()` (which also has no overriding caller today), and removing it would touch the
signature every other Markdown caller depends on for no behavioral gain.

### 2. Refresh the one stale inline comment

`src/sase/llm_provider/preprocessing.py` line 189 reads
`# 6. Prettier formatting (agent prompts wrap narrower than saved artifacts)`. Replace the parenthetical so it no longer
asserts a narrower prompt width — e.g. `# 6. Prettier formatting (shared agent-prompt Markdown policy)`.

### 3. Confirm no other surface encodes 80

A repo sweep found no other agent-prompt width coupling to update:

- `src/sase/ace/tui/widgets/_prompt_input_bar_g_prefix_actions.py` renders the hint as the literal string
  `"format prompt"`, with no width in the label, help modal, or docs.
- The ACE PNG visual snapshots do not exercise prompt Markdown formatting; the visual conftest disables Prettier.
- The only other `80`-column comments in the tree (`prompt_panel/_agent_context_common.py`,
  `tests/ace/tui/widgets/test_agent_memory_reads.py`) are Rich render-width contracts, unrelated to Prettier.

The implementing agent should re-run `rg 'AGENT_PROMPT_WRAP_WIDTH|print_width|80 column|80-column'` after the edit to
confirm nothing was missed.

## Tests

1. `tests/test_format_with_prettier.py`
   - Drop `AGENT_PROMPT_WRAP_WIDTH` from the import block.
   - Rename `test_agent_prompt_formatter_uses_80` to reflect the new policy (e.g.
     `test_agent_prompt_formatter_uses_default_width`) and invert its assertions: the captured argv must contain
     `--print-width=120` and must not contain `--print-width=80`.
   - In `test_preprocess_prompt_late_uses_named_agent_prompt_formatter`, drop the trailing
     `assert AGENT_PROMPT_WRAP_WIDTH == 80`. The test's real value is `mock_formatter.assert_called_once()` — that
     launch-time preprocessing goes through the shared named policy rather than calling Prettier directly. Keep that
     assertion.
   - Leave the `print_width=80` arguments in the missing-prettier / disabled / failure / timeout fallback tests alone:
     they assert the fallback path ignores the width, and an explicit non-default value is still the clearest way to
     show the parameter is honored as a parameter.

2. Add one regression test asserting the widths now agree. In `tests/test_format_with_prettier.py`, capture argv for
   `format_agent_prompt_markdown("some prose")` and `format_with_prettier("some prose")` under the same `shutil.which` /
   `subprocess.run` patches and assert the two argv lists are equal. This is the test that would have caught the
   original split and will fail loudly if a future change re-forks the prompt width without updating the
   published-artifact path.

3. `tests/ace/tui/widgets/test_prompt_format.py` needs no change — every case patches
   `sase.ace.tui.widgets._prompt_format.format_agent_prompt_markdown` and asserts lifecycle behavior (staleness, cursor
   mapping, Vim mode, notifications), not wrap width.

4. Watch for width-sensitive assertions elsewhere. `b09728a8c` had to make two assertions in
   `tests/test_fix_just_workflow.py` whitespace-insensitive because prompt prose began straddling the 80-column
   boundary; those assertions are already whitespace-insensitive and should pass at 120, but any test that asserts on
   post-preprocessing prompt text with embedded newlines is a candidate for breakage in either direction. Fix such
   assertions by making them whitespace-insensitive rather than by re-hardcoding a wrapped shape.

## Verification

- `just install`
- `just check`
- `just check-full` before landing. The wrap width feeds `preprocess_prompt_late()`, which is reached by
  `invoke_agent()`, workflow prompt steps, and `sase xprompt expand`, so the blast radius is wider than the scoped test
  selection is likely to infer from the import graph.
- Manual smoke check in the ACE TUI: type a long single-line paragraph into the prompt input, press `gf`, and confirm it
  reflows at 120 rather than 80.

## Rollout notes

This is a user-visible text-shape change to prompts sent to providers and to prompt text stored in agent artifacts. It
partially reverses `b09728a8c`, which chose 80 columns for launch-time prompts in June 2026; the project owner has asked
for the longer width, and the published prompt artifact has been rendering at 120 the whole time. Existing archived
prompt documents are unaffected — they are already 120-column. No migration, config, or memory-file change is required.
