---
tier: epic
title: Make `sase bead show` much faster
goal: "`sase bead show` returns in well under a second instead of ~1.8 s (and ~3.2 s for beads that carry refs), with
  byte-identical output at every format, style, and wrap setting, by removing a 418-call subprocess storm, the full-CLI
  parser import, and two redundant full bead-store reductions.

  "
phases:
  - id: inventory
    title: Stop re-probing git remotes and re-merging config in repo inventory
    depends_on: []
    size: medium
    description: "inventory: memoize the per-primary git-origin probe and the sidecar identity/config derivation that
      collect_repo_inventory recomputes hundreds of times per command, with an explicit reset hook for the long-lived
      ACE TUI, and guard it with a subprocess-count regression test.

      "
  - id: parser
    title: Build only the invoked command's subparser
    depends_on: []
    size: medium
    description: "parser: give create_parser an opt-in only= hint so a normal sase <cmd> run registers just that
      command's subparser, drive it from a single shared command registry so the full and narrow paths cannot drift, and
      stop parser_artifact from importing the heavy artifact facade for one tuple of argparse choices.

      "
  - id: store
    title: Resolve bead detail from one bead-store read
    depends_on: []
    size: medium
    description: "store: add a single-pass bead detail read to the Rust core and its binding so sase bead show reduces
      the event store once instead of three times, and route the Python detail resolver through it.

      "
  - id: guard
    title: End-to-end budget guard and documentation
    depends_on:
      - inventory
      - parser
      - store
    size: small
    description:
      "guard: assert the combined end-to-end cost ceiling and the output-identity invariant across formats, then record
      the new performance characteristics in the changelog and bead docs."
proposed_by: bbugyi200.athena.sl.f1
create_time: 2026-08-03 08:39:40
status: wip
---

- **PROMPT:**
  [prompts/202608/bead_show_speed.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/bead_show_speed.md)

# Plan: Make `sase bead show` much faster

## Problem

`sase bead show` is the most-used bead read command in this repo, and it is the slowest. Measured on this workspace with
`hyperfine -w 2 -m 8`:

| Command                                   | Mean        |
| ----------------------------------------- | ----------- |
| `sase bead show sase-bv`                  | **1.841 s** |
| `sase bead show sase-cl` (a bead w/ refs) | **3.184 s** |
| `sase bead show sase-bv --format compact` | 903 ms      |
| `sase bead blocked` (Rust fast path)      | 400 ms      |
| `python -c pass`                          | 17 ms       |

A read that renders 42 lines of already-stored text takes 4.6× longer than the sibling bead command that goes through
the Rust fast path, and 105× bare interpreter startup.

### Where the time actually goes

Measured in-process with `time.perf_counter`, not `cProfile` — the profiler inflates the pure-Python YAML parsing by
roughly 3× and would send the implementer after the wrong line. For `sase bead show sase-bv`:

| Phase                                           | ms      | %     |
| ----------------------------------------------- | ------- | ----- |
| `resolve_bead_creator_url()`                    | **906** | 52.6% |
| `import sase.main.parser`                       | **364** | 21.1% |
| `resolve_issue_detail()`                        | 189     | 11.0% |
| `get_read_view()`                               | 106     | 6.2%  |
| `view.show()`                                   | 74      | 4.3%  |
| `create_parser()` + `parse_args()`              | 57      | 3.3%  |
| `render_issue_detail()` + page URL + plan roots | 27      | 1.6%  |

Four independent root causes account for essentially all of it.

#### 1. A 418-call subprocess storm to render one hyperlink (906 ms)

`resolve_bead_creator_url` (`src/sase/bead/cli_detail_context.py:58`) exists to print a single line:

```
  → https://github.com/sase-org/sase--agents/blob/main/families/<agent>.md#member-plan
```

