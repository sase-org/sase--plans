---
create_time: 2026-07-08 18:35:00
status: done
prompt: 202607/prompts/fix_failing_ci_jobs.md
tier: tale
---
# Fix Failing GitHub Actions Jobs (sase repo)

## Problem

GitHub Actions is red on `sase-org/sase` master. The most recent completed CI run (`CI #9485`) plus the `Deploy Docs`
workflow fail across four jobs. `actstat` and the per-job logs show four independent root causes. The `build`,
`phase7-perf-floor`, `visual-test`, `fmt-md-check`, `install-smoke`, and all three `test (3.12/3.13/3.14)` jobs pass, so
scope is limited to the four failing jobs below.

## Root-cause diagnosis

### 1. `bead-backend` — "Run Rust bead checks" (rustfmt on the Rust core) — ALREADY FIXED

This job runs `just rust-check`, which `cd`s into the checked-out `sase-core` and runs `cargo fmt --all -- --check`. It
failed on a formatting diff in `crates/sase_core_py/src/lib.rs` (the `core_aggregate_commit_log` binding added by the
new "unified VCS commit-log parser and aggregator" work).

The sase CI's "Check out Rust core" step checks out `sase-org/sase-core` with **no pinned ref**, i.e. the default branch
(master) HEAD. The formatting was already fixed on `sase-core` master (commit `chore: Format with cargo fmt`), and
`cargo fmt --all -- --check` is now clean there. **No change is needed in the sase repo for this job — it resolves on
the next CI run.** This item is documented here only so the job is not mistaken for an outstanding sase-repo defect.

### 2. `lint` — mypy type-var errors (needs a fix in this repo)

`just lint` fails mypy with:

```
src/sase/llm_provider/_subprocess_stream.py:66: error: Value of type variable "_R" of "select" cannot be "object"  [type-var]
src/sase/llm_provider/_subprocess_plain.py:94:  error: Value of type variable "_R" of "select" cannot be "object"  [type-var]
```

Both modules build a `readable: list[object] = []`, append `process.stdout` / `process.stderr` to it, and pass it to
`select.select(readable, [], [], 0.1)`. mypy's stub for `select.select` constrains its element type variable to a
file-descriptor-like bound; `object` does not satisfy that bound, so the call is rejected. (This surfaced from updated
type stubs — the previously-accepted `list[object]` annotation is now too broad.)

Both functions type their subprocess as `subprocess.Popen[str]`, so `process.stdout` and `process.stderr` are `IO[str]`,
and the streams are appended only inside `if process.stdout:` / `if process.stderr:` guards. `IO` is already imported in
both files.

### 3. `launch-perf-floor` — missing perf baseline fixture (needs a fix in this repo)

`just launch-perf-check` runs `tests/perf/check_agent_launch_regression.py`, which reads its baseline from
`DEFAULT_BASELINE_PATH = <repo>/sdd/tales/202605/perf_artifacts/agent_launch_phase1_baseline.json` and dies with
`FileNotFoundError`.

Cause: the commit "Remove migrated in-tree SDD files" deleted the entire in-tree `sdd/` tree as part of migrating SDD
content into the gitignored `.sase/sdd/` location (only `sdd/research/202607/` remains tracked). The launch baseline is
a genuine **committed CI test fixture**, not SDD plan content, but it lived under `sdd/tales/...` and was swept out with
the migration. Fresh CI checkouts therefore no longer contain it. (The Phase-7 test that reads a sibling fixture
survives only because it calls `pytest.skip()` when the file is absent; the launch regression check has no such guard
and hard-fails.)

The working-tree copy under `.sase/sdd/tales/202605/perf_artifacts/agent_launch_phase1_baseline.json` is byte-identical
to the last committed version
(`git show <removal-commit>^:sdd/tales/202605/perf_artifacts/agent_launch_phase1_baseline.json`), so the fixture content
is recoverable exactly.

### 4. `docs-build` / `Deploy Docs` — stale docs deploy-artifact assertions (needs a fix in this repo)

`docs-build` runs `just docs-check && just docs-pdf-check && just docs-deploy-artifact-check`; the `Deploy Docs`
workflow runs the same `docs-deploy-artifact-check`. The mkdocs build and PDF succeed — only the artifact assertions
fail.

Cause: the commit "docs: publish SASE launch post" reworked the published blog set — published
`structured-agentic-software-engineering` (`draft: false`), set `hello-sase-your-first-15-minutes` and
`why-coding-agents-need-orchestration` back to `draft: true`, deleted the `docs/series/agentic-software-engineering.md`
page, and pointed both the homepage (`docs/index.md`) and `docs/_redirects` at the launch post — but it did **not**
update the `docs-deploy-artifact-check` recipe in the `Justfile`. The recipe still asserts the previous state:

- expects `site/blog/posts/hello-sase-your-first-15-minutes/index.html` and
  `site/blog/posts/why-coding-agents-need-orchestration/index.html` (both now drafts → not built),
- expects exactly `2` published post directories (now `1`),
- expects `site/series/agentic-software-engineering/index.html` (page deleted → now a redirect),
- greps the homepage for a link to `hello-sase-...` (homepage now links `structured-agentic-software-engineering`).

The current, intended published set is a single post: `structured-agentic-software-engineering`.

## Proposed changes

### A. mypy fix (`lint`)

