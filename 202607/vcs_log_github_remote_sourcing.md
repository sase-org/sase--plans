---
create_time: 2026-07-08 18:44:26
status: done
prompt: 202607/prompts/vcs_log_github_remote_sourcing.md
tier: tale
---
# Plan: Source `sase vcs log` from the GitHub remote, marking local presence

## Summary

`sase vcs log` currently builds its cross-repo commit timeline from each repo's **local** `git log HEAD`. That shows
whatever the local checkout happens to be at — stale relative to GitHub, and it silently omits commits that landed on
the remote but were never pulled.

This plan changes the timeline's source of truth to **the latest state of the GitHub remote**, sourced through the
primary repo checkouts with a minimal-footprint `git fetch` (no working-tree / index / branch changes), and it makes
every commit's **presence** explicit: is this commit in your current local checkout, only on the GitHub remote, or in
both?

The result is a `vcs log` that answers "what is actually on GitHub right now, and what am I missing (or holding locally
that isn't pushed yet)?" at a glance — intuitive, reliable, and beautiful.

---

## Problem statement

Today (see `src/sase/vcs_log/`, `src/sase/vcs_provider/plugins/_git_query_ops.py`):

- The per-repo commit list comes from `vcs_log()` → `git log --no-merges --format=... HEAD` executed in the repo
  checkout. It is **local-only**.
- Consequences:
  - Commits merged/pushed to GitHub that the local checkout hasn't fetched are invisible.
  - There is no signal distinguishing "this is on GitHub" from "this is a local commit I haven't pushed" — the two are
    rendered identically.
  - The timeline drifts from reality the moment the remote moves.

We want the timeline to reflect GitHub's current state, while clearly flagging which of those commits the local checkout
already contains — and still surfacing local commits that aren't on the remote yet (so a just-made commit never vanishes
from the log).

---

## Goals

1. Build the timeline from the remote's latest state (default branch or the current branch's remote counterpart),
   refreshed via a safe `git fetch`.
2. For every commit, mark its **presence**: `synced` (local + remote), `remote_only` (on GitHub, not in local checkout),
   or `local_only` (in local checkout, not yet on the remote).
3. Never mutate working-tree files, the index, the checked-out branch, or the stash of any repo. Only remote-tracking
   refs + the object store are touched (exactly what `git fetch` does).
4. Degrade gracefully offline / when a repo has no usable remote: fall back to local history with a clear note rather
   than failing.
5. Keep the presentation as polished as today's `pretty`/`full` output, extended with legible presence cues that survive
   `--color never` and colorblind use.
6. Preserve the existing surface: `--repo`, `--current-only`, `--author`, `--since/--until`, `--limit`, `--reverse`, and
   the `pretty|full|oneline|json` formats all keep working.

## Non-goals

- No `git pull`/rebase/checkout/branch mutation — read-only w.r.t. user work.
- No new persistence; the timeline stays computed on demand.
- No change to the `sase vcs list` command (only shares the resolver).
- Not attempting to show every remote branch — we compare against one remote ref per repo (with an override flag), not
  the full remote ref graph.

---

## Key design decisions (rationale + alternatives)

Each of these is a genuine fork; call out any you want changed at review.

### D1. Fetch mechanism: `git fetch` (recommended) vs GitHub API

**Recommendation: minimal-footprint `git fetch`.**

- Reuses the existing git-subprocess provider path and the Rust `parse_git_log` parser unchanged; identical
  author/subject/body fidelity to local commits.
- Uniform across every VCS provider (GitHub _and_ bare-git), matching the provider-abstraction philosophy — no
  GitHub-only branch.
- Answers the user's "just fetch the right commits" directly.

**Footprint (addresses "without changing any files or working state"):** a `git fetch` writes only inside `.git/` —
remote-tracking refs (`refs/remotes/origin/*`) and downloaded objects. It never touches the working tree, the
index/staging area, the checked-out branch, HEAD, or the stash. We fetch **narrowly** (only the ref(s) we compare
against) with `--no-tags --no-write-fetch-head` and no `--prune`, so nothing else in `.git` moves either.

**Alternative considered — GitHub compare/commits API (`gh api`, as used by `src/sase/updates/incoming_commits.py`):**
truly zero writes to the repo dir, but GitHub-only, bypasses the provider + Rust parser, needs owner/repo slug
resolution, and still needs local git to compute presence. Rejected as the default; `--no-fetch` (below) is the escape
hatch for anyone who wants zero network.

### D2. Which remote ref to compare against

**Recommendation:** per repo, use the current branch's remote counterpart (`@{upstream}` / `origin/<current-branch>`)
when it exists on the remote; otherwise fall back to the repo's default branch (`origin/<default>`, via the existing
`_get_default_branch`). A `--branch/--ref <ref>` flag overrides (e.g. always compare to `main`).