It reaches `_resolve_agents_remote` (`src/sase/sdd/hosted_links.py:312`), which calls `resolve_sync_targets((project,))`
→ `collect_repo_inventory(root, project=...)`. Collecting an **11-record** inventory for one project calls
`merged_sidecar_entries_from_config` **13** times, which calls `resolve_sidecar_repo_identity` **209** times, which
calls `_github_repo_identity_from_origin` (`src/sase/_linked_repo_identity.py:140`) **418** times — and that function
shells out every single time:

```
$ # subprocess.run calls during one `sase bead show`
TOTAL subprocess.run calls: 431
  418  git remote get-url origin
    7  git config --get remote.origin.url
    ...
```

All 418 run `git remote get-url origin` in the **same** directory and return the **same** answer. `resolution_config` is
likewise recomputed 13 times, re-merging and re-parsing the YAML config layers on each pass.

This is not specific to the creator URL. `artifact_reference_context()` (`src/sase/artifact_ref_context.py:59`) hits the
same `collect_repo_inventory` path, so **a bead that has any `refs` pays the whole cost a second time** — measured at
**999 ms** on `sase-cl`, which is exactly why that bead takes 3.18 s.

#### 2. The whole CLI parser is imported to parse one subcommand (364 + 57 ms)

`sase bead show` is deliberately excluded from the Rust fast path (`src/sase/main/bead_fast_path.py:35`), so it falls
through to `create_parser()`, which registers **every** `sase` subcommand. That import is dominated by a single line:

```python
# src/sase/main/parser_artifact.py:7
from sase.core.artifact_file_facade import ARTIFACT_FILE_KINDS
```

`ARTIFACT_FILE_KINDS` is a tuple of strings used only as argparse `choices=` (lines 71, 152, 256). It is _defined_ in
the light `src/sase/core/artifact_file_types.py:18`, but importing it through the facade drags in `artifact_consumption`
→ `agent.identity` → `agent.multi_prompt` → `xprompt`:

```
python -c "from sase.core.artifact_file_types  import ARTIFACT_FILE_KINDS"   →  63.5 ms
python -c "from sase.core.artifact_file_facade import ARTIFACT_FILE_KINDS"   → 447.8 ms
python -c "import sase.main.parser"                                          → 449.5 ms
```

The facade import alone accounts for practically the entire parser import. But even after fixing it, building the full
tree still costs 206 ms of imports plus 53 ms of `add_parser` calls, because `parser_vcs` (65 ms) and `parser_ace` (55
ms) remain. Building **only** the `bead` subparser instead:

```
import parser_bead only:   7.5 ms
build bead subparser:     16.6 ms
parse_args:                0.2 ms
TOTAL:                    24.4 ms      # vs 259 ms for the full tree
```

and it parses the identical namespace (`bead show sase-bv plain 120 full`).

#### 3. The bead store is fully reduced three times per command (220 ms)

Every Rust read entry point calls `read_store_issues` from scratch (`crates/sase_core/src/bead/read.rs:88,116,142` in
`sase-core`); nothing is shared within a command. One `sase bead show` therefore performs three complete reductions of a
539-stream, 8.5 MB event store:

```
Rust store reads in ONE bead show:
  show                   1 calls     73.0 ms
  get_epic_children      1 calls     54.5 ms
  list_issues            1 calls     92.3 ms
  TOTAL                  3 calls    219.8 ms
```

`list_issues` is the worst offender: `resolve_issue_detail` (`src/sase/bead/cli_detail_resolution.py:126-130`) loads
**all 2691 issues** purely to compute the reverse-dependency (`blocks`) list. One read of the store costs 83 ms, so two
of those three reductions are pure waste.

#### 4. All of it is recomputed, not cached

None of this survives the process, and none of it needs recomputing _within_ a process either. The command is slow not
because it does hard work, but because it does the same easy work hundreds of times.

## Goal

Get `sase bead show` under **600 ms** for a typical bead — the modeled budget below lands near 400 ms — and bring
ref-bearing beads like `sase-cl` from 3.18 s into the same range, without changing a byte of output.

### Modeled post-fix budget

