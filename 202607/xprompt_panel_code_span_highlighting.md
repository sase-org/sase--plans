---
tier: tale
title: Inline code highlighting in the agent XPROMPT panel
goal: 'Inline code spans in the AGENT XPROMPT section of the agent metadata panel
  render with the same monokai code styling the AGENT PROMPT section already gives
  them, instead of collapsing to plain text whenever the line begins with an xprompt
  invocation, and every other markdown surface keeps rendering exactly as it does
  today.

  '
create_time: 2026-07-16 17:13:02
status: wip
prompt: 202607/prompts/xprompt_panel_code_span_highlighting.md
---

# Plan: Inline code highlighting in the agent XPROMPT panel

## Problem

In the AGENT XPROMPT section of the agent metadata panel, inline code spans render as plain white text with literal
backticks. The AGENT PROMPT section directly beneath it renders the same spans in monokai's khaki (`#e6db74`). The two
sections show near-identical prose, so the inconsistency is obvious on screen.

Both sections already run monokai markdown highlighting, so the highlighter is not missing — it is being defeated by the
content.

### Root cause

Pygments' `MarkdownLexer` (2.19.2) matches ATX headings with:

```python
(r'(^#[^#].+)(\n)', bygroups(Generic.Heading, Text)),
```

Any line starting with `#` followed by a non-`#` character is consumed whole as a single `Generic.Heading` token. Every
xprompt begins with a VCS ref (`#gh:...`, `#git:...`), and further invocation-led lines are common, so those lines are
swallowed entirely and **no inline rule ever runs inside them** — code spans, emphasis, and links all stay plain.

Confirmed directly against the shared highlighter:

| Source line                                    | Lexed as                                        |
| ---------------------------------------------- | ----------------------------------------------- |
| ``Can you merge the `SASE PLAN` section?``     | code span → `#e6db74`                           |
| ``#gh:sase Can you merge the `SASE PLAN` ...`` | one `Generic.Heading` token for the entire line |

Two facts make this purely a defect with no upside:

1. In monokai, `Generic.Heading` resolves to `#f8f8f2` — **identical to plain text**. Treating an invocation line as a
   heading buys zero visual benefit while suppressing all inline highlighting on it.
2. The rule is not CommonMark: real ATX headings require whitespace after the `#` run. It also contradicts SASE's own
   xprompt grammar, which ignores `# Heading` as an invocation. The two grammars are exactly complementary — `#` +
   non-space is an invocation, `#` + space is a heading — and pygments is the only layer that conflates them.

### What is _not_ broken (verified, and scopes this plan down)

- **Multi-line / fenced code blocks already highlight correctly**, including inner language highlighting, because a
  fence opens on its own line and never collides with the heading rule. No fix needed; the plan only locks this in with
  a test.
- **`xprompt_inspect.tokenize` already honours literal zones.** Inside a fenced block, `#tale` and `#!/bin/bash` are not
  tokenized, and neither is an inline `` `#beau` ``. The semantic overlay layer is correct as-is.

So the defect is confined to inline constructs on invocation-led lines, in the one path that applies a markdown base:
`highlight_prompt_text` in `src/sase/ace/tui/util/xprompt_syntax.py`.

## Design

Teach the markdown base lexer the one rule it is missing: a line-leading `#` that starts an xprompt invocation is not a
heading.

Add a private `MarkdownLexer` subclass in `src/sase/ace/tui/util/xprompt_syntax.py` that **prepends a single rule** to
the `root` state:

```python
(r'^#(?=[^#\s])', Token.Text)
```

It consumes only the line-leading invocation marker as plain text. Because pygments anchors `^` to line starts, the
heading rules can no longer claim that line, and the remainder is lexed as an ordinary paragraph — so code spans,
emphasis, and links all highlight normally. Expose it as a module-level singleton and pass it to `Syntax(...)` in
`highlight_prompt_text` in place of the `"markdown"` string.

