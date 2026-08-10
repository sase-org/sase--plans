---
tier: tale
size: small
title: Show the shipped @smarter reference on the Models panel @default row
goal:
  The ACE Models panel renders the @default row's provenance tag as "implicit ->
  @smarter" instead of a bare "implicit", matching every other implicit alias row and
  the already-correct %model completion, docs, and picker surfaces.
proposed_by: bbugyi200.athena.wv.f4.f0
create_time: 2026-08-10 13:15:01
status: wip
---

# Fix the `@default` row's missing `-> @smarter` provenance tag

## Symptom

In the ACE Models panel (leader `,m`), the `default` row renders:

```
default   default   CODEX(gpt-5.6-sol) @ xhigh   implicit
```

while its sibling rows render their delegation edge:

```
role      epic_lander       CODEX(gpt-5.6-sol) @ xhigh   implicit → @default
role      big_epic_lander   CLAUDE(opus) @ max           implicit → @smartest
```

The provider/model badge is already correct — `@default` really does resolve through
`@smarter` — so this is a display-only defect, not a routing defect.

## Root cause

One stale guard in the Models-panel state-tag renderer:

`src/sase/ace/tui/modals/models_panel_rendering.py:166-174`

```python
    reference = ""
    if view.configured and view.references is not None:
        reference = view.references
    elif (
        not view.configured
        and view.name != DEFAULT_MODEL_ALIAS_NAME   # <-- suppresses the edge
        and view.implicit_fallback is not None
    ):
        reference = view.implicit_fallback
```

`AliasView.implicit_fallback` (`src/sase/llm_provider/alias_view.py:194-199`) already
returns the right answer. Verified live in this workspace:

```
implicit_model_alias_fallback("default")  -> 'smarter'
state_tag(default-row)                    -> 'implicit'          # bug
state_tag(epic_lander-row)                -> 'implicit → @default'
```

### Why the guard exists and why it is now wrong

