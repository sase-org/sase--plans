---
tier: tale
title: Stop ACE crashing on stale artifact-read file hints
goal:
  Selecting a `research:<file>` (or any) hint in the Agents tab never crashes ACE; a
  hint whose recorded path was cleaned up is re-resolved from its durable artifact
  reference, and an unrecoverable one degrades to a warning toast.
size: medium
proposed_by: bbugyi200.athena.0f1
create_time: 2026-08-27 14:20:40
status: wip
---

# Plan: Stop ACE crashing on stale artifact-read file hints

## Symptom

In the Agents tab, pressing `v` and selecting the hint for an `ARTIFACTS · Reads` row
whose ref is `research:202608/obsidian_vault_git_sync/obsidian_vault_git_sync.md` killed
the whole TUI with an uncaught

```
FileNotFoundError: [Errno 2] No such file or directory:
'<workspace>/sase/repos/research/202608/obsidian_vault_git_sync/obsidian_vault_git_sync.md'
```

## Root cause

Two independent defects compose into a crash.

### 1. The hint points at a path that only ever existed inside one ephemeral workspace

`append_agent_artifact_read_rows` maps the hint number straight to the audit row's
recorded path:

- `src/sase/ace/tui/widgets/prompt_panel/_agent_artifact_reads.py:51` —
  `hint_state.hint_mappings[hint_number] = event.resolved_path`

That value is written by the read CLI at read time (`src/sase/artifact_cli/read.py:90` →
`src/sase/artifact_read_log.py:108`), and for a sidecar-backed kind such as `research:`
it resolves through the reading agent's _own_ workspace clone of that sidecar:
`<workspace>/sase/repos/<role>/...`. For a sidecar without `auto_clone`, that clone is
created on demand by `sase repo open` and removed when the workspace is released, so the
recorded absolute path decays while the artifact itself is untouched.

Evidence from the crash (project `bob-cli`, workspace 11):

- `repo_opens.jsonl` records `sase repo open research` creating
  `.../bob-cli_11/sase/repos/research` at `2026-08-27T16:26:43Z`.
- `artifact_reads.jsonl` records the read 15 minutes later at `16:41:14Z` with
  `resolved_path` under that same clone — so the read genuinely succeeded then
  (`_prepare_body` reads the file before the audit row is appended).
- That workspace's `sase/repos/` now holds only `beads`, `external`, `plans`; the
  `research` clone is gone. A sibling workspace's `research` clone appeared and then
  vanished again within minutes while this was being investigated — these clones are
  transient by design.
- The same document is live in the project's primary checkout at
  `~/projects/github/bobs-org/bob-cli/sase/repos/research/202608/...`, and the ref still
  resolves `exact` there today. The _reference_ never went stale — only the recorded
  path did.

The durable identity is `event.ref` (`research:202608/...`), which the display event
already carries and the hint layer currently throws away.

### 2. Nothing on the `v` view path tolerates a missing file, and the failure is fatal

The submit path never stats a hinted file:

- `src/sase/ace/hints.py:146` `parse_view_input` expands hint numbers to paths only.
- `src/sase/ace/tui/actions/hints/_processing.py:357` awaits
  `asyncio.to_thread(build_pager_document, files, ...)`.
- `src/sase/ace/tui/actions/hints/_files.py:51` `build_pager_document` →
  `src/sase/pager/adapters.py:43` `path_section` → `Path.read_text(...)`, which raises.

`_finish_view_request` runs as a Textual worker (`_processing.py:199`,
`run_worker(..., group="hint-view-request")`). Textual's `exit_on_error` default is
`True`, so the `FileNotFoundError` is escalated to a fatal app error and dumps the
traceback seen above — the panel's `errors='replace'`, `encoding='utf-8'` locals match
`path_section` exactly.

By contrast the `sase pager` CLI is safe: it resolves through
`src/sase/pager/resolve.py:154` `_resolve_file_path_target`, whose `path.exists()` guard
turns a missing file into a clean `Error: could not resolve ...` (exit 2).

Note the exposure is not specific to artifact reads: every hint producer
(`_file_path_hints.py:284`, `_agent_context.py:125`, `_agent_memory_reads.py:73`,
`_agent_display_clan_sections.py:353`, ...) stores a raw path with no existence check,
so any hint whose file is deleted between render and selection crashes ACE the same way.

## Design decisions

- **Guard at the ACE view path, not inside `path_section`.** `path_section` stays strict
  because its other caller (`pager/resolve.py`) already pre-checks existence and relies
  on a real read; the ACE hint path is the one that hands over unvalidated paths.
- **Recover by reference, not by path search.** Only re-resolve refs we actually have.
  Artifact-read rows get recovery; other hint kinds get the crash guard only.
