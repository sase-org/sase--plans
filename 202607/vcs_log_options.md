---
create_time: 2026-07-08 15:16:50
status: wip
prompt: 202607/prompts/vcs_log_options.md
tier: tale
---
# Plan: `sase vcs log` — filter, format & ordering options

## 1. Goal & product context

`sase vcs log` already renders a beautiful, day-grouped, cross-repository commit timeline (primary + linked repos + a
separate-repo SDD store). This iteration makes it a genuinely _useful daily driver_ by adding the options people reach
for the moment a timeline is more than one screen tall: **filter by time window, filter by author, choose ordering, and
choose how much detail to show** — including a new verbose format that finally surfaces the commit _body_ the pipeline
already collects but currently discards.

The north star is unchanged: **intuitive** (mirror `git log` muscle memory and the house `sase` option idioms so nothing
needs re-learning), **reliable** (filters must never silently return the wrong commits), and **beautiful** (every new
surface keeps the swimlane-colored, gold-SHA, day-grouped aesthetic).

### Why these options, and why now

The constellation view is only as good as your ability to narrow it. "What did we change this week?", "what did _this
author_ touch across all three repos?", "show me the full messages, not just subjects", and "read it oldest-first" are
the questions the current command can't answer. Each maps to a well-known `git log` flag, so there is a correct,
unsurprising answer for every one — we just need to thread it through our provider-agnostic, Rust-backed pipeline
without breaking the top-N correctness guarantee.

### The one hard design constraint (drives everything below)

The timeline's newest-N guarantee comes from git applying `-n <limit>` _during the revision walk_, and the collect layer
fetching `limit` commits **per repo** before the aggregator merges and truncates to the global top-N. Any "which
commits" filter (date, author) therefore **must be pushed down into the git query** so git applies it _before_ `-n`.
Filtering host-side after the fetch would silently drop in-window commits that sit past position `limit` — the classic
"filter-then-limit vs limit-then-filter" bug. So date/author filters extend the **provider hook contract**, exactly as
the original plan foreshadowed ("leaves room to add them later as optional abstract parameters"). Pure _display_ options
(ordering, format) stay entirely host-side.

### What this iteration deliberately does NOT touch

- **No `sase-core` (Rust) changes.** The `body` field is already in `VcsCommitWire` and the wire format (`%b`);
  date/author filters are git-query arguments; ordering/format are host-side. The wire schema, the parser, and the
  aggregator are all unchanged. This is a **single-repo change** — a major simplification versus the original two-repo
  build. (One assumption to confirm during build, not change: the aggregator already treats a **negative limit as
  "unlimited"** — the Python golden is `rows[:limit] if limit >= 0 else rows` — which we reuse for `--limit 0`.)
- No commit mutation, no diff/patch view, no branch-topology/`--all-refs` graph — all still non-goals.
- No `--grep` (message search) or `--path` (pathspec) filter this round. Both slot into the _same_ pushdown contract
  we're adding, so they become trivial follow-ups; kept out now to keep the surface curated.

## 2. Command surface (UX)

All new options attach to `sase vcs log` (and the bare `sase vcs` default), matching `git log` names so muscle memory
transfers. Existing options (`-r/--repo`, `--current-only`, `--color`) are unchanged.

| Option                                | Meaning                                                                                                             | Layer                   |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------- | ----------------------- |
| `-n, --limit N`                       | Max commits in the merged timeline (default 20). **Changed: `0` now means unlimited** (mirrors `sase bead search`). | parser + collect + hook |
| `--since, --after DATE`               | Only commits at/after `DATE`.                                                                                       | **pushdown** (hook)     |
| `--until, --before DATE`              | Only commits at/before `DATE`.                                                                                      | **pushdown** (hook)     |
| `--author PATTERN`                    | Only commits whose author name/email contains `PATTERN` (case-insensitive substring). Repeatable → OR.              | **pushdown** (hook)     |
| `--reverse`                           | Oldest-first ordering (of the selected top-N).                                                                      | display only            |
| `--format {pretty,full,oneline,json}` | Output format. **New: `full`** shows the commit body + full metadata.                                               | display only            |

