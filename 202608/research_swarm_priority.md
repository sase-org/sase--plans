---
tier: tale
title: Add optional priority input to
goal:
  hard-coded `%wait(priority=20)` so callers can set the swarm's runner-queue priority
  without changing omitted-arg behavior.
size: small
proposed_by: bbugyi200.athena.08b
create_time: 2026-08-19 18:00:07
status: wip
---

# Plan: Add Optional `priority` Input To `#research_swarm`

## Context

`#research_swarm` is a four-segment xprompt swarm shipped by the
**sase-research-artifacts** plugin (linked repo of the same name). It is not a sase
package default. The packaged definition is
`src/sase_research_artifacts/xprompts/research_swarm.md`.

The swarm already has two inputs:

- `prompt` (`text`, required) — the research topic
- `wait` (`word`, optional, `default: null`) — extra agent dependency on the two
  researcher segments only

Every segment also hard-codes a runner-queue wait:

```text
%wait(priority=20)
```

Those four occurrences live on:

1. `<clan>.cdx` — `%wait(priority=20) %model:@research_a …`
2. `<clan>.cld` — `%wait(priority=20) %m:@research_b …`
3. `<clan>.final` — `%wait(priority=20) %m:@research_lead`
4. `<clan>.image` — `%wait(priority=20) %model:@image`

`20` is intentional, not leftover. SASE's implicit `%wait` priority is `10`; lower
numbers start first. `20` deprioritizes the whole research clan relative to ordinary
work (and takes the 30s deference window documented for `priority=20`). Omitting the new
argument must keep emitting `priority=20` on every segment. Do **not** drop the
directive when the default is used — that would silently promote the swarm to implicit
`10`.

`%wait(priority=N)` is parsed after xprompt expansion. Jinja
`%wait(priority={{ priority }})` is the right substitution.
`sase.xprompt.input_binding.bind_input_args` fills declared defaults into the Jinja
named context, so an omitted `priority` renders as `20`.

The optional `wait` input is a **second**, separate directive
(`{% if wait %} %wait:{{ wait }} {% endif %}`) on the researcher segments only.
`resolve_wait_priority_args` rejects two `%wait(priority=...)` values in one prompt.
Keep that extra agent wait **without** a `priority=` kwarg. Do not merge the two
directives into `%wait({{ wait }}, priority={{ priority }})`.

## Scope

All file changes belong in the **sase-research-artifacts** linked repo. Open it (and its
Justfile build dependency) through `/sase_repo` before reading or writing anything, and
use only the paths those commands print:

```bash
sase repo open sase-research-artifacts -r "Add optional priority input to #research_swarm"
sase repo open sase-core -r "Justfile build dependency for sase-research-artifacts checks"
```

All paths below are relative to the sase-research-artifacts checkout root.

Do not edit the sase primary repo, sase-core, or any SASE memory file. Host-repo tests
that mention `research_swarm` or `%wait(priority=20)` use mock catalogs, not this
plugin's packaged body.

This is not a feature flag. `priority` is an optional launch-time input the caller
chooses per invocation, with a default that preserves today's behavior.

## Implementation

### 1. Declare the input

In `src/sase_research_artifacts/xprompts/research_swarm.md`, add a third frontmatter
input after `wait`:

```yaml
- name: priority
  type: int
  default: 20
  description:
    Runner-queue priority passed to every segment's `%wait(priority=...)`. Lower numbers
    start first. Defaults to 20, the swarm's current hard-coded value (deprioritized
    versus SASE's implicit 10). Must be a non-negative integer; `%wait` rejects anything
    else.
```

Requirements:

- `type` must be `int` (alias `integer` is accepted, but the file uses canonical type
  names). Invalid values such as `priority=abc` then fail at expansion instead of
  producing `%wait(priority=abc)`.
- `default` must be the YAML integer `20`, not the string `"20"` and not `null`.
  `default: null` would make `{% if priority %}`-style guards tempting and would change
  omitted-arg rendering. `UNSET` would make the input required.
- Keep the file's existing YAML block-scalar / indentation style and 88-column wrap.
- Do not add min/max fields (xprompt `int` inputs have none). The description is enough;
  `%wait` already requires a non-negative integer at directive parse.
- Leave `prompt` and `wait` unchanged.

Confirm the updated frontmatter produces no diagnostics from
`sase.xprompt.frontmatter_schema.validate_frontmatter`. An `int` input with integer
default `20` is an already-supported pattern.

### 2. Substitute on every hard-coded wait

Replace all four `%wait(priority=20)` tokens with `%wait(priority={{ priority }})`. No
space after `=`. Do not wrap the substitution in `{% if %}` and do not use
`{{ priority | default(20) }}` — the frontmatter default is the source of truth.

Do not touch:

- `{% if wait %} %wait:{{ wait }} {% endif %}`
- `%clan` / `%id` / `%model` / `%m` / `#fork` / `#research` / `#research/image`
- `{% raw %}{{ wait_chats }}{% endraw %}`
- `{@1}` dispatch keys
- segment count (still four `---` separators)

