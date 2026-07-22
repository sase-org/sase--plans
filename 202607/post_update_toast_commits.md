---
tier: tale
title: Post-restart update toast — commits grouped by repo
goal: 'After a ,U update restarts ACE, the post-restart "update complete" toast lists
  the commits the update actually pulled in, grouped by repository and styled to match
  the y/n confirmation panel, captured reliably from the real merge ranges.

  '
create_time: 2026-07-22 09:18:15
status: wip
---

- **PROMPT:** [202607/prompts/post_update_toast_commits.md](prompts/post_update_toast_commits.md)

# Plan: Post-restart update toast — commits grouped by repo

## Context & motivation

When the user updates SASE (via the `,U` keymap, or `sase update`), ACE writes a one-shot **update receipt** to
`~/.sase/pending_update_toast.json`, restarts the TUI, and — once the new code is running — reads-and-clears that
receipt to show a "welcome back / update complete" toast (`src/sase/ace/tui/actions/post_update_toast.py`).

Today that toast reports, per repo, the **version transition** (`old → new`) and a **diffstat** (`+ins / −del`, files
changed), plus an Agent-CLIs section and a summary tail. It does **not** tell the user _which commits_ the update
actually included.

Separately, the "SASE Admin Center → Updates" tab already shows, when the user presses `u`, a y/n confirmation panel
that lists the **incoming commits** the update _would_ pull, **grouped by repository**. We want to bring that same "here
are the commits, by repo" experience to the _post-restart_ toast — but reporting the commits that were **actually
applied**, not a pre-update forecast.

## Reference: how the `u` panel presents commits (the inspiration)

The confirm panel (`src/sase/ace/tui/modals/plugin_action_confirm_modal.py`) renders each repo group via
`build_incoming_commits_renderable()` in `src/sase/plugins/render_common.py`:

- A per-repo header: `↑ <label> — <N> incoming commits` (glyph + label in **bold cyan**, the `— N commits` tail in
  **cyan**).
- Indented commit lines: two-space indent, **dim** short SHA, two spaces, then the subject (`no_wrap`, ellipsis on
  overflow).
- An overflow line `  +<extra> more…` in **dim** when the total exceeds the shown cap.
- Groups are separated by a blank line; a multi-repo aggregate line reads
  `N repositories · M incoming commits · K unavailable`.

The data vocabulary is `RepoIncomingCommits(label, incoming)` where
`IncomingCommits(total, commits: tuple[CommitSummary(short_sha, subject), …], source, error)` and
`extra = total − shown`. Our post-restart section should reuse this **visual language** (glyph, cyan header, dim SHAs,
`+N more…`, per-repo grouping) so the two surfaces feel like one system. Two deliberate differences: the toast body is a
**Textual markup string**, not a Rich renderable (so we emit equivalent markup, we do not reuse the renderable), and the
header noun drops "incoming" (`— N commits`) because these commits are already applied.

## Design overview — "commits ride the diffstat rail"

The receipt already carries a per-repo diffstat that is **captured at the moment of the merge** (where old→new HEADs are
known), threaded through `DevUpdateOutcome` → `DevUpdateResult` → `build_update_receipt()` → the JSON codec → the toast
renderer. This is the reliable, offline, ground-truth rail. **We piggyback the applied-commit list onto that exact
rail** rather than re-deriving it later or reusing the pre-update forecast:

1. **Capture** the applied commits at the same merge point in `src/sase/dev_update/execute.py`, from the exact
   `old_head..new_head` range, right beside the existing diffstat capture.
2. **Carry** them on `DevUpdateOutcome`, then aggregate them into a new receipt field.
3. **Serialize** them through the receipt codec (with a format-version bump + back-compat).
4. **Render** them as a grouped-by-repo section in the post-restart toast, styled after the `u` panel, capped for
   compactness, and config-gated.

