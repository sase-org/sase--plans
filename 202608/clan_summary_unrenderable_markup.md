---
tier: tale
title: Stop ACE crashing on clan summaries with unrenderable Rich markup tags
goal:
  Selecting any agent clan in `sase ace` renders its summary instead of raising
  `MissingStyle`, keeping intended markup styled and rendering bracketed prompt tokens
  such as `[@file:<file>]` literally.
size: small
proposed_by: bbugyi200.athena.y0
create_time: 2026-08-11 09:41:22
status: wip
---

# Fix `sase ace` crash on clan summaries containing unrenderable Rich markup tags

## Problem

Selecting the `research.07` agent clan in `sase ace` crashes the TUI with:

```
MissingStyle: Failed to get style '@file:<file>'; unable to parse '@file:<file>' as
color; '@file:<file>' is not a valid color
```

The same crash reproduces for any clan whose summary contains a bracketed token that
looks like a Rich markup tag but is not a resolvable style.

## Root cause (confirmed, reproduced end to end)

Three facts combine:

1. **Clan summaries are Rich markup that embeds raw prompt text.** The `research_swarm`
   xprompt declares its clan summary as
   `summary=[[[bold]RESEARCH PROMPT:[/bold] {{ prompt }}]]`, so the user's prompt is
   interpolated into a markup string **without escaping**. The `research.07` prompt is
   about artifact refs and contains literal tokens such as `[@file:<file>][<N>]` and
   `[@<ref>:<arg>][<N>]`.

2. **Rich's markup parser accepts these tokens as tags but never validates them.**
   `rich.markup.RE_TAGS` matches any `[...]` whose first character is in `[a-z#/@]`, so
   `[@file:<file>]` parses as an opening tag. Rich only treats `@`-prefixed tags as meta
   tags when they are _explicitly closed_; an unclosed `@` tag falls through to the
   end-of-markup drain in `rich.markup.render()`, which appends
   `Span(start, len(text), str(open_tag))` — a raw **style name string** that is never
   parsed or validated. So `Text.from_markup()` succeeds and returns a `Text` carrying
   spans with the style names `'@file:<file>'` and `'@<ref>:<arg>'`.

3. **Textual resolves those style names with no default, so resolution failure is
   fatal.** When the clan detail `Text` is handed to the panel widget via
   `self.update(text)`, Textual converts it with
   `textual.content.Content.from_rich_text`, which calls `app.console.get_style(style)`
   **without a `default=`** argument. Rich's `Console.get_style` re-raises a
   `StyleSyntaxError` as `rich.errors.MissingStyle` when no default is supplied. (Rich's
   own `Text.render` is safe here because it passes `default=Style.null()`, which is why
   the plain-Rich CLI paths never crash.)

The defect in this repo is in
`src/sase/ace/tui/widgets/prompt_panel/_agent_clan_summary_text.py`:

```python
def clan_summary_text(agent: Agent) -> Text:
    raw = agent.clan_summary or ""
    try:
        return Text.from_markup(raw)
    except MarkupError:
        return Text(raw)
```

The guard only covers `MarkupError`, which Rich raises for _structurally_ invalid markup
(for example a mismatched closing tag — the case covered by the existing
`test_clan_summary_invalid_markup_falls_back_to_raw_text`). It does not cover markup
that parses cleanly but yields **unresolvable style names**, because that failure
happens later, at render time, inside Textual — well outside this `try` block. The
result is an unhandled exception in the ACE render path rather than a
degraded-but-readable panel.

### Reproduction

The crash was reproduced through the real ACE code path (not a synthetic mock), using
the persisted `research.07` clan summary from its dismissed bundle under
`~/.sase/dismissed_bundles/`:

```python
container = project_clan_tree([member_with_that_clan_summary])[0]
text = build_clan_detail_text(container)   # succeeds
Content.from_rich_text(text, console=Console())
# rich.errors.MissingStyle: Failed to get style '@file:<file>'; ...
```

The error message matches the reported one byte for byte.

## Scope

`clan_summary_text` has exactly two callers, both in the clan panel:

- `_agent_display_clan.py:240` — appends the parsed summary into the clan detail `Text`
  that is later pushed to the widget (**the crash site**).
- `_agent_clan_aggregation.py:328` — reads only `.plain` for plan-reference hint
  scanning (never renders, so it cannot crash today).

No other `Text.from_markup` call site in `src/sase/` parses agent- or user-controlled
text: the remaining call sites parse sase-generated badge markup or literal constants.
The CLI/`rich.Console` render paths are unaffected for the reason given in fact 3.

This is presentation-only Textual/Rich glue, so per the repo's Rust-core boundary rule
it correctly stays in this repo; no `sase-core` change is required.

## Approach

Make `clan_summary_text` return a `Text` that is **guaranteed renderable by Textual**,
while preserving as much intent as possible. Use a three-layer fallback, best fidelity
first:

1. **Parse and validate.** `Text.from_markup(raw)`, then check every span whose style is
   a `str`: a style is renderable iff `rich.style.Style.parse(style)` does not raise.
   (`Style.parse` is `lru_cache`d by Rich, so this is cheap and safe on a render path.)
   If every span is renderable, return the `Text` unchanged — this is the common case
   and preserves today's behavior exactly.