- **Re-resolve in the _reading agent's_ project, never ACE's.** `research:` is
  project-scoped and this repo has its own `research` sidecar; resolving a `bob-cli` ref
  in ACE's own context would silently open the wrong project's document. Anchor the
  context on `event.cwd`. Derive the project from that workspace (pass `project=None` to
  `artifact_ref_context`, which falls back to `_workspace_project_ref`) rather than
  `event.project`: the audit row stores the display name (`bob-cli`) while repo
  inventory is keyed by project key (`gh_bobs-org__bob-cli`).
- **Fall back to the primary checkout, which is the durable anchor.** A document kind
  resolves against exactly one root per workspace — `SddStore.kind_root(role)` =
  `<workspace>/sase/repos/<role>` (`sdd/_store_resolution.py:88`,
  `_linked_repo_paths.py:59`); unlike `plans`, a custom sidecar gets no second root, and
  the repository fallbacks in `artifact_ref_context._repository_checkout_paths` only
  feed `vcs_backed` _file_ refs. So re-resolving in the agent's own ephemeral workspace
  fails exactly when the clone was cleaned up. The primary checkout keeps its clone, so
  recovery must try the primary workspace next. Measured against the crashing ref:

  | context                                      | research root | status             |
  | -------------------------------------------- | ------------- | ------------------ |
  | `bob-cli_11` (agent's workspace)             | absent        | `missing`          |
  | primary `~/projects/github/bobs-org/bob-cli` | present       | `exact`, live file |

  `get_primary_workspace_dir(str(workspace), workspace_num)` (`sase/sdd/files.py:90`)
  returns that primary from the recorded `cwd`, verified for this exact case.

- **Recovery is lazy and off-thread.** `artifact_ref_context` measured **273 ms** for
  this project (repo inventory + project records); `resolve_cli_reference` against a
  built context is ~1 ms. It runs only inside the existing `asyncio.to_thread` step of
  `_finish_view_request`, and only for a selected path that is actually missing — never
  on a render or keystroke path (tui_perf rules 1, 8, 11). The common case costs one
  `os.path.exists` per selected file.

## Implementation

### Step 1 — carry the durable ref alongside the artifact-read hint

1. Add a small frozen spec next to the read display models in
   `src/sase/ace/tui/artifact_reads.py` (keeps widget code from importing
   `artifact_cli`):

   ```python
   @dataclass(frozen=True)
   class ArtifactReadRefSpec:
       ref: str
       cwd: str | None
   ```

2. `src/sase/ace/tui/widgets/prompt_panel/_agent_display_state.py`: add
   `artifact_read_refs: dict[str, ArtifactReadRefSpec] = field(default_factory=dict)` to
   both `HeaderHintState` and `AgentHintRender`, mirroring `glossary_reports` /
   `memory_reports` (keyed by the recorded path, defaulted so the many positional
   `HeaderHintState(1, {}, None, {})` test constructions keep working).

3. `src/sase/ace/tui/widgets/prompt_panel/_agent_artifact_reads.py`: where the hint
   mapping is written, also record
   `hint_state.artifact_read_refs[event.resolved_path] = ArtifactReadRefSpec(ref=event.ref, cwd=event.cwd)`.
   This is pure dict assignment — no I/O is added to the render path.

4. `src/sase/ace/tui/widgets/prompt_panel/_agent_display_hint_render.py`: pass
   `artifact_read_refs=<hint_state>.artifact_read_refs` at the `AgentHintRender(...)`
   returns that already forward `commit_views`/`glossary_reports`/`memory_reports`
   (currently the header render near line 643 and the clan render near line 699). The
   early empty-render returns keep the default. Both paths render the ARTIFACTS lane
   through the single `append_agent_artifacts_lane` call site (`_agent_context.py:161`),
   so no other renderer needs touching.

5. `src/sase/ace/tui/actions/hints/_files.py`: add `self._hint_artifact_read_refs` next
   to the other `_hint_*` dicts — reset it wherever `_hint_memory_reports` is reset
   (`_action_view_files_impl`, `_view_agent_files_impl`) and publish it from
   `_render_agent_hint_document` alongside `hint_render.memory_reports`.

### Step 2 — resolve view files off-thread, repairing and filtering

In `src/sase/ace/tui/actions/hints/_processing.py`:

1. Snapshot the refs into `_PreparedViewRequest` in `_prepare_view_input`, the same way
   `report_items` is snapshotted: build
   `ref_items = tuple((f, refs[f]) for f in request.files if f in refs)` from
   `getattr(self, "_hint_artifact_read_refs", {})`. Capturing on the UI thread keeps the
   request valid across later navigation.

2. Replace the `_write_selected_hint_reports` call with one off-thread function that
   materializes reports, then repairs and partitions the result — extend
   `_MaterializedReports` with `missing_paths: tuple[str, ...]`:
   - materialize reports exactly as today (report files must be written before any
     existence check, since that step creates them);
   - for each surviving path, `os.path.exists` → keep;
   - if it does not exist and a spec is known, attempt recovery (Step 3) and keep the
     recovered path;
   - otherwise drop it into `missing_paths`.

3. In `_finish_view_request`, after the existing `failed_paths` notifications, notify
   once per missing path, e.g.
   `self.notify(f"File no longer exists: {path}", severity="warning")`, and continue
   with the survivors. The existing "No selected files could be opened" warning already
   covers the empty case. Because the filter runs before routing, no stale path reaches
   `build_pager_document`, the artifact viewer, `$EDITOR`, or the clipboard.

4. Belt and braces for the check→read race (or any other unreadable file): wrap the
   `await asyncio.to_thread(build_pager_document, files, request.commit_specs)` call in
   `try/except OSError` and notify (`f"Could not open pager: {exc}"`,
   `severity="error"`) instead of letting the worker die. A file that vanishes
   mid-flight must degrade to a toast, never a fatal worker error.

### Step 3 — reference recovery helper

Add a small module-level helper (new file
`src/sase/ace/tui/actions/hints/_artifact_ref_repair.py`, or a private function in
`_processing.py` if it stays under ~40 lines) called only from the off-thread step:

```python
def repair_artifact_read_path(spec: ArtifactReadRefSpec) -> str | None:
    """Return a live path for a recorded artifact read, or None."""
    if not spec.cwd:
        return None
    try:
        workspace, workspace_num = workspace_context_for_plan_resolution(spec.cwd)
        primary = get_primary_workspace_dir(str(workspace), workspace_num)
        anchors = [(workspace, workspace_num)]
        if Path(primary) != Path(workspace):
            anchors.append((Path(primary), 1))
        for anchor, num in anchors:
            context = artifact_ref_context(anchor, num)
            result = resolve_cli_reference(spec.ref, context=context)
            if result.resolution.status not in {"exact", "drifted", "vcs_backed"}:
                continue
            path = resolved_file_path(result)
            if path is not None and path.is_file():
                return str(path)
    except (ImportError, OSError, RuntimeError, ValueError):
        return None
    return None
```

Try the agent's own workspace first — it is correct when the clone survived and it also
picks up a document that only exists there — then the primary checkout, which is the
anchor that actually recovers the crashing ref. Imports are local to the function so ACE
startup does not pay for the artifact-CLI import graph, and every failure mode
(unresolvable project key, deleted workspace, malformed ref) returns `None` so the
caller just reports the file as missing.

## Tests

- `tests/ace/tui/actions/` (extend `test_view_files_pager.py` or add
  `test_view_files_missing_paths.py`):
  - selecting a hint whose file was deleted notifies with a warning and does **not**
    raise, and `push_screen` is never called;
  - selecting two hints where one file is missing still opens the pager with the
    surviving file only;
  - a stubbed `build_pager_document` raising `OSError` is reported through `notify`
    rather than propagating out of `_finish_view_request`.
- New test for reference recovery: a prepared request whose path is missing but whose
  `ArtifactReadRefSpec` resolves opens the pager on the recovered path; when recovery
  returns `None`, the path is reported missing instead. Monkeypatch the
  `artifact_ref_context` / `resolve_cli_reference` seams inside
  `repair_artifact_read_path` so the test stays hermetic and fast (a real context build
  costs ~273 ms), and assert the primary-workspace fallback is attempted when the
  workspace anchor resolves `missing`.
- `tests/ace/tui/widgets/test_agent_artifact_reads.py`: rendering a read row with a
  `hint_state` records the ref spec keyed by the row's `resolved_path`, and rows with no
  `resolved_path` add neither a hint nor a spec.
- `tests/ace/tui/actions/_view_files_helpers.py`: give `_ViewApp` the new
  `_hint_artifact_read_refs = {}` attribute so existing routing tests keep passing.

## Verification

- `just install` if this workspace's env is stale, then `just check`.
- If the scoped test lane escalates or reports unusual selection, run `just check-full`
  through the `/sase_monitor` skill with the `TESTING` / `TESTED` status pair.
- Manual smoke (optional): `sase ace`, Agents tab, `v` on an agent with an
  `ARTIFACTS · Reads` row whose sidecar clone has been cleaned up — expect the document
  to open (recovered) or a warning toast, never a crash.

## Non-goals / follow-ups

- Do **not** change what `sase artifact read` records. `resolved_path` is accurate at
  read time and is useful audit data; the fix treats it as a hint, not a promise.
- Do not change sidecar clone lifetime or workspace-release cleanup.
- Reference recovery is added for artifact-read hints only. Memory/glossary/plan/inline
  file-path hints get the crash guard but no recovery; if that proves worth doing, file
  a follow-up task bead rather than widening this tale.