`git`-parity aliases (`--after`/`--before`, `--since`/`--until`) are provided so the command reads naturally regardless
of which name the user knows; this also directly satisfies the "before/after date" phrasing in the request.

### `DATE` grammar (host-parsed, provider-neutral, deterministic)

Rather than lean on git's fuzzy approxidate (git-specific, whitespace-heavy, needs quoting), `sase vcs log` accepts a
small, **documented, testable** grammar and resolves it to an epoch second **in the configured timezone**
(`sase.core.time`) before pushing it down. Accepted forms:

- **Relative shorthand:** `<N>h`, `<N>d`, `<N>w` (e.g. `--since 2w`, `--since 36h`) — measured back from _now_.
- **Keywords:** `today`, `yesterday` (local midnight boundaries).
- **ISO:** `YYYY-MM-DD` or `YYYY-MM-DDTHH:MM` (via `datetime.fromisoformat`, interpreted in the configured tz).

Anything else → a clear error listing the accepted forms, exit `2`. This keeps date handling identical across every VCS
provider (the hook only ever sees an epoch int) and immune to git-version approxidate drift, while staying friendly and
quote-free for the common cases.

### Rendering

- **`pretty`** (default) — unchanged day-grouped compact timeline (colored `●`, dim `HH:MM`, gold short-SHA, bold
  per-repo badge, subject, dim ` · author`).
- **`full`** (new) — same day-group headers, but each commit becomes a small multi-line block that finally shows the
  **body**: a repo-colored `▌` left rule (house accent idiom), bold subject, the wrapped dim body, and a dim metadata
  footer (`short-sha · author <email> · HH:MM · Nd ago`, relative time via the
  `notifications.models.format_relative_time` style). This is the "read the actual messages" view; it is _not_ a diff
  (still a non-goal).
- **`oneline`** — unchanged pipe-friendly one-line-per-commit.
- **`json`** — unchanged shape, **plus** the active filters echoed under a `query` key (`limit`, `since`, `until`,
  `authors`, `reverse`) so machine consumers can see exactly what window produced the result.

**Self-describing filters.** When any filter is active, `pretty`/`full` print a dim filter-summary line beside the
legend (e.g. `since 2026-07-01 · author bryan`), and the empty-result message names the active filters
(`No commits found (since 2026-07-01, author bryan)`) so an empty timeline is never mysterious.

**Ordering.** `--reverse` reverses the _final_ selected top-N (newest-N chosen first by time, then displayed
oldest-first), matching `git log -n … --reverse`. Day-group headers simply run ascending.

## 3. Architecture & layered changes (single repo: `sase`)

The layering from the original build is preserved; the new work slots cleanly into it.

### Layer C — provider hook contract (the pushdown)

The `vcs_log` hook grows an **optional, provider-agnostic filter surface**; a future Mercurial/jj provider maps the same
neutral inputs to its own query. Because there is exactly one hookimpl (the shared `GitQueryOpsMixin.vcs_log`, inherited
by both bare-git and GitHub), the signature is updated in lockstep across the four layers with no per-provider
special-casing:

1. `_hookspec.py` — `vcs_log(self, cwd, limit, *, since=None, until=None, authors=())`.
2. `_base.py` — `log(...)` gains the same keyword-only params (documented contract: `since`/`until` are epoch seconds or
   `None`; `authors` is a tuple of case-insensitive substrings matched against the author identity, ORed; `limit <= 0`
   means unbounded).