- Rationale: "the latest state of the GitHub repo" for what you're actually working on is your branch's pushed state;
  when your branch was never pushed, the mainline is the meaningful baseline (and all your branch commits then show as
  `local_only`, which is correct and informative).

**Alternatives:** always the default branch (simpler, but hides pushed feature-branch state and would drop most current
output for feature branches); always `@{upstream}` with no fallback (breaks for never-pushed branches).

### D3. Keep showing local-only (unpushed) commits

**Recommendation: yes** — the timeline is the **union** of the remote ref's history and local `HEAD`, each commit tagged
with presence.

- Rationale: switching to "remote only" would make a commit you _just made_ disappear from `sase vcs log` — a jarring
  regression from today's local view. The union is strictly more informative: you see GitHub's truth _and_ your
  not-yet-pushed work, unmistakably labeled.

### D4. Presence is a first-class wire field in the Rust core (schema v2)

**Recommendation:** add `presence` to `VcsCommitWire` (Rust + Python, schema bump to v2) and add a pure
`classify_commit_presence(...)` function to `sase-core`, keeping the split the codebase already uses: **git subprocess
I/O stays in the Python provider; pure data transforms (parse, classify, aggregate) stay in Rust core.**

- Rationale: per the `rust_core_backend_boundary` rule and its litmus test, the commit model and the local/remote
  classification are backend behavior any frontend (a future web/editor `vcs log`) must match, so they belong in
  `sase-core`. The actual `git fetch`/`git rev-list` calls remain Python, exactly like `git log` execution is Python
  today while `parse_git_log` is Rust.

**Alternative — Python-only presence** (attach in the service layer, leave `sase-core` untouched, schema stays v1):
smaller and single-repo, and there is prior art for a Python-only GitHub-commits feature (the Updates-tab incoming
commits, kept in Python and flagged for veto). Faster to ship, but leaves the core model incomplete and duplicates
classification if another frontend needs it. Flagging D4 explicitly as the most reviewable decision.

---

## UX design (the beautiful part)

### Presence vocabulary

| State         | Meaning                                      | Glyph | Accent             |
| ------------- | -------------------------------------------- | ----- | ------------------ |
| `synced`      | on GitHub **and** in local checkout          | `●`   | repo color (today) |
| `remote_only` | on GitHub, **not** in local checkout         | `↓`   | cyan ("incoming")  |
| `local_only`  | in local checkout, **not** on the remote     | `↑`   | amber ("unpushed") |
| `unknown`     | remote state unavailable (offline/no remote) | `·`   | dim                |

Distinction is carried by **glyph shape first**, color second, so `--color never` and colorblind users still read it.
`↑`/`↓` deliberately mirror git's ahead/behind mental model.

### `pretty` (default) — illustrative

```
  sase-org/sase (14)  ↑2 ↓1   ·   sase-core (3)      ↑ unpushed  ↓ GitHub-only  ● synced
  vs origin/master · fetched just now

  ── Today ───────────────────────────────────────────
   ↑ 14:22  a1b2c3d  sase-org/sase   Add remote-aware vcs log        · Bryan Bugyi
   ● 13:05  e4f5a6b  sase-org/sase   Refactor doctor checks          · Bryan Bugyi
   ↓ 11:30  9c8d7e6  sase-org/sase   Fix flaky snapshot (from GitHub)· Someone Else

  ── Yesterday ───────────────────────────────────────
   ● 18:40  2b3c4d5  sase-core        Tighten wire schema             · Bryan Bugyi
```

- The leading `●`/`↑`/`↓` replaces today's fixed `●` bullet; `synced` keeps the repo-colored `●` so repo identity still
  reads in the common case, while `↑`/`↓` pop in their accent colors to highlight the exceptions.
- The legend gains a compact `↑<ahead> ↓<behind>` per repo (true counts from the partition, even when they exceed
  `--limit`) plus a one-line key and the `vs <remote-ref> · fetched <when>` subheader.

### `full`

Adds an explicit presence tag word in the commit footer (e.g. `· unpushed` / `· GitHub-only`) alongside the existing
short-id/author/age line, so the state is spelled out where there's room.

### `oneline`

Prefix each row with the 1-char glyph: `↓ 9c8d7e6 sase-org/sase Fix flaky ...`.

### `json`

- Each commit gains `"presence": "synced|remote_only|local_only|unknown"`.
- Each repo entry gains `"remote_ref"`, `"ahead"`, `"behind"`, `"fetched"`.
- Add a top-level note when remote state was unavailable for any repo.

---

## Technical design

Layering mirrors today's: **provider = git subprocess I/O; `sase-core` = pure transforms; service = orchestration;
render = presentation.**

