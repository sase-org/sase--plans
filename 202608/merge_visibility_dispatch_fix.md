---
tier: epic
title: Make merge visibility work through the real provider dispatch path
goal: '`sase vcs log --merges hide|show|only` and the ACE Commits merge cycle actually
  re-slice history at runtime, the partition law holds against a real repository,
  the epic''s own test-isolation flake is gone, and epic sase-i8 lands with recorded
  acceptance evidence.

  '
phases:
- id: dispatch
  title: Stop pluggy from silently dropping optional VCS hook arguments
  depends_on: []
  size: medium
  description: 'dispatch: make optional VCS hook parameters reach hook implementations
    by declaring them positional-or-keyword without defaults in both the hookspec
    and BareGitPlugin, add a structural guard test over every hookspec family, and
    re-point the optional-argument tests through VCSPluginManager so they can fail.

    '
- id: isolation
  title: Give each remote-fixture test its own origin repository
  depends_on: []
  size: small
  description: 'isolation: remove the shared bare-remote path that makes two provider
    tests collide inside one pytest worker, which is the reproducible flake recorded
    three times against this epic.

    '
- id: accept
  title: Redo end-to-end acceptance against real merge history
  depends_on:
  - dispatch
  - isolation
  size: small
  description: 'accept: run the full acceptance matrix the closed verify phase never
    recorded, against a repository that really contains merge commits, and write the
    evidence onto the epic bead.

    '
- id: land
  title: Land the epic and file the remaining follow-ups
  depends_on:
  - accept
  size: small
  description: 'land: refresh the stale contract manifest, file the surviving follow-up
    proposals as sized task beads, close the epic with a verification note, run symvision,
    and mark both plan files done.'
proposed_by: bbugyi200.athena.sase-i8.land
parent_bead: sase-i8
create_time: 2026-08-10 08:25:55
status: done
bead_id: sase-i8.10
---

