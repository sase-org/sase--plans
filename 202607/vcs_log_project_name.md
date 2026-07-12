---
create_time: 2026-07-08 16:04:15
status: done
prompt: 202607/prompts/vcs_log_project_name.md
tier: tale
---
# Plan: `sase vcs log` — show the project name, not the spec-file slug

## 1. The bug (what the user sees vs. what they should see)

`sase vcs log` renders a per-repo "badge" for every repo in the constellation (legend counts + a colored label on every
commit row). For the **primary** repo the badge currently reads the raw project-directory slug — e.g.
`gh_sase-org__sase` — instead of the configured project name `sase` (the `PROJECT_NAME` field of the project spec).

This is the recurring "never show the on-disk project key to a human" problem: `gh_sase-org__sase` is the GitHub
workspace-provider directory key (`gh_` prefix, `__` owner/repo separator), not a name any user typed or should read.
The correct label is the project's display name, `sase`.

Linked-repo badges (`chezmoi`, `sase-core`, `sase-github`, …) are already correct — they use the linked repo's own
`name`. Only the **primary** badge is wrong.

## 2. Root cause (single point of failure)

The primary badge string is set exactly once, in `src/sase/vcs_log/resolve.py`:

- `resolve_log_repos()` calls `ensure_project_file_and_get_workspace_num(create_missing=False)` and gets back
  `project_name`.
- `_resolve_project_repos()` then builds the primary entry as
  `LogRepo(name=project_name, path=primary_dir, kind="primary")`.

The problem is what `project_name` actually is. `ensure_project_file_and_get_workspace_num` → `_get_project_name()` →
`get_workspace_name(cwd)` returns the **project directory key / slug** (`gh_sase-org__sase`), then resolves aliases to
the _canonical directory key_ — never to the user-facing `PROJECT_NAME`. So the slug flows straight into `LogRepo.name`.

Because every downstream surface keys off `LogRepo.name`, that one wrong string leaks everywhere:

- the legend badge (`render.py` `_legend` → `repo.name`),
- every commit row's repo badge (`collect.py` tags each commit with `repo.name`, which the Rust aggregator carries into
  `entry.repo`, rendered by `_commit_line` / `_print_full_commit`),
- the per-repo accent color keying (`_repo_colors` keyed on `repo.name`),
- the `json` output (`repos[].name` and `commits[].repo`),
- per-repo warning prefixes.

**Verified empirically** in this workspace: `_get_project_name()` returns `'gh_sase-org__sase'`, and the canonical
display-name helper `project_display_name_for('gh_sase-org__sase')` returns `'sase'`.

### Why the existing tests didn't catch it

`tests/test_vcs_log_resolve.py` mocks `ensure_project_file_and_get_workspace_num` to return a `project_name` of `"sase"`
— a value that is _already_ a display name. The test therefore asserts `name == "sase"` and passes, but it never
exercises the real slug→display-name gap. The fixture used a pre-resolved name, so the bug lived entirely in the
untested boundary between the directory key and the display name.

## 3. The fix (one resolution point, reuse the canonical helper)

Resolve the primary repo's label through the **existing, canonical display-name helper** rather than using the raw
directory key. `src/sase/project_display_names.py` already provides:

- `project_display_name_for(key) -> str` — maps a project directory key to its user-facing name
  (`effective_project_name(record)` = `record.display_name or record.project_name`), falling back to the key when no
  display name is configured. It is cached by projects-dir mtime and is the same mechanism the ACE TUI, Telegram, and
  other surfaces use to show project names.

Change: in `src/sase/vcs_log/resolve.py`, when constructing the **primary** `LogRepo`, set
`name=project_display_name_for(project_name)` instead of `name=project_name`. This is a targeted change at the single
source of truth; no edits are needed in `render.py`, `collect.py`, `models.py`, the wire/facade layers, or `sase-core`.

This aligns `sase vcs log` with the module's own stated philosophy ("`sase vcs log` sees exactly the constellation every
other SASE surface does") — the badge now matches what the ACE Agents tab and every other surface already show for the
same project.

### Behavior properties

