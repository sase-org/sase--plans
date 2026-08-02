---
tier: epic
title: 'Excellent @commit: argument completion in the prompt bar and LSP'
goal: 'Typing `@commit:` offers the project''s recent revisions across every one of
  its repos — in the ACE prompt bar and in any LSP editor — ranked by relevance then
  recency, rendered as a short SHA plus the commit subject, and every row it offers
  resolves at launch.

  '
phases:
- id: core_menu
  title: Scoped payload rows and stable payload ranking
  depends_on: []
  size: medium
  description: 'core_menu: add `scope`, `rank`, and `body` to the at-reference payload
    row wire, match a qualified `scope@title` target so a repo fragment and subject
    words combine in one query, and make provider rank the tiebreak after tier and
    score so recency survives a typed query.

    '
- id: core_commits
  title: Local commit enumeration in the shared payload inventory
  depends_on:
  - core_menu
  size: medium
  description: 'core_commits: enumerate each repository checkout''s recent revisions
    with a bounded, timed `git log`, emit rows keyed by a 12-char-floored abbreviated
    SHA merged across repos in recency order, and drop the `commit` early-out that
    made the kind unenumerable.

    '
- id: core_bridge
  title: Python binding for the payload inventory
  depends_on:
  - core_commits
  size: small
  description: 'core_bridge: expose `artifact_ref_payload_inventory(kind, context)`
    through `sase_core_rs`, release the sase-core version that carries it, and widen
    the `sase-core-rs` dependency window so the host can consume it.

    '
- id: lsp_items
  title: Commit-aware LSP completion items
  depends_on:
  - core_commits
  size: small
  description: 'lsp_items: describe payload items by their artifact kind instead of
    always "file", mark identifier-backed kinds as references, and render the commit
    body into item documentation.

    '
- id: tui_commits
  title: Pane-independent commit snapshots in the prompt bar
  depends_on:
  - core_bridge
  size: medium
  description: 'tui_commits: replace the mounted-Commits-pane projection with a TTL-revalidated
    background snapshot from the shared inventory, show a loading row while the first
    snapshot lands, and render commit rows with a dimmed repo segment and no duplicated
    detail.

    '
- id: docs_verify
  title: Documentation and end-to-end verification
  depends_on:
  - lsp_items
  - tui_commits
  size: small
  description: 'docs_verify: correct the editor and getting-started documentation
    that still says commit references are not enumerated, and verify the completion-resolution
    invariant end to end in both surfaces.'
proposed_by: bbugyi200.athena.ry
create_time: 2026-08-02 10:03:46
status: wip
bead_id: sase-e8
---

- **PROMPT:** [prompts/202608/commit_ref_completion.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/commit_ref_completion.md)
- **BEAD:** [sase-e8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-e8/README.md)

# Plan: Excellent `@commit:` Argument Completion

## Goal

`@commit:<repo>@<sha>` is the only artifact-reference kind whose payload a user must type entirely from memory. This
plan makes it the easiest one to reach — in the ACE prompt bar and in every LSP editor — without weakening the
guarantees the other kinds already have.

Success is three properties, in priority order:

1. **Reliable.** Every row the menu offers resolves at launch. Completion never depends on which TUI pane the user has
   visited, and never contacts the network.
2. **Intuitive.** An empty query lists the project's most recent commits, newest first. A query narrows by repo, by SHA
   prefix, or by subject words, in any combination, without the user having to know which field they are matching.
3. **Beautiful.** A row is a short SHA and a commit subject — not forty hex characters — and the inserted reference is
   short enough to read inside a sentence.

## Current state

Established by reading the code; implementing agents should trust these as the starting point but re-verify line
numbers, which will drift.

### The prompt bar completes commits only from a mounted pane

`src/sase/ace/tui/widgets/_file_completion_base.py:499` projects commit candidates by querying the live widget
`#artifacts-commits-pane` and reading its `result` attribute. `ArtifactsView` composes `CommitsPane` eagerly
(`src/sase/ace/tui/widgets/artifacts/view.py:85`) but pane collection is lazy, so `result` stays `None` until the user
visits **Artifacts → Commits** in that session. Consequences:

- `@commit:` offers **zero** rows in a fresh session until an unrelated tab is visited.
- Once populated, the rows are whatever that pane's author/date/project filters left behind. A filter set for browsing
  silently censors prompt completion.
- The pane snapshot ignores the prompt's own target project, unlike the sibling bug projection
  (`_snapshot_artifact_ref_bug_candidates`, same file), which does scope to `_xprompt_arg_assist_project_from_text()`.

### The LSP completes nothing at all

`crates/sase_core/src/editor/completion.rs:414` returns an empty inventory for `commit` and `bug`, and
`crates/sase_xprompt_lsp/src/server.rs:854` short-circuits the same two kinds before consulting its cache.
`docs/editor.md` documents this as intended: _"`commit` and `bug` references receive shape validation but no completion
enumeration or resolution request."_

That framing conflates two very different kinds. `bug:` needs an issue tracker, which is a network provider and
correctly out of scope. `commit:` needs only a local checkout, and every checkout path is already in
`ArtifactRefContextWire.repositories[].checkout_paths`. The same document's governing constraint — _"Artifact assistance
is local-only"_ — permits commit enumeration; nothing but the missing implementation prevents it.

### Rows are forty hex characters, and typing destroys recency

The TUI projection inserts `commit.full_id` (`_file_completion_base.py:519`), so the menu label, the inserted text, and
the fuzzy-match target are all `<repo>@<40 hex>`. Two consequences:

- A completed reference is a 55-character token in prose.
- With a non-empty query, `build_payload_menu` (`crates/sase_core/src/editor/at_reference.rs:544`) ranks by tier, then
  score, then character count, then case-insensitive text. Every `sase-core@<40 hex>` row has the same length and the
  same score for the query `core`, so the final tiebreak decides — and that tiebreak is **lexicographic SHA order**. The
  moment a user types a repo fragment, the recency ordering they were relying on is replaced by hexadecimal noise.

### Resolution already works, and is stricter than completion

`resolve_commit` (`crates/sase_core/src/artifact_ref/mod.rs:595`) prefix-matches `ArtifactRefRepository.shas`, which
`artifact_ref_context()` never populates, so it always reports `missing`. `_resolve_for_launch`
(`src/sase/artifact_ref_prompt.py:173`) covers that by resolving `<sha>^{commit}` through the repository's checkout and
re-resolving. So a 7-to-40-character SHA prefix resolves correctly today, an ambiguous or unknown one fails loudly at
launch, and `resolution.locator` canonicalizes to the full 40-character SHA regardless of what was typed.

This is what makes a short inserted SHA safe, and it is the invariant the whole plan hangs on.

## Design

### D1 — Enumeration lives in `sase_core`, reading local checkouts

Per the Rust core backend boundary rule, behavior an editor integration must match the TUI on is core backend logic.
Commit enumeration is exactly that, so it goes in `sase_core::editor::completion::build_artifact_ref_payload_inventory`
alongside every other kind. Both hosts then share one implementation, one ranking, and one row shape, which is what
makes the two surfaces agree by construction rather than by discipline.

The source is a bounded `git log` per checkout, spawned with `std::process::Command`. Precedent exists in
`crates/sase_core/src/bead/mutation.rs:6`. This is local I/O — no different in kind from the directory walks and file
reads the same function already performs for document roles.

**Rejected: materialize a commit catalog from Python at `sase lsp` startup**, mirroring
`_materialize_vcs_project_catalog`. Rejected because commits are the most volatile payload SASE has, and the single most
common use of `@commit:` is referencing the commit you _just_ made. A catalog written at editor startup would
systematically miss exactly that commit. Freshness is not a nice-to-have here; it is the feature.

