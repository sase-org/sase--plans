---
status: done
tier: epic
title: Fix xprompt free-text argument parsing (`[[...]]` text blocks)
goal: "Free-text xprompt arguments survive prose that contains `]]`, `+`, commas, and
  apostrophes; the `[[...]]` text-block rule has one authoritative definition that every
  Python and Rust argument scanner shares; and a failed xprompt argument binding reports
  itself once, accurately, instead of leaking a stray error from a best-effort
  diagnostic and then failing later with an unrelated directive error.

  "
phases:
  - id: grammar
    title: Canonical text-block closing rule in the Python scanners
    depends_on: []
    size: medium
    description: "grammar: close a `[[...]]` text block at the first `]]` in
      argument-terminator position instead of the first `]]` anywhere, and apply that
      rule to all three Python argument scanners.

      "
  - id: shorthand
    title: Stop round-tripping shorthand free text through `[[...]]`
    depends_on: []
    size: medium
    description: "shorthand: bind `#name: text`, `#name:: text`, and `#name(args): text`
      payloads structurally during expansion so user prose is never re-serialized into
      source syntax and re-lexed.

      "
  - id: diagnostic
    title: Silence and sharpen expansion-failure reporting
    depends_on: []
    size: small
    description: "diagnostic: keep the best-effort unresolved-reference pre-scan from
      printing a fatal expansion error it then swallows, and make an argument-binding
      failure name the call and the surplus positional that caused it.

      "
  - id: decode
    title: Narrow the `+`-to-space decoding to bare colon arguments
    depends_on:
      - grammar
    size: small
    description: "decode: apply the `+` space substitution only on the
      whitespace-delimited bare colon argument form, so `C++` in prose, quoted values,
      and text blocks stop being corrupted.

      "
  - id: rust
    title: Rust core parity for the shared argument grammar
    depends_on:
      - grammar
      - decode
    size: medium
    description: "rust: mirror the text-block closing rule and the narrowed `+` decoding
      in the sase-core editor and agent-launch argument scanners, and reconcile their
      divergent dialects against one shared corpus.

      "
  - id: regression
    title: End-to-end regression coverage and documentation
    depends_on:
      - grammar
      - shorthand
      - diagnostic
      - decode
      - rust
    size: small
    description:
      "regression: add a launch-level regression test built from the real failing
      prompt, add a shared cross-language corpus, and update the xprompt grammar
      documentation."
proposed_by: bbugyi200.athena.0c5
bead_id: sase-sn
create_time: 2026-09-09 19:52:11
---

- **PROMPT:**
  [prompts/202608/xprompt_text_block_args.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/xprompt_text_block_args.md)