After the edit the researcher lines still look like:

```text
%wait(priority={{ priority }}) %model:@research_a {% if wait %} %wait:{{ wait }} {% endif %}
```

and the lead / image lines still have a single `%wait(priority={{ priority }})` plus
their existing intra-clan waits.

### 3. Tests

Update `tests/test_xprompt_loading.py`. Keep loading through `_research_xprompts()`
(`load_xprompts_from_plugins()`) and rendering through `_swarm_segments()` /
`expand_single_xprompt(..., preserve_segment_separators=True)` so tests exercise the
packaged definition, not the ambient catalog.

- **Contract.** Extend `test_research_swarm_declares_typed_input` to the three-input
  list `prompt`/`text`, `wait`/`word`, `priority`/`int`. Assert defaults explicitly:
  `prompt` is `UNSET`, `wait` is `None`, `priority` is the integer `20` (not `"20"`).
  Import `UNSET` is already there. Leave `test_research_prompt_declares_typed_input`
  alone.
- **Default rendering.** Existing wait tests already assert `%wait(priority=20)` on
  `cdx` and `cld` when `priority` is omitted. Keep those, and pin the same default on
  `final` and `image` as well — all four segments share the input. Also assert no
  leftover `{{ priority }}` (and still no leftover `{{ wait }}` / `{%`).
- **Override.** Add a rendering test that calls `_swarm_segments({"priority": "5"})` and
  asserts `%wait(priority=5)` on **every** segment, and that **no** segment still
  contains `%wait(priority=20)`. Identity directives (`%clan`, `%id`, models, intra-clan
  waits, `#fork`, `#research/image`) must survive.
- **Override plus `wait`.** One combined case,
  `_swarm_segments({"wait": "research.0f.final", "priority": "5"})`, must put
  `%wait(priority=5)` and `%wait:research.0f.final` on `cdx`/`cld`, put only
  `%wait(priority=5)` (not the extra agent wait) on `final`/`image`, and must not emit a
  second `priority=` on the researcher `%wait:…` directive.
- **Invalid value (optional but cheap).** Expanding with `priority=abc` should raise
  `sase.xprompt.processor.XPromptArgumentError` (or the binding error it wraps)
  mentioning `int`. Skip this if it forces a private import; the type contract plus
  override test are the required coverage.

Do not add tests in the sase primary repo.

### 4. Docs

- `docs/xprompts.md` — add a `priority` row to the `#research_swarm` Input table (`int`,
  optional, default 20, lower numbers start first, applies to every segment's
  `%wait(priority=...)`). The four-segment walkthrough should mention that all four
  members honor `priority` (unlike `wait`, which gates only `cdx`/`cld`).
- `README.md` — the `#research_swarm` bullet already names optional `wait`. Extend that
  sentence to mention optional `priority` (default 20). Do not turn the bullet into an
  input table.

Keep existing markdown table formatting and 88-column wrapping. Do not hand-edit
`CHANGELOG.md` or `pyproject.toml`; release-please owns both.

## Verification

From the opened sase-research-artifacts checkout:

```bash
just install
just check
```

`just check` is the plugin's lint + default pytest lane. The first run may build
sase-core with maturin and take several minutes — use `/sase_monitor` with
`--start-status TESTING --stop-status TESTED` and a `--next` action that reacts to the
result. `just test-wheel` is not required.

Confirm:

- omitted `priority` still expands to `%wait(priority=20)` on all four segments
- `#research_swarm(priority=5):: topic` expands to `%wait(priority=5)` on all four
- `wait` plus `priority` still compose as two directives on the researchers
- `test_research_swarm_declares_typed_input` and the new rendering tests pass

Do not run `just check` in the sase primary repo unless that tree was actually edited
(it should not be).

## Out Of Scope

- **Reinstalling the plugin** on the owner's machine
  (`sase plugin update research-artifacts`). Until the installed distribution picks up
  this commit, `priority=` is an unknown named argument and is preserved as runtime
  context instead of substituting. That upgrade is the project owner's call.
- **Changing the default from 20 to 10**, or omitting `%wait(priority=...)` when the
  default is used.
- **Per-segment priorities** or applying the input only to the two researchers.
- **Merging** the optional `wait` agent dependency into the priority `%wait`.
- **Extra numeric validation** beyond `type: int` (negatives fail later at `%wait`
  parse; document non-negative in the description).
- **Feature flags, sase primary-repo edits, sase-core edits, memory-file edits.**
- **Bumping the plugin version or editing `CHANGELOG.md`.**

## Acceptance

- `#research_swarm` declares optional `priority: int = 20`.
- Every swarm segment renders `%wait(priority={{ priority }})`; omitted invocations
  still produce `%wait(priority=20)`.
- `#research_swarm(priority=N)` sets that `N` on all four segments.
- The optional `wait` input is unchanged and still does not carry `priority=`.
- Plugin tests and docs describe the new input; no sase-primary, sase-core, memory, or
  changelog edits.