**Rejected: route through the helper host bridge** (`CommandHelperHostBridge`) to reuse the Python `vcs_provider`
abstraction. Rejected because it spawns a Python interpreter on the completion path, and because `@commit:` is already
git-shaped end to end: the grammar demands 7-40 lowercase hex characters (`validate_sha`, `artifact_ref/mod.rs:997`) and
the existing launch resolver already issues `<sha>^{commit}`. Non-git checkouts are skipped rather than mis-served.

### D2 — Insert a 12-character SHA, floored by git's own unique length

`git log` is asked for both `%h` and `%H`. The emitted SHA is `%h` when git already needed 12 or more characters for
uniqueness, and the first 12 characters of `%H` otherwise. Because `%h` is unique at its length, any longer prefix of
the same commit is also unique, so the result is always unambiguous in the repository that produced it.

Twelve is a constant in `sase_core` (`ARTIFACT_REF_COMMIT_ABBREV`), not configuration. A knob here would let the prompt
bar and the LSP disagree about what the same commit is called, which is the one thing this plan exists to prevent. It
stays inside the grammar's 7-40 window, resolves through the existing launch path, and canonicalizes to the full SHA in
`resolution.locator` at launch, so nothing durable is shortened — only what the user reads and types.

### D3 — One query matches repo, SHA, and subject together

A payload row may declare a `scope`: a prefix of its own payload that names the qualifier segment. Commit rows set
`scope` to the repository name, so `scope` is the payload text before the `@`.

Matching a scoped row considers **three** targets instead of two:

| Target              | Text                             | Example match for `core@fix` |
| ------------------- | -------------------------------- | ---------------------------- |
| payload             | `sase-core@5143cb981f0a`         | —                            |
| title               | `fix(stats): expose …`           | —                            |
| **qualified title** | `sase-core@fix(stats): expose …` | tier 2, contiguous           |

The qualified target is `payload[..=scope.len()] + label` — the scope, its separator character taken verbatim from the
payload, then the title. The rule a user internalizes is simply: **the `@` you type means the same `@` that is in the
reference.** `core` alone still narrows to that repo through the payload target; `fix` alone still finds the subject
through the title target; `core@fix` now composes them, which it cannot today.

Highlight runs stay exact because the qualified text is a literal concatenation: runs below `scope.len()` map onto the
payload unchanged, and runs at or past `scope.len() + 1` map onto the title with that offset subtracted. A run
straddling the separator is reported on neither field.

**Rejected: split the payload stage into a repo stage and a SHA stage.** It would change the shared cursor policy in
`detect_at_reference_context` for every kind, and it would put a mandatory repo-selection step in front of the dominant
"show me my recent commits" flow.

**Accepted limitation:** a hex-only query such as `add` or `beef` can subsequence-match a SHA and produce a tier-3 row.
Tier 3 is the lowest tier, so genuine subject and repo matches always outrank it, and shortening the SHA from 40
characters to 12 shrinks the surface substantially. Document it rather than adding a prefix-only matching mode.

### D4 — Provider rank is the tiebreak after tier and score

Rows may declare `rank`, a provider-assigned ordinal. When both sides of a comparison carry one, it breaks ties
immediately after tier and score, ahead of the length and text comparisons. Commit rows set `rank` to their position in
the merged recency timeline, so recency survives narrowing instead of collapsing into SHA order.

Kinds that do not set `rank` keep today's ordering exactly — this is opt-in, and no existing golden changes. Chats are
also recency-ordered and are a natural later adopter; file a task bead for that rather than widening this plan.

Relevance still dominates recency: a strong subject match from last month outranks a weak one from an hour ago, which is
consistent with every other kind and is what makes ranking predictable.

### D5 — Rows show a short SHA and a subject, and nothing redundant

The row is `<repo>@<short sha>` as the label, the commit subject as the title, and the relative age as `age`. `detail`
is deliberately **empty**: the repository name is already in the label, and the author is not what distinguishes one
candidate from another — the subject is. Row width is scarce and it goes to the discriminator.

`age` reproduces `age_label` (`_artifact_ref_completion_menu.py:478`) exactly — `now`, `12m`, `3h`, `5d`, then an ISO
date past a week — so commit rows read the same as every other kind's rows.

