---
tier: tale
title: Fix the Python 3.12 slice[...] import crash in sase tui
goal:
  sase tui imports cleanly on Python 3.12 again, and agents verifying on 3.14 catch this
  class of crash locally.
size: small
decisions:
  guard:
    ask:
      Also add a lint stage (~5s per run) that catches 3.12 eager-annotation crashes
      from a 3.14 checkout?
    default: true
    why:
      agents verify on 3.14, which hides this bug class; Master Gate only sees it after
      landing
    answer: true
proposed_by: bbugyi200.athena.1d3
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-16 16:01:51
status: wip
---

# Fix the Python 3.12 `slice[...]` import crash in `sase tui`

## Problem

`sase tui` (and anything else that imports `sase.ace.tui.models`) crashes at import on
Python 3.12:

```
File "src/sase/ace/tui/models/_loaders/_done_common.py", line 158, in _LazyInstructionBundleFiles
    index: slice[
TypeError: type 'slice' is not subscriptable
```

## Root cause

1. Commit `fadbfdc497` ("feat(ace): memoize per-load agent loader work") added
   `_LazyInstructionBundleFiles`, a `list[str]` subclass, to
   `src/sase/ace/tui/models/_loaders/_done_common.py`. Its `__getitem__` overload and
   implementation annotate the slice parameter as
   `slice[SupportsIndex | None, SupportsIndex | None, SupportsIndex | None]`, which
   matches typeshed's `list.__getitem__`.
2. `slice` is generic only in typeshed. At runtime, `builtins.slice` has no
   `__class_getitem__` on any CPython version (3.12, 3.13, and 3.14 all raise
   `TypeError: type 'slice' is not subscriptable`), so the annotation is valid only if
   it is never evaluated.
3. `_done_common.py` is one of the few `src/` modules (about 790 of 6,500) without
   `from __future__ import annotations`. Python 3.12 and 3.13 evaluate function
   annotations as soon as the `def` runs, so the class body crashes during import.
   Python 3.14 defers annotation evaluation (PEP 649/749), so the bad annotation is
   never evaluated there.
4. Agent dev venvs and the athena `uv tool` install run Python 3.14, so the author's
   local tests, mypy (which configures `python_version = "3.12"` but does not model
   runtime subscriptability), and ruff all passed. The apollo install runs on system
   Python 3.12, which is the project's floor (`requires-python = ">=3.12"`).
5. CI did catch it, but only after it landed. Master Gate run 43951346220 for
   `fadbfdc497` failed on 3.12 in `lint` and in all 8 test shards with this exact
   `TypeError` (`ImportError while loading conftest 'tests/ace/tui/conftest.py'`). The
   gate was already red for unrelated reasons on earlier commits, so a new red run did
   not stand out. Those earlier failures are out of scope here.

