---
tier: tale
title: Make batched glossary reads the taught, reliable, and cheap path
goal:
  Agents learn from their always-loaded instruction files to fetch every glossary term
  they need in one `sase glossary read` call, a batch that hits an unknown term reports
  every unresolved reference at once instead of only the first, and the rendered closure
  stops spending tokens on invisible padding and non-actionable footer hints.
size: medium
proposed_by: bbugyi200.athena.069
create_time: 2026-08-18 10:50:58
status: wip
---

# Make Batched Glossary Reads The Taught, Reliable, And Cheap Path

## Goal

Cut the tokens agents spend on the glossary by making one batched read the obvious,
safe, and lean way to consult it.

Three things must be true when this lands:

1. Every agent instruction file tells agents to pass **all** the terms they need to a
   single `sase glossary read` invocation.
2. A batch that names an unknown term reports **every** unresolved reference in one
   error, with near-miss candidates for each, so one retry fixes the whole batch.
3. The rendered closure contains no invisible padding and no footer line that is not
   actionable for the arguments that were actually passed.

## Context: Multi-Term Already Works — Nothing Teaches It

`sase glossary show` and `sase glossary read` **already accept multiple terms today**.
This was verified against the current tree, not assumed:

- `src/sase/main/parser_glossary.py` — `_add_closure_arguments` declares the positional
  as `nargs="+"` for both `show` and `read`.
- `src/sase/glossary/resolution.py` — `resolve_glossary_closure` takes a `roots`
  sequence and `_resolve_roots` resolves and de-duplicates every one of them.
- `src/sase/glossary/render.py` — all three formats iterate `closure.nodes`, which
  already contains multiple `origin == "requested"` roots.
- `src/sase/glossary/cli_read.py` — `_closure_audit_fields` records every requested term
  and every related term in the one audit event.
- `src/sase/completion/emit_zsh.py` — `_nargs_prefix` emits the `*:` repeated-positional
  spec for `nargs="+"`, so `sase glossary show <TAB>` already completes term after term.
- `docs/memory.md` already documents `sase glossary show TERM [TERM ...]`.

Confirmed live:

```console
$ sase glossary show Stitch Patch -d 0
GLOSSARY  sase                                 2 terms · 2 requested · 0 related
...
```

**So there is no argument-parsing work in this plan.** The capability exists; what does
not exist is any reason for an agent to use it, any safe failure mode when a batch has a
typo, or any pressure on the per-read byte cost. This plan closes those three gaps.

### The Measured Cost Of Not Batching

Measured on this project's real 29-term glossary, requesting five terms (`Stitch`,
`Patch`, `Sase Agent`, `Agent Hood`, `Xprompt`) at the default depth and the default
`rich` format:

| Approach                                         | Bytes  | Closure nodes |
| ------------------------------------------------ | ------ | ------------- |
| Five separate reads (what agents are taught now) | 40,545 | 60            |
| One batched read (already possible today)        | 9,179  | 14            |
| One batched read + the render fixes in this plan | ~7,043 | 14            |

Batching alone is a **4.4x** reduction; with the render fixes it is **5.8x (-83%)**.

The saving is structural, not incidental: a glossary definition's closure overlaps
heavily with its neighbours' closures. Five sequential reads print the same related
terms over and over (60 nodes for 5 requested terms); one batched read prints each term
exactly once (14 nodes), and any requested term that another requested term references
is collapsed into the single node it already has.

Of the 9,179 bytes in today's batched output, **2,021 (22%) are trailing whitespace**
that no human ever sees and every agent pays for, plus two footer lines that are printed
unconditionally.

## Design

### 1. Teach Batching Where Agents Actually Read It

The `Glossary Terms` H3 that `sase memory init` renders into `AGENTS.md` and every
provider instruction shim is the only glossary guidance most agents ever see. It is
generated from one constant, `_GLOSSARY_TERMS_INTRO` in `src/sase/amd/_memory.py`, and
today it reads:

> Run `sase glossary read <term> -r "<why>"` before relying on any of these SASE terms;
> it prints that term's definition plus every term the definition depends on. Aliases
> follow in parentheses.

Singular `<term>`, singular "that term's definition". An agent following this literally
issues one call per term, which is exactly the 40,545-byte path.

Replace the constant with:

```python
_GLOSSARY_TERMS_INTRO = (
    "Run `sase glossary read <term> [<term> ...] -r \"<why>\"` before relying on "
    "any of these SASE terms; it prints each term's definition plus every term "
    "those definitions depend on. Pass every term you need in one command — one "
    "batched read costs far fewer tokens than one read per term, because terms "
    "shared between definitions are printed once. Aliases follow in parentheses."
)
```

Design constraints this wording respects:

- It keeps the original sentence shape, so the change reads as a refinement rather than
  a rewrite, and the existing `Aliases follow in parentheses.` sentence survives
  verbatim.
- `<term> [<term> ...]` mirrors argparse's own `TERM [TERM ...]` usage string, so the
  instruction and `--help` teach the identical shape.
- It states the _reason_ ("terms shared between definitions are printed once"), because
  an agent that understands why batching is cheaper will batch in situations the rule
  did not literally anticipate.
- It stays one short paragraph. This text sits in every agent's always-loaded context;
  spending ~20 extra words there to save thousands per read is the right trade, but
  spending 200 would not be.

**This is a memory-file change.** `_GLOSSARY_TERMS_INTRO` feeds `AGENTS.md`,
`CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, and `QWEN.md`. The user explicitly requested
this update in the originating conversation, so the required permission is already
granted; the implementer MUST run `sase memory init` afterward and commit the
regenerated files. Do not hand-edit `AGENTS.md` or the shims.

Mirror the same guidance, in each file's own voice, in:

- `src/sase/main/parser_glossary.py` — `read` and `show` `description=` text and
  `epilog=` examples. Lead each `epilog` with a multi-term example so the first thing a
  reader sees is a batch:

  ```
  sase glossary read Stitch Patch "Agent Hood" -r "Need the patch/stitch vocabulary"
  ```

  Keep the existing single-term examples after it; single-term reads stay valid.

- `docs/memory.md` — the `sase glossary show`/`read` paragraphs and the fenced example
  block near line 145.
- `docs/init.md` (~line 210) and `docs/configuration.md` (~line 542) — both currently
  point at `sase glossary read <term> -r "<why>"`. While editing the
  `docs/configuration.md` sentence, also correct its stale claim that the glossary
  renders a `**GLOSSARY TERMS:**` paragraph; it renders the `Glossary Terms` H3 section
  described in `docs/memory.md`.

### 2. Make A Batch Fail Loudly And Completely

This is the reliability cost that batching introduces, and it is the one design change
that matters most.

Today `_resolve_roots` calls `index.lookup(root)` in a loop, and `lookup` raises on the
first failure. Batch six terms with two typos and you learn about one typo, retry, and
learn about the second — two wasted round trips carrying the full token cost of the
retry loop. The blast radius of a single typo scales with batch size, which is precisely
backwards for a feature whose whole point is larger batches.

Change `_resolve_roots` to resolve every reference, collect the failures, and raise
once.

In `src/sase/glossary/resolution.py`:

```python
@dataclass(frozen=True, slots=True)
class GlossaryLookupFailure:
    """One reference that did not resolve, with its near-miss candidates."""

    reference: str
    candidates: tuple[str, ...]
```

Keep `GlossaryLookupError.__init__(reference, candidates=())` exactly as it is — the
single-reference message is depended on by `sase/glossary/mutation.py:157`,
`tests/test_glossary_mutation.py:248`, and `tests/test_glossary_resolution.py:69-82`,
and those messages must not change. Add a classmethod for the batch case:

```python
@classmethod
def from_failures(
    cls,
    failures: Sequence[GlossaryLookupFailure],
    *,
    total: int,
) -> GlossaryLookupError:
```

Contract:

- Exactly one failure -> return an instance whose `str()` is byte-identical to today's
  (`unknown glossary term: widget (did you mean: Widget Box)`). A one-term batch must
  not regress into batch-shaped prose.
- More than one failure -> a multi-line message:

  ```
  2 of 5 glossary references did not resolve:
    Bogus — no close matches
    xpromt — did you mean: Xprompt, Xprompt Part, Xprompt Swarm
  ```

  The `N of M` count is what makes this intuitive under batching: it tells the agent
  immediately that the other three references were fine and only two need fixing.
  `_resolve_roots` knows `M`, so pass it in rather than recomputing it.

- The instance always exposes `failures: tuple[GlossaryLookupFailure, ...]`. Keep
  `.reference` and `.candidates` populated from the first failure so existing callers
  and `src/sase/ace/tui/modals/glossary_panel_actions.py:323` keep working unchanged.

`cli_show.py` and `cli_read.py` already print `f"sase glossary show: {exc}"` to stderr
and `sys.exit(1)`; a multi-line `str(exc)` flows through both unchanged, so no handler
edit is needed beyond confirming it reads well.

**Failure semantics: all-or-nothing.** If any reference fails, print nothing to stdout
and exit 1. Do not print the closure for the references that did resolve. An agent that
asked for five terms and silently received four would reason from an incomplete
glossary, and the exit code is easy to miss when the command also produced a large,
plausible-looking definition dump. Failing closed with a complete, actionable error is
the behaviour that makes batching safe to recommend. `read` additionally must not record
an audit event for a batch that never printed, which the current
`resolve_glossary_view`-before-`append` ordering in `cli_read.py` already guarantees.

### 3. Stop Paying For Invisible Bytes

`_build_node_block` in `src/sase/glossary/render.py` returns
`Padding(Group(*lines), (0, 0, 0, indent))`. Rich's `Padding` defaults to `expand=True`,
which right-pads every wrapped definition line out to the console width. Verified
directly against Rich:

```
expand=True  -> '  hello world                                               \n'
expand=False -> '  hello world\n'
```

All 2,021 padding characters in the batched sample come from these blocks; the
`Table.grid(expand=True)` header rows contribute zero, because their right-justified
`REQUESTED` / `RELATED · depth N` tag genuinely ends the line.

Pass `expand=False`:

```python
return Padding(Group(*lines), (0, 0, 0, indent), expand=False)
```

The left indent, the tree shape, and every visible glyph are unchanged — a terminal
renders trailing spaces as nothing. This is a pure deletion of bytes no reader ever saw.
Keep `expand=True` on the two `Table.grid` calls; they need the full width to
right-justify their tags.

### 4. Trim The Footer To What Is Actionable

`_build_footer` currently emits, unconditionally:

```
Related terms were printed because the definitions above mention them.
Use -d 0 to print only the requested terms.
```

plus a truncation line when applicable. Two problems:

- `Use -d 0 to print only the requested terms.` is printed **even when `-d 0` was
  passed**, and even when there are no related terms to suppress. In both cases it is
  advice to do the thing that was already done or that would change nothing.
- `Related terms were printed because the definitions above mention them.` restates, in
  prose, what every related node already says more precisely on its own
  `↳ mentioned as "X" in Y` line, next to the term it explains.

Replace the body with:

```python
lines: list[RenderableType] = []
if related > 0 and depth_limit != 0:
    lines.append(Text("Use -d 0 to print only the requested terms.", style="dim"))
if truncated:
    lines.append(
        Text(
            f"Truncated at depth {depth_limit}; raise -d or omit it to see more.",
            style="dim",
        )
    )
return Group(*lines)
```