In the prompt bar, the commit row's `segment_split` is set just past the `@`, which reuses the existing two-segment path
styling to dim the repository and keep the SHA bright. That is a one-expression change to a helper the row renderer
already calls.

### D6 — Freshness by TTL, warm reads on the keystroke path

Neither host may do git I/O while the user is typing.

- **LSP:** the payload inventory is already cached per project and kind and invalidated by catalog signature, explicit
  refresh, or a two-second TTL (`ARTIFACT_REF_CACHE_TTL`, `crates/sase_xprompt_lsp/src/server.rs:75`). Commits inherit
  that unchanged, which is the right interval for this data.
- **Prompt bar:** the warm catalog in `_artifact_ref_highlight.py` is loaded once per project per widget and never
  refreshed, so commits must not live in it. They get their own snapshot slot, modeled directly on
  `prompt_path_inventory.py`: a background worker loads it, a cheap TTL check revalidates it, and the existing
  `commits=` parameter of `build_artifact_ref_completion_result` remains the seam. The keystroke path only ever reads a
  warm tuple, and `snapshot_payload_memo` continues to memoize the native index by object identity.

### D7 — Bounds are enforced and disclosed

Per repository: at most `ARTIFACT_REF_COMMIT_SCAN_LIMIT = 200` commits from `HEAD`, and a wall-clock cap of two seconds
after which the child is killed and that repository contributes nothing. Across repositories: the merged timeline is
capped at `ARTIFACT_REF_COMMIT_MAX_ROWS = 1000`. Commits dropped by the merge cap are counted into `truncated_payloads`,
which both hosts already surface as `at least N additional payloads not shown`.

One checkout per repository — `checkout_paths[0]`, the workspace-preferred clone. Repositories with no checkout, or
whose checkout is not a git working tree, are skipped silently.

### D8 — Scope boundaries

- `bug:` is untouched. It needs a network tracker; the LSP's local-only contract stands.
- No new configuration surface (see D2).
- No change to the reference grammar, to `resolve_commit`, or to `_resolve_for_launch`.
- The reference invariant below is not aspirational — it is the acceptance test.

> **Invariant.** Every reference this feature completes must resolve at launch. A row is only emitted for a repository
> present in `ArtifactRefContextWire.repositories`, using that repository's canonical `name`, and a SHA long enough for
> git itself to call unique. `docs_verify` proves it end to end.

---

## Phase `core_menu` — Scoped payload rows and stable payload ranking

Work in the linked `sase-core` repository. Open it with the `/sase_repo` skill and use the path it prints.

### Wire

Extend `AtReferencePayloadRowWire` (`crates/sase_core/src/editor/at_reference.rs:71`) with three fields, each
`#[serde(default)]` so every existing producer, golden vector, and the JSON-passthrough PyO3 binding keep working
untouched:

- `scope: String` — a prefix of `payload` naming the qualifier segment. Empty means unscoped.
- `rank: Option<u32>` — provider-assigned ordinal, ascending.
- `body: String` — optional long-form text for editor documentation.

Validate defensively at index-construction time: if `scope` is non-empty but is not a prefix of `payload`, or if
`payload` has no character immediately after it, treat the row as unscoped rather than panicking. Wire data crossing a
process boundary must never be able to abort the server.

### Matching

In `AtReferencePayloadIndex::new`, prepare a third `FuzzyText` per scoped row from `payload[..=scope.len()] + label`.
Extend `MatchedPayload` with that target and its match, and include it in `best_match` / `best_prepared_match` selection
using the same `compare_fuzzy` rule that already picks between payload and title.

Run reporting when the qualified target wins:

- runs entirely within `[0, scope.len())` are reported as `label_match` on the payload;
- runs entirely at or past `scope.len() + 1` are reported as `title_match`, shifted down by `scope.len() + 1`;
- a run straddling the separator is reported on neither.

Mirror the same behavior in the non-indexed `inventory.payloads` branch of `build_payload_menu` so the two paths cannot
diverge.

