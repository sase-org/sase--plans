---
create_time: 2026-07-11 19:16:09
status: done
prompt: 202607/prompts/fix_vcs_ref_alias_prefix_mangle.md
tier: tale
---
# Fix `#gh:` VCS Ref Canonicalization Mangling Display-Prefixed ChangeSpec Names

## Problem

A launch of the prompt

```
#gh:sase_fix_just_linters_14 Can you help me review this CL ... #tale #m_fable
```

fails with:

```
Cannot resolve #gh:gh_sase-org__sase_fix_just_linters_14; not launching
```

even though the ChangeSpec `sase_fix_just_linters_14` exists and is active in
`~/.sase/projects/gh_sase-org__sase/gh_sase-org__sase.sase`, and `resolve_ref("sase_fix_just_linters_14", "gh")`
resolves it successfully.

## Root Cause

Commit `4cce6a46b` ("fix: humanize project-prefixed ChangeSpec names", tale `eradicate_raw_project_keys`) made two
changes that contradict each other:

1. **New ChangeSpec names are generated from the project display name** and stored that way on disk. `_derive_cl_name()`
   / `ensure_project_prefix()` in `src/sase/workspace_provider/changespec.py` prefix new names with the _display_
   project name, so a new spec in project `gh_sase-org__sase` (display name `sase`) is canonically stored as
   `sase_fix_just_linters_14`.

2. **VCS alias canonicalization now rewrites prefixes unconditionally.** `_rewrite_ref_with_known_prefix()` in
   `src/sase/project_aliases.py` was added so that legacy display-prefixed refs round-trip back to canonical
   project-key-prefixed names: any ref `<alias>_<suffix>` whose `<alias>` is in the alias map is rewritten to
   `<canonical_project>_<suffix>`. It assumes _every_ display-prefixed ref refers to a legacy key-prefixed spec.

For any ChangeSpec created after (1), the two collide: the launch pipeline (`canonicalize_project_aliases_in_prompt()` —
called from `_launch_body_impl.py`, `_ref_resolution.py:resolve_ref_from_prompt()`, `multi_prompt_launcher.py`,
`launch_request.py`, `vcs_xprompt_mru.py`, etc.) rewrites `#gh:sase_fix_just_linters_14` to
`#gh:gh_sase-org__sase_fix_just_linters_14`. The workspace provider then parses that as project `gh_sase-org__sase` +
ChangeSpec `fix_just_linters_14`, which does not exist (the stored name is the full `sase_fix_just_linters_14`), so
every per-workflow resolution returns `None` and the explicit-VCS-tag guard in `_launch_body_single.py` aborts the
launch with the "Cannot resolve" error. Because `_launch_body_impl.py` canonicalizes the prompt text itself before
dispatch, the mangled ref also leaks into the error message, prompt history, and the prompt stash.

Reproduced directly:

- `canonicalize_project_aliases_in_prompt("#gh:sase_fix_just_linters_14 …")` →
  `"#gh:gh_sase-org__sase_fix_just_linters_14 …"`
- `resolve_ref("sase_fix_just_linters_14", "gh")` → resolves (project `gh_sase-org__sase`)
- `resolve_ref("gh_sase-org__sase_fix_just_linters_14", "gh")` → `ValueError`

This bug affects **every ChangeSpec created since commit `4cce6a46b`** whose name is referenced in a `#gh:`/`#git:`
launch tag (completion inserts these full names, so it affects the standard "review this CL" flow).

## Design

The invariant to restore: **a ref that exactly names an existing ChangeSpec must never be reinterpreted as
`<alias>_<changespec>`.** The legacy round-trip rewrite should only fire when the ref does _not_ literally name a spec
on disk.

### 1. Guard the prefix rewrite in `canonicalize_project_aliases_in_prompt()`

In `src/sase/project_aliases.py`, before committing to the prefix branch of `_rewrite_ref_with_known_prefix()` (exact
alias-map hits are unaffected):