Because capture happens from the real merge heads, the toast reports exactly what landed on disk — not a forecast that
could drift between preview and apply. Whatever repos can show a diffstat can show commits; whatever can't (managed/PyPI
installs with no local checkout, mode-switches) simply show no commit section, exactly as they show no diffstat today.
This "commits appear wherever diffstats appear" rule keeps the feature intuitive and its failure modes graceful.

## Data model changes

### 1. `src/sase/dev_update/models.py` (lowest layer — no upward imports)

Add two lean, frozen dataclasses and one field:

- `RepoCommit(short_sha: str, subject: str)` — one applied commit.
- `RepoCommitLog(total: int, commits: tuple[RepoCommit, …])` with an `extra` property (`max(0, total − len(commits))`),
  mirroring `IncomingCommits.extra`.
- Add `commits: RepoCommitLog | None = None` to `DevUpdateOutcome`, exactly paralleling the existing
  `diffstat: RepoDiffStat | None = None`.

Define these here (not by reusing `updates.incoming_commits.CommitSummary`) to preserve layering:
`updates/incoming_commits.py` already imports `dev_update.models`, so `dev_update` must not import back into `updates`.
`RepoCommit` intentionally mirrors `CommitSummary`'s shape so the two subsystems stay visually and semantically
parallel.

### 2. `src/sase/ace/_update_receipt_models.py`

- Import `RepoCommitLog` from `sase.dev_update.models` (this module already imports `RepoDiffStat` from there — same
  lean dependency, no cycle).
- Add a grouping struct: `RepoCommitGroup(label: str, commits: RepoCommitLog)`.
- Add to `UpdateToastReceipt`: `commit_groups: tuple[RepoCommitGroup, …] = ()` and `commit_group_overflow: int = 0`.
- Add `MAX_COMMIT_GROUPS = 5` (repos shown in the commits section), alongside the existing `MAX_PLUGIN_LINES` /
  `MAX_PROVIDER_LINES`.
- Bump `FORMAT_VERSION` from `2` to `3`.

A **dedicated `commit_groups` field** (rather than nesting commits under the existing primary/plugin transitions) is
chosen deliberately: the transition list buckets repos as primary / plugins / dependency-**count**, which hides
`sase-core` (role `core`) inside a number. The user asked for commits "separated by which repo they belong to," so the
commits section must be able to show **every** changed repo — sase, sase-core, and each plugin — independent of that
bucketing. Keeping it a separate field also leaves all existing transition/diffstat logic untouched (lower risk); we
only add.

## Capture — `src/sase/dev_update/execute.py`

`_merge_actionable_roots()` already computes, per actionable git root: `old_head = _best_effort_head(...)`, performs the
ff-only merge, `new_head = _best_effort_head(...)`, and
`root_diffstats[root] = _best_effort_diffstat(root, old_head, new_head, …)`.

Add a sibling `_best_effort_commit_log(git_root, old_head, new_head, run, commands, *, limit)` that returns
`RepoCommitLog | None`, built exactly like the incoming-commit fetch in `updates/incoming_commits.py`:

- Bail to `None` if either head is missing (mirrors `_best_effort_diffstat`).
- Total: `git -C <root> rev-list --count <old>..<new>` → int.
- Capped list: `git -C <root> log -n<limit> --format=%h%x1f%s%x1e <old>..<new>`, parsed with a small local splitter on
  the `\x1e` / `\x1f` separators into `tuple[RepoCommit, …]`. Reuse the same format sentinels the update subsystem
  already uses (define local module constants; do **not** import `updates.incoming_commits` from `dev_update` — that
  would invert the layering).
- Because the merge is fast-forward-only (guaranteed by the preflight `behind > 0`, `ahead == 0` check), `<old>..<new>`
  is exactly the set of pulled commits.

