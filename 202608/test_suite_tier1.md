---
tier: epic
title: Test suite Tier 1 — two-speed verification with diff-scoped test selection
goal:
  Agents verify a change with a diff-scoped, gate-free `just check` that costs ~1 core-minute instead of ~61
  worker-minutes, `just check-full` preserves today's exhaustive contract for landing and CI, and selection health (what
  was skipped, and whether skipping it was ever wrong) is a measured, machine-readable metric rather than an assumption.
phases:
  - id: engine
    title: Static import-graph selection engine
    depends_on: []
    size: medium
    description:
      "engine: build the cached import-graph selector, its depth-bounded reverse closure, the broadening/escalation
      rules, and the JSON selection manifest, plus the tools/select_tests CLI. No runner or Justfile behavior changes."
  - id: contract
    title: Curated contract/audit test set
    depends_on: []
    size: small
    description:
      "contract: add the `contract` pytest marker, curate the repository-wide audit tests behind it to a measured
      serial-runtime budget, generate a committed manifest from the marker, and add drift and budget guards."
  - id: runner
    title: Scoped run mode and the no-lease path
    depends_on:
      - engine
      - contract
    size: medium
    description:
      "runner: add a `scoped` mode to tools/run_pytest that runs the selection serially with the suite-gate explicitly
      disabled, escalates to the governed full lane when the selection is too large, and never queues for tokens."
  - id: check-split
    title: just check / just check-full split
    depends_on:
      - runner
    size: small
    description:
      "check-split: repoint `just check` at the scoped lane, add `just check-full` carrying today's exhaustive
      behaviour, and update docs/development.md, README.md, CONTRIBUTING.md, and the CI guard tests."
  - id: health
    title: Selection health metrics and false-negative detection
    depends_on:
      - runner
    size: medium
    description:
      "health: persist selection manifests to a durable host-local store, detect when a full run fails a test a recent
      scoped run excluded, and add `just selection-health` to summarize coverage, escalation, and false-negative rates."
  - id: contexts
    title: Coverage-context ground truth for selection
    depends_on:
      - engine
      - health
    size: medium
    description:
      "contexts: add --cov-context=test to the CI coverage leg, publish the contexts database as an artifact, and teach
      the engine to prefer per-test coverage ground truth over the static closure when a fresh baseline is available."
  - id: policy
    title: Two-speed verification policy in SASE memory
    depends_on:
      - check-split
      - health
    size: small
    description:
      "policy: obtain live user permission, then record the two-speed verification contract in sase/memory and
      regenerate the derived instruction files."
proposed_by: bbugyi200.athena.tn
create_time: 2026-08-05 20:55:58
status: wip
---

- **PROMPT:**
  [prompts/202608/test_suite_tier1.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/test_suite_tier1.md)

# Plan: Test suite Tier 1 — two-speed verification

## Background

The research report
[`202608/test_suite_verification_architecture/test_suite_verification_architecture.md`](https://github.com/sase-org/sase--research/blob/main/202608/test_suite_verification_architecture/test_suite_verification_architecture.md)
in the `sase--research` sidecar concludes that the suite is slow because of **admitted demand**, not slow tests: 200–400
full-suite runs per day at ~61 worker-minutes each, against a host supplying ~46,000 worker-minutes per day — 26% to 53%
of the machine, continuously.

**Tier 0 is done** (commit `9672c5602`): proportional gate CPU reserve, 950 MiB per-worker memory reserve, visual lane
out of the default `just test`, three scale tests behind `slow` with a CI home. That fixed per-run cost and CI
throughput. It did not touch demand.

This epic implements the report's **Tier 1** section. Tier 2 (gate fair-share, per-PR runtime budget, cross-workspace
result cache, stale-binding detection, TUI harness cost) is explicitly **not** in scope.

## Measurements taken while writing this plan

Everything below was measured directly at commit `01398f5af` on the 64-CPU / 62-GiB development host with the
workspace's own `.venv` (Python 3.14.3). **Where these figures contradict the report, they supersede it**, and the
reason for each difference is stated. The prototype scripts are throwaway; the phase agents will reimplement, not
inherit, them.

### The import graph