- Determine the target project for the matched alias prefix (already known: `known_replacement`).
- Check whether a ChangeSpec whose NAME is exactly the original ref exists in that project's ProjectSpec files — both
  the active file and the archive file (archived refs appear in resume/history flows and should keep their typed form
  for a truthful error message).
- If such a spec exists, **skip the rewrite** and leave the ref as typed; the workspace provider resolves bare
  ChangeSpec names directly (verified above).
- Otherwise keep today's behavior (rewrite), preserving both the legacy round-trip (`sase_foo` → `gh_sase-org__sase_foo`
  when the legacy spec exists) and compound `<alias>_<changespec>` refs (`sase_foo` where spec `foo` lives in project
  `sase`).

Implementation notes:

- The existence check belongs in the canonicalize direction only. `humanize_project_refs_in_prompt()` (canonical →
  display, rendering-only) keeps using the unguarded helper: its prefix branch keys on canonical project keys, which
  cannot collide with display-prefixed names, and the round-trip back through the guarded canonicalizer stays correct
  for both legacy and new specs.
- Pass the guard as a predicate (or do the check inline in `replace()`), with a lazy import of the existing ProjectSpec
  parsing helpers (`sase.ace.changespec.parse_project_file` / `preferred_project_spec_path`-style path resolution),
  consistent with the other lazy imports in `project_aliases.py`.
- Cost: the check only runs when a ref contains `_` AND starts with a known alias prefix, i.e. once per launch-ish call,
  and parses at most one project's two spec files. No hot rendering path is affected. Cache per-call (the `replace`
  closure) if the same ref appears multiple times.

### 2. Repair already-mangled refs (reverse rewrite)

The bug has already minted mangled refs into prompt history, the prompt stash, and MRU records (e.g.
`#gh:gh_sase-org__sase_fix_just_linters_14`). Replaying those should work instead of failing forever. In the same
`replace()` hook, add the inverse repair: when a ref starts with a canonical project key prefix (`<project>_<suffix>`),
does **not** resolve as-is (no literal spec `<project>_<suffix>` and no spec `<suffix>` in that project), but
`<display>_<suffix>` names an existing ChangeSpec in that project — rewrite the ref to `<display>_<suffix>`. This makes
the poisoned history entries self-heal at the next launch.

### 3. Regression tests

In `tests/test_xprompt_aliases.py` (following the existing monkeypatch style) and/or
`tests/test_project_alias_services.py`:

- Display-prefixed ref naming an existing spec is **not** rewritten: alias map `{"sase": "gh_sase-org__sase"}`, spec
  `sase_fix_just_linters_14` present → `#gh:sase_fix_just_linters_14` unchanged.
- Legacy round-trip still works: same map, no literal spec → `#gh:sase_legacy_fix` → `#gh:gh_sase-org__sase_legacy_fix`.
- Archived literal spec also blocks the rewrite.
- Reverse repair: mangled `#gh:gh_sase-org__sase_fix_just_linters_14` with literal spec `sase_fix_just_linters_14`
  present → rewritten back to `#gh:sase_fix_just_linters_14`.
- Exact alias-map hits (`#gh:sase` → `#gh:gh_sase-org__sase`) and fenced-block protection are unchanged (covered by
  existing tests; keep them green).

### 4. Out of scope

- The `gh` workspace-provider plugin (`sase-github` linked repo) needs no change: bare ChangeSpec-name resolution
  already works; the fix stops the Python launch boundary from destroying the ref before the provider sees it.
- No Rust-core change: `canonicalize_project_aliases_in_prompt()` and its helper are Python-only (no counterpart under
  `src/sase/core/` or the Rust xprompt LSP parity surface, which only mirrors completion-tag expansion).
- New-ChangeSpec name generation (display-prefixed) is intentional per `4cce6a46b` and stays as is.

## Verification

- New unit tests above.
- Manual: `canonicalize_project_aliases_in_prompt("#gh:sase_fix_just_linters_14 hi")` returns the input unchanged, and a
  TUI launch of the original failing prompt resolves to project `gh_sase-org__sase` instead of erroring.
- `just check` (after `just install`).
