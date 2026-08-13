---
tier: tale
title: Resolve artifact-ref completion catalogs from the target project's workspace
goal:
  Prompt-bar artifact-reference completion (and highlighting) offers every configured document-sidecar role of the
  project targeted by the prompt's VCS tag, including custom roles, regardless of which session context owns the prompt
  bar.
create_time: 2026-07-29 16:44:35
status: done
---

- **PROMPT:** [prompts/202607/artifact_ref_completion_target_project_workspace.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/artifact_ref_completion_target_project_workspace.md)
- **AGENTS:**
  - [bbugyi200.athena.or--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.or.md#member-code)
  - [bbugyi200.athena.or--plan](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.or.md#member-plan)
- **COMMITS:**
  - [03739dc](https://github.com/sase-org/sase/commit/03739dcecdc357d73dba1e83c3edce0b4309a58d) — fix(tui): load artifact catalog from target project

# Artifact-Reference Completion: Resolve the Catalog from the Target Project's Workspace

## Problem

Typing `@res` in the `sase ace` prompt input widget offers no completion, even though `research` is a configured
document-sidecar role for the prompt's target project (the prompt begins with a `#gh:<project>` VCS workflow tag).
`@plans` completion works in the same prompt. The docs already promise the correct behavior — `docs/ace.md` (the
"Artifact-reference completion" bullet, near line 3058) says kind completion filters "the four builtin kinds plus
document roles configured for the prompt's target project" — but the implementation only delivers it for prompt bars
whose session context belongs to that project.

The same defect also degrades artifact-reference highlighting: a well-formed `@research:x.md` reference in a home-mode
prompt bar paints with the `artifact_ref.unknown` style because the warmed known-kind set is missing the custom role.

## Root Cause

`_load_known_artifact_ref_kinds` in `src/sase/ace/tui/widgets/_artifact_ref_highlight.py` (lines 52–86) selects the
workspace used to build the artifact-reference context with this precedence:

1. the caller-supplied `workspace_dir` (taken from the ace app's `_prompt_context` by
   `_warm_current_artifact_ref_known_kinds`, lines 220–257 of the same file);
2. `known_project_namespaces().get(project)` (from `sase.xprompt.project_identity`) only when `workspace_dir` is empty;
3. `Path.cwd()` when there is no project.

The `project` argument is the _target_ project derived from the prompt text by `_xprompt_arg_assist_project_from_text()`
(`src/sase/ace/tui/widgets/_xprompt_arg_hints.py:214`): a leading VCS workflow tag such as `#gh:sase` wins over the
session's own project. But the `workspace_dir` always describes the _session_, never the target:

- Project-mode prompt bars mount with `workspace_dir=""` because workspace allocation is deferred to spawn time
  (`src/sase/ace/tui/actions/agent_workflow/_prompt_bar_mount.py:106-107`).
- Every home-mode-style prompt context sets `workspace_dir=str(Path.home())`: `_prompt_bar_mount.py:323`,
  `_entry_relaunch.py:404`, `_entry_prompt_history.py:82`, and `_mentor_review.py:206` (all under
  `src/sase/ace/tui/actions/agent_workflow/`).

So in a home-mode prompt bar with the text `#gh:sase @res`, the warm worker builds
`artifact_ref_context(Path.home(), 1, project="sase")`. `resolve_sdd_store(Path.home(), 1)` finds no `sdd-store.json`
record for the home directory, falls back to local storage, and `split_sidecar_roles()`
(`src/sase/sdd/_store_types.py:168`) then returns only `("plans",)`. The resulting `known_kinds` is the four builtins
plus `plans` — every custom document-sidecar role configured for the target project is dropped. `@plans` still completes
(the `plans` role is unconditional), which is exactly the reported asymmetry.

The wrong catalog is then cached under the target project's key for the app's lifetime
(`_warm_current_artifact_ref_known_kinds` skips re-warming once
`project in self._artifact_ref_completion_catalogs_by_project`), so the menu never recovers.

Confirmed empirically: `artifact_ref_context(<home dir>, 1, project=<project>)` yields kinds
`(commit, chat, bug, file, plans)`, while `artifact_ref_context(<project's registered workspace>, 1, project=<project>)`
yields the same plus every configured document-sidecar role, and `load_artifact_ref_completion_catalog` then returns
that role's documents (payload rows) as expected. The downstream pipeline (kind candidates, payload candidates,
highlighting, launch-time expansion) is already role-agnostic and correct; only the workspace selection is wrong.

## Fix

Change the workspace-selection precedence in `_load_known_artifact_ref_kinds` so the catalog cached under a project key
is always built from that project's workspace:

1. When `project` is not `None` and `known_project_namespaces()` has an entry for it, use that workspace and workspace
   number `1` (the map points at the project's registered primary workspace; the caller's `workspace_num` describes the
   session, not the target).
2. Otherwise, when the caller's `workspace_dir` is non-empty, use it with the caller's effective workspace number
   (current `workspace_num if workspace_num > 0 else 1` behavior is preserved by the callers).
3. Otherwise fall back to `Path.cwd()` with workspace number `1`.

This is a reorder of the existing branches, not new machinery. Consequences by scenario:

- Home-mode bar + `#gh:<project>` tag (the reported bug): now resolves the target project's registered workspace, so
  custom document-sidecar roles appear in kind completion, payload rows list that role's documents, and highlighting
  recognizes the kind. **Fixed.**
- Project-mode bar (with or without a tag naming the session's own project): `workspace_dir` is `""` today, so these
  bars already resolve through `known_project_namespaces()`. **Unchanged.**
- Home-mode bar with no tag: `project` is `None`, so the home `workspace_dir` is still used and the catalog keeps the
  builtins-plus-global-`plans` shape. **Unchanged.**
- Tag naming a project with no `known_project_namespaces()` entry (disabled or unregistered): previously produced the
  builtins-only fallback catalog; now falls back to the session workspace (then cwd), which still yields builtins plus
  `plans`. Strictly more useful; no role can be invented for an unknown project either way.

The `workspace is None` early-return branch (lines 61–67) becomes unreachable and should be removed; the existing
`try/except` around `artifact_ref_context` keeps the builtins-only fallback for genuine context-build failures.

Implementation notes and constraints:

- The change is confined to the worker function that already runs via `run_worker(..., thread=True)`; no new work may be
  added to the UI/keystroke path (`sase/memory/tui_perf.md` rules 1 and 11). `known_project_namespaces()` is already
  called inside this worker today; the fix only moves it earlier in the precedence order.
- Do NOT hardcode the role name `research` anywhere in code or tests. Per the `sase-av` epic plan
  (`@plan:202607/artifact_refs_and_prompt_bar.md`), document-role kinds are dynamic and tests must use a fixture role
  such as `designs`. The existing suites already follow this convention.
- This is presentation-layer wiring (which workspace feeds an existing Python context builder), so it stays in this
  repo; no `sase-core` (Rust) changes are involved. The Rust grammar/scanner is untouched.
- No config, keymap, schema, or help-modal changes are needed. `docs/ace.md` already documents the intended behavior;
  verify the "Artifact-reference completion" bullet needs no wording change (it should not).

## Files to Change

- `src/sase/ace/tui/widgets/_artifact_ref_highlight.py` — reorder workspace selection in
  `_load_known_artifact_ref_kinds` as specified above; update the function docstring to state the precedence
  (target-project namespace workspace → caller workspace → cwd) and why (catalog cache is keyed by target project).
- `tests/ace/tui/widgets/test_prompt_artifact_ref_highlight.py` (or a sibling module if it fits better) — unit coverage
  for the new precedence.
- `tests/ace/tui/widgets/test_artifact_ref_completion.py` — widget-level regression coverage for a tag-targeted prompt.

## Tests

Unit tests for `_load_known_artifact_ref_kinds` (monkeypatch `known_project_namespaces` and `artifact_ref_context`
within the `_artifact_ref_highlight` module to capture the `(workspace, workspace_num, project)` arguments and return a
stub context whose `document_roots` include a `designs` role; monkeypatch `load_artifact_ref_completion_catalog` to a
pure stub so no disk is touched):

1. **Target project wins over caller workspace**: with `known_project_namespaces()` mapping `proj` to a distinct path,
   calling `_load_known_artifact_ref_kinds("proj", "<some home dir>", 1)` must build the context from the mapped path
   (workspace number 1) and the result's kinds must include `designs`.
2. **No project keeps caller workspace**: with `project=None` and a non-empty `workspace_dir`, the context must be built
   from `workspace_dir` (current behavior preserved).
3. **Unknown project falls back to caller workspace**: with an empty namespace map, `project="proj"` and a non-empty
   `workspace_dir` must build from `workspace_dir` rather than returning the builtins-only result.

Widget-level regression test in `test_artifact_ref_completion.py`, following the existing seeded-catalog pattern
(`_seed_catalog` keys the warm caches by project): seed the known-kinds and catalog caches under the project key that
`_xprompt_arg_assist_project_from_text()` derives for a prompt beginning with a VCS tag (derive the key in the test via
the same helper, or patch `canonical_xprompt_project` for determinism), load text like `#gh:proj @des` with the cursor
at the end, trigger completion, and assert the kind menu offers `designs` with insertion `@designs:`. This pins the
end-to-end keying path (tag → target project → per-project warm cache) that had no coverage.

Also verify existing suites still pass unchanged:

- `tests/ace/tui/widgets/test_artifact_ref_completion.py`
- `tests/ace/tui/widgets/test_prompt_artifact_ref_highlight.py`
- the ACE PNG visual snapshot suite (`just test-visual`), since `_ace_prompt_png_snapshot_helpers.py` stubs
  `_load_known_artifact_ref_kinds` — confirm the stub's signature still matches.

## Acceptance

- In a home-mode ace prompt bar, typing `#gh:<project> @<two letters of a custom document-sidecar role>` opens the
  artifact-kind menu including that role; accepting inserts `@<role>:` and immediately lists that role's documents.
- `@plans` completion, no-tag home-mode behavior, and project-mode behavior are unchanged.
- A well-formed reference using a custom document-sidecar role in a tag-targeted home-mode prompt highlights with the
  normal kind styles rather than `artifact_ref.unknown`.
- `just check` passes.