### 1. Remote-ref resolution (provider)

New provider op `vcs_resolve_remote_log_ref(cwd) -> str | None`:

- If the current branch has a remote-tracking counterpart (`origin/<current-branch>` verifies), return it.
- Else return `origin/<default-branch>` (reuse `_get_default_branch`).
- Return `None` when there's no `origin` remote / nothing resolvable (drives the `unknown` degradation).
- `--branch/--ref` overrides the resolved value.

### 2. Minimal-footprint fetch (provider)

New provider op `vcs_fetch_remote(cwd, *, refs) -> (ok, error)`:

- Runs `git fetch --no-tags --no-write-fetch-head origin +refs/heads/<b>:refs/remotes/origin/<b>` for each needed branch
  (current + default). Explicit refspec → only the named remote-tracking refs move.
- Bounded timeout; failure returns `(False, reason)` (never raises through to a crash). `--no-write-fetch-head` needs
  git ≥ 2.29; fall back to a plain narrow fetch if unsupported.
- Skipped entirely under `--no-fetch` (compare against existing, possibly stale, remote-tracking refs and label the
  output "not fetched").

### 3. Union log + ahead/behind partition (provider + core)

- Extend `vcs_log(...)` with `revs: Sequence[str] = ("HEAD",)` (backward compatible; default reproduces today's
  behavior). Remote mode passes `("HEAD", <remote-ref>)` so `git log` returns the **deduped union**, still
  `--no-merges`, still honoring `--since/--until/--author/--limit`. Only the shared `GitQueryOpsMixin` changes;
  `sase-github` does **not** override `vcs_log`, so no plugin edit is needed. Update `_base.py` + `_hookspec.py`
  signatures accordingly.
- New provider op `vcs_partition_commits(cwd, *, local_ref, remote_ref) -> (ahead_ids, behind_ids)` via
  `git rev-list --no-merges <local> ^<remote>` and the reverse. Cheap (SHA-only), and returns complete sets so the
  legend's `↑/↓` counts are accurate beyond the display limit.
- New pure `sase-core` fn `classify_commit_presence(commits, ahead_ids, behind_ids) -> [VcsCommitWire]`: stamp each
  commit `local_only` if its `full_id ∈ ahead`, `remote_only` if `∈ behind`, else `synced`. Exposed via PyO3 with a
  Python golden mirror (same pattern as `parse_git_log` / `aggregate_commit_log`).

### 4. Service orchestration (`src/sase/vcs_log/collect.py`)

Per repo, in remote mode (default):

1. `remote_ref = resolve_remote_log_ref(...)`. If `None` → local-only mode, presence `unknown`, add a note.
2. If not `--no-fetch`: `fetch_remote(refs=[current, default])`; on failure add a warning and continue with existing
   refs (stale) or fall back to `unknown`.
3. `commits = vcs_log(revs=("HEAD", remote_ref), limit=..., since/until/authors)`.
4. `ahead, behind = partition_commits(local_ref="HEAD", remote_ref=remote_ref)`.
5. `commits = classify_commit_presence(commits, ahead, behind)` (Rust).
6. Record per-repo remote state (`remote_ref`, `len(ahead)`, `len(behind)`, `fetched`).

Aggregation across repos still goes through Rust `aggregate_commit_log`, which now carries `presence` through the
flattened `AggregatedCommitWire` unchanged.