2. **Escape the offending tags and re-parse.** If any span is unrenderable, walk the raw
   markup with `rich.markup.RE_TAGS`, compute for each match the style string Rich would
   produce for that tag, and backslash-escape only those tags whose style string fails
   `Style.parse`. Re-run `Text.from_markup` on the escaped string and re-validate.

   Computing the style string must mirror `rich.markup`'s own behavior, which is small
   and well defined: split the tag text at the first `=`, then `Tag.__str__` renders
   `name` when there are no parameters and `f"{name} {parameters}"` when there are.
   Deriving it this way (rather than guessing from the raw tag text) is required for
   correctness — `[link=https://example.org]` produces the _valid_ style
   `link https://example.org`, so a naive check on the raw tag text would wrongly escape
   legitimate link markup. Skip matches that are already backslash-escaped, and keep
   closing tags whose bare name is renderable plus the implicit `[/]` close.

   Decide escaping **per style string**, so an opening tag and its matching closing tag
   are always escaped together and the re-parse cannot orphan a closing tag.

3. **Plain fallback.** If step 1 or step 2 raises `MarkupError`, or step 2 still yields
   an unrenderable span, return `Text(raw)` — the existing behavior, extended to this
   new failure class.

### Why escape-and-retry rather than only falling back to plain text

Falling straight to `Text(raw)` would fix the crash in one line, but every
research-swarm clan summary embeds a full user prompt, and prompts routinely contain
bracketed tokens. That fallback would therefore surface a literal
`[bold]RESEARCH PROMPT:[/bold]` header on a large fraction of clans. Escape-and-retry
keeps the intended bold header **and** renders `[@file:<file>][<N>]` literally, which is
what the prompt actually said.

This was prototyped against the real `research.07` summary and verified: unrenderable
spans drop from `['@file:<file>', '@<ref>:<arg>']` to none, the only surviving span is
`bold`, `Content.from_rich_text` succeeds, the `RESEARCH PROMPT:` header is still
styled, and the literal `` `[@file:<file>][<N>]` `` text is preserved in the output.

## Implementation steps

1. In `src/sase/ace/tui/widgets/prompt_panel/_agent_clan_summary_text.py`, add two
   module -private helpers and rewrite `clan_summary_text` to use the three-layer
   fallback above:
   - a predicate that reports whether a span style string is renderable (`Style.parse`
     succeeds), memoized or relying on Rich's own `lru_cache`;
   - a function that returns the raw markup with unrenderable tags backslash-escaped.

   Keep `clan_summary_text`'s signature and `__all__` unchanged so both callers are
   untouched. Keep the module docstring accurate.

2. Confirm the aggregation caller still behaves correctly. `_agent_clan_aggregation.py`
   reads `clan_summary_text(agent).plain`; after this change that plain text now
   _includes_ previously-swallowed literal tag text, which can only help the
   `_LOGICAL_PLAN_REFERENCE_RE` hint scan. No change expected there — verify, do not
   pre-emptively edit.

## Tests

Add to `tests/ace/tui/widgets/test_agent_display_clan.py` (alongside the existing
`test_clan_summary_invalid_markup_falls_back_to_raw_text`, which must keep passing):

1. **Regression test for the reported crash.** Build a clan whose `clan_summary` is
   `"[bold]RESEARCH PROMPT:[/bold] replace `[@file:<file>][<N>]` with a link"`, run
   `build_clan_detail_text`, and assert `textual.content.Content.from_rich_text` on the
   result does not raise. This is the assertion that actually pins the bug — asserting
   only on `.plain` would have passed before the fix.

2. **Intent preservation.** For that same summary, assert the `RESEARCH PROMPT:` header
   still carries the `bold` style (reuse the existing `style_at` helper) and that the
   literal `@file:<file>` text appears in `.plain`.

3. **Legitimate markup is not over-escaped.** A summary using
   `[link=https://example.org]` keeps its `link https://example.org` span and still
   converts cleanly — this guards the `=`-parameter case called out in the approach.

4. **Structurally invalid markup still falls back to plain text.** Covered by the
   existing test; confirm it is unchanged.

5. **Empty / `None` clan summary** still yields an empty `Text` (no regression in the
   `agent.clan_summary or ""` path).

Where a test needs the Textual conversion, pass an explicit `rich.console.Console` to
`Content.from_rich_text` so no running `App` is required.

## Verification

- `just install` first (workspace virtualenvs are ephemeral and may be stale).
- `just check` — whole-repo lint gates plus the diff-scoped test lane.
- Run the clan panel tests directly:
  `pytest tests/ace/tui/widgets/test_agent_display_clan.py tests/ace/tui/widgets/test_agent_clan_aggregation.py`.
- Manual confirmation: launch `sase ace`, select the `research.07` clan, and confirm the
  panel renders with a bold `RESEARCH PROMPT:` header and literal `[@file:<file>][<N>]`
  text instead of crashing.

## Performance note

The change adds only cached `Style.parse` calls over a clan summary's spans (typically a
handful), and the escape-and-retry branch runs only when a summary is actually broken.
This sits in the debounced detail-panel build, not the highlight path, so it does not
affect j/k key-to-paint latency. No disk I/O, globbing, or stat calls are introduced.

## Out of scope (file as a follow-up task bead, do not implement here)

The durable upstream fix is to stop interpolating unescaped prompt text into a markup
string: the `research_swarm` xprompt's
`summary=[[[bold]RESEARCH PROMPT:[/bold] {{ prompt }}]]` should escape `{{ prompt }}`,
which would require a Rich-escaping Jinja filter in sase's xprompt environment
(`src/sase/xprompt/_jinja.py`, currently `autoescape=False`) plus a change in the
chezmoi linked repo that owns that xprompt.

Without it, a prompt containing a _valid_ style token such as `[dim]` still has that
token silently consumed and restyles the remainder of the summary. That is a cosmetic
text-loss bug, not a crash, and it is unchanged by this plan. The renderer fix above is
the correct backstop regardless: `sase ace` must never crash on persisted agent data.