| Component                    | ms       | Source                       |
| ---------------------------- | -------- | ---------------------------- |
| Python startup               | 17       | measured                     |
| Thin parser (import + build) | 24       | measured                     |
| `get_read_view()`            | 106      | measured (unchanged)         |
| One store reduction          | 83       | measured (`bead_read_store`) |
| `resolve_bead_creator_url()` | ~99      | measured after memoization   |
| Page URL, plan roots, render | ~20      | measured                     |
| Post-command refresh hook    | ~15      | measured in-context          |
| **Total**                    | **~364** | **≈ 5× faster than 1.841 s** |

## The central invariant

This is a pure performance change. The keystone safety property, which every phase must hold and the final phase must
assert:

```
for every bead B, format F, style S, and wrap W:
    render(B, F, S, W) is byte-identical before and after this epic
```

This was already validated against the fixes in phases `inventory` and `parser`, using an in-process simulation that
left the real modules intact (a lazy shim, so genuine facade consumers still loaded the heavy module):

```
sase-bv: IDENTICAL (exit 0)
sase-cl: IDENTICAL (exit 0)
```

with the following measured effect, at 433 → 16 subprocesses:

| Command                  | Before  | Simulated | Speedup |
| ------------------------ | ------- | --------- | ------- |
| `sase bead show sase-bv` | 1.841 s | 1.064 s   | 1.73×   |
| `sase bead show sase-cl` | 3.184 s | 1.710 s   | 1.86×   |

That simulation covered only the memoization and the light-import fix. The `store` phase adds the remaining ~160 ms, and
the `parser` phase's structural half (the `only=` hint, measured at 259 ms → 24 ms in isolation) was not part of the
end-to-end simulation.

### Deliberate non-goals

- **No output changes.** Not one column, not one escape byte, at any `--format`/`--style`/`--wrap`.
- **No routing `show` through the Rust CLI fast path.** `crates/sase_core/src/bead/cli.rs:170`'s `handle_show` is a
  parity renderer that lacks prose wrapping, the styling palette, artifact refs, creator/page URLs, and plan links.
  Reaching parity is a separate project; `src/sase/main/bead_fast_path.py:35` keeps `show` on the Python renderer
  deliberately, and this epic does not change that.
- **No cross-process or on-disk caching.** Every fix here memoizes work that is already repeated inside a single
  command. On-disk caches buy invalidation bugs for a win we do not need.
- **No change to what `sase bead show` computes.** The creator URL, page URL, and reverse-dependency list are all still
  computed; they just stop being computed the slow way.
- **No speculative micro-optimization.** Anything not on the measured budget table above stays untouched.

---

## Phase `inventory` — Stop re-probing git remotes and re-merging config in repo inventory

**Files:** `src/sase/_linked_repo_identity.py`, `src/sase/_linked_repo_config.py`, `src/sase/_linked_repo_paths.py`,
plus the ACE inventory refresh call site.

### The fix

Add a process-lifetime memo in front of the three derivations that are recomputed from identical inputs:

1. `_github_repo_identity_from_origin(primary, *, config)` — keyed on the resolved `primary` path and the configured
   GitHub-host set. This is the one that shells out; memoizing it collapses 418 subprocesses to 1.
2. `resolution_config(primary_workspace_dir, config)` — keyed on `primary_workspace_dir`, and **only** when the caller
   passed `config=None`. An explicitly supplied `config` must keep being honored verbatim, never served from cache.
3. `merged_sidecar_entries_from_config(config, *, primary_workspace_dir)` and `_sdd_sidecar_repo_dirnames` — keyed on
   `primary_workspace_dir` plus a stable token for the config. This is what removes the 13×/209× recomputation itself,
   rather than only making each repetition cheap.

Memoizing (1) and (2) alone was measured at 906 ms → 376 ms with 421 → 4 subprocesses; (3) removes the remaining
redundant derivation.

### Invalidation — the part that needs care