| Fact                                             | Value                                                       |
| ------------------------------------------------ | ----------------------------------------------------------- |
| Python files in `src/` + `tests/`                | 5,451 (2,750 src modules, 2,701 test modules)               |
| `test_*.py` files (the selection universe)       | **2,408**                                                   |
| In-repo import edges                             | 23,112                                                      |
| Cold graph build (AST parse, one core)           | **8.35 s**                                                  |
| Serialized graph size / reload cost              | 1.4 MB JSON / **8 ms**                                      |
| Test files importing no `sase.*` module directly | 81 (3.4%) — they reach production code only through helpers |

### The finding that changes the mechanism: a 497-module import cycle

The report proposes "static import-graph closure from changed `src/sase/**` modules to test files." **Unbounded closure
does not work in this repo.** `src/sase` contains a single strongly-connected component of **497 modules** (next
largest: 38). Every member of that SCC — which includes `sase.config`, `sase.core.paths`, `sase.core.time`,
`sase.ace.changespec`, `sase.bead.model`, `sase.xprompt` — transitively reaches 2,116 of 2,750 src modules and **2,088
of 2,408 test files (86.7%)**.

Measured over the last 25 commits, unbounded reverse closure selects a **median of 89.6% of all test files**. It is not
a selector; it is a slow way to write `tests/`.

Depth-bounding the reverse walk is therefore not a tuning knob, it is the mechanism. Selection sizes as a percentage of
all 2,408 test files, over the same 25 commits, expanding through both src and test-support modules:

| Reverse-closure depth |   Median |     Mean |       Max | Commits over 25% |
| --------------------- | -------: | -------: | --------: | ---------------: |
| 0 (direct importers)  |     0.3% |     0.7% |      4.9% |           0 / 25 |
| 1                     |     1.3% |     2.1% |      7.7% |           0 / 25 |
| **2 (proposed)**      | **3.8%** | **4.8%** | **31.0%** |       **1 / 25** |
| 3                     |     5.5% |     9.0% |     65.0% |           1 / 25 |
| unbounded             |    89.6% |    52.0% |     90.0% |          13 / 25 |

Depth 2 is the proposed default: it is the deepest bound whose tail stays bounded. Depth 3's median is only 1.7 points
higher but its worst case triples.

**The closure must expand through test-support modules, not just src modules.** A first pass that expanded only through
`src/**` produced a median of 3.2%; adding `tests/**` non-`test_*.py` helper modules to the expansion set raised it to
3.8% and, more importantly, made the 81 test files that import no `sase.*` module at all reachable — a src-only closure
can never select one of them, whatever the depth. There are 199 such helper modules; the largest has 116 test importers.

### What a scoped run actually costs

Two real selections, run serially on one core with `TMPDIR` on the disk-backed scratch root the runner uses:

| Selection               | Test files | Tests collected | Collection alone | pytest run (incl. collection) |  Wall |
| ----------------------- | ---------: | --------------: | ---------------: | ----------------------------: | ----: |
| depth-2 for `256da2887` | 106 (4.4%) |           1,055 |           4.91 s |                   **79.95 s** |  86 s |
| depth-2 for `99eedf749` | 228 (9.8%) |           2,292 |          not run |                  **265.07 s** | 268 s |
| full suite, 12 workers  |      2,408 |          25,937 |           16.1 s |       3,650 worker-s (report) | 385 s |

The two scoped rows are this session's measurements; the full-suite row is the report's, and its worker-seconds figure
is a sum of per-test durations across workers, not a wall time.

So a median-shaped scoped check costs **~86 worker-seconds against the full suite's ~3,650 — about 42×** — and it is
also **4.5× faster in wall clock** than the full suite running on twelve cores. Even the 9.8% case is ~13× cheaper in
host demand and still finishes faster than a full run.

This is better than the report's projected 5–15×, and the reason is the SCC finding cutting the other way: because
depth-2 selections are small, they are cheap; the report's 398-file / 4,966-test / 308 s sample corresponds to roughly
depth 3–4 here.

**Be honest about the shape of the win.** In an _uncontended_ moment the agent sees ~86 s instead of ~385 s — good, not
transformative. The transformation is that the check no longer competes for the shared token pool at all, so it never
waits behind the 45-minute acquisition timeout, and the host stops being asked for 12,000–24,000 worker-minutes a day.

### Two cheaper facts that shape the design

