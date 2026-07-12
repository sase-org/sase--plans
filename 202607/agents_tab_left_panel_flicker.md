---
create_time: 2026-07-08 22:43:03
status: done
prompt: 202607/prompts/agents_tab_left_panel_flicker.md
tier: tale
---
# Fix the Agents-tab left-panel flicker

## Problem / product context

On the `sase ace` TUI **Agents tab**, the left-hand agent list (`AgentList`, `#agent-list-panel`) visibly **flickers** —
a repaint sweeping across the whole left side that recurs every few seconds. It is much more noticeable over a
laggy/remote (SSH) connection to the host.

The flicker is **not** primarily a network problem. It is a rendering regression that the slow link merely _amplifies_:
work that is imperceptible on a fast local terminal (because Textual only ships changed cells to the terminal) becomes a
visible sweep when the same repaint is streamed over a high-latency link.

Two independent, compounding causes were found by tracing the refresh path.

### Cause 1 (primary): auto-refresh escalates to a full left-panel rebuild on cosmetic churn

The Agents-tab auto-refresh already has a good incremental path that patches single rows in place and does **scoped**
per-panel rebuilds. But a coarse guard demotes the whole display to a full rebuild far too often:

- The auto-refresh tick reloads agents from disk and calls the incremental refresh
  (`_try_refresh_agents_display_incremental` in `src/sase/ace/tui/actions/agents/_display.py`).
- Before patching, it calls `diff_touches_workflow_tree` (`src/sase/ace/tui/actions/agents/_display_diff.py`). That
  function folds **`changed_same_position`** rows into its `touched` set and returns `True` if any touched row is
  "workflow-shaped". `is_workflow_shaped` matches almost everything real: `agent_type is WORKFLOW`, `is_workflow_child`
  (i.e. any child row), `parent_timestamp is not None`, or `parent_workflow is not None`.
- When it returns `True`, the code records a `workflow_tree_change` fallback and runs the **whole-display** rebuild
  (`_refresh_agents_display(list_changed=True)`), which calls `AgentList.update_list` → `build_list` for **every**
  panel. `build_list` does an unconditional `clear_options()` + `add_options()` and posts a `WidthChanged` message
  (re-padding every row to a freshly computed `target_width`).

The trap: a **same-position content change to any workflow-shaped row** triggers this — even when the change is purely
cosmetic and the tree structure is unchanged. Running workflows / agent families churn _equality-compared_ fields on
**every** disk poll — `activity` (from `workflow_state.json`), `step_output`, and, recursively, `runtime_children` /
`followup_agents` / `retry_chain_siblings`. Because SASE views are usually full of workflow/family rows, the full
rebuild fires on essentially **every** auto-refresh (~5 s while agents are active, plus a 60 s idle sanity refresh).

This escalation is unnecessary: the incremental impl (`_try_refresh_agents_display_incremental_impl`) already knows how
to patch the individual changed rows (`_try_patch_agent_row`) and scope-rebuild only the panels whose banner/membership
changed (`changed_same_position_panel_membership_keys` + `_refresh_affected_panel_widgets`). The `workflow_tree_change`
gate pre-empts all of that.

### Cause 2 (secondary amplifier): the pencil badge flips on a `git diff` timeout

The live "workspace has edits" pencil badge (`live_file_change_hint`) feeds the row render key (via
`agent_file_change_hint` in `src/sase/ace/tui/widgets/_agent_list_render_cache.py`) and, since the pencil was moved into
the **right-aligned suffix**, it also affects a row's rendered width.

`get_agent_diff` (`src/sase/ace/tui/widgets/file_panel/_diff.py`) runs a local `diff_with_untracked(..., timeout=10)`.
On **timeout/error** it returns `None`, which is conflated with "clean tree", so `live_agent_file_change_hint` returns a
concrete **`False`**. The stale-while-revalidate guard in `_apply_live_hint_results`
(`src/sase/ace/tui/actions/agents/_loading_live_hints.py`) only protects a `None` result, so a `False` **overwrites** a
previously-`True` pencil. On a laggy link the 10 s diff intermittently times out, so the pencil flips
`True → False → True`. Each flip:

1. is an equality-visible change on the row → (pre-fix) trips Cause 1; and
2. changes the suffix width → can push a row past the cached `target_width`, which makes even the _targeted_ `patch_row`
   fail and fall back to a full rebuild (`_agent_list_build.py`).

`linked_file_change_hint` was investigated and is **not** a factor: it is derived from persisted on-disk diff artifacts
(mtime-cached), never a live VCS/network call, so it does not oscillate with connection quality.

### Why "recently" and "worse over bad internet" both fit

- _Recently_: the incremental refresh + `workflow_tree_change` gate is the current design; as SASE workflow/agent-family
  usage has grown, workflow-shaped rows now dominate the view, so the escalation that used to be occasional now fires on
  nearly every tick.
- _Worse over bad internet_: (a) full-panel repaints that are invisible locally become visible when streamed over
  latency; and (b) the 10 s diff timeout that flips the pencil is itself triggered by a slow/remote workspace.

## Goals

1. Eliminate the recurring full-panel rebuild on the Agents tab when only cosmetic, same-position content changed —
   patch those rows in place instead.
2. Stop the pencil badge from flipping on a transient `git diff` timeout.
3. Preserve correctness of banner status chips and fold-count annotations (the reason the coarse gate existed).
4. Confirm the fix empirically with the existing refresh telemetry/trace, not just by eye.

Non-goals: redesigning the refresh architecture, changing refresh cadence, touching the per-second runtime-tick path
(already a correct single-row patch), or reworking the ChangeSpec tab (see "Follow-ups").

## Proposed changes

### Change A — narrow `diff_touches_workflow_tree` to structural changes only (primary fix)

Stop treating every `changed_same_position` workflow-shaped row as a tree change. Only escalate to the full/scoped
rebuild when a **tree-structural** field actually changed; route everything else through the existing in-place patch
path.

- In `diff_touches_workflow_tree` (`_display_diff.py`), keep escalating on `removed` / `moved` / `added` workflow-shaped
  identities (genuine structure changes). For `changed_same_position`, only add a row to `touched` when its **structural
  signature** differs between `previous` and `next`.
- Structural signature = the fields that drive banner chips and fold annotations, i.e. `status`, `hidden`,
  `is_workflow_child`, plus the tree linkage fields `raw_suffix`, `parent_timestamp`, `parent_workflow`. (These are
  exactly the member fields `banner_render_key` folds in, plus the linkage identity — so nothing a banner/fold depends
  on can change without escalating.)
- Cosmetic same-position churn (`activity`, `step_output`, `live_file_change_hint`, `diff_has_real_edits`,
  runtime/elapsed, retry counters, aggregated output vars) no longer escalates; it flows into
  `_try_refresh_agents_display_incremental_impl`, which patches the single affected row via `_try_patch_agent_row`.

**Why this is provably banner/fold-safe:** a group banner's rendered content depends only on member
`(identity, status, hidden, is_workflow_child)`, and a parent's fold annotation depends only on child count/visibility.
Any change to those shows up either as a structural-signature change on the row itself, or as an add/remove/move of a
child — all of which still escalate. Cosmetic fields do not participate in banner/fold rendering, so patching just the
row leaves neighbors correct.

### Change B — treat a `git diff` timeout as "unknown", not "clean" (secondary fix)

Make the pencil badge sticky across transient timeouts.

- Distinguish "diff timed out / errored" (unknown) from "no workspace / clean tree" (a real `False`) in the live-hint
  path (`file_panel/_diff.py` → `live_agent_file_change_hint`). On timeout/error, the live classification should return
  **`None`** (unknown) rather than `False`.
