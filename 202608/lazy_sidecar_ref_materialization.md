---
tier: tale
title: Materialize lazily-cloned document sidecars on the launch ref path
goal:
  A document artifact reference whose sidecar repo is not yet cloned in the launch
  workspace materializes that sidecar and resolves, instead of failing the launch with
  an unexplained "(missing)" and holding the workspace.
size: medium
proposed_by: bbugyi200.athena.ym
create_time: 2026-08-12 11:12:09
status: wip
---

# Plan: Materialize lazily-cloned document sidecars on the launch ref path

## Problem

The `yl` agent launch died 12 seconds in with:

```
❌ ERROR: The following artifact reference(s) could not be resolved:
  - @research:202608/xprompt_role_binding/xprompt_role_binding.md (missing)

⚠️ Artifact reference validation failed. Terminating workflow.
```

The prompt was
`#gh:sase Describe the @research:202608/xprompt_role_binding/xprompt_role_binding.md file. #m_sonnet`.

The referenced file exists and has existed for a while. Resolving that exact reference
from a workspace that has the `research` sidecar cloned succeeds:

```
$ sase artifact show research:202608/xprompt_role_binding/xprompt_role_binding.md
status              exact
path                .../sase/repos/research/202608/xprompt_role_binding/xprompt_role_binding.md
```

The launch failed only because the agent was assigned a workspace that did not have the
sidecar cloned.

## Root cause

`@<kind>` document references resolve **exclusively** against the target workspace's own
sidecar clone, and nothing on the reference path ever materializes a lazily-cloned
sidecar.

The chain:

1. `artifact_ref_context()` (`src/sase/artifact_ref_context.py:32-136`) builds one
   `ArtifactRefDocumentRoot` per document sidecar role from `store.kind_root(role)` —
   i.e. `<workspace>/sase/repos/<role>`. It never checks that the directory exists and
   never materializes it. Note that the `plans` role alone also gets a second,
   machine-global root (`sase_subdir("plans")`, lines 63-70); no other document role has
   any fallback.
2. Launch-time workspace prep is the only thing that clones sidecars, and it clones only
   a fixed subset. `prepare_launch_workspace_repos()`
   (`src/sase/axe/runner_workspace.py:406-448`) first calls `clear_workspace_repos()`,
   which for `workspace_num > 1` renames the entire `sase/repos` directory into
   `.sase/trash` and deletes it in the background
   (`src/sase/_linked_repo_workspaces.py:79-109`). It then recreates only `plans` (via
   `ensure_workspace_sdd_clone`, which hard-codes the `plans` role at
   `src/sase/sdd/_store_workspace.py:48-58`) and `beads`. Linked/sidecar repo prep
   afterwards filters on `repo.auto_clone`
   (`src/sase/axe/run_agent_runner_setup.py:118`).
