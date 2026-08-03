---
tier: tale
title: Fix clan summary view hints for logical plan references and archived prompts
goal:
  Pressing `v` on an agent clan row numbers each clan-summary path from its true first character and opens the exact
  file that path names, so a `plans:` plan reference is marked and resolved as one token and the `Prompt:` row opens the
  archived prompt in the agents sidecar instead of the plan.
proposed_by: bbugyi200.athena.sm
create_time: 2026-08-03 07:31:52
status: wip
---

# Plan: Fix clan summary view hints for logical plan references and archived prompts

## Problem

`v` (view files) on an agents-tab clan row annotates the clan detail document in place with `[N]` markers. For an epic
clan the annotated summary currently renders like this (real output, epic `sase-ej`):

```
 Counts: 6 phases · 5 waves
   Path: plans:[1] 202608/async_sidecar_publication.md
 Prompt: [2] prompts/202608/async_sidecar_publication.md
   Bead: sase-ej
   Page: https://github.com/sase-org/sase--beads/blob/main/pages/sase-ej/README.md
```

Three defects are visible or latent here.

1. **The marker lands inside the plan reference.** `[1]` must precede `plans:`, not sit between `plans:` and the path.
2. **`2` opens the plan file, not the archived prompt.** The `Prompt:` row names a path inside the _agents sidecar_
   repository, and nothing maps it there; an over-eager alias fallback then silently redirects it to the plan.
3. **The clan worker's plan-reference index is dead code for every epic summary**, because it scans Rich _markup_
   instead of the rendered plain text.

### Where the summary comes from