Each repo stays isolated in its own `try/except` (today's guarantee) so one unreachable/remoteless repo becomes a
warning, not a failure.

### 5. Models (`src/sase/vcs_log/models.py`)

- `Presence = Literal["synced", "remote_only", "local_only", "unknown"]`.
- New `RepoRemoteState(name, remote_ref, ahead, behind, fetched)`.
- `VcsLogResult` gains `remote_states: tuple[RepoRemoteState, ...]` for the legend/JSON. Presence itself rides on each
  commit via the wire.

### 6. Rendering (`src/sase/vcs_log/render.py`, `_style.py`)

- Map presence → glyph + style; update `_commit_line`, `_print_full_commit`, `_render_oneline`, and `_render_json` per
  the UX section.
- Extend `_legend` with `↑/↓` counts, the key, and the `vs <ref> · fetched <when>` subheader; add presence accent colors
  to `_style.py`.
- `unknown` renders like today (no `↑/↓`), plus a warning line.

### 7. CLI (`src/sase/main/parser_vcs.py`, `vcs_handler.py`)

- Add `-N/--no-fetch` (skip the fetch; compare against existing remote-tracking refs) — mirrors the spirit of
  `sase vcs list --no-fetch`.
- Add `--branch/--ref REF` to override the compared remote ref.
- Fetch is **on by default** (the whole point). Thread both flags through `run_vcs_log`. Help text updated;
  `README`/command docs refreshed if present.

### 8. `sase-core` changes (Rust)

In `crates/sase_core/src/vcs_log/`:

- `wire.rs`: add `presence` to `VcsCommitWire` (serde snake_case enum, default `unknown` for schema evolution); bump
  `VCS_LOG_WIRE_SCHEMA_VERSION` → `2`. Mirror in `src/sase/core/vcs_log_wire.py` (dataclass field with default; update
  `vcs_commit_from_dict`).
- `aggregate.rs`: no logic change — `presence` flows through the flatten; extend tests to assert it survives.
- New `classify.rs` (or fn in `mod.rs`): pure `classify_commit_presence`; wire through `crates/sase_core_py/src/lib.rs`
  (PyO3) and `src/sase/core/vcs_log_facade.py` (with the `*_python` golden).
- Update `crates/sase_core/tests/vcs_log_parity.rs` for the new field/fn.

Coordination note: this is an editable/dev cross-repo change; `just install` rebuilds the `sase_core_rs` binding from
the sibling core repo, so the Rust and Python sides must land together.

---

## Reliability & edge cases

- **Offline / fetch fails:** warn, fall back to existing remote-tracking refs (labeled "not fetched") or to `unknown`
  when there are none; never crash.
- **No `origin` / non-GitHub remote:** `vcs_resolve_remote_log_ref` returns `None` → `unknown` presence, renders like
  today with a subtle note. Bare-git remotes work identically to GitHub (uniform `git fetch`).
- **Branch never pushed:** remote ref falls back to default branch; local branch commits correctly show as `local_only`.
- **SDD separate-repo store:** same treatment; if it has no remote it renders as `unknown`.
- **Detached HEAD:** no upstream → compare against default branch.
- **Large divergence:** `rev-list` sets are SHA-only and bounded by repo size; counts may be capped in display but
  reported truthfully.
- **Ephemeral workspaces:** we operate on the **primary** checkouts (the resolver already prefers `primary_dir`), the
  stable place to fetch — matching the user's "use the primary repo directory locations."
- **Performance:** one narrow fetch + one `git log` + two `rev-list` per repo, each bounded by timeout; repos are
  independent and could be parallelized later if needed.

---

## Testing strategy

- **`sase-core` (Rust):** unit tests for `classify_commit_presence` (synced/ahead/behind/unknown, unknown-passthrough);
  `aggregate` retains presence; parity tests for the v2 wire; schema-version assertion.
- **Python core facade:** golden-parity for `classify_commit_presence`; wire rehydrate round-trips presence.
- **Provider (real git, `tests/test_vcs_provider_git_integration.py` style):** temp repo + a local bare repo as `origin`
  (hermetic, no network) exercising `vcs_resolve_remote_log_ref`, `vcs_fetch_remote` (asserts working tree / index /
  branch untouched), `vcs_partition_commits`, and `vcs_log(revs=...)` across synced / ahead / behind / never-pushed /
  no-remote scenarios.
- **Service (`tests/test_vcs_log_collect.py`):** fake provider returning canned commits + ahead/behind → assert
  presence, `RepoRemoteState`, `--no-fetch`, and graceful degradation on fetch failure.
- **Render (`tests/test_vcs_log_render.py`):** text goldens for pretty/full/oneline/json with mixed presence, empty, and
  `unknown`; verify glyphs survive `--color never`.
- **CLI/handler:** `--no-fetch`, `--branch`, exit codes, help text.
- Run `just check` (lint + mypy + tests) in `sase`, and the core test suite in `sase-core`.

---

## Suggested sequencing

1. **Core model:** `sase-core` wire v2 + `classify_commit_presence` + PyO3, and the mirrored Python facade/wire. Land
   Rust+Python together.
2. **Provider ops:** `revs` on `vcs_log`, `vcs_fetch_remote`, `vcs_resolve_remote_log_ref`, `vcs_partition_commits` (+
   hookspec/base), with integration tests.
3. **Service + models:** remote-aware `collect_vcs_log`, `RepoRemoteState`, degradation, `--no-fetch`/`--branch`
   threading.
4. **Render + CLI:** presence glyphs/legend across all four formats, flags, help.
5. **Docs + polish:** command docs, final `just check`, visual review of `pretty`/`full`.

---

## Open decisions for review

- **D1** `git fetch` (recommended) vs `gh api` for sourcing.
- **D2** compare against current branch's upstream w/ default-branch fallback (recommended) vs always the default
  branch.
- **D4** presence as a `sase-core` wire field + pure fn (recommended, boundary-correct) vs Python-only presence
  (lighter, single-repo).
- Should `synced` keep the repo-colored `●`, or should all three states use their presence accent for maximum
  consistency? (Recommended: keep repo color for `●`.)