In both `src/sase/llm_provider/_subprocess_stream.py` and `src/sase/llm_provider/_subprocess_plain.py`, change the
loop-local declaration `readable: list[object] = []` to `readable: list[IO[str]] = []`. No runtime behavior changes;
`IO` is already imported in both files.

### B. Relocate the perf baseline fixture (`launch-perf-floor`)

Move the baseline to a tracked, migration-safe home co-located with the perf test code:

- Add `tests/perf/agent_launch_phase1_baseline.json`, restored byte-for-byte from the last committed version (via
  `git show <removal-commit>^:...`, verified identical to the `.sase` working copy).
- Update `DEFAULT_BASELINE_PATH` in `tests/perf/check_agent_launch_regression.py` to
  `REPO_ROOT / "tests" / "perf" / "agent_launch_phase1_baseline.json"`.
- Leave `DEFAULT_REPORT_PATH` unchanged. The report is a generated output: `main()` already
  `mkdir(parents=True, exist_ok=True)`s its parent, it is gitignored, and CI uploads it from
  `sdd/tales/202605/perf_artifacts/agent_launch_regression_check.json`. Keeping it avoids touching `.gitignore` and the
  CI upload path.
- Update `tests/perf/test_agent_launch_regression.py`: the `test_default_artifact_paths_live_under_sdd_tales` assertion
  must reflect the new baseline location (baseline under `tests/perf/`, report still under `sdd/tales/...`). Rename the
  test to match its new meaning.
- Update the two references to the baseline path in `docs/rust_backend.md` to the new
  `tests/perf/agent_launch_phase1_baseline.json` location.

Rationale for `tests/perf/` over re-adding under `sdd/`: the fixture is test input, and `sdd/` in-tree content is being
actively phased out into gitignored `.sase/sdd/`. Re-adding under `sdd/tales/...` would fix CI today but invite the same
regression on the next SDD cleanup. Homing the fixture with its test keeps it durable.

### C. Align the docs deploy-artifact check (`docs-build` / `Deploy Docs`)

Rewrite `docs-deploy-artifact-check` in the `Justfile` to assert the current published blog structure:

- Keep the site/PDF sanity checks (`site/index.html`, `site/_headers`, `site/downloads/sase-handbook.pdf` + `%PDF`
  magic, `site/blog/index.html`).
- Assert the single published post is built and linked from the homepage:
  `site/blog/posts/structured-agentic-software-engineering/index.html` exists, exactly `1` published post directory, and
  the homepage links `blog/posts/structured-agentic-software-engineering/`.
- Assert the drafted/removed pages are **not built**: `site/blog/posts/hello-sase-your-first-15-minutes`,
  `site/blog/posts/why-coding-agents-need-orchestration`, and `site/series/agentic-software-engineering` directories are
  absent. Do **not** add `hello-sase-...`/`why-coding-agents-...` to the "no references anywhere" grep loop — they
  legitimately appear in `site/_redirects` as 301 targets, so only their built page directories should be asserted
  absent.
- Keep the fully-hidden draft slug loop (`xprompts-in-depth`, `axe-background-daemon`, `beads-and-sdd`,
  `commit-workflows-plugins`, `changespecs-in-practice`, `telegram-mobile-agents`, `prompt-widget-and-nvim`,
  `whats-next-memory-mobile-web`) that asserts both "not built" and "not referenced anywhere in `site/`".

The exact recipe text must be validated against a real build (see Verification) and adjusted if any assertion is off —
the goal is that the recipe passes against the site mkdocs actually produces from the current frontmatter/redirects, not
a hand-guessed shape.

## Files to change

- `src/sase/llm_provider/_subprocess_stream.py` (mypy annotation)
- `src/sase/llm_provider/_subprocess_plain.py` (mypy annotation)
- `tests/perf/agent_launch_phase1_baseline.json` (new; restored fixture)
- `tests/perf/check_agent_launch_regression.py` (`DEFAULT_BASELINE_PATH`)
- `tests/perf/test_agent_launch_regression.py` (path assertion + test name)
- `docs/rust_backend.md` (2 baseline-path references)
- `Justfile` (`docs-deploy-artifact-check` recipe)

No change needed for `bead-backend` (resolved via the Rust core repo).

## Verification

Run from the workspace (recall: `just install` first, because ephemeral workspaces may have stale deps):

1. `just install`
2. `just lint` — mypy clean (both `_subprocess_*` errors gone).
3. `just launch-perf-check` — passes (baseline found, gates green).
4. `just test` — the `tests/perf/test_agent_launch_regression.py` unit tests pass with the updated path assertion.
5. Build docs and run the deploy check the way CI does:
   `just docs-check && just docs-pdf-check && just docs-deploy-artifact-check` — green. Iterate on the recipe text until
   it passes against the real `site/`.
6. `just check` — overall lint + type + fast tests clean.

Note: the `bead-backend` rustfmt job cannot be re-exercised from this repo alone; it is confirmed green by running
`cargo fmt --all -- --check` on `sase-core` master, which is already clean.

## Out of scope / notes

- Do not "fix" the Phase-7 perf fixture (`bench_status_state_machine_phase4a.json`), also removed by the SDD migration:
  its test `pytest.skip()`s when absent, so it is not a CI failure. Relocating it is optional cleanup, not part of this
  fix.
- The Node 20 deprecation warnings and the transient GitHub Actions cache-service warnings in the logs are non-fatal and
  unrelated to the failures.
