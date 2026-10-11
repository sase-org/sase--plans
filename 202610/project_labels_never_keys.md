---
tier: tale
title: Show projects by PROJECT_NAME, never by directory key
goal:
  Long-lived SASE processes always show a project's PROJECT_NAME, never its directory
  key or .sase filename, even when another process created or renamed the project, with
  behavioral and static guards that fail on any reintroduced leak.
size: medium
decisions:
  admin_identity_fields:
    ask:
      Should `sase project list`/`show` keep the directory key and spec path as labeled
      identity fields?
    choices:
      keep:
        Keep them only as labeled identity fields; never use them as a project's name
      hide: Drop them from human project list/show output too; JSON keeps them
    default: keep
    why:
      Only place users learn which directory/spec backs a project; labeled as identity
    answer: keep
  decision_strand:
    ask:
      Add a decisions strand recording that projects show PROJECT_NAME, never the
      directory key?
    memory:
      - decisions:project-labels-never-keys
    requested:
      Think hard about your fix, add validation to make sure that the raw project
      direction / `*.sase` filename is never shown to a user, and then add an
      appropriate decision memory web strand that records this decision.
    default: true
    answer: true
proposed_by: bbugyi200.apollo.6t
decided_by: reviewer
decided_via: tui
create_time: 2026-10-11 13:07:34
status: wip
---

# Show Projects By PROJECT_NAME, Never By Directory Key

## Problem and evidence

The Agents tab shows `gh_bbugyi200__job` for a project whose ProjectSpec
(`~/.sase/projects/gh_bbugyi200__job/gh_bbugyi200__job.sase`) declares
`PROJECT_NAME: job`. The raw key appears in the clan row
(`gh_bbugyi200__job (RUNNING) ×7 [R5 W3] research.0w`) and in the CLAN header
(`gh_bbugyi200__job · @research · 8 agents`). The same TUI process also toasted
`Launching agent for gh_bbugyi200__job...`. Meanwhile `sase project list` correctly
prints `job (gh_bbugyi200__job)` and the status bar shows `project: +job`, because those
paths read fresh data. The project tag catalog is keyed by ProjectSpec mtimes.

Timeline from `~/.sase/logs`:

- `sase tui` (pid 411340) started at 15:34:07Z (`tui_startup.jsonl`).
- At 15:37:33Z a `#gh:bbugyi200/job` launch created the project, as shown by
  `runs.jsonl` and the ProjectSpec lock mtime. The project was created inside the
  **agent runner subprocess**. sase-github's `_resolve_repo_path_ref` calls
  `set_workspace_dir()` and then `ensure_useful_repo_name()`, which calls
  `ensure_project_name_locked`. Both invalidate only their own process's display cache.

## Root cause

`src/sase/project_display_names.py` keeps one process-wide convenience
`ProjectDisplaySnapshot` (`_get_project_display_snapshot`). Its contract (sase-89.1)
says that cache freshness never depends on filesystem metadata and that callers invoke
`invalidate_project_display_snapshot()` after a supported mutation. That contract
assumes mutations happen in the same process.

In practice, projects are created and named by other processes: the runner resolving
`#gh:` refs, `sase project alias`, `sase init`, and hand edits. The long-lived TUI
loaded its snapshot at startup, before `gh_bbugyi200__job` existed, and never reloaded
it. After that:

1. `attach_project_display_names()` (called from the agent-load worker in
   `src/sase/ace/tui/actions/agents/_loading_compute.py`) finds no entry for the key and
   leaves `Agent.project_display_name = None`.
2. `clan_project_labels()` (`src/sase/ace/tui/models/_agent_clan.py`) then falls back to
   `project_file_parent_name(member.project_file)`, which is the **directory basename**.
   That value feeds both the agent row (`_agent_list_render_agent.py`) and the CLAN
   header (`widgets/prompt_panel/_agent_display_clan_identity.py`).
3. `humanize_cl_name()`, `humanize_cl_names_in_text()`, and `project_display_name_for()`
   read the same stale map, so toasts, the compact identity header
   (`_identity_header_compact.py`), and the group-by-project labels
   (`models/agent_groups/_keys.py`, which also has a `project_file_parent_name`
   fallback) leak the key too.

The same staleness affects every other long-lived process that uses the convenience
cache: the axe lumberjack log humanizer and `sase agents wait` live rows.

## Design contract

