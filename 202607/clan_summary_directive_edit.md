---
tier: tale
title: Preserve clan summaries in prompt edits, then land sase-7r
goal: 'Tribe assignment and clan-declaration rewrites preserve the %clan summary arguments
  and shorthand text introduced by epic sase-7r, and the epic is closed out (beads
  closed, symvision cleaned, plan marked done).

  '
create_time: 2026-07-19 21:24:32
status: wip
prompt: 202607/prompts/clan_summary_directive_edit.md
---

# Plan: Preserve clan summaries in prompt edits, then land sase-7r

## Context

Epic sase-7r taught the declaring `%clan` directive two new named arguments — `summary=` (quoted string or `[[ ... ]]`
text block) and `summary_script=` — plus a `::` double-colon shorthand (`%clan:<name>:: <text>` /
`%clan(<args>):: <text>`) that directive parsing rewrites into `summary=[[...]]`
(`src/sase/xprompt/_directive_shorthand.py`). Epic clans launched by `sase bead work` now always declare
`%clan(<epic_id>, tribe=epic, summary_script=sase_clan_summary_epic)` (`src/sase/bead/work.py`), and the chezmoi
`research_swarm` xprompt declares an inline `summary=[[...]]`.

The prompt-edit helpers in `src/sase/xprompt/directive_edit.py` predate the epic and were never taught the new
arguments. Two confirmed gaps (both reproduced against the current tree):

1. **Tribe assignment fails on summary-declaring clans.** `set_prompt_clan_tribe` rejects any `%clan(...)` named
   argument other than `tribe=` ("Cannot update clan tribe: only tribe= is supported on %clan.") and, on success,
   rebuilds the directive as `%clan(<name>, tribe=<tribe>)` from scratch. The Agents-tab tribe modal (`N` keymap,
   `src/sase/ace/tui/actions/agents/_tribe_assignment.py`, including its `set_prompt_tribe` clan path) therefore errors
   for every epic clan member and every research_swarm clan — and would silently drop the summary if the allowlist were
   naively widened without preserving the argument text.
2. **`::` shorthand text dangles after clan-declaration removal.** `demote_prompt_clan_declaration` and
   `rewrite_prompt_clan_member_name` (the retry/relaunch joiner-conversion paths) remove only the directive span itself,
   so a prompt like `%clan:review.x:: [bold]summary[/bold]` becomes `%id(...)` followed by a stray
   `:: [bold]summary[/bold]` line that leaks into the relaunched member's visible prompt. Paren-form declarations
   (including multiline `[[...]]` summary blocks) are already removed cleanly; only the shorthand-captured block is
   missed.

There is no Rust-side counterpart to update: prompt mutation is Python-only glue (the sase-core editor layer covers
completion/hover/diagnostics, which sase-7r.1 already updated), so this fix stays entirely in this repo.

## Fix 1: preserve summary arguments in `set_prompt_clan_tribe`

In `src/sase/xprompt/directive_edit.py`:

- Widen the named-argument allowlist to `{tribe, summary, summary_script}`. Anything else keeps a targeted rejection
  message.
- Preserve non-tribe arguments **verbatim**. `parse_args` returns processed values (dequoted strings, dedented `[[...]]`
  blocks), so rebuilding the directive from parsed values is lossy — especially for multiline text blocks. Instead, edit
  the tribe argument in place within the original paren span: locate top-level argument boundaries in the raw args text
  (respecting quoted strings and `[[ ... ]]` blocks — check `src/sase/xprompt/_parsing_args.py` for a reusable
  boundary/tokenizer helper before writing a new one, and prefer exposing spans from the existing parser over
  duplicating its quoting rules), then replace, insert, or remove just the `tribe=<value>` argument while leaving every
  other byte of the directive untouched.
- Behavior matrix to honor:
  - add tribe (none declared) → insert `tribe=<value>` after the positional clan name, before any other arguments' text
    is disturbed;
  - replace tribe → swap only the tribe value;
  - remove tribe (`tribe=None`) → drop only the tribe argument; a declaration left with summary arguments must remain in
    paren form (never collapse to `%clan:<name>` / bare `%clan(<name>)` reconstruction that discards them);
  - colon form `%clan:<name>` and the `%c` alias keep their current conversions;
  - shorthand forms keep working: editing tribe on `%clan:<name>:: <text>` or `%clan(<args>):: <text>` must leave the
    captured summary text semantically intact (verify by re-extracting directives on the edited prompt and comparing
    `clan_summary`).
- `set_prompt_tribe`'s clan branch and the tribe-assignment mutators need no signature changes; they inherit the fix.

## Fix 2: include shorthand-captured text in clan-declaration removal

When `_set_prompt_directive` / `_directive_spans` remove a `clan` directive (the `demote_prompt_clan_declaration` and
`rewrite_prompt_clan_member_name` paths), extend the removal span to cover a trailing `::`-captured text block. Reuse
the parse-time capture semantics from `src/sase/xprompt/_directive_shorthand.py` (the directive-aware end finder used by
`preprocess_directive_double_colon_shorthand`) so edit-time spans match parse-time capture exactly — capture runs to EOF
or the next line-start `%` directive / `#` reference, blank lines included. Only strip the block when the clan directive
itself is being removed; other directive kinds and non-removal edits are untouched.

## Tests

Extend `tests/test_directive_edit.py` (and the tribe-assignment action tests if any exist under `tests/ace/tui/`):

- `set_prompt_clan_tribe` add/replace/remove tribe on declarations carrying `summary="..."`, inline `summary=[[...]]`,
  multiline `[[\n...\n]]` blocks, and `summary_script=...` — asserting the summary argument text survives byte-for-byte
  and re-extraction yields the same `clan_summary` / `clan_summary_script`;
- the epic-shaped declaration `%clan(<id>, tribe=epic, summary_script=sase_clan_summary_epic)` round-trips through a
  tribe change (this is the exact prompt every `sase bead work` epic member carries);
- unknown named arguments are still rejected with the targeted message; duplicate-argument and summary/summary_script
  mutual-exclusion behavior is unchanged;
- `%c` alias and shorthand (`:: `) forms round-trip through tribe edits;
- `demote_prompt_clan_declaration` and `rewrite_prompt_clan_member_name` on shorthand-form declarations remove the
  captured text (no dangling `:: ...` in the result), while paren and multiline-block forms keep their current clean
  removal.

Run `just install`, then `just check` before finishing.

## Land the epic (final step)

After the fix is green, finish landing epic sase-7r:

1. Close the straggler phase bead first — its implementation is verified complete (commit 734f67a25, plus re-exec
   preservation in `run_agent_directives.py` and the persistence/binding test suites): `sase bead close sase-7r.3`.
2. Close the epic: `sase bead close sase-7r`.
3. AFTER closing, run `just symvision` (epic-symbol whitelist entries for sase-7r expire at close) and remove the stale
   whitelist entries and any unused code it reports.
4. Open the plans sidecar with the `/sase_repo` skill and set `status: done` in the frontmatter of
   `202607/clan_rich_summary.md` (the epic's plan file).

## Risks

- The top-level argument scanner is the delicate piece: it must mirror the parser's quoting and `[[ ... ]]` rules
  (including `]]` terminating a block regardless of markup) or the in-place edit could split an argument mid-value.
  Reusing/parameterizing the existing parser is strongly preferred over a parallel implementation.
- Shorthand-span removal must not regress non-clan directives or prompts where `:: ` appears in ordinary text without a
  preceding clan declaration on the same line.