- **Excluding
  `tests/ace/tui/visual/**`from selections makes the scoped lane Pillow-free.** The depth-2 selection for`256da2887`included 8 visual test files (they import shared helpers), which drags in`tests/ace/tui/visual/conftest.py`→`PIL`at collection time. Those 8 files contribute **zero** selected tests — everything in them carries`visual`, which the marker expression deselects. With them filtered out, collection succeeds with `PIL`blocked (1,055 tests in 5.00 s), so the scoped lane can depend on`_setup`rather than`_setup-visual`
  and skip the visual dependency install in every fresh workspace.
- **A grep-derived "audit test" set is far too expensive to always run.** The 71 test files matching
  `parents\[[0-9]\]|git ls-files|rglob\(` take **100.5 s serially** — more than the median scoped selection itself. The
  contract set must be curated against a budget, not discovered by pattern.

## Design decisions, including deliberate deviations from the report

1. **Depth-2 bounded reverse closure, not unbounded closure.** Forced by the 497-module SCC. This makes selection a
   _heuristic_, and the plan treats it as one: broadening rules, an escalation threshold, a contract set, and measured
   false-negative tracking all exist because the closure is unsound.
2. **Expand through test-support modules as well as src modules.** See above.
3. **Drop the report's "format/lint on changed files."** The report's own gate table shows all non-test gates total ~35
   s warm / ~66 s cold, and the two expensive ones are inherently whole-repo: `mypy` is only fast because of its
   incremental cache (31 s cold, 0.55 s warm) and scoping it to changed files would break its own correctness model, and
   `symvision` is a whole-repo unused-symbol analysis that is meaningless on a subset. `ruff` is 0.33 s. There is
   nothing worth scoping here. **Both `just check` and `just check-full` keep every lint gate whole-repo.**
4. **Scoped runs are serial (`-n 1`) with the suite-gate explicitly disabled — not "at most 2 workers with no lease."**
   Two workers is a trap: `tests/conftest.py::configure_suite_gate` fires for any xdist run with more than one worker
   and acquires its own _exact_ lease with the full 45-minute timeout, so "no lease in `run_pytest`" would silently move
   the queue rather than remove it. `-n 1` makes `_xdist_worker_count` return `None`, so no gate path runs at all;
   setting `SASE_TEST_GATE_DISABLED=1` for the exec'd child is belt-and-braces and also covers subprocess-spawning
   tests. Serial also avoids paying xdist's per-worker collection multiplier on a selection whose collection is already
   ~5% of its cost.
5. **Escalation, not an ungoverned partial full-suite.** If the selection exceeds a ratio of the suite (default 25%) or
   any full-suite broadening rule fires, `just check` runs the **governed** full lane (`just test`, normal lease, full
   parallelism) instead. A 30%-of-suite selection at one core is ~20 minutes; the same work under a 12-token grant is ~2
   minutes. Never let the "fast path" become the slow path.
6. **Selection excludes `tests/ace/tui/visual/**`unconditionally**, so the scoped lane needs no Pillow.`just
   test-visual`remains the sole local visual execution and the dedicated CI`visual-test` job remains the sole CI one.
7. **The engine lives in Python in this repo, not in `sase-core`.** Per `rust_core_backend_boundary`, the litmus test is
   whether another frontend would need the behavior to match the TUI. Test selection is repository build tooling with no
   product surface; it belongs beside `tests/_suite_gate.py` and `tools/run_pytest`.
8. **Deletions and renames need no special machinery.** Any file still importing a deleted module fails at import and
   surfaces as a collection error; any module whose importers were updated for a rename has those importers in the
   change set as seeds. The engine records a `rename-or-delete` note on the manifest and bumps effective depth by one
   for that run, and does nothing else.

## Risk this epic accepts, stated plainly

Making `just check` scoped means a change can be committed on the strength of a heuristic selection. CI still runs
everything on every push, and Tier 0 cut CI's test leg from ~44 to ~15 minutes, so a false negative surfaces within
about a quarter hour rather than silently. **`master` will go red slightly more often.** The `health` phase exists to
measure exactly how much more often; if the false-negative rate is not effectively zero, the correct response is to
raise the default depth or grow the contract set, not to explain the failures away.

---

## Phase `engine` — Static import-graph selection engine

Pure addition. Nothing in `Justfile`, `tools/run_pytest`, or CI changes in this phase.

### Deliverables

- `tests/_test_selection.py` — the library. It sits with `tests/_suite_gate.py` for the same reason: `tools/run_pytest`
  already imports its shared runner logic from a private `tests/_*.py` module, and code under `tests/` is outside
  `symvision`'s `src/sase` scan and outside `mypy`'s `files = ["src"]`.