Thread a `root_commits: dict[str, RepoCommitLog | None]` out of `_merge_actionable_roots` (extend its return tuple) and
into `_success_outcomes(...)`, which sets `DevUpdateOutcome.commits = root_commits.get(pkg.git_root)` next to where it
already sets `diffstat`. Add a capture cap constant `DEV_UPDATE_COMMIT_LOG_CAPTURE_LIMIT` (e.g. `20`) — generous so the
display cap can be tuned without recapture; `total` (from `rev-list --count`) stays authoritative for the "+N more…"
math regardless of the capture cap.

Keep all of this behind the injected `DevCommandRunner`, so tests drive it with canned git output and no real
subprocesses, exactly like the diffstat path.

## Receipt plumbing

### `src/sase/ace/_update_receipt_builders.py`

Add a helper `_commit_groups_from_dev_outcomes(updated_outcomes) -> (groups, overflow)` and call it from
`_build_dev_receipt` and `_build_combined_receipt` (the dev leg). It:

- Iterates updated dev outcomes whose `commits` is non-None and `commits.total > 0`.
- **Dedups by `git_root`** (a root can back multiple packages) so each repo yields one group.
- Orders groups **primary (sase) → core → plugins → rest** to mirror the confirm panel's ordering; label each group from
  the representative outcome's `record.name` (the same name the diffstat transitions already use, keeping labels
  consistent across the two blocks).
- Caps at `MAX_COMMIT_GROUPS`, returning the remainder as `commit_group_overflow`.

Populate `commit_groups` / `commit_group_overflow` on the returned `UpdateToastReceipt`. `_build_comprehensive_receipt`
builds its base via `build_update_receipt(result.sase.payload)` and then `replace(base, provider_results=…)`, which
**preserves** `commit_groups` — so the `,U` comprehensive path and the plain `sase update` CLI path both gain commits
uniformly. Managed (`_build_managed_receipt`), mode-switch, and plugin-op receipts leave `commit_groups` empty — they
have no local git range to read, matching their absent diffstats.

### `src/sase/ace/_update_receipt_codec.py` (format bump + compatibility)

The **writer runs the pre-update (old) code and the reader runs the post-update (new) code**, so cross-version
compatibility is load-bearing:

- Set `FORMAT_VERSION = 3`; accept `format ∈ {1, 2, 3}` in `receipt_from_json` (replace the
  `{LEGACY_FORMAT_VERSION, FORMAT_VERSION}` membership check with a supported-range check).
- **Fix the provider-results gate**: today provider results decode only when `format_version == FORMAT_VERSION`. With
  the bump that would wrongly drop providers from a format-2 receipt written by the outgoing version during the very
  first upgrade. Change the gate to `format_version >= 2`.
- Serialize commits in `_transition`-adjacent helpers: add `commit_groups` (list of
  `{label, total, commits: [{short_sha, subject}, …]}`) and `commit_group_overflow` to `receipt_to_json`. In
  `receipt_from_json`, decode them **only when the key is present** (format-2 payloads simply have no key → empty
  groups), with strict validation: `label` non-empty str, `total` non-negative int, each commit `{short_sha, subject}`
  non-empty str, and reject the whole receipt on malformed entries (consistent with the existing strict decoders). No
  explicit format gate is needed at the field level — "decode if present" is robust to both older (absent) and newer
  receipts.

Downgrades (a format-3 receipt read by an older format-2-only reader) are already handled by the strict format check →
the reader ignores the receipt and shows no toast, an acceptable degradation.

## Rendering — `src/sase/ace/tui/actions/post_update_toast.py` (the beautiful part)

Add a grouped commits section, gated and capped, inserted into `_format_post_update_toast_message` between the
version/diffstat summary block and the Agent-CLIs block. Structure per repo, mirroring the `u` panel in markup:

```
[bold cyan]↑ sase[/] [cyan]— 7 commits[/]
  [dim]abc1234[/]  fix: prevent restart race before receipt flush
  [dim]def5678[/]  feat: add grouped commit toast
  [dim]+5 more…[/]

[bold cyan]↑ sase-core[/] [cyan]— 2 commits[/]
  [dim]9a1b2c3[/]  perf: faster numstat parse
  [dim]4d5e6f7[/]  refactor: tidy diff facade

• sase-github  1.2 → 1.3   [bold green]+4[/] [#D75F5F]−1[/]
  … (version/diffstat summary block, unchanged, above the commits block)
```

Details:

- New helper `_commit_group_lines(receipt, *, max_per_repo) -> str`. Returns `""` when the gate is off or there are no
  groups. For each group emit the cyan header `↑ <label> — <N> commits` (singular/plural noun), then up to
  `max_per_repo` commit lines `  [dim]<short_sha>[/]  <subject>`, then `  [dim]+<extra> more…[/]` where
  `extra = total − shown`. Separate groups with a blank line; append
  `[dim]…and <commit_group_overflow> more repositories[/]` when nonzero.
- **Escape** every SHA/subject/label with `textual.markup.escape` (subjects can contain `[`).
- **Truncate** long subjects to a max width (add `_COMMIT_SUBJECT_MAX`, e.g. `58`, matching the pre-update toast's
  `_COMMIT_SUBJECT_WIDTH`) with a trailing `…`.
- Fold the aggregate into the existing tail: extend `_tail_line` to prepend `"<M> commits"` (from
  `sum(g.commits.total for g in receipt.commit_groups)`) when the commits section is shown, so the tail reads e.g.
  `9 commits · 540 files changed · +3 dependencies · Reloaded into the new version.` — the compact toast analogue of the
  confirm panel's aggregate line.
- Assemble message parts as `[summary, commits, agent_clis]` joined by `"\n\n"`, skipping empty parts. Works for both
  the legacy and diffstat summary layouts.
- Severity/timeout/`markup=True` are unchanged.

## Config — `src/sase/default_config.yml` + `update_toast.py`

Add two knobs under `ace.updates`, mirroring the existing `post_update_toast*` and `startup_toast_max_commits`
conventions:

- `default_config.yml`: `post_update_toast_commits: true` and `post_update_toast_max_commits: 5` in the `updates:`
  block.
- `_UpdateToastConfig` (`update_toast.py`): add `post_update_toast_commits: bool = True` and
  `post_update_toast_max_commits: int = 5`.
- `_load_update_toast_config()`: read both via the existing `_coerce_bool` / `_coerce_nonnegative_int` helpers.
- `post_update_toast.py` reads `config.post_update_toast_commits` (gate) and passes
  `config.post_update_toast_max_commits` into `_commit_group_lines`.

Per `CLAUDE.md`, `default_config.yml` is the source of truth for config values — keep it in sync with the dataclass
defaults.

## Rust Core Backend Boundary assessment

`CLAUDE.md` flags a Rust core boundary. Per the research, the entire update subsystem (`updates/`, `dev_update/`,
receipts) is Python and does **not** route through `sase_core_rs`; its git operations (`rev-list`, `log`,
`diff --numstat`, ancestry, ff-merge) all shell out via `subprocess` and parse in Python. The Rust-backed
`parse_git_log` in `core/vcs_log_facade.py` serves a _different_ domain (`sase vcs log` / ChangeSpec queries), not
updates.

Decision: implement commit capture in Python inside `dev_update`, consistent with the surrounding diffstat/ancestry code
it sits beside. This does not deepen the boundary violation — it follows the established local pattern of a
self-contained Python update subsystem, and a future wholesale migration of update-git logic behind a `sase_core_rs`
binding would carry this along with `_fetch_git_incoming_commits`/`_best_effort_diffstat`. Deliberately not reused here:
the `vcs_log` Rust facade (avoids scope creep and a heavier, author/date-rich wire type for a trivial short-sha+subject
capture). Called out explicitly so review does not read this as an oversight.

## Edge cases & reliability

- **Missing head / failed `git`**: `_best_effort_commit_log` returns `None` (no section for that repo), same graceful
  degradation as the diffstat.