- **BEAD:**
  [sase-sn](https://github.com/sase-org/sase--beads/blob/main/pages/sase-sn/README.md)

# Plan: Fix xprompt free-text argument parsing (`[[...]]` text blocks)

## Problem

A `#research_swarm:: <prose>` launch failed with two errors even though the user
supplied no `wait` argument:

```
❌ XPrompt '#research_swarm' argument error: Argument 'wait' expects word (no
spaces), got 'for example).
  - If the `sase memory read <web>` command is used without providing any
`<keyword>`'
Error: DirectiveError: %clan accepts exactly one positional clan name argument.
```

`#research_swarm` declares three inputs: `prompt` (`text`), `wait` (`word`, default
`null`), and `priority` (`int`, default `null`). The user passed exactly one — the
research prose — through the `::` shorthand.

## Root cause

The prose contained this line:

```
`sase memory read <web>:<keyword> [<web>:<keyword> [...]]`
```

Note the `]]`.

`#name:: text` shorthand does not carry the payload as a value. It **re-serializes** the
prose back into source syntax and re-lexes it:
`src/sase/xprompt/_parsing_shorthand.py:114` (`preprocess_shorthand_syntax`) rewrites
`#research_swarm:: <prose>` into `#research_swarm([[<prose>]])`, and
`src/sase/xprompt/processor.py:460` runs that rewrite on every expansion iteration
before the argument parser sees the call.

The argument scanner then closes the synthesized text block at the **first** `]]` it
meets — the one inside `[<web>:<keyword> [...]]`, roughly a third of the way through the
prose. Everything after that point is treated as ordinary argument text, so top-level
commas in the prose become argument separators:

- `src/sase/xprompt/_parsing_args.py:189` `_top_level_delimiter_positions` sets
  `in_text_block = False` on the first `]]`.
- `src/sase/xprompt/_parsing_args.py:99` `_find_matching_delimiter_for_args` does the
  same when locating the closing `)`.

One argument became ten. Argument 2 bound to `wait`, which is typed `word`, and
`InputArg.validate_and_convert` (`src/sase/xprompt/models.py:148`) rejected it.

Verified reproduction (exact input recovered from prompt history):

```python
from sase.xprompt._parsing_shorthand import preprocess_shorthand_syntax
from sase.xprompt._parsing_args import parse_args, find_matching_paren_for_args

out = preprocess_shorthand_syntax(prompt_text, {"research_swarm"})
open_paren = out.index("#research_swarm(") + len("#research_swarm")
close = find_matching_paren_for_args(out, open_paren)
positional, named = parse_args(out[open_paren + 1 : close])
# len(positional) == 10   (expected 1)
# positional[1] == "for example).\n  - If the `sase memory read <web>` command is
#                   used without providing any `<keyword>`"
# positional[0] still starts with a literal "[["  (process_text_block declined to
#                   strip, because the token no longer ends with "]]")
```

### Why the `%clan` error followed

The same defect fires a second time, one layer down. `research_swarm`'s body embeds the
rendered prompt inside a directive text block:

```
%clan(research.{@1}, tribe=research,
summary=[[[bold]RESEARCH PROMPT:[/bold] {{ prompt }}]])
```

`%` directives use the same argument grammar
(`src/sase/xprompt/_directive_collect.py:82-91` calls `find_matching_paren_for_args` and
`parse_args`). Once the prose lands inside `summary=[[...]]`, its `]]` closes the block
early again and the prose commas become extra `%clan` positionals, which
`_collect_clan_paren_args` (`src/sase/xprompt/_directive_collect.py:326`) rejects.

Confirmed directly:

```python
from sase.agent.xprompt_swarm import expand_xprompt_swarms_with_metadata
from sase.agent.multi_prompt_reference_directives import extract_static_clan_directive
seg0 = expand_xprompt_swarms_with_metadata([prompt_text], {})[0].prompt
extract_static_clan_directive(seg0)
# DirectiveError: %clan accepts exactly one positional clan name argument.
```

This second failure is what actually aborted the launch, and it is **not** fixed by
correcting the shorthand alone: any xprompt that interpolates user text into a `[[...]]`
directive argument hits it.

### Why the user saw both errors

`src/sase/main/query_handler/_launch.py:51` runs `scan_query_for_unresolved_references`
as a best-effort diagnostic before the real launch. That function
(`src/sase/xprompt/unresolved.py:48-78`) calls `process_xprompt_references` inside
`except (Exception, SystemExit): return ()`. The processor prints the failure to stdout
and then exits (`src/sase/xprompt/processor.py:604-605`), so the pre-scan **prints the
fatal error and swallows the failure**. The launch then proceeds and dies later on the
`%clan` error, which reads as unrelated. Confirmed: calling
`scan_query_for_unresolved_references(prompt_text)` prints the exact `❌` line from the
proc log and returns `()`.

### The deeper cause: six divergent copies of one grammar

There is no single definition of "split xprompt/directive arguments". Six scanners
implement overlapping dialects:

| #   | Location                                                                                                                                    | Text block                                 | String quotes   | Bracket nesting    | Escapes |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------ | --------------- | ------------------ | ------- |
| 1   | `src/sase/xprompt/_parsing_args.py` `_top_level_delimiter_positions`                                                                        | first `]]`                                 | `'` `"`         | none               | none    |
| 2   | `src/sase/xprompt/_parsing_args.py` `_find_matching_delimiter_for_args`                                                                     | first `]]`                                 | `'` `"`         | matching pair only | none    |
| 3   | `src/sase/xprompt/alt_inspect.py` `_top_level_offsets`                                                                                      | first `]]`                                 | `'` `"` `` ` `` | `([{`/`)]}` depth  | `\`     |
| 4   | sase-core `crates/sase_core/src/editor/xprompt_args.rs` `ArgScanner`                                                                        | first `]]`                                 | `'` `"`         | none               | none    |
| 5   | sase-core `crates/sase_core/src/editor/directive.rs` `QuoteState`                                                                           | first `]]`                                 | `'` `"` `` ` `` | none               | none    |
| 6   | sase-core `crates/sase_core/src/agent_launch/mod.rs` `parse_directive_args`, `parse_directive_args_with_names`, `split_named_directive_arg` | **none** — `[[` is just bracket depth `+2` | `` ` `` `"`     | `([{`/`)]}` depth  | none    |

Scanner 6 happens to parse the failing `%clan(...)` line correctly, because the prose's
single brackets are balanced and keep the depth above zero. Scanners 1–5 do not. That
disagreement is why the ACE editor, the LSP diagnostics, the fan-out planner, and the
launch path can each reach a different conclusion about the same prompt.

## Proposed rule

A `[[` opens a text block. It closes at the first `]]` whose next non-whitespace
character is an **argument terminator** for the region being scanned — one of `,`, `)`,
`}`, `|`, or the end of the scanned region. A `]]` anywhere else is ordinary content.

This is backward compatible with every well-formed existing usage and repairs both
failure modes. Prototyped and verified against the real inputs:

| Input                                                                           | Today                                | With the rule                        |
| ------------------------------------------------------------------------------- | ------------------------------------ | ------------------------------------ |
| shorthand-synthesized call from the failing prompt                              | 10 args, arg 0 keeps a literal `[[`  | 1 arg, correctly stripped            |
| rendered `%clan(research.X, tribe=research, summary=[[...prose with `]]`...]])` | many positionals -> `DirectiveError` | 1 positional + `tribe=` + `summary=` |
| `[[a]], [[b]]`                                                                  | `['[[a]]', '[[b]]']`                 | unchanged                            |
| `foo=[[a]], bar=1`                                                              | `['foo=[[a]]', 'bar=1']`             | unchanged                            |
| `[[x, y]]`                                                                      | `['[[x, y]]']`                       | unchanged                            |
| `[[a [b [c]] d, e]]`                                                            | `['[[a [b [c]] d', 'e]]']`           | `['[[a [b [c]] d, e]]']`             |