The resulting footer is exactly as long as it needs to be:

| Invocation                      | Footer                    |
| ------------------------------- | ------------------------- |
| default depth, related terms    | one line: the `-d 0` hint |
| default depth, no related terms | nothing                   |
| `-d 0`, closure was cut off     | one line: the truncation  |
| `-d 0`, nothing was cut off     | nothing                   |

Keep the truncation line: it is the only signal that content was withheld, and dropping
it would let an agent mistake a capped closure for a complete one.

Check `tests/main/test_glossary_cli_show.py:55` and `:73` — both surviving assertions
still hold under the new rules (line 55 runs at default depth with related terms; line
73 runs `-d 0` against a closure that is genuinely truncated). No test asserts the
dropped `Related terms were printed…` sentence.

### 5. Tell The Truth When References Collapse

With batching encouraged, an agent will sooner or later pass both a term and one of its
aliases — the instruction file lists them together as
`Agent Hood (hood, agent neighborhood)` — or repeat a term. `_resolve_roots`
de-duplicates silently, so `sase glossary show Stitch stitch` prints
`1 term · 1 requested`. De-duplicating is the right behaviour, but a header that reports
fewer requested terms than were asked for, with no explanation, reads like a dropped
argument.

Add a `requested_references: int` field to `GlossaryClosure` (default `0`, so the two
construction sites in `resolution.py` are the only ones to update — nothing else in
`src/` or `tests/` constructs it), set to the number of references passed in before
de-duplication. In `_glossary_closure_renderable`, when
`closure.requested_references > len(closure.roots)`, emit one dim line under the header:

```
6 references resolved to 5 terms (duplicates and aliases collapsed)
```

Only when it actually happened, so the common case pays nothing. Add
`requested_references` to the `-f json` payload for symmetry with the existing
`depth_limit` / `truncated` fields. Leave `-f markdown` alone: it is the lean
paste-into-a-prompt format and should carry definitions, not diagnostics.

## Rejected Alternatives

- **Partial success on a bad batch** — print the terms that resolved, warn about the
  rest, exit 0. Rejected: it saves one round trip at the cost of an agent silently
  reasoning from an incomplete glossary. See "all-or-nothing" above.
- **Auto-selecting `-f markdown` when stdout is not a TTY.** Batched markdown is ~18%
  smaller again than batched rich. Rejected: format would depend on invisible context,
  so the same command would produce different bytes in a terminal and in a pipe, and
  every `-f`-sensitive test would need a TTY story. Fixing the padding captures most of
  the win with none of the surprise; agents that want the smallest output can still pass
  `-f markdown` explicitly.
- **A `--all` flag to read the whole glossary.** Rejected: it optimises the wrong thing.
  It makes it trivial to pull all 29 definitions when four were needed, which increases
  token use in exactly the situation this plan exists to improve.
- **Changing the default depth.** Rejected: out of scope and independently risky. The
  closure is the feature; batching already shrinks it for free.

## Implementation Steps

1. `src/sase/glossary/resolution.py` — add `GlossaryLookupFailure`; add
   `GlossaryLookupError.from_failures(failures, *, total)` and the `.failures` attribute
   while preserving the existing single-reference constructor and message; rewrite
   `_resolve_roots` to accumulate failures and raise once; add
   `requested_references: int = 0` to `GlossaryClosure` and populate it at both
   construction sites; export the new name in `__all__` (Symvision requires new public
   symbols to be exported and used — read `sase/memory/symvision.md` via
   `/sase_memory_read` if a lint failure appears).
2. `src/sase/glossary/render.py` — `Padding(..., expand=False)`; rewrite `_build_footer`
   per section 4; add the collapse note to `_glossary_closure_renderable`; add
   `requested_references` to `_glossary_closure_json_payload`.