- **Managed/PyPI or mode-switch updates**: no local range → no `commit_groups`; toast is unchanged from today.
- **Empty range** (defensive; preflight guarantees `behind > 0`): `total == 0` → group is skipped by the builder filter.
- **Huge update**: capped at capture (`DEV_UPDATE_COMMIT_LOG_CAPTURE_LIMIT`) and again at display
  (`post_update_toast_max_commits`); `+N more…` stays accurate via the authoritative `rev-list --count` total; repos
  beyond `MAX_COMMIT_GROUPS` collapse into `…and N more repositories`.
- **Toast height**: aggressive caps (few commits/repo, few repos) keep the 10s toast compact.
- **Freshness/one-shot**: unchanged — the 30-minute freshness gate and read-and-clear semantics still apply.
- **Markup safety**: all dynamic text escaped; commit subjects truncated.

## Testing

Extend the existing suites (locate via ripgrep for the current test modules — e.g. `test_dev_update_execute`,
`test_update_receipt_codec`, `test_update_receipt_builders`, `test_post_update_toast`):

- **Capture** (`dev_update/execute`): with an injected runner returning canned `rev-list` `--count` and `log` output, a
  merge populates `DevUpdateOutcome.commits` with the right total and capped list; a failing/absent head yields `None`.
- **Codec**: round-trip a format-3 receipt with `commit_groups`; a format-2 payload (no commit keys) decodes to empty
  groups **and still decodes `provider_results`** (regression guard for the gate change); malformed commit entries
  reject the receipt.
- **Builders**: a dev/combined result with per-root commits yields ordered, git-root-deduped `commit_groups` labelled
  correctly, capped with overflow; managed / mode-switch results yield no groups.
- **Rendering**: the toast message contains the grouped section with the cyan header, dim SHAs, `+N more…`,
  `…and N more repositories`, subject truncation, and the commit count in the tail; the gate off (or zero groups) omits
  the section entirely; markup in a subject is escaped.
- If the post-update toast has a **PNG visual snapshot** in the ACE visual suite, add/refresh a golden via
  `just test-visual --sase-update-visual-snapshots` for a receipt with commits.

Run `just install` then `just check` (and `just test-visual` if goldens change) before finishing, per `CLAUDE.md`.

## Out of scope / possible follow-ups

- Bringing applied-commit groups to the `sase update` **CLI** render output (this plan targets the ACE post-restart
  toast; the CLI already restarts axe separately).
- GitHub-`compare`-sourced commits for **managed/PyPI** updates (would need a best-effort, time-boxed network fetch at
  receipt-build; intentionally excluded to keep the restart fast and offline-reliable).
- Prettier repo labels / linking commit SHAs to GitHub.

## File-change checklist

- `src/sase/dev_update/models.py` — `RepoCommit`, `RepoCommitLog`, `DevUpdateOutcome.commits`.
- `src/sase/dev_update/execute.py` — `_best_effort_commit_log`, thread `root_commits` into outcomes, capture-limit
  constant + log-format sentinels.
- `src/sase/ace/_update_receipt_models.py` — `RepoCommitGroup`, receipt fields, `MAX_COMMIT_GROUPS`,
  `FORMAT_VERSION = 3`.
- `src/sase/ace/_update_receipt_builders.py` — `_commit_groups_from_dev_outcomes`, wire into dev/combined receipts.
- `src/sase/ace/_update_receipt_codec.py` — serialize/deserialize commit groups; accept format 3; fix provider gate to
  `>= 2`.
- `src/sase/ace/tui/actions/post_update_toast.py` — `_commit_group_lines`, tail commit count, message assembly,
  subject-truncation constant.
- `src/sase/ace/tui/actions/update_toast.py` — two new `_UpdateToastConfig` fields + loader.
- `src/sase/default_config.yml` — two new `ace.updates` keys.
- Tests across the suites listed above.
