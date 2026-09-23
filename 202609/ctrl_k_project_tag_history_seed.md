---
tier: tale
title: Ctrl+K prompt history seeds project:<name> from +<project> tags
goal:
  "Pressing Ctrl+K on a prompt draft that names its project with a +<project> tag
  (including the +sase default the prompt bar pre-fills) seeds the prompt history filter
  with project:<name> plus the remaining text, exactly as #gh:<project> drafts already
  do."
size: small
proposed_by: bbugyi200.athena.0px
create_time: 2026-09-23 09:56:37
status: wip
---

# Plan: Ctrl+K prompt history seeds `project:<name>` from `+<project>` tags

## Problem

Pressing `Ctrl+K` in the prompt input opens the prompt history modal. The modal should
turn the draft's first workspace reference into a `project:<name>` filter, and keep the
rest of the draft as literal search text. `docs/ace.md` ("Prompt History Modal") already
describes this with a `+sase` example.

Since the project tag epic (sase-16n), the prompt bar is pre-filled with a `+<project>`
tag (for example `+sase `), not `#gh:sase `. The seed builder does not recognize tags,
so the tag passes into the filter as literal text:

| Draft passed to Ctrl+K | Seeded filter today  | Expected                   |
| ---------------------- | -------------------- | -------------------------- |
| `#gh:sase fix the bug` | `project:sase fix …` | `project:sase fix …`       |
| `+sase fix the bug`    | `+sase fix the bug`  | `project:sase fix the bug` |
| `+sase ` (bar default) | `+sase `             | `project:sase `            |
| `fix the bug +sase`    | `fix the bug +sase`  | `project:sase fix the bug` |

These results were reproduced against the live project catalog with
`build_prompt_history_seed_from_draft`.

## Root cause

`src/sase/history/prompt_history_project_filter.py::build_prompt_history_seed_from_draft`
finds the draft's workspace reference with two helpers that know only `#` refs:

- `find_vcs_workflow_tag_span`
- `extract_project_from_vcs_tag`

A `+sase` draft gets no span, so `raw_ref=None` is passed with the whole draft as
`remainder_text`. The sase-core `build_prompt_history_seed` then correctly echoes that
remainder as the seed.

The parent epic's backend step 5 (plan `202609/project_tags.md`) listed this module
among the raw-text consumers to switch to tag-aware helpers. Phase sase-16n.3 switched
only `_segment_active_ref`, so history rows written with `+sase` do resolve. The Ctrl+K
draft path was missed. The sase-16n.11 landing-gaps plan does not mention it either. Its
prompt-history items are the red preview label tests and the `,.` MRU bar label. Neither
touches the Ctrl+K seed.

The sase-core seed builder is correct and needs no change: given `raw_ref="sase"` (or
`SASE`), it returns `project:sase …`. The fix belongs only in the thin Python
span-detection adapter. It reuses the sase-core tag scan binding (`find_project_tags`),
so no new core logic, binding, or `sase-core-revision.txt` pin bump is needed.

## Changes

### 1. Tag-aware draft span detection

In `build_prompt_history_seed_from_draft`
(`src/sase/history/prompt_history_project_filter.py`), choose the draft's first **active
workspace reference** from two candidate kinds:

- **VCS ref** (existing behavior):
  - Get the span from `find_vcs_workflow_tag_span(draft)`.
  - Get `raw_ref` from `extract_project_from_vcs_tag(draft[start:end])`.
- **Project tag** (new):
  - Get spans from `sase.project_tags.find_project_tags(draft)`. These are core scan
    spans with Python code-point `start`/`end`, `name`, and `anchored`.
  - Skip a span whose `start` falls inside `literal_zone_ranges(draft)`. The core scan
    already skips inline code, but use the same literal-zone rule as
    `find_vcs_workflow_tag_span` so fenced and disabled regions are covered too.
  - Follow the parent plan's D3 rule. An **anchored** tag always counts as a workspace
    target. An **unanchored** tag counts only if `catalog.resolve_ref(name)` resolves
    it. Otherwise it stays plain text, as D3 says for unknown unanchored tags.
  - `raw_ref` is the span's `name`, without the `+`.