### Ranking

Insert `rank` into the payload comparator immediately after tier and score, before the character count and text
comparisons, in both `compare_payload_matches` and the prepared path. Compare ascending. When either side is `None`,
skip the rank comparison entirely so unranked corpora are bit-for-bit unchanged.

### Tests

- A scoped row matched by `core@fix` when neither payload nor title alone matches.
- Highlight runs from a qualified match landing on the correct field with the correct offsets, including the
  straddling-run case.
- Two rows with identical tier and score ordering by ascending `rank`, and the same pair with no ranks preserving
  today's text ordering.
- A malformed `scope` (not a prefix of `payload`) degrading to an unscoped row.
- Every existing at-reference menu test still passing unmodified. If a golden changes, the change is a defect in this
  phase, not in the golden.

---

## Phase `core_commits` — Local commit enumeration in the shared payload inventory

Work in the linked `sase-core` repository.

### Enumeration

Add `append_commit_candidates(payloads, seen, context) -> usize` to `crates/sase_core/src/editor/completion.rs` and
dispatch to it from `build_artifact_ref_payload_inventory` for `kind == "commit"`. Narrow the existing
`matches!(kind, "commit" | "bug")` guard at the top of that function to `bug` only.

For each `ArtifactRefRepositoryWire` in `context.repositories`, take `checkout_paths[0]`, skip it unless it is a
directory containing a `.git` entry (a file or a directory — worktrees and linked clones use a file), and run:

```
git --no-pager -C <checkout> log --no-color -n 200 -z \
    --format=%H%x1f%h%x1f%at%x1f%s%x1f%b HEAD
```

Records are NUL-separated, fields are unit-separated, so subjects and multi-line bodies survive verbatim. Spawn with
`stdin` null and `stderr` null. Enforce the two-second cap by polling `try_wait` against a deadline and killing the
child on expiry; a timed-out or non-zero-exit repository contributes nothing and does not fail the inventory.

Set `GIT_OPTIONAL_LOCKS=0` in the child environment so completion cannot contend with an interactive git command in the
same checkout.

### Rows

Per commit, with `ARTIFACT_REF_COMMIT_ABBREV = 12`:

- `payload` = `<repository.name>@<sha>`, where `sha` is `%h` when `%h.len() >= 12`, else `%H[..12]`. Use
  `repository.name`, never a directory basename or an alias — the invariant depends on it.
- `label` = the subject, trimmed. Empty subjects fall back to the abbreviated SHA.
- `scope` = `repository.name`.
- `detail` = `""` (D5).
- `age` = a relative label reproducing Python's `age_label` exactly. Add `commit_age_label(timestamp, now)` with `now`
  injectable so tests are deterministic.
- `body` = the trimmed body, empty when there is none.
- `rank` = assigned after the merge, below.

Merge every repository's rows into one timeline sorted by author timestamp descending, breaking ties by repository name
then abbreviated SHA so the order is total and reproducible. Truncate to `ARTIFACT_REF_COMMIT_MAX_ROWS = 1000`,
returning the number dropped as `truncated_payloads`. Assign `rank` as the post-merge index, so rank ordering _is_
recency ordering.

Deduplicate on `payload` through the existing `seen` set: the same commit reachable through two repository records must
appear once.

### Tests

Use `tempfile` and real `git init` fixtures with pinned author dates via `GIT_AUTHOR_DATE` / `GIT_COMMITTER_DATE`,
following the crate's existing filesystem-fixture style.

- Rows in recency order across two repositories, with `rank` matching position.
- The abbreviation floor: a short history whose `%h` is 7 characters still yields a 12-character payload SHA.
- Every emitted `payload` parsing through `parse_artifact_ref` and naming a repository present in the context — the
  invariant, asserted mechanically.
- A subject containing a NUL-hostile mix of quotes, tabs, and non-ASCII surviving verbatim.
- A multi-line body landing in `body` intact.
- A checkout path that does not exist, is not a git tree, or has no commits, each skipped without failing the inventory.
- The merge cap reporting a non-zero `truncated_payloads`.
- `bug` still returning an empty inventory.

