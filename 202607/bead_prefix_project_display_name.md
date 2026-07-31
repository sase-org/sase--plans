---
tier: tale
title: Derive bead issue prefixes from PROJECT_NAME instead of the ProjectSpec key
goal:
  New bead stores get an issue prefix built from the project's user-facing PROJECT_NAME (`bob-cli`), never the
  ProjectSpec directory key (`gh_bobs-org__bob-cli`), and `sase bead doctor` detects and can repair stores that already
  leaked a key-shaped prefix.
create_time: 2026-07-31 08:25:10
status: wip
---

- **PROMPT:** [202607/prompts/bead_prefix_project_display_name.md](prompts/bead_prefix_project_display_name.md)

# Plan: Derive bead issue prefixes from PROJECT_NAME instead of the ProjectSpec key

## Observed symptom

In the enabled `bob-cli` project (ProjectSpec key `gh_bobs-org__bob-cli`, `PROJECT_NAME: bob-cli`), the epic bead that
`sase bead work` created after an epic was approved from the TUI is named with the key, not the project name:

```
❯ pwd
/home/bryan/projects/github/bobs-org/bob-cli

❯ sase bead list
◐ gh_bobs-org__bob-cli-2 · Capture sub-bullets onto existing Obsidian tasks
◐ gh_bobs-org__bob-cli-2.3 · bob capture-tasks discovery command ← gh_bobs-org__bob-cli-2
◐ gh_bobs-org__bob-cli-2.4 · Hammerspoon task picker ← gh_bobs-org__bob-cli-2
```

Every downstream artifact inherits that string: bead IDs, the agent names `sase bead work` derives from them, TUI rows,
notifications, and commit trailers.

## Root cause

Bead IDs are `<issue_prefix>-<base36 counter>` (`src/sase/bead/ids.py:33-51`). `issue_prefix` is chosen exactly once,
when a bead store is initialized, and then persisted in the store's `config.json` (`src/sase/bead/project.py:123-138` →
`get_default_config` → `src/sase/bead/config.py:82-88`).

The defect is in `_detect_prefix` (`src/sase/bead/config.py:56-79`):

```python
def _detect_prefix(root_dir: Path) -> str:
    """Detect issue prefix from git remote or directory name."""
    project_name = infer_project_name_from_cwd()
    if project_name:
        return project_name
    ...
```

`infer_project_name_from_cwd` (`src/sase/bead/project_name.py:67-101`) is a _canonical identity_ resolver: every one of
its three strategies returns the `~/.sase/projects/<key>` directory key — the checkout marker's `project_name`, the
workspace-provider hook's name canonicalized through `resolve_project_alias_ref`, or a directory scan of
`sase_projects_dir()`. For a `#gh` project that key is `gh_<org>__<repo>`, so `_detect_prefix` returns
`gh_bobs-org__bob-cli` and every bead in the store is stamped with it.

The bug was introduced by commit `502037305` ("fix: Project name resolution for 'sase bead create'"), which added the
`infer_project_name_from_cwd()` branch. That commit correctly stopped prefixes from being derived from an ephemeral
workspace directory name, but it wired the prefix to the storage key rather than to the user-facing name — precisely
what the repo's own `gotchas` memory forbids: _"Show Project Names, Never ProjectSpec Keys — user-facing text must
render the configured `PROJECT_NAME:` (`sase`, not `gh_sase-org__sase`)."_

The correct value is already available: the ProjectSpec at
`~/.sase/projects/gh_bobs-org__bob-cli/gh_bobs-org__bob-cli.sase` carries `PROJECT_NAME: bob-cli`, and
`sase.project_display_names.project_display_name_for(key)` maps a canonical key to that label (falling back to the key
when no `PROJECT_NAME` is set).

### Why `sase bead work` is where it showed up

`sase bead work` on an approved epic plan materializes the store on demand when it does not yet exist:
`resolve_plan_file_context` calls `init_beads(location.root, location.beads_dirname)`
(`src/sase/bead/cli_work_from_plan_store.py:68-69`) → `BeadProject.init` → `get_default_config` → `_detect_prefix`. So
the TUI epic approval → background `sase bead work` path was simply the first thing to create `bob-cli`'s store, and it
captured the wrong prefix permanently. `sase bead init`, `sase bead create`, and the SDD store materialization/adoption
paths (`src/sase/sdd/beads.py:63,93`, `src/sase/sdd/_store_materialization.py:456`,
`src/sase/sdd/_store_adoption.py:300`) all share the same defect — `sase bead work` is not special.

