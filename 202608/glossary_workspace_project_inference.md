---
tier: tale
title: Make `sase glossary` infer its project from workspace directories
goal:
  Every `sase glossary` subcommand resolves its project from a numbered managed
  workspace the same way `sase repo`, `sase workspace`, and `sase memory` already do, so
  agents no longer need `-p/--project` from their own cwd.
size: medium
proposed_by: bbugyi200.athena.05l
create_time: 2026-08-18 06:40:36
status: wip
---

# Make `sase glossary` project inference work from workspace directories

## Problem

`sase glossary` cannot infer its project from a numbered managed workspace. Every
subcommand exits with:

```
$ cd /home/bryan/.local/state/sase/workspaces/sase-org/sase/sase_14
$ sase glossary show Stitch
sase glossary show: no enabled project matched the active workspace
```

Reproduced from a `sase` workspace and originally reported from a `bob-cli` workspace
(`bob-cli_11`), so it is not project-specific. Agents run almost exclusively from
numbered workspaces, which means the glossary command group — the mechanism `AGENTS.md`
tells every agent to use for `GLOSSARY TERMS:` — is unusable from an agent's normal cwd
unless the agent knows to pass `-p`.

Meanwhile every other cwd-sensitive command resolves the same directory correctly:

```
$ sase workspace list | head -1
Project: sase  policy=xdg-state
$ sase memory read sase/memory/xprompts.md -r "..."      # works
$ sase glossary show Stitch -p sase                      # works
```

`docs/memory.md` currently documents the gap as an accepted limitation. This plan closes
it and deletes that documented workaround.

## Root cause

`sase glossary` does not use SASE's normal project inference. It has its own, in
`src/sase/xprompt/glossary_catalog.py`:

- `editor_glossary_catalog_for_project()` (line 104) → `select_project()` (line 268) →
  `glossary_project_record_for_workspace()` (line 312) → `_record_for_workspace()` (line
  332).
- `_record_for_workspace()` resolves cwd and returns the first enabled record whose
  `record.workspace_dir` **contains** it.

`record.workspace_dir` is the ProjectSpec `WORKSPACE_DIR:` — the primary checkout
(`/home/bryan/projects/github/sase-org/sase/`). Under the `xdg-state` workspace policy a
numbered workspace lives at `~/.local/state/sase/workspaces/<org>/<repo>/<repo>_<N>/`,
an entirely different root. It is never inside the primary checkout, so the containment
test cannot match and the function returns `None`.

The canonical inference, `infer_project_name_from_cwd()`
(`src/sase/bead/project_name.py:67`), resolves this correctly because it consults the
managed-checkout marker first. Every numbered workspace carries one:

```
$ cat .sase/checkout.json
{"primary_workspace_dir": "/home/bryan/projects/github/sase-org/sase/",
 "project_key": "sase-org/sase", "project_name": "sase", "workspace_num": 14, ...}
```

`sase repo`, `sase workspace`, `sase monitor`, `sase bead`, `sase proc`, and
`sase doctor` all route through that helper (directly or via
`resolve_project_context()`). Glossary is the outlier.

## Design

Add resolution tiers to `glossary_project_record_for_workspace()`, ordered
cheap-to-expensive, **after** the existing containment check. Tiers 2 and 3 only run
where resolution returns `None` today, so no currently-working call can change behavior
and no hot path pays for the new work.

**Tier 1 — containment (unchanged).** `_record_for_workspace()` exactly as it is today.
Pure in-memory over the passed-in records; still answers every primary-checkout call.
Markers are deliberately not written for primary checkouts (`write_marker()` returns
`None` for `PRIMARY_WORKSPACE_NUM`), so this tier must stay first.

**Tier 2 — managed-checkout marker (new).** `find_marker_from_cwd(str(workspace))` walks
up for the nearest `.sase/checkout.json`. Join the marker to a record in this order:

1. `marker.primary_workspace_dir` fed straight back through the existing
   `_record_for_workspace(Path(marker.primary_workspace_dir), records)`. This is the
   exact, unambiguous join — the marker's primary dir _is_ the ProjectSpec
   `WORKSPACE_DIR:` — and reusing the Tier 1 helper means one resolve-and-compare
   implementation, not two. Verified against live data: the marker's
   `/home/bryan/projects/github/sase-org/sase/` matches record `gh_sase-org__sase`.
2. `marker.project_key` (e.g. `sase-org/sase`) through the existing `_record_for_ref()`,
   whose `resolve_known_project_ref()` already maps `owner/repo` onto a project whose
   workspace is `~/projects/github/<owner>/<repo>`.
3. `marker.project_name` (e.g. `sase`) through `_record_for_ref()`, which matches
   `project_name`, `effective_project_name()`, and aliases.

