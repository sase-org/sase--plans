---
tier: tale
title: Make v resolve logical plan references reliably
goal:
  Picking a `v` hint on any `plan:` reference opens the real archived plan, with no new
  render-path I/O.
size: medium
decisions:
  agent_text:
    ask:
      Should ordinary agent prompt/reply text also hint `plan:` refs as whole tokens
      resolved on pick?
    choices:
      include:
        Marker sits before `plan:` in all agent bodies and opens the plan; may regen a
        hint golden
      clan_only:
        Only clan/session text changes; agent bodies keep splitting `plan:` refs into
        dead paths
    default: include
    why: Same defect class and mechanism, with no added render-path cost
    answer: include
  prompt_repair:
    ask:
      Should a missing archived-prompt hint be retried in the project's agents sidecar
      on pick?
    choices:
      repair:
        Summary Prompt row opens even before clan enrichment lands; one off-thread stat
        on a miss
      skip: Prompt row stays dead until enrichment lands and hints are re-published
    default: repair
    why: Second row of the same summary block, with the same race; tiny fix
    answer: repair
proposed_by: bbugyi200.athena.10s
decided_by: auto
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.10s](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.10s.md)
- **COMMITS:**
  - [a9701c3](https://github.com/sase-org/sase/commit/a9701c36e6cd7f251f3d0b1308eaf22156a12c7e)
    — feat(ace): resolve logical plan hint targets

# Plan: Make `v` resolve logical `plan:` references reliably

## Problem

On the Agents tab, select an epic clan (e.g. `bob-cli-87`), press `v`, and pick the hint
on the summary's `Path: plan:202610/note_snapshots.md` row. The TUI shows:

```
File no longer exists: <bob-cli workspace>/plan:202610/note_snapshots.md
No selected files could be opened
```

The plan does exist.
`resolve_plan_reference("plan:202610/note_snapshots.md", workspace_dir=<that workspace>, ...)`
returns `status='exact'` and finds the workspace's
`sase/repos/plans/202610/note_snapshots.md`. The project primary and `~/.sase/plans/`
also have copies. So the hint pipeline never asked the plan resolver; it built a dead
path.

## Root causes (verified against live data and code)

1. **The clan PLAN context lane naively joins logical refs to a workspace path, and that
   bad target poisons the alias index.** Gate, monitor, and land members of an epic clan
   now carry `sdd_plan_path = "plan:<YYYYMM>/<name>.md"` in `agent_meta.json`. This was
   confirmed on the bob-cli-87 members (`.2--gate`, `.2--mon`, `.land`, ...).
   `_in_memory_plan_paths()` (`src/sase/ace/tui/models/_agent_clan_sections.py`) passes
   those raw strings into `ClanInMemorySnapshot.plan_paths`. They become PLAN
   context-lane entries whose `values` are plain strings
   (`aggregate_clan_context_lanes()` in `_agent_clan_disk_aggregation.py`, and
   `minimal_context_lanes()` in `_agent_display_clan_sections_context.py`). Then
   `_typed_context_value_path()` in `_agent_display_clan_context.py` runs
   `_resolve_path(value, member_workspace)`, which is
   `os.path.join(workspace, "plan:202610/note_snapshots.md")`.
   - In the worker, `_build_clan_hint_paths()` (`_agent_clan_aggregation.py`) indexes
     context-lane targets **first**, through `_index_hint_path()`, which uses
     `setdefault` (first wins). The logical token `plan:202610/note_snapshots.md`
     therefore maps to the joined dead path. The later summary loop correctly resolves
     the same ref through the shared resolver, but `setdefault` silently drops its
     result.
   - The summary resolver (`container_hint_path_resolver()` in
     `_container_hint_text.py`) then finds the poisoned exact alias. That is exactly the
     path in the screenshot.
   - At fold level 3, the clan `SASE CONTEXT` PLAN row hint itself also targets the
     joined dead path.
2. **Before clan enrichment lands, logical refs fall back to a blind join.** With no
   disk snapshot, `hint_paths` is `{}`, so the container resolver strips `plan:` and
   joins: `<workspace>/202610/note_snapshots.md`. That path is also dead. Mappings are
   re-published when enrichment lands during hint mode
   (`_render_agent_detail_with_hints`), but a quick submission uses the dead target. The
   `Prompt: prompts/<YYYYMM>/<name>.md` row has the same window and resolves to
   `<workspace>/prompts/...` instead of the agents sidecar.
3. **The container resolver's fuzzy suffix fallback can hijack logical tokens.** For an
   unindexed `plan:X/Y.md` it tries the stripped `X/Y.md` key, then any indexed token
   ending in `/X/Y.md`. That includes the archived prompt `prompts/X/Y.md` and any
   suffix alias of an ARTIFACTS path. So a plan hint can silently open a different file.
4. **(decision `agent_text`) Ordinary agent bodies split logical refs.**
   `append_bounded_text_with_file_hints()` (`_hint_caps.py`) defaults to the generic
   `iter_file_path_matches`. For `plan:202610/x.md` that matcher yields only
   `202610/x.md` (`plan:[N] 202610/x.md`), targeting `<workspace>/202610/x.md`. This
   affects AGENT PROMPT, AGENT CHAT/REPLY, monitor, gate, named-proc, and traceback
   text, where phase prompts and replies routinely cite `plan:` refs.

## Design

A logical `plan:` reference is an **identity, not a path**, the same way `bead:<id>` and
`toolrun-log:<id>` hint targets already are (`src/sase/ace/tui/bead_hint_targets.py`,
`src/sase/ace/tui/tool_runs/hints.py`).

- **Render path (on the event loop, string work only):** a logical plan token maps to
  one of two targets. If the clan alias index holds an exact alias keyed by the **full
  logical token**, use that worker-verified absolute path. Otherwise use the canonical
  `plan:<YYYYMM>/<name>.md` string itself as the hint target. Never workspace-join a
  logical ref, and never fuzzy-match one.
- **Clan worker (already off-thread):** resolve each unique logical ref once through the
  shared resolver and make those aliases authoritative, so the fast path is correct and
  never poisoned.
- **Submit (already off-thread in `materialize_selected_view_files`, via
  `asyncio.to_thread`):** a selected target that is a canonical `plan:` ref is resolved
  through the captured link context. Its anchors are the selected agent's workspace, the
  project primary, then the defaults, and `~/.sase/plans` is always a plan root.
  Resolution runs only for selected targets and is memoized per request. View (`v N`),
  editor (`N@`), and copy (`N%`) all go through materialization first, so all three get
  a real path.

Rust boundary: no new domain logic. Plan grammar and resolution already live in
`sase_core_rs`, and are consumed via `sase.sdd.plan_refs`. What maps a hint number to a
target is TUI glue, so no `sase-core` change and no `sase-core-revision.txt` bump.

## Implementation

### 1. Logical-plan hint target helpers (render-safe)

`src/sase/ace/tui/widgets/prompt_panel/_file_path_hints.py`

- Add `logical_plan_hint_target(token: str) -> str | None`. It strips whitespace and a
  leading `@`. If the result starts with `LOGICAL_PLAN_REFERENCE_PREFIX` and has a
  non-empty remainder, it returns the canonical `plan:<path>` string; otherwise `None`.
  **Pure string work:** no `parse_plan_reference`, no Rust, no filesystem. Malformed
  refs are reported at submit time.
- Add `resolve_hint_target(path, workspace_dir) -> str`. It returns
  `logical_plan_hint_target(path)` when that is not `None`, else
  `resolve_file_path(path, workspace_dir)`.
- `append_text_with_file_hints()` uses `resolve_hint_target` when no `path_resolver` is
  passed (today it calls `resolve_file_path` directly).

### 2. Container resolver: exact-or-identity for logical tokens

`src/sase/ace/tui/widgets/prompt_panel/_container_hint_text.py`,
`container_hint_path_resolver()`

- If `logical_plan_hint_target(path)` returns a ref, look up **only**
  `hint_paths.get(<canonical ref>)`. Also try the raw token, for an `@`-prefixed
  spelling. Return that alias if present, else the canonical ref. Skip the stripped-key
  lookup, the suffix loop, and the workspace join for logical tokens. This fixes root
  causes 2 and 3.
- Non-logical tokens keep today's behavior exactly: exact alias, longest suffix alias,
  then `resolve_file_path`.
- `_strip_logical_plan_prefix()` should become dead; delete it rather than leave an
  unused symbol for symvision.

### 3. Clan PLAN context lane never joins logical refs

`src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan_context.py`,
`_typed_context_value_path()`

- For `lane_label == "PLAN"` with a `str` value: return
  `logical_plan_hint_target(value)` when it is a logical ref, otherwise keep
  `_resolve_path(value, member_workspace)` for legacy relative/absolute plan paths.
  `AssociatedPlanSummary` values are unchanged (`actual_path`).
- This fixes the fold-level-3 CONTEXT row on both the minimal (in-memory) and enriched
  lanes, because both call `clan_context_entry_hint_target()`.

### 4. Clan worker: authoritative, memoized logical aliases

`src/sase/ace/tui/widgets/prompt_panel/_agent_clan_aggregation.py`,
`_build_clan_hint_paths()`

- Build one per-call memo, `dict[str, str | None]`, from canonical logical ref to
  resolved absolute path, filled with
  `resolve_plan_reference(ref, workspace_dir=fallback_workspace, workspace_num=workspace_num)`.
  Only `resolution.resolved_path` counts (exact/drifted). Wrap each call in the existing
  `try/except Exception: continue` shape.
- Collect logical refs from **both** the summary plain text (existing
  `_LOGICAL_PLAN_REFERENCE_RE` scan) and PLAN context-lane `str` values. Resolve each
  unique ref once, and **index these aliases before** the generic context-lane loop, so
  first-wins can no longer let a lane target shadow them. Keep the existing
  `_index_hint_path(..., parsed.path, ...)` and target self-alias registrations.
- In the context-lane loop, index a lane target only when it is an absolute path. A
  canonical `plan:` identity returned by step 3 must never be indexed as an alias
  target. Use the memo's resolved path for logical PLAN values when present.
- Performance: an epic clan typically has one unique plan ref repeated across ~20
  members. The summary loop already resolved it, so the shared memo keeps worker
  resolver calls at **≤ the number of unique logical refs**, which is net zero new calls
  in the common case.

### 5. Submit-time resolution of logical targets (+ archived prompts)

`src/sase/ace/tui/actions/hints/_artifact_ref_repair.py` (or a sibling
`_view_target_repair.py` if clearer) and
`src/sase/ace/tui/actions/hints/_view_materialize.py`,
`materialize_selected_view_files()`

- Add `resolve_logical_plan_target(ref, link_context) -> str | None`. For each
  `link_context.anchors` entry in order, call
  `resolve_plan_reference(ref, workspace_dir=anchor.directory, workspace_num=anchor.workspace_num or 1)`
  and return the first `resolved_path` that `is_file()`. Catch
  `ValueError`/`RuntimeError`/`OSError` per anchor; malformed refs just fail to resolve.
  Memoize per materialize call (`dict`), since several hints can name the same plan.
- In the per-file loop, check `logical_plan_hint_target(file_path)` **before** any
  `os.path.exists`. A logical target either materializes to its resolved path or is
  reported as unresolved (below). It must never be stat'd relative to the TUI's CWD.
- Missing logical refs: carry them in `MaterializedReports` so `_finish_view_request()`
  (`src/sase/ace/tui/actions/hints/_view_input.py`) can show
  `Could not resolve plan:<YYYYMM>/<name>.md` instead of
  `File no longer exists: <bogus path>`. A separate tuple field or a tagged missing
  entry are both fine; keep the dataclass change minimal (`_view_models.py`).
- Confirm `bead_id_from_hint_target()` and `run_id_from_hint_target()` return `None` for
  `plan:` strings. They key off their own prefixes, so `_prepare_view_input()` routes
  `plan:` targets into `files`. Add an assertion test.

> [!decision] prompt_repair = repair After the existing artifact-read repair, a path may
> still be missing. If its tail matches `prompts/<6 digits>/<name>.md` (reuse the shape
> of `_ARCHIVED_PROMPT_REFERENCE_RE`) and the captured context has a `project_file`, try
> `Path(hidden_sidecar_clone_dir(Path(project_file).parent.name, AGENTS_SIDECAR_ROLE)) / <tail>`,
> mirroring `_index_archived_prompt_references()`. Use it if `is_file()`. Pass the
> captured `project_file` into materialization; `CapturedLinkContext.project_file`
> already exists for agent sources, including clan containers. Test: under a tmp
> `SASE_HOME` (or whatever env/fixture existing tests use for `sase_subdir`), a missing
> `<ws>/prompts/202610/x.md` repairs to the agents-sidecar copy and stays missing when
> that copy is absent.

### 6. Logical-aware matching for ordinary agent bodies

> [!decision] agent_text = include In
> `src/sase/ace/tui/widgets/prompt_panel/_hint_caps.py`, change
> `append_bounded_text_with_file_hints()`'s default `matcher` to the logical-plan-aware
> `iter_container_file_path_matches`. That pattern equals the generic one plus an
> optional `plan:` prefix inside group 2, so every other match is unchanged. Keep
> passing the same matcher to `bound_hint_content()` so budget accounting stays
> consistent. Path resolution comes from step 1's default `resolve_hint_target`. Callers
> relying on the default pick this up with no call-site edits:
> `_agent_display_hint_body.py` AGENT PROMPT/CHAT/REPLY,
> `_agent_display_hint_annotators.py` reply/monitor/gate/named-proc, and
> `_traceback_section.py`. The raw-prompt call that passes
> `iter_raw_prompt_file_path_matches` stays as is: typed refs there are artifact-ref
> overlays, not file hints. **Do not** touch `iter_file_path_matches` itself. It is
> shared with the prompt-editor jump/preview targets (`_prompt_jump_target.py`,
> `_prompt_preview_target.py`), whose token boundaries must not change. Test: a
> single-agent reply containing `see plan:202610/x.md and src/a.py` renders
> `see [1] plan:202610/x.md and [2] src/a.py`, with mappings
> `{1: "plan:202610/x.md", 2: "<ws>/src/a.py"}`. Also assert that
> `iter_file_path_matches` boundaries are unchanged. Check whether any Agents hint-mode
> PNG golden renders a `plan:` ref in an ordinary body. If one does, regenerate only
> that golden with `just fix-tui-screenshots`, scoped to the affected test, and say why
> in the commit.

> [!decision] agent_text = clan_only Skip this step and its tests. Ordinary agent bodies
> keep the generic matcher; everything else in this plan stands.

### 7. Docs

Before editing, read `docs_writing.md` through `/sase_memory_read`. In
`docs/reference/tui/agents-tab-panels.md`, update the clan `v` paragraph, which
currently says a logical `plan:` reference "is marked and resolved as one token". State
that a logical `plan:` reference opens the archived plan: it is resolved when you pick
it, through the selected agent's workspace, the project primary, and the machine plan
archive, never by joining it to a workspace path. If step 6 or the step 5 prompt repair
lands, mention it in the same sentence. Ordinary agent prompt/reply text then behaves
the same way, and an archived prompt reference opens from the agents sidecar even before
enrichment finishes. Keep it to one or two sentences. No keybinding, help-modal, or
`default_config.yml` change is involved.

## Performance invariants (must hold)

- **No new render-path I/O.** Hint rendering stays string-only: steps 1–3 and 6 add
  prefix checks and dict lookups. No `resolve_plan_reference`, `parse_plan_reference`,
  stat, or subprocess on the event loop (`tui_perf` rules 1, 8, and 11). Tests enforce
  this by monkeypatching `sase.sdd.plan_refs.resolve_plan_reference` to raise during
  `build_clan_detail_text()` and during a single-agent hint render. Extend the existing
  guard in `tests/ace/tui/widgets/test_agent_display_clan_hints.py`.
- **Worker cost is flat:** resolver calls ≤ unique logical refs per
  `_build_clan_hint_paths()` call. Test this with a counting fake over a multi-member
  clan that repeats one ref.
- **Submit cost only on demand:** resolution happens only for selected logical targets
  (plus one stat for an archived prompt miss), inside the existing `asyncio.to_thread`
  materialization, memoized per request. Measured on this host: about 4–13 ms per anchor
  warm. The common clan case still opens the worker-verified absolute path without any
  resolution.
- The matcher change in step 6 is a single optional group on an existing regex: no extra
  passes over the text.

## Tests

Add or update focused unit tests next to the existing ones:

- `tests/ace/tui/widgets/test_agent_display_clan_hints.py`
  - A summary `Path: plan:X/Y.md` with **empty** `hint_paths` maps to `"plan:X/Y.md"`.
    It must not map to `<ws>/X/Y.md` or `<ws>/plan:X/Y.md`.
  - A summary logical token with only a stripped or suffix alias present (e.g.
    `prompts/X/Y.md` → prompt file, or a bare `X/Y.md` alias) is **not** hijacked and
    maps to `"plan:X/Y.md"`.
  - `test_clan_summary_prefers_worker_resolved_plan_path`: key its `hint_paths` by the
    full logical token `plan:202608/clan.md`, which is what the worker registers, and
    keep the assertion that the resolved path wins. Explain the key change in the test
    docstring.
  - Fold level 3: a CONTEXT PLAN row whose value is the string `plan:X/Y.md` maps to
    `"plan:X/Y.md"`.
- Aggregation tests (`tests/ace/tui/widgets/test_agent_clan_aggregation.py` or a new
  sibling): `_build_clan_hint_paths()` with a PLAN lane string value `plan:X/Y.md` plus
  a summary naming the same ref, with `resolve_plan_reference` monkeypatched to an exact
  tmp path:
  - `hint_paths["plan:X/Y.md"]` equals the resolved path.
  - No value in `hint_paths` contains `/plan:`.
  - The resolver is called once.
- Materialization (`tests/ace/tui/actions/...`, next to the existing
  `materialize_selected_view_files` coverage):
  - A `plan:` target resolves through a `CapturedLinkContext` whose anchor has a real
    tmp plan store (or a monkeypatched resolver).
  - An unresolvable `plan:` target is reported as unresolved and is never stat'd
    relative to CWD.
  - `_prepare_view_input` routes a `plan:` target to `files`, not beads or tool runs.
- Decision-specific tests live in the step 5 and step 6 callouts.

Visual goldens: the clan hint PNG (`clan_panel_epic_logical_prompt_hints_120x40.png`)
should not change, because only targets change, not marker placement.

## Verification

- `sase tool run check` (the agent default). Do not run `just check-full`.
- Manual (optional, if a live TUI is handy): restart `sase tui`, since an editable
  install keeps running the imported snapshot (tui_perf rule 15). Select an epic clan,
  press `v`, and pick the summary `Path:` hint and the `Prompt:` hint both immediately
  after selecting and after enrichment lands. Both should open, and `N@` / `N%` should
  use the real plan path.

## Non-goals

- No change to what `sase_clan_summary_epic.py` writes, or to stored `agent_meta.json`
  (`sdd_plan_path` stays a logical ref).
- No new typed-artifact resolution beyond `plan:` (`prompt:` is not a registered
  artifact kind). No Rust or `sase-core` changes.
- No changes to prompt-editor jump/preview token boundaries, the pager's own link
  navigation, or hint numbering order.
- The single-agent PLAN lane hint keeps using `AssociatedPlanSummary.actual_path`, which
  already goes through the shared resolver.