- `tools/select_tests` — executable CLI wrapper, in the style of `tools/run_pytest`.
- `tests/test_test_selection.py` — unit tests over a synthetic fixture tree, not the real repo graph.

### Graph construction

Enumerate `git ls-files -z 'src/**/*.py' 'tests/**/*.py' 'tests/*.py'` plus
`git ls-files -z --others --exclude-standard` filtered the same way, so a brand-new untracked test file participates.
Map each path to a dotted module name (`src/sase/foo/bar.py` → `sase.foo.bar`, `tests/x/test_y.py` → `tests.x.test_y`,
`__init__.py` → the package name). AST-parse each file, recording `Import` and `ImportFrom` targets — including, for
`ImportFrom`, both the module and each `module.name` alias, since `from sase.core import paths` names a module. Resolve
relative imports against the file's own package. Keep only edges whose target is an in-repo module.

**Cache it.** 8.35 s of AST parsing on every check is 10% of a median scoped run. Cache to
`.pytest_cache/sase-selection/graph.json`, keyed per file by `(path, size, mtime_ns)`; on load, re-parse only entries
whose key changed and drop entries whose file is gone. Reload of the full graph is 8 ms. Include a schema version in the
cache and discard the cache on mismatch. `--no-cache` forces a cold build.

### Selection algorithm

Inputs: the change set, a depth (default 2, `SASE_TEST_SELECTION_DEPTH`), and a max ratio (default 0.25,
`SASE_TEST_SELECTION_MAX_RATIO`).

The change set is `git diff --name-only $(git merge-base HEAD <base>)` union `git status --porcelain` paths (staged,
unstaged, and untracked). `<base>` defaults to `origin/master` and is overridable with `SASE_CHECK_BASE`. If the base
ref cannot be resolved, record rule `base-unresolved` and escalate to the full suite — a stale or missing remote must
fail toward more testing, never less.