- A project's human-facing name is `effective_project_name(record)`, meaning
  `PROJECT_NAME` when it is set. The directory key and the `*.sase` filename are storage
  identity. They are never used as a project's name. Under
  `admin_identity_fields = keep`, they may still appear as explicitly labeled identity
  fields in project-admin output and in machine-readable (JSON) output.
- **Lookups stay pure memory.** Render paths never stat, glob, or read project records
  (tui_perf rule 8; sase-89 invariant 5).
- **Long-lived processes revalidate at worker or tick boundaries.** A cheap ProjectSpec
  freshness token (one `scandir` of the projects root plus one `stat` per active spec,
  about 0.1 ms for 15 projects) gates a reload (`list_project_records`, about 0.8 ms).
  The cached snapshot is swapped only when the visible label mapping actually changes,
  so RUNNING-claim churn reloads cheaply but never invalidates render caches.
- **Labels never come from a path.** `Path(project_file).parent.name` and `.sase` stems
  are identity derivations only. The single sanctioned key-to-label fallback, for a key
  absent from the inventory such as a deleted project's history, lives in
  `ProjectDisplaySnapshot.label_for` and is reached through one agent-level chokepoint.
- No sase-core change. Labels already come from Rust lifecycle records; the freshness
  check is process-local cache glue.

## Implementation

### 1. Revalidatable display snapshot (`src/sase/project_display_names.py`)

- Store each cached snapshot together with the freshness token it was built from. The
  cache is keyed exactly as it is today, by `_projects_root_cache_key`.
- Add `_project_display_freshness_token(root)`. It scans the projects root once for
  directory membership and records `(name, mtime_ns, size)` for each project's active
  ProjectSpec filenames. Use `active_project_spec_filename` /
  `legacy_active_project_spec_filename` from `sase.ace.patch.project_spec_path`, and
  import it lazily to avoid import cycles, as `project_tags/catalog.py` already does.
  Return `None` when the root cannot be scanned, so the caller treats the snapshot as
  changed and reloads. If a shared helper with `project_tags.catalog._catalog_signature`
  or event_refresh's `_probe_patches_token` falls out naturally, reuse it, but do not
  force a refactor.
- Add a public `refresh_project_display_snapshot(projects_root=None) -> bool`. It is for
  worker and tick use only.
  - If the token is unchanged, return `False` without loading records.
  - Otherwise, load records. **If the load fails, keep the previous snapshot and token
    and return `False`** so the next call retries. Do not swap in the empty snapshot
    that `load_project_display_snapshot()` returns on error; that would revert every
    label to its key.
  - On success, store the new token. If the label `signature` changed, swap the snapshot
    **first**, then bump `_HUMANIZE_CACHE_GEN`, clear `_HUMANIZE_CACHE`, and call
    `invalidate_macro_project_identity()`. Return `True`. This ordering keeps a
    concurrent UI-thread `humanize_cl_name` from memoizing an old-map result under the
    new generation. A first population counts as a change.
- Record the token on the initial synchronous load in `_get_project_display_snapshot` as
  well. Leave `invalidate_project_display_snapshot()` (the in-process mutation path) and
  all lookup helpers unchanged.
- Rewrite the module docstring and the `_get_project_display_snapshot` docstring to
  state the new contract: lookups are memory-only, and long-lived processes call
  `refresh_project_display_snapshot()` at their refresh boundaries.
- Export the new function in `__all__`.

### 2. Wire the refresh into long-lived processes

- **TUI agent load:** call `refresh_project_display_snapshot()` once per load on the
  worker side, before the first `attach_project_display_names` of that load. Find the
  earliest worker entry so the windowed
  `agent_loader._filter_and_cap_windowed_agents_live` path and
  `ace/dismissed_agents_bundles.py` also see fresh labels.
  `_loading_compute.compute_apply_loaded_agents` is the minimum.
- **TUI auto-refresh tick:** in the existing pump-free auto-refresh body
  (`actions/event_refresh/_auto_refresh_surfaces.py`), revalidate off-thread
  (`asyncio.to_thread`) on every tick. When it returns `True`, mark the agents surface
  dirty, along with any other surface that renders project labels, so the **existing**
  refresh path reloads and re-renders. This covers a rename or a new `PROJECT_NAME` with
  no accompanying agent activity. Do not add a new refresh path or watcher (tui_perf
  rules 2, 5, and 10).