- Because `_apply_live_hint_results` already preserves the last-known value when the new result is `None`, the pencil
  then stays put across an intermittent timeout instead of flipping. `carry_over_live_hints` similarly keeps the prior
  value on reload.
- Implementation note: `get_agent_diff` currently swallows the timeout into a `None` return that is indistinguishable
  from "empty diff". The plan is to surface the timeout/error distinctly (e.g. a sentinel/raise that the live-hint
  classifier maps to `None`) **without** changing the persisted-diff / clean-tree behavior, and without caching the
  unknown result (so the next scan re-probes).

### Change C — verification instrumentation (no behavior change)

Use the existing refresh telemetry to prove the regression and its fix:

- The fallback path already emits a `display_full_rebuild` / `workflow_tree_change` trace
  (`_record_display_full_rebuild_fallback`). Confirm (via `SASE_TUI_TRACE=1` and/or the refresh telemetry taxonomy) that
  `workflow_tree_change` is the dominant fallback **before** the fix and effectively disappears for cosmetic-churn ticks
  **after**.

## Testing & verification

- **Unit — gate narrowing (`_display_diff.py`):**
  - A same-position change to `activity` / `step_output` / `live_file_change_hint` on a workflow / workflow-child /
    follow-up row does **not** trip `diff_touches_workflow_tree`.
  - A same-position change to `status` / `hidden` / `is_workflow_child` / linkage fields **does** trip it.
    Added/removed/moved workflow-shaped rows still trip it.
- **Unit/integration — incremental refresh:** a cosmetic same-position change on a workflow row takes the incremental
  path (row patched, no `workflow_tree_change` fallback recorded); banner chip and fold annotation for that group remain
  correct after the patch.
- **Unit — live-hint timeout (`file_panel/_diff.py` + `_loading_live_hints.py`):** a `diff_with_untracked` timeout
  yields `None` (unknown), and applying that result preserves a previously-`True` pencil (no flip); a genuinely clean
  tree still yields `False`.
- **Perf/telemetry:** with active workflow rows, capture the refresh trace before/after and confirm the per-tick
  full-rebuild (`workflow_tree_change`) disappears; sanity-check j/k paint latency is unaffected (`SASE_TUI_PERF=1`,
  target p95 < 16 ms).
- **Manual:** on the Agents tab with a running workflow/family, observe that the left panel no longer sweeps/repaints
  every few seconds (ideally over a throttled/remote connection).
- Run `just install` then `just check` (ephemeral workspace may have stale deps); include the ACE PNG snapshot suite
  since row rendering is touched.

## Risks & mitigations

- **Banner/fold staleness after narrowing the gate** — mitigated by choosing the structural signature to be a superset
  of every field banners/fold-counts depend on; covered by the banner/fold assertions above. If a banner/fold input
  beyond `(status, hidden, is_workflow_child, linkage)` is discovered during implementation, add it to the structural
  signature.
- **`get_agent_diff` contract change (Change B)** — keep the persisted-diff and clean-tree paths returning exactly as
  today; only the timeout/error branch changes meaning. Ensure the unknown result is not cached.
- **Rust core boundary** — the VCS `diff_with_untracked` provider call lives behind the existing binding; this change is
  confined to the TUI-side _classification policy_ ("timeout ⇒ unknown pencil"), which is presentation glue and stays in
  this repo. Flag at review if the reviewer considers the timeout policy a core concern.

## Follow-ups (out of scope, note during review)

- The ChangeSpec tab has a parallel incremental-refresh path with its own fallback gate; check whether it has the same
  over-broad escalation and file a follow-up if so.
- Minor inefficiency: the full-rebuild path passes `now=None` to `build_list`, so ticking rows bypass the render cache
  on each rebuild (currently masked by the per-second `patch_row` cache invalidation). Not the flicker; worth a cleanup
  once the rebuilds are rare.
