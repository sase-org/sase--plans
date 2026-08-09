---
tier: tale
title:
  Stop swarm expansion from splicing an inherited VCS ref into a multi-line directive
goal:
  An xprompt swarm whose first segment opens with a multi-line %clan(...) directive
  launches successfully when the prompt carries a VCS/project ref, instead of failing
  with a bogus "Unsupported keyword on %clan" error.
size: medium
proposed_by: bbugyi200.athena.ut
create_time: 2026-08-07 13:19:32
status: done
---

- **PROMPT:**
  [prompts/202608/swarm_multiline_directive_vcs_split.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/swarm_multiline_directive_vcs_split.md)
- **AGENTS:**
  - [bbugyi200.athena.ut](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.ut.md)
- **COMMITS:**
  - [4c7c635](https://github.com/sase-org/sase/commit/4c7c635d2db3b6b882a8f1844c4153771d73dc91)
    — fix(agent): stop swarm expansion splicing VCS refs into multi-line %clan(...)
    directives

# Plan: Fix VCS-ref injection inside multi-line `%clan(...)` args during swarm expansion

## Problem

Launching an xprompt swarm from Telegram fails when the launch prompt carries a
VCS/project ref and the swarm's first segment starts with a `%clan(...)` directive whose
argument list spans more than one line:

```
#gh@sase #research_swarm:: I want to better understand and improve the way that SASE
gates are able to accept custom user input/arguments ...
```

Telegram replies with:

```
Failed to launch agent: Unsupported keyword on %clan: #gh:gh_sase-org__sase summary=.
Only summary=, summary_script=, and tribe= are supported.
```

The message is self-contradictory (`summary=` is listed as both unsupported and
supported), which is the tell that the prompt text itself was corrupted before it
reached the directive parser.

## Root cause

`_split_leading_directive_prefix` in `src/sase/agent/_xprompt_swarm_parsing.py:54`
splits a segment into "leading `%directive` run" + "body" **one physical line at a
time**. Both matchers it uses are line-scoped and paren-naive:

- `_DIRECTIVE_PREFIX_RE` (`src/sase/xprompt/_parsing_vcs_tags.py:20`) requires a
  _balanced_ `(...)` inside the text it is handed. Handed only the first line of a
  directive whose argument list continues on the next line, it does not match.
- `_DIRECTIVE_LINE_RE` (`src/sase/agent/_xprompt_swarm_parsing.py:23`) then accepts that
  same first line as a whole "directive-only line", because it only checks that the line
  starts with `%name(` — it never verifies the parenthesis closes.

The `#research_swarm` xprompt's first segment starts with exactly that shape (definition
at `/home/bryan/sase/xprompts/research_swarm.md`; view it with
`sase xprompt show research_swarm`):

```
%clan(research.{@1}, tribe=research,
summary=[[[bold]RESEARCH PROMPT:[/bold] {{ prompt }}]]) %id:research.{@1}.cdx
%wait(priority=20) %model:@research_a {{ prompt }} #research
```

So the split lands _inside_ the `%clan(...)` argument list: prefix is line 1 only, body
starts at `summary=`. `_prefix_segment_vcs_ref`
(`src/sase/agent/_xprompt_swarm_parsing.py:240`) then joins
`prefix + vcs_ref + " " + body`, producing:

```
%clan(research.abc, tribe=research,
#gh:gh_sase-org__sase summary=[[...]]) %id:research.abc.cdx
```

`parse_args` now sees a named argument whose key is the literal string
`#gh:gh_sase-org__sase summary`, and `_collect_clan_paren_args`
(`src/sase/xprompt/_directive_collect.py:188`) rejects it with the confusing message
above. Full failure chain from the Telegram log
(`~/.sase/axe/logs/lumberjack-telegram.log`, 2026-08-07 13:01:49):
`launch_agents_from_cwd` → `launch_cwd_agents.py:216` → `multi_prompt_launcher.py:103` →
`multi_prompt_launch_execution.py:111` → `multi_prompt_launch_plan.py:167` →
`extract_static_clan_directive` → `collect_prompt_directive_matches`.

Confirmed reproduction (current `master`, workspace venv):