- **Fallback is safe.** If a project has no `PROJECT_NAME`/display name, `project_display_name_for` returns the key
  unchanged, so nothing regresses for projects that legitimately have no display name.
- **Non-project fallback unchanged.** The "not in a project" path (`_resolve_fallback_repos`, which uses the cwd
  basename) is untouched — there is no project key there to humanize.
- **SDD store label unchanged.** The separate-repo SDD badge (`sase-org/sdd`) comes from the SDD store record's own
  `repo` field (`_sdd_label` → `record.repo`), which is a real git repo slug (`owner/repo`), not a project-spec
  directory key. It is a legitimate, readable label for a distinct repository and is **deliberately out of scope** here.
  (Flagging it so you can confirm; if you'd rather it read differently, that's a separate, small follow-up.)

## 4. `--repo` filter interaction

`--repo` matching (`resolve.py` `_apply_filters`) compares against `repo.name`. After the fix, the primary repo matches
on its **display name** (`--repo sase`) — which is exactly the label the user now sees in the output, so this is the
intuitive, correct behavior. The `--repo` help text ("names are the project, each linked repo, and 'sdd'") stays
accurate.

The `sase vcs log` feature is brand new (landed today), so no scripts or muscle memory depend on filtering by the old
slug. Accepting the raw directory key as an _additional_ alias for the primary is therefore an optional, low-value
nicety; this plan **does not** add it, to keep the filter surface minimal. (Easy to add later in `_apply_filters` if
desired, mirroring how `sdd` is accepted as a literal alias.)

## 5. Reliability & edge cases

- **Single source of truth.** Fixing `LogRepo.name` once fixes the legend, every commit row, JSON, colors, and warnings
  simultaneously — there is no second place the slug can leak.
- **Display-name resolution never crashes the command.** `project_display_name_for` already swallows lookup errors
  internally and falls back to the key, so a malformed/absent record degrades gracefully rather than breaking the
  timeline.
- **No performance concern.** The helper is mtime-cached and is called once per invocation for the single primary repo.

## 6. Testing strategy

- **`tests/test_vcs_log_resolve.py` (reproduce + guard the bug):**
  - Update the `project` fixture so `ensure_project_file_and_get_workspace_num` returns a realistic **directory-key**
    `project_name` (e.g. `gh_sase-org__sase`), and patch `project_display_name_for` (as imported/used in the `resolve`
    module) to map that key → `sase`.
  - Assert the resolved primary `LogRepo.name == "sase"` (the display name), never the slug. This directly reproduces
    the reported bug and locks the fix in.
  - Add a **fallback** case: when `project_display_name_for` returns the key unchanged (no display name configured), the
    primary name equals the key — proving safe degradation.
  - Keep the existing linked/SDD/filter cases green (adjust their expectations only where they depended on the fixture's
    pre-resolved name).
- **Rendering sanity (optional, light):** confirm via the existing render tests that the primary badge in the
  legend/commit rows reflects the resolved display name (the render tests build `LogRepo` directly, so they already
  exercise "whatever name resolve produced" — a short assertion that a display-name primary renders as that name is
  enough; no golden churn expected).
- No new tests needed for `collect.py`/`render.py` internals — they are unchanged and already covered.

## 7. Build & coordination notes

- **Single repo (`sase`), no `sase-core` change**, no binding rebuild.
- Ephemeral workspace: run `just install` before `just check`.
- Run `just check` in `sase` before completion (this is a production code change, not in the bead/research exception
  categories).
- No `--help`/docs changes required (option surface is unchanged; only the primary label value changes).

## 8. Deliverables checklist

- [ ] `src/sase/vcs_log/resolve.py`: resolve the primary `LogRepo.name` via `project_display_name_for(...)` instead of
      using the raw project directory key.
- [ ] `tests/test_vcs_log_resolve.py`: fixture returns a directory-key `project_name`; new assertion that the primary
      label is the display name (`sase`), plus a fallback-to-key case; keep other cases green.
- [ ] `just check` green; a live `sase vcs log` smoke shows the primary badge reading `sase`.