`src/sase/scripts/sase_clan_summary_epic.py` renders the epic summary once at launch through `render_plan_document()`
(`src/sase/sdd/_plan_display_rendering.py`) and stores the resulting **Rich markup string** on the agent as
`clan_summary` (persisted in `agent_meta.json`). The TUI later parses it with `Text.from_markup()` in
`build_clan_detail_text()` (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan.py`) and, in hint mode, annotates
it via `container_text_with_file_hints()`.

The stored markup for the two interesting rows is:

```
[dim]   Path: [#87AFFF][/dim]plans:202608/[bold #87AFFF][/#87AFFF]async_sidecar_publication.md[/bold #87AFFF]
[dim] Prompt: [#87AFFF][/dim]prompts/202608/async_sidecar_publication.md[/#87AFFF]
```

Note that `_path_value()` styles the dirname and the basename separately, so **the plan reference is split by a style
tag in the markup** even though it is contiguous in the plain text.

### Root cause 1 — marker placement

`_FILE_PATH_PATTERN` in `src/sase/ace/tui/widgets/prompt_panel/_file_path_hints.py` has no alternative that includes a
logical SDD reference prefix, and its lookbehind `(?<![/\w@.])` does not exclude `:`. So the match starts at `202608/`
and `append_text_with_file_hints()` inserts `[1] ` there, producing `plans:[1] 202608/…`.

`plans:` is the only canonical logical reference kind. Verified against the Rust grammar:

```python
sase_core_rs.plan_reference_parse("plans:a/b.md")    # {'kind': 'plans', 'path': 'a/b.md',       'legacy': False}
sase_core_rs.plan_reference_parse("prompts:a/b.md")  # {'kind': 'plans', 'path': 'prompts:a/b.md','legacy': True}
```

### Root cause 2 — `2` opens the plan

`_build_clan_hint_paths()` in `src/sase/ace/tui/widgets/prompt_panel/_agent_clan_aggregation.py` builds a
displayed-token → absolute-path index off-thread. For the `PLAN` context lane it indexes the entry label
(`plans:202608/async_sidecar_publication.md`) and key (the plan's absolute path) through `_index_hint_path()`, which
registers **every path-component suffix**. That produces, among others:

```
"202608/async_sidecar_publication.md" -> <plan absolute path>
"async_sidecar_publication.md"        -> <plan absolute path>
```

`container_hint_path_resolver()` in `src/sase/ace/tui/widgets/prompt_panel/_container_hint_text.py` then matches a
displayed token against index tokens in **both** directions:

```python
if not (
    normalized_token.endswith(f"/{normalized_path}")
    or normalized_path.endswith(f"/{normalized_token}")   # <-- the defect
):
    continue
```

The second direction lets the _shorter_ index alias `202608/async_sidecar_publication.md` capture the _longer_ displayed
token `prompts/202608/async_sidecar_publication.md`, so hint `2` resolves to the plan file. Because `_index_hint_path()`
already registers every suffix of every indexed value, this direction can never match the same file — it can only ever
match a _different_ file that happens to share a trailing path segment. It is a pure false-positive source, and it is
also why the bare-basename aliases exist at all (`_FILE_PATH_PATTERN` never produces a bare `name.md` token, so
`"async_sidecar_publication.md"` is only ever reachable through it).

Even with that direction removed, `prompts/<YYYYMM>/<name>.md` has no correct target. It is the canonical archived
prompt reference written by `_archived_prompt_reference_for_plan()` in `src/sase/sdd/plan_header_writes.py`, and it is
relative to the **agents sidecar checkout**, not to the agent workspace. On this machine:

```
/home/bryan/.sase/projects/gh_sase-org__sase/repos/agents/prompts/202608/async_sidecar_publication.md
```

which is exactly `hidden_sidecar_clone_dir(project_key, AGENTS_SIDECAR_ROLE)` from `src/sase/_linked_repo_paths.py`
(re-exported from `sase.linked_repos`). Today both hints fall through to a blind workspace join and name files that do
not exist.

### Root cause 3 — the worker scans markup

```python
_LOGICAL_PLAN_REFERENCE_RE = re.compile(r"(?<![\w])@?([A-Za-z][\w-]*:[\w.+\-/]+)")
...
for match in _LOGICAL_PLAN_REFERENCE_RE.finditer(agent.clan_summary or ""):
```

`agent.clan_summary` is markup. Against the real summary this yields:

- `plans:202608/` — truncated at the `[bold #87AFFF]` tag, which then fails with
  `validation: plan reference path contains an empty segment` and is dropped.
- `https://github.com/sase-org/sase--beads/blob/main/pages/sase-ej/` — the generic `[A-Za-z][\w-]*:` prefix also matches
  URL schemes, so every render wastes a Rust `plan_reference_parse` + `plan_reference_resolve` round trip on a reference
  that always resolves `missing`.

So the whole "resolve clan hint paths off-thread" index contributes nothing for epic clan summaries; hint `1` only
happens to resolve because the `PLAN` **context lane** independently indexes the same plan.

## Scope decisions to preserve

1. **Do not change what `sase_clan_summary_epic.py` writes.** `clan_summary` is captured at launch and already exists in
   thousands of stored `agent_meta.json` files; the fix must work against the markup that is already persisted.
2. **Do not do filesystem or Rust work on the render thread.** Path resolution stays in `_build_clan_hint_paths()`,
   which runs inside the clan enrichment worker. `build_clan_detail_text()` must keep resolving only through the
   worker-supplied index plus the pure workspace-join fallback. `tests/ace/tui/widgets/test_agent_display_clan_hints.py`
   already asserts this by monkeypatching `resolve_plan_reference` to raise.
3. **Do not change prompt-editor jump/preview token boundaries.** `iter_file_path_matches()` is shared with
   `src/sase/ace/tui/widgets/_prompt_jump_target.py` and `_prompt_preview_target.py`. The new logical-reference-aware
   matching must be opt-in and used only by container (clan/family/tribe) hint text.
4. **Do not edit `sase/memory/*.md`, `AGENTS.md`, or the generated provider shims** (`CLAUDE.md`, `GEMINI.md`,
   `OPENCODE.md`, `QWEN.md`).
5. **Do not modify the Rust core.** Everything here is TUI presentation and TUI-side path resolution; the reference
   grammar is already owned by `sase_core_rs` and is only being _consumed_.

## Implementation

### 1. Teach the hint matcher about logical plan references (opt-in)

`src/sase/ace/tui/widgets/prompt_panel/_file_path_hints.py`

- Add a module constant for the one canonical logical kind, e.g. `LOGICAL_PLAN_REFERENCE_PREFIX = "plans:"`, with a
  comment pointing at `sase_core_rs.plan_reference_parse` as the source of truth for why it is the only kind.
- Build a second path pattern that wraps the existing alternatives with an optional `(?:plans:)?` **inside group 2**, so
  the matched span already contains the prefix and group numbering is unchanged:

  ```python
  _CONTAINER_FILE_PATH_PATTERN = (
      r"(?<![/\w@.])"
      r"(@?)"
      r"((?:plans:)?(?:" + <existing alternatives> + r"))"
  )
  ```

  Keep the HTTP/HTTPS alternative first in the combined regex exactly as today so URLs still win.

- Expose `iter_container_file_path_matches(content)` alongside the existing `iter_file_path_matches(content)`. Define a
  `FileHintMatcher` protocol/alias for `Callable[[str], Iterator[re.Match[str]]]`.
- Add `file_hint_match_span(match) -> tuple[int, int]`, returning
  `(match.start(1) if match.group(1) else match.start(2), match.end(2))`. Replace the four hand-rolled copies of that
  expression with calls to it: `append_text_with_file_hints()`, `container_text_with_file_hints()`,
  `_drop_partial_trailing_path()` in `_hint_caps.py`, and the two prompt-target modules if they duplicate it. This is
  what keeps the marker insertion point and the span-shift arithmetic from drifting.
- Add a keyword-only `matcher: FileHintMatcher = iter_file_path_matches` parameter to `append_text_with_file_hints()`
  and use it in place of the direct `iter_file_path_matches()` call.

`src/sase/ace/tui/widgets/prompt_panel/_hint_caps.py`

- Thread the same keyword-only `matcher` through `bound_hint_content()` and `_drop_partial_trailing_path()` so a
  byte-truncated fragment cannot split `plans:` away from its path, and through `append_bounded_text_with_file_hints()`.

### 2. Resolve logical references and archived prompts correctly

`src/sase/ace/tui/widgets/prompt_panel/_container_hint_text.py`

- `container_text_with_file_hints()` passes `iter_container_file_path_matches` as the matcher to both
  `bound_hint_content()` and `append_text_with_file_hints()`, and uses it for the `insertions` tuple it builds for span
  shifting. All three must use the same matcher or spans will be misplaced.
- Rework `container_hint_path_resolver()`:
  - It must **always** return a resolver (never `None`), because the logical-prefix stripping below has to happen even
    when the index is empty. Accept an empty/absent mapping.
  - Resolution order for a displayed token:
    1. exact hit on the raw token, then on its normalized form;
    2. exact hit on the token with a leading `plans:` removed (the index stores `parsed.path`, i.e.
       `202608/async_sidecar_publication.md`), then on its normalized form;
    3. the suffix scan, keeping **only** `normalized_token.endswith(f"/{normalized_path}")`. Delete the
       `normalized_path.endswith(f"/{normalized_token}")` branch — it is the hijack.
    4. fallback `resolve_file_path(<token with any leading `plans:` stripped>, workspace_dir)`, so an unindexed
       reference joins as `<workspace>/202608/…` rather than the nonsense `<workspace>/plans:202608/…`.
- Update the callers so the resolver is always installed for container text. In
  `src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan.py`, drop the `snapshot.disk is not None` guard around
  `container_hint_path_resolver(...)` (pass `snapshot.disk.hint_paths` when a disk snapshot exists, otherwise an empty
  mapping). Prefer routing this through `container_text_with_file_hints()` itself if that keeps the family/tribe callers
  (`_agent_display_clan_sections.py`, `_agent_display_family_render.py`) consistent without extra plumbing; either shape
  is acceptable as long as every container fragment gets the prefix-aware fallback.

### 3. Fix and extend the worker-side index

`src/sase/ace/tui/widgets/prompt_panel/_agent_clan_aggregation.py`

- **Scan plain text, not markup.** Extract the summary-to-`Text` conversion currently inlined in
  `build_clan_detail_text()` (`Text.from_markup()` with a `MarkupError` fallback to `Text(raw)`) into one shared helper
  — put it next to the other clan helpers, e.g. `clan_summary_text(agent) -> Text` in `_agent_display_clan.py` or a
  small shared module — and have both the renderer and `_build_clan_hint_paths()` use it. The worker scans
  `clan_summary_text(agent).plain`.
- **Anchor the reference regex on the canonical kind.** Replace `_LOGICAL_PLAN_REFERENCE_RE` with a `plans:`-anchored
  pattern built from `LOGICAL_PLAN_REFERENCE_PREFIX`, e.g. `re.compile(r"(?<![\w])@?(plans:[\w.+\-/]+)")`. This also
  removes the `https://…` false positive and its wasted Rust calls.
- **Index the archived prompt reference.** After the plan-reference pass, scan the same plain summary for archived
  prompt references with a pattern equivalent to the one already used in `src/sase/llm_provider/commit_finalizer_git.py`
  (`prompts/\d{6}/[^/]+\.md`), guarded so it is not preceded by a word character or `/`. Resolve the agents sidecar root
  once per build via `hidden_sidecar_clone_dir(project_key, AGENTS_SIDECAR_ROLE)` where `project_key` is
  `Path(row.project_file).parent.name` for the same row `_clan_hint_workspace_context()` selected. Index
  `<root>/<reference>` only when it `is_file()`; otherwise leave the token unindexed so it keeps the workspace fallback.
  Wrap the whole pass in a broad `except Exception: pass`-style guard consistent with the surrounding best-effort code —
  a missing or unmaterialized sidecar must never break clan enrichment.
  - Index this reference **exactly** (the displayed token plus the absolute target), _not_ through
    `_index_hint_path()`'s suffix expansion. Otherwise `202608/<name>.md` would become an alias for the prompt and could
    capture the plan row's own `plans:`-stripped lookup. Add an `expand_suffixes: bool = True` keyword to
    `_index_hint_path()` (or a small sibling helper) for this.
- **Stop registering single-component suffix aliases.** In `_index_hint_path()`, skip derived suffixes with fewer than
  two path components. The full displayed value and the `kind:`-stripped remainder are still indexed as-is, so a context
  entry whose label genuinely _is_ a bare filename keeps working. After step 2 removes the reverse suffix match,
  bare-basename aliases are unreachable anyway, and dropping them shrinks the linear scan in
  `container_hint_path_resolver()`.

### 4. Expected result

For the real `sase-ej` summary the annotated rows become:

```
   Path: [1] plans:202608/async_sidecar_publication.md
 Prompt: [2] prompts/202608/async_sidecar_publication.md
```

with

```
1 -> <workspace>/sase/repos/plans/202608/async_sidecar_publication.md
2 -> ~/.sase/projects/<project key>/repos/agents/prompts/202608/async_sidecar_publication.md
```

## Tests

Update these existing tests, which currently pin the buggy behavior:

- `tests/ace/tui/widgets/test_agent_display_clan_hints.py::test_clan_summary_prefers_worker_resolved_plan_path` —
  asserts `"Plan: plans:[1] 202608/clan.md"`. Change to `"Plan: [1] plans:202608/clan.md"`. Its index is
  `{"202608/clan.md": resolved}`, so it also proves the `plans:`-stripped exact lookup works; keep the
  `resolve_plan_reference` monkeypatch that fails the test if the renderer touches the plan store.
- `tests/ace/tui/widgets/test_agent_clan_aggregation_async.py::test_clan_worker_indexes_known_context_path_suffixes` —
  asserts `hint_paths["findings.md"]`. Replace that line with an assertion that the bare-basename alias is **not**
  indexed while `hint_paths["reports/findings.md"]` still is.
- `tests/ace/tui/widgets/test_agent_display_clan_hints.py::test_clan_summary_matches_known_artifact_suffix` must keep
  passing unchanged — it is the regression guard for the suffix direction that is being kept.

Add these:

- **Marker placement.** A `plans:` reference in a clan summary renders `[1] plans:<path>` and the whole reference,
  prefix included, carries the hint's path style.
- **Prompt row is not hijacked.** A clan summary containing both `Path: plans:202608/x.md` and
  `Prompt: prompts/202608/x.md`, with a `hint_paths` index built the way the worker builds it (plan reference indexed
  with suffix expansion), resolves hint 1 to the plan and hint 2 to the prompt — never both to the plan. This is the
  direct regression test for the reported bug.
- **Worker parses a markup summary.** `build_clan_disk_snapshot()` fed a `clan_summary` that is real
  `render_plan_document()`-style markup (with the style tag splitting the reference) indexes `plans:<month>/<name>.md`
  and `<month>/<name>.md`. Building the fixture through `_render_plan_summary()` from
  `sase.scripts.sase_clan_summary_epic`, as `tests/ace/tui/visual/test_ace_png_snapshots_agents_clan_panel.py` already
  does, keeps it honest.
- **Worker ignores HTTP URLs.** A summary whose only colon-bearing token is a `https://…` bead page URL performs no
  plan-reference resolution (assert with a `Mock` on `resolve_plan_reference`) and indexes nothing from it.
- **Archived prompt indexing.** With a fake project key whose
  `hidden_sidecar_clone_dir(project_key, "agents")/prompts/<month>/<name>.md` exists (monkeypatch the projects dir into
  `tmp_path`), the worker indexes that exact token to that file; when the file is absent, the token stays unindexed and
  the renderer falls back to the workspace join.
- **Prompt token is exact-only.** The archived prompt indexing does not register `<month>/<name>.md`, so a plan row in
  the same summary still resolves to the plan.
- **Matcher isolation.** `iter_file_path_matches()` (the non-container matcher) still does **not** swallow a `plans:`
  prefix, so prompt-editor jump/preview token boundaries are unchanged. Assert this directly in
  `tests/test_file_path_hints.py`.
- **Byte-truncation.** `bound_hint_content()` with the container matcher does not leave a dangling `plans:` when the cap
  falls inside the reference.

Visual coverage: add **one new** PNG snapshot in `tests/ace/tui/visual/test_ace_png_snapshots_agents_clan_panel.py` for
hint mode over a summary that has both a `plans:` `Path:` row and a `prompts/` `Prompt:` row. Do **not** change the
shared `_EPIC_CLAN_SUMMARY` fixture — it currently uses a plain `display_path` and no `PROMPT` provenance, so leaving it
alone keeps `agents_clan_panel_epic_120x40`, `_level_2`, `_level_3`, and `_hints` goldens untouched. Generate the new
golden with `just test-visual --sase-update-visual-snapshots` and inspect the produced PNG before committing it.

## Docs

`docs/ace.md`, in the paragraph added by commit `624db9a9f` describing clan `v` hints (Clan and Family Detail Panels
section): add a sentence that a logical `plans:` reference is marked and resolved as one token including its prefix, and
that an archived `prompts/<YYYYMM>/<name>.md` reference resolves into the project's agents sidecar checkout rather than
the agent workspace. No help-modal or keymap change is needed — `v`'s label and bindings are unaffected.

## Proposed follow-up (do not implement here)

The per-agent PLAN lane (`ResponsivePlanSection` in `src/sase/ace/tui/widgets/prompt_panel/_agent_plan_section.py`) only
hints the `Path:` row via its explicit `hint_number`; the `Prompt:`, `Parent:`, and `Bead:` provenance rows produced by
`plan_provenance_rows()` are never hintable, so `v` on an ordinary agent cannot open the archived prompt at all. That is
a feature gap rather than a regression and belongs in its own task bead.

## Verification

```bash
just install
just check
```

`just install` first: these workspaces are ephemeral and dependencies may have moved since this one was last used.

Also run the focused suites while iterating:

```bash
.venv/bin/pytest tests/ace/tui/widgets/test_agent_display_clan_hints.py \
                 tests/ace/tui/widgets/test_agent_clan_aggregation_async.py \
                 tests/ace/tui/actions/test_view_files_clan_hints.py \
                 tests/test_file_path_hints.py
just test-visual
```

Finally, sanity-check against real data. This one-liner reproduces the reported bug today and must show the corrected
markers and targets afterwards (substitute any stored epic clan summary):

```bash
.venv/bin/python - <<'PY'
import glob
import json
import os

from rich.text import Text

from sase.ace.tui.widgets.prompt_panel._agent_display_state import HeaderHintState
from sase.ace.tui.widgets.prompt_panel._container_hint_text import (
    container_hint_path_resolver,
    container_text_with_file_hints,
)

pattern = os.path.expanduser("~/.sase/projects/*/artifacts/ace-run/*/*/*/agent_meta.json")
for path in sorted(glob.glob(pattern)):
    summary = json.load(open(path)).get("clan_summary") or ""
    if "plans:" not in summary:
        continue
    state = HeaderHintState(1, {}, None, {})
    text = container_text_with_file_hints(
        Text.from_markup(summary),
        state,
        workspace_dir="/tmp/ws",
        budget=None,
        path_resolver=container_hint_path_resolver({}),
    )
    print(path)
    print("\n".join(line for line in text.plain.splitlines() if "[" in line))
    print(state.hint_mappings)
    break
PY
```

Before the fix this prints `Path: plans:[1] 202608/…`; after it, `Path: [1] plans:202608/…` with hint 1 joined from the
plan reference's path rather than from the raw `plans:`-prefixed token.
