---
tier: tale
title: Fix test and doc fallout from the
goal:
  The sase-research-artifacts test suite passes against the two-input research_swarm
  contract, the new wait conditional has rendering coverage in both directions, and the
  input description and docs state the quoting rule that multi-agent waits require.
size: small
proposed_by: bbugyi200.athena.004
create_time: 2026-08-13 18:30:27
status: wip
---

# Fix Fallout From The `#research_swarm` `wait` Input

## Background

Commit `a7d9e04` ("feat: Add `wait` argument to #research_swarm") in the
**sase-research-artifacts** repo added a second, optional input to
`src/sase_research_artifacts/xprompts/research_swarm.md`:

```yaml
- name: wait
  type: word
  default: null
  description:
    Name of the sase agent (or agents if comma-separated) to wait for before starting
    the swarm. If null, the swarm will start immediately.
```

and made the two researcher segments emit `%wait:{{ wait }}` behind `{% if wait %}`.

The mechanism itself was verified end to end against the real dispatch path and **works
as intended**. Rendering `#research_swarm(wait=research.0f.final):: <topic>` gives the
`cdx` and `cld` segments `wait=['research.0f.final']` (alongside the existing
`%wait(priority=20)`), leaves `%clan` / `%id` / `%model` untouched, and lets the `final`
and `image` segments inherit the gate transitively. Omitting the argument renders no
extra directive at all, because `default: null` is a real default (sase distinguishes an
explicit `None` from its `UNSET` "required" sentinel). The frontmatter also produces
zero diagnostics from `sase.xprompt.frontmatter_schema.validate_frontmatter`.

This plan fixes three pieces of fallout the commit left behind. **No change to the
directive-emitting template logic is needed or wanted.**

### Fallout 1 — a packaged-contract test is now stale (CI break)

`tests/test_xprompt_loading.py` still pins the old single-input contract:

```python
def test_research_swarm_declares_typed_input() -> None:
    xp = _research_xprompts()["research_swarm"]
    assert [(arg.name, arg.type.value) for arg in xp.inputs] == [("prompt", "text")]
```

`research_swarm` now declares two inputs, so this assertion fails. The repo's CI
installs the package editable, so its `just check` lane fails today.

(The sibling `test_research_swarm_dependency_graph_preserved` and
`test_research_swarm_has_four_top_level_segments` still pass — the commit added no new
`---` separator and did not touch the `%clan` / `%id` / `%wait:research.{@1}.*` strings
those tests assert on.)

### Fallout 2 — the new conditional rendering has no coverage

Nothing asserts that `wait` actually produces `%wait:<name>` on the researcher segments
when supplied, or that it stays absent when omitted. That conditional is the whole point
of the commit and is exactly the kind of Jinja whitespace/`{% if %}` construct that
breaks silently.

### Fallout 3 — the comma-separated wording is a footgun, and the docs table drifted

`%wait` genuinely accepts comma-separated targets, and
`wait="research.0f.final,research.0e.final"` (quoted) works correctly — it renders
`%wait:research.0f.final,research.0e.final`, which sase splits into two dependencies.

But the description advertises "or agents if comma-separated" without saying the value
must be quoted, and the **unquoted** form fails silently and destructively. In
`#research_swarm(wait=research.0f.final,research.0e.final):: my topic`:

- the xprompt arg grammar splits the parenthesized argument list on commas, so
  `research.0e.final` is parsed as a _positional_ argument, not part of `wait`;
- positional 0 binds to `prompt`, so the research topic becomes the literal string
  `research.0e.final`;
- the real topic text (`my topic`, supplied via the `::` shorthand) binds to positional
  1 (`wait`), where the explicit named `wait=` value wins and it is **silently
  discarded**;
- only `research.0f.final` ends up gating the swarm.

The result is two researcher agents researching the string "research.0e.final", with no
error or warning anywhere. The description should therefore document the quoting
requirement.

Separately, `docs/xprompts.md` documents `#research_swarm`'s inputs in a table that
still lists only `prompt`.

## Scope

All changes are in the **sase-research-artifacts** repo. Nothing in the sase repo, the
sase-core repo, or any other repo changes.

Open the repo (and its build dependency) through the `/sase_repo` skill before reading
or writing anything, and use only the paths it prints:

```bash
sase repo open sase-research-artifacts -r "Fix test/doc fallout from the #research_swarm wait input"
sase repo open sase-core -r "Justfile build dependency for sase-research-artifacts checks"
```

All file paths below are relative to the sase-research-artifacts checkout root.

## Steps

### 1. Fix the stale input-contract test

In `tests/test_xprompt_loading.py`, update `test_research_swarm_declares_typed_input` to
assert the current two-input contract, including that `wait` is optional with a `None`
default. Assert the default explicitly — that `None` (not the `UNSET` sentinel) is what
keeps `wait` optional is the single most fragile property of this frontmatter, and it is
what makes `{% if wait %}` render nothing when the argument is omitted. Import `UNSET`
from `sase.xprompt.models` if a direct comparison reads better than an `is not UNSET`
check.

Keep the existing style of the file: load through `_research_xprompts()` (which uses
`load_xprompts_from_plugins()`), so the test exercises the packaged definition rather
than whatever the ambient user/project xprompt catalog resolves.

Leave `test_research_prompt_declares_typed_input` (line 27) alone — that one covers
`research/prompt`, which is unchanged and still has exactly one input.