`collect_repo_inventory` is **not** called only from short-lived CLI processes. The ACE TUI calls it from
`src/sase/ace/tui/modals/project_inventory_panes.py:517` and `src/sase/ace/tui/modals/project_inventory_counts.py:51`,
and that process lives for hours. An unbounded process-lifetime memo would make the repo-inventory pane show a stale
remote after someone re-points one.

Follow the convention already in this repo — `functools.lru_cache` with an explicit `cache_clear` escape hatch, as in
`src/sase/xprompt/project_identity.py:51` — and:

- Expose one public `reset_repo_identity_caches()` that clears all three memos together.
- Call it from the ACE repo-inventory refresh path, so an explicit user refresh is always authoritative.
- Do **not** derive an mtime token from `.git/config`. `.git` is a _file_, not a directory, in worktrees, and a
  worktree's config lives in the common dir — a naive stat is wrong in exactly the layouts SASE uses most.

### Regression guard

A timing test would be flaky; assert the **structure** instead. Add a test that performs a full `sase bead show` render
with `subprocess.run` patched to count invocations, asserting `git remote get-url origin` runs **at most twice** (down
from 418). Add a second test asserting that `reset_repo_identity_caches()` makes the next probe re-run, so the escape
hatch cannot silently rot.

### Verification

```bash
sase bead show sase-bv --style plain > /tmp/before.txt      # capture BEFORE editing anything
sase bead show sase-cl --style plain > /tmp/before_cl.txt
# ...implement...
sase bead show sase-bv --style plain | diff - /tmp/before.txt      # must print nothing
sase bead show sase-cl --style plain | diff - /tmp/before_cl.txt   # must print nothing
hyperfine -w 2 -m 8 'sase bead show sase-bv' 'sase bead show sase-cl'
```

`sase-cl` is the important one: it exercises the second `collect_repo_inventory` call through
`artifact_reference_context()` and should drop by roughly a full second on this phase alone.

---

## Phase `parser` — Build only the invoked command's subparser

**Files:** `src/sase/main/parser.py`, `src/sase/main/entry.py`, `src/sase/main/parser_artifact.py`.

### Part 1: the one-line import fix

```python
- from sase.core.artifact_file_facade import ARTIFACT_FILE_KINDS
+ from sase.core.artifact_file_types import ARTIFACT_FILE_KINDS
```

The facade's re-export must stay: `src/sase/artifact_ref_context.py:15` and other real consumers import genuinely heavy
names from it, and removing the re-export would break them. This change alone takes `import sase.main.parser` from 449
ms to 269 ms and benefits every `sase` command, `sase -h`, and shell completion.

### Part 2: the structural fix

Give `create_parser` an opt-in narrowing hint:

```python
def create_parser(*, only: str | None = None) -> argparse.ArgumentParser: ...
```

- `only=None` keeps **exactly** today's behavior: the full tree, every subcommand registered. All 576 existing test
  references to `create_parser` keep passing untouched, and that default is what makes the change safe.
- `only="bead"` registers the root flags, the subparsers object, and just that one registration function.

Drive both paths from **one** registry so they cannot drift:

```python
_COMMAND_REGISTRARS: dict[str, Callable[[Any], None]] = {
    "ace": register_ace_parser,
    ...
    "artifact-file": register_artifact_parser,   # alias, see parser_artifact.py:24
}
```

The full path iterates the registry in its current alphabetical order; the narrow path looks up one entry. Add a test
asserting that the set of commands the full parser exposes equals the registry's key set, so a newly added command
cannot be wired into one path and missed by the other. `artifact-file` is currently the only alias
(`src/sase/main/parser_artifact.py:24`), and `src/sase/main/entry.py:59` already treats it as `artifact`.

`entry.main()` computes the hint and **falls back to the full tree** whenever narrowing could change behavior:

| `sys.argv[1]`                     | Parser built | Why                                             |
| --------------------------------- | ------------ | ----------------------------------------------- |
| a registry key (e.g. `bead`)      | narrow       | the fast path                                   |
| missing, or starts with `-`       | full         | `sase`, `sase -h`, `sase -H` list every command |
| not a registry key (e.g. `bogus`) | full         | the usage error must still list every command   |

That table is this phase's correctness contract; test every row. In particular, assert that `sase bogus`, `sase --help`,
`sase -H`, and bare `sase` produce byte-identical stdout, stderr, and exit codes to today.

One implementation note: the registration functions are imported at module scope in `parser.py` today, so narrowing must
also defer those imports — import the selected registrar inside `create_parser` — or the import cost returns even when
only one subparser is built.

### Regression guard

Assert that dispatching `sase bead show` never imports an unrelated parser module: run the entry point in a subprocess
and assert `sase.main.parser_ace` and `sase.main.parser_vcs` are absent from `sys.modules` afterward. That is
deterministic, and it catches a regression no timing threshold would.

### Verification

```bash
hyperfine -w 3 -m 10 'python -c "import sase.main.parser"'   # ~449 ms → ~100 ms
sase --help; sase -H; sase bogus; echo "exit=$?"             # unchanged output and exit codes
sase bead show sase-bv --style plain | diff - /tmp/before.txt
```

---

## Phase `store` — Resolve bead detail from one bead-store read

**Files:** `crates/sase_core/src/bead/read.rs`, the core's public export surface, `crates/sase_core_py/src/lib.rs`,
`crates/sase_core/tests/bead_read_parity.rs` (all in the linked `sase-core` repo), then
`src/sase/core/bead_read_facade.py` and `src/sase/bead/cli_detail_resolution.py` here.

This phase crosses the Rust core boundary, which is correct per `CLAUDE.md`: "if a web app, CLI, editor integration, or
another frontend would need the behavior to match the TUI, treat it as core backend logic." Reverse-dependency
resolution is exactly that. Open the checkout with `/sase_repo` before touching it.

### The fix

`read.rs` already separates I/O from computation — `show_issue_in_issues`, `get_epic_children_in_issues`, and
`list_issues_in_issues` all operate on an already-loaded `Vec<IssueWire>`. Only the public entry points re-read. So add
one entry point that reads once and answers everything `resolve_issue_detail` needs:

```rust
pub fn show_issue_detail(
    beads_dir: &Path,
    issue_id: &str,
) -> Result<BeadIssueDetailWire, BeadError>
```

returning the resolved issue, its epic children, its resolved dependency issues, and the issues that depend on it (the
`blocks` set), all computed from a single `read_store_issues_with_resolver` call.

Computing `blocks` in Rust also removes the `list_issues()` call that currently materializes all 2691 issues into Python
just to scan their dependency lists.

Then export it, add the `#[pyo3(name = "bead_show_issue_detail")]` binding next to `bead_get_epic_children`
(`crates/sase_core_py/src/lib.rs:3974`), add a `show_issue_detail` wrapper in `src/sase/core/bead_read_facade.py`
alongside `get_epic_children` (line 152), and rewrite `resolve_issue_detail`
(`src/sase/bead/cli_detail_resolution.py:106`) to consume it.

`resolve_issue_detail` keeps its exact signature and `IssueDetail` return type; only its body changes. Its other caller
path (`_render_list_full` / `_render_search_full`) therefore benefits with no edit.

### Ordering and parity

Preserve the current ordering semantics exactly: `phases` and `child_epics` are partitioned from `get_epic_children` in
store order, `depends_on` follows `issue.dependencies` order, and `blocks` follows `list_issues()` order. Unresolvable
IDs must still degrade to the `_unresolved_ref` placeholder rather than raising — that is why `depends_on` and `blocks`
currently sit inside `try`/`except KeyError`.

Extend `crates/sase_core/tests/bead_read_parity.rs` with a case asserting the new call returns the same issue, children,
dependencies, and blockers as the existing per-query functions on the same fixture store. That parity test is what lets
a reviewer trust collapsing three reads into one.

