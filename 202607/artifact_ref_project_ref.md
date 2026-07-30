---
tier: tale
title: Repair workspace project-ref resolution and land epic sase-b2
goal:
  Workspace-derived artifact-reference contexts populate their bead stores, agent roots, and repositories, so `@bead:`
  and `@agent:` resolve through prompt expansion, the artifact CLI, and ACE on a machine with several registered
  projects — after which epic sase-b2 is closed, its symvision whitelist retired, and its plan file marked done.
bead: sase-b2
create_time: 2026-07-29 23:55:36
status: done
---

- **PROMPT:** [202607/prompts/artifact_ref_project_ref.md](prompts/artifact_ref_project_ref.md)
- **PARENT:**
  [202607/bead_and_agent_artifact_refs.md](https://github.com/sase-org/sase--plans/blob/main/202607/bead_and_agent_artifact_refs.md)
- **BEAD:** [sase-b2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-b2/README.md)

# Repair workspace project-ref resolution so `@bead:`/`@agent:` actually resolve, then land epic sase-b2

## Context

Epic `sase-b2` ("Add `@bead:` and `@agent:` artifact reference kinds") has all nine phases closed. Verification of the
landed code confirms the Rust side (`sase-core` `c1ae5f5`, `858d24c`, `aaa4e05`, released as `0.12.17`) and the Python,
ACE, docs, and pin phases are implemented as specified, and 167 focused tests pass.

But the feature is dead on any machine with more than one registered SASE project, including this one. The epic cannot
be closed until this is fixed.

## The defect

`_workspace_project_ref()` in `src/sase/artifact_ref_context.py:178-188` ends with:

```python
    marker = found[1]
    return marker.project_key or marker.project_name or None
```

For a GitHub-provider workspace the marker's `project_key` is the **provider slug**, not a SASE project reference.
Measured in workspace `sase_16`:

```
project_key:  'sase-org/sase'      <- returned, understood by nothing downstream
project_name: 'sase'               <- ignored, and is the value that works
```

Downstream consumers accept the ProjectSpec key (`gh_sase-org__sase`), the project name (`sase`), or an alias. The slug
is none of those.

### Consequence 1 — `@bead:` and `@agent:` never resolve

`collect_entity_context()` (`src/sase/artifact_ref_entity_context.py:33-47`) calls
`_project_for_ref("sase-org/sase", projects)`, which matches nothing and returns `None`. The
`if selected is None and len(projects) == 1` fallback on lines 34-35 does not fire when several projects are registered,
so `bead_stores` and `agent_roots` are returned empty.

Rust `resolve_bead`/`resolve_agent` short-circuit to `missing` on an empty store list
(`crates/sase_core/src/artifact_ref/mod.rs:611`, `:677`), so every reference fails — with an actively misleading hint:

```
$ .venv/bin/sase artifact path 'bead:sase-b2'
Error: artifact reference 'bead:sase-b2' resolved with status 'missing'
hint: no published page for sase-b2; run `sase bead page refresh`
```

The page exists at `sase/repos/beads/pages/sase-b2/README.md`.

Worse, `validate_artifact_references` hard-fails the launch, so any prompt carrying a bead or agent reference
terminates:

```
❌ ERROR: The following artifact reference(s) could not be resolved:
  - @bead:sase-b2 (missing: hint: no published page for sase-b2; run `sase bead page refresh`)
  - @agent:sase-b2.9 (missing: hint: no published page for sase-b2.9; run `sase agent sync`)
⚠️ Artifact reference validation failed. Terminating workflow.
```

### Consequence 2 — `commit:` references degraded (pre-existing)

The same ref feeds `collect_repo_inventory(project=project_filter)` at `src/sase/artifact_ref_context.py:57`, which
raises `RepoInventoryProjectNotFoundError: project 'sase-org/sase' was not found`. The bare `except Exception` on line
58 swallows it and leaves `repositories` empty, so `commit:sase@<sha>` resolves `unknown_repo`. This predates the epic
(`9988b6161`, same day) but shares the one root cause and should be fixed with it.

### Proof the rest of the epic is sound

Passing the project explicitly makes everything work:

```python
ctx = artifact_ref_context(cwd, 16, project="sase")
# bead_stores: (ArtifactRefBeadStore(project='sase', prefix='sase', root=.../sase/repos/beads),)
# agent_roots: (ArtifactRefAgentRoot(project='sase', root=~/.sase/projects/gh_sase-org__sase/repos/agents),)
resolve_artifact_ref("bead:sase-b2", context=ctx)
# status='exact', resolved_path='.../sase/repos/beads/pages/sase-b2/README.md'
```

### Blast radius

Broken (workspace-derived context, `project=None`):

- prompt expansion at launch — `launch_artifact_ref_context()`
- `sase artifact show/path/open` — `src/sase/artifact_cli/references.py:73`
- ACE `@` menu and prompt highlighting whenever no target-project namespace is selected
  (`src/sase/ace/tui/widgets/_artifact_ref_highlight.py:77`)

Unaffected: the LSP catalog (`artifact_ref_lsp_catalog_payload`) iterates project records and passes each project
explicitly; all three projects report correct `bead_stores`/`agent_roots`.

### Why the test suite missed it

`tests/test_artifact_refs.py:178-184` monkeypatches `collect_entity_context` away for the context test, and the direct
`collect_entity_context` test at `:250-268` passes a **matching alias** with exactly one project — which would also have
been rescued by the single-project fallback. No test drives the real `_workspace_project_ref` → `_project_for_ref` path,
and none registers more than one project.

## Work

### 1. Return a resolvable project reference

In `_workspace_project_ref()` (`src/sase/artifact_ref_context.py:178-188`), prefer `marker.project_name` — the value
that downstream resolvers understand — and fall back to `marker.project_key` only when the name is absent. Keep the
return type and the `except Exception` guard as they are.

### 2. Make an unmatched ref non-silent

Two independent silent failures let a bad ref look like an empty-but-valid environment. Close both:

- `collect_entity_context()` (`src/sase/artifact_ref_entity_context.py:33-47`) — when `project_ref` is non-`None` but
  matches no project, the current code silently returns empty tuples. Additionally match `project_ref` against each
  project's provider slug so a slug-form ref resolves instead of vanishing, and keep the existing behavior when the ref
  is genuinely unknown.
- `src/sase/artifact_ref_context.py:56-59` — the bare `except Exception` around `collect_repo_inventory` hides
  `RepoInventoryProjectNotFoundError`. Narrow it or log at debug so a mis-derived project ref is diagnosable rather than
  invisible. Do not make context construction raise; best-effort remains correct here.

### 3. Regression coverage

Add tests that would have caught this. The key property: **more than one registered project**, so the
`len(projects) == 1` fallback cannot mask the bug.

- `collect_entity_context` with three `ArtifactRefProject` records and a slug-form `project_ref` populates `bead_stores`
  and `agent_roots` for the right project.
- `_workspace_project_ref` returns the project name for a marker carrying both a slug `project_key` and a
  `project_name`.
- An end-to-end test: build a context from a workspace with several registered projects and assert a `bead:` reference
  whose page exists on disk resolves `exact` with the expected `resolved_path` (and correspondingly for `agent:`).

### 4. Verify against the real environment

Beyond the suite, confirm in workspace `sase_16`:

```bash
.venv/bin/sase artifact path 'bead:sase-b2'          # prints the real page path
.venv/bin/sase artifact path 'agent:sase-b2.9'       # resolves or reports a truthful status
.venv/bin/python -c "from sase.artifact_refs import launch_artifact_ref_context as f; c=f(is_home_mode=False); print(c.bead_stores, c.agent_roots, len(c.repositories))"
```

`agent:sase-b2.9` may legitimately still be `missing` if that agent's page is not published locally — what must change
is that `bead_stores`/`agent_roots` are non-empty and the reported status reflects the real filesystem.

### 5. Full gates

Run `just install` first (ephemeral workspace), then `just check`. Pre-existing SDD plan-prompt link errors in the plans
sidecar are a known blocker several sase-b2 phases already recorded; anything else must be green.

### 6. Land epic sase-b2

Only after 1-5 pass:

1. `sase bead close sase-b2 --note "<what was verified and fixed>"`
2. Run `just symvision`. The three `sase-b2` epic-symbol whitelist entries at `Justfile:273-275` expire on close.
   Confirmed pre-check of what it will report:

   ```
   Unused public functions/classes...
     collect_agent_roots in src/sase/artifact_ref_entity_context.py
     collect_bead_stores in src/sase/artifact_ref_entity_context.py
     local_agent_owner in src/sase/artifact_ref_entity_context.py
   ```

   All three are called only by `collect_entity_context` in the same module. Make them private (`_`-prefixed), drop them
   from that module's `__all__`, update the two references in `tests/test_artifact_refs.py`, and delete the three
   `--epic-symbol` lines from `Justfile:273-275`. Re-run `just symvision` to confirm it is clean.

3. Set `status: done` in the frontmatter of `sase/repos/plans/202607/bead_and_agent_artifact_refs.md`.

## Out of scope

- Widening bead/agent resolution beyond the context's own project. The epic plan names this an explicit non-goal and a
  clean follow-up.
- Any change to the Rust `sase-core` crate. The Rust side is correct; the defect is entirely in the Python context
  construction.