```python
from sase.agent.xprompt_swarm import expand_xprompt_swarms_with_metadata
records = expand_xprompt_swarms_with_metadata(
    ["#gh:gh_sase-org__sase #research_swarm:: Improve how SASE gates accept input"], {}
)
print(records[0].prompt)
# %clan(research.{...}, tribe=research,
# #gh:gh_sase-org__sase summary=[[[bold]RESEARCH PROMPT:[/bold] Improve ...]]) %id:...
```

### Why this only shows up on mobile/Telegram-shaped launches

The corrupting insert happens only on the _inherited_ VCS-ref path
(`prepend_inherited_vcs_ref` → `_prefix_segment_vcs_ref`), which runs when the launch
prompt itself carries a ref (`#gh:sase`, `#gh@sase`, …) that each generated sub-segment
must inherit. A bare `#research_swarm:: ...` launch takes the _default_ VCS path
(`normalize_default_vcs_workflow_segment`), which matches `_DIRECTIVE_PREFIX_RE` against
the whole segment body rather than one line, handles the multi-line `%clan(...)`
correctly, and yields the placement we want:

```
%clan(research.abc, tribe=research,
summary=[[...]]) %id:research.abc.cdx
%wait(priority=20) %model:@research_a #git:home Improve gates #research
```

That existing, working output is the reference shape the fixed inherited path must
produce.

## Fix

Make the swarm splitter's notion of "leading `%directive` run" paren-aware instead of
line-scoped, by resolving parenthesized argument lists with the same matcher the
directive parser already uses.

### 1. Add a paren-aware directive-run scanner

Add a module-private helper to `src/sase/agent/_xprompt_swarm_parsing.py` (keep it
beside its only caller; do not widen `sase.xprompt`'s public surface):

```python
def _directive_run_end(segment: str, pos: int) -> int | None:
    """Return the offset just past the ``%directive`` run starting at *pos*."""
```

Semantics — deliberately byte-identical to `_DIRECTIVE_PREFIX_RE` except for how
argument lists are delimited:

- Match one directive token with `%[^\s(]+`. If none, stop.
- If the token is immediately followed by `(`, resolve the argument list with
  `find_matching_paren_for_args` (import it from `sase.xprompt._parsing`, which already
  re-exports it — `src/sase/xprompt/_parsing.py:296` — and is already this module's
  import source for `_DIRECTIVE_PREFIX_RE`). Advance past the matching `)`. If there is
  no matching `)`, the directive is genuinely malformed: stop **without** consuming the
  token, so the prompt reaches the directive parser intact and the user gets the real
  error (`Malformed %clan(...) directive: missing closing ')'.`) instead of a corrupted
  one.
- Require at least one whitespace character after the directive, bounded to the current
  physical line (`[^\S\n]*\n` or `[^\S\n]+`), then absorb the next line's `[ \t]*`
  indent. This preserves today's line-bounded whitespace behavior exactly; a directive
  token with no trailing whitespace (end of segment) is still left to the existing
  `_DIRECTIVE_LINE_RE` branch.
- Return the end offset of the last consumed whitespace, or `None` if nothing matched.

Using `find_matching_paren_for_args` (`src/sase/xprompt/_parsing_args.py:164`) — rather
than widening the regex — is what makes this correct, because it respects `[[...]]` text
blocks, quoted strings, and nested parens. That matters here: the swarm interpolates
arbitrary user prose into `summary=[[...]]`, so an unbalanced `(` or a smiley in the
research prompt would defeat any regex-based paren matching.

### 2. Use the scanner in `_split_leading_directive_prefix`

In the existing loop (`src/sase/agent/_xprompt_swarm_parsing.py:68-95`), replace the
per-line `_DIRECTIVE_PREFIX_RE.match(...)` branch with `_directive_run_end`. Keep
everything else as-is: the leading-whitespace capture, the `_DIRECTIVE_LINE_RE`
directive-only-line fallback, `directives.extend(_directive_tokens(...))`, and the
returned `_LeadingDirectiveSplit(directives, prefix, body)` contract. `prefix` must
still preserve original spacing verbatim, since `_prefix_segment_vcs_ref` and
`extract_top_level_xprompt_reference` both rebuild segments from it.

Harden the fallback while you are there: do not let `_DIRECTIVE_LINE_RE` accept a line
that opens a directive argument list it never closes. That is the branch that actually
performed the mid-argument split, and refusing it keeps a malformed directive whole.

Expected result for the reported prompt (matches the bare-launch reference shape above):

```
%clan(research.abc, tribe=research,
summary=[[[bold]RESEARCH PROMPT:[/bold] Improve gates]]) %id:research.abc.cdx
%wait(priority=20) %model:@research_a #gh:gh_sase-org__sase Improve gates #research
```

### Scope notes

- `sase.xprompt`'s other `_DIRECTIVE_PREFIX_RE` consumers (`extract_vcs_workflow_tag`,
  `normalize_default_vcs_workflow_segment`, `_inherit_vcs_workflow_tag_segment`,
  `find_vcs_workflow_tag_prepend_offset`) all match against a whole body, not a single
  line, so none of them can split inside a multi-line argument list. Leave them alone:
  switching them to the scanner would change their greedy-`[\s]+` whitespace handling
  across blank lines and buys nothing for this bug.