3. The `research` sidecar is deliberately `auto_clone: false`. That is the documented
   design — see `src/sase/sdd/assets/research-directory-map.png.prompt.md` ("Lazy clone:
   `ON DEMAND`, `Ensure clone`, `LAZY`, `auto_clone: false`") — because the clone is
   large (the `sase--research` `.git` is ~257 MB versus ~78 MB for `plans`) and most
   agents never touch it. It is materialized only by an explicit
   `sase repo path research --ensure`, which routes to `ensure_sdd_kind_clone()`
   (`src/sase/main/repo_handler_path.py:146-157`).
4. So the `research` document root points at a directory that usually does not exist in
   a launch workspace. The Rust resolver returns `missing`, and
   `_expand_artifact_references()` collects that into `failures` and calls `sys.exit(1)`
   (`src/sase/artifact_ref_prompt.py:246-248`), killing the run and pinning the
   workspace as a held failed run.

### Reproduction

Resolving the same reference against two workspaces that differ only in whether the
sidecar happens to be cloned:

```python
from sase.artifact_ref_context import artifact_ref_context
from sase.artifact_ref_kinds import parse_artifact_ref_canonical
from sase.artifact_ref_operations import resolve_artifact_ref

ref = parse_artifact_ref_canonical(
    "research:202608/xprompt_role_binding/xprompt_role_binding.md"
).reference
ctx = artifact_ref_context(workspace_dir, workspace_num, project="gh_sase-org__sase")
print(resolve_artifact_ref(ref, context=ctx).status)
```

- Workspace with `sase/repos/research` present → `exact`, correct path.
- Workspace without it → `missing`, no path, **no diagnostic**.

For the failing workspace the store record is fully intact —
`store.remote_url_for_kind("research")` returns
`git@github.com:sase-org/sase--research.git` — so the clone is materializable at that
moment. Nothing asks for it.

### Why this looks intermittent, and why it is not user error

Authoring is green and launching is red, because the two use different workspaces. ACE
and the CLI build their context from `cwd` via `build_launch_artifact_ref_context()`
(`src/sase/artifact_ref_launch_context.py`), which resolves against the host checkout —
and the host checkout _does_ have the research sidecar. The launch path deliberately
does not use `cwd`; it builds a `PromptRefContext` per prompt segment from the segment's
`#gh`/`#git` tag (`src/sase/artifact_ref_prompt_context.py`), pointing at the freshly
claimed workspace. So a reference that completes, highlights, and validates while being
typed can still be fatal at launch, decided entirely by which workspace the agent draws.

This is not specific to `@research`. It applies to any document sidecar with
`auto_clone: false`, which is the documented default for custom sidecars
(`src/sase/default_config.yml:27`).

### Relationship to the `sase-js` epic

`sase-js` (Artifact reference contract) shipped all nine phases. The epic plan
(`plans:202608/artifact_ref_contract.md`) addresses lazy sidecars only through
`$(sase repo path research --ensure)` inside the `#research*` xprompts (§ around
line 946) — that is, only for the _xprompt_ surface it was replacing. When the
`@research` ref surface took over, it inherited the resolution contract but not the
ensure step. This plan closes that gap; it is a defect in the shipped contract, not new
capability.

## Approach

Add an explicit, launch-only materialization pre-pass to artifact-reference expansion:
before resolving, for each document kind actually cited in the prompt, if that kind is
backed by a sidecar role whose root directory is absent in the target workspace, clone
it and rebuild that segment's context.

Trigger on **"the root directory does not exist"**, not on **"resolution returned
`missing`"**. That distinguishes "sidecar not cloned" from "file genuinely absent", so a
typo'd path in a workspace that already has the sidecar never triggers a 257 MB clone,
and the pre-pass costs one `is_dir()` call in the common case.

### Why not the alternatives

- **Make `research` (or any sidecar with a `ref:` block) `auto_clone: true`.** Rejected:
  it inverts a deliberate design decision and would clone ~257 MB into every one of ~24
  workspaces on prep, for a sidecar most agents never read. The epic's own asset doc
  pins `auto_clone: false` as intended behavior.
- **Add a machine-global fallback root for every document role, mirroring the
  `~/.sase/plans` root.** Rejected: a document ref expands to `@<absolute path>`
  (`src/sase/artifact_ref_prompt_rendering.py:57`), so falling back would hand the agent
  a path outside its own workspace, which breaks workspace isolation and the
  `/sase_repo` rule.
- **Ensure every lazy sidecar during workspace prep.** Rejected: pays the clone cost on
  every launch regardless of whether any ref cites it.

### Cost note (accept, do not fix here)

`ensure_sdd_kind_clone` → `ensure_sidecar_sdd_clone` clones from the recorded remote URL
(`src/sase/sdd/_store_link.py:37-82`), so the first `@research` ref in a fresh workspace
pays a full network clone on the launch's critical path. This is not a regression in
kind: sidecar prep for `plans`/`beads` already clones from the remote via
`_materialize_remote_identified_sidecar`
(`src/sase/_linked_repo_workspaces.py:206-218`), and
`$(sase repo path research --ensure)` inside the existing `#research` xprompts already
pays exactly this cost. Do **not** try to optimize the clone source here — just make
sure the launch prints a progress line so it does not look hung (see step 5), and file a
follow-up task bead for "clone sidecars from the local primary checkout instead of the
remote" via `/sase_new_task`, since that change would affect all sidecar
materialization, not just this path.

## Implementation

All paths below are repo-relative.

### 1. Carry the project key on `PromptRefContext`

`src/sase/artifact_ref_prompt_context.py`

`_safe_artifact_ref_context()` is called with `project=key or ref or "home"` (line ~97)
but that value is then discarded, so the context cannot be rebuilt after a clone. Add a
`project_ref: str | None = None` field to `PromptRefContext` and populate it in
`prompt_ref_context_for_vcs_ref()` and `prompt_ref_context_from_launch_identity()` with
exactly the value passed to `_safe_artifact_ref_context()`. Leave it `None` for
`empty_prompt_ref_context()` and `explicit_prompt_ref_context()`.

Add a module-level helper:

```python
def refresh_prompt_ref_context(context: PromptRefContext) -> PromptRefContext:
    """Rebuild one context's ArtifactRefContext after its sidecars changed."""
```

It returns the context unchanged when `workspace_dir` or `workspace_num` is `None`;
otherwise it rebuilds `artifact_context` through `_safe_artifact_ref_context()` with the
stored `project_ref` and returns a `replace()`d copy. Export it in `__all__`.

### 2. Add the materialization pre-pass

New module `src/sase/artifact_ref_prompt_materialize.py`, so the existing
`artifact_ref_prompt.py` stays a thin surface (matching the neighboring
`artifact_ref_prompt_*` split its docstring describes).

```python
def materialize_missing_document_roots(
    kinds: Sequence[str],
    context: PromptRefContext,
) -> tuple[PromptRefContext, tuple[MaterializationFailure, ...]]:
```

Behavior:

- Return immediately when `context.workspace_dir is None` or
  `context.workspace_num is None`.
- Resolve the store: `resolve_sdd_store(context.workspace_dir, context.workspace_num)`.
  Build the kind → role map by recomputing the sidecar ref policies for that workspace
  the same way `artifact_ref_context._sidecar_ref_policies()` does — via
  `effective_sidecar_ref_policies()` over
  `document_sidecar_roles(store.split_sidecar_roles(), include_plans=True)` — and
  keeping `policy.ref_kind -> policy.role` for policies where `policy.is_document`. Do
  **not** add a `role` field to `ArtifactRefDocumentRoot`: that dataclass has a
  `to_wire()` and crosses into the Rust resolver, so extending it would drag in a
  `sase-core` wire change for no benefit.
- For each requested kind present in that map whose `store.kind_root(role)` is not a
  directory: call
  `ensure_sdd_kind_clone(workspace_dir, workspace_num, role, strict=True)`. Catch
  `SddMaterializationError` (and any other `Exception`) per role and record a
  `MaterializationFailure(kind, role, remote_url, detail)` rather than raising — a
  failed clone must produce a good error message through the existing failure channel,
  not a traceback.
- If at least one role was cloned, return `refresh_prompt_ref_context(context)`;
  otherwise return the context unchanged.

Skip roles for which `store.remote_url_for_kind(role)` is `None` (nothing to clone) and
record no failure for them — that is the in-tree/local-store case, where a genuinely
missing file is the right answer.

### 3. Wire the pre-pass into expansion, launch-only

`src/sase/artifact_ref_prompt.py`

- Add a keyword-only `materialize_missing_roots: bool = False` to
  `process_artifact_references()` and to `_expand_artifact_references()`. Do **not** add
  it to `validate_artifact_references()`; validation must never clone.
- In `_expand_artifact_references()`, after `segment_contexts` is built and before the
  candidate loop, when `materialize_missing_roots` is true: group the scanned candidates
  by their segment context, collect each group's distinct well-formed kinds that are in
  that context's `known_kinds`, call `materialize_missing_document_roots()`, and replace
  that entry in `segment_contexts` with the returned context.
- Keep any returned `MaterializationFailure` and, when a later candidate of that kind
  still fails to resolve, attach the failure's detail as that failure's hint so the user
  sees why the clone did not happen rather than a bare `(missing)`.

The default of `False` is the safety property: `ArtifactRefContext` consumers that must
never trigger network I/O — the ACE completion catalog
(`src/sase/ace/tui/widgets/_artifact_ref_completion_catalog.py`), the LSP catalog
(`artifact_ref_lsp_catalog_payload`), `sase doctor`, and `sase artifact *` — keep
today's behavior with no change at their call sites.

### 4. Opt in from the agent-launch prompt step only

- `src/sase/llm_provider/preprocessing.py`: add keyword-only
  `materialize_missing_roots: bool = False` to `preprocess_prompt_late()` and thread it
  into the `process_artifact_references()` call at line ~186. `preprocess_prompt` (the
  early+late composition, line ~251) threads it through too.
- `src/sase/xprompt/workflow_executor_steps_prompt.py:249`: pass
  `materialize_missing_roots=True`. This is the agent-launch site — the one in `yl`'s
  traceback.
- `src/sase/main/xprompt_handler.py:65`: leave at the default `False`. `sase xprompt` is
  a preview surface and should not clone a 257 MB sidecar as a side effect of expanding
  a prompt.

### 5. Make both remaining failure modes legible

Even with the fix, two cases still fail, and both currently print a bare `(missing)`.

- **Clone failed** (offline, credentials, remote gone). Surface the
  `MaterializationFailure` detail through the hint, naming the role, the remote URL, and
  `sase repo path <role> --ensure`.
- **Sidecar present, file genuinely absent.** `artifact_ref_resolution_hint()`
  (`src/sase/artifact_ref_prompt_resolution.py:154-183`) returns `None` for every
  `document` kind today — only `bead` and `agent` get hints. Add a `document` branch:
  when `resolution.status == "missing"`, name the document root that was searched, so
  the message distinguishes "wrong path" from "wrong workspace".

Also print a single progress line before a clone starts (matching the existing
`=== Preparing Workspace ===`-style runner output), e.g.
`Materializing 'research' sidecar for @research references...`. A silent multi-second
clone on the launch path reads as a hang.

## Tests

Add `tests/artifact_refs/test_prompt_materialization.py`, plus the fixture pattern in
`tests/sdd_store/test_workspace_clone.py` (which already exercises
`ensure_sdd_kind_clone(workspace, 2, "research", strict=True)`) for a workspace whose
sidecar is absent but whose store record carries a remote.

1. **The regression.** A prompt citing `@<kind>:<path>` in a workspace whose sidecar
   role root is absent expands successfully: the clone is materialized once, the context
   is rebuilt, and the replacement text is the resolved path inside that workspace.
2. **No-op when present.** With the root already a directory, `ensure_sdd_kind_clone` is
   never called (assert on a patched spy), and expansion behaves exactly as today.
3. **Default is off.** `process_artifact_references()` without
   `materialize_missing_roots=True` never calls `ensure_sdd_kind_clone`.
4. **Validation never clones.** `validate_artifact_references()` never calls it, for a
   prompt with a missing-root document ref.
5. **Non-launch surfaces never clone.** Assert `main/xprompt_handler` expansion and the
   LSP catalog build leave `ensure_sdd_kind_clone` uncalled.
6. **Clone failure is actionable.** With `ensure_sdd_kind_clone` raising
   `SddMaterializationError`, expansion still exits non-zero but the printed failure
   names the role, the remote, and `sase repo path <role> --ensure` — assert on captured
   output, not just the exit code.
7. **Genuinely-missing file.** Sidecar present, path absent → failure hint names the
   searched document root and does not suggest a clone.
8. **Kind→role mapping.** `@research` maps to the `research` role and `@plan` to
   `plans`; a configured document kind whose name differs from its role
   (`ref: kind: <other>`) still maps correctly.
9. **Context rebuild.** `refresh_prompt_ref_context()` picks up a document root that
   appeared after the context was first built, and is a no-op for a context with no
   workspace.

Extend `tests/artifact_refs/test_prompt_context.py` for the new `project_ref` field
(populated for `vcs_workflow` and `launch_identity`, `None` for `explicit` and `none`).

## Verification

- `just install` first — workspaces are ephemeral and dependencies may have moved.
- `just check`. Run `just check-full` before landing: this touches shared prompt
  preprocessing that many suites import.
- End-to-end check on the real failure. Pick a workspace with no `sase/repos/research`
  (confirm with `sase repo list`) and launch an agent with a prompt citing an
  `@research:` path that exists in the sidecar. It should materialize the sidecar and
  run, instead of failing in ~12 s. `yl`'s original prompt is the exact case:
  `Describe the @research:202608/xprompt_role_binding/xprompt_role_binding.md file.`
- Confirm ACE's `@`-completion and `sase artifact show` still work with no new network
  I/O.

## Out of scope

- Changing which sidecars are `auto_clone`, or the `~/.sase/plans` fallback root.
- Optimizing the sidecar clone source (remote → local primary). File a follow-up task
  bead through `/sase_new_task`; it affects all sidecar materialization.
- The `sase-research` plugin not being installed in the per-workspace `.venv`s (only in
  the `uv` tool environment that `sase` on `PATH` resolves to). That yields
  `unknown_kind`, a different symptom with a different cause, and did not cause this
  failure — the runner uses the tool environment, where the plugin is installed. Worth a
  separate task bead if in-workspace `sase` invocations should also resolve `@research`.
- Releasing `yl`'s held workspace #16; that is a normal `sase ace` dismissal.