3. `src/sase/amd/_memory.py` — replace `_GLOSSARY_TERMS_INTRO`.
4. `src/sase/main/parser_glossary.py` — batch-first `epilog` examples and updated
   `description` text for `read` and `show`; also update the group-level `epilog` near
   the top of `register_glossary_parser` so `sase glossary --help` shows a batch.
5. `docs/memory.md`, `docs/init.md`, `docs/configuration.md`, and the
   `sase glossary show` / `sase glossary read` rows in `docs/cli.md` — batching
   guidance, plus the `**GLOSSARY TERMS:**` correction in `docs/configuration.md`.
6. Run `sase memory init` and commit the regenerated `AGENTS.md`, `CLAUDE.md`,
   `GEMINI.md`, `OPENCODE.md`, `QWEN.md`, and memory README.
7. Add a `CHANGELOG.md` entry if the repo's changelog lint requires one for user-visible
   CLI changes (`just _lint-changelog` will say).

## Tests

Extend the existing suites; do not create new files where a matching one exists.

- `tests/test_glossary_resolution.py`
  - A batch with two unknown references raises once, and the message names both, in the
    order they were passed, with each one's candidates.
  - The `N of M` prefix reports the true totals for a mixed batch (some resolve, some do
    not).
  - A one-element failing batch produces the byte-identical legacy message — this is the
    regression guard for `mutation.py` and the ACE panel.
  - `.failures` is populated for both the single and multi constructors.
  - `requested_references` counts pre-de-duplication references; a term passed alongside
    its own alias yields `requested_references == 2` and `len(roots) == 1`.
- `tests/main/test_glossary_cli_show.py`
  - Multi-term rich output marks every requested term `REQUESTED` and prints a shared
    related term exactly once.
  - No trailing whitespace on any rendered line (assert
    `all(line == line.rstrip() for line in text.splitlines())`) — this is the guard that
    keeps the padding regression from silently returning.
  - Footer matrix: the `-d 0` hint is absent under `-d 0`, absent when there are no
    related terms, and present at default depth with related terms.
  - The collapse note appears when a term and its alias are both passed, and is absent
    for a clean batch.
  - `-f json` for a multi-term closure carries every requested root and the new
    `requested_references` field.
  - An unresolvable reference inside a batch writes the aggregate message to stderr,
    exits 1, and writes nothing to stdout.
- `tests/main/test_glossary_cli_read.py`
  - A multi-term read records one audit event whose `terms` holds every requested term
    and whose `related_terms` holds the closure remainder with no overlap.
  - A batch containing an unknown term records **no** audit event and prints nothing.
- `tests/main/test_init_memory_glossary.py:146` — update the intro assertion to the new
  string, keep asserting `Aliases follow in parentheses.`, and add an assertion that the
  rendered Tier 2 section instructs batching (e.g. contains `in one command`).

## Verification

```bash
just install    # ephemeral workspace: required before anything else
just check
sase memory init
```

`just check-full` before landing, run through `/sase_monitor` with a `--next` action —
never inline, per this project's two-speed verification rule.

Manual confirmation of the headline claim, run from a workspace with a glossary:

```bash
for t in Stitch Patch "Sase Agent" "Agent Hood" Xprompt; do sase glossary show "$t"; done | wc -c
sase glossary show Stitch Patch "Sase Agent" "Agent Hood" Xprompt | wc -c
```

The second number should be roughly `7,000` against the first's roughly `40,500`.

## Out Of Scope

- Argument parsing for multiple terms (already implemented).
- Shell completion (already emits the repeated-positional spec).
- The ACE glossary panel and preview card. `src/sase/glossary/render.py` is imported
  only by `src/sase/glossary/cli_show.py`, and
  `src/sase/ace/tui/modals/glossary_preview_render.py` calls `resolve_glossary_closure`
  with an already-resolved `GlossaryEntry` root, so it never reaches the lookup path
  this plan changes. No TUI or PNG snapshot should move.
- `sase glossary list`, `add`, `del`, and `log`.