### Verification

```bash
# in the sase-core checkout opened via /sase_repo
cargo test -p sase_core bead_read
# back in this workspace
just install                       # rebuilds the local binding
sase bead show sase-bv --style plain | diff - /tmp/before.txt
sase bead show sase-cl --style plain | diff - /tmp/before_cl.txt
```

Also verify on a bead that actually has dependents, so the `blocks` path is exercised rather than assumed: find one via
`sase bead show <id> --format json` and confirm a non-empty `blocks` array both before and after.

---

## Phase `guard` — End-to-end budget guard and documentation

Depends on all three fixes.

### The identity assertion

Add a test that renders a corpus of beads — one with `refs`, one with a `parent`, one with dependents, one closed with a
resolution, and one with `+1` evidence — across `--format {full,compact,json}` × `--style {plain,rich}` ×
`--wrap {none,80,120}`, comparing against goldens. Where `tests/test_bead/golden/cli/show*.stdout` already covers part
of that matrix, extend rather than duplicate. **No golden byte may change in this epic**; a golden that needs editing
means a phase broke the invariant, and the fix is the code, not the fixture.

### The budget assertion

Assert the ceiling in a way that will not flake in CI. Prefer structural budgets over wall-clock ones:

- total `subprocess.run` invocations during one `sase bead show` ≤ a small constant,
- full bead-store reductions during one `sase bead show` == 1,
- `sase.main.parser_ace` and `sase.main.parser_vcs` absent from `sys.modules` after dispatch.

If a wall-clock assertion is added at all, give it a generous ceiling — the modeled budget is ~364 ms, so assert against
something like 900 ms — and make it skippable on loaded runners. A tight timing threshold in a suite that already
carries known contention flakes would be a liability, not a guard.

### Documentation

- `CHANGELOG.md`: record the speedup with the measured before/after numbers.
- `docs/beads.md`: note in the `sase bead show` section that detail resolution is a single store read, and update any
  performance characteristics it documents.
- Call out in the changelog that `create_parser(only=...)` speeds up **every** `sase` command, not just `bead show` —
  that is the broadest user-visible effect of this epic.

### Final verification

```bash
just install
just check
hyperfine -w 3 -m 10 'sase bead show sase-bv' 'sase bead show sase-cl' 'sase bead blocked'
```

Report the final measured means against the 1.841 s / 3.184 s baseline in the PR description.

---

## Follow-ups

File as `task` beads via `/sase_new_task` rather than widening this epic:

- `src/sase/main/bead_fast_path.py:9` imports `sase.bead.mutation_commit` at module scope (~108 ms of transitive
  imports) purely for `_apply_mutation_side_effects`, so every **read** fast-path command (`blocked`, `ready`,
  `dep list`) pays it for nothing. `sase bead show` would not benefit — it loads `sase.bead` through the renderer
  anyway, which is why this is not a phase — but `sase bead blocked`'s 400 ms would.
- `sase.main.parser_vcs` (65 ms, via `sase.vcs_log.dates`) and `sase.main.parser_ace` (55 ms, via
  `sase.ace.saved_queries`) are heavy for what registration needs. The `only=` hint routes around them for most
  commands, but `sase -h` and completion still pay.
- `yaml.safe_load` uses the pure-Python loader although `yaml.CSafeLoader` is available in this environment (10.06 ms →
  1.70 ms per load of the 9.7 KB user config). A repo-wide switch is a small change with a broad win.
- `crates/sase_core/src/bead/cli.rs:170`'s `handle_show` parity renderer trails the Python renderer by prose wrapping,
  the styling palette, and now single-pass detail resolution; decide whether to port it or delete the deferred parity
  path. (Already noted as a follow-up by the prose-wrap tale.)
- `bead_read_legacy_jsonl` reads the 3.0 MB projection in 38.5 ms versus 83 ms to reduce the 8.5 MB event store. Whether
  reads can safely prefer an up-to-date projection is worth investigating on its own.