- Confirm that render caches pick up a swap. The prompt-panel caches key on
  `project_display_name_map_signature()`. The agent-row cache
  (`widgets/_agent_list_render_cache.py`) keys on `clan_project_labels` and
  `project_display_name`. Fix any cache whose key omits the visible project label.
- **axe lumberjack:** refresh once per cycle, not per `_log` line.
- **`sase agents wait` live rows** (`src/sase/agents/_wait_live_rows.py`): refresh once
  per poll iteration.
- Grep `src/sase/axe`, `src/sase/agents`, and `src/sase/integrations` for other
  long-lived loops that call `humanize_*` or `project_display_name_for` and wire each
  one. One-shot CLI commands need nothing, because their first lookup loads fresh data.

### 3. One project-label chokepoint for agents

- Add a pure `Agent.project_label` property (`src/sase/ace/tui/models/agent.py` or the
  state mixin that owns `project_display_name`). It returns `project_display_name` if
  set, and otherwise `project_display_name_for(<canonical key from project_file>)`,
  which is a memory-only lookup. Agents without a `project_file` return `None`.
- Use it in `clan_project_labels()`, which removes the `project_file_parent_name`
  fallback. Also use it in `agent_groups/_keys.py` (group-by-project label) and in the
  `_identity_header_compact.py` sites that render a **project** label. Keep
  `humanize_cl_name` where the value is a Patch label rather than a project label.
- Keep `project_file_parent_name` for identity-only uses such as
  `Agent.is_project_agent`, node-finder facet matching, and grouping keys.

### 4. Validation that the key never reaches a user

Behavioral (runtime-rendered output):

- Extend `tests/_project_display_case.py` with
  `ProjectDisplayCase.assert_no_key_leak(text)`. It fails when the rendered plain text
  contains `project_key`, `f"{project_key}.sase"`, or the project directory path.
- Add a TUI test module, for example
  `tests/ace/tui/test_agent_project_label_freshness.py`, using the mismatched
  `gh_acme__widgets` → `widgets` case:
  - **Stale at startup (the reported bug):** warm the convenience cache against a
    projects root that lacks the project. Write the layout with `write_project_layout()`
    while monkeypatching `invalidate_project_display_snapshot` to fail if it is called;
    this proves no in-process hook rescued the test. Run the worker agent-load compute,
    or `refresh_project_display_snapshot()` followed by attach. Then assert `widgets`
    and `assert_no_key_leak` for:
    - a clan-container row from `format_agent_option` with two members in the project;
    - both CLAN header renderers in `_agent_display_clan_identity.py`;
    - the group-by-project label;
    - the compact identity fallback context line;
    - the launch toast label.
  - **Rename:** change `PROJECT_NAME` on disk to `gizmos`. The refresh returns `True`,
    the humanize memo generation advances, and rows render `gizmos`.
  - **RUNNING churn:** a spec edit that changes only `RUNNING:` makes the refresh return
    `False`, and the memo generation does not change.
  - **Unchanged token:** a monkeypatched counter shows `list_project_records` is not
    called.
  - **Load failure:** the previous snapshot survives, the call returns `False`, and the
    next call retries.
- Add an auto-refresh tick test: a label change detected on a tick marks the agents
  surface dirty through the existing harness.
- Add contract tests for `refresh_project_display_snapshot` in
  `tests/test_project_display_names.py`.

Static (`tests/test_project_display_presentation_audit.py`, marked `contract`):

- **G1, no path-derived labels.** Under `src/sase/ace/tui/**`, forbid calls to
  `project_file_parent_name(...)`, and forbid `.parent.name` / `.stem` on an expression
  whose source mentions `project_file`, except at allowlisted identity-only
  `(path, function, expression)` sites. Each allowlisted site carries a rationale.
  Include a stale-allowlist check modeled on
  `test_canonical_exemptions_name_existing_sites_and_rationales`.
- **G2, single fallback.** Forbid a `BoolOp(Or)` with a `.project_display_name` operand
  anywhere except `Agent.project_label` and `src/sase/project_display_names.py`. This
  catches future `agent.project_display_name or <path/key>` fallbacks.
- **G3, refresh stays at worker boundaries.** Add `refresh_project_display_snapshot` to
  the loader names tracked by
  `test_project_metadata_loaders_stay_at_reviewed_tui_worker_boundaries`. Add the files
  that call it to `_PROJECT_METADATA_FLOW_FILES`, and allowlist exactly the reviewed
  worker and tick sites. Add it to `_FORBIDDEN_RENDER_CALLS`, and add the agent-row and
  clan-header render modules to `_PURE_TUI_RENDER_FILES` if they are already free of
  I/O. If they are not, file the pre-existing I/O as a bug through `/sase_new_task`
  instead of widening scope.