Partition the module universe into **terminal** nodes (`tests/**/test_*.py`) and **expandable** nodes (everything else:
all of `src/**`, plus `tests/**` helpers, fixtures, and `_*.py` modules). Seed with the expandable modules in the change
set. Walk reverse edges `depth` times; every terminal node reached at any point joins the selection, and every
expandable node reached joins the next frontier. After the walk, harvest the direct terminal importers of every
expandable node reached. Always add test files that are themselves in the change set. Always add the contract set (from
the `contract` phase's manifest; until that phase lands, an empty manifest is a valid input). Always subtract
`tests/ace/tui/visual/**`.

### Broadening rules

Evaluated against the change set before the walk. Every rule that fires is recorded on the manifest by name.

| Changed path                                                                                                                                       | Effect                         |
| -------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------ |
| `tests/conftest.py` or any module the root conftest imports (`tests/_suite_gate.py`, `tests/_tmp_leak_guard.py`, `tests/_project_display_case.py`) | full suite                     |
| `pyproject.toml`, `uv.lock`, `tox.ini`                                                                                                             | full suite                     |
| `Justfile`                                                                                                                                         | full suite                     |
| `src/sase/default_config.yml`, any `src/sase/**/*.yml`, `*.json`                                                                                   | full suite                     |
| `tools/run_pytest`, `tools/select_tests`, `tests/_test_selection.py`, the contract manifest                                                        | full suite                     |
| A changed `sase_core_rs` identity (see below)                                                                                                      | full suite                     |
| `tests/**/conftest.py` other than the root                                                                                                         | every test under its directory |
| everything else not otherwise matched (docs, `sdd/**`, `.github/**`, `*.md`)                                                                       | contract set only              |

The `sase_core_rs` identity is not visible in `git diff` — the wheel lives in the venv and its source lives in a sibling
repo. Reuse the fingerprinting `tools/validate_test_environment` already performs (`_input_fingerprint` hashes
`pyproject.toml`, `uv.lock`, `pyvenv.cfg`, `../sase-core/Cargo.toml`, the validator implementations, environment
metadata, and stats the installed extension files). Record that digest on the manifest as `baseline.environment`; when
it differs from the previous run's manifest in the same workspace, fire the `core-identity-changed` rule.

### Escalation

If `len(selected) > max_ratio * len(all test files)`, or any full-suite rule fired, the result is the sentinel
`FULL_SUITE` rather than a file list. The runner phase acts on that sentinel; the engine only reports it.

### Manifest

Write JSON to `.pytest_cache/sase-selection/manifest.json` (the `health` phase adds the durable copy). Fields:

```json
{
  "schema": 1,
  "base": { "ref": "origin/master", "merge_base": "<sha>" },
  "changed_files": ["src/sase/..."],
  "depth": 2,
  "max_ratio": 0.25,
  "rules_fired": ["contract-set-always"],
  "escalated": false,
  "selected": ["tests/..."],
  "selected_count": 106,
  "universe_count": 2408,
  "baseline": { "environment": "<sha256>", "head": "<sha>", "tree_dirty": true },
  "graph": { "modules": 5451, "edges": 23112, "build_seconds": 0.4, "cache_hit": true }
}
```

`duration` and `outcome` are appended by the runner phase, which is the only component that knows them.

### CLI

```
tools/select_tests [--base REF] [--depth N] [--max-ratio R] [--no-cache]
                   [--format paths|json] [--manifest PATH] [--explain]
```

`--format paths` prints one repo-relative path per line, or the single token `FULL_SUITE` on escalation. `--explain`
prints the rules that fired and, for a sample of selected files, the seed and hop that pulled them in — this is the
debugging affordance that makes the heuristic auditable when someone asks "why is my check running 900 tests?".

### Tests

Build a synthetic tree under `tmp_path` with a known import shape (a chain, a fan-out hub, a cycle, a test-only helper,
a test importing nothing from src) and assert: depth bounding stops where it should; cycles terminate; the SCC shape
does not explode a depth-2 selection; test-support modules are traversed; every broadening rule fires on its trigger and
is named on the manifest; escalation trips at the ratio; visual paths are always excluded; untracked new test files are
selected; a cold and a warm cached build produce identical selections. Do **not** assert against the real repo graph —
those assertions rot on every commit.

---

## Phase `contract` — Curated contract/audit test set

The report calls for "a small contract/system safety set covering repository-wide audits — the tests that scan strings,
paths, bindings, symbols, command catalogs, schemas, and packaging metadata, which no dependency engine can select
correctly." It is right that they exist and right that no import graph finds them. It is silent on cost, and cost is the
whole problem: the obvious grep-derived set of 71 files runs for **100.5 s serially**, which would more than double the
median scoped check.

### Deliverables

- A `contract` marker registered in `pyproject.toml`'s `[tool.pytest.ini_options] markers`.
- `@pytest.mark.contract` applied to a curated set of module-level audit tests.
- `tests/contract_manifest.txt` — the generated, committed file list the engine consumes, sorted, one path per line.
- `just refresh-contract-manifest` — regenerates it from `pytest -m contract --collect-only -q`.
- A drift test asserting the manifest matches what the marker currently selects, in the same spirit as the existing
  generated-content guards (`tools/render_model_alias_docs`, `tools/validate_changelog`, keep-sorted).
- A budget test asserting the contract set's serial runtime stays within budget.

### Budget

**30 s serial, hard.** The engine adds this set to _every_ scoped selection, so it is a fixed tax on every check an
agent runs. Enforce it with a test that reads the `--durations` data the runner already emits, or, if that proves
awkward, by asserting a ceiling on the number of manifest entries calibrated against a measured run. State the measured
number in a comment so the next person can tell whether they are near the limit.

### Curation procedure

Start from `grep -rln 'parents\[[0-9]\]|git ls-files|rglob\(' tests --include='test_*.py'` (71 files today), then run
that set with `--durations=0`, aggregate by file, and admit files by _value per second_. Keep the tests that assert a
repository-wide invariant no import edge can express — command catalogs, schema and config validation, packaging
metadata, generated-file drift, CI workflow shape, cross-cutting terminology and presentation audits, the runner's own
guards (`test_run_pytest_tool.py`, `test_suite_gate.py`, `test_github_actions_ci.py`). Reject files that merely
_compute_ a repo path on their way to testing one module's behaviour — those are already reachable through the import
graph, and the `agents_sync` integration tests in particular are expensive and well-covered by their own imports.

Err small. A contract test that is not in the set is caught by CI within ~15 minutes; a contract test that is in the set
is paid for on every check by every agent, forever.

---

## Phase `runner` — Scoped run mode and the no-lease path

### `tools/run_pytest scoped`

Add `scoped` to the `mode` choices. Its behaviour:

1. Invoke the engine to obtain a selection.
2. If the result is `FULL_SUITE`, print one clear line saying which rule escalated, then run the **`fast` mode path
   unchanged** — governed lease, full parallelism. This is the single most important behaviour in the phase; see design
   decision 5.
3. Otherwise: `worker_count = 1`, no `-n`, no `--dist`, no lease, `SASE_TEST_GATE_DISABLED=1` in the exec'd environment,
   marker expression `FAST_MARKER_EXPRESSION` as usual, and the selected paths appended as positional arguments.
4. If the selection is empty, print what was checked and exit 0 without invoking pytest. A docs-only change should not
   pay 5 s of collection to learn it selected nothing. (The contract set makes a genuinely empty selection rare.)
5. Append `duration` and `outcome` to the manifest after pytest exits.

Step 3 conflicts with `os.execv`, which the runner currently uses to hand off to pytest and which cannot return to write
the manifest. Switch `scoped` mode to `subprocess.run` and keep `execv` for every other mode, or move the manifest
completion into a small `pytest_sessionfinish` hook. Either is acceptable; the subprocess form is simpler and the extra
process is noise against an 80-second run.

`_reject_numprocesses_args` must keep rejecting `-n` in scoped mode, and `SASE_PYTEST_WORKERS` must be rejected too with
a message pointing at `just check-full` — a governed parallel run of an arbitrary subset is exactly the
unaccounted-demand shape the gate exists to prevent.

### Justfile

```
# Diff-scoped test lane: selects tests from the change set, runs them serially
# without taking a suite-gate lease, and escalates to the governed full lane
# when the selection is too large or a broadening rule fires.
[positional-arguments]
test-scoped *args: _setup (_header "test-scoped")
    @SASE_JUST_INVOCATION_DIR="{{ invocation_directory() }}" {{ venv_bin }}/python tools/run_pytest scoped "$@"
```

Note `_setup`, not `_setup-visual` — justified by the measured Pillow result above, and load-bearing on the engine's
unconditional exclusion of `tests/ace/tui/visual/**`. **If a future change removes that exclusion, this recipe must go
back to `_setup-visual`.** Say so in a comment, and add a test asserting the engine never returns a path under
`tests/ace/tui/visual/`.

### Tests

Extend `tests/test_run_pytest_tool.py`: scoped mode emits no `-n`/`--dist`; scoped mode sets
`SASE_TEST_GATE_DISABLED=1`; scoped mode passes the selected paths through; the `FULL_SUITE` sentinel produces exactly
the `fast` command line; `-n` and `SASE_PYTEST_WORKERS` are rejected in scoped mode; an empty selection exits 0 without
invoking pytest. Add an integration-shaped test alongside `tests/test_suite_gate_integration.py` proving a scoped run
acquires **zero** tokens while a lease is held elsewhere — that is the property the whole phase exists to deliver, and
it is the one that will regress silently.

---

## Phase `check-split` — `just check` / `just check-full` split

`just check-full` takes today's `check` body verbatim. `just check` keeps every non-test gate identically and swaps its
final stage:

```
# Agent default: whole-repo lint gates + a diff-scoped test lane.
check: _setup
    ... (nine lint/validation lines unchanged) ...
    @tools/run_silent "test (scoped)"      just test-scoped

# Exhaustive verification: every gate plus the full test suite. Run this before
# landing, and in CI.
check-full: _setup
    ... (nine lint/validation lines unchanged) ...
    @tools/run_silent "test"               just test
```

`tools/run_silent` swallows output on success, so scoped mode must print its "selected N of 2,408 test files (rules: …);
manifest at …" summary on **failure** paths too, or agents will never see what was skipped. Simplest: print the summary
to stderr before running pytest, and let `run_silent` show it as part of the failure dump; additionally print the
escalation line unconditionally so an escalated run is never mistaken for a fast one.

### Documentation

- `docs/development.md`: the Verification Commands block gains `just check-full` and describes `just check` as the
  diff-scoped agent default; add a short subsection covering the selection mechanism, the depth and ratio environment
  variables, `tools/select_tests --explain`, the manifest location, and the honest statement that selection is a
  heuristic backstopped by CI.
- `README.md:120` "Run `just check` before submitting changes." → name `just check-full` for the pre-submit gate.
- `CONTRIBUTING.md:22` `just check  # All checks (fmt-check + lint + test)` → document both.
- `docs/rust_backend.md:508` mentions `just check` in a verification list — point it at `just check-full`.

### Tests

`tests/test_justfile_lint.py` (or a sibling) should assert both recipes exist, that `check` ends in the scoped lane and
`check-full` in the full lane, and that the two share an identical non-test gate list — the drift that will actually
happen is someone adding a tenth gate to one and not the other. `tests/test_github_actions_ci.py` should assert CI still
runs the full lane, not the scoped one.

---

## Phase `health` — Selection health metrics and false-negative detection

The report is right that "selection health is a production metric, not a one-time validation," and right that the fast
path should not be trusted until its false-negative rate is measured. This phase supplies the measurement. It is the
phase that decides whether this epic was a good idea.

### Durable manifest store

On every scoped run, also write the manifest to
`${SASE_HOME:-~/.sase}/test-selection/<project-key>/<utc-timestamp>-<head-sha>-<pid>.json`. The store is host-local and
shared across the numbered workspaces, which is exactly the correlation surface needed: phase agents in `sase_3` and
`sase_11` write manifests that the land agent in `sase_7` can read. Prune entries older than 30 days on write.

### False-negative detection

A false negative is: **a test that failed in a full run and was excluded by a scoped run over an ancestor of the same
change.** Detect it where the evidence is:

- `just check-full` (and `just test`) write the set of failing test node IDs to the same store.
- A correlator matches a full-run failure record against scoped manifests whose `baseline.head` is an ancestor of the
  full run's `HEAD` (`git merge-base --is-ancestor`) and whose manifest excluded the failing test's file.
- Each match is recorded as a `false_negative` entry naming the test, the manifest, and which rules fired — the rule
  histogram is what tells you whether to raise the depth or grow the contract set.

Do **not** try to correlate against GitHub Actions in this phase. It needs the manifest to travel with the change and is
a separate design; note it as a follow-up.

### `just selection-health`

Summarize the store: run count, escalation rate, median and p90 selected-file count, median scoped duration, total
worker-seconds saved versus the same runs at full-suite cost, the broadening-rule histogram, and the false-negative
count with each instance listed. This recipe's output is what the project owner reads to decide whether depth 2 is
right; make it readable, not a JSON dump.

### Exit criterion for the epic

The report's bar is "zero false negatives across at least 30 varied changes." Adopt it as the criterion for keeping the
default, and record the reading in the epic's land notes. If the rate is not zero, the response is to raise
`SASE_TEST_SELECTION_DEPTH` to 3 or add the missed tests to the contract set and re-measure — not to remove the metric.

---

## Phase `contexts` — Coverage-context ground truth for selection

This is the report's "selection mechanism, phase 2," and it is the only part of the design that can be _sound_: per-test
coverage says which tests actually executed a line, including the dynamic dispatch, plugin lookup, and config discovery
an import graph cannot see.

### Production side

Add `--cov-context=test` to the `cov` branch of `_pytest_command`. Measure the added runtime on the CI coverage leg
before and after and record it in the commit message; contexts are not free and the coverage leg is already the longest
CI job. If the cost is material, the fallback is a dedicated scheduled job rather than the per-PR leg — decide with the
measurement, not in advance.

Upload the resulting `.coverage` SQLite database as a CI artifact (`sase-coverage-contexts`) alongside the existing
coverage upload, with the commit SHA in the artifact name.

### Consumption side

Teach the engine to accept a contexts database as an alternative selection source:

- Resolve a baseline via `gh run download` into `${SASE_HOME}/test-selection/contexts/<sha>.sqlite`, cached by SHA.
- Selection from contexts: changed `src/sase/**` file → the set of test contexts that executed any changed line.
- **Union, never replace.** The static closure stays in the selection. Contexts are ground truth for the code that
  existed when the baseline was recorded; they say nothing about code added since, and a new test file has no context
  rows at all. Selection is `static_closure ∪ context_selection ∪ contract_set ∪ changed_test_files`.
- Staleness: if the baseline SHA is not an ancestor of `HEAD`, or is more than N commits behind, use it anyway but
  record `context-baseline-stale` on the manifest so `just selection-health` can show whether staleness correlates with
  false negatives.
- Absent or unreadable baseline is **not** an error — record `context-baseline-missing` and fall back to the static
  closure alone. An agent in a fresh workspace with no network must still get a working `just check`.

### Why this phase depends on `health`

Ordering it after `health` is deliberate: the false-negative histogram from real use is what says whether contexts are
worth their CI cost and where they help. Building it first would be guessing.

---

## Phase `policy` — Two-speed verification policy in SASE memory

The verification contract lives in `sase/memory/build_and_run.md`, which is rendered into `AGENTS.md`, `CLAUDE.md`,
`GEMINI.md`, `OPENCODE.md`, and `QWEN.md` by `sase memory init`.

**This phase must not begin by editing that file.** Per `gotchas`, SASE memory files may only be changed with the user's
explicit permission granted in a live conversation, and _a plan file does not grant that permission_ — including this
one. The phase agent's first action is to use its `/sase_questions` skill to ask the project owner for permission,
showing the exact proposed diff. If permission is refused or the owner wants different wording, that is a complete and
successful outcome for the phase: report it and stop.

With permission granted, the edit should say:

- `just check` is the default verification command: whole-repo lint gates plus a diff-scoped test lane that takes no
  suite-gate lease.
- `just check-full` runs every gate plus the full suite. Run it before landing an epic's combined tree, when the change
  touches the broadening set, and any time the scoped run escalated or reported an unusual selection.
- Selection is a heuristic backstopped by CI; `tools/select_tests --explain` shows why a test was or was not chosen, and
  `just selection-health` shows whether the heuristic has ever been wrong.

Then run `sase memory init` — mandatory, and explicitly not requiring a second permission request.

**While the file is open with permission in hand, also correct the stale Tier 0 text** at `build_and_run.md:12–15`,
which still claims `just test` "includes PNG visual snapshots" and that `just test-cov` "also runs the visual snapshot
suite". Tier 0 made both statements false and deliberately left them alone for exactly this permission reason.

---

## Explicitly out of scope

- **All of Tier 2**: gate fair-share, per-PR test-runtime budget, the cross-workspace result cache, stale `sase_core_rs`
  binding detection, TUI harness cost reduction.
- **Breaking the 497-module SCC.** It is the reason selection must be heuristic, and shrinking it would make this system
  strictly better — but it is a large architectural project and belongs in its own epic (see follow-ups).
- **Repo split, Pants, Bazel, testmon.** The report rejects all four; nothing measured here changes that.
- **Rewriting expensive tests.** Tier 0 left the three dismissed-bundle scale tests as-is behind `slow`; that follow-up
  is still open and still separate.
- **Correlating GitHub Actions failures with local manifests.** Noted as a follow-up in the `health` phase.

## Verification

Each phase runs `just install` first — workspaces are ephemeral and may be stale — and then `just check-full` (or
`just check` before the `check-split` phase lands). Note that `just _lint-symvision` is known-red on master for an
unrelated reason (`progress_fingerprint` in `src/sase/llm_provider/commit_finalizer_git.py`, tracked as `sase-fj`); if
that is the only symvision failure it is pre-existing — do not fix it here and do not let it mask a real one.

Phase-specific evidence to produce, beyond passing tests:

- `engine`: on at least ten recent commits, report the depth-2 selection size and confirm the median lands near the 3.8%
  measured here. A materially different median means the implementation diverges from the measured design.
- `contract`: the measured serial runtime of the committed manifest, stated in the commit message.
- `runner`: the zero-token integration test, plus one demonstration that an escalating change runs the governed full
  lane rather than a serial partial one.
- `check-split`: `just check` and `just check-full` both green on a real change; the wall-clock difference reported.
- `health`: `just selection-health` output over whatever runs exist at that point, even if the sample is small.
- `contexts`: the before/after CI coverage-leg runtime, and one selection where contexts chose a test the static closure
  missed.

## Follow-ups to propose (do not implement here)

Use `/sase_new_task` for each; do not create beads without it.

- Break up the 497-module `src/sase` import cycle. It is the root cause of static selection being unsound here, and it
  is also what the report's Tier 2 note about driving non-`ace` imports of `sase.ace` toward zero is really about.
- Correlate GitHub Actions test failures with local selection manifests, so CI — not just local `check-full` — feeds the
  false-negative metric.
- Make `tests/ace/tui/visual/conftest.py` import Pillow lazily, so `just test` and `just test-cov` can drop
  `_setup-visual` the way `just test-scoped` already does. (Carried over from Tier 0.)
- Rewrite the three dismissed-bundle scale tests to build fixtures directly and return them to the fast lane. (Carried
  over from Tier 0.)