The sase repo itself is unaffected only by accident: its store was initialized while the project was still keyed `sase`,
so it kept prefix `sase` even after the key became `gh_sase-org__sase`.

## Scope

In scope:

1. Fix prefix derivation at store-init time (the root cause).
2. Detect stores that already carry a leaked key prefix, via `sase bead doctor`.
3. Give those stores a forward-only repair so future beads use the right prefix.

Explicitly out of scope: **renaming bead IDs that already exist**. There is no rename primitive in the bead mutation
facade (`src/sase/core/bead_mutation_facade.py`), and existing IDs are referenced from canonical event streams, plan
frontmatter `bead_id`, design refs, ChangeSpecs, agent names, artifact refs, and commit history. That is an epic, not
this tale. Close this tale by filing a `sase bead create -T task` follow-up describing a full historical re-prefix
migration, so the project owner can triage it.

## Implementation

### 1. New module `src/sase/bead/prefix_policy.py`

Single home for prefix rules, so `config.py` and the doctor path share one implementation.

- `is_safe_bead_prefix(prefix: str) -> bool` — a prefix is safe when it is non-empty and contains no whitespace, no `.`,
  and no path separator (`/`, `\`), does not contain `--`, and does not end with `-`. Rationale: bead IDs must keep
  matching `^[^\s.]+-[0-9a-z]+(?:\.\d+)*$` (`src/sase/agent/bead_display.py:18-19`) so that agent names launched by
  `sase bead work` still resolve back to their bead, and `--` is the reserved agent-family separator, so a prefix
  containing it would make bead-named agents parse as family members.
- `default_issue_prefix(root_dir: Path) -> str` — the replacement policy, in order:
  1. `key = infer_project_name_from_cwd()`. If set, resolve `label = project_display_name_for(key)` (lazy import of
     `sase.project_display_names` inside the function, mirroring the lazy imports already used in
     `src/sase/bead/project_name.py:105-139`, to avoid an import cycle through the project lifecycle facade). Return
     `label` when `is_safe_bead_prefix(label)`; otherwise return `key` when `is_safe_bead_prefix(key)`.
  2. Git `origin` remote basename, `.git` stripped (unchanged from today's fallback).
  3. `root_dir.resolve().name` (unchanged).
- `stale_key_prefix_report(beads_dir: Path) -> tuple[str, str] | None` — return `(stored_prefix, corrected_prefix)`
  when, and only when, the store's `issue_prefix` is exactly the inferred ProjectSpec key **and** that key resolves to a
  different, safe `PROJECT_NAME` label. Returning `None` in every other case is deliberate: a deliberately customized
  prefix (`beads`, `gold`, a legacy name) must not be flagged, because we cannot distinguish it from an intentional
  choice.

Keep resolution keyed off the _process CWD_, not `root_dir`. Sidecar bead stores are materialized under `~/.sase/...` or
`<workspace>/sase/repos/beads`, whose own path says nothing about the owning project; today's cwd-based inference is
what makes those paths resolve correctly. Note this explicitly in the module docstring so a future reader does not "fix"
it into `root_dir`-based inference.

### 2. Rewire `src/sase/bead/config.py`

Replace the body of `_detect_prefix` with a delegation to `prefix_policy.default_issue_prefix(root_dir)`. Keep
`_detect_prefix` as the name so the existing tests and any callers keep working. `get_default_config` is unchanged.

### 3. Doctor diagnostic

`BeadProject.doctor` and `BeadProject.doctor_report` (`src/sase/bead/project.py:522-575`) already append Python-side
messages after the Rust `doctor` call — `bead_sync_diagnostics` (`src/sase/bead/sync.py:364`) is the precedent. Follow
it exactly: append, from `stale_key_prefix_report(self.beads_dir)`, a message of the form

```
WARNING: bead issue prefix 'gh_bobs-org__bob-cli' is a ProjectSpec key; project name is 'bob-cli' (repair with: sase bead doctor --fix-issue-prefix)
```

Apply the same `"OK: no issues found"` suppression logic the existing appenders use, so the warning is not printed
alongside a bare OK line. This stays host-side glue rather than moving into `sase-core`: it reads SASE project lifecycle
state that only the host resolves, and the prefix default it validates already lives in Python while the Rust
`init_store` merely receives `issue_prefix` as a parameter (`src/sase/core/bead_mutation_facade.py:28-37`).

### 4. `sase bead doctor --fix-issue-prefix`

Register in `register_bead_doctor_parser` (`src/sase/main/parser_bead_store.py:126-155`), inserted so the options stay
alphabetically sorted: `--fix-design-refs` (`-F`), `--fix-issue-prefix` (`-I`), `--fix-projection` (`-P`), `--yes`
(`-y`). `-I` is free. Help text: `Preview and, after confirmation, reset the store's issue prefix to the project name`.

Handle it in `handle_bead_doctor` (`src/sase/bead/cli_admin.py:38-108`), following the shape of the existing repairs:

- Compute the preview from `stale_key_prefix_report`. If it is `None`, print `No issue prefix to repair.` and return.
- Render old → new and the consequence, then require confirmation unless `--yes`.
- Apply under `bead_store_write_lock(project.beads_dir)`: `save_config` with `issue_prefix` set to the corrected value,
  leaving `next_counter` and `owner` untouched, then commit with `auto_commit_bead_store(..., already_locked=...)`.
  Note: do **not** route this through `bead_store_mutation`, because that helper only commits when
  `project.mutation_changed` is set (`src/sase/bead/cli_common.py:170-199`), and a `config.json` rewrite is not a Rust
  bead mutation.
- Print an explicit consequence line, because this repair is forward-only:
  `Existing bead IDs keep the old prefix; only new top-level beads use 'bob-cli'.`

Confirm that the mixed-prefix state is safe before finishing: `_next_top_level_counter`
(`src/sase/bead/project.py:615-621`) takes `max(config next_counter, max_top_level_counter(prefix) + 1)`, and
`next_counter` is preserved by the repair, so the first bead after a repair is `bob-cli-<next_counter>` and cannot
collide with any `gh_bobs-org__bob-cli-*` ID.

### 5. Docs and help

- `src/sase/bead/cli_admin.py:349-352` quick-start block: add
  `sase bead doctor --fix-issue-prefix            Reset a leaked ProjectSpec-key issue prefix`, kept in the block's
  existing order next to the other `doctor` lines.
- `src/sase/xprompts/skills/sase_beads.md:350-352`: document the new flag next to `--fix-design-refs` /
  `--fix-projection`, including the forward-only caveat.
- `docs/beads.md`: same addition wherever `--fix-projection` is described.

## Tests

- `tests/test_bead/test_config.py` — extend the existing `_detect_prefix` tests:
  - a project key that maps to a different `PROJECT_NAME` yields the `PROJECT_NAME` (`gh_bobs-org__bob-cli` →
    `bob-cli`);
  - a project with no `PROJECT_NAME` still yields the key (regression guard for the `502037305` behavior that must be
    preserved);
  - an unsafe label (contains a space, a `.`, or `--`) falls back to the key;
  - no inferred project falls through to the git-remote and directory-name branches, as today.
- New unit tests for `is_safe_bead_prefix` and `stale_key_prefix_report`, including the negative case where a
  deliberately custom prefix (e.g. `beads`) is **not** flagged.
- `tests/test_bead/test_cli_doctor.py` — the warning appears for a key-prefixed store, is absent for a correctly
  prefixed store, and `--fix-issue-prefix --yes` rewrites `config.json` while leaving `issues.jsonl` bead IDs and
  `next_counter` untouched.
- An end-to-end guard that a store initialized through `init_beads` in a project whose key differs from its
  `PROJECT_NAME` gets the `PROJECT_NAME` prefix — this is the `sase bead work` path from the bug report, and it is the
  test that would have caught the original defect.

## Verification

- `just install` first (ephemeral workspace directories may have stale dependencies), then `just check`.
- Sanity-check the doctor path by hand in a scratch project whose key and `PROJECT_NAME` differ; confirm the warning
  text and that `--fix-issue-prefix` leaves the store git-clean afterwards.

## Follow-up to file

`sase bead create -T task` describing a full historical bead re-prefix migration for stores already stamped with a
ProjectSpec key (rename primitive in `sase-core`, plus rewriting plan frontmatter `bead_id`, design refs, ChangeSpec
links, and artifact refs), then `sase bead update <id> -s ready`.