### 2. Add rendering coverage for the `wait` conditional

Add tests to `tests/test_xprompt_loading.py` that render the packaged swarm body with
and without the argument and assert on the resulting segments.

Use `sase.xprompt.processor.expand_single_xprompt` for the substitution — it is the
public entry point `render_xprompt_swarm` itself calls, it takes the plugin-loaded
`XPrompt` object directly (so the test stays isolated from the ambient catalog), and it
avoids importing sase's private `sase.agent._xprompt_swarm_rendering` module. Split the
result with the already-imported `split_segments_protecting_fences`:

```python
from sase.xprompt.processor import expand_single_xprompt

def _swarm_segments(named_args: dict[str, str]) -> list[str]:
    xp = _research_xprompts()["research_swarm"]
    body = expand_single_xprompt(
        xp, ["some topic"], named_args, preserve_segment_separators=True
    )
    return split_segments_protecting_fences(body)
```

Cover both directions:

- **With `wait`**: `_swarm_segments({"wait": "research.0f.final"})` puts
  `%wait:research.0f.final` in the `cdx` segment and in the `cld` segment, and does
  **not** add it to `final` or `image` (those two keep only their existing intra-clan
  `%wait:research.{@1}.*` dependencies). Also assert the topic text still survives in
  the researcher segments, and that `%wait(priority=20)`, `%model:@research_a` /
  `%m:@research_b` and the `%clan(research.{@1}` / `%id:research.{@1}.cdx` /
  `%id(cld, clan=research.{@1})` identity directives are all still intact — the
  `{% if %}` block sits directly between those directives, so a whitespace mistake there
  would corrupt them.
- **Without `wait`**: `_swarm_segments({})` yields researcher segments containing no
  `%wait:` agent dependency at all (only `%wait(priority=20)`), and no leftover Jinja
  (`{%` / `{{`) anywhere in any segment.

Segment count stays 4 in both cases; the existing
`test_research_swarm_has_four_top_level_segments` covers the no-argument case, so
asserting the 4-way unpack in the helper's callers is enough.

### 3. Correct the `wait` input description

In `src/sase_research_artifacts/xprompts/research_swarm.md`, rewrite the `wait` input's
`description` so it states the quoting requirement for multiple agents. Something like:

> Name of the sase agent to wait for before starting the swarm. Quote the value to pass
> several comma-separated agents (`wait="a,b"`); an unquoted comma is parsed as a
> separate xprompt argument. If null, the swarm starts immediately.

Requirements for the wording:

- It must name the quoted form explicitly, because the unquoted form fails silently and
  eats the research topic.
- Keep it accurate about the null default.
- Preserve the file's existing YAML block-scalar/indentation style and its 88-column
  wrapping.

Do **not** change the `type`, the `default`, or anything in the body template.

### 4. Update the docs

In `docs/xprompts.md`, add a `wait` row to the `#research_swarm` "Input" table (around
line 33), matching the corrected description, and note that it is optional. Then extend
the four-segment walkthrough below the table so the `<clan>.cdx` and `<clan>.cld`
entries mention that both additionally wait on the `wait` argument's agent(s) when one
is supplied — that is what makes the whole swarm start only after a previous swarm's
lead has finished.

Check `README.md` too: its `#research_swarm` bullet (around line 87) is a one-line
summary that does not enumerate inputs, so it likely needs no change — but confirm
rather than assume.

Keep both files' existing markdown table formatting and 88-column wrapping.

### 5. Verify

Run the repo's own gate from the sase-research-artifacts checkout root:

```bash
just check
```

That runs `ruff check`, `mypy`, and `pytest`. It needs both the sase and sase-core
checkouts, which is why step 0 opens `sase-core` as well — the Justfile finds it as a
sibling of the plugin checkout. The first run builds sase-core with maturin and can take
several minutes, so run it through the `/sase_monitor` skill rather than inline, with a
`--next` action that reacts to the result.

Confirm the previously-failing `test_research_swarm_declares_typed_input` now passes
along with the new rendering tests.

## Explicitly Out Of Scope

- **Reinstalling the plugin.** The `wait` argument has no effect on the owner's machine
  until the installed plugin is upgraded: it is a git install currently pinned at
  `807e209`, the commit _before_ `a7d9e04`. Against that older definition `wait=` is an
  unknown named argument, which sase preserves for runtime context instead of rejecting
  — so `#research_swarm(wait=...)` today launches an ungated swarm with no error. The
  fix is `sase plugin update research-artifacts` (its dry run confirms it reinstalls
  from `git+https://github.com/sase-org/sase-research-artifacts`, so the already-pushed
  master commit is sufficient and no PyPI release is required). That is the project
  owner's call to make on their own environment; **do not run it as part of this work.**
- **Changing the template's directive placement or whitespace.** The rendered researcher
  prompts carry a couple of stray spaces around the substituted `{% if %}` block. This
  is cosmetic, partly pre-existing, and does not affect directive parsing (verified) or
  the `#research` reference that follows. Leave it alone.
- **Making `wait` a `repeatable` input.** It would not fix the unquoted-comma case (an
  explicit named value still wins over positionals), and it would force the template to
  join a list, which is a larger change for no gain over documenting the quoted form.
- **Bumping the version in `pyproject.toml` or editing `CHANGELOG.md`.** Release-please
  owns both.