- **PROMPT:** [prompts/202608/merge_visibility_dispatch_fix.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/merge_visibility_dispatch_fix.md)
- **PARENT:** [202608/merge_commit_support.md](https://github.com/sase-org/sase--plans/blob/main/202608/merge_commit_support.md)
- **BEAD:** [sase-i8.10](https://github.com/sase-org/sase--beads/blob/main/pages/sase-i8/sase-i8.10.md)

# Plan: Make merge visibility work through the real provider dispatch path

## Why

Epic sase-i8 shipped merge visibility across nine phases. Every phase bead is closed
`done`. The feature does not work.

Run against this repository's own history, which contains 101 real GitHub pull-request
merge commits:

```
$ sase vcs log -m only --limit 6 --format oneline
● babbb2a7  chezmoi chore: run sase init
● f42a68c07 sase    feat(memory): generate SASE size guidance note
● 8ed11bb80 sase    build(deps): raise sase-core-rs floor
...
$ git rev-list --parents -n1 8ed11bb80
8ed11bb80b6a218dcd49fed5529573e036bc32ca 8658abee6a733ddadd7b4b5bb01225ec66c8300d
```

`--merges only` returns ordinary single-parent commits. The JSON confirms it:
`"is_merge": false`, `"merge": null`. `-m show` returns exactly what `-m hide` returns.
The partition law — the design's spine, the thing that makes the three modes trustworthy
— does not hold at runtime in any repository.

### Root cause

pluggy 1.6.0 decides which arguments to hand a hook implementation in `varnames()`
(`pluggy/_hooks.py`):

```python
_valid_param_kinds = (
    inspect.Parameter.POSITIONAL_ONLY,
    inspect.Parameter.POSITIONAL_OR_KEYWORD,
)
```

Keyword-only parameters are not a valid kind, so they are invisible. What survives is
then split by whether the parameter has a default, and `pluggy/_callers.py` passes only
the ones **without** a default:

```python
args = [caller_kwargs[argname] for argname in hook_impl.argnames]
```

SASE's VCS hooks declare every optional parameter as keyword-only with a default. So
pluggy passes none of them and the implementation silently falls back to its own
defaults. Confirmed directly:

```python
>>> hc = get_vcs_provider(".")._pm.hook.vcs_log
>>> [(i.argnames, i.kwargnames) for i in hc.get_hookimpls()]
[(('cwd', 'limit'), ())]
```

`merges` never reaches `BareGitPlugin.vcs_log`. `hide` is what you always get, because
`hide` is the implementation's own default.

This is not limited to `merges`, and it is not new. Auditing every hookspec:

| Hook                         | Silently dropped                              |
| ---------------------------- | --------------------------------------------- |
| `vcs_log`                    | `since`, `until`, `authors`, `revs`, `merges` |
| `vcs_partition_commits`      | `merges`                                      |
| `vcs_fetch_remote`           | `timeout`                                     |
| `vcs_resolve_remote_log_ref` | `ref_name`                                    |

`since`, `until`, `authors` and `revs` predate this epic, and they are broken today:

```
$ sase vcs log --author zzz-nobody --limit 5 --format oneline
● babbb2a7  chezmoi chore: run sase init
...
```

An author filter matching nobody returns rows. `revs` being dropped means the
`("HEAD", remote_ref)` union log the collection layer asks for never happens, so
incoming commits cannot appear as rows even though the ahead/behind counts above them
say they exist. That is the exact "counts describe a different slice than the rows"
failure this epic set out to prevent, arriving through a different door.

The fix site is one signature per hook, so fixing only `merges` and leaving its siblings
broken in the same signature is not a smaller change — it is the same change made twice,
with the second half left undone.

Scope is bounded: `workspace_provider` and `llm_provider` hookspecs have no dropped
parameters, and the GitHub plugin in the `sase-github` repo implements none of the four
affected hooks and drops no arguments of its own. All work is in this repo.

### Why every phase's tests passed

The merge tests call the plugin object directly:

```python
subjects = {c.subject for c in BareGitPlugin().vcs_log(repo, 10, merges=merges)}
```

That is a plain Python call. Python honours the keyword; pluggy is never involved. Every
optional-argument test in `tests/test_vcs_provider_vcs_log.py` has this shape, so the
suite proves the plugin works and never proves the dispatch works.

The two tests that do go through `VCSPluginManager` cannot fail:

- `test_provider_log_delegates_to_hook` passes `authors=("bryan",)` and asserts the
  commit **is** returned. Dropping the filter returns the commit, so it passes for the
  wrong reason.
- `test_merge_visibility_keyword_is_optional_for_hookimpls` registers a stub that omits
  `merges` and asserts the call succeeds. It tests the forward-compatibility contract
  and nothing else.

A negative assertion in either would have caught this on day one.

### The other two findings

**The flake on this epic's bead is this epic's flake.** Three separate agents recorded
`tests/test_vcs_provider_vcs_log.py::test_remote_log_ops_fetch_partition_and_union_log`
failing the flake-baseline gate at `git push -u origin main`. Phase `provider` added
`test_vcs_partition_commits_honors_merge_visibility_modes`, which builds its bare remote
at `Path(repo).parent / "origin.git"`. `tmp_path.parent` is the worker-wide pytest
basetemp, shared by every test in that worker, and the pre-existing test uses the same
literal path. Both tests then create a root commit from identical inputs — same file,
same content, same message, same author. Same second, identical SHAs, the second push is
a no-op and passes. Different second, different SHAs, unrelated histories, and
`git push` is rejected non-fast-forward under `check=True`. Simulated directly:

```
TEST1 push OK: 051ff2f4de9a06e30dd111e1c177d3a15dc58392
TEST2 head:    f6f8e01daec23d632c24814083bc830a942b2b6b
hint: Updates were rejected because the remote contains work that you do not have locally.
```

**The verify phase verified nothing.** Bead sase-i8.9, "End-to-end acceptance against
real merge history", is closed `done` with zero notes. Its seven-item checklist is
precisely what would have surfaced the dispatch bug; items 3 and 4 fail on the first
command. Acceptance has to actually run.

## What is already correct

Confirmed working, and out of scope here:

- **Rust core and Python wire.** `vcs_log_wire_schema_version()` returns 3;
  `parse_merge_summary("Merge pull request #123 from org/feat", "PR title here")`
  returns the expected dict; `tools/validate_sase_core_rs` exits 0 against the installed
  wheel; the tolerant 7/8-field parser and `parent_ids` round-trip.
- **Renderers.** Merge marker, accent, legend key, parent lines, JSON `parent_ids` /
  `is_merge` / `merge`, and the condensed pull-request headline are covered by tests
  over synthetic wire objects and are genuinely correct — they simply never receive a
  merge commit at runtime.
- **CLI surface.** `-m/--merges` parses, rejects bad values, reaches `CommitFilterSpec`,
  and JSON reports `query.merges` correctly.
- **TUI touch points.** Binding, action, availability, keymap metadata,
  `default_config.yml`, help row, hint, and the `artifacts_commits_merge_row_120x40.png`
  golden are all present.
- **Dependency floor.** `sase-core-rs>=0.23.0,<0.24.0`, and the core-window ratchet tool
  added meanwhile agrees with it.
- **Docs.** `docs/vcs.md`, `docs/cli.md`, and `docs/configuration.md` describe the three
  modes, the partition law, and the git relationship accurately.

No commit landed since this epic started conflicts with or duplicates merge support.

---

## Phase `dispatch` — Stop pluggy from silently dropping optional VCS hook arguments

### The signature rule

For pluggy to pass a parameter, **both** the hookspec and the hook implementation must
declare it as positional-or-keyword **with no default**. Verified empirically across
every combination:

| Hookspec               | Hookimpl               | Result                    |
| ---------------------- | ---------------------- | ------------------------- |
| keyword-only, default  | keyword-only, default  | silently dropped          |
| positional, default    | positional, default    | silently dropped          |
| positional, default    | positional, no default | `PluginValidationError`   |
| positional, no default | positional, no default | **passed correctly**      |
| positional, no default | omits the parameter    | loads, keeps own behavior |

The last row matters: a third-party provider that has never heard of `merges` still
registers and still works. The forward-compatibility contract the hookspec docstring
promises survives this change, and
`test_merge_visibility_keyword_is_optional_for_hookimpls` should keep passing untouched.

### Change the four hookspecs

In `src/sase/vcs_provider/_hookspec.py`, remove the `*` and the defaults from the
dropped parameters of `vcs_log`, `vcs_partition_commits`, `vcs_fetch_remote`, and
`vcs_resolve_remote_log_ref`. The hookspec becomes the declaration of what callers must
supply, which is already true — `VCSPluginManager` passes every one of them explicitly
on every call.

Update the hookspec module docstring: it currently tells plugin authors that defaulted
keyword additions such as `merges` are optional. Keep the "an implementation may omit a
parameter it does not care about" contract, which is real, but delete any suggestion
that a defaulted keyword-only parameter in the _spec_ is a working way to add one,
because it is the bug.

### Change the implementations

In `src/sase/vcs_provider/plugins/_git_query_ops.py`, mirror the new signatures on
`BareGitPlugin.vcs_log`, `vcs_partition_commits`, `vcs_fetch_remote`, and
`vcs_resolve_remote_log_ref`. The bodies do not change; they already handle every value
correctly.

Callers keep their ergonomic defaults. `VCSPluginManager.log(...)`,
`partition_commits(...)`, `fetch_remote(...)` and `resolve_remote_log_ref(...)` in
`_plugin_manager.py` are ordinary Python methods, not hooks, so leave their keyword
defaults exactly as they are. `VCSProvider` in `_base.py` is the documented interface
and also keeps its defaults; reconcile its docstrings if the signatures drift.

### The guard test

The valuable artifact of this phase is a test that makes this class of bug impossible to
reintroduce, on any hook, in any family. Add a structural test that walks `VCSHookSpec`,
the workspace-provider hookspec, and the llm-provider hookspec, and asserts that for
every hook, every declared parameter other than `self` appears in
`pluggy._hooks.varnames(fn)[0]`:

```python
declared = [p for p in inspect.signature(fn).parameters if p != "self"]
passed, _ = varnames(fn)
assert [d for d in declared if d not in passed] == []
```

Today that test reports exactly the four hooks in the table above and nothing else, so
it goes red before the fix and green after, and it will catch the next keyword-only
parameter someone adds to any hookspec.

### Tests that can actually fail

Re-point the optional-argument tests in `tests/test_vcs_provider_vcs_log.py` at
`_make_git_provider()` — the `VCSPluginManager` helper already defined in that file —
instead of `BareGitPlugin()`. There are about twenty direct call sites across
`tests/test_vcs_provider_vcs_log.py`, `tests/test_vcs_provider_git_ops.py`, and
`tests/test_vcs_provider_commit_first_rebase.py`; the ones that matter are those passing
`since`, `until`, `authors`, `revs`, or `merges`. Tests that pass only `cwd` and `limit`
exercise nothing pluggy can break and may stay as they are.

Every converted assertion must be able to fail. Add, through the plugin manager:

- `merges="only"` returns merge commits and nothing else; `merges="hide"` returns no
  merge commits; `merges="show"` returns both.
- The partition law: on one repo, revision set, and limit, the `hide` and `only` id sets
  are disjoint and their union equals the `show` set. This is the law phase `provider`
  was supposed to own, asserted for the first time through the path users take.
- `vcs_partition_commits` with `merges="only"` returns merge ids only.
- `authors=("nobody-matches-this",)` returns an empty list — the negative assertion
  whose absence let this ship.
- `since` in the future returns nothing, and `until` in the past excludes recent
  commits.
- `revs=("HEAD", <remote ref>)` returns the union, including a commit reachable only
  from the remote ref.
- `resolve_remote_log_ref` with an explicit `ref_name` honours it rather than the
  default.

Fix `test_provider_log_delegates_to_hook` so that it asserts something a dropped filter
would break.

### Verification

`just check`, plus `sase vcs log -m only --limit 5` against this repository returning
only real merge commits.

---

## Phase `isolation` — Give each remote-fixture test its own origin repository

In `tests/test_vcs_provider_vcs_log.py`, both
`test_vcs_partition_commits_honors_merge_visibility_modes` and
`test_remote_log_ops_fetch_partition_and_union_log` build their bare remote at
`Path(repo).parent / "origin.git"`, which is the pytest basetemp shared by every test in
the worker.

Give each test a distinct remote. The natural fix is a small fixture beside the `repo`
fixture that returns a unique bare-repo path — deriving it from the `repo` fixture's own
directory name keeps it unique per test without any global counter, since `tmp_path` is
already unique per test. Whatever shape you choose, no two tests may write the same bare
repository, and the remote must stay outside the working repo so `git push` still has a
real remote to talk to.

Grep the whole `tests/` tree for other uses of `Path(...).parent` as a scratch location
before finishing; the same trap is available to any test that needs a sibling directory.

### Verification

Run the two tests together in one worker, in both orders, and repeatedly:

```bash
.venv/bin/python -m pytest tests/test_vcs_provider_vcs_log.py -p no:randomly -q
```

Then confirm the flake-baseline gate is clean with `just selection-health`. Because the
failure depends on whether the two root commits land in the same wall-clock second, a
single green run proves little; construct the failing interleaving deliberately (an
explicit delay between the two tests reproduces it every time on the current code) and
show it passing after the fix.

---

## Phase `accept` — Redo end-to-end acceptance against real merge history

Bead sase-i8.9 closed with no evidence. Run its checklist for real, against this
repository's own history, which contains 101 pull-request merge commits, and record the
output.

1. `sase vcs log` with no merge flag — unchanged from before the epic: no merge column,
   no legend key, same alignment.
2. `-m show` — merges appear, marked with `◆`, aligned, and the legend teaches the
   glyph.
3. `-m only` — merge commits **only**. Recognized GitHub pull-request merges render as
   `#<n>  <PR title>`. Confirm every returned commit has two or more parents with
   `git rev-list --parents -n1`.
4. The partition law in the wild: on one scope and limit, the `hide` and `only` row
   counts sum to the `show` row count.
5. All four formats under all three modes — `pretty`, `full`, `oneline`, `json`. Confirm
   `query.merges` and the per-commit `parent_ids`, `is_merge`, and `merge` fields.
6. `sase ace` → Artifacts → Commits: `s` cycles the three modes, the filter bar shows
   the `merges:` token, and for a merge row the detail pane shows the badge, the
   parents, and a **non-empty** diff labeled as being against the first parent. The
   commit modal agrees. A merge commit is only reachable here once the query window
   includes one, so widen the window rather than concluding there is nothing to see.
7. `tools/validate_sase_core_rs` enforces the schema-3 requirement, so a stale wheel
   fails loudly.

Anything that does not behave as specified is a finding for this epic, not something to
paper over. Record the evidence as a note on bead sase-i8.

Also run `just check-full` — every lint gate plus the full suite — and note any failure
that is not attributable to this work, with the test id and whether it reproduces in
isolation. A full-suite baseline taken during verification was still running when this
plan was written, and an earlier phase reported roughly twenty-one failures that it
attributed to unrelated ACE/TUI assertions and the stale contract manifest; establish
which of those are real so phase `land` files accurate follow-ups.

---

## Phase `land` — Land the epic and file the remaining follow-ups

### Refresh the stale contract manifest

`tests/contract_manifest.txt` is missing `tests/test_probe_core_floor_tool.py`, added by
the unrelated core-floor-probe commit.
`.venv/bin/python tools/refresh_contract_manifest` regenerates it as a one-line
addition. Not caused by this epic, but it breaks the gate this epic must pass through,
so fix it here and say so.

### File the surviving follow-up proposals

Every one of these must go through `/sase_new_task`, which decides between corroborating
a duplicate, attaching to a causally related active epic, or creating a sized task. Name
the proposing bead in each. Record every outcome — including any that are declined and
why — in the close note.

Already resolved, so file nothing; just confirm and record:

- Published `sase-core-rs` 0.21.3 lacking `parse_merge_summary` (sase-i8.8) — resolved
  by the 0.23.0 floor.
- The markdown format gate blocking on `sase/memory/build_and_run.md` (sase-i8.7,
  sase-i8.8) — resolved by the later formatting commit.

Still open, so propose:

- `tools/validate_sase_core_rs` has pre-existing mypy errors that `just check` never
  sees, because `[tool.mypy].files` in `pyproject.toml` covers only `src`. Widening mypy
  to `tools/` would catch this whole class (sase-i8.2).
- `tests/test_multi_prompt_launcher_xprompt_groups.py::test_launcher_qualifies_research_swarm_per_dispatch`
  fails under xdist with `research.0.cdx` already reserved on the second launch, and
  passes alone (sase-i8.3).
- `tests/ace/tui/widgets/test_prompt_xprompt_highlight.py::test_xprompt_highlight_overlay_marks_spans_and_registers_styles`
  failed once in a full lane and passed on isolated and xdist reruns (sase-i8.4).
- `tests/ace/tui/test_prompt_bar_xprompt_selector_requests.py::test_vcs_tag_offers_project_local_xprompts_by_canonical_name`
  and `::test_vcs_tag_directory_key_spelling_also_resolves` are xdist-order sensitive
  (sase-i8.6).
- `tests/ace/tui/test_tasks_pane_store.py::test_following_a_live_store_row_bypasses_the_mtime_cache[success-True]`
  reproduces independently with `known_mtime` None (sase-i8.7).
- Generated memory README drift: `sase init memory --check` wants changes in the
  chezmoi-managed memory README, which phase agents cannot make without explicit user
  permission (sase-i8.8). Confirm whether it still reproduces before filing.
- `vcs_repo_stats` counts commits with `git rev-list --count HEAD` and contributors with
  `git shortlog -sne`; both start including merge commits once squash-merge is off. The
  epic plan deliberately deferred this and asked for exactly this follow-up.
- `src/sase/updates/incoming_commits.py` lists and counts incoming commits with plain
  `git log`/`git rev-list --count` over a revision range and is not merge-aware. It is a
  second commit-listing surface that will start showing merge commits for the same
  reason, and it now has `MergeVisibility` available to it.
- Any failure phase `accept` establishes as real and unrelated.

### Close out

Close the epic with a note covering what was verified in phases `dispatch` through
`accept` and what the follow-up outcomes were:

```bash
sase bead close sase-i8 --note "<verification summary>"
```

If the close is rejected, the named phases were never completed: finish or reopen them.
Never force merely to make the command succeed.

After closing, run `just symvision` — the epic-symbol whitelist entries for sase-i8 in
the `Justfile` expire at close — and remove the stale entries and any unused code it
reports. `MergeSummary` and `merge_summary` were whitelisted by phase `wire`; they are
consumed by the render path now, so confirm rather than assume before deleting anything.

Finally set `status: done` in the frontmatter of both this plan file and the original
epic plan file, `@plan:202608/merge_commit_support.md`.