3. `_plugin_manager.py` — the delegating `log(...)` threads the new kwargs to `self._pm.hook.vcs_log(...)`.
4. `plugins/_git_query_ops.py` — the git impl builds the args:
   - `limit >= 1` → `-n <limit>`; `limit <= 0` → **omit `-n`** (critical: `git log -n 0` means _zero_ commits, so
     unlimited must drop the flag, not pass `0`).
   - `since`/`until` → `--since=@<epoch>` / `--until=@<epoch>` (git's unambiguous unix-timestamp date form; no
     approxidate).
   - each `authors` entry → a repeatable `--author=<pattern>`, plus `--fixed-strings --regexp-ignore-case` so matching
     is a literal case-insensitive substring ORed across patterns (no manual regex escaping, git ANDs date filters with
     the ORed author set as usual).
   - `--no-merges` and the pinned `--format` are unchanged, so the Rust parser sees the identical stream.

### Layer D — resolution + collection service (`src/sase/vcs_log/`)

5. `models.py` — add a frozen `CommitFilters(since: int | None, until: int | None, authors: tuple[str, ...])` value
   object (the neutral "which commits" query) and an `UNLIMITED` sentinel for the limit mapping.
6. `collect.py` — `collect_vcs_log(...)` / `run_vcs_log(...)` accept the `CommitFilters` object and pass it through to
   each `provider.log(...)` call. Per-repo failure isolation is unchanged (a filter that a provider can't honor becomes
   a warning, timeline still renders). The user-facing `--limit 0` is mapped once, at the service boundary, to the
   internal unlimited sentinel used for both the git fetch (omit `-n`) and the aggregator (negative limit → all).
7. `resolve.py` — unchanged (repo _set_ resolution is orthogonal to commit _filters_).

### Layer B / A — core facade & Rust — **unchanged**

`vcs_log_wire.py`, `vcs_log_facade.py`, and everything in `sase-core` need no edits. The `full` renderer consumes the
already-present `body`; the aggregator's existing negative-limit-unlimited behavior is reused. (Build-time check:
confirm the Rust aggregator matches the Python golden's negative-limit path — it is covered by the existing parity test
— and add a one-line guard only if a gap surfaces.)

### Layer E — CLI wiring (`src/sase/main/`)

8. `parser_vcs.py` — add the new arguments to `_add_log_options` and to the bare-`vcs` `set_defaults(...)`:
   - `--limit` switches to a `nonnegative_int`-style type (mirroring `parser_bead.py`), with help "0 means unlimited".
   - `--since`/`--after` and `--until`/`--before` as aliased string args (raw string stored; resolved in the handler so
     the parser stays import-light).
   - `--author` as `action="append"` with a plural `authors` dest and `default=[]`.
   - `--reverse` as `store_true`.
   - `--format` choices extended to include `full`.
9. `vcs_handler.py` — the `log` handler resolves the raw date strings to epochs via a small tested util (invalid →
   stderr error + exit `2`), rejects an empty window early (`since > until` → clear message), builds `CommitFilters`,
   and passes `filters` + `reverse` into `run_vcs_log` / `render`.
10. A new `src/sase/vcs_log/dates.py` (or a private helper in the handler) — `parse_time_bound(str) -> int` implementing
    the grammar above against `sase.core.time.get_timezone()`; small, pure, and unit-tested in isolation.

### `render.py`

11. Add the `full` renderer (day headers reused; per-commit `▌` block with body + metadata footer) and the `--reverse`
    ordering (reverse the aggregated list before grouping). Add the dim filter-summary line and filter-aware empty
    message. The `json` renderer gains the `query` echo block. All new output honors the existing
    `_make_console`/`--color` and timezone contracts.

## 4. Reliability & edge cases

- **Top-N correctness under filtering.** All "which commits" filters are applied by git _before_ `-n`, preserving the
  newest-N-per-repo → global-newest-N guarantee. This is the single most important correctness property and the reason
  the filters extend the hook rather than post-filter host-side.
- **Unlimited is really unlimited.** `--limit 0` maps to the negative sentinel everywhere: git omits `-n` (never `-n 0`,
  which would return nothing) and the aggregator returns all rows. Naturally paired with `--since` for "everything since
  X".
- **Timezone correctness.** Date bounds are parsed in the configured tz and converted to epoch; sorting and windowing
  stay epoch-based and tz-immune; wall-clock rendering stays on `sase.core.time`. A `--since`/`--until` and the day
  headers can never disagree about "what day it is."