Pick the candidate with the lowest start offset. That keeps the documented "first active
workspace reference" rule however the two forms are mixed.

Remove the chosen span with the existing `_remove_span_with_boundary_whitespace`, then
call `catalog.build_seed(raw_ref=..., remainder_text=...)` as today. With no candidate,
keep the current `raw_ref=None` path.

Resulting behavior:

- An anchored unknown tag (`+nosuch fix parser`) seeds `fix parser` with the existing
  "Project scope unavailable; searching all loaded prompts" hint, exactly like an
  unknown `#gh:` ref.
- An unanchored unknown tag (`fix +nosuch now`) stays in the seed text with no hint.

Keep everything off the keystroke path. The function already runs through
`asyncio.to_thread` in `PromptHistoryModal`, and `find_project_tags` short-circuits when
the draft has no `+`. Import `find_project_tags` lazily inside the function, as
`_segment_active_ref` already does for its tag helpers. Update the function docstring to
say that `+<project>` tags count as workspace references.

### 2. Adapter tests

Extend the parametrized
`tests/history/test_prompt_history_project_filter.py::test_build_prompt_history_seed_from_draft_examples`
table. It uses a catalog with key `gh_sase-org__sase` and label `sase`; add an alias if
you need one. New cases:

- `+sase fix parser` → `project:sase fix parser`
- `+sase` → `project:sase `
- `+sase ` (the prompt-bar default) → `project:sase `
- `+SASE fix parser` → `project:sase fix parser` (case-insensitive)
- `%m:opus +sase fix parser` → `project:sase %m:opus fix parser` (a tag after a
  directive is still anchored, matching the existing `#gh:` directive case)
- `fix +sase now` → `project:sase fix now` (a resolvable unanchored tag)
- ``fix `+sase` now`` → unchanged, no hint (literal zone)
- `fix +nosuch now` → unchanged, no hint (an unknown unanchored tag is plain text)
- `+nosuch fix parser` → `fix parser` with the "Project scope unavailable…" hint
- Mixed forms, the earliest span wins. Example: `+sase fix #gh:sase-org/sase` →
  `project:sase fix #gh:sase-org/sase`. Add the reverse order too.

Keep every existing `#gh:`/`#git:` case passing unchanged.

### 3. Modal-level regression test

In `tests/ace/tui/modals/test_prompt_history_modal_project_filter.py`, add an async test
for the real Ctrl+K flow:

- Open `PromptHistoryModal(prompt_seed="+sase fix parser")`.
- Stub `PromptHistoryProjectCatalog.load` with a catalog containing `sase`, and stub
  `load_prompt_record_page` with an empty page. Follow the stubbing pattern of
  `test_typing_during_seed_resolution_wins_over_the_seed`.
- Assert that the filter input settles on `project:sase fix parser`.

This test must use the real `build_prompt_history_seed_from_draft`, not a stub.

### 4. Docs

`docs/ace.md` ("Prompt History Modal") already says a `+sase` workspace reference
becomes the `project:<name>` scope, so no doc change is expected. Re-read the paragraph
after the fix and adjust it only if the implemented rule differs, for example the
unanchored-unknown-tag case.

## Out of scope

- The `,.` MRU bar label (`_entry_prompt_history.py`, `_entry_custom.py`) and the red
  prompt-history preview label tests. Phase sase-16n.11.3 owns those; do not edit those
  files here.
- Any change to the sase-core `build_prompt_history_seed` /
  `compile_prompt_history_query` API or to the `project:` query grammar.

## Verification

- Run `just install`, then `sase tool run check`. Do not run `just check-full`.
- If `check` still fails only on the known master red
  `symvision: ExpandedLaunchSegments`, record that and move on. That failure belongs to
  task sase-16u.
- Manual spot check through a Python one-liner against the real catalog:
  `build_prompt_history_seed_from_draft("+sase fix the bug", PromptHistoryProjectCatalog.load())`
  returns seed text `project:sase fix the bug`.