Steps 2 and 3 are the fallback for providers that leave `primary_workspace_dir` empty —
`docs/workspace.md` states some plugins do exactly that.

**Tier 3 — canonical inference backstop (new).**
`infer_project_name_from_cwd(str(workspace))` returns a project _key_ (verified:
`sase workspace list` from this workspace resolves via this helper and then reads
`~/.sase/projects/<key>/<key>.sase`). Match it against `record.project_name`. This buys
exact parity with `sase repo`/`sase workspace` for the two tiers Tier 2 does not cover —
the workspace-provider `ws_get_workspace_name` hook and the legacy sibling-workspace
scan — and it self-heals if marker semantics ever change.

Constraints on the implementation:

- Import `find_marker_from_cwd` and `infer_project_name_from_cwd` **lazily inside the
  new helpers**, matching how every other caller of `infer_project_name_from_cwd`
  imports it. `glossary_catalog` is imported by ACE prompt-highlighting warmers and the
  LSP payload builder; do not add these to its module import graph.
- Wrap each new tier so a missing provider plugin, unreadable marker, or raising hook
  degrades to `None` (today's behavior) instead of propagating. The existing code in
  this module already swallows `OSError, RuntimeError, ValueError` around path
  resolution; follow that style rather than a bare `except Exception` unless the
  provider surface genuinely requires it (`_project_name_from_marker()` in
  `bead/project_name.py` uses broad guards for the same plugin calls — mirror it there
  and only there).
- Records the tiers match against remain the `records` sequence passed in, so a
  disabled, system-managed, or missing-on-disk project still resolves to `None` and
  still errors. Tier 3 is the one place that reads the ambient projects root
  (`SASE_HOME`), because `infer_project_name_from_cwd()` takes no `projects_root`; that
  is acceptable — every production caller passes `projects_root=None` already, and the
  parameter exists for tests, which Tier 2 satisfies without touching the root.
- Keep new helpers module-private (`_`-prefixed) and out of `__all__` so Symvision does
  not flag new public surface.

### What this reaches

One funnel, so a single fix covers all of it:

- `sase glossary list | show | read | log | add | del`, via
  `sase/glossary/cli_common.py` and `sase/glossary/cli_write.py`.
- `@repo` mention highlighting — `src/sase/xprompt/repo_mention_catalog.py:114` calls
  the same `select_project()` and emits the identical error string.
- The ACE glossary panel project ring — `src/sase/ace/tui/glossary_panel_catalog.py:77`
  calls `glossary_project_record_for_workspace()` directly to add the launch project to
  the ring.
- The LSP catalog payload's `default_project` (`glossary_catalog.py:217`).

### Deliberately unchanged: which `sase.yml` is read and written

The resolved project's config path still comes from `record.workspace_dir` — the
project's primary checkout — via `resolve_project_config_read_path()` /
`resolve_project_config_write_path()`. Inference decides **project identity only**.

That means `sase glossary add` run from a numbered workspace writes the _primary
checkout's_ `sase/sase.yml`, not the workspace's own copy. This is not a regression: it
is byte-for-byte what the currently documented workaround (`sase glossary add -p sase`)
already does from a workspace today. Making the invoking checkout's own `sase/sase.yml`
win instead is a real and arguably better design — it would keep an agent's glossary
edits inside the tree the agent commits from, matching `sase flag new` and
`sase memory init` — but it changes `-p` semantics too and deserves its own decision. It
is out of scope here; recommend filing it as a `feature` task bead after this lands
rather than bundling it into an inference fix.

### Error message

When all three tiers fail, the message should point at the escape hatch. In
`editor_glossary_catalog_for_project()` and `editor_repo_mention_catalog_for_project()`,
change the no-ref detail from:

```
no enabled project matched the active workspace
```

to something that names the fix, e.g.:

```
no enabled project matched the active workspace; pass -p/--project
```

Keep both call sites identical — `tests/xprompt/test_repo_mention_catalog.py` and
`tests/xprompt/test_glossary_catalog.py` both assert on this string, and the two modules
are intentionally symmetric.

## Files to change

| File                                           | Change                                                                                                                    |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `src/sase/xprompt/glossary_catalog.py`         | Tier 2 + Tier 3 in `glossary_project_record_for_workspace()`; new private helpers; updated docstring; error-detail string |
| `src/sase/xprompt/repo_mention_catalog.py`     | Matching error-detail string only                                                                                         |
| `tests/xprompt/test_glossary_catalog.py`       | New resolution tests (below)                                                                                              |
| `tests/xprompt/test_repo_mention_catalog.py`   | One test that the shared fix reaches repo mentions                                                                        |
| `tests/ace/tui/test_glossary_panel_catalog.py` | Ring includes the launch project when launched from a numbered workspace                                                  |
| `docs/memory.md`                               | Rewrite the limitation paragraph (lines ~139-147)                                                                         |

## Tests

Existing fixtures to build on: `tests/xprompt/test_glossary_catalog.py` already has
`_record()` (builds a `ProjectRecordWire`), `_write_config()`, and an autouse
`_fake_glossary_rust` fixture that stubs the Rust matcher; tests monkeypatch
`catalog.list_project_records`. Reuse all of it.

**Trap the implementer must know:** `find_marker_from_cwd()` is gated by
`pytest_path_is_sandboxed()` (`src/sase/core/state_write_guard.py:33`) and returns
`None` for any path outside `SASE_PYTEST_SANDBOX_DIR` while pytest is running. That
variable is the session `tmp_path_factory` basetemp
(`tests/_conftest_environment.py:35`), so marker files and fake workspaces **must** live
under `tmp_path`. A marker written anywhere else silently resolves to `None` and the
test will look like the fix failed. `SASE_HOME` is already isolated per test by the
autouse `_isolate_sase_home` fixture, which is what makes Tier 3 safe to exercise.

New cases in `tests/xprompt/test_glossary_catalog.py`:

1. **Numbered workspace resolves via marker.** Record's `workspace_dir` is
   `tmp_path/primary` (with a glossary in `sase/sase.yml`); `launch_workspace` is
   `tmp_path/state/proj_7` containing `.sase/checkout.json` whose
   `primary_workspace_dir` is `tmp_path/primary`. Assert the catalog loads and
   `result.project.key` is the record's key. This is the regression test for the bug.
2. **Nested subdirectory of a numbered workspace resolves.** Same setup,
   `launch_workspace = tmp_path/state/proj_7/src/sase`, exercising the marker walk-up.
3. **Empty `primary_workspace_dir` falls back to the ref tiers.** Marker with
   `primary_workspace_dir: ""` and `project_name` set to the record's display name;
   assert it still resolves. Add a sibling case keyed on `project_key` as `owner/repo`
   against a record whose workspace is `<home>/projects/github/<owner>/<repo>`.
4. **A marker for a project that is not in the enabled records stays unresolved.**
   Marker points at a primary dir no record declares; assert `result.project is None`
   and the diagnostic is the new no-match message. Guards against the fix resolving
   disabled or system-managed projects.
5. **Primary checkout still resolves without a marker.** Explicit coverage that Tier 1
   is untouched and still answers first.
6. **An explicit bad `-p` still never falls back to workspace inference.** The existing
   `test_catalog_without_ref_uses_launch_workspace_and_never_falls_back_from_bad_ref`
   covers this; extend it so the bad-ref call is made from a marker-bearing workspace,
   proving the new tiers are unreachable when `project_ref` is set.

In `tests/xprompt/test_repo_mention_catalog.py`: one test that a numbered workspace with
a marker resolves to the expected project's mention catalog.

In `tests/ace/tui/test_glossary_panel_catalog.py`: one test that
`build_glossary_project_ring()` includes the launch project when `launch_workspace` is a
marker-bearing numbered workspace of a project that declares no glossary — the ring's
documented "so `a` can bootstrap that project's first term" behavior currently drops it.

## Verification

```bash
just install                 # ephemeral workspace; required before anything else
just check                   # whole-repo lint gates + diff-scoped tests
```

Then verify the real repro against the built tree, from a numbered workspace:

```bash
.venv/bin/sase glossary show Stitch          # expect the definition, not the error
.venv/bin/sase glossary list                 # expect the sase term table
.venv/bin/sase glossary show Stitch -p sase  # unchanged, still works
```

Confirm the originally-reported case too: from a `bob-cli` numbered workspace,
`sase glossary show pomodoro` should print the `Pomodoro` definition. `bob-cli` does
declare a glossary containing that term (verified via `-p bob-cli`), so a correct fix
produces the definition, not any error.

Also confirm a directory that belongs to no project (e.g. `/tmp`) still fails, now with
the `-p/--project` hint.

`just check-full` is not expected to be required for a change this narrow, but if the
scoped run escalates or reports an unusual selection, run it through `/sase_monitor`
(`sase monitor start --command 'just check-full' … --next …`) rather than inline.

## Out of scope

- Moving project inference into the Rust core. The boundary rule in `CLAUDE.md` applies
  to _new_ shared backend behavior; this change consolidates glossary onto an existing
  Python helper (`infer_project_name_from_cwd`) that the CLI, TUI, and LSP already share
  through other commands. Relocating that helper is a much larger, separate change.
- Making the invoking checkout's `sase/sase.yml` the read/write target (see
  "Deliberately unchanged" above) — recommend a follow-up `feature` task bead.
- Refactoring `_record_for_ref` / `select_project` signatures, or threading
  `projects_root` into `glossary_project_record_for_workspace()`.
