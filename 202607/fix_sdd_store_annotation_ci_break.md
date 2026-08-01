---
tier: tale
title: Unbreak CI by making SDD store annotations lazy on Python 3.12
goal:
  Every GitHub Actions CI job on master passes again, and a ruff gate makes the
  TYPE_CHECKING-only-name-in-a-runtime-annotation failure class impossible to reintroduce on any supported Python
  version.
create_time: 2026-07-28 13:00:26
status: done
---

- **PROMPT:** [prompts/202607/fix_sdd_store_annotation_ci_break.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/fix_sdd_store_annotation_ci_break.md)
- **AGENTS:**
  - [bbugyi200.athena.n7--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.n7.md#member-code)
- **COMMITS:**
  - [ee087a3](https://github.com/sase-org/sase/commit/ee087a3df01a59617c8a8650ee333b127c5393b3) — fix(sdd): defer type-only annotations at runtime

# Plan: Unbreak CI by making SDD store annotations lazy on Python 3.12

## Problem

Every CI run on `master` has been failing since commit `97015111b`
(`feat(sdd)!: write plan provenance headers (sase-ag.4)`). Two modules annotate a parameter with a name that is only
imported under `if TYPE_CHECKING:`, while neither module has `from __future__ import annotations`:

- `src/sase/sdd/_write.py:66` — `store: SddStore | None = None` in `write_sdd_files()`
- `src/sase/sdd/files.py:142` — `store: SddStore | None = None` in `write_sdd_files()`

In both files `SddStore` comes from a `TYPE_CHECKING` guard (`_write.py:13-14`, `files.py:12-13`).

On Python 3.13 and older, function-signature annotations are evaluated eagerly at `def` time, so importing either module
raises:

```
File "src/sase/sdd/_write.py", line 66, in <module>
    store: SddStore | None = None,
           ^^^^^^^^
NameError: name 'SddStore' is not defined
```

### Blast radius

The import chain `sase.sdd/__init__.py` → `sase.sdd.beads` → `sase.sdd.files` → `sase.sdd._write` means practically any
`import sase.*` explodes. On run [#11803](https://github.com/sase-org/sase/actions/runs/30378282770) the following jobs
failed directly, and the remainder (`test (3.12)`, `test (3.13)`, `test (3.14)`, `build`, `docs-build`, `fmt-md-check`,
`launch-perf-floor`) were cancelled by the fail-fast cascade:

| Job                     | Failing step                           | Symptom                               |
| ----------------------- | -------------------------------------- | ------------------------------------- |
| `lint`                  | Initialize SASE home                   | `NameError` from `sase init memory`   |
| `install-smoke`         | Health check (Rust extension loadable) | `NameError`                           |
| `visual-test`           | Run visual tests                       | `NameError` (collection)              |
| `bead-backend`          | Run Python bead parity and CLI tests   | 43 collection errors, all `NameError` |
| `phase7-perf-floor`     | Run Phase 7E regression floor          | `NameError`                           |
| `view-hints-perf-floor` | Run view-hints regression floor        | `NameError`                           |

### Why local `just check` did not catch it

The workspace dev venv runs **Python 3.14**, where [PEP 649](https://peps.python.org/pep-0649/) makes annotations lazily
evaluated — the broken annotation is simply never evaluated, so the import succeeds. CI installs **Python 3.12** for
`lint`, `install-smoke`, `visual-test`, `bead-backend`, and every perf-floor job (`.github/workflows/ci.yml`, the
`uv python install 3.12` steps), and 3.12/3.13/3.14 for the `test` matrix.

Minimal reproduction of the divergence:

```python
from typing import TYPE_CHECKING
if TYPE_CHECKING:
    from decimal import Decimal
def f(x: Decimal | None = None): ...
```

`python3.12` → `NameError: name 'Decimal' is not defined`; `python3.14` → imports fine.

### Not in scope (already fixed)

`published-core-minimum-smoke` failed on the older commit `ad3c751` because `pyproject.toml` floored
`sase-core-rs>=0.12.2`, and the 0.12.2 wheel is missing the six `sdd_plan_header_block_*` bindings that
`src/sase/sdd/plan_header_block.py` requires. Commit `a643a864c` already raised the floor to `sase-core-rs>=0.12.3`, and
that job no longer appears in the failure set. No action needed — do not change the dependency floor.

## Goal

1. Restore a green CI run on `master`.
2. Add a static gate so this failure class cannot recur, and so it is caught by local `just lint` even though local dev
   runs Python 3.14.

## Implementation

### Step 1 — Make the two annotations lazy

Add `from __future__ import annotations` as the first statement after the module docstring in each file:

- `src/sase/sdd/_write.py`
- `src/sase/sdd/files.py`

Keep the existing `if TYPE_CHECKING:` guards and the `SddStore | None` annotations exactly as they are. Do **not**
promote `SddStore` to a runtime import — the future import is the correct fix and matches how the sibling modules
already handle this.

Rationale for this over quoting the annotation:

- It is the dominant convention in this repo — **1856 of 2445** files under `src/sase/` already carry the future import,
  including `src/sase/sdd/store.py`, `src/sase/sdd/hosted_links.py`, and `src/sase/sdd/plan_archive.py`.
- It fixes the whole module at once rather than one annotation, so a later edit adding a second `TYPE_CHECKING`-only
  annotation in the same file stays safe.

Placement note: in both files the future import must precede the existing `from datetime import ...` /
`from pathlib import Path` / `from typing import TYPE_CHECKING` imports, and must come after the module docstring.
`keep-sorted` / `just fmt` conventions in this repo treat `__future__` as its own leading block.

### Step 2 — Add the ruff gate

In `pyproject.toml`, under `[tool.ruff.lint]`, add `TC004` to `select`:

```toml
select = [
    "E",     # pycodestyle errors
    "W",     # pycodestyle warnings
    "F",     # pyflakes
    "B",     # flake8-bugbear
    "C4",    # flake8-comprehensions
    "TC004", # flake8-type-checking: TYPE_CHECKING import used at runtime
    "UP",    # pyupgrade
]
```

`TC004` ("Move import out of type-checking block. Import is used for more than type hinting") is exactly the right
detector: ruff knows `target-version = "py312"` and that the file lacks the future import, so it treats the annotation
as a runtime use.

This gate has been measured against the current tree and is **zero-noise**:

```
$ ruff check --target-version py312 --select TC004 --output-format concise src tests tools
src/sase/sdd/_write.py:14:32: TC004 Move import `sase.sdd.store.SddStore` out of type-checking block. ...
src/sase/sdd/files.py:13:32: TC004 Move import `sase.sdd.store.SddStore` out of type-checking block. ...
Found 2 errors.
```

Those are precisely the two bugs above and nothing else — after Step 1 the rule reports clean. Both fix styles (future
import, or quoting the annotation) satisfy it, so it constrains correctness without dictating style.

Select **only** `TC004`, not the broad `TC` ruleset. The full ruleset reports **3720** findings, dominated by
`TC001`/`TC002`/`TC003`, which push imports _into_ `TYPE_CHECKING` blocks — the opposite direction, and the very shape
that caused this outage. `TC006` alone is 219 findings of unrelated `cast()` quoting. (`TC005`, empty type-checking
block, has 2 findings in `src/sase/ace/tui/actions/agent_workflow/_workflow_exec.py` and
`src/sase/ace/tui/actions/hints/_types.py` — unrelated to this outage; leave them alone.)

Note that `just lint` runs `ruff check src/ tests/` (Justfile `_lint-ruff`), so the gate applies to both trees.

### Step 3 — Lock the gate in place with a regression test

Following the existing `tests/test_justfile_lint.py` precedent of asserting lint-pipeline invariants, add a small test
asserting the ruff configuration still selects `TC004`, so a future config cleanup cannot silently drop it. Read
`pyproject.toml` with `tomllib` and assert `TC004` is present in `tool.ruff.lint.select`. Include a docstring/comment
pointing at this outage so the reason survives.

Do **not** write a runtime test that tries to resolve the annotation (for example
`typing.get_type_hints(write_sdd_files)` or `inspect.get_annotations(..., eval_str=True)`). With the future import,
`SddStore` is intentionally unresolvable at runtime, so such a test would fail by design. The invariant here is
genuinely static.

## Verification

Run from the repo checkout. `just install` first — workspace venvs go stale.

```bash
just install
just check
```

Targeted checks:

```bash
# 1. Gate reports clean on the fixed tree.
ruff check --target-version py312 --select TC004 src tests tools

# 2. The new regression test passes.
just test -- tests/test_ruff_config.py    # or wherever Step 3 lands
```

The most faithful end-to-end verification is a Python 3.12 import smoke, since that is what CI actually does and what
the local 3.14 venv cannot exercise:

```bash
uv venv --python 3.12 /tmp/sase-py312-smoke
VIRTUAL_ENV=/tmp/sase-py312-smoke uv pip install -e .
/tmp/sase-py312-smoke/bin/python -c "import sase.sdd; print('sase.sdd imports OK')"
```

This pulls the published `sase-core-rs` wheel rather than building the sibling Rust core, which is the same thing CI's
`published-core-minimum-smoke` does. If that install is too slow or the Rust extension is unavailable there, checks 1
and 2 plus a green CI run are sufficient — the mechanism is already proven.

### Confirm CI is actually green

After pushing, the fail-fast cascade means the previously _cancelled_ jobs (`test (3.12)`, `test (3.13)`, `test (3.14)`,
`build`, `docs-build`, `fmt-md-check`, `launch-perf-floor`) will run to completion for the first time since the break.
Watch that whole run, not just the six jobs that were failing directly — those cancelled jobs have not had a real signal
in several commits and may surface separate, unrelated problems. Use:

```bash
actstat --repo sase-org/sase -n 3
```

If a newly-revealed failure is unrelated to the `SddStore` annotation, report it rather than folding it into this
change.

## Acceptance criteria

- [ ] `src/sase/sdd/_write.py` and `src/sase/sdd/files.py` each begin (after the module docstring) with
      `from __future__ import annotations`.
- [ ] `SddStore` remains a `TYPE_CHECKING`-only import in both files; no runtime import of `sase.sdd.store` is added to
      either.
- [ ] `pyproject.toml` `[tool.ruff.lint] select` contains `TC004`, and not the broad `TC` ruleset.
- [ ] `ruff check --target-version py312 --select TC004 src tests tools` reports no findings.
- [ ] A test asserts `TC004` is still selected in the ruff config.
- [ ] `just check` passes.
- [ ] A full CI run on `master` is green, including the jobs that were previously cancelled rather than failed.
- [ ] `pyproject.toml`'s `sase-core-rs>=0.12.3,<0.13.0` floor is unchanged.