---

## Phase `core_bridge` — Python binding for the payload inventory

Work in the linked `sase-core` repository, then in this repository for the dependency window.

Add `artifact_ref_payload_inventory(kind: str, context: dict) -> dict` to `crates/sase_core_py`, following the
JSON-passthrough shape of the neighbouring `at_reference_menu` binding: deserialize `context` into
`ArtifactRefContextWire` with a `PyValueError` on malformed input, call `build_artifact_ref_payload_inventory`, and
return the serialized `AtReferenceInventoryWire`. Register it in the module init and add it to the binding inventory
listed in the crate's module docstring.

Add a binding test asserting the returned dict carries `payloads` and `truncated_payloads` and that the new row fields
round-trip.

Release the sase-core version carrying it, then widen the `sase-core-rs` window in this repository's `pyproject.toml`
(currently `>=0.17.8,<0.18.0`) to require the new floor. Run `tools/validate_sase_core_rs_version` — the Justfile
already invokes it and warns when the published window lags a local checkout.

Note for later phases: `just install` builds `sase_core_rs` from the local sase-core checkout when one is present, so
`tui_commits` can proceed against the local build without waiting on a publish.

---

## Phase `lsp_items` — Commit-aware LSP completion items

Work in the linked `sase-core` repository. Independent of `core_bridge`; both depend only on `core_commits`.

In `crates/sase_xprompt_lsp/src/lsp_convert.rs:85`, payload rows are currently hardcoded to `CompletionItemKind::FILE`
with the description `"file"`, so a commit appears in editor menus with a file icon labelled "file". Thread
`context.kind` into `at_reference_completion_item` and derive:

- `labelDetails.description` = the artifact kind word (`commit`, `chat`, `research`, `bead`, …). This is a strict
  improvement for every kind.
- `CompletionItemKind::REFERENCE` for identifier-backed kinds — `commit`, `bead`, `agent` — and `FILE` for
  filesystem-backed kinds. Kind-stage and file-stage rows are unchanged.

Append the row's `body` to `at_reference_documentation` as a fenced block below the existing matched payload and title
lines, omitted when empty, and truncated to a sane number of lines so a large commit message cannot flood a hover popup.

Remove the `commit` short-circuit in `cached_at_reference_payload_inventory`
(`crates/sase_xprompt_lsp/src/server.rs:854`), leaving `bug`. Commits then inherit the existing per-project, per-kind
cache and its two-second TTL with no further change.

### Tests

- A commit payload row converting to an item with `REFERENCE` kind, a `commit` description, the subject in
  `labelDetails.detail`, and the body in `documentation`.
- Existing conversion tests for document, chat, and file rows updated only where the description word changed, with no
  change to `label`, `filterText`, `sortText`, or `textEdit`.
- A server-level test driving `completion_for_text` over `@commit:` against a temporary catalog whose project points at
  a real git fixture, asserting non-empty ranked items.

---

## Phase `tui_commits` — Pane-independent commit snapshots in the prompt bar

Work in this repository.

### Snapshot source

Add `src/sase/ace/tui/widgets/prompt_commit_inventory.py`, modeled closely on `prompt_path_inventory.py`:

- `PromptCommitSnapshot(project, rows, loaded_at)` — immutable, with `rows` a tuple of `ArtifactRefCommitRow` mapped
  one-to-one from the core inventory rows (`payload`, `label`, `detail`, `age`, `scope`, `rank`, `body`). Map, do not
  recompute: the host must not re-derive a label or an age that the core already decided, or the two surfaces will
  drift.
- `load_prompt_commit_snapshot(project, context)` — calls the new binding. Performs I/O; worker only.
- `revalidate_prompt_commit_snapshot(previous, ...)` — returns `previous` unchanged inside the TTL so an unchanged
  snapshot skips a menu refresh, exactly as the path snapshot does.

Use a two-second TTL to match the LSP.