- **G4.** Add the agent-row, clan-identity, compact identity-header, and launch-toast
  modules to `_AUDITED_RENDERERS`. Classify every hit as projected or as a documented
  canonical exemption, for example option ids built from `agent.cl_name`.

> [!decision] admin_identity_fields = hide Also audit
> `src/sase/main/project_handler_render.py` and the Admin Center Projects tab.
>
> - Remove the `(key)` suffix from human `sase project list` output.
> - Remove `Directory key:` and `Project file:` from `sase project show`.
>
> Keep both fields in `--json`. Update the affected tests and the `docs/cli.md`
> examples, and add these renderers to `_AUDITED_RENDERERS` with no key exemptions.

### 5. Docs

Update the `PROJECT_NAME` bullet in `docs/project_spec.md` with two points:

- Long-running processes (`sase tui`, axe, `sase agents wait`) pick up projects created
  or renamed by other processes on their next refresh.
- The directory key and ProjectSpec filename are never shown as a project's name. They
  appear only as labeled identity fields, per the decision above.

### 6. Memory (decision `decision_strand`)

> [!decision] decision_strand

Use `/sase_memory_write` and add `sase/memory/decisions/project-labels-never-keys.md`.
Its frontmatter carries:

- `keyword: Projects Are Shown By Name, Never By Directory Key`;
- aliases such as `project display names` and `project key leak`;
- a one-line `summary`;
- `metadata: {status: accepted, decided: 2026-10-11}`.

Write the body in the Claim / Why / Cost / Reopens when shape:

- **Claim:** the contract above. A project's label is `PROJECT_NAME` through the display
  snapshot. The key and `*.sase` filename are identity only, appearing at most as
  labeled identity fields. Long-lived processes revalidate at worker or tick boundaries
  through a ProjectSpec stat token, and lookups stay memory-only. Labels never come from
  paths.
- **Why:** the `gh_bbugyi200__job` incident. sase-89's in-process-only invalidation
  missed cross-process creation and naming. Rejected alternatives:
  - in-process invalidation only (the status quo);
  - per-lookup stat freshness (render-path I/O, `[[tui_perf.md]]`);
  - a cross-process generation pulse bumped by mutation funnels (misses hand edits,
    other sase versions, and Rust-side writers);
  - deriving a label from the provider key format (`gh_owner__repo` → `repo`), which can
    disagree with the allocated `PROJECT_NAME` such as `job_1` and would belong in
    sase-core under `[[rust-core-required]]`;
  - a stale-while-revalidate daemon thread in every process.
- **Cost:** a stat sweep per tick in long-lived processes, plus a reload of about 1 ms
  whenever any ProjectSpec changes, including RUNNING churn. Every long-lived consumer
  must call the refresh. The static-guard allowlists need maintenance.
- **Reopens when:** a non-Python frontend needs labels (then move the label and
  freshness rules into sase-core), the sweep becomes measurable on the tick at high
  project counts, or spec churn makes reloads costly enough to justify a content-scoped
  token.

Then run `sase memory init` to regenerate the decisions roster and the agent instruction
files.

## Verification

- Run the targeted tests first, then `sase tool run check`.
- No PNG goldens should change, because existing fixtures do not use mismatched keys. If
  rendered TUI output does change, follow `lint_and_test.md` and run a targeted
  `just fix-tui-screenshots`.

## Acceptance criteria

- A TUI started before a project exists, or before its `PROJECT_NAME` is written by
  another process, shows the configured name in agent rows, clan rows, the CLAN header,
  group labels, identity headers, and toasts. This happens no later than the next
  auto-refresh tick, with no in-process invalidation call.
- Renames made by another process propagate the same way. RUNNING-only churn never
  clears humanize or render caches.
- A failed inventory read never reverts labels to keys.
- The new behavioral leak tests and static guards G1–G4 pass, and fail if a
  `project_file_parent_name` / `project_display_name or …` label fallback is
  reintroduced.
- Render paths gain no disk I/O. Identity, persistence, JSON, and query semantics are
  unchanged.
- If the decision is accepted, the decision strand exists and `sase memory init` has
  been run.
- `sase tool run check` passes.