- **Empty / contradictory windows.** `since > until` is caught in the handler with a friendly message (exit `2`), never
  a silent empty. An empty _result_ echoes the active filters so the user sees why.
- **Author matching is neutral & predictable.** Documented as case-insensitive substring over the author identity;
  implemented with git `--fixed-strings --regexp-ignore-case` so a pattern like `a.b` or `O'Neil` is matched literally,
  not as a regex. Multiple `--author` OR together.
- **Provider-agnostic contract preserved.** The hook only ever receives epoch ints and neutral substrings — no git
  syntax leaks through the abstraction; a future provider implements the same contract in its own query language.
- **Per-repo isolation unchanged.** A repo that errors under a filter still degrades to a warning; the rest of the
  constellation renders.
- **Non-TTY / `NO_COLOR` / not-in-a-project** paths are all unchanged and continue to work for every format.

## 5. Testing strategy

- **Date grammar** (`dates.py`): unit tests for every accepted form (`Nh/Nd/Nw`, `today`, `yesterday`, ISO date &
  datetime) resolving to the expected epoch under a pinned configured tz, plus invalid-input errors.
- **Provider hook** (`test_vcs_provider_vcs_log.py`): extend the real-temp-git-repo test to assert `--since`/`--until`
  actually narrow the returned commits by time, `--author` narrows by author (and ORs across two patterns,
  case-insensitively), and `limit=0` returns _all_ commits (proving `-n` is omitted, not passed as `0`).
- **Collection** (`test_vcs_log_collect.py`): a fake provider asserts the `CommitFilters` (since/until/authors) and the
  unlimited sentinel are threaded through to `provider.log(...)`; interleave/failure-isolation behavior is unaffected.
- **Rendering** (`test_vcs_log_render.py`): golden strings (color off) for the new `full` format (body shown, metadata
  footer), `--reverse` ordering (oldest-first, ascending day headers), the filter-summary line, the filter-aware empty
  message, and the `json` `query` echo.
- **CLI parser** (`tests/main/test_vcs_parser.py`): `--since`/`--after` (and `--until`/`--before`) aliases populate the
  same dest; `--author` is repeatable; `--format full` is accepted and bad values rejected; `--reverse` sets the flag;
  `--limit 0` parses; bare `sase vcs` still defaults to `log` with the new defaults present.
- Existing `sase-core` Rust tests and the wire parity test are expected to remain green untouched (no core changes).

## 6. Build & coordination notes

- **Single repo (`sase`).** No `sase-core` edit, no binding rebuild, no `sase-core-rs` version bump. (If the build-time
  negative-limit parity check surfaces a real gap, that would be the _only_ thing pulling `sase-core` back in — not
  expected.)
- Ephemeral workspace: run `just install` before `just check`.
- Run `just check` in `sase` before completion (these changes are not in the bead/research exception categories).
- Update the `sase vcs log` `--help` text to document the new options and the `DATE` grammar; refresh any command
  reference doc that enumerates the flag set.

## 7. Deliverables checklist

- [ ] Provider hook filter pushdown: `_hookspec.py`, `_base.py`, `_plugin_manager.py`, `plugins/_git_query_ops.py`
      (`since`/`until`/`authors`, `limit <= 0` → omit `-n`).
- [ ] `src/sase/vcs_log/`: `CommitFilters` + `UNLIMITED` in `models.py`; filter threading in `collect.py`; new
      `dates.py` grammar parser.
- [ ] `render.py`: `full` format, `--reverse`, filter-summary line, filter-aware empty message, `json` `query` echo.
- [ ] CLI: `parser_vcs.py` (new options + defaults, `--limit` 0-unlimited), `vcs_handler.py` (date resolution,
      empty-window guard, `CommitFilters` construction).
- [ ] Tests across date grammar, provider hook, collection, rendering, and CLI parsing.
- [ ] `--help`/docs refresh; `just check` green.