A Python 3.14 sweep that imports every `src/sase` module lacking the future import and
evaluates every function, method, overload, and class annotation with
`annotationlib.Format.VALUE` (which reproduces 3.12's eager evaluation) found exactly
two failures. Both are this class's `__getitem__` (the overload and the implementation).
Nothing else in `src/` currently has this problem.

## Change 1 — the fix (always)

In `src/sase/ace/tui/models/_loaders/_done_common.py`, add
`from __future__ import annotations` right after the module docstring. Most of the repo
already follows this convention. With it, every annotation in the module becomes a lazy
string on every Python version, so `slice[...]` is never evaluated at runtime. Keep the
precise typeshed-matching overload annotations unchanged so mypy's view of the `list`
override does not change.

Do not switch to a bare `slice` or quote only the two annotations. The future import is
the repo's established pattern and also protects any later annotation added to this
module.

Verification for this change:

- Emulate 3.12's eager evaluation from the 3.14 venv:
  `.venv/bin/python -c "import annotationlib, typing; from sase.ace.tui.models._loaders._done_common import _LazyInstructionBundleFiles as C; [annotationlib.get_annotations(f, format=annotationlib.Format.VALUE) for f in [C.__getitem__, *typing.get_overloads(C.__getitem__)]]"`.
  Before the fix this raises `TypeError: type 'slice' is not subscriptable`. After the
  fix it raises nothing, because the annotations are plain strings. The definitive 3.12
  proof is the Master Gate `test`/`lint` legs getting past conftest import.
- The existing tests still pass: `tests/ace/tui/models/test_done_agent_image_files.py`
  and `tests/ace/tui/models/test_agent_loader_memos.py`.

## Change 2 — eager-annotation lint stage

> [!decision] guard = no Skip this whole section. Change 1 is the complete deliverable.

The process gap is that every local verification runs on 3.14, which cannot see this
class of bug. Close it with a whole-repo lint stage that always runs; it is not part of
the diff-scoped test lane. A pytest sweep would not work here: it imports modules
dynamically, so the static import-graph selector behind `just test-scoped` would never
pick it for a change like this one.

### `tools/check_eager_annotations` (new, extensionless, executable)

Follow the structure of `tools/check_test_wait_helpers`: shebang
`#!/usr/bin/env python3`, module docstring, `from __future__ import annotations`, typed
functions (`tools/typecheck_extensionless_tools` type-checks it automatically under
`_lint-mypy`), and `main(argv) -> int`.

Behavior:

- Arguments: `--src DIR` (default `<repo>/src`) and `--package NAME` (default `sase`).
  Prepend `--src` to `sys.path`.
- Walk `<src>/<package>/**/*.py` in sorted order. Skip `__main__.py` files. Use `ast` to
  skip any module that has a top-level `from __future__ import annotations`, because its
  annotations are always lazy strings. Derive the dotted module name from the path, with
  `__init__.py` mapping to the package name.
- Import each remaining module with `importlib.import_module`. Any exception raised
  during import (catch `Exception`, not `BaseException`) is a finding. On 3.12/3.13 this
  is how eager-annotation errors show up, and it is the only check those versions need,
  so the tool is meaningful in CI's 3.12 `lint` job too.
- On Python 3.14 and later (`sys.version_info >= (3, 14)`, importing `annotationlib`
  lazily inside that branch so the tool still runs on 3.12), also emulate eager
  evaluation by calling
  `annotationlib.get_annotations(obj, format=annotationlib.Format.VALUE)` on:
  - the module object;
  - every top-level function and class whose `__module__` equals the module name;
  - each class's methods, unwrapping `staticmethod` and `classmethod` to `__func__` and
    `property` to `fget`, `fset`, and `fdel`;
  - nested classes defined in a class body (recursively);
  - every `typing.get_overloads(fn)` variant of each function or method. Overload stubs
    are replaced by the implementation in the class namespace, but on 3.12 their
    annotations were still evaluated, and this bug lived in one.

  Any exception is a finding. Skip objects whose `__annotate__` is missing or `None`.

- Print one line per finding:
  `<relative path>: <qualified name or 'import'>: <ExcType>: <message>`. End with a
  one-line hint: "Python 3.12 evaluates annotations eagerly; add
  `from __future__ import annotations` or quote the annotation". Exit 1 if there were
  any findings and 0 otherwise. On success print nothing, or one short summary line, to
  match the other `_lint-*` tools under `tools/run_silent`.
- Running it against the fixed tree must take only a few seconds (the prototype sweep
  took about 4.4s on athena) and must report zero findings.

### Justfile wiring

- Add a recipe `_lint-eager-annotations: _setup` that runs
  `{{ venv_bin }}/python tools/check_eager_annotations`, next to `_lint-test-waits`.
  Give it a short comment saying why it exists (3.14 hides eager-annotation crashes that
  break the 3.12 floor).
- In `lint`, add a stage after the test-waits stage:
  `@printf "\n---------- Checking eager annotation evaluation (Python 3.12 floor)... ----------\n"`
  followed by `@just _lint-eager-annotations`.
- In both `check` and `check-full`, add
  `@tools/run_silent "lint (eager annotations)" just _lint-eager-annotations` right
  after the `lint (test waits)` line, keeping the column alignment of the neighboring
  lines.

### Tests

- `tests/test_check_eager_annotations_tool.py`: build a throwaway package under
  `tmp_path/src/fakepkg/` and run the tool as a subprocess
  (`sys.executable tools/check_eager_annotations --src ... --package fakepkg`) so
  `sys.modules` stays clean. Cover:
  - a module without the future import whose `list[str]` subclass has an `@overload`ed
    `__getitem__` annotated with `slice[int | None, int | None, int | None]` (mirroring
    this bug) is reported, and the tool exits 1. This holds on every Python version:
    3.12 reports it as an import finding, and 3.14 reports it through VALUE evaluation;
  - the same module with `from __future__ import annotations` is clean (exit 0);
  - a module whose annotation names a type imported only under `TYPE_CHECKING`, without
    the future import, is reported (`NameError`);
  - a `__main__.py` with side effects is never imported.
- `tests/test_justfile_lint_check.py`: add the new
  `'tools/run_silent "lint (eager annotations)" just _lint-eager-annotations'` line to
  `_CHECK_GATE_LINES`, in its recipe order.
- `tests/test_justfile_lint_lint.py`: add `test_lint_includes_eager_annotation_stage`
  (dry-run `lint` contains the header and `just _lint-eager-annotations`) and
  `test_check_mirrors_eager_annotation_stage`, following the test-waits pair.

## Verification

- `sase tool run check` passes. Read the lint-and-test memory note for how to run it.
- If Change 2 is in scope, `.venv/bin/python tools/check_eager_annotations` exits 0 on
  the fixed tree. Temporarily removing the future import from `_done_common.py` makes it
  exit 1 and name `_LazyInstructionBundleFiles.__getitem__`. Restore the import
  afterward.
- After landing, the next Master Gate run gets past `tests/ace/tui/conftest.py` import
  on 3.12. Other pre-existing Master Gate failures are out of scope.

## Out of scope

- Updating the apollo machine. Its `sase` is an editable install of the `sase-org/sase`
  checkout, so a `git pull` there picks up the fix with no reinstall.
- The unrelated Master Gate failures that predate `fadbfdc497`.
- Requiring `from __future__ import annotations` repo-wide, for example with ruff
  `I002`. That would touch about 790 files, could break modules that inspect annotations
  at runtime, and the future import itself is slated for eventual deprecation.