Known residual ambiguity, acceptable and to be documented: a text block whose content
_ends_ with `]]` (`[[foo]]]])`) resolves one `]` short, because first-match cannot
distinguish content from terminator there. Prefer an explicit argument form in that
case.

## Phases

### grammar: Canonical text-block closing rule in the Python scanners

Add one shared helper in `src/sase/xprompt/_parsing_args.py` that, given text and the
index of a `[[`, returns the index of the closing `]]` under the rule above, or `None`
when no `]]` is in terminator position (in which case the block runs to the end of the
scanned region, matching today's unterminated-block behavior).

Use it from all three Python scanners so they stop disagreeing:

1. `_find_matching_delimiter_for_args` (`_parsing_args.py:99`) — backs
   `find_matching_paren_for_args` and `find_matching_brace_for_args`, used by the
   processor, the directive collector, `_directive_alt.py`, `_directive_shorthand.py`,
   and `_directive_edit_core.py`.
2. `_top_level_delimiter_positions` (`_parsing_args.py:189`) — backs `parse_args`,
   `parse_arg_spans`, and `_parse_named_arg` for both `,` and `=`.
3. `alt_inspect._top_level_offsets` (`src/sase/xprompt/alt_inspect.py:135`) — the
   `%{a | b}` / `%alt(...)` branch splitter. Its terminator set must include `|`, which
   is why `|` belongs in the shared set.

Keep `process_text_block` (`_parsing_args.py:59`) as-is: it strips the outer `[[`/`]]`
from a token that both starts and ends with them, which becomes correct again once the
token boundaries are right.

Scope note: do **not** change quote handling or add bracket-depth tracking to scanners 1
and 2 in this phase. Those are separate behavior changes (see Out of scope) and would
enlarge the blast radius of a bug fix.

Tests: extend `tests/test_xprompt_parsing.py` and add directive coverage exercising
`%clan(..., summary=[[...]])` with `]]` in the summary. Every row of the compatibility
table above becomes a case.

### shorthand: Stop round-tripping shorthand free text through `[[...]]`

The `grammar` phase makes the round-trip survive `]]`. This phase removes the round-trip
from the expansion path entirely, because re-serializing arbitrary user prose into
source syntax is lossy by construction — the trailing-`]]` ambiguity above and the `+`
corruption in the `decode` phase are both consequences of it.

The structural path already exists and already works. `XPromptReference.parse_arguments`
(`src/sase/xprompt/_parsing_references.py:88-99`) returns the shorthand payload as a
single positional argument without re-lexing, which is why the xprompt-swarm launch path
(`src/sase/agent/_xprompt_swarm_parsing.py:140-153`) parsed this prompt correctly while
the processor did not. `processor.py:518-540` also already reads a trailing `: `/`:: `
payload directly, as a fallback "for references introduced mid-iteration".

Work:

- In `src/sase/xprompt/processor.py`, resolve bare `#name: text` and `#name:: text`
  references structurally (shared reference lexer + `parse_arguments`) instead of
  relying on `preprocess_shorthand_syntax` passes 2 and 3.
- Handle `#name(args): text` structurally too, replacing pass 1
  (`_preprocess_paren_shorthand`).
- Leave `preprocess_shorthand_syntax` in place for the detection-only callers
  (`src/sase/agent/multi_prompt_xprompts.py:22`,
  `src/sase/xprompt/workflow_validator.py:81`,
  `src/sase/xprompt/workflow_validator_checks.py:339`), which only need names and now
  parse correctly thanks to the `grammar` phase.

Decision to make and record, with a recommendation: the two existing paren-shorthand
implementations disagree. `_preprocess_paren_shorthand` appends the payload as an extra
**positional** (`#pr(foo, status=ready): do the thing` becomes
`#pr(foo, status=ready, [[do the thing]])`), while the `processor.py` fallback assigns
it to the first declared input not already supplied by **name**. Unify on the positional
append, which is what runs today for user-typed prompts and what `docs/xprompt.md`
documents; if a test pins the named-binding behavior, keep the fallback semantics only
for the mid-iteration case it was written for and say so in a comment.

Tests: `tests/test_xprompt_processor_shorthand.py` and
`tests/test_xprompt_processor_args.py`. Add cases where the payload contains `]]`, `[[`,
commas, apostrophes, and an unbalanced `(`.

### diagnostic: Silence and sharpen expansion-failure reporting

Two independent problems, both small:

1. `scan_query_for_unresolved_references` (`src/sase/xprompt/unresolved.py:48-78`) is
   documented as best-effort and swallows `SystemExit`, but the failure it swallows has
   already been printed to stdout by `process_xprompt_references`
   (`src/sase/xprompt/processor.py:604-605`). Give `process_xprompt_references` an
   opt-in that raises `XPromptError` instead of printing and exiting, and have the
   pre-scan use it. Prefer that to redirecting stdout, and prefer it to changing the
   default, which many callers rely on. Regression test: the pre-scan emits nothing on a
   prompt whose expansion fails.
2. `InputBindingError` currently surfaces only the offending value
   (`src/sase/xprompt/input_binding.py:70-88` via `models.py:148`). When a call supplies
   more positionals than the author intended, say so: name the xprompt, the count
   received versus declared, and which declared input the surplus value landed on. Had
   this message existed, the reported failure would have been self-diagnosing.

### decode: Narrow the `+`-to-space decoding to bare colon arguments

`decode_xprompt_arg_value` (`src/sase/xprompt/_parsing_args.py:26`) unconditionally
rewrites `+` to a space. Its own docstring scopes the rule to bare colon arguments,
which are whitespace-delimited — but it is applied to every argument value on every
syntax. Confirmed corruption:

```python
iter_xprompt_references("#research_swarm:: Compare C++ and Rust for A+B work.")[0].parse_arguments()
# (['Compare C   and Rust for A B work.'], {})
```

Apply the substitution only on the bare, unquoted `#name:a,b` form. Do not apply it to
paren arguments, quoted values, `[[...]]` text blocks, backtick colon arguments, or
`: `/`:: ` free text. Call sites to audit:

- `src/sase/xprompt/_parsing_args.py` — `parse_args`, `parse_workflow_reference`
- `src/sase/xprompt/_parsing_references.py:88-99` — `parse_arguments`
- `src/sase/agent/_xprompt_swarm_parsing.py:145-153`
- `src/sase/xprompt/workflow_executor_steps_embedded_types.py:55-59`
- `src/sase/xprompt/workflow_validator_extract.py:124`
- `src/sase/xprompt/processor.py:541-549`

Keep a regression test for the documented `Application+Support` bare-colon case so the
narrowing does not become a removal.

### rust: Rust core parity for the shared argument grammar

The argument grammar is shared backend behavior, so the Rust core must agree with
Python. Open the core repo with `/sase_repo` (`sase repo open sase-core`) and use only
the path it prints.

- `crates/sase_core/src/editor/xprompt_args.rs`: apply the terminator-position rule in
  `ArgScanner::step`, in `find_matching_bracket_for_args`, and in `decoded_value`;
  narrow `decode_xprompt_arg_value` to match the `decode` phase.
- `crates/sase_core/src/editor/directive.rs`: apply the same rule in `QuoteState`, which
  backs `split_top_level_clauses`, `find_top_level_equals`, and
  `find_matching_paren_quoted`.
- `crates/sase_core/src/agent_launch/mod.rs`: `parse_directive_args`,
  `parse_directive_args_with_names`, and `split_named_directive_arg` have no text-block
  state at all and treat `[[` as bracket depth. Reconcile them with the shared rule.
  Where full convergence is too large for this phase, add text-block handling and record
  the remaining dialect differences (backtick quoting, escape handling) as
  `PROPOSED FOLLOW-UP:` notes on the phase bead rather than widening scope.

Add Rust unit tests using the same corpus as the Python tests, including the failing
`%clan(..., summary=[[...]])` line. After the core change lands, rebuild the binding
(`just rust-install`) and confirm the Python callers still pass.

### regression: End-to-end regression coverage and documentation

- Add a launch-level regression test built from the real failing prompt shape: a
  `#name:: <prose containing "]]" and commas>` reference to a multi-segment xprompt
  whose body interpolates the payload into `%clan(..., summary=[[...]])`. Assert one
  positional binding, no `[[` leakage into the bound value, a clean
  `extract_static_clan_directive` result, and no stray output from the launch pre-scan.
  Do not vendor the user's research prose into the repository; construct an equivalent
  minimal fixture.
- Add a small shared corpus file used by both the Python and Rust tests so the two
  implementations cannot drift again silently.
- Update the "Text Blocks" section of `docs/xprompt.md:588-620`: state the closing rule,
  state that shorthand payloads are bound structurally rather than rewritten, and
  document the trailing-`]]` ambiguity and its workaround.

## Verification

Every phase runs `just check` before returning. The combined tree runs `just check-full`
through `/sase_monitor`, because this change touches the shared argument grammar used by
xprompt expansion, `%` directives, fan-out planning, the LSP diagnostics, the snippet
catalog, editor completion, and the workflow validator.

## Out of scope

Deliberately excluded; file as task beads rather than expanding this epic.

- **Unbalanced apostrophes swallow separators.** `_top_level_delimiter_positions` treats
  `'` and `"` as string quotes even in free prose, so `#f(it's fine, b)` parses as one
  argument. After this epic, prose lives inside a correctly delimited text block where
  quotes are inert, so the practical impact is much smaller. Changing quote handling is
  a separate behavior change.
- **Comma splitting ignores nested parentheses.** `find_matching_paren_for_args` tracks
  nesting but `_top_level_delimiter_positions` does not, so `#f(outer (a, b) c)` splits
  into two arguments. Fixing the asymmetry is cheap but changes existing parses; it
  deserves its own change.
- **`sase/memory/xprompts.md`.** The `Invoke` section documents `[[ ... ]]` multi-line
  text without a closing rule and should gain one. Memory files must not be edited
  without explicit user permission in the conversation that requests it, so this epic
  does not touch them. The land agent should surface the proposed wording for the
  project owner to approve, after which `sase memory init` regenerates the derived
  instruction files.
- **No feature flag.** This is a defect fix that restores intended behavior on an
  existing path. It is not a disabled beta, not an early-landed path, and not a
  deprecation whose old branch must stay reachable, so it does not meet the flag
  criteria.