### Widget wiring

In `_file_completion_base.py`, replace `_snapshot_artifact_ref_commit_candidates` — delete the `#artifacts-commits-pane`
query and the `_artifact_ref_commit_projection` cache — with a warm read of the new snapshot, keyed by
`_xprompt_arg_assist_project_from_text()` so commits are project-scoped like bugs already are. On a cold or expired key,
start a worker in a new `prompt-commit-inventory` group and handle its completion in the existing
`on_worker_state_changed` chain, refreshing the open menu when the landing snapshot belongs to the project currently in
the prompt.

While the first snapshot for a project is in flight, `build_artifact_ref_completion_result` emits a single
non-selectable `AtReferenceLoadingCompletionMetadata` row. That metadata type and its dim renderer already exist; today
the row is only emitted in the kind stage (`_artifact_ref_completion_menu.py:199`). Extend the condition to the payload
stage when the provider is loading and there are no candidates yet, and give it commit-appropriate text.

Keep the `commits=` parameter of `build_artifact_ref_completion_result` and the `ArtifactRefCommitCandidate` type name
if it eases the diff, but the rows must now carry core-supplied `label`/`age`/`scope`/`rank` rather than a repo string
and a raw timestamp. `payload_rows` must map `rank` and `scope` into the inventory dicts it hands to
`at_reference_inventory`, or the ranking work from `core_menu` will not reach the menu.

### Rendering

In `_prompt_input_bar_completion_rows_artifacts.py`, compute `segment_split` for commit rows as the offset just past the
`@` in the display text, instead of `_basename_start`, so the repository segment renders dim and the SHA bright. The
`[G]` badge stays. With `detail` now empty, the tail collapses to `subject  ·  age`, which is the intended shape.

### Tests

- `@commit:` in a fresh app with the Commits pane never activated producing ranked rows — the regression that motivates
  this phase.
- The offered insertion round-tripping through `parse_artifact_ref` and rendering as `@commit:<repo>@<12 hex>`.
- Commits scoping to the project named by a leading `#gh:`/`#git:` token in the prompt.
- The loading row appearing before the first snapshot lands and being non-selectable.
- A stale snapshot inside the TTL not triggering a reload; an expired one triggering exactly one.
- No test, and no source file, still referencing `#artifacts-commits-pane` for completion.

---

## Phase `docs_verify` — Documentation and end-to-end verification

Work in this repository. Runs after both surfaces are complete.

### Documentation

`docs/editor.md` is the authority on LSP completion behavior and currently contradicts the shipped feature in several
places. Correct all of them:

- The sentence excluding `commit` from enumeration must now exclude only `bug`, and should say why: commit assistance
  reads local checkouts, whereas bug assistance would need a network tracker.
- The capability table row for artifact references, and the fuzzy-matching section, gain the qualified `scope@title`
  target with `@commit:core@fix` as the worked example.
- Add the ranking tiebreak: tier, score, provider rank where declared, then length and text.
- Document the commit bounds — 200 commits per repository, 1000 merged rows, a two-second per-repo cap — alongside the
  existing 5000-payload and 200-row disclosures.
- Correct the `labelDetails.description` description, which no longer always says `file`.
- Note the accepted tier-3 hex-subsequence limitation from D3.

`docs/getting_started.md:159` documents `@commit:<repo>@<sha>` in the reference table; extend the surrounding prose to
say the prompt bar and LSP both complete it. Check `docs/ace.md` and `docs/vcs.md` for any statement this plan
invalidates.

### Verification

- Both hosts offering the same rows for the same project: compare the LSP's items for `@commit:` against the prompt
  bar's candidates and assert equal payload sequences. This is the check that keeps the two surfaces honest as they
  evolve.
- The invariant: for every row both surfaces offer, `parse_artifact_ref` succeeds and `_resolve_for_launch` returns a
  non-`missing` status against a real repository context.
- `sase doctor` reporting no new failures.
- `just install && just check` clean. Per this repository's instructions, `just install` first — workspace virtual
  environments go stale.