- Rust core boundary: prompt directive/VCS-prefix parsing lives entirely in Python today
  (`src/sase/xprompt/` has no `sase_core_rs` calls outside frontmatter handling). This
  is a localized bug fix in that existing Python parser, not new shared backend
  behavior, so it stays in this repo.

## Tests

Add to `tests/test_xprompt_swarm_vcs_inheritance.py`, using the existing `patch_catalog`
/ `patch_vcs_patterns` / `xp` / `expand_xprompt_swarms` helpers from
`tests/_xprompt_swarm_helpers.py`:

1. **Regression (the reported failure).** A catalog xprompt whose first segment opens
   with a `%clan(...)` directive split across two lines exactly like `#research_swarm`
   (`tribe=` on line 1, `summary=[[...]]` closing the parens on line 2), expanded from
   `#gh:sase #swarm:: <text>`. Assert the inherited `#gh:sase` lands _after_ the whole
   leading directive run and that `%clan(` … `)` is contiguous — i.e. the segment does
   not contain `,\n#gh:sase`.
2. **Parser agreement.** Feed that expanded segment to
   `sase.agent.multi_prompt_reference_directives.extract_static_clan_directive` and
   assert it returns the clan directive without raising `DirectiveError`. This is the
   assertion that actually pins the user-visible bug shut.
3. **User prose with an unbalanced paren.** Same shape, with `<text>` containing an
   unmatched `(`, so the fix is pinned to `find_matching_paren_for_args`' text-block
   awareness rather than to a regex that happens to balance.
4. **Malformed directive stays malformed.** A segment whose `%clan(` never closes must
   not get a VCS ref spliced into its argument list; the ref goes ahead of the directive
   run and the prompt still parses to the parser's own malformed-directive error rather
   than an "unsupported keyword" one.

Keep the existing single-line-directive inheritance assertions in that file passing
unchanged — they encode the byte-parity requirement for the scanner.

## Verification

```bash
just install
just test-scoped                 # or: .venv/bin/pytest tests/test_xprompt_swarm_*.py
just check
```

Then re-run the end-to-end reproduction and confirm segment 0 is intact:

```bash
.venv/bin/python -c "
from sase.agent.xprompt_swarm import expand_xprompt_swarms_with_metadata
r = expand_xprompt_swarms_with_metadata(
    ['#gh:gh_sase-org__sase #research_swarm:: Improve how SASE gates accept input'], {})
print(r[0].prompt)
"
```

The `%clan(...)` argument list must be contiguous and `#gh:gh_sase-org__sase` must
appear after `%model:@research_a`.

## Out of scope (follow-ups worth filing separately)

- The same Telegram launch logs a non-fatal
  `Failed to resolve bead project 'sase' from launch_prompt` →
  `No workspace plugin detected a workflow type for '/home/bryan/.sase/projects/sase/sase.sase'`.
  That is `_resolve_workspace_for_project` in the `sase-telegram` plugin resolving a
  project _name_ (`sase`) where a ProjectSpec _key_ (`gh_sase-org__sase`) is required.
  Different repo, different bug, harmless here.
- `sase xprompt expand '#gh@sase ...'` crashes with
  `TypeError: argument of type 'NoneType' is not a container` in `sase_github`'s
  `gh_setup.py` because the expand path runs the workflow's `setup` step with
  `gh_ref=None`. Also a separate defect; it made local reproduction of this bug harder.