It predates the delegated `default`. `git log -S` traces it to `d57e2207c` ("feat(ace):
show model pool effort details"), which rewrote an even older
`if view.name == DEFAULT_MODEL_ALIAS_NAME: return Text("implicit")` early return into
the current `elif` clause. At the time, `_parse_model_alias_defaults` structurally
forbade `default` from declaring a `fallback`, so `implicit_fallback` was
_unconditionally_ `None` for that row and the guard was pure defensive redundancy.

Commit `012e1a88b` ("feat: add smarter model alias routing") relaxed the parser
(`src/sase/llm_provider/model_alias_policy.py:254-259`) so `default` may declare a
`fallback`, and pointed the shipped `default` entry at `@smarter`. The guard was not
revisited, so it flipped from harmless to actively wrong: it is now the only thing
suppressing a real edge.

### This is the only affected surface

Every other consumer of the same data reads `view.references or view.implicit_fallback`
with no `default` special case, and all were verified correct against the shipped graph:

| Surface                      | Site                                                                           | Current output for `@default`               |
| ---------------------------- | ------------------------------------------------------------------------------ | ------------------------------------------- |
| `%model` completion catalog  | `src/sase/xprompt/model_completion.py:395`                                     | `reference='smarter'` (correct)             |
| Prompt-bar completion row    | `src/sase/ace/tui/widgets/_prompt_input_bar_completion_rows_directives.py:154` | renders the reference (correct)             |
| Model picker cycle detection | `src/sase/ace/tui/modals/model_picker_rows.py:104`                             | `default -> smarter` edge present (correct) |
| Generated docs table         | `docs/llms.md:985`                                                             | `` `@smarter` `` (correct)                  |

`tools/render_model_alias_docs:50` still carries an
`elif alias == DEFAULT_MODEL_ALIAS_NAME` branch, but it sits _after_ the
`alias in fallbacks` branch, so it is correctly dormant while `default` delegates and
correctly re-activates if a future `default` entry drops its fallback. **Leave it
alone.**

`src/sase/llm_provider/model_alias_resolution.py:169` and `:256` are also legitimate
`default` special cases (known-alias classification and the provider-tier-default last
resort). **Leave both alone.**

## Implementation

### 1. `src/sase/ace/tui/modals/models_panel_rendering.py`

Delete the `and view.name != DEFAULT_MODEL_ALIAS_NAME` clause from `state_tag`, leaving:

```python
    elif not view.configured and view.implicit_fallback is not None:
        reference = view.implicit_fallback
```

Then delete the now-unused
`from sase.llm_provider.config import DEFAULT_MODEL_ALIAS_NAME` import at line 20 — that
line-20 import and line 171 are the only two occurrences of the name in the file, so
leaving it trips ruff `F401`.

No new branch is needed for the "undelegated `default`" case: if a future shipped YAML
drops `default`'s `fallback`, `implicit_fallback` returns `None` and the row falls
through to a bare `implicit` exactly as before. A user-configured `default` pinned to a
concrete model likewise still renders a bare `configured`, because `view.references` is
`None` for a non-`@alias` configured value.

## Tests

### `tests/test_models_panel_alias_rendering.py:90-92`

`test_state_tag_implicit_default` currently pins the bug:

```python
def test_state_tag_implicit_default() -> None:
    text = _state_tag(make_alias_view("default", "default"), now=0.0)
    assert text.plain == "implicit"
```

Retarget it to the shipped edge — `assert text.plain == "implicit → @smarter"` — and
rename/redocument it so it reads as a regression guard for the delegated `default` (e.g.
`test_state_tag_implicit_default_shows_shipped_delegation`). Note the assertion uses the
real Unicode arrow `→` and a real-looking space layout; copy the exact string shape from
the neighbouring `test_state_tag_implicit_big_epic_lander` rather than hand-typing it.

`make_alias_view` builds a bare `AliasView`, and `AliasView.implicit_fallback` reads
live policy, which under `tests/conftest.py`'s autouse fixture is the frozen graph in
`tests/_model_alias_defaults_fixture.py`. That fixture already declares
`DEFAULT_MODEL_ALIAS_NAME: {"fallback": "@smarter"}`, so no fixture change is needed.

### Add the inverse guard

Add a sibling test asserting that an _undelegated_ `default` still renders a bare
`implicit`, so a future YAML change that drops the fallback is covered and the deleted
guard is not silently reintroduced. Monkeypatch the role-fallback map (or construct the
view via a patched `implicit_model_alias_fallback`) rather than mutating the frozen
fixture, which is a shared shipped-graph contract.

### Re-derive the rest rather than trusting this list

`tests/test_models_panel_alias_rendering.py:92` is the only exact-match `"implicit"`
assertion for a `default` row found by grep, and
`tests/test_models_panel_alias_rendering.py:211` is a substring `in` check that stays
true. Still re-grep `tests/` for `state_tag` / `"implicit"` / `render_alias_row` after
the change instead of assuming, and let the suite report the truth.

## Visual snapshots

The `default` row's state tag grows by ` → @smarter`, so the ACE Models-panel PNG
goldens shift.

`tests/ace/tui/visual/_ace_models_panel_png_snapshot_fixtures.py:90` `calm_views()`
contains an unconfigured, un-overridden `default` row, and `bucket_views()`,
`ownership_views()`, `builtin_only_views()`, `custom_builtin_warning_views()`, and the
picker/effort variants all derive from or mirror it — so most `models_panel_*.png`
goldens in `tests/ace/tui/snapshots/png/` are expected to change. `override_views()`
puts an active override on `default`, which short-circuits `state_tag` before the
reference branch, so its golden should be unchanged; treat a diff there as a signal to
stop and re-read rather than a snapshot to accept.

Procedure:

1. Run `just test-visual` (or scope it to the Models-panel visual files via
   `just test-visual -- <paths>`, which the Justfile passes through).
2. **Inspect the actual/expected/diff artifacts under `.pytest_cache/sase-visual/`
   before accepting anything.** The only intended pixel delta is the added ` → @smarter`
   suffix on the `default` row.
3. Accept with `--sase-update-visual-snapshots` once the diffs are confirmed to be
   exactly that.

Two hazards worth checking explicitly while reviewing the actuals:

- **Row-index drift.** Several Models-panel visual cases navigate with hard-coded
  `j`-press counts. This change adds no rows, so counts should be stable — but confirm
  the selected alias in each refreshed actual matches the alias its case name implies.
- **Column truncation.** The state tag is the rightmost column and `render_alias_row`
  builds a `no_wrap=True, overflow="ellipsis"` line. Check the narrow
  `models_panel_alias_picker_reordered_70x32.png` case in particular: at 70 columns the
  longer `implicit → @smarter` tag could ellipsize. If it does, that is a real finding —
  report it rather than accepting a truncated golden, since the `default` row is the
  panel's most important row.

The `%model` completion goldens (`prompt_model_completion_aliases_120x40.png`,
`prompt_model_completion_mixed_120x40.png`) are built from synthetic completion rows on
a code path that already emitted the reference, so they should not move.

## Out of scope

- No change to alias resolution, the shipped YAML, the parser, the docs renderer, or
  `docs/*.md` — all four are already correct.
- Do not hand-edit `CHANGELOG.md` (release-please owns it).
- Do not touch `sase/memory/*.md`, `AGENTS.md`, or the generated provider instruction
  shims (`CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`) — no memory edit is
  authorized for this work.

## Verification

1. `just install` first (workspaces are ephemeral; dependencies may be stale).
2. `just check` is the right gate here: the change is one clause in a leaf TUI rendering
   module and does not touch the config schema, generated docs, or shared alias policy.
   Escalate to `just check-full` only if the scoped selection reports something unusual.
3. `just test-visual` for the PNG lane, per the procedure above.
4. Manual confirmation in the real panel — open `sase ace`, press leader `,m`, and read
   the `default` row. It should show `implicit → @smarter`, styled identically to
   `epic_lander`'s `implicit → @default`.

### Known-unrelated failures

Two active blockers are already recorded and are **not** caused by this change; do not
chase them and do not file duplicates:

- `just check-full` fails at committed-plan validation because 21 existing August 2026
  `tier: tale` plans declare `size: large`. Tracked on epic `sase-il.7`.
- The full `just test-visual` lane hits prompt-catalog convergence timeouts unrelated to
  the Models panel. Tracked on epic `sase-iy.2`. Scope the visual run to the
  Models-panel files to get a clean signal.