Why this shape:

- **Purely additive.** It does not remove, rewrite, or pattern-match pygments' existing rules, so it cannot silently
  break on a pygments upgrade. The alternative — filtering the heading rules out by matching their regex text — is
  brittle and would regress silently the moment upstream edits a pattern.
- **The `[^#\s]` lookahead is load-bearing.** `[^\s]` alone would eat the first `#` of `## Sub` and demote every h2–h6
  heading to plain text. Excluding `#` preserves `# Heading` (h1) and `## Sub` (h2–h6) byte-for-byte.
- **Keeps the markdown base.** A hand-rolled inline-code regex overlay would duplicate markdown parsing and lose
  fenced-block inner highlighting, which already works.

Keep `highlight_prompt_text`'s existing fail-open `try/except` and the `Syntax` import intact — an existing test patches
`Syntax.highlight`, and oversized input must still degrade to plain `Text`.

Callers change in exactly one place. The stashed-prompt, pinned-stash, and stash-preview modals also render xprompt text
through `highlight_prompt_text`, so they inherit the fix for free.

## Scope and non-goals

- **The file-hint path (`_agent_display_hints.py`) stays plain text.** Hint mode deliberately renders the whole panel —
  AGENT PROMPT included — as plain text so `[N]` markers can be inserted inline; only semantic xprompt token overlays
  are layered on. Adding a markdown base there would be internally inconsistent and would put marker offsets at risk.
  Code spans staying plain in hint mode is consistent with every other section in that mode.
- **`lazy_syntax`'s markdown path (AGENT PROMPT, replies) is unchanged.** It renders general markdown that rarely opens
  a line with a space-less `#`, and touching it would shift unrelated goldens for no reported symptom.
- No new theme work, no changes to the editable prompt input, and no Rust core changes — highlighting is
  presentation-only by design, and the canonical grammar already lives in the core mirrors `xprompt_inspect` consumes.

## Performance

A lexer swap on an existing call; no new work, no new render path. Measured at **0.24 ms** for a typical one-line
xprompt (the stock lexer's 0.058 ms only "wins" by collapsing the line into one token — i.e. by doing the work we want
skipped). This sits behind the 150 ms `DetailPanelDebouncer` and the existing 24-entry highlight cache, and the j/k
highlight fast path (`update_header_only`) is untouched. No stat, glob, or disk access is added.

## Testing

`tests/ace/tui/util/test_xprompt_syntax.py`:

- **Regression test for the report:** an invocation-led line containing `` `foo` `` carries the monokai code-span style.
- `# Heading` still lexes as a heading and `## Sub` still lexes as a subheading — the guard rule does not over-reach.
- A `#!workflow`-led line also highlights its inline code.
- A fenced block following an invocation line still highlights its inner code (locks in today's behavior).
- All existing tests in this module must stay green unmodified.

`tests/ace/tui/widgets/test_agent_display_xprompt.py`: the standard render path styles a code span on the humanized
xprompt, and the section's plain text is unchanged.

PNG visual: extend the existing `agents_xprompt_panel_highlighting` fixture's raw xprompt with an inline code span on
its invocation-led first line and regenerate that single golden with `--sase-update-visual-snapshots`, then confirm via
`just test-visual`. The fixture currently contains no backticks, so this is the snapshot that proves the fix.

Finish with `just install && just check`.

## Risks

- **Style precedence where a code span and an xprompt token overlap** (e.g. `` #quoted:`two words` ``): the semantic
  overlays are applied after the markdown base and win, which is the desired ordering. Existing tests already cover this
  form and pass.
- **Pygments upgrade.** The guard is additive and cannot crash; worst case is loss of effect, which the behavioral tests
  above turn into a loud CI failure rather than a silent regression.
- **Prototype validation.** The full design was exercised in-memory against the real test suites before this plan was
  written: 23/23 existing tests pass unmodified, with heading, subheading, fence, and `#!` cases all verified.
